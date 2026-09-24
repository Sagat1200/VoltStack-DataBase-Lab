# 219_DATABASE_TRANSACTION_TELEMETRY_SYSTEM.md

# VoltStack Quantum Database
## Transaction Telemetry System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 219 — Transaction Telemetry System  
**Bloque:** 21 — Telemetry and Debugging  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `218_DATABASE_CONNECTION_TELEMETRY_SYSTEM.md`  
**Siguiente documento:** `220_DATABASE_ORM_TELEMETRY_SYSTEM.md`

---

# 1. Propósito

`Transaction Telemetry System` define cómo VoltStack observará el ciclo de vida completo de las transacciones de base de datos sin participar en su control.

El sistema deberá permitir responder:

```text
¿Cuándo comenzó la transacción?

¿Cuánto duró?

¿Qué isolation level solicitó?

¿Qué isolation level obtuvo realmente?

¿Qué conexión física utilizó?

¿Qué queries ejecutó?

¿Cuánto tiempo permaneció sin actividad?

¿Utilizó savepoints?

¿Tuvo scopes anidados?

¿Experimentó deadlocks?

¿Fue reintentada?

¿Cuántos intentos existieron?

¿Cuánto tardó el commit?

¿Cuánto tardó el rollback?

¿El commit realmente ocurrió?

¿El resultado quedó UNKNOWN?

¿Se detectaron transacciones excesivamente largas?

¿La transacción quedó abierta accidentalmente?
```

La regla central será:

> **Transaction Telemetry observa la frontera transaccional completa y preserva con precisión su evidencia y resultado, pero nunca inicia, confirma, revierte, reintenta, anida, crea savepoints ni modifica una transacción.**

Formalmente:

```text
Transaction Manager
        ↓
Transaction Lifecycle
        ↓
Transaction Telemetry
        ↓
Observe
        ↓
Correlate
        ↓
Measure
        ↓
Diagnose
```

Nunca:

```text
Transaction Telemetry
        ↓
begin()
commit()
rollback()
retry()
savepoint()
```

---

# 2. Relación con la arquitectura general

```text
Database Telemetry
│
├── Query Telemetry
├── Connection Telemetry
├── Transaction Telemetry     ← este documento
├── ORM Telemetry
├── Query Profiler
├── Slow Query Detection
├── N+1 Telemetry
├── Debug Information
└── Developer Debug Toolbar
```

Transaction Telemetry utilizará información proveniente de:

```text
Transaction Manager
Transaction Context
Connection System
Query Telemetry
Deadlock Handling
Retry System
Savepoint System
Concurrency Control
Read/Write Routing
Distributed Database
Persistent Runtime
```

sin asumir responsabilidad funcional sobre ellos.

---

# 3. Distinciones fundamentales

VoltStack deberá preservar:

```text
Transaction
≠
Transaction Scope

Logical Transaction
≠
Physical Transaction

Transaction
≠
Transaction Attempt

Transaction Attempt
≠
Retry

Nested Transaction
≠
Independent Physical Transaction

Savepoint
≠
Nested Transaction

Savepoint
≠
Transaction

Transaction
≠
UnitOfWork

Transaction
≠
EntityManager

Transaction
≠
Flush

Flush
≠
Commit

Statement Success
≠
Transaction Success

Query Failure
≠
Transaction Failure

Rollback
≠
Object Graph Rewind

Commit Sent
≠
Commit Confirmed

Transaction Duration
≠
Query Duration

Transaction Telemetry
≠
Transaction Manager

Transaction Telemetry
≠
Transaction Retry System

Transaction Telemetry
≠
Deadlock Handler
```

---

# 4. Transaction identity model

La telemetría deberá distinguir al menos:

```text
TransactionOperationId
TransactionId
TransactionAttemptId
TransactionScopeId
SavepointId
```

---

# 5. TransactionOperationId

Representa la operación transaccional lógica solicitada por la aplicación.

```php
final readonly class TransactionOperationId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Ejemplo:

```php
DB::transaction(function () {
    // ...
});
```

La llamada completa constituye una:

```text
TransactionOperation
```

aunque internamente pueda requerir múltiples intentos.

---

# 6. TransactionId

Representa una instancia lógica de transacción.

```php
final readonly class TransactionId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 7. TransactionAttemptId

Cuando existe retry:

```text
TransactionOperation
       │
       ├── Attempt 1
       ├── Attempt 2
       └── Attempt 3
```

cada intento tendrá identidad propia.

```php
final readonly class TransactionAttemptId
{
    public function __construct(
        public TransactionOperationId $operationId,
        public int $attempt,
    ) {}
}
```

---

# 8. Por qué OperationId y AttemptId

Supongamos:

```text
Attempt 1
↓
Deadlock
↓
Rollback
↓
Retry

Attempt 2
↓
Commit
↓
Success
```

El resultado de la operación completa puede ser:

```text
SUCCESS_AFTER_RETRY
```

mientras el primer intento fue:

```text
ROLLED_BACK_AFTER_DEADLOCK
```

No deberán fusionarse.

---

# 9. TransactionScopeId

Los scopes anidados tendrán identidad independiente:

```php
final readonly class TransactionScopeId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Ejemplo:

```text
Transaction
│
├── Scope 0
│
├── Scope 1
│
└── Scope 2
```

---

# 10. SavepointId

```php
final readonly class SavepointId
{
    public function __construct(
        public string $value,
    ) {}
}
```

No necesariamente contendrá el nombre físico enviado al motor.

---

# 11. Transaction hierarchy

```text
TransactionOperation
│
├── Attempt 1
│   │
│   └── Physical Transaction
│       ├── Scope
│       ├── Scope
│       └── Savepoint
│
└── Attempt 2
    │
    └── Physical Transaction
        ├── Scope
        └── Savepoint
```

---

# 12. Logical vs physical transaction

Una transacción lógica representa:

```text
application transactional boundary
```

Una transacción física representa:

```text
actual database transaction
```

Por ejemplo:

```php
DB::transaction(function () {

    serviceA();

    DB::transaction(function () {
        serviceB();
    });
});
```

puede producir:

```text
Logical Scope A
    ↓
Physical Transaction T1

Logical Scope B
    ↓
JOIN T1
```

No existen necesariamente dos transacciones físicas.

---

# 13. Nested transaction strategy

De acuerdo con la arquitectura previamente definida:

```text
JOIN
SAVEPOINT
REJECT
REQUIRES_NEW
```

podrán existir como estrategias.

Telemetry deberá observar cuál fue utilizada.

---

# 14. NestedTransactionTelemetry

```php
final readonly class NestedTransactionTelemetry
{
    public function __construct(
        public TransactionScopeId $scopeId,
        public int $depth,
        public NestedTransactionStrategy $strategy,
        public ?SavepointId $savepointId,
    ) {}
}
```

---

# 15. Depth

La profundidad deberá ser explícita.

```text
Root       depth=0
Nested A   depth=1
Nested B   depth=2
```

---

# 16. Depth ≠ physical transaction count

```text
depth = 4
```

no implica:

```text
4 physical transactions
```

---

# 17. TransactionTelemetryContext

```php
final readonly class TransactionTelemetryContext
{
    public function __construct(
        public TransactionOperationId $operationId,
        public TransactionId $transactionId,
        public TransactionAttemptId $attemptId,
        public ?TransactionScopeId $scopeId,
        public ?PhysicalConnectionId $connectionId,
        public ?ConnectionLeaseId $leaseId,
        public ?PersistenceDomainId $domainId,
        public ?TenantId $tenantId,
        public ?ShardId $shardId,
        public TransactionTelemetryPolicy $policy,
    ) {}
}
```

---

# 18. Transaction lifecycle

Modelo principal:

```text
REQUESTED
    ↓
PREPARING
    ↓
BEGINNING
    ↓
ACTIVE
    ↓
COMMITTING
    ↓
COMMITTED
```

o:

```text
ACTIVE
   ↓
ROLLING_BACK
   ↓
ROLLED_BACK
```

o:

```text
ACTIVE
   ↓
FAILED
```

o:

```text
COMMITTING
   ↓
UNKNOWN
```

---

# 19. TransactionTelemetryState

```php
enum TransactionTelemetryState
{
    case REQUESTED;
    case PREPARING;
    case BEGINNING;
    case ACTIVE;
    case COMMITTING;
    case COMMITTED;
    case ROLLING_BACK;
    case ROLLED_BACK;
    case FAILED;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 20. Telemetry state ≠ functional state authority

El estado funcional pertenece al:

```text
Transaction Manager
+
Transaction Context
```

Telemetry únicamente refleja evidencia observada.

---

# 21. Transaction outcome

Se deberá modelar explícitamente.

```php
enum TransactionOutcome
{
    case COMMITTED;
    case ROLLED_BACK;
    case FAILED_TO_BEGIN;
    case FAILED;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 22. UNKNOWN como estado de primera clase

Caso crítico:

```text
Application
    ↓
COMMIT
    ↓
Database receives commit
    ↓
Database commits
    ↓
Network connection dies
    ↓
Application receives no confirmation
```

Desde VoltStack:

```text
CommitOutcome = UNKNOWN
```

No:

```text
FAILED
```

y tampoco:

```text
COMMITTED
```

---

# 23. Regla fundamental del commit

```text
Commit Sent
≠
Commit Confirmed
```

---

# 24. Statement success ≠ commit success

Puede ocurrir:

```text
INSERT ✓
UPDATE ✓
DELETE ✓
COMMIT ?
```

Por tanto:

```text
All Statements Successful
≠
Transaction Committed
```

---

# 25. Transaction phases

```php
enum TransactionTelemetryPhase
{
    case RESOLUTION;
    case CONNECTION_ACQUISITION;
    case BEGIN;
    case ACTIVE;
    case QUERY_EXECUTION;
    case IDLE;
    case SAVEPOINT_CREATE;
    case SAVEPOINT_RELEASE;
    case SAVEPOINT_ROLLBACK;
    case COMMIT;
    case ROLLBACK;
    case RETRY_DELAY;
    case CLEANUP;
}
```

---

# 26. Transaction timing

Se deberán medir:

```text
begin duration
active duration
query execution time
idle time
commit duration
rollback duration
retry delay
total operation duration
```

---

# 27. Monotonic clock

Toda duración deberá utilizar un reloj monotónico cuando esté disponible.

---

# 28. Transaction total duration

Para un intento:

```text
T_attempt
=
T_end
-
T_begin_request
```

---

# 29. Physical transaction duration

```text
T_physical
=
T_transaction_end
-
T_database_begin_confirmed
```

cuando ambas observaciones existan.

---

# 30. Operation duration

Con retries:

```text
T_operation
=
Σ T_attempt
+
Σ T_retry_delay
+
T_orchestration
```

---

# 31. Operation duration ≠ final attempt duration

Ejemplo:

```text
Attempt 1 = 800 ms
Backoff   = 200 ms
Attempt 2 = 500 ms

Operation ≈ 1.5 s
```

No:

```text
500 ms
```

---

# 32. Time to first statement

Métrica diagnóstica:

```text
T_first_statement
=
T_first_query_start
-
T_transaction_begin
```

---

# 33. Por qué importa

Una transacción puede abrirse:

```text
BEGIN
↓
application computation
↓
HTTP request
↓
filesystem operation
↓
first SQL query
```

manteniendo recursos innecesariamente.

---

# 34. Idle-in-transaction

Se deberá poder medir:

```text
Transaction Active
↓
No Database Activity
↓
Transaction Still Open
```

---

# 35. Idle time

Conceptualmente:

```text
T_idle
=
T_transaction_active
-
Σ T_database_activity
```

con las limitaciones de observabilidad correspondientes.

---

# 36. Idle ≠ CPU idle

Se refiere a:

```text
database transaction open without observed database operation
```

no a uso de CPU del proceso.

---

# 37. Long transaction detection

Transaction Telemetry proporcionará evidencia para detectar:

```text
long-running transaction
```

---

# 38. Long transaction ≠ failure

Una migración o proceso administrativo puede requerir una transacción larga.

Por tanto:

```text
LongTransaction
≠
FailedTransaction
```

---

# 39. Transaction duration thresholds

Podrán existir thresholds distintos por:

```text
QUERY
APPLICATION
MIGRATION
MAINTENANCE
BATCH
INTERNAL
```

---

# 40. Transaction purpose

```php
enum TransactionPurpose
{
    case APPLICATION;
    case ORM_FLUSH;
    case MIGRATION;
    case SEEDING;
    case IMPORT;
    case BATCH;
    case MAINTENANCE;
    case TEST;
    case INTERNAL;
    case UNKNOWN;
}
```

---

# 41. Purpose bounded

`TransactionPurpose` podrá utilizarse como dimensión métrica por ser bounded.

---

# 42. Transaction ownership

Telemetry deberá poder observar:

```text
TransactionOwnership
```

---

# 43. Ownership model

```php
enum TransactionOwnership
{
    case FRAMEWORK;
    case CALLER;
    case EXTERNAL;
    case UNKNOWN;
}
```

---

# 44. Ownership ≠ telemetry control

Conocer al owner no autoriza a telemetry a:

```text
commit
rollback
```

---

# 45. Connection acquisition

Una transacción normalmente requiere:

```text
Transaction
↓
Connection Lease
↓
Physical Connection
```

---

# 46. Connection pinning

Durante una transacción:

```text
TransactionAttemptId
        ↓
PhysicalConnectionId
```

deberá mantenerse estable salvo arquitectura explícita.

---

# 47. Transaction connection affinity

Telemetry podrá comprobar:

```text
all transaction queries
↓
same physical connection
```

---

# 48. Affinity violation

Si se observa:

```text
BEGIN on Connection A
QUERY on Connection A
QUERY on Connection B
COMMIT on Connection A
```

podrá producir:

```text
TRANSACTION_CONNECTION_AFFINITY_VIOLATION
```

---

# 49. Affinity telemetry ≠ enforcement

El Transaction Manager/Connection Manager debe impedirlo.

Telemetry únicamente diagnostica.

---

# 50. Connection lease duration

Una transacción puede mantener un lease durante toda su duración.

Por ello:

```text
TransactionDuration
≈
ConnectionLeaseDuration
```

en algunos casos.

Pero no siempre son idénticos.

---

# 51. Connection Telemetry correlation

```text
TransactionAttemptId
       ↓
ConnectionLeaseId
       ↓
PhysicalConnectionId
       ↓
EndpointId
```

---

# 52. Query correlation

Cada query dentro de la transacción deberá poder correlacionarse:

```text
TransactionAttemptId
        │
        ├── QueryOperationId Q1
        ├── QueryOperationId Q2
        └── QueryOperationId Q3
```

---

# 53. Queries per transaction

Podrán calcularse:

```text
query count
statement count
read count
write count
failed query count
total query duration
```

---

# 54. Query count ≠ statement count

Un Query Operation puede potencialmente producir más de una operación física en arquitecturas especiales.

Por tanto ambas métricas permanecerán conceptualmente distintas.

---

# 55. Transaction query duration

```text
T_queries
=
Σ QueryExecutionDuration
```

---

# 56. Query percentage

Diagnóstico posible:

```text
QueryTimeRatio
=
T_queries
/
T_transaction
```

---

# 57. Low query ratio

Una transacción:

```text
duration = 10s
SQL time = 200ms
```

puede indicar mucho tiempo de aplicación dentro de la frontera transaccional.

No prueba automáticamente un problema.

---

# 58. Isolation requested

Telemetry deberá registrar:

```text
RequestedIsolationLevel
```

---

# 59. Isolation effective

Separadamente:

```text
EffectiveIsolationLevel
```

---

# 60. Requested ≠ effective

Ejemplo:

```text
Requested:
SERIALIZABLE

Effective:
REPEATABLE_READ
```

si existiese una política explícita de downgrade.

La diferencia debe ser visible.

---

# 61. Silent downgrade prohibited

La arquitectura transaccional ya establece:

> No se degradará silenciosamente un isolation level solicitado.

Telemetry reforzará la observabilidad de esta regla.

---

# 62. Isolation model

```php
final readonly class TransactionIsolationTelemetry
{
    public function __construct(
        public TransactionIsolationLevel $requested,
        public ?TransactionIsolationLevel $effective,
        public IsolationResolutionOutcome $resolution,
    ) {}
}
```

---

# 63. UNKNOWN isolation

Si no puede comprobarse el nivel efectivo:

```text
effective = UNKNOWN
```

no:

```text
effective = requested
```

por suposición.

---

# 64. Read-only transaction

Telemetry podrá registrar:

```text
read_only=true
```

cuando sea una propiedad explícita.

---

# 65. Read-only ≠ no writes observed

```text
NoWritesObserved
≠
ReadOnlyTransaction
```

---

# 66. Transaction nesting

Ejemplo:

```text
Root Transaction
│
├── Scope A
│   └── JOIN
│
├── Scope B
│   └── SAVEPOINT sp_1
│
└── Scope C
    └── JOIN
```

---

# 67. Nested telemetry record

```php
final readonly class TransactionScopeTelemetryRecord
{
    public function __construct(
        public TransactionScopeId $scopeId,
        public int $depth,
        public NestedTransactionStrategy $strategy,
        public TransactionScopeOutcome $outcome,
        public Duration $duration,
    ) {}
}
```

---

# 68. Scope outcome

```php
enum TransactionScopeOutcome
{
    case COMPLETED;
    case ROLLED_BACK_TO_SAVEPOINT;
    case MARKED_ROLLBACK_ONLY;
    case FAILED;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 69. Scope success ≠ transaction commit

Un nested scope puede completar correctamente y posteriormente la transacción root hacer rollback.

---

# 70. Savepoint telemetry

Deberá observar:

```text
savepoint create
savepoint release
rollback to savepoint
savepoint failure
```

---

# 71. Savepoint identity

Preferir:

```text
SavepointId
```

en telemetry.

El nombre SQL físico solo será diagnóstico opcional.

---

# 72. Savepoint lifecycle

```text
REQUESTED
↓
CREATED
↓
ACTIVE
↓
RELEASED
```

o:

```text
ACTIVE
↓
ROLLBACK_REQUESTED
↓
ROLLED_BACK
```

---

# 73. Savepoint outcome

```php
enum SavepointOutcome
{
    case CREATED;
    case RELEASED;
    case ROLLED_BACK;
    case FAILED;
    case UNKNOWN;
}
```

---

# 74. Savepoint ≠ transaction

Rollback a savepoint:

```text
ROLLBACK TO SAVEPOINT
```

no significa:

```text
whole transaction rolled back
```

---

# 75. Retry architecture

Una operación transaccional puede tener:

```text
Operation
├── Attempt 1
├── Attempt 2
└── Attempt 3
```

---

# 76. Retry reason

```php
enum TransactionRetryReason
{
    case DEADLOCK;
    case SERIALIZATION_FAILURE;
    case TRANSIENT_CONNECTION_FAILURE;
    case LOCK_TIMEOUT;
    case RETRYABLE_PLATFORM_ERROR;
    case APPLICATION_POLICY;
    case UNKNOWN;
}
```

---

# 77. Retry decision ownership

La decisión pertenece al:

```text
Transaction Retry System
```

Telemetry solo observa:

```text
retry scheduled
retry reason
attempt number
backoff
final outcome
```

---

# 78. Retry count

```text
RetryCount
=
AttemptCount - 1
```

cuando todos los intentos forman una secuencia lineal normal.

---

# 79. Retry backoff

Se medirá:

```text
db.transaction.retry.delay
```

---

# 80. Retry attempt metrics

```text
db.transaction.attempts
db.transaction.retries
db.transaction.retry.delay
```

---

# 81. Retry safety

Telemetry podrá registrar la evaluación:

```text
SAFE
UNSAFE
UNKNOWN
```

si el Transaction Retry System la proporciona.

---

# 82. UNKNOWN commit prohibits blind retry

Caso:

```text
COMMIT sent
↓
connection lost
↓
outcome UNKNOWN
```

No deberá convertirse en:

```text
retry attempt
```

automáticamente.

---

# 83. Telemetry diagnostic

Deberá hacer visible:

```text
Retry suppressed:
previous commit outcome UNKNOWN
```

cuando el sistema funcional proporcione esa evidencia.

---

# 84. Deadlock telemetry

Una transacción podrá correlacionarse con:

```text
DeadlockDetected
```

---

# 85. Deadlock ≠ transaction failure final

Puede ocurrir:

```text
Attempt 1
DEADLOCK
ROLLBACK

Attempt 2
COMMIT
```

Resultado global:

```text
SUCCESS_AFTER_RETRY
```

---

# 86. Deadlock metrics

```text
db.transaction.deadlocks
```

con dimensiones bounded como:

```text
db.system
transaction.purpose
```

---

# 87. Deadlock details

Información detallada como:

```text
table
index
SQL
lock resource
process ID
```

estará sujeta a security/cardinality policy.

---

# 88. Lock wait correlation

Transaction Telemetry podrá consumir información del sistema de locking.

```text
Transaction
↓
Lock Wait
↓
Query
```

---

# 89. Lock wait duration

Podrá exponerse:

```text
db.transaction.lock_wait.duration
```

si la plataforma proporciona evidencia confiable.

---

# 90. Lock wait ≠ idle time

```text
WaitingForDatabaseLock
≠
ApplicationIdleInTransaction
```

---

# 91. Pessimistic locking

Telemetry podrá registrar que una operación utilizó:

```text
FOR UPDATE
FOR SHARE
platform equivalent
```

de forma semántica.

---

# 92. Optimistic locking

Los conflictos optimistas pertenecen principalmente al ORM/Persistence Telemetry.

Pero podrán correlacionarse con la transacción.

---

# 93. Commit telemetry

Se medirá:

```text
commit requested
commit sent
commit confirmed
commit duration
commit outcome
```

cuando el driver permita distinguir las fases.

---

# 94. Commit phases

```text
COMMIT_REQUESTED
↓
COMMIT_SENT
↓
COMMIT_CONFIRMED
```

---

# 95. Driver limitations

Algunos drivers solo permiten observar:

```text
commit() called
↓
returned success/failure
```

En ese caso:

```text
coverage = PARTIAL
```

---

# 96. Commit duration

```text
T_commit
=
T_commit_return
-
T_commit_call
```

en la observación mínima.

---

# 97. Commit failure categories

```php
enum TransactionCommitFailureCategory
{
    case CONNECTION_LOST;
    case DEADLOCK;
    case SERIALIZATION_FAILURE;
    case CONSTRAINT;
    case TIMEOUT;
    case DRIVER;
    case SERVER;
    case UNKNOWN;
}
```

---

# 98. Commit UNKNOWN

`CONNECTION_LOST` durante commit puede implicar:

```text
UNKNOWN
```

y no necesariamente:

```text
FAILED
```

---

# 99. Rollback telemetry

Se observará:

```text
rollback requested
rollback duration
rollback confirmed
rollback failure
```

---

# 100. Rollback outcome

```php
enum TransactionRollbackOutcome
{
    case ROLLED_BACK;
    case FAILED;
    case CONNECTION_LOST;
    case UNKNOWN;
}
```

---

# 101. Rollback failure

Una falla de rollback puede dejar:

```text
TransactionContext
```

en estado:

```text
TAINTED
```

según el Transaction System.

Telemetry deberá reflejarlo.

---

# 102. Rollback ≠ object graph rewind

La telemetría nunca deberá comunicar:

```text
Application state restored
```

solo porque el DB rollback tuvo éxito.

---

# 103. ORM correlation

Ejemplo:

```text
ORM Flush
↓
Transaction
↓
INSERT
UPDATE
DELETE
↓
Commit
```

Transaction Telemetry podrá correlacionar:

```text
ORMFlushId
```

sin asumir semántica del UnitOfWork.

---

# 104. flush() ≠ commit()

Una telemetría deberá evitar mensajes como:

```text
ORM flush committed
```

si solo ocurrió:

```text
flush
```

---

# 105. Transaction event correlation

El sistema definido en:

```text
175_DATABASE_TRANSACTION_EVENT_SYSTEM.md
212_DATABASE_TRANSACTION_EVENT_PIPELINE.md
```

es distinto de Transaction Telemetry.

---

# 106. Event ≠ telemetry signal

```text
Transaction Event
≠
Telemetry Signal
```

---

# 107. Transaction Event

Puede formar parte de extensibilidad funcional.

---

# 108. Telemetry Signal

Existe exclusivamente para observabilidad.

---

# 109. Event listener failure

Si un listener afterCommit falla:

```text
Database commit
=
COMMITTED
```

debe permanecer así.

---

# 110. afterCommit failure ≠ transaction failure

Regla:

```text
COMMIT confirmed
+
afterCommit listener failure
=
Transaction COMMITTED
```

con:

```text
PostCommitSideEffectFailure
```

separado.

---

# 111. External side effects

Ejemplo:

```text
Transaction
↓
Send Email
↓
Commit fails
```

El rollback no puede deshacer el email.

---

# 112. Telemetry representation

Deberá distinguir:

```text
Database Transaction Outcome
```

de:

```text
External Side Effect Outcome
```

---

# 113. Outbox correlation

Cuando se use Outbox:

```text
Transaction
↓
Outbox Record
↓
Commit
↓
Async Dispatch
```

podrá existir:

```text
OutboxCorrelationId
```

sin convertir Transaction Telemetry en Outbox System.

---

# 114. Transaction leak

Conceptualmente:

```text
BEGIN
↓
ACTIVE
↓
operation scope ends
↓
no COMMIT
no ROLLBACK
```

puede producir:

```text
TRANSACTION_LEAK_CANDIDATE
```

---

# 115. Leak candidate ≠ proven leak

Una transacción puede legítimamente sobrevivir un scope particular si la arquitectura explícitamente lo permite.

---

# 116. TransactionLeakAssessment

```php
final readonly class TransactionLeakAssessment
{
    public function __construct(
        public LeakConfidence $confidence,
        public Duration $age,
        public TransactionTelemetryState $state,
        public TransactionPurpose $purpose,
    ) {}
}
```

---

# 117. Leak evidence

Puede considerar:

```text
scope ended
connection lease still active
transaction still ACTIVE
no query activity
deadline exceeded
owner unavailable
```

---

# 118. Long idle transaction

Será una categoría diagnóstica particularmente importante.

```text
Transaction age: 60s
Last query: 55s ago
State: ACTIVE
```

---

# 119. Resource consequences

Una transacción larga puede contribuir a:

```text
held locks
MVCC version retention
connection occupation
replication effects
vacuum pressure
deadlocks
```

dependiendo de la plataforma.

Telemetry deberá evitar afirmar estos efectos sin evidencia.

---

# 120. Transaction budget

Podrá existir:

```php
final readonly class TransactionTelemetryBudget
{
    public function __construct(
        public Duration $longTransactionThreshold,
        public Duration $idleTransactionThreshold,
        public int $maxQueriesForDiagnostics,
        public int $maxSavepointsForDiagnostics,
        public int $maxTrackedTransactions,
    ) {}
}
```

---

# 121. Telemetry budget ≠ transaction execution budget

No confundir con:

```text
max transaction duration policy
```

que pertenece a Resource Governance.

---

# 122. Resource Governance correlation

Transaction Telemetry podrá observar:

```text
deadline
budget
cancellation
```

pero no imponerlos.

---

# 123. Cancellation

Una transacción puede terminar por:

```text
request cancellation
deadline
worker shutdown
application cancellation
```

---

# 124. Cancellation outcome

Si rollback se confirma:

```text
CANCELLED_ROLLED_BACK
```

puede ser diagnóstico.

Si el estado DB queda incierto:

```text
CANCELLED_UNKNOWN
```

---

# 125. Cancellation ≠ rollback

Solicitar cancelación no garantiza rollback exitoso.

---

# 126. Timeout

Distinguiremos:

```text
transaction timeout
query timeout
lock timeout
connection timeout
```

---

# 127. Timeout taxonomy

```php
enum TransactionTimeoutCategory
{
    case TRANSACTION_DEADLINE;
    case QUERY;
    case LOCK;
    case CONNECTION;
    case COMMIT;
    case ROLLBACK;
    case UNKNOWN;
}
```

---

# 128. Transaction metrics

Conjunto principal:

```text
db.transaction.started
db.transaction.completed
db.transaction.duration

db.transaction.active

db.transaction.begin.duration
db.transaction.commit.duration
db.transaction.rollback.duration

db.transaction.queries
db.transaction.query.duration

db.transaction.idle.duration

db.transaction.savepoints
db.transaction.nested_scopes

db.transaction.retries
db.transaction.retry.delay

db.transaction.deadlocks
db.transaction.lock_wait.duration

db.transaction.failures
db.transaction.unknown_outcomes

db.transaction.long_running
db.transaction.leak_candidates
```

---

# 129. Bounded metric dimensions

Recomendadas:

```text
db.system
transaction.outcome
transaction.purpose
transaction.isolation
transaction.read_only
transaction.retry_reason
transaction.failure_category
transaction.nesting_strategy
runtime
```

---

# 130. Dangerous dimensions

No deberán utilizarse como labels métricos por default:

```text
transaction.id
operation.id
attempt.id
connection.id
lease.id
query.id
tenant.id
shard.id
savepoint.id
raw SQL
table name
user ID
request ID
```

---

# 131. Transaction ID usage

Sí será útil en:

```text
traces
diagnostic records
debug toolbar
structured logs
```

---

# 132. Transaction spans

Podrá existir:

```text
db.transaction
```

como span.

---

# 133. Attempt spans

Con retries:

```text
db.transaction
│
├── db.transaction.attempt
│
└── db.transaction.attempt
```

o mediante links según el tracing provider.

---

# 134. Query child spans

Conceptualmente:

```text
db.transaction.attempt
│
├── db.query
├── db.query
├── db.query
└── db.transaction.commit
```

---

# 135. Commit span

Puede ser span/event según policy.

Evitar instrumentación excesiva.

---

# 136. Savepoint spans

No deberán producir spans por default en aplicaciones con alto nesting.

Preferir:

```text
events
counters
diagnostics
```

---

# 137. Span attributes

Ejemplo seguro:

```text
db.system=postgresql
db.transaction.isolation=read_committed
db.transaction.read_only=false
db.transaction.attempt=2
db.transaction.outcome=committed
```

---

# 138. Attempt number

Es bounded por operation local, pero deberá existir un límite para evitar valores absurdos.

---

# 139. Transaction events for tracing

Ejemplos:

```text
transaction.begin
transaction.savepoint
transaction.retry
transaction.commit
transaction.rollback
transaction.unknown
```

---

# 140. Event explosion protection

Guardar cada statement como transaction event sería redundante porque Query Telemetry ya existe.

---

# 141. Canonical ownership

```text
Query duration
→ Query Telemetry

Connection acquisition
→ Connection Telemetry

Transaction duration
→ Transaction Telemetry
```

---

# 142. No duplicate timing

Otros subsistemas deberán correlacionar las mediciones canónicas en vez de producir valores incompatibles.

---

# 143. Transaction summary

```php
final readonly class TransactionTelemetrySummary
{
    public function __construct(
        public TransactionOperationId $operationId,
        public int $attempts,
        public int $queries,
        public int $writes,
        public int $savepoints,
        public Duration $duration,
        public Duration $queryDuration,
        public Duration $idleDuration,
        public TransactionOutcome $outcome,
    ) {}
}
```

---

# 144. Attempt record

```php
final readonly class TransactionAttemptTelemetryRecord
{
    public function __construct(
        public TransactionAttemptId $attemptId,
        public TransactionIsolationTelemetry $isolation,
        public TransactionPurpose $purpose,
        public ?PhysicalConnectionId $connectionId,
        public int $queryCount,
        public int $savepointCount,
        public Duration $duration,
        public TransactionOutcome $outcome,
        public TransactionTelemetryCoverage $coverage,
    ) {}
}
```

---

# 145. Observation coverage

```php
enum TransactionTelemetryCoverage
{
    case COMPLETE;
    case PARTIAL;
    case MINIMAL;
    case UNKNOWN;
}
```

---

# 146. Missing observation ≠ zero

Si no sabemos cuánto tiempo estuvo idle:

```text
idle_duration = UNKNOWN
```

No:

```text
0ms
```

---

# 147. Distributed database boundaries

VoltStack no deberá fingir una transacción global ACID sobre múltiples shards.

---

# 148. Single-shard transaction

Caso normal:

```text
Transaction
↓
Shard 17
↓
Writer
↓
Physical Connection
```

---

# 149. Cross-shard operation

Si una operación de aplicación toca:

```text
Shard A
Shard B
```

sin protocolo transaccional distribuido:

```text
Application Operation
≠
Single Database Transaction
```

---

# 150. Telemetry representation

Preferir:

```text
DistributedOperationId
├── Transaction A / Shard A
└── Transaction B / Shard B
```

No:

```text
GlobalTransaction(COMMITTED)
```

si no existe realmente.

---

# 151. No fake 2PC

Transaction Telemetry nunca inferirá:

```text
distributed atomic commit
```

solo porque todas las transacciones observadas terminaron en success.

---

# 152. Partial distributed outcome

Puede existir:

```text
Shard A = COMMITTED
Shard B = FAILED
```

La operación distribuida deberá conservar esa realidad.

---

# 153. Tenant context

Cada Transaction Telemetry Context podrá incluir:

```text
TenantId
```

cuando Multitenancy esté instalado.

---

# 154. TenantId cardinality

No será metric label por default.

---

# 155. Tenant drift

Durante una transacción:

```text
TenantContext(begin)
```

deberá coincidir con:

```text
TenantContext(query)
```

salvo arquitectura explícita.

---

# 156. Tenant context violation

Podrá producir:

```text
TRANSACTION_TENANT_CONTEXT_DRIFT
```

como diagnóstico.

---

# 157. Shard affinity

Igualmente:

```text
TransactionAttempt
↓
ShardId
```

deberá permanecer estable para una transacción single-shard.

---

# 158. Shard drift

```text
Transaction begins on Shard A
↓
Query routed to Shard B
```

será una violación arquitectónica salvo modelo explícito.

---

# 159. Read/write routing

Una transacción normalmente deberá pinnearse al writer cuando sus semánticas lo requieran.

Transaction Telemetry podrá observar:

```text
connection.role=writer
```

---

# 160. Routing decision ownership

La decisión sigue perteneciendo a:

```text
Read/Write Routing System
```

---

# 161. Replica transaction

Si se permiten transacciones read-only en replicas:

```text
read_only=true
role=replica
```

deberá representarse explícitamente.

---

# 162. Replica consistency

Telemetry podrá correlacionar:

```text
replica position
freshness evidence
sticky state
```

cuando esté disponible.

No decidirá elegibilidad.

---

# 163. Failover during transaction

Regla previa:

> No existe failover transparente seguro en mitad de una transacción física ordinaria.

---

# 164. Connection loss mid-transaction

Normalmente:

```text
ACTIVE
↓
CONNECTION LOST
↓
Transaction outcome
=
FAILED or UNKNOWN
```

según la fase/evidencia.

---

# 165. Connection loss before commit

Si se sabe que no se envió commit:

```text
FAILED
```

puede ser razonable.

---

# 166. Connection loss during commit

Si commit pudo llegar al servidor:

```text
UNKNOWN
```

deberá preservarse.

---

# 167. Persistent runtime

En FrankenPHP:

```text
Worker
├── Request A
│   └── Transaction TA
├── Request B
│   └── Transaction TB
└── Request C
```

TA no deberá contaminar TB.

---

# 168. Request end

Al terminar un request:

```text
active transaction
```

deberá ser manejada por Transaction Lifecycle/Reset System.

Telemetry podrá observar:

```text
ACTIVE_TRANSACTION_AT_SCOPE_END
```

---

# 169. Telemetry does not rollback leaked transaction

Nunca:

```text
Telemetry detects leak
↓
Telemetry rollback()
```

---

# 170. RoadRunner jobs

Cada job deberá tener:

```text
fresh TransactionTelemetryContext
```

---

# 171. OpenSwoole

Con coroutines:

```text
Coroutine A
└── Transaction A

Coroutine B
└── Transaction B
```

los contextos deberán permanecer separados.

---

# 172. No static current transaction

Prohibido:

```php
static ?TransactionId $current;
```

como mecanismo global mutable.

---

# 173. Context propagation

Deberá realizarse mediante:

```text
Operation Scope
Request Scope
Coroutine Context
Explicit Context
```

según Runtime System.

---

# 174. Transaction context reuse

Objetos de metadata/configuración inmutables podrán compartirse.

Estado mutable de una transacción no.

---

# 175. Worker reset

Al finalizar operación:

```text
TransactionTelemetryScope
↓
close
↓
references released
```

---

# 176. Security model

Transaction telemetry puede revelar:

```text
transaction patterns
database usage
tenant activity
business workflow timing
query relationships
```

por lo que deberá someterse a:

```text
Minimization
Redaction
Cardinality Policy
Access Policy
```

---

# 177. SQL capture

El SQL pertenece principalmente a Query Telemetry.

Transaction Telemetry deberá referenciar:

```text
QueryId
```

en lugar de copiar SQL repetidamente.

---

# 178. Query parameters

Nunca deberán copiarse automáticamente al transaction record.

---

# 179. User identifiers

No deberán añadirse como metric dimensions.

---

# 180. Transaction names

Si la aplicación permite:

```php
DB::transaction(
    name: 'checkout',
    callback: ...
);
```

un nombre lógico controlado podría utilizarse en diagnostics.

---

# 181. Transaction name cardinality

Solo podrá utilizarse como métrica si proviene de un registry bounded.

Nunca:

```php
name: "checkout-user-{$userId}"
```

como label.

---

# 182. Diagnostic record

Ejemplo:

```text
TRANSACTION
────────────────────────────────

Operation:
  txop_93

Attempt:
  2 / 2

Purpose:
  APPLICATION

Database:
  PostgreSQL

Isolation:
  Requested: READ_COMMITTED
  Effective: READ_COMMITTED

Read Only:
  false

Connection:
  pc_0042

Role:
  WRITER

Queries:
  12

Writes:
  4

Savepoints:
  1

Duration:
  482 ms

Query Time:
  126 ms

Idle Time:
  319 ms

Commit:
  14 ms

Previous Attempt:
  DEADLOCK

Final Outcome:
  COMMITTED

Telemetry Coverage:
  COMPLETE
```

---

# 183. UNKNOWN diagnostic

```text
TRANSACTION
────────────────────────────────

Attempt:
  1

State:
  COMMITTING

Commit Request:
  sent

Connection:
  lost before confirmation

Database Outcome:
  UNKNOWN

Automatic Retry:
  suppressed

Reason:
  commit may have reached database

Recommended Action:
  reconcile using application-level idempotency
  or durable business operation identifier
```

---

# 184. Evidence vs recommendation

Diagnostics deberán separar:

```text
Observed
Inferred
Recommended
```

---

# 185. Example

```text
Observed:
  Transaction remained open for 18.2s.
  SQL execution consumed 320ms.
  Last query completed 16.9s before commit.

Inference:
  Most transaction lifetime occurred outside database execution.

Recommendation:
  Review application work performed inside the transaction boundary.
```

---

# 186. No false certainty

No:

```text
Your PHP code is slow.
```

sin evidencia suficiente.

---

# 187. TransactionTelemetryInstrumentation

```php
interface TransactionTelemetryInstrumentation
{
    public function operationStarted(
        TransactionTelemetryContext $context,
    ): TransactionTelemetryScope;

    public function attemptStarted(
        TransactionAttemptId $attempt,
    ): TransactionAttemptTelemetryScope;
}
```

---

# 188. Attempt scope

```php
interface TransactionAttemptTelemetryScope
{
    public function beginStarted(): void;

    public function beginCompleted(): void;

    public function queryObserved(QueryOperationId $query): void;

    public function savepointObserved(SavepointId $savepoint): void;

    public function commitStarted(): void;

    public function commitCompleted(TransactionOutcome $outcome): void;

    public function rollbackStarted(): void;

    public function rollbackCompleted(TransactionRollbackOutcome $outcome): void;

    public function close(): void;
}
```

---

# 189. Hot path requirements

Métodos como:

```text
queryObserved()
```

deberán ser extremadamente baratos.

---

# 190. No SQL parsing in transaction telemetry

Prohibido:

```text
Transaction Telemetry
↓
parse every SQL statement
```

para determinar su semántica.

Query Telemetry/Query Model deberá proporcionar clasificación.

---

# 191. No stack trace by default

No capturar stack traces para cada:

```text
begin
query
savepoint
commit
```

---

# 192. Stack trace diagnostic mode

Podrá activarse selectivamente para:

```text
long transaction
transaction leak
unexpected nesting
```

con budget.

---

# 193. Transaction telemetry configuration

```php
return [
    'database' => [
        'telemetry' => [
            'transaction' => [
                'enabled' => true,

                'metrics' => true,
                'tracing' => true,

                'track_queries' => true,
                'track_idle_time' => true,
                'track_savepoints' => true,
                'track_retries' => true,

                'long_transaction_threshold' => '2s',
                'idle_transaction_threshold' => '1s',

                'diagnostics' => [
                    'enabled' => true,
                    'detect_leaks' => true,
                    'capture_stack_on_long_transaction' => false,
                ],
            ],
        ],
    ],
];
```

---

# 194. TransactionTelemetryPolicy

```php
final readonly class TransactionTelemetryPolicy
{
    public function __construct(
        public bool $metrics,
        public bool $tracing,
        public bool $trackQueries,
        public bool $trackIdleTime,
        public bool $trackSavepoints,
        public bool $trackRetries,
        public bool $detectLeaks,
        public Duration $longTransactionThreshold,
        public Duration $idleTransactionThreshold,
        public TransactionTelemetryBudget $budget,
    ) {}
}
```

---

# 195. Compiled policy

La configuración deberá compilarse durante bootstrap:

```text
Config
↓
Validation
↓
Compilation
↓
Immutable TransactionTelemetryPolicy
```

---

# 196. Sampling

Podrán existir:

```text
metrics always
trace sampled
diagnostics conditionally promoted
```

---

# 197. Long transaction promotion

Una transacción inicialmente no trazada podrá promoverse a diagnostic record cuando exceda:

```text
long_transaction_threshold
```

si existe contexto mínimo suficiente.

---

# 198. UNKNOWN promotion

Todo:

```text
TransactionOutcome::UNKNOWN
```

deberá considerarse de alta importancia diagnóstica.

---

# 199. Failure sampling

Failures podrán utilizar una tasa de sampling mayor que operaciones exitosas.

---

# 200. Bounded transaction registry

Para detectar transacciones activas:

```text
ActiveTransactionRegistry
```

deberá ser bounded.

---

# 201. Registry record

```php
final class ActiveTransactionTelemetryRecord
{
    public TransactionAttemptId $attemptId;

    public MonotonicTimestamp $startedAt;

    public MonotonicTimestamp $lastActivityAt;

    public int $queryCount;

    public TransactionTelemetryState $state;
}
```

---

# 202. Registry overflow

Si se supera el budget:

```text
transaction.telemetry.records_dropped++
```

No deberá causar memory exhaustion.

---

# 203. Telemetry paradox

> **VoltStack no deberá agotar memoria, conexiones o CPU intentando observar transacciones que ya están bajo presión de recursos.**

---

# 204. Exporter failure

Una falla del exporter:

```text
OpenTelemetry unavailable
metrics backend down
```

no deberá provocar rollback de una transacción de aplicación.

---

# 205. Fail-open runtime policy

Normalmente:

```text
Telemetry Export Failure
↓
Drop / Buffer / Internal Diagnostic
↓
Transaction continues
```

---

# 206. Security configuration failure

Una política insegura puede rechazarse durante bootstrap.

---

# 207. Testing API

```php
TransactionTelemetry::assertTransactionCount(1);
```

---

# 208. Commit assertion

```php
TransactionTelemetry::assertCommitted();
```

---

# 209. Rollback assertion

```php
TransactionTelemetry::assertRolledBack();
```

---

# 210. Query count assertion

```php
TransactionTelemetry::assertQueryCount(5);
```

---

# 211. Isolation assertion

```php
TransactionTelemetry::assertIsolation(
    TransactionIsolationLevel::SERIALIZABLE
);
```

---

# 212. Retry assertion

```php
TransactionTelemetry::assertRetried(times: 2);
```

---

# 213. Deadlock assertion

```php
TransactionTelemetry::assertDeadlockObserved();
```

---

# 214. Unknown assertion

```php
TransactionTelemetry::assertUnknownOutcome();
```

---

# 215. No leak assertion

```php
TransactionTelemetry::assertNoOpenTransactions();
```

---

# 216. Connection affinity assertion

```php
TransactionTelemetry::assertSinglePhysicalConnection();
```

---

# 217. Nested transaction testing

Deberán probarse:

```text
JOIN
SAVEPOINT
REJECT
REQUIRES_NEW
```

según soporte.

---

# 218. Retry testing

Casos:

```text
deadlock → retry → success
serialization failure → retry → success
retry exhausted
non-retryable failure
UNKNOWN commit → no retry
```

---

# 219. Commit failure injection

Testing deberá poder simular:

```text
before commit sent
after commit sent
before confirmation
after database commit
```

cuando el fake driver lo permita.

---

# 220. UNKNOWN outcome testing

Será obligatorio comprobar que:

```text
commit possibly executed
+
connection lost
```

no se convierte en:

```text
FAILED
```

---

# 221. Persistent runtime testing

Caso obligatorio:

```text
Request A
BEGIN
COMMIT

Request B
BEGIN
ROLLBACK
```

y comprobar:

```text
TransactionContext A
≠
TransactionContext B
```

---

# 222. Coroutine testing

```text
Coroutine A → Transaction A
Coroutine B → Transaction B
```

sin intercambio de:

```text
TransactionId
ConnectionId
TenantId
ShardId
TelemetryScope
```

---

# 223. Directory structure

```text
src/Quantum/Database/Telemetry/Transaction/
│
├── Contract/
│   ├── TransactionTelemetryInstrumentation.php
│   ├── TransactionTelemetryScope.php
│   ├── TransactionAttemptTelemetryScope.php
│   └── TransactionLeakDetector.php
│
├── Identity/
│   ├── TransactionOperationId.php
│   ├── TransactionId.php
│   ├── TransactionAttemptId.php
│   ├── TransactionScopeId.php
│   └── SavepointId.php
│
├── Context/
│   ├── TransactionTelemetryContext.php
│   ├── TransactionTelemetryContextFactory.php
│   └── TransactionTelemetryContextResolver.php
│
├── Lifecycle/
│   ├── TransactionTelemetryState.php
│   ├── TransactionOutcome.php
│   ├── TransactionPurpose.php
│   ├── TransactionOwnership.php
│   └── TransactionLifecycleTracker.php
│
├── Isolation/
│   ├── TransactionIsolationTelemetry.php
│   └── IsolationResolutionOutcome.php
│
├── Attempt/
│   ├── TransactionAttemptTelemetryRecord.php
│   ├── TransactionAttemptTracker.php
│   └── TransactionAttemptSummary.php
│
├── Scope/
│   ├── TransactionScopeTelemetryRecord.php
│   ├── TransactionScopeOutcome.php
│   └── NestedTransactionTelemetry.php
│
├── Savepoint/
│   ├── SavepointTelemetryRecord.php
│   ├── SavepointOutcome.php
│   └── SavepointTelemetryTracker.php
│
├── Retry/
│   ├── TransactionRetryTelemetry.php
│   ├── TransactionRetryReason.php
│   └── TransactionRetryRecord.php
│
├── Commit/
│   ├── TransactionCommitTelemetry.php
│   ├── TransactionCommitFailureCategory.php
│   └── TransactionCommitRecord.php
│
├── Rollback/
│   ├── TransactionRollbackTelemetry.php
│   ├── TransactionRollbackOutcome.php
│   └── TransactionRollbackRecord.php
│
├── Timing/
│   ├── TransactionTelemetryPhase.php
│   ├── TransactionTiming.php
│   └── TransactionIdleTimeTracker.php
│
├── Failure/
│   ├── TransactionFailureCategory.php
│   ├── TransactionTimeoutCategory.php
│   └── TransactionFailureRecord.php
│
├── Leak/
│   ├── ActiveTransactionRegistry.php
│   ├── TransactionLeakAssessment.php
│   └── DefaultTransactionLeakDetector.php
│
├── Record/
│   ├── TransactionTelemetrySummary.php
│   └── TransactionTelemetryCoverage.php
│
├── Metric/
│   ├── TransactionMetricRecorder.php
│   ├── TransactionMetricRegistry.php
│   └── TransactionMetricDescriptor.php
│
├── Trace/
│   ├── TransactionSpanFactory.php
│   ├── TransactionSpanEnricher.php
│   └── TransactionTracePolicy.php
│
├── Diagnostics/
│   ├── TransactionTelemetryInspector.php
│   ├── TransactionTelemetryExplainer.php
│   └── TransactionDiagnosticBuffer.php
│
├── Policy/
│   ├── TransactionTelemetryPolicy.php
│   ├── CompiledTransactionTelemetryPolicy.php
│   └── TransactionTelemetryBudget.php
│
├── Testing/
│   ├── RecordingTransactionTelemetry.php
│   ├── TransactionTelemetryAssertions.php
│   └── FakeTransactionTelemetryClock.php
│
└── Exception/
    ├── TransactionTelemetryException.php
    ├── TransactionTelemetryStateException.php
    ├── TransactionTelemetrySecurityException.php
    └── TransactionTelemetryBudgetException.php
```

---

# 224. Architectural invariants

## DB-TTEL-001
Transaction Telemetry observará transacciones sin controlarlas.

## DB-TTEL-002
Transaction Telemetry no ejecutará `begin()`.

## DB-TTEL-003
Transaction Telemetry no ejecutará `commit()`.

## DB-TTEL-004
Transaction Telemetry no ejecutará `rollback()`.

## DB-TTEL-005
Transaction Telemetry no decidirá retries.

## DB-TTEL-006
Transaction Telemetry no creará savepoints.

## DB-TTEL-007
Transaction Telemetry no cambiará isolation levels.

## DB-TTEL-008
Transaction Operation será distinta de Transaction Attempt.

## DB-TTEL-009
Logical Transaction será distinta de Physical Transaction.

## DB-TTEL-010
Transaction Scope será distinto de Physical Transaction.

## DB-TTEL-011
Nested Transaction no implicará una nueva transacción física.

## DB-TTEL-012
Savepoint será distinto de Transaction.

## DB-TTEL-013
Savepoint será distinto de Nested Scope.

## DB-TTEL-014
Transaction será distinta de UnitOfWork.

## DB-TTEL-015
Transaction será distinta de EntityManager.

## DB-TTEL-016
Flush será distinto de Commit.

## DB-TTEL-017
Statement success será distinto de transaction success.

## DB-TTEL-018
Query failure será distinta de transaction failure.

## DB-TTEL-019
Rollback será distinto de object graph rewind.

## DB-TTEL-020
Commit sent será distinto de commit confirmed.

## DB-TTEL-021
UNKNOWN será un resultado válido.

## DB-TTEL-022
UNKNOWN no será convertido a FAILED.

## DB-TTEL-023
UNKNOWN no será convertido a COMMITTED.

## DB-TTEL-024
Commit posiblemente ejecutado no será reintentado ciegamente.

## DB-TTEL-025
Cada retry attempt tendrá identidad propia.

## DB-TTEL-026
Operation outcome será distinto de attempt outcome.

## DB-TTEL-027
Un deadlock de un attempt no implicará failure final.

## DB-TTEL-028
Retry count será observable.

## DB-TTEL-029
Retry reason será observable.

## DB-TTEL-030
Retry decision pertenecerá al Retry System.

## DB-TTEL-031
Retry delay será observable.

## DB-TTEL-032
Requested isolation será distinto de effective isolation.

## DB-TTEL-033
Isolation desconocido permanecerá UNKNOWN.

## DB-TTEL-034
No se inferirá effective isolation desde requested isolation.

## DB-TTEL-035
Read-only transaction será distinta de no-writes-observed.

## DB-TTEL-036
Transaction nesting depth será observable.

## DB-TTEL-037
Nesting depth no representará physical transaction count.

## DB-TTEL-038
Nested strategy será observable.

## DB-TTEL-039
Scope success no implicará root commit.

## DB-TTEL-040
Savepoint creation será observable.

## DB-TTEL-041
Savepoint release será observable.

## DB-TTEL-042
Rollback-to-savepoint será observable.

## DB-TTEL-043
Rollback-to-savepoint no será transaction rollback.

## DB-TTEL-044
Transaction duration será distinta de query duration.

## DB-TTEL-045
Transaction operation duration será distinta de final attempt duration.

## DB-TTEL-046
Begin duration será observable.

## DB-TTEL-047
Commit duration será observable.

## DB-TTEL-048
Rollback duration será observable.

## DB-TTEL-049
Idle-in-transaction será observable cuando exista cobertura suficiente.

## DB-TTEL-050
Missing idle observation no será 0ms.

## DB-TTEL-051
Time-to-first-statement será observable.

## DB-TTEL-052
Long transaction no será failure automáticamente.

## DB-TTEL-053
Long transaction threshold podrá depender del purpose.

## DB-TTEL-054
Transaction purpose será bounded.

## DB-TTEL-055
Transaction ownership será observable.

## DB-TTEL-056
Ownership no otorgará control a telemetry.

## DB-TTEL-057
Transaction connection affinity será observable.

## DB-TTEL-058
Connection switching indebido será diagnosticable.

## DB-TTEL-059
Affinity enforcement pertenecerá al Transaction/Connection System.

## DB-TTEL-060
TransactionAttemptId no será metric label.

## DB-TTEL-061
TransactionId no será metric label.

## DB-TTEL-062
TransactionOperationId no será metric label.

## DB-TTEL-063
ConnectionId no será metric label.

## DB-TTEL-064
QueryId no será metric label.

## DB-TTEL-065
SavepointId no será metric label.

## DB-TTEL-066
TenantId no será metric label por default.

## DB-TTEL-067
ShardId no será metric label por default.

## DB-TTEL-068
Raw SQL no será transaction metric label.

## DB-TTEL-069
Query parameters no serán copiados al transaction record.

## DB-TTEL-070
Query duration tendrá Query Telemetry como canonical owner.

## DB-TTEL-071
Connection acquisition tendrá Connection Telemetry como canonical owner.

## DB-TTEL-072
Transaction duration tendrá Transaction Telemetry como canonical owner.

## DB-TTEL-073
No existirán mediciones duplicadas incompatibles.

## DB-TTEL-074
Queries per transaction serán correlacionables.

## DB-TTEL-075
Query count será distinto de statement count.

## DB-TTEL-076
Query time ratio será diagnóstico, no conclusión automática.

## DB-TTEL-077
Deadlock será observable.

## DB-TTEL-078
Deadlock details estarán sujetos a security policy.

## DB-TTEL-079
Lock wait será distinto de application idle.

## DB-TTEL-080
Pessimistic lock usage podrá correlacionarse.

## DB-TTEL-081
Optimistic conflict permanecerá principalmente ORM/Persistence telemetry.

## DB-TTEL-082
Commit phases serán observadas según capacidades.

## DB-TTEL-083
Unsupported commit phases no serán inventadas.

## DB-TTEL-084
Telemetry coverage será explícita.

## DB-TTEL-085
Commit connection loss podrá producir UNKNOWN.

## DB-TTEL-086
Rollback outcome será explícito.

## DB-TTEL-087
Rollback failure será observable.

## DB-TTEL-088
Rollback DB exitoso no implicará rewind de objetos.

## DB-TTEL-089
ORM Flush podrá correlacionarse con Transaction.

## DB-TTEL-090
Flush no será reportado como commit.

## DB-TTEL-091
Transaction Event será distinto de Telemetry Signal.

## DB-TTEL-092
afterCommit failure no cambiará COMMITTED a FAILED.

## DB-TTEL-093
External side effects serán distintos del transaction outcome.

## DB-TTEL-094
Outbox correlation será posible.

## DB-TTEL-095
Telemetry no implementará Outbox.

## DB-TTEL-096
Transaction leak podrá detectarse como candidate.

## DB-TTEL-097
Long transaction no será automáticamente leak.

## DB-TTEL-098
Leak confidence será explícita.

## DB-TTEL-099
Telemetry no hará rollback automático de leaks.

## DB-TTEL-100
Active transaction registry será bounded.

## DB-TTEL-101
Registry overflow será observable.

## DB-TTEL-102
Telemetry no agotará memoria observando transacciones.

## DB-TTEL-103
Telemetry budget será distinto de execution resource budget.

## DB-TTEL-104
Resource Governance conservará autoridad.

## DB-TTEL-105
Cancellation será distinta de rollback.

## DB-TTEL-106
Cancellation no implicará rollback confirmado.

## DB-TTEL-107
Transaction timeout será distinto de query timeout.

## DB-TTEL-108
Lock timeout será distinto de connection timeout.

## DB-TTEL-109
Timeout category será explícita.

## DB-TTEL-110
Transaction metrics usarán dimensiones bounded.

## DB-TTEL-111
Transaction tracing será sampleable.

## DB-TTEL-112
Savepoints no generarán spans por default.

## DB-TTEL-113
Statement events no duplicarán Query Telemetry.

## DB-TTEL-114
Transaction summary podrá ser immutable.

## DB-TTEL-115
Attempt record podrá ser immutable.

## DB-TTEL-116
Observation coverage será preservada.

## DB-TTEL-117
Missing observation no será equivalente a zero.

## DB-TTEL-118
VoltStack no fingirá distributed ACID.

## DB-TTEL-119
Cross-shard application operation no será automáticamente global transaction.

## DB-TTEL-120
Cada shard transaction conservará outcome propio.

## DB-TTEL-121
Partial distributed outcome será preservado.

## DB-TTEL-122
Telemetry no inferirá 2PC.

## DB-TTEL-123
Tenant context podrá correlacionarse.

## DB-TTEL-124
Tenant context drift será diagnosticable.

## DB-TTEL-125
Shard affinity será observable.

## DB-TTEL-126
Shard drift será diagnosticable.

## DB-TTEL-127
Routing authority permanecerá fuera de telemetry.

## DB-TTEL-128
Replica read-only transaction será representable.

## DB-TTEL-129
Replica eligibility no será decidida por telemetry.

## DB-TTEL-130
Failover mid-transaction no será fingido como transparente.

## DB-TTEL-131
Connection loss before commit será distinguible de during commit.

## DB-TTEL-132
Persistent workers tendrán transaction scopes aislados.

## DB-TTEL-133
Transaction state no sobrevivirá accidentalmente al request.

## DB-TTEL-134
RoadRunner tendrá contexto fresco por job.

## DB-TTEL-135
OpenSwoole tendrá contexto coroutine-safe.

## DB-TTEL-136
No existirá static mutable current transaction.

## DB-TTEL-137
Immutable telemetry metadata podrá compartirse.

## DB-TTEL-138
Mutable transaction telemetry state será scoped.

## DB-TTEL-139
Security minimization será obligatoria.

## DB-TTEL-140
Transaction SQL capture no duplicará Query Telemetry.

## DB-TTEL-141
Transaction names dinámicos no serán metric labels.

## DB-TTEL-142
Diagnostic output separará observation de inference.

## DB-TTEL-143
Diagnostic recommendations no se presentarán como hechos.

## DB-TTEL-144
Transaction instrumentation será provider-agnostic.

## DB-TTEL-145
Hot path no parseará SQL.

## DB-TTEL-146
Hot path no capturará stack traces por default.

## DB-TTEL-147
Stack traces diagnósticos estarán sujetos a budget.

## DB-TTEL-148
Telemetry configuration será compilable.

## DB-TTEL-149
Compiled policy será immutable.

## DB-TTEL-150
Successful transactions podrán samplearse.

## DB-TTEL-151
Failures podrán usar mayor sampling.

## DB-TTEL-152
UNKNOWN outcomes tendrán alta prioridad diagnóstica.

## DB-TTEL-153
Long transactions podrán promoverse a diagnostics.

## DB-TTEL-154
Exporter failure no provocará rollback.

## DB-TTEL-155
Exporter backpressure no bloqueará indefinidamente commit.

## DB-TTEL-156
Unsafe telemetry configuration podrá rechazarse en bootstrap.

## DB-TTEL-157
Testing podrá verificar commits.

## DB-TTEL-158
Testing podrá verificar rollbacks.

## DB-TTEL-159
Testing podrá verificar query count.

## DB-TTEL-160
Testing podrá verificar isolation.

## DB-TTEL-161
Testing podrá verificar retries.

## DB-TTEL-162
Testing podrá verificar deadlocks.

## DB-TTEL-163
Testing podrá verificar UNKNOWN.

## DB-TTEL-164
Testing podrá detectar leaked transactions.

## DB-TTEL-165
Testing podrá verificar connection affinity.

## DB-TTEL-166
Commit failure injection será soportada.

## DB-TTEL-167
UNKNOWN outcome tendrá pruebas obligatorias.

## DB-TTEL-168
Persistent runtime isolation tendrá pruebas obligatorias.

## DB-TTEL-169
Coroutine isolation tendrá pruebas obligatorias.

## DB-TTEL-170
Transaction Telemetry nunca sustituirá Transaction Manager.

## DB-TTEL-171
Transaction Telemetry nunca sustituirá Transaction Context.

## DB-TTEL-172
Transaction Telemetry nunca sustituirá Retry System.

## DB-TTEL-173
Transaction Telemetry nunca sustituirá Deadlock Handling.

## DB-TTEL-174
Transaction Telemetry nunca sustituirá Connection Manager.

## DB-TTEL-175
Transaction Telemetry nunca sustituirá Resource Governance.

## DB-TTEL-176
Transaction Telemetry nunca sustituirá ORM UnitOfWork.

## DB-TTEL-177
Transaction Telemetry nunca será fuente de Database Truth.

## DB-TTEL-178
Telemetry preservará evidencia incierta.

## DB-TTEL-179
Telemetry no inventará rollback exitoso.

## DB-TTEL-180
Telemetry no inventará commit exitoso.

---

# 225. Modelo formal de transacción

Sea:

```text
O
```

una operación transaccional lógica.

Puede tener intentos:

```text
A(O) = {A1, A2, ..., An}
```

Cada intento puede producir:

```text
Outcome(Ai)
∈
{
    COMMITTED,
    ROLLED_BACK,
    FAILED,
    CANCELLED,
    UNKNOWN
}
```

---

# 226. Resultado de operación

Para una secuencia segura de retry:

```text
A1 = ROLLED_BACK_RETRYABLE
A2 = ROLLED_BACK_RETRYABLE
A3 = COMMITTED
```

podemos representar:

```text
Outcome(O)
=
COMMITTED_AFTER_RETRY
```

sin perder los outcomes individuales.

---

# 227. UNKNOWN formal

Si:

```text
CommitSent(Ai) = true
```

y:

```text
CommitConfirmation(Ai) = unavailable
```

entonces:

```text
Outcome(Ai) = UNKNOWN
```

si no existe evidencia adicional suficiente.

---

# 228. Regla de retry formal

En términos generales:

```text
RetryAllowed(Ai)
=
RetryableFailure(Ai)
∧
KnownNotCommitted(Ai)
∧
ReplaySafe(O)
∧
PolicyAllowsRetry
```

Por tanto:

```text
Outcome(Ai) = UNKNOWN
⇒
RetryAllowed(Ai) = false
```

por default.

---

# 229. Modelo temporal

Para un intento:

```text
T_attempt
=
T_begin
+
T_active
+
T_commit_or_rollback
```

donde:

```text
T_active
=
T_query
+
T_lock_wait
+
T_application_idle
+
T_other
```

según cobertura.

---

# 230. Coverage caveat

No todas estas fases serán siempre observables independientemente.

Por ello:

```text
Σ observed phases
```

no deberá asumirse siempre igual a:

```text
T_attempt
```

---

# 231. Modelo de nesting

Para scopes:

```text
S0
├── S1
│   └── S2
└── S3
```

podemos definir:

```text
depth(S0)=0
depth(S1)=1
depth(S2)=2
depth(S3)=1
```

sin implicar:

```text
PhysicalTransactions = 4
```

---

# 232. Modelo de conexión

Para un intento físico normal:

```text
∀ q ∈ Queries(A):
PhysicalConnection(q)
=
PhysicalConnection(A)
```

salvo arquitectura explícita que permita otra semántica.

---

# 233. Modelo de shard

Para una transacción single-shard:

```text
∀ q ∈ Queries(A):
Shard(q)
=
Shard(A)
```

---

# 234. Modelo de observabilidad

La telemetría deberá construir:

```text
TransactionObservation
=
Identity
+
Timing
+
Isolation
+
ConnectionCorrelation
+
QueryCorrelation
+
Nesting
+
Savepoints
+
RetryEvidence
+
FailureEvidence
+
Outcome
+
Coverage
```

---

# 235. Arquitectura de interacción

```text
                    Application
                        │
                        ▼
                Transaction Manager
                        │
                        ▼
                Transaction Operation
                        │
              ┌─────────┴─────────┐
              │                   │
              ▼                   ▼
        Attempt #1             Attempt #2
              │                   │
              ▼                   ▼
       Connection Lease     Connection Lease
              │                   │
              ▼                   ▼
          BEGIN                 BEGIN
              │                   │
       ┌──────┼──────┐      ┌────┼────┐
       ▼      ▼      ▼      ▼    ▼    ▼
     Query  Query Savepoint Query Query Query
       │      │      │      │    │    │
       └──────┼──────┘      └────┼────┘
              ▼                   ▼
           DEADLOCK             COMMIT
              │                   │
              ▼                   ▼
          ROLLBACK             COMMITTED
              │
              ▼
            RETRY
```

Transaction Telemetry observa:

```text
Operation
Attempts
Connection Affinity
BEGIN
Queries
Idle Time
Savepoints
Deadlock
Rollback
Retry
COMMIT
Final Outcome
```

sin controlar ninguno.

---

# 236. Correlation graph

```text
Request / Job
      │
      ▼
TransactionOperationId
      │
      ├───────────────┐
      ▼               ▼
Attempt 1          Attempt 2
      │               │
      ▼               ▼
TransactionId     TransactionId
      │               │
      ▼               ▼
ConnectionLease  ConnectionLease
      │               │
      ▼               ▼
PhysicalConn     PhysicalConn
      │               │
      ├── Query        ├── Query
      ├── Query        ├── Query
      └── Savepoint    └── Query
```

Esto permitirá reconstruir una operación sin convertir IDs de alta cardinalidad en dimensiones métricas.

---

# 237. Filosofía arquitectónica

VoltStack seguirá:

```text
Explicit transaction identity
over
implicit transaction state

Operation + attempts
over
single flattened transaction record

Logical vs physical distinction
over
nested transaction ambiguity

UNKNOWN
over
false certainty

Requested vs effective isolation
over
assumption

Connection affinity
over
hidden session switching

Savepoint semantics
over
fake nested transactions

Retry evidence
over
erasing failed attempts

Canonical timing ownership
over
duplicate measurements

Bounded metrics
over
per-transaction metric series

Persistent runtime isolation
over
request-lifetime assumptions

Observation
over
control
```

---

# 238. Regla maestra

> **VoltStack deberá poder reconstruir y explicar la historia transaccional completa —incluyendo intentos, conexión, isolation, queries, nesting, savepoints, deadlocks, retries, commit, rollback y resultados inciertos— sin permitir que la capa de telemetría adquiera autoridad sobre la transacción.**

En forma compacta:

```text
Transaction Request
↓
Resolve
↓
Acquire Connection
↓
BEGIN
↓
Observe Activity
↓
Nested Scopes / Savepoints
↓
Commit / Rollback
↓
Preserve Outcome
↓
Correlate
↓
Diagnose
```

Nunca:

```text
Telemetry
↓
Begin / Commit / Rollback / Retry
```

---

# 239. Estado del Bloque 21

```text
BLOCK 21 — TELEMETRY AND DEBUGGING

✓ 216_DATABASE_TELEMETRY_ARCHITECTURE.md
✓ 217_DATABASE_QUERY_TELEMETRY_SYSTEM.md
✓ 218_DATABASE_CONNECTION_TELEMETRY_SYSTEM.md
✓ 219_DATABASE_TRANSACTION_TELEMETRY_SYSTEM.md
○ 220_DATABASE_ORM_TELEMETRY_SYSTEM.md
○ 221_DATABASE_QUERY_PROFILER_SYSTEM.md
○ 222_DATABASE_SLOW_QUERY_DETECTION_SYSTEM.md
○ 223_DATABASE_N_PLUS_ONE_TELEMETRY_SYSTEM.md
○ 224_DATABASE_DEBUG_INFORMATION_SYSTEM.md
○ 225_DATABASE_DEVELOPER_DEBUG_TOOLBAR_INTEGRATION.md
```

---

# 240. Siguiente documento

```text
220_DATABASE_ORM_TELEMETRY_SYSTEM.md
```

El siguiente documento deberá llevar la observabilidad desde la frontera transaccional hacia el ORM:

```text
ORM Operation
↓
EntityManager
↓
IdentityMap
↓
UnitOfWork
↓
Change Tracking
↓
Persistence Planning
↓
Flush
↓
Hydration
↓
Relationship Loading
↓
Query Engine
↓
Transaction
```

incluyendo:

```text
ORMOperationId
EntityManager scope
managed entity count
IdentityMap size
IdentityMap hit/miss
UnitOfWork state
NEW/MANAGED/DIRTY/REMOVED counts
change-set computation
flush planning
flush duration
entities inserted
entities updated
entities deleted
batch persistence
hydration duration
entities hydrated
IdentityMap reuse
partial entities
relationship loading
eager loading
lazy loading
N+1 correlation
repository operations
Model API operations
Active Record/Data Mapper convergence
optimistic lock conflicts
persistence consistency
TAINTED EntityManager
memory growth
persistent runtime isolation
telemetry cardinality
sampling
security
diagnostics
testing
```

bajo la regla:

> **ORM Telemetry deberá explicar el costo y comportamiento del ORM sin convertir entidades, IdentityMap, UnitOfWork, Hydrator o EntityManager en objetos de telemetría ni alterar sus semánticas de persistencia.**