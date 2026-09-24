# 175_DATABASE_TRANSACTION_EVENT_SYSTEM.md

# VoltStack Quantum Database
## Database Transaction Event System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 175 — Database Transaction Event System  
**Bloque:** 15 — Transactions & Concurrency  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `174_DATABASE_CONCURRENCY_CONTROL_SYSTEM.md`  
**Siguiente documento:** `176_DATABASE_READ_WRITE_CONNECTION_SYSTEM.md`

---

# 1. Propósito

`Database Transaction Event System` define la arquitectura mediante la cual VoltStack podrá observar, extender y coordinar acontecimientos relevantes del ciclo de vida de una transacción sin acoplar el `TransactionManager` a Telemetry, ORM, Cache, Domain Events, Queue, Outbox u otros subsistemas.

El sistema permitirá representar acontecimientos como:

```text
transaction created
transaction started
transaction marked rollback-only
before commit
commit confirmed
before rollback
rollback confirmed
transaction failed
transaction outcome unknown
savepoint created
savepoint rolled back
retry scheduled
transaction completed
```

La regla central será:

> **Un `TransactionEvent` representa un hecho o transición observable del ciclo de vida transaccional; nunca deberá utilizarse para inventar atomicidad sobre recursos externos, sustituir el estado autoritativo del `TransactionContext`, ni convertir listeners arbitrarios en participantes de la transacción física.**

Formalmente:

```text
Transaction State
        ↓
Authoritative Lifecycle Transition
        ↓
Transaction Event
        ↓
Observation / Coordination
```

Nunca:

```text
Transaction Event
        ↓
Invent Transaction State
```

---

# 2. Objetivos

El sistema deberá proporcionar:

1. modelo canónico de eventos transaccionales;
2. eventos tipados;
3. orden determinista;
4. dispatch síncrono controlado;
5. callbacks transaccionales;
6. `beforeCommit`;
7. `afterCommit`;
8. `afterRollback`;
9. `afterCompletion`;
10. integración con savepoints;
11. integración con nested transactions;
12. integración con retries;
13. representación explícita de `UNKNOWN`;
14. failure semantics de listeners;
15. reentrancy protection;
16. event buffering;
17. integración con ORM;
18. integración con Cache;
19. integración con Domain Events;
20. límites claros con Outbox;
21. Telemetry;
22. diagnostics;
23. extensibilidad;
24. persistent-runtime safety;
25. testing exhaustivo.

---

# 3. No objetivos

Este sistema no será:

- un Event Bus general;
- el Event System completo de VoltStack;
- un Domain Event Bus;
- un Message Broker;
- un Queue;
- un Outbox;
- un transaction log del DBMS;
- un Change Data Capture system;
- un distributed transaction coordinator;
- un replacement de `TransactionManager`;
- un replacement de `TransactionContext`;
- un replacement de ORM lifecycle events.

---

# 4. Distinciones fundamentales

```text
Transaction Event
≠
Domain Event
≠
ORM Lifecycle Event
≠
Query Event
≠
Connection Event
≠
Telemetry Event
≠
Outbox Message
≠
Queue Message
```

Estas categorías pueden relacionarse, pero no deberán fusionarse.

---

# 5. Transaction Event

Un `TransactionEvent` describe:

```text
algo relevante que ocurrió
o está a punto de ocurrir
dentro del lifecycle transaccional
```

Ejemplo:

```text
TransactionCommitted
```

significa:

```text
VoltStack posee evidencia suficiente
de que COMMIT fue confirmado
```

No significa:

```text
todos los listeners terminaron correctamente
```

---

# 6. Event taxonomy

Arquitectura propuesta:

```text
TransactionEvent
│
├── TransactionLifecycleEvent
│   ├── TransactionCreated
│   ├── TransactionStarting
│   ├── TransactionStarted
│   ├── TransactionBeforeCommit
│   ├── TransactionCommitted
│   ├── TransactionBeforeRollback
│   ├── TransactionRolledBack
│   ├── TransactionFailed
│   ├── TransactionOutcomeUnknown
│   └── TransactionCompleted
│
├── TransactionControlEvent
│   └── TransactionRollbackOnlyMarked
│
├── TransactionParticipationEvent
│   ├── TransactionParticipantJoined
│   ├── TransactionParticipantExited
│   ├── TransactionSuspended
│   └── TransactionResumed
│
├── TransactionSavepointEvent
│   ├── SavepointCreated
│   ├── SavepointReleased
│   └── SavepointRolledBack
│
└── TransactionRetryEvent
    ├── TransactionRetryScheduled
    ├── TransactionRetryStarted
    └── TransactionRetryExhausted
```

---

# 7. Event phases

Los eventos deberán clasificarse conceptualmente en:

```text
INTENT
TRANSITION
OUTCOME
COMPLETION
```

---

# 8. Intent events

Ejemplo:

```text
TransactionBeforeCommit
```

indica que VoltStack se prepara para intentar el commit.

No indica que el commit ocurrirá.

---

# 9. Transition events

Ejemplo:

```text
TransactionStarting
```

representa una transición en progreso.

---

# 10. Outcome events

Ejemplos:

```text
TransactionCommitted
TransactionRolledBack
TransactionOutcomeUnknown
```

representan outcomes conocidos o explícitamente desconocidos.

---

# 11. Completion events

```text
TransactionCompleted
```

representa que VoltStack terminó el procesamiento local del lifecycle.

Esto puede ocurrir después de:

```text
COMMITTED
ROLLED_BACK
FAILED
UNKNOWN
```

---

# 12. Event ≠ state

El evento:

```text
TransactionCommitted
```

no será el almacenamiento autoritativo de estado.

La autoridad continuará siendo:

```text
TransactionContext
+
TransactionStateMachine
+
TransactionManager
```

---

# 13. Event envelope

Modelo:

```php
final readonly class TransactionEventEnvelope
{
    public function __construct(
        public TransactionEventId $eventId,
        public TransactionOperationId $operationId,
        public TransactionId $transactionId,
        public TransactionAttempt $attempt,
        public TransactionEvent $event,
        public TransactionEventMetadata $metadata,
        public TransactionEventTimestamp $timestamp,
    ) {}
}
```

---

# 14. TransactionEventId

Cada evento tendrá identidad propia:

```text
TransactionEventId
≠
TransactionId
≠
TransactionOperationId
```

---

# 15. Retry identity

Ejemplo:

```text
Operation O
│
├── Transaction T1
│   ├── E1 Started
│   ├── E2 RolledBack
│   └── E3 Completed
│
└── Transaction T2
    ├── E4 Started
    ├── E5 Committed
    └── E6 Completed
```

---

# 16. Event sequence

Cada `TransactionContext` podrá mantener:

```text
TransactionEventSequence
```

monótona localmente:

```text
1
2
3
4
...
```

---

# 17. Sequence scope

La secuencia será:

```text
per transaction attempt
```

No necesariamente global.

---

# 18. No global sequence requirement

VoltStack no requerirá un contador global compartido entre workers.

Esto evitará:

```text
global contention
distributed coordination
persistent worker leakage
```

---

# 19. Base contract

```php
interface TransactionEvent
{
    public function type(): TransactionEventType;
}
```

---

# 20. Immutable events

Los eventos serán:

```text
immutable value objects
```

Una vez creados no podrán ser modificados por listeners.

---

# 21. Transaction event snapshot

Los eventos no deberán exponer directamente un `MutableTransactionContext`.

Preferir:

```php
final readonly class TransactionEventContext
{
    public function __construct(
        public TransactionContextSnapshot $transaction,
        public TransactionEventPhase $phase,
        public int $sequence,
    ) {}
}
```

---

# 22. Why snapshot

Evita que un listener pueda:

```text
change state
replace connection
modify nesting
clear savepoints
fake outcome
```

---

# 23. Mutable transaction control

Cuando una extensión necesite solicitar una acción legítima:

```text
mark rollback-only
register callback
```

deberá utilizar contratos explícitos.

No modificar el context directamente.

---

# 24. TransactionControl

Ejemplo:

```php
interface TransactionControl
{
    public function markRollbackOnly(
        RollbackOnlyReason $reason
    ): void;

    public function registerCallback(
        TransactionCallback $callback
    ): void;
}
```

---

# 25. Read-only event context

Regla:

```text
EventContext
=
ReadOnly
```

---

# 26. Dispatcher

Contrato conceptual:

```php
interface TransactionEventDispatcher
{
    public function dispatch(
        TransactionEvent $event,
        TransactionEventContext $context,
    ): TransactionEventDispatchResult;
}
```

---

# 27. Dispatcher responsibility

Deberá:

- resolver listeners;
- aplicar orden;
- ejecutar listeners;
- capturar resultados;
- aplicar failure policy;
- proteger reentrancy;
- producir diagnostics.

---

# 28. Dispatcher does not

No deberá:

- ejecutar COMMIT;
- ejecutar ROLLBACK;
- ejecutar SQL;
- cambiar isolation;
- adquirir connection;
- crear savepoints directamente;
- modificar UnitOfWork.

---

# 29. Listener contract

```php
interface TransactionEventListener
{
    public function handle(
        TransactionEvent $event,
        TransactionEventContext $context,
    ): void;
}
```

---

# 30. Subscriber contract

Para múltiples tipos:

```php
interface TransactionEventSubscriber
{
    public static function subscriptions(): array;
}
```

Ejemplo conceptual:

```php
public static function subscriptions(): array
{
    return [
        TransactionCommitted::class => 'onCommitted',
        TransactionRolledBack::class => 'onRolledBack',
    ];
}
```

---

# 31. Listener registration

El registro deberá ocurrir principalmente durante:

```text
bootstrap
```

y posteriormente quedar:

```text
compiled/frozen
```

---

# 32. No runtime listener mutation

En persistent runtimes deberá evitarse:

```php
$dispatcher->listeners[] = ...
```

durante requests.

---

# 33. Listener priority

Podrá existir:

```php
final readonly class TransactionListenerPriority
{
    public function __construct(
        public int $value
    ) {}
}
```

---

# 34. Deterministic ordering

Orden:

```text
priority
↓
registration sequence
```

con reglas deterministas.

---

# 35. Priority direction

Deberá documentarse una convención única.

Ejemplo:

```text
higher priority executes first
```

---

# 36. Equal priority

Si dos listeners tienen igual prioridad:

```text
stable registration order
```

---

# 37. Event ordering

Un commit exitoso deberá observar conceptualmente:

```text
TransactionCreated
TransactionStarting
TransactionStarted
...
TransactionBeforeCommit
TransactionCommitted
TransactionCompleted
```

---

# 38. Rollback ordering

```text
TransactionCreated
TransactionStarting
TransactionStarted
...
TransactionBeforeRollback
TransactionRolledBack
TransactionCompleted
```

---

# 39. Unknown commit ordering

```text
TransactionBeforeCommit
        ↓
COMMIT dispatched
        ↓
connection failure
        ↓
TransactionOutcomeUnknown
        ↓
TransactionCompleted
```

Nunca:

```text
TransactionCommitted
```

sin evidencia.

---

# 40. Start failure

Si `BEGIN` falla antes de confirmarse:

```text
TransactionCreated
TransactionStarting
TransactionFailed
TransactionCompleted
```

No:

```text
TransactionStarted
```

---

# 41. TransactionCreated

Representa:

```text
framework transaction context created
```

No:

```text
physical DB transaction active
```

---

# 42. TransactionStarting

Se emite antes de intentar iniciar la transacción física.

---

# 43. TransactionStarted

Solo después de que VoltStack tenga evidencia suficiente de:

```text
physical transaction active
```

---

# 44. TransactionBeforeCommit

Se emite antes de dispatch del commit físico.

Este evento tiene semántica especial.

---

# 45. BeforeCommit failure

Un listener `beforeCommit` podrá, según policy:

```text
prevent commit
mark rollback-only
cause rollback path
```

porque el commit aún no fue enviado.

---

# 46. TransactionCommitted

Solo después de:

```text
confirmed physical commit
```

---

# 47. Committed is irreversible locally

Una vez confirmado:

```text
TransactionCommitted
```

un listener fallido no puede transformar la DB en:

```text
ROLLED_BACK
```

---

# 48. Post-commit failure

Por tanto:

```text
DB Outcome = COMMITTED
Local Completion = COMPLETED_WITH_CALLBACK_FAILURE
```

es un estado válido.

---

# 49. TransactionBeforeRollback

Se emite antes de intentar rollback físico.

---

# 50. BeforeRollback failure

Un listener no deberá impedir el rollback necesario.

La policy recomendada será:

```text
record failure
continue rollback
```

---

# 51. Why

Cuando una transacción debe abortarse:

```text
listener failure
```

no puede convertirla en:

```text
commit
```

---

# 52. TransactionRolledBack

Solo después de rollback confirmado.

---

# 53. Rollback failure

Si no puede confirmarse:

```text
TransactionOutcomeUnknown
```

o failure classification apropiada.

No se fabricará `RolledBack`.

---

# 54. TransactionFailed

Representa fallo transaccional que no equivale necesariamente a outcome físico desconocido.

---

# 55. Failed vs Unknown

```text
FAILED
```

significa que VoltStack conoce suficiente información para clasificar el fallo.

```text
UNKNOWN
```

significa que no conoce suficientemente el outcome.

---

# 56. Unknown dominates convenience

Ningún listener podrá convertir:

```text
UNKNOWN
```

en:

```text
COMMITTED
ROLLED_BACK
SUCCESS
```

---

# 57. TransactionCompleted

Siempre que el lifecycle local pueda cerrarse, se emitirá un evento final:

```text
TransactionCompleted
```

con:

```text
final transaction state
database outcome
local completion status
callback status
cleanup status
```

---

# 58. Completion ≠ success

```text
TransactionCompleted
```

no significa:

```text
successful transaction
```

---

# 59. Completion result

Modelo:

```php
final readonly class TransactionCompletionResult
{
    public function __construct(
        public TransactionOutcome $outcome,
        public TransactionCompletionStatus $status,
        public ?ThrowableSummary $failure,
    ) {}
}
```

---

# 60. Transaction callbacks

Además de eventos generales, VoltStack soportará callbacks asociados a una transacción concreta:

```text
beforeCommit
afterCommit
afterRollback
afterCompletion
```

---

# 61. Listener ≠ callback

```text
TransactionEventListener
```

normalmente está registrado globalmente como infraestructura.

```text
TransactionCallback
```

normalmente pertenece a una transacción concreta.

---

# 62. Callback registration

Ejemplo:

```php
DB::transaction(function () {
    // ...

    DB::transactions()->afterCommit(function () {
        // ...
    });
});
```

---

# 63. Callback registry

Los callbacks vivirán en:

```text
TransactionContext
```

mediante el `TransactionCallbackRegistry` definido conceptualmente en documentos anteriores.

---

# 64. Callback types

```php
enum TransactionCallbackPhase
{
    case BEFORE_COMMIT;
    case AFTER_COMMIT;
    case AFTER_ROLLBACK;
    case AFTER_COMPLETION;
}
```

---

# 65. Callback identity

Cada callback podrá poseer:

```text
CallbackId
Priority
RegistrationSequence
Phase
FailurePolicy
```

---

# 66. Callback ordering

Orden determinista:

```text
phase
→ priority
→ registration sequence
```

---

# 67. beforeCommit callbacks

Podrán participar en preparación local previa al commit.

Ejemplos:

```text
flush deferred transactional state
validate transaction-local invariant
prepare cache invalidation metadata
```

---

# 68. afterCommit callbacks

Solo ejecutarán cuando:

```text
commit confirmed
```

Nunca ante:

```text
UNKNOWN
```

---

# 69. afterRollback callbacks

Solo ejecutarán cuando:

```text
rollback confirmed
```

---

# 70. afterCompletion callbacks

Ejecutarán para cualquier terminal outcome permitido:

```text
COMMITTED
ROLLED_BACK
FAILED
UNKNOWN
```

con información del resultado.

---

# 71. afterCompletion purpose

Adecuado para:

```text
release framework-local resources
clear local registries
record diagnostics
```

---

# 72. Callback failure policies

```php
enum TransactionCallbackFailurePolicy
{
    case FAIL_BEFORE_COMMIT;
    case RECORD_AND_CONTINUE;
    case AGGREGATE_AND_CONTINUE;
}
```

Las opciones válidas dependerán de la fase.

---

# 73. Invalid policy

No se permitirá:

```text
AFTER_COMMIT → rollback transaction
```

porque es físicamente imposible.

---

# 74. beforeCommit failure

Default:

```text
mark rollback-only
+
abort commit
+
rollback
```

---

# 75. afterCommit failure

Default:

```text
record
+
continue remaining callbacks according to policy
+
mark local completion failure
```

---

# 76. afterRollback failure

No cambiará:

```text
DatabaseOutcome = ROLLED_BACK
```

---

# 77. afterCompletion failure

No cambiará el DB outcome.

---

# 78. Callback aggregation

Múltiples fallos podrán representarse mediante:

```text
TransactionCallbackFailureCollection
```

con límites de memoria.

---

# 79. Original failure preservation

Si la transacción ya falló y además falla un callback:

```text
PrimaryFailure
+
SecondaryCallbackFailure
```

deberán conservarse separadamente.

---

# 80. Exception masking forbidden

Un listener secundario no deberá ocultar:

```text
deadlock
serialization failure
commit unknown
```

original.

---

# 81. Reentrancy

Un listener puede intentar:

```php
DB::transaction(...);
```

o incluso:

```php
$currentTransaction->commit();
```

durante dispatch.

Esto requiere reglas explícitas.

---

# 82. Reentrant completion

Prohibido:

```text
commit listener
→ commit same transaction again
```

---

# 83. Reentrant rollback

Prohibido:

```text
rollback listener
→ rollback same transaction recursively
```

---

# 84. Reentrancy guard

```php
final class TransactionEventReentrancyGuard
{
    // scoped dispatch stack
}
```

---

# 85. Dispatch stack

Conceptualmente:

```text
DispatchStack
[
    TransactionBeforeCommit,
    ...
]
```

permite detectar ciclos.

---

# 86. Event cycle

Ejemplo prohibido:

```text
Listener A
→ dispatch X
→ Listener B
→ dispatch X
→ ...
```

---

# 87. Nested event dispatch

No todo nested dispatch será ilegal.

Ejemplo válido:

```text
TransactionBeforeCommit
→ ORM flush preparation
→ ORM lifecycle event
```

si pertenece a otro event domain y no rompe invariantes.

---

# 88. Same-event recursion

Por default:

```text
same transaction
+
same event type
+
same lifecycle transition
```

no podrá reingresar.

---

# 89. Starting another transaction

Un listener podría iniciar una transacción independiente solo si:

```text
propagation rules
+
runtime
+
connection availability
+
reentrancy policy
```

lo permiten.

No será permitido implícitamente en fases críticas.

---

# 90. Critical phases

Se considerarán especialmente sensibles:

```text
STARTING
BEFORE_COMMIT
COMMITTING
BEFORE_ROLLBACK
ROLLING_BACK
COMPLETION
```

---

# 91. Event dispatch modes

Core deberá favorecer:

```text
SYNCHRONOUS
```

para eventos de lifecycle.

---

# 92. Why synchronous

El orden transaccional requiere saber:

```text
listener completed
listener failed
rollback-only requested
```

antes de continuar en determinadas fases.

---

# 93. Async transaction event

No deberá utilizarse para representar lifecycle autoritativo.

---

# 94. Async projection

Un evento transaccional podrá originar posteriormente:

```text
Outbox Message
Queue Message
Telemetry Export
```

pero esa proyección es otra capa.

---

# 95. Domain Events

Considérese:

```php
$order->confirm();
```

produce:

```text
OrderConfirmed
```

Ese evento es:

```text
Domain Event
```

No:

```text
Transaction Event
```

---

# 96. Domain event publication problem

Si se publica externamente antes del commit:

```text
publish OrderConfirmed
↓
transaction rollback
```

se genera inconsistencia.

---

# 97. Deferred domain event publication

VoltStack podrá soportar:

```text
Domain Event
↓
Transaction-aware buffer
↓
Commit confirmed
↓
publish
```

para eventos in-process apropiados.

---

# 98. Deferred publication limitation

Esto no crea atomicidad con:

```text
Kafka
RabbitMQ
HTTP API
email
external queue
```

---

# 99. Outbox boundary

Para publicación durable externa:

```text
Business Changes
+
Outbox Record
```

deberán persistirse dentro de la misma DB transaction.

Después:

```text
Outbox Processor
→ external broker
```

---

# 100. afterCommit ≠ outbox

Regla obligatoria:

```text
afterCommit callback
≠
Transactional Outbox
```

---

# 101. Why

Caso:

```text
DB COMMIT succeeds
↓
process crashes
↓
afterCommit callback never sends message
```

La DB está committed, pero el side effect se pierde.

---

# 102. Outbox atomicity

```text
business state
+
outbox row
```

pueden compartir la misma DB transaction.

Eso permite recuperación posterior.

---

# 103. Email example

No asumir:

```php
DB::afterCommit(fn () => Mail::send(...));
```

como durable exactly-once messaging.

Puede ser aceptable para ciertas tareas best-effort, pero no como garantía transaccional fuerte.

---

# 104. Exactly-once warning

VoltStack no prometerá:

```text
exactly-once external side effect
```

solo por usar transaction callbacks.

---

# 105. At-least-once patterns

Outbox + idempotent consumer podrá ofrecer semánticas más robustas.

Eso pertenece a sistemas posteriores.

---

# 106. ORM integration

ORM podrá registrar callbacks/event participants para:

```text
persistence consistency
snapshot reconciliation
post-commit state
rollback handling
EntityManager tainting
```

---

# 107. ORM lifecycle events remain separate

Ejemplos:

```text
prePersist
postPersist
preUpdate
postUpdate
postLoad
```

son ORM events.

No transaction events.

---

# 108. postPersist ≠ committed

Especialmente:

```text
postPersist
```

no deberá significar:

```text
database transaction committed
```

---

# 109. Flush events

```text
preFlush
postFlush
```

también son distintos de:

```text
beforeCommit
afterCommit
```

---

# 110. Flush ≠ commit

Se mantiene:

```text
flush()
≠
commit()
```

---

# 111. ORM after-commit semantics

Si se requiere una acción ORM después del commit:

```text
transaction callback
```

o bridge explícito será utilizado.

---

# 112. Rollback ORM state

Ante rollback:

```text
DB state
```

puede restaurarse mientras:

```text
PHP objects
```

siguen modificados.

---

# 113. Rollback event does not rewind objects

Regla:

```text
TransactionRolledBack
≠
ObjectGraphRewound
```

---

# 114. ORM rollback listener

Podrá:

```text
mark persistence context stale
clear UnitOfWork
taint EntityManager
schedule reconciliation
```

según policy.

---

# 115. Cache integration

Cache invalidation presenta un problema similar.

Incorrecto:

```text
invalidate cache
↓
DB rollback
```

---

# 116. Deferred invalidation

Podrá utilizarse:

```text
transaction-local invalidation set
↓
commit
↓
invalidate
```

---

# 117. Cache invalidation after commit

Adecuado cuando:

```text
cache can temporarily tolerate stale entry
```

y la failure policy está definida.

---

# 118. Durable cache consistency

Si la invalidación debe ser recuperable después de crash:

```text
afterCommit callback alone
```

puede ser insuficiente.

---

# 119. Cache transaction resource

El documento 166 ya permite recursos como:

```text
cache.deferred_invalidation
```

dentro de:

```text
TransactionResourceRegistry
```

---

# 120. Event bridge

Podrá existir:

```text
TransactionCommitted
→ CacheTransactionBridge
→ consume deferred invalidations
```

---

# 121. Savepoint events

Eventos:

```text
SavepointCreated
SavepointRolledBack
SavepointReleased
```

representarán operaciones físicas/lógicas confirmadas según el Savepoint System.

---

# 122. Savepoint event ≠ transaction completion

```text
SavepointRolledBack
```

no significa:

```text
TransactionRolledBack
```

---

# 123. Savepoint callback semantics

Por default, callbacks:

```text
afterCommit
afterRollback
```

pertenecen a la transacción física completa.

No a cada savepoint.

---

# 124. Nested callback scopes

Sin embargo, nested transaction scopes pueden registrar callbacks.

Debe definirse cómo sobreviven al rollback de su savepoint.

---

# 125. Nested callback ownership

Cada callback podrá asociarse con:

```text
TransactionParticipantId
```

y opcionalmente:

```text
SavepointId
```

---

# 126. Savepoint rollback

Si un nested scope asociado a savepoint se revierte:

```text
callbacks created exclusively inside that reverted scope
```

no deberán ejecutarse posteriormente como si su trabajo hubiese sobrevivido.

---

# 127. Callback pruning

El `NestedTransactionCoordinator` y `SavepointManager` deberán poder solicitar:

```text
discard callbacks registered after savepoint boundary
```

cuando corresponda.

---

# 128. Resource pruning

La misma regla podrá aplicar a:

```text
deferred domain events
cache invalidations
outbox preparation metadata
```

si su estado lógico fue revertido por savepoint.

---

# 129. Savepoint rollback ≠ PHP state rewind

Aunque callbacks/resources se puedan podar:

```text
arbitrary application objects
```

no se restauran automáticamente.

---

# 130. Joined nested transaction

En estrategia:

```text
JOIN
```

el inner scope no tiene commit físico propio.

---

# 131. Inner success

Por tanto:

```text
inner scope returns successfully
```

no genera:

```text
TransactionCommitted
```

---

# 132. Physical commit authority

Solo el owner de la transacción física puede producir:

```text
TransactionCommitted
```

---

# 133. Nested logical events

Si se necesitan eventos de participación:

```text
TransactionParticipantJoined
TransactionParticipantExited
```

deberán utilizarse en lugar de falsos commit/rollback events.

---

# 134. Nested rollback-only

Un inner scope puede:

```text
mark rollback-only
```

generando:

```text
TransactionRollbackOnlyMarked
```

---

# 135. Rollback-only event

Debe contener:

```text
reason category
participant
depth
timestamp
```

sin exponer datos sensibles.

---

# 136. Rollback-only is monotonic

Una vez marcado:

```text
rollbackOnly = true
```

no podrá desmarcarse.

---

# 137. Event does not unmark

Ningún listener podrá revertirlo.

---

# 138. Retry integration

El documento 170 define retries a nivel de operación.

Los transaction events deberán distinguir:

```text
attempt lifecycle
```

de:

```text
operation retry lifecycle
```

---

# 139. TransactionRetryScheduled

Representa:

```text
Retry System decided another attempt may run
```

No significa que ya inició.

---

# 140. TransactionRetryStarted

Representa inicio del nuevo attempt.

El nuevo attempt tendrá:

```text
new TransactionId
```

---

# 141. TransactionRetryExhausted

Representa:

```text
no more retry attempts permitted
```

por policy/budget.

---

# 142. Retry event scope

Eventos de retry podrán pertenecer conceptualmente al:

```text
TransactionOperation
```

más que a una única transaction.

---

# 143. Operation event context

Podrá existir:

```php
final readonly class TransactionOperationEventContext
{
    public function __construct(
        public TransactionOperationId $operationId,
        public int $attempt,
        public int $maxAttempts,
    ) {}
}
```

---

# 144. Attempt completion

Cada attempt seguirá emitiendo su lifecycle normal:

```text
Started
RolledBack
Completed
```

antes del siguiente retry.

---

# 145. No event replay

Cuando una transaction se reintenta:

```text
events from attempt 1
```

no deberán volver a emitirse como si pertenecieran al attempt 2.

---

# 146. Listener idempotency

Infraestructura que observe retries deberá considerar que múltiples attempts son normales.

---

# 147. External side effects in listeners

Un listener de:

```text
TransactionStarted
TransactionBeforeCommit
```

no deberá realizar side effects externos irreversibles por default.

---

# 148. BeforeCommit side effect risk

Porque:

```text
beforeCommit
→ send external request
→ commit fails
```

produce inconsistencia.

---

# 149. Committed listener side effect risk

Incluso:

```text
TransactionCommitted
→ send external request
```

puede perderse si el proceso muere antes del listener.

---

# 150. Therefore

```text
Transaction Event
```

es una primitive de coordinación local.

No una garantía durable cross-system.

---

# 151. Event buffering

VoltStack podrá mantener un:

```text
TransactionEventBuffer
```

para eventos que no deban publicarse inmediatamente.

---

# 152. Buffer use cases

Ejemplos:

```text
deferred domain events
deferred cache invalidation intents
diagnostic aggregation
```

---

# 153. Buffer ≠ durable queue

El buffer será:

```text
memory scoped
```

salvo que una extensión lo persista explícitamente.

---

# 154. Crash semantics

Si el worker muere:

```text
memory event buffer
```

se pierde.

Esto deberá estar documentado.

---

# 155. Buffer ownership

El buffer deberá pertenecer a:

```text
TransactionContext
```

o recurso transaccional asociado.

Nunca global.

---

# 156. Buffer entries

Cada entrada deberá registrar:

```text
registration sequence
participant
savepoint boundary
delivery phase
payload/reference
```

---

# 157. Memory safety

Buffers deberán tener límites:

```text
max events
max approximate bytes
max callbacks
max diagnostics
```

---

# 158. Buffer overflow

No se resolverá con crecimiento ilimitado.

Resultado:

```text
TransactionEventResourceLimitException
```

o policy equivalente.

---

# 159. Payload rules

Evitar almacenar:

```text
huge query results
full object graphs
credentials
raw sensitive parameters
```

---

# 160. Event payload minimization

Preferir:

```text
IDs
typed metadata
bounded snapshots
semantic references
```

cuando sea suficiente.

---

# 161. Event listener capabilities

Los listeners podrán declararse:

```text
OBSERVATIONAL
COORDINATING
CRITICAL
```

---

# 162. Observational listener

Ejemplos:

```text
telemetry
logging
debugging
```

No deberá modificar transaction control.

---

# 163. Coordinating listener

Puede:

```text
register callback
consume transaction resource
prepare deferred work
```

dentro de contratos permitidos.

---

# 164. Critical listener

Solo casos internos cuidadosamente controlados podrán:

```text
fail beforeCommit
mark rollback-only
```

---

# 165. Capability declaration

```php
enum TransactionListenerCapability
{
    case OBSERVE;
    case COORDINATE;
    case CONTROL_BEFORE_COMMIT;
}
```

---

# 166. Least privilege

Un telemetry listener no deberá recibir:

```text
TransactionControl
```

si solo necesita observar.

---

# 167. Internal vs extension listeners

Podrán distinguirse:

```text
CORE
FRAMEWORK
PACKAGE
APPLICATION
```

---

# 168. Ordering domains

No depender únicamente de números mágicos como:

```text
priority = 999999
```

Podrán existir fases:

```text
CORE_PREPARE
FRAMEWORK
EXTENSION
APPLICATION
TELEMETRY
```

con prioridad local.

---

# 169. Stable ordering

Modelo:

```text
Phase
→ Priority
→ RegistrationSequence
```

---

# 170. Event dispatch result

```php
final readonly class TransactionEventDispatchResult
{
    public function __construct(
        public int $listenersExecuted,
        public int $listenersFailed,
        public bool $rollbackOnlyRequested,
        public TransactionEventDispatchStatus $status,
    ) {}
}
```

---

# 171. Dispatch status

```php
enum TransactionEventDispatchStatus
{
    case SUCCESS;
    case SUCCESS_WITH_FAILURES;
    case ABORTED;
    case FAILED;
}
```

---

# 172. Dispatch result ≠ transaction outcome

Ejemplo:

```text
TransactionCommitted
listener fails
```

produce:

```text
DispatchStatus = SUCCESS_WITH_FAILURES
TransactionOutcome = COMMITTED
```

---

# 173. Exception hierarchy

Propuesta:

```text
DatabaseTransactionEventException
├── TransactionEventDispatchException
├── TransactionEventListenerException
├── TransactionCallbackException
├── TransactionCallbackRegistrationException
├── TransactionCallbackExecutionException
├── TransactionEventReentrancyException
├── TransactionEventCycleException
├── TransactionEventResourceLimitException
├── TransactionEventInvalidPhaseException
├── TransactionEventInvalidStateException
├── TransactionEventMutationException
└── TransactionEventInvariantViolationException
```

---

# 174. Event registration errors

Deberán detectarse preferentemente durante bootstrap:

```text
invalid listener
unknown event
invalid capability
invalid priority
duplicate registration
incompatible phase
```

---

# 175. Duplicate listeners

Policy explícita:

```text
same listener identity
+
same event
+
same registration scope
```

no deberá registrarse accidentalmente múltiples veces.

---

# 176. Persistent runtime problem

En FrankenPHP:

```text
worker boot
→ register listener
request 1
→ register listener again
request 2
→ register again
```

produciría multiplicación de handlers.

---

# 177. Solution

```text
compile listeners once at boot
freeze registry
```

---

# 178. Runtime callback registry

A diferencia de listeners globales:

```text
transaction callbacks
```

sí serán runtime-scoped.

---

# 179. Shared vs scoped state

Compartible:

```text
CompiledTransactionEventRegistry
ListenerDefinitions
ImmutableFailurePolicies
EventMetadata
```

Scoped:

```text
TransactionCallbackRegistry
EventDispatchStack
EventBuffer
SequenceCounter
Failures
DeferredResources
```

---

# 180. Worker cleanup

Al finalizar scope:

```text
callback registry empty/disposed
event buffer empty/disposed
dispatch stack empty
transaction context unbound
```

---

# 181. Leak detection

Si queda:

```text
TransactionCallback
EventBufferEntry
DispatchFrame
```

después de finalizar el scope:

```text
runtime leak diagnostic
```

---

# 182. Never carry callbacks

Callbacks de request A jamás podrán ejecutarse en request B.

---

# 183. Coroutine isolation

Para OpenSwoole:

```text
Coroutine A callbacks
∩
Coroutine B callbacks
=
∅
```

---

# 184. Fiber isolation

Misma regla para fibers concurrentes.

---

# 185. Telemetry integration

Transaction Event System será una fuente natural para Telemetry.

Ejemplos:

```text
transaction.started
transaction.committed
transaction.rolled_back
transaction.unknown
transaction.callback.failed
```

---

# 186. Telemetry bridge

Preferir:

```text
Transaction Event
        ↓
TransactionTelemetryBridge
        ↓
Telemetry System
```

en lugar de introducir Telemetry directamente dentro de cada transition.

---

# 187. Telemetry listener capability

Será:

```text
OBSERVATIONAL
```

---

# 188. Telemetry failure

Por default:

```text
telemetry exporter failure
```

no deberá provocar rollback de una business transaction.

---

# 189. Metrics

Ejemplos:

```text
db.transactions.started
db.transactions.committed
db.transactions.rolled_back
db.transactions.unknown
db.transactions.callback_failures
db.transactions.rollback_only
db.transactions.savepoints
db.transactions.retries
```

---

# 190. Metrics cardinality

Labels apropiados:

```text
platform
outcome
isolation
ownership
retry_attempt_bucket
```

---

# 191. Forbidden labels

No usar:

```text
TransactionId
UserId
TenantId
raw SQL
bound values
credentials
```

como metric labels.

---

# 192. Tracing

Un transaction span podrá consumir events para marcar:

```text
started
commit requested
commit confirmed
rollback
unknown
retry
```

---

# 193. Transaction IDs in traces

Podrán existir como:

```text
bounded diagnostic attributes
```

según Telemetry security policy, pero no como metric cardinality labels.

---

# 194. Logging

Structured log example:

```text
event = db.transaction.completed
outcome = committed
attempt = 2
duration_ms = 47
callbacks_failed = 0
```

---

# 195. Sensitive data

Transaction events no deberán transportar por default:

```text
SQL parameters
credentials
full entity snapshots
personal information
```

---

# 196. Diagnostics API

Conceptualmente:

```php
DB::transactions()
    ->events()
    ->inspect();
```

---

# 197. Diagnostic registry

Podrá mostrar:

```text
TRANSACTION EVENT SYSTEM

Listeners:
    14

Subscribers:
    4

Compiled:
    yes

Frozen:
    yes

Critical listeners:
    2

Observational listeners:
    8

Max callbacks/transaction:
    256

Max buffered events:
    1024
```

---

# 198. Transaction timeline

Developer tooling podrá producir:

```text
TRANSACTION TIMELINE

00.000 ms  TransactionCreated
00.120 ms  TransactionStarting
00.900 ms  TransactionStarted
03.100 ms  SavepointCreated
04.400 ms  SavepointReleased
08.200 ms  TransactionBeforeCommit
09.600 ms  TransactionCommitted
09.700 ms  AfterCommitCallbacksStarted
10.100 ms  TransactionCompleted
```

---

# 199. Timeline is diagnostic

No deberá utilizarse como fuente autoritativa para reconstruir DB state.

---

# 200. Unknown timeline

Ejemplo:

```text
00.000  TransactionStarted
12.300  TransactionBeforeCommit
12.700  CommitDispatched
13.800  ConnectionLost
13.900  TransactionOutcomeUnknown
14.100  TransactionCompleted
```

---

# 201. Query events

`QueryExecuted` no deberá emitirse desde Transaction Event System.

Pertenece al sistema de query events/telemetry.

---

# 202. Connection events

Igualmente:

```text
ConnectionOpened
ConnectionClosed
ConnectionLost
```

pertenecen al Connection Event System.

Una pérdida de conexión puede causar un transaction event, pero ambos eventos son diferentes.

---

# 203. Causality metadata

Un transaction event podrá contener:

```text
CauseCategory
```

por ejemplo:

```text
CONNECTION_FAILURE
DEADLOCK
SERIALIZATION_FAILURE
USER_EXCEPTION
ROLLBACK_ONLY
CANCELLATION
TIMEOUT
```

---

# 204. Cause metadata ≠ Throwable

No será obligatorio transportar la excepción completa.

Preferir:

```text
sanitized ThrowableSummary
```

---

# 205. Causation

Podrá existir:

```text
CausationId
CorrelationId
```

para observabilidad.

---

# 206. Correlation scope

`CorrelationId` podrá unir:

```text
operation
transaction attempts
events
telemetry
```

sin convertirlo en estado global mutable.

---

# 207. Retry correlation

```text
OperationId = O1

Attempt 1:
    TransactionId = T1

Attempt 2:
    TransactionId = T2
```

ambos correlacionados por `O1`.

---

# 208. Event extension model

Paquetes podrán registrar nuevos:

```text
listeners
subscribers
bridges
diagnostic observers
```

---

# 209. Custom transaction events

Core deberá ser conservador permitiendo nuevos event types.

Un package no deberá fingir que un custom event es una transición core.

---

# 210. Namespaced custom event types

Ejemplo:

```text
package.audit.transaction_observed
```

separado de:

```text
core.transaction.committed
```

---

# 211. Core event names protected

No se permitirá sobrescribir:

```text
TransactionCommitted
TransactionRolledBack
TransactionOutcomeUnknown
```

---

# 212. Event registry

```php
interface TransactionEventRegistry
{
    public function listenersFor(
        TransactionEventType $type
    ): CompiledTransactionListenerList;
}
```

---

# 213. Registry lifecycle

```text
COLLECTING
    ↓
NORMALIZING
    ↓
VALIDATING
    ↓
COMPILING
    ↓
FROZEN
```

---

# 214. Frozen registry

Después de boot:

```text
listener topology immutable
```

---

# 215. Dynamic application callbacks

No pasan por registry global.

Van al:

```text
TransactionCallbackRegistry
```

del context actual.

---

# 216. No current transaction

Llamar:

```php
DB::transactions()->afterCommit(...)
```

sin una transacción activa deberá seguir policy explícita.

---

# 217. Recommended default

Lanzar:

```text
NoActiveTransactionException
```

en lugar de ejecutar inmediatamente.

---

# 218. Why

Ejecutar inmediatamente haría que:

```text
afterCommit
```

cambie silenciosamente de significado dependiendo del contexto.

---

# 219. Explicit convenience API

Si se desea:

```text
run now if no transaction
```

deberá ser otro API explícito.

Ejemplo conceptual:

```php
DB::transactions()->afterCommitOrNow(...);
```

No será semántica oculta.

---

# 220. Callback registration during completion

Registrar:

```text
afterCommit
```

mientras ya se están ejecutando `afterCommit` callbacks requiere policy.

---

# 221. Recommended rule

Por default:

```text
callback phase already entered
→ registration rejected
```

---

# 222. Why

Evita:

```text
unbounded callback chains
order ambiguity
reentrancy
```

---

# 223. afterCompletion registration

Igualmente deberá cerrarse cuando la fase haya comenzado.

---

# 224. Callback registry lifecycle

```text
OPEN
    ↓
BEFORE_COMMIT_CLOSED
    ↓
COMPLETION_PHASE
    ↓
CLOSED
    ↓
DISPOSED
```

---

# 225. Callback cancellation

Un callback registrado podrá opcionalmente devolver un handle:

```php
interface TransactionCallbackHandle
{
    public function cancel(): void;
}
```

solo mientras la fase permanezca abierta.

---

# 226. Cancellation after phase start

Será rechazada o no-op explícito según contrato.

Nunca ambiguo.

---

# 227. Callback deduplication

Para infraestructura podrá existir:

```text
CallbackKey
```

permitiendo registrar:

```text
invalidate:users:42
```

una sola vez.

---

# 228. Deduplication semantics

No deberá aplicarse automáticamente a closures arbitrarias.

---

# 229. Transaction-local coalescing

Especialmente útil para:

```text
cache invalidation
search index refresh
derived projections
```

---

# 230. Coalescing ≠ durable delivery

Sigue siendo transaction-local memory.

---

# 231. Event performance

El hot path deberá minimizar:

- reflection;
- dynamic discovery;
- allocations innecesarias;
- listener lookup repetitivo;
- unbounded payloads.

---

# 232. Compiled dispatch tables

Preferir:

```text
EventType
→ precompiled listener array
```

---

# 233. Empty listener optimization

Si un evento no tiene listeners/callbacks/telemetry requirements:

```text
fast no-op path
```

---

# 234. Event object allocation

Podrán existir optimizaciones internas, pero nunca sacrificando:

```text
correct ordering
outcome semantics
context isolation
```

---

# 235. Listener timeout

Listeners críticos no deberán ejecutarse indefinidamente.

El runtime podrá aplicar:

```text
deadline awareness
cancellation awareness
resource budgets
```

cuando sea viable.

---

# 236. Listener timeout before commit

Puede producir:

```text
rollback-only
```

según policy.

---

# 237. Listener timeout after commit

No puede deshacer el commit.

Se registra como:

```text
post-commit completion failure
```

---

# 238. Transaction deadline

Event dispatch deberá respetar el:

```text
TransactionContext deadline
```

cuando la fase aún pueda afectar la transacción.

---

# 239. Post-commit deadline

Podrá utilizar un budget separado para cleanup/callbacks.

---

# 240. Event resource budget

```php
final readonly class TransactionEventBudget
{
    public function __construct(
        public int $maxCallbacks,
        public int $maxBufferedEvents,
        public int $maxFailures,
        public int $maxDispatchDepth,
    ) {}
}
```

---

# 241. Max dispatch depth

Evitará recursión infinita.

---

# 242. Failure collection bound

Después del límite:

```text
failure count continues
details truncated
```

en lugar de consumir memoria ilimitada.

---

# 243. Testing architecture

Suite propuesta:

```text
TransactionEventLifecycleTests
TransactionEventOrderingTests
TransactionEventListenerTests
TransactionEventSubscriberTests
TransactionEventCallbackTests
TransactionEventFailureTests
TransactionEventUnknownOutcomeTests
TransactionEventSavepointTests
TransactionEventNestedTransactionTests
TransactionEventRetryTests
TransactionEventReentrancyTests
TransactionEventBufferTests
TransactionEventOutboxBoundaryTests
TransactionEventOrmIntegrationTests
TransactionEventCacheIntegrationTests
TransactionEventTelemetryTests
TransactionEventPersistentRuntimeTests
TransactionEventCoroutineIsolationTests
TransactionEventResourceLimitTests
```

---

# 244. Commit lifecycle test

Verificar:

```text
Created
Starting
Started
BeforeCommit
Committed
Completed
```

en orden.

---

# 245. Rollback lifecycle test

Verificar:

```text
Created
Starting
Started
BeforeRollback
RolledBack
Completed
```

---

# 246. Begin failure test

Verificar ausencia de:

```text
TransactionStarted
```

si BEGIN nunca se confirmó.

---

# 247. Commit unknown test

Simular:

```text
commit sent
connection lost
```

Verificar:

```text
OutcomeUnknown
Completed
```

y ausencia de:

```text
Committed
RolledBack
afterCommit
afterRollback
```

---

# 248. beforeCommit failure test

Listener falla.

Esperado:

```text
commit not sent
rollback attempted
```

---

# 249. afterCommit failure test

DB commit confirmado.

Callback falla.

Esperado:

```text
DatabaseOutcome = COMMITTED
CompletionStatus = callback failure
```

---

# 250. afterRollback failure test

Rollback confirmado.

Callback falla.

Esperado:

```text
DatabaseOutcome = ROLLED_BACK
```

sin alteración.

---

# 251. Exception preservation test

Fallo principal:

```text
deadlock
```

callback secundario falla.

Verificar que deadlock permanezca primary cause.

---

# 252. Savepoint test

```text
create S1
register callback C1
create S2
register callback C2
rollback S2
```

Verificar policy de pruning de `C2`.

---

# 253. Joined nested test

Inner scope success no deberá emitir:

```text
TransactionCommitted
```

---

# 254. Retry test

```text
Attempt 1 → rollback
RetryScheduled
Attempt 2 → commit
```

Verificar:

```text
same OperationId
different TransactionIds
correct event ordering
```

---

# 255. Reentrancy test

Listener intenta commit del mismo transaction.

Esperado:

```text
TransactionEventReentrancyException
```

o invariant exception correspondiente.

---

# 256. Runtime isolation test

Request A registra callback.

Finaliza.

Request B inicia transaction.

Callback A jamás deberá aparecer.

---

# 257. Coroutine test

Dos transactions concurrentes deberán mantener:

```text
sequences
callbacks
buffers
dispatch stacks
```

completamente aislados.

---

# 258. Performance test

Medir:

```text
dispatch with zero listeners
dispatch with one listener
dispatch with N listeners
callback registration
callback execution
buffering
```

---

# 259. Proposed directory structure

```text
src/Quantum/Database/Transaction/Event/
│
├── TransactionEvent.php
├── TransactionEventType.php
├── TransactionEventPhase.php
├── TransactionEventId.php
├── TransactionEventEnvelope.php
├── TransactionEventContext.php
├── TransactionEventSequence.php
│
├── Lifecycle/
│   ├── TransactionCreated.php
│   ├── TransactionStarting.php
│   ├── TransactionStarted.php
│   ├── TransactionBeforeCommit.php
│   ├── TransactionCommitted.php
│   ├── TransactionBeforeRollback.php
│   ├── TransactionRolledBack.php
│   ├── TransactionFailed.php
│   ├── TransactionOutcomeUnknown.php
│   └── TransactionCompleted.php
│
├── Control/
│   ├── TransactionRollbackOnlyMarked.php
│   └── TransactionControl.php
│
├── Participation/
│   ├── TransactionParticipantJoined.php
│   ├── TransactionParticipantExited.php
│   ├── TransactionSuspended.php
│   └── TransactionResumed.php
│
├── Savepoint/
│   ├── SavepointCreated.php
│   ├── SavepointReleased.php
│   └── SavepointRolledBack.php
│
├── Retry/
│   ├── TransactionRetryScheduled.php
│   ├── TransactionRetryStarted.php
│   ├── TransactionRetryExhausted.php
│   └── TransactionOperationEventContext.php
│
├── Dispatch/
│   ├── TransactionEventDispatcher.php
│   ├── DefaultTransactionEventDispatcher.php
│   ├── TransactionEventDispatchResult.php
│   ├── TransactionEventDispatchStatus.php
│   ├── TransactionEventReentrancyGuard.php
│   └── TransactionEventDispatchStack.php
│
├── Listener/
│   ├── TransactionEventListener.php
│   ├── TransactionEventSubscriber.php
│   ├── TransactionListenerDefinition.php
│   ├── TransactionListenerPriority.php
│   ├── TransactionListenerCapability.php
│   ├── TransactionListenerPhase.php
│   ├── TransactionEventRegistry.php
│   └── CompiledTransactionEventRegistry.php
│
├── Callback/
│   ├── TransactionCallback.php
│   ├── TransactionCallbackId.php
│   ├── TransactionCallbackKey.php
│   ├── TransactionCallbackPhase.php
│   ├── TransactionCallbackHandle.php
│   ├── TransactionCallbackRegistry.php
│   ├── TransactionCallbackFailurePolicy.php
│   └── TransactionCallbackFailureCollection.php
│
├── Buffer/
│   ├── TransactionEventBuffer.php
│   ├── TransactionEventBufferEntry.php
│   └── TransactionEventBudget.php
│
├── Bridge/
│   ├── TransactionTelemetryBridge.php
│   ├── TransactionOrmBridge.php
│   ├── TransactionCacheBridge.php
│   └── TransactionDomainEventBridge.php
│
├── Diagnostics/
│   ├── TransactionEventInspector.php
│   ├── TransactionEventTimeline.php
│   └── TransactionEventDiagnosticReport.php
│
└── Exception/
    ├── DatabaseTransactionEventException.php
    ├── TransactionEventDispatchException.php
    ├── TransactionEventListenerException.php
    ├── TransactionCallbackException.php
    ├── TransactionCallbackRegistrationException.php
    ├── TransactionCallbackExecutionException.php
    ├── TransactionEventReentrancyException.php
    ├── TransactionEventCycleException.php
    ├── TransactionEventResourceLimitException.php
    ├── TransactionEventInvalidPhaseException.php
    └── TransactionEventInvariantViolationException.php
```

---

# 260. Architectural invariants

## DB-TXE-001
Transaction events serán tipados.

## DB-TXE-002
Transaction events serán inmutables.

## DB-TXE-003
Transaction events no serán la autoridad del transaction state.

## DB-TXE-004
TransactionContext seguirá siendo la fuente autoritativa del estado mutable.

## DB-TXE-005
TransactionManager seguirá controlando lifecycle físico.

## DB-TXE-006
Dispatcher no ejecutará COMMIT.

## DB-TXE-007
Dispatcher no ejecutará ROLLBACK.

## DB-TXE-008
Dispatcher no ejecutará SQL.

## DB-TXE-009
EventContext será read-only.

## DB-TXE-010
Listeners no modificarán MutableTransactionContext directamente.

## DB-TXE-011
TransactionEventId será distinto de TransactionId.

## DB-TXE-012
TransactionId será distinto de TransactionOperationId.

## DB-TXE-013
Retry attempts tendrán TransactionId diferente.

## DB-TXE-014
Retry attempts podrán compartir OperationId.

## DB-TXE-015
Event sequence será local al transaction attempt.

## DB-TXE-016
No se requerirá secuencia global.

## DB-TXE-017
TransactionCreated no significará physical BEGIN confirmado.

## DB-TXE-018
TransactionStarted requerirá BEGIN confirmado.

## DB-TXE-019
TransactionBeforeCommit no significará commit exitoso.

## DB-TXE-020
TransactionCommitted requerirá commit confirmado.

## DB-TXE-021
TransactionRolledBack requerirá rollback confirmado.

## DB-TXE-022
UNKNOWN nunca emitirá Committed sin evidencia.

## DB-TXE-023
UNKNOWN nunca emitirá RolledBack sin evidencia.

## DB-TXE-024
TransactionCompleted no significará success.

## DB-TXE-025
DB outcome será distinto de local completion status.

## DB-TXE-026
afterCommit solo ejecutará después de commit confirmado.

## DB-TXE-027
afterRollback solo ejecutará después de rollback confirmado.

## DB-TXE-028
afterCompletion podrá observar cualquier terminal outcome.

## DB-TXE-029
beforeCommit podrá impedir commit según policy.

## DB-TXE-030
beforeRollback listener failure no deberá impedir rollback requerido.

## DB-TXE-031
afterCommit failure no revertirá DB commit.

## DB-TXE-032
afterRollback failure no revertirá DB rollback.

## DB-TXE-033
afterCompletion failure no cambiará DB outcome.

## DB-TXE-034
Original transaction failure no será ocultado por callback failure.

## DB-TXE-035
Callback failures secundarios serán preservados separadamente.

## DB-TXE-036
Listener ordering será determinista.

## DB-TXE-037
Callback ordering será determinista.

## DB-TXE-038
Equal priority mantendrá stable registration order.

## DB-TXE-039
Global listener registry será compilable.

## DB-TXE-040
Global listener registry será frozen después de boot.

## DB-TXE-041
Transaction callbacks serán runtime-scoped.

## DB-TXE-042
Transaction callbacks no se almacenarán globalmente.

## DB-TXE-043
Reentrant commit de la misma transaction estará prohibido.

## DB-TXE-044
Reentrant rollback de la misma transaction estará prohibido.

## DB-TXE-045
Event dispatch cycles serán detectados.

## DB-TXE-046
Dispatch depth será bounded.

## DB-TXE-047
Lifecycle events críticos serán síncronos.

## DB-TXE-048
Async event delivery no será fuente autoritativa del lifecycle.

## DB-TXE-049
Transaction Event será distinto de Domain Event.

## DB-TXE-050
Transaction Event será distinto de ORM Lifecycle Event.

## DB-TXE-051
Transaction Event será distinto de Query Event.

## DB-TXE-052
Transaction Event será distinto de Connection Event.

## DB-TXE-053
Transaction Event será distinto de Outbox Message.

## DB-TXE-054
Transaction Event será distinto de Queue Message.

## DB-TXE-055
Deferred Domain Event publication no creará atomicidad externa.

## DB-TXE-056
afterCommit no será equivalente a Transactional Outbox.

## DB-TXE-057
External side effects no serán exactly-once por usar callbacks.

## DB-TXE-058
Durable external publication requerirá mecanismo durable separado.

## DB-TXE-059
Outbox record podrá participar en la DB transaction.

## DB-TXE-060
Outbox processor permanecerá fuera de la transaction original.

## DB-TXE-061
postPersist no significará committed.

## DB-TXE-062
postFlush no significará committed.

## DB-TXE-063
flush seguirá siendo distinto de commit.

## DB-TXE-064
Rollback event no rebobinará object graph.

## DB-TXE-065
ORM podrá marcar context stale/tainted tras rollback.

## DB-TXE-066
Cache invalidation previa a commit no será default.

## DB-TXE-067
Cache invalidation podrá diferirse hasta commit.

## DB-TXE-068
Deferred cache invalidation en memoria no será durable.

## DB-TXE-069
SavepointCreated no significará nueva physical transaction.

## DB-TXE-070
SavepointRolledBack no significará TransactionRolledBack.

## DB-TXE-071
SavepointReleased no significará TransactionCommitted.

## DB-TXE-072
Callbacks podrán asociarse a participant/savepoint boundaries.

## DB-TXE-073
Callbacks pertenecientes a trabajo revertido podrán ser podados.

## DB-TXE-074
Deferred resources pertenecientes a trabajo revertido podrán ser podados.

## DB-TXE-075
Savepoint rollback no restaurará arbitrary PHP state.

## DB-TXE-076
Joined nested scope success no emitirá TransactionCommitted.

## DB-TXE-077
Solo physical transaction owner podrá completar physical transaction.

## DB-TXE-078
Nested participation utilizará participation events cuando sea necesario.

## DB-TXE-079
Rollback-only marking será observable.

## DB-TXE-080
Rollback-only será monotonic.

## DB-TXE-081
Listener no podrá desmarcar rollback-only.

## DB-TXE-082
RetryScheduled no significará retry iniciado.

## DB-TXE-083
RetryStarted utilizará nuevo attempt.

## DB-TXE-084
Retry events preservarán OperationId.

## DB-TXE-085
Eventos del attempt anterior no serán replayed como nuevos.

## DB-TXE-086
Listeners deberán tolerar múltiples attempts.

## DB-TXE-087
BeforeCommit external side effects serán considerados inseguros por default.

## DB-TXE-088
Committed listeners tampoco proporcionarán durable delivery por sí solos.

## DB-TXE-089
Event buffer será scoped.

## DB-TXE-090
Event buffer no será durable queue.

## DB-TXE-091
Event buffer tendrá límites.

## DB-TXE-092
Callback registry tendrá límites.

## DB-TXE-093
Failure collection tendrá límites.

## DB-TXE-094
Payloads deberán ser bounded.

## DB-TXE-095
Credentials no se almacenarán en event payloads.

## DB-TXE-096
Raw sensitive parameters no se almacenarán por default.

## DB-TXE-097
Full entity graphs no serán event payload default.

## DB-TXE-098
Observational listeners no recibirán control privileges innecesarios.

## DB-TXE-099
Telemetry listeners serán observational.

## DB-TXE-100
Telemetry failure no hará rollback por default.

## DB-TXE-101
Critical control listeners estarán restringidos.

## DB-TXE-102
Listener capability seguirá least privilege.

## DB-TXE-103
Core event names estarán protegidos.

## DB-TXE-104
Custom event no podrá falsificar core lifecycle transition.

## DB-TXE-105
Listener registration errors deberán detectarse temprano.

## DB-TXE-106
Duplicate registration accidental deberá detectarse.

## DB-TXE-107
Persistent workers no acumularán listeners por request.

## DB-TXE-108
Persistent workers no acumularán callbacks entre requests.

## DB-TXE-109
FrankenPHP requests estarán aisladas.

## DB-TXE-110
RoadRunner jobs/requests estarán aislados.

## DB-TXE-111
OpenSwoole coroutines estarán aisladas.

## DB-TXE-112
Fiber-scoped dispatch state estará aislado.

## DB-TXE-113
Worker cleanup verificará dispatch stack vacío.

## DB-TXE-114
Worker cleanup verificará callback registry disposed.

## DB-TXE-115
Worker cleanup verificará event buffer disposed.

## DB-TXE-116
Callbacks de scope anterior jamás ejecutarán en scope posterior.

## DB-TXE-117
Transaction events podrán alimentar Telemetry mediante bridge.

## DB-TXE-118
Telemetry no será dependencia obligatoria del TransactionManager.

## DB-TXE-119
Metrics tendrán bounded cardinality.

## DB-TXE-120
TransactionId no será metric label.

## DB-TXE-121
TenantId no será metric label de alta cardinalidad.

## DB-TXE-122
Raw SQL no será metric label.

## DB-TXE-123
Bound values no serán metric labels.

## DB-TXE-124
Transaction timeline será diagnóstica.

## DB-TXE-125
Transaction timeline no será DB transaction log.

## DB-TXE-126
Cause metadata deberá ser sanitizada.

## DB-TXE-127
Throwable completo no será obligatorio en event payload.

## DB-TXE-128
Correlation metadata no creará global mutable state.

## DB-TXE-129
Callback registration sin active transaction fallará por default.

## DB-TXE-130
afterCommit no se ejecutará inmediatamente de forma implícita sin transaction.

## DB-TXE-131
Convenience behavior alternativo requerirá API explícita.

## DB-TXE-132
Callbacks no podrán registrarse indefinidamente durante su propia fase.

## DB-TXE-133
Callback registry tendrá lifecycle explícito.

## DB-TXE-134
Callback cancellation solo será válida mientras su fase esté abierta.

## DB-TXE-135
Callback deduplication requerirá key explícita.

## DB-TXE-136
Closures arbitrarias no serán deduplicadas mágicamente.

## DB-TXE-137
Coalescing será transaction-local.

## DB-TXE-138
Coalescing no proporcionará durable delivery.

## DB-TXE-139
Dispatch tables podrán compilarse.

## DB-TXE-140
Zero-listener dispatch tendrá fast path.

## DB-TXE-141
Performance optimizations no alterarán ordering.

## DB-TXE-142
Performance optimizations no alterarán outcome semantics.

## DB-TXE-143
Performance optimizations no romperán scope isolation.

## DB-TXE-144
Listener execution estará sujeto a resource governance.

## DB-TXE-145
beforeCommit timeout podrá causar rollback-only.

## DB-TXE-146
afterCommit timeout no podrá deshacer commit.

## DB-TXE-147
Transaction deadline podrá limitar pre-completion listeners.

## DB-TXE-148
Post-commit work podrá tener budget independiente.

## DB-TXE-149
Dispatch recursion será bounded.

## DB-TXE-150
Failure details podrán truncarse sin perder failure count.

## DB-TXE-151
Transaction event tests deberán verificar ordering.

## DB-TXE-152
Tests deberán verificar commit unknown.

## DB-TXE-153
Tests deberán verificar callback failures.

## DB-TXE-154
Tests deberán verificar savepoint pruning.

## DB-TXE-155
Tests deberán verificar nested participation.

## DB-TXE-156
Tests deberán verificar retry correlation.

## DB-TXE-157
Tests deberán verificar reentrancy.

## DB-TXE-158
Tests deberán verificar persistent-runtime isolation.

## DB-TXE-159
Tests deberán verificar coroutine isolation.

## DB-TXE-160
Tests deberán verificar resource limits.

## DB-TXE-161
UNKNOWN nunca será convertido en success por un event listener.

## DB-TXE-162
UNKNOWN nunca disparará afterCommit.

## DB-TXE-163
UNKNOWN nunca disparará afterRollback sin rollback confirmado.

## DB-TXE-164
Commit confirmado permanecerá COMMITTED aunque falle afterCommit.

## DB-TXE-165
Rollback confirmado permanecerá ROLLED_BACK aunque falle afterRollback.

## DB-TXE-166
Event listener no podrá alterar pinned connection.

## DB-TXE-167
Event listener no podrá alterar effective isolation.

## DB-TXE-168
Event listener no podrá alterar TransactionId.

## DB-TXE-169
Event listener no podrá alterar ownership.

## DB-TXE-170
Event listener no podrá alterar nesting stack directamente.

## DB-TXE-171
Event listener no podrá alterar savepoint stack directamente.

## DB-TXE-172
TransactionControl expondrá únicamente mutaciones permitidas.

## DB-TXE-173
Control mutations serán validadas contra lifecycle state.

## DB-TXE-174
No se registrarán callbacks sobre disposed context.

## DB-TXE-175
No se emitirá lifecycle event desde disposed context.

## DB-TXE-176
Terminal transaction no podrá volver a estado ACTIVE mediante eventos.

## DB-TXE-177
Transaction events no crearán distributed atomicity.

## DB-TXE-178
Transaction events no sustituirán database constraints.

## DB-TXE-179
Transaction events no sustituirán concurrency control.

## DB-TXE-180
Transaction events nunca fabricarán certeza sobre un outcome físico desconocido.

---

# 261. Master lifecycle

Modelo final:

```text
Context Created
      │
      ▼
TransactionCreated
      │
      ▼
TransactionStarting
      │
      ▼
    BEGIN
      │
 ┌────┴─────┐
 │          │
 ▼          ▼
OK        FAIL
 │          │
 ▼          ▼
Started   Failed
 │          │
 │          ▼
 │       Completed
 │
 ▼
ACTIVE WORK
 │
 ├───────────────┐
 │               │
 ▼               ▼
COMMIT PATH    ROLLBACK PATH
 │               │
 ▼               ▼
BeforeCommit   BeforeRollback
 │               │
 ▼               ▼
COMMIT          ROLLBACK
 │               │
 ├───────┐       ├───────┐
 │       │       │       │
 ▼       ▼       ▼       ▼
OK    UNKNOWN    OK    UNKNOWN
 │       │       │       │
 ▼       ▼       ▼       ▼
Committed Unknown RolledBack Unknown
 │       │       │       │
 ▼       ▼       ▼       ▼
callbacks/resources/completion
             │
             ▼
     TransactionCompleted
```

---

# 262. Callback lifecycle

```text
Transaction ACTIVE
       │
       ▼
callbacks may register
       │
       ▼
BeforeCommit phase begins
       │
       ├── beforeCommit callbacks
       │
       ▼
physical outcome
       │
   ┌───┼───────────┐
   ▼   ▼           ▼
COMMIT ROLLBACK   UNKNOWN
   │   │           │
   ▼   ▼           │
after after         │
commit rollback     │
   │   │           │
   └───┼───────────┘
       ▼
afterCompletion
       │
       ▼
registry disposed
```

---

# 263. Domain event bridge lifecycle

```text
Domain Mutation
      │
      ▼
Domain Event Produced
      │
      ▼
Transaction-local Buffer
      │
      ├───────────────┐
      │               │
      ▼               ▼
COMMIT             ROLLBACK
      │               │
      ▼               ▼
Publish eligible    Discard
in-process events
```

Para durable external delivery:

```text
Domain Mutation
      +
Outbox Record
      │
      ▼
DB COMMIT
      │
      ▼
Independent Outbox Processor
```

---

# 264. Failure semantics matrix

| Phase | Listener failure | Puede impedir commit | Cambia DB outcome confirmado |
|---|---|---:|---:|
| `TransactionCreated` | abortable según policy | Sí | No |
| `TransactionStarting` | abortable | Sí | No |
| `TransactionStarted` | puede marcar rollback-only | Sí | No |
| `BeforeCommit` | fail/rollback-only | Sí | No |
| `Committed` | record/continue | No | No |
| `BeforeRollback` | record/continue rollback | No | No |
| `RolledBack` | record/continue | No | No |
| `OutcomeUnknown` | record/continue | No | No |
| `Completed` | record/aggregate | No | No |

---

# 265. Event vs callback matrix

| Característica | Event Listener | Transaction Callback |
|---|---|---|
| Registro | Bootstrap | Runtime |
| Scope | Framework/application | Transaction |
| Reutilizable | Sí | Normalmente no |
| Compartible entre workers | Definición sí | No |
| `afterCommit` específico | Puede observar | Sí |
| Transaction-local capture | No recomendado | Sí, bounded |
| Frozen registry | Sí | No |
| Savepoint pruning | Normalmente no | Sí |
| Telemetry | Ideal | No principal |
| Business deferred work | Bridge | Posible |
| Durable messaging | No | No |

---

# 266. Final architecture

```text
                    Transaction Manager
                           │
                           ▼
                  Transaction State Machine
                           │
                           ▼
                  Transaction Context
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
   Lifecycle Events    Callback Registry   Resources
          │                │                │
          ▼                ▼                ▼
   Event Dispatcher   Callback Executor   Bridges
          │                │                │
   ┌──────┼──────┐         │        ┌──────┼───────┐
   ▼      ▼      ▼         │        ▼      ▼       ▼
Telemetry ORM   Cache      │      Domain  Cache   Outbox
   │      │      │         │      Bridge  Bridge  Bridge
   └──────┴──────┴─────────┴────────┴──────┴───────┘
                           │
                           ▼
                  Completion Coordinator
```

---

# 267. Regla maestra

> **Los eventos transaccionales de VoltStack observarán y coordinarán un lifecycle cuya autoridad pertenece al `TransactionManager`, `TransactionStateMachine` y `TransactionContext`; nunca serán utilizados para fingir que una operación externa forma parte de la transacción física ni para reinterpretar un outcome que el motor de base de datos no pudo confirmar.**

En forma compacta:

```text
Event observes state.
Event does not invent state.
```

Y:

```text
afterCommit
≠
Outbox
≠
Distributed Transaction
```

---

# 268. Cierre del Bloque 15

Con este documento queda completado:

```text
BLOCK 15 — TRANSACTIONS & CONCURRENCY

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
```

La arquitectura resultante puede resumirse:

```text
Transaction Architecture
        │
        ├── Manager
        ├── Context
        ├── Isolation
        ├── Nesting
        ├── Savepoints
        ├── Retry
        ├── Deadlock Handling
        ├── Optimistic Locking
        ├── Pessimistic Locking
        ├── Concurrency Control
        └── Transaction Events
```

Con cuatro invariantes transversales especialmente importantes:

```text
flush ≠ commit
rollback ≠ object graph rewind
statement success ≠ transaction success
unknown outcome ≠ rollback
```

---

# 269. Siguiente bloque

A partir del siguiente documento inicia:

```text
BLOCK 16 — READ/WRITE AND DISTRIBUTION
```

con la secuencia:

```text
176_DATABASE_READ_WRITE_CONNECTION_SYSTEM.md
177_DATABASE_READ_WRITE_ROUTING_SYSTEM.md
178_DATABASE_REPLICA_SYSTEM.md
179_DATABASE_REPLICA_LAG_AWARENESS_SYSTEM.md
180_DATABASE_STICKY_CONNECTION_SYSTEM.md
181_DATABASE_FAILOVER_SYSTEM.md
182_DATABASE_LOAD_BALANCING_SYSTEM.md
183_DATABASE_DISTRIBUTED_DATABASE_ARCHITECTURE.md
184_DATABASE_SHARDING_SYSTEM.md
185_DATABASE_PARTITION_ROUTING_SYSTEM.md
```

---

# 270. Siguiente documento

```text
176_DATABASE_READ_WRITE_CONNECTION_SYSTEM.md
```

Este documento establecerá la separación entre conexiones de lectura y escritura:

```text
Logical Connection
        │
        ▼
Read/Write Connection System
        │
   ┌────┴────┐
   ▼         ▼
 READ       WRITE
   │         │
   ▼         ▼
Replica    Primary
Pool       Pool
```

y definirá conceptos como:

```text
ConnectionRole
READ
WRITE
READ_WRITE
ConnectionIntent
ReadConnection
WriteConnection
ConnectionRoleResolver
ReadWriteConnectionSet
ReadWriteConnectionPolicy
authoritative reads
transaction pinning
read-after-write requirements
connection promotion
connection affinity
replica eligibility
platform capabilities
persistent-worker reset
```

manteniendo desde el principio:

```text
Read Connection
≠
Replica
```

y:

```text
Write Connection
≠
Primary Server
```

porque el **rol lógico de una conexión** y la **topología física de distribución** deberán permanecer como conceptos separados.