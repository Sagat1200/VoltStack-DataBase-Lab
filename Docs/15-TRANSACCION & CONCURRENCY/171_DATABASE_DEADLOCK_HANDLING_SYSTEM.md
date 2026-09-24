# 171_DATABASE_DEADLOCK_HANDLING_SYSTEM.md

# VoltStack Quantum Database
## Database Deadlock Handling System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 171 — Database Deadlock Handling System  
**Bloque:** 15 — Transactions & Concurrency  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `170_DATABASE_TRANSACTION_RETRY_SYSTEM.md`  
**Siguiente documento:** `172_DATABASE_OPTIMISTIC_LOCKING_SYSTEM.md`

---

# 1. Propósito

`Database Deadlock Handling System` define cómo VoltStack identificará, normalizará, clasificará y propagará los deadlocks detectados por los motores de base de datos.

Un deadlock ocurre conceptualmente cuando varias transacciones mantienen recursos que otras necesitan y forman una dependencia circular.

Ejemplo:

```text
Transaction A
    owns Resource X
    waits Resource Y

Transaction B
    owns Resource Y
    waits Resource X
```

Grafo:

```text
TX-A ─────waits────► TX-B
 ▲                    │
 │                    │
 └──────waits─────────┘
```

El sistema gestor deberá romper el ciclo.

Normalmente seleccionará una transacción como víctima.

Pero VoltStack no deberá confundir:

```text
Deadlock Detected
```

con:

```text
Retry Automatically
```

ni:

```text
Deadlock
=
Lock Timeout
```

ni:

```text
Deadlock
=
Serialization Failure
```

La regla central será:

> **VoltStack tratará un deadlock como una condición de concurrencia clasificada y contextual: primero determinará qué ocurrió y qué efecto tuvo sobre la transacción; solo después una política superior decidirá si existe un boundary seguro que pueda reintentarse.**

---

# 2. Objetivos

El sistema deberá proporcionar:

1. clasificación canónica de deadlocks;
2. normalización de errores de plataforma;
3. separación entre deadlock y lock timeout;
4. separación entre deadlock y serialization failure;
5. representación del efecto sobre la transacción;
6. deadlock victim semantics;
7. integración con `TransactionContext`;
8. integración con `TransactionManager`;
9. integración con `TransactionRetrySystem`;
10. rollback coordination;
11. manejo de outcome uncertainty;
12. ORM consistency awareness;
13. query correlation;
14. transaction correlation;
15. diagnostics;
16. telemetry;
17. tracing;
18. logging estructurado;
19. resource governance;
20. testing de conformidad por plataforma;
21. persistent-runtime safety.

---

# 3. No objetivos

El sistema no deberá:

- implementar el detector interno de deadlocks del DBMS;
- construir necesariamente el wait-for graph real del servidor;
- matar sesiones arbitrariamente;
- reintentar automáticamente transacciones;
- implementar optimistic locking;
- implementar pessimistic locking;
- implementar transaction isolation;
- ejecutar rollback sin coordinación con Transaction Manager;
- interpretar deadlocks como errores de negocio;
- garantizar ausencia de deadlocks;
- implementar distributed deadlock detection;
- realizar parsing frágil de mensajes desde el core.

---

# 4. Distinciones fundamentales

```text
Deadlock
≠
Lock Wait
≠
Lock Timeout
≠
Serialization Failure
≠
Optimistic Lock Conflict
≠
Constraint Violation
≠
Connection Failure
≠
Transaction Retry
```

---

# 5. Deadlock conceptual

Considérese:

```text
TX-A:

UPDATE accounts
SET balance = ...
WHERE id = 1;

-- owns lock Account 1

UPDATE accounts
SET balance = ...
WHERE id = 2;

-- waits Account 2
```

Simultáneamente:

```text
TX-B:

UPDATE accounts
SET balance = ...
WHERE id = 2;

-- owns lock Account 2

UPDATE accounts
SET balance = ...
WHERE id = 1;

-- waits Account 1
```

Resultado:

```text
TX-A owns A1
TX-A waits A2

TX-B owns A2
TX-B waits A1
```

Grafo:

```text
TX-A ─────► TX-B
  ▲           │
  │           ▼
  └───────────┘
```

Existe ciclo.

---

# 6. Wait-for graph

Formalmente:

```text
G = (T, E)
```

donde:

```text
T = set of transactions
```

y:

```text
Ti → Tj
```

significa:

> `Ti` espera un recurso actualmente bloqueado de forma incompatible por `Tj`.

Existe deadlock si:

```text
G contains a cycle
```

---

# 7. VoltStack no construye necesariamente G

La mayoría de DBMS ya implementan detección de deadlocks.

Por tanto VoltStack normalmente consumirá:

```text
Database Server
    ↓
Driver Error
    ↓
Platform Error Adapter
    ↓
Canonical Deadlock Classification
```

---

# 8. Deadlock victim

Cuando el DBMS detecta un ciclo, selecciona una víctima.

Conceptualmente:

```text
Deadlock Cycle
      ↓
Victim Selection
      ↓
Abort / Rollback Victim
      ↓
Release Locks
      ↓
Other Transaction Continues
```

---

# 9. Victim selection is DB responsibility

VoltStack no deberá asumir que controla qué transacción será seleccionada.

Por tanto:

```text
VictimSelection
≠
VoltStackRetryPolicy
```

---

# 10. Deadlock classification

Tipo canónico:

```php
enum ConcurrencyFailureType
{
    case DEADLOCK;
    case LOCK_TIMEOUT;
    case SERIALIZATION_FAILURE;
    case OPTIMISTIC_LOCK_CONFLICT;
    case LOCK_NOT_AVAILABLE;
    case UNKNOWN;
}
```

---

# 11. DeadlockClassification

```php
final readonly class DeadlockClassification
{
    public function __construct(
        public DeadlockConfidence $confidence,
        public TransactionImpact $transactionImpact,
        public TransactionOutcomeCertainty $certainty,
        public RetryRecommendation $retryRecommendation,
        public PlatformErrorIdentity $platformError,
    ) {}
}
```

---

# 12. Confidence

```php
enum DeadlockConfidence
{
    case CERTAIN;
    case HIGH;
    case MEDIUM;
    case LOW;
    case UNKNOWN;
}
```

---

# 13. CERTAIN

Se utilizará cuando exista señal inequívoca del driver/plataforma:

```text
canonical SQLSTATE
```

o:

```text
vendor error code
```

documentado como deadlock.

---

# 14. Message matching

Matching de texto podrá existir como fallback del adapter específico de plataforma.

No deberá ser la estrategia principal del core.

---

# 15. Central error normalization

Arquitectura:

```text
PDO / Driver Exception
          ↓
Driver Error Extractor
          ↓
Platform Error Normalizer
          ↓
DatabaseFailureDescriptor
          ↓
Concurrency Failure Classifier
          ↓
DeadlockClassification
```

---

# 16. DatabaseFailureDescriptor

```php
final readonly class DatabaseFailureDescriptor
{
    public function __construct(
        public ?string $sqlState,
        public ?int $vendorCode,
        public FailureCategory $category,
        public TransactionFailurePhase $phase,
        public TransactionImpact $transactionImpact,
        public TransactionOutcomeCertainty $certainty,
    ) {}
}
```

---

# 17. Vendor information remains available

La normalización no deberá eliminar información útil.

Podrá conservar:

```text
SQLSTATE
vendor code
driver error identity
platform
server version
```

para diagnostics.

Pero las capas superiores operarán con categorías canónicas.

---

# 18. Deadlock vs lock timeout

Deadlock:

```text
cycle exists
    ↓
DB detects cycle
    ↓
victim selected
```

Lock timeout:

```text
TX waits lock
    ↓
wait exceeds configured threshold
    ↓
timeout
```

No requiere ciclo.

---

# 19. Lock timeout example

```text
TX-A owns Resource X

TX-B waits Resource X
```

No existe necesariamente:

```text
TX-A waits something from TX-B
```

Por tanto:

```text
LockTimeout
≠
Deadlock
```

---

# 20. Deadlock vs serialization failure

Una serialization failure puede ocurrir debido a mecanismos de control de concurrencia que detectan que una ejecución concurrente no puede serializarse correctamente.

No implica necesariamente un ciclo de locks clásico.

Por tanto:

```text
SerializationFailure
≠
Deadlock
```

aunque ambos frecuentemente puedan resolverse mediante:

```text
full transaction retry
```

---

# 21. Deadlock vs optimistic conflict

Optimistic locking suele detectar:

```text
expected version = 5
actual version = 6
```

Esto es un conflicto lógico de versión.

No es un deadlock.

```text
OptimisticConflict
≠
Deadlock
```

---

# 22. Transaction impact

El sistema deberá modelar explícitamente qué efecto tuvo el error.

```php
enum TransactionImpact
{
    case NONE;
    case STATEMENT_FAILED;
    case TRANSACTION_ROLLBACK_ONLY;
    case TRANSACTION_ABORTED;
    case CONNECTION_INVALIDATED;
    case UNKNOWN;
}
```

---

# 23. Why transaction impact matters

Un error clasificado:

```text
DEADLOCK
```

no es información suficiente.

También necesitamos saber:

```text
what happened to the transaction?
```

---

# 24. Canonical result

Ejemplo:

```text
Failure:
    DEADLOCK

Impact:
    TRANSACTION_ABORTED

Certainty:
    CERTAIN

Retry Recommendation:
    RETRY_TRANSACTION
```

---

# 25. Platform-specific semantics

Cada plataforma deberá proporcionar:

```php
interface ConcurrencyErrorProfile
{
    public function classify(
        PlatformError $error,
        TransactionRuntimeState $transaction,
    ): ConcurrencyFailureClassification;
}
```

---

# 26. MySQL profile

`MySQLPlatform` tendrá:

```text
MySqlConcurrencyErrorProfile
```

responsable de mapear las señales específicas del servidor hacia categorías canónicas.

---

# 27. MariaDB profile

`MariaDbConcurrencyErrorProfile` será independiente.

Nunca:

```text
MariaDB error semantics
=
MySQL error semantics by assumption
```

---

# 28. PostgreSQL profile

PostgreSQL tendrá su propio profile de:

```text
deadlocks
serialization failures
lock availability
transaction-aborted states
```

---

# 29. SQLite profile

SQLite requiere especial cuidado.

Muchos conflictos pueden manifestarse como:

```text
database busy
database locked
```

pero:

```text
BUSY/LOCKED
≠
automatically classic deadlock
```

VoltStack no deberá etiquetar todo lock conflict de SQLite como:

```text
DEADLOCK
```

---

# 30. Capability-driven normalization

Nunca:

```php
if ($driver === 'sqlite') {
    return DEADLOCK;
}
```

El profile deberá distinguir lo que realmente puede afirmarse.

---

# 31. Error map architecture

```text
Platform
   │
   ├── SQLSTATE Map
   ├── Vendor Code Map
   ├── Transaction Semantics
   └── Error Evidence Rules
            ↓
     Canonical Classification
```

---

# 32. SQLSTATE

VoltStack podrá utilizar SQLSTATE como señal importante.

Pero:

```text
SQLSTATE alone
```

no siempre contiene toda la semántica necesaria.

Podrá complementarse con:

```text
vendor code
driver metadata
failure phase
transaction state
server capabilities
```

---

# 33. No application string parsing

El código de aplicación no debería necesitar:

```php
if (str_contains(
    $e->getMessage(),
    'Deadlock found'
)) {
    // ...
}
```

VoltStack deberá proporcionar una excepción canónica.

---

# 34. DeadlockDetectedException

```php
final class DeadlockDetectedException
    extends DatabaseConcurrencyException
{
    public function classification(): DeadlockClassification;

    public function transactionImpact(): TransactionImpact;

    public function isRetryCandidate(): bool;
}
```

---

# 35. Exception ≠ retry command

Aunque:

```php
$e->isRetryCandidate() === true
```

eso significa:

```text
failure type can potentially participate in retry
```

No:

```text
retry is definitely safe
```

---

# 36. Retry recommendation

```php
enum RetryRecommendation
{
    case NONE;
    case RETRY_STATEMENT_IF_PROVEN_SAFE;
    case RETRY_SAVEPOINT_SCOPE_IF_PROVEN_SAFE;
    case RETRY_TRANSACTION;
    case RECONCILE_REQUIRED;
    case UNKNOWN;
}
```

---

# 37. Default deadlock recommendation

Para deadlocks que abortan la transacción:

```text
RETRY_TRANSACTION
```

será una recomendación frecuente.

Pero el Retry System seguirá evaluando:

```text
BoundaryOwned
OutcomeSafe
PolicyAllows
BudgetAvailable
ContextRecoverable
SideEffectsSafe
```

---

# 38. Retry ownership

El Deadlock System nunca hará:

```php
return $transaction->retry();
```

por sí mismo.

Arquitectura:

```text
Deadlock Handling
      ↓
Classification
      ↓
Recommendation
      ↓
Transaction Retry System
      ↓
Policy Decision
```

---

# 39. Deadlock handling pipeline

```text
Driver throws error
       ↓
Extract structured error
       ↓
Platform normalization
       ↓
Concurrency classifier
       ↓
DEADLOCK
       ↓
Determine transaction impact
       ↓
Update TransactionContext
       ↓
Create canonical exception
       ↓
Notify telemetry
       ↓
Propagate to Retry System
```

---

# 40. TransactionContext integration

Ante deadlock, `TransactionContext` deberá actualizarse según la semántica confirmada.

Ejemplo:

```text
ACTIVE
   ↓
deadlock
   ↓
ABORTED
```

si el servidor ha abortado toda la transacción.

---

# 41. Rollback-only state

Si la plataforma deja la transacción técnicamente existente pero no segura para commit:

```text
ACTIVE
   ↓
DEADLOCK
   ↓
MARKED_ROLLBACK_ONLY
```

---

# 42. Transaction aborted

Si existe evidencia de rollback/abort completo:

```text
TransactionState:
    ABORTED
```

---

# 43. Unknown transaction impact

Si VoltStack no puede determinarlo:

```text
TransactionImpact::UNKNOWN
```

y el contexto podrá quedar:

```text
TAINTED
```

---

# 44. No fake recovery

Prohibido:

```text
deadlock exception caught
    ↓
transaction state reset to ACTIVE
```

sin evidencia de que el DBMS permite continuar.

---

# 45. Statement retry hazard

Considérese:

```text
BEGIN

UPDATE A
UPDATE B  ← deadlock

retry UPDATE B
```

Si el DBMS abortó toda la TX:

```text
UPDATE A
```

ya no pertenece a una transacción válida.

Por ello:

```text
deadlock
→ retry failed statement
```

no será la estrategia general.

---

# 46. Full transaction retry

Preferido:

```text
Attempt 1
    BEGIN
    UPDATE A
    UPDATE B
    DEADLOCK
    ABORT

Attempt 2
    BEGIN
    UPDATE A
    UPDATE B
    COMMIT
```

---

# 47. Savepoint interaction

Un deadlock no deberá asumirse recuperable mediante:

```text
ROLLBACK TO SAVEPOINT
```

---

# 48. Why

Algunos errores de concurrencia pueden afectar toda la transacción independientemente de la existencia de savepoints.

Por tanto:

```text
SavepointExists
≠
DeadlockRecoverableAtSavepoint
```

---

# 49. Savepoint recovery classification

El platform profile podrá devolver:

```php
enum DeadlockSavepointRecovery
{
    case SUPPORTED;
    case NOT_SUPPORTED;
    case TRANSACTION_ABORTED;
    case UNKNOWN;
}
```

---

# 50. Safe default

```text
Deadlock
+
Unknown Savepoint Recovery
→
do not retry savepoint scope
```

---

# 51. Nested transaction interaction

Supongamos:

```text
Outer TX
    ↓
Nested Scope
        ↓
deadlock
```

Si el deadlock aborta toda la transacción:

```text
Nested scope
```

no puede aislar el fallo.

El error deberá propagarse al:

```text
physical transaction owner
```

---

# 52. REQUIRED

Para:

```text
REQUIRED
```

el inner comparte physical transaction.

Por tanto un deadlock transaction-aborting afecta:

```text
outer + inner
```

---

# 53. NESTED

Para:

```text
NESTED
```

solo podrá recuperarse localmente si la plataforma y el estado permiten recuperación mediante savepoint.

No se asumirá.

---

# 54. REQUIRES_NEW

Un:

```text
REQUIRES_NEW
```

deadlock afecta su propia physical transaction.

El outer suspendido no queda automáticamente abortado.

---

# 55. REQUIRES_NEW propagation

El outer podrá decidir:

```text
catch
continue
rollback
mark rollback-only
```

según lógica superior.

---

# 56. ORM integration

Un deadlock puede ocurrir durante:

```text
flush()
```

Ejemplo:

```text
UnitOfWork
    ↓
Persistence Plan
    ↓
UPDATE A
    ↓
UPDATE B
    ↓
DEADLOCK
```

---

# 57. Flush failure

Recordatorio:

```text
flush()
≠
commit()
```

Pero un deadlock durante flush puede invalidar la transacción física.

---

# 58. UoW implications

Después de:

```text
flush
→ partial statements executed
→ deadlock
→ transaction aborted
```

el estado del UoW puede no corresponder con la realidad final de DB.

---

# 59. ORM rule

```text
Deadlock Rollback
≠
Object Graph Rewind
```

---

# 60. PersistenceContext tainting

Por default, después de un deadlock transaction-aborting:

```text
PersistenceContext
→
TAINTED
```

o:

```text
CLOSED/CLEAR REQUIRED
```

según política.

---

# 61. Retry integration with ORM

El documento 170 estableció como modelo seguro:

```text
Attempt 1
    EM-1
    UoW-1
    IdentityMap-1

deadlock

Attempt 2
    EM-2
    UoW-2
    IdentityMap-2
```

---

# 62. No reuse assumption

No:

```text
deadlock
rollback
reuse same dirty EntityManager
```

por default.

---

# 63. Generated identifiers

Si antes del deadlock se ejecutó:

```text
INSERT
generated id = 400
```

y posteriormente la TX fue abortada:

```text
row may not exist
```

aunque el objeto PHP conserve:

```text
id = 400
```

---

# 64. Identifier reconciliation

El Deadlock System no restaurará el identificador.

Esto pertenece a:

```text
Persistence Recovery / Retry Coordination
```

---

# 65. Query correlation

Para diagnostics, un deadlock podrá relacionarse con:

```text
QueryExecutionId
TransactionId
ConnectionId
RetryBoundaryId
```

cuando estén disponibles.

---

# 66. Query fingerprint

Preferir:

```text
QueryFingerprint
```

sobre almacenar SQL completo.

---

# 67. Why fingerprints

Permiten identificar patrones:

```text
query A ↔ query B
```

sin introducir:

- parámetros sensibles;
- alta cardinalidad excesiva;
- enormes logs SQL.

---

# 68. Deadlock participant knowledge

VoltStack normalmente conoce:

```text
our transaction
our query
```

pero no necesariamente:

```text
all remote transactions
all lock owners
full wait graph
```

---

# 69. No invented participants

Si el DBMS no expone el grafo:

```text
participants = UNKNOWN
```

No inventar:

```text
TX-A deadlocked with TX-B
```

sin evidencia.

---

# 70. Server diagnostic enrichment

Una integración administrativa opcional podría consultar información adicional del servidor.

Pero:

```text
deadlock handling hot path
```

no deberá depender obligatoriamente de queries administrativas.

---

# 71. Privilege separation

El runtime normal de la aplicación no deberá requerir permisos administrativos únicamente para detectar deadlocks.

---

# 72. DeadlockReport

```php
final readonly class DeadlockReport
{
    public function __construct(
        public DeadlockId $id,
        public TransactionId $transaction,
        public ?QueryExecutionId $query,
        public DeadlockClassification $classification,
        public DeadlockDiagnosticContext $context,
    ) {}
}
```

---

# 73. DeadlockId

Será una identidad local de observabilidad.

```text
DeadlockId
≠
Database Server Deadlock ID
```

salvo que exista uno explícito.

---

# 74. DeadlockDiagnosticContext

Podrá contener:

```text
platform
server version
transaction phase
isolation level
query fingerprint
attempt number
nested depth
lock intent if known
duration before failure
```

---

# 75. Sensitive data

No deberá contener por default:

```text
raw passwords
credentials
PII
raw bound parameters
full entity payloads
```

---

# 76. Deadlock pattern

Diagnostics podrán identificar:

```text
Pattern:
    inverse update order
```

si existe evidencia suficiente.

Ejemplo:

```text
Flow A:
    UPDATE account 1
    UPDATE account 2

Flow B:
    UPDATE account 2
    UPDATE account 1
```

---

# 77. Deterministic lock ordering

Una de las principales recomendaciones será:

```text
acquire locks in consistent order
```

Ejemplo:

```text
sort entity IDs before updating
```

---

# 78. But no automatic semantic rewrite

VoltStack no deberá reordenar arbitrariamente:

```text
UPDATE operations
```

si esto puede cambiar semántica de negocio.

---

# 79. Persistence planner opportunity

`PersistencePlanner` podrá intentar generar un orden determinista cuando varias operaciones sean semánticamente independientes.

Pero:

```text
Deadlock System
≠
Persistence Planner
```

---

# 80. Deadlock prevention vs handling

```text
Prevention
≠
Detection
≠
Handling
≠
Retry
```

---

# 81. Prevention mechanisms

Otros subsistemas podrán reducir deadlocks mediante:

- deterministic write ordering;
- shorter transactions;
- proper indexes;
- reduced lock scope;
- optimistic concurrency;
- correct isolation;
- batch sizing;
- avoiding user I/O inside transactions.

---

# 82. Long-running transactions

Diagnostics podrán advertir:

```text
transaction duration unusually high
```

porque incrementa la ventana de conflictos.

---

# 83. External I/O inside transaction

Ejemplo problemático:

```php
DB::transaction(function () {
    lockRows();

    $response = $http->get(...);

    updateRows();
});
```

Los locks permanecen mientras se espera I/O externo.

---

# 84. Diagnostic recommendation

Podrá recomendar:

```text
move external I/O outside lock-holding transaction
```

cuando sea detectable.

---

# 85. Lock ordering metadata

Futuras APIs de persistence/concurrency podrán exponer:

```text
LockOrderKey
```

para facilitar ordenamiento determinista.

---

# 86. Batch operations

Un batch grande:

```text
UPDATE 100,000 rows
```

puede aumentar lock contention.

Deadlock diagnostics podrán correlacionar:

```text
batch size
transaction duration
deadlock frequency
```

---

# 87. Retry backoff integration

Si Transaction Retry System decide reintentar:

```text
Deadlock
    ↓
Rollback/Abort Confirmed
    ↓
Backoff + Jitter
    ↓
New Attempt
```

---

# 88. Why jitter matters

Sin jitter:

```text
TX-A retries immediately
TX-B retries immediately
```

pueden colisionar nuevamente.

---

# 89. Retry storm

Ante alta contención:

```text
deadlocks
    ↓
many immediate retries
    ↓
more contention
    ↓
more deadlocks
```

Por ello:

```text
retry budget + backoff + jitter
```

son importantes.

---

# 90. Deadlock frequency policy

Podrá existir:

```text
DeadlockRateMonitor
```

para detectar problemas sistémicos.

---

# 91. Rate monitor is observational

No deberá modificar transacciones directamente.

Puede alimentar:

- telemetry;
- alerts;
- diagnostics;
- circuit breaker integration;
- performance tooling.

---

# 92. Telemetry architecture

```text
Deadlock Detected
      ↓
DeadlockEvent
      ↓
Telemetry Bridge
   ┌────┼─────┐
   ▼    ▼     ▼
Metrics Logs Traces
```

---

# 93. Metrics

Ejemplos:

```text
db.deadlock.count
db.deadlock.retry_candidate
db.deadlock.transaction_aborted
db.deadlock.retry_success
```

---

# 94. Metric labels

Permitidos:

```text
platform
isolation
failure_phase
transaction_impact
retry_recommendation
```

con cardinalidad controlada.

---

# 95. Labels prohibited

Evitar:

```text
TransactionId
DeadlockId
UserId
raw SQL
table primary key
bound values
```

---

# 96. Trace event

Ejemplo:

```text
event:
    db.deadlock.detected

attributes:
    db.system=postgresql
    transaction.phase=execute
    transaction.impact=aborted
    retry.recommendation=transaction
```

---

# 97. Structured log

```text
Database deadlock detected

platform:
    PostgreSQL

transaction:
    tx-42

phase:
    EXECUTE

impact:
    TRANSACTION_ABORTED

retry:
    candidate

query fingerprint:
    qf-a42...
```

---

# 98. Production logging

Los logs no deberán incluir automáticamente:

```text
full SQL + all parameters
```

---

# 99. Developer mode

En development podrá habilitarse información adicional bajo políticas de redaction.

---

# 100. Explain API

Podrá existir:

```php
DB::concurrency()
    ->deadlocks()
    ->explain($exception);
```

---

# 101. Explain output

```text
DEADLOCK ANALYSIS

Classification:
    DEADLOCK

Confidence:
    CERTAIN

Platform:
    PostgreSQL

Transaction Impact:
    ABORTED

Retry Candidate:
    YES

Automatic Retry:
    Depends on active RetryPolicy

Isolation:
    SERIALIZABLE

Attempt:
    1 / 3

Recommendation:
    Retry complete transaction boundary.
```

---

# 102. More detailed diagnostics

Cuando exista suficiente información:

```text
Possible Cause:
    inconsistent write ordering

Observed Fingerprints:
    update-account
    update-ledger

Recommendation:
    normalize lock acquisition order
```

---

# 103. Recommendation confidence

Cada recomendación podrá incluir:

```php
enum DiagnosticConfidence
{
    case HIGH;
    case MEDIUM;
    case LOW;
}
```

No presentar heurísticas como hechos.

---

# 104. Exception hierarchy

```text
DatabaseConcurrencyException
│
├── DeadlockDetectedException
├── LockTimeoutException
├── SerializationFailureException
├── LockNotAvailableException
├── OptimisticLockException
└── UnknownConcurrencyException
```

---

# 105. Deadlock exception data

Deberá preservar:

```text
canonical classification
original driver exception
platform error identity
transaction impact
outcome certainty
retry recommendation
```

---

# 106. Original exception

La excepción original deberá mantenerse como:

```text
previous/cause
```

para debugging.

---

# 107. No vendor exception leakage requirement

La API de aplicación podrá capturar:

```php
catch (DeadlockDetectedException $e)
```

sin depender de PDO/vendor exception types.

---

# 108. Retry abstraction

Preferiblemente la aplicación no tendrá que hacer:

```php
catch (DeadlockDetectedException $e) {
    retryManually();
}
```

si ya utiliza:

```php
DB::transaction(..., retry: $policy);
```

---

# 109. Manual handling still possible

Sin retry policy:

```php
try {
    DB::transaction(...);
} catch (DeadlockDetectedException $e) {
    // application-specific response
}
```

seguirá siendo válido.

---

# 110. HTTP integration

Deadlock Handling System no decidirá directamente:

```text
HTTP 409
HTTP 500
HTTP 503
```

Eso corresponde a capas superiores.

---

# 111. Job integration

Tampoco decidirá si:

```text
queue job
```

debe reintentarse.

Debe distinguirse:

```text
Transaction Retry
```

de:

```text
Job Retry
```

---

# 112. Double retry hazard

Ejemplo:

```text
Transaction retry:
    3 attempts

Queue retry:
    5 attempts
```

Máximo potencial:

```text
15 transaction executions
```

---

# 113. Retry amplification

Formalmente:

```text
TotalPotentialAttempts
=
TransactionAttempts
×
OuterOperationAttempts
```

Esto deberá ser observable.

---

# 114. Retry budget coordination

Una futura integración con Jobs podrá compartir:

```text
OperationRetryBudget
```

para limitar amplificación.

---

# 115. Deadlock and cancellation

Si se detecta deadlock y la operación ya está cancelada:

```text
RetryRecommendation
```

podrá seguir siendo `RETRY_TRANSACTION`, pero:

```text
RetryDecision
=
CANCELLED
```

---

# 116. Deadlock and deadline

Igualmente:

```text
deadlock is retryable
```

pero:

```text
deadline expired
→ no retry
```

---

# 117. Transaction timeout

Deadlock Handling no deberá reiniciar automáticamente el deadline global.

---

# 118. Persistent runtime requirements

Mutable deadlock context deberá ser:

```text
operation-local
transaction-local
```

---

# 119. Shared immutable state

Podrán compartirse:

```text
Platform error maps
Concurrency error profiles
Classifier definitions
Diagnostic rules
```

si son inmutables.

---

# 120. No static current deadlock

Prohibido:

```php
DeadlockContext::$current = $deadlock;
```

en workers persistentes.

---

# 121. FrankenPHP

Request A:

```text
deadlock tx-A
```

no deberá contaminar Request B.

---

# 122. RoadRunner

Cada request/job deberá tener su propio:

```text
ConcurrencyDiagnosticContext
```

---

# 123. OpenSwoole

Dos coroutines:

```text
Coroutine A → deadlock A
Coroutine B → transaction B
```

deberán permanecer completamente aisladas.

---

# 124. Resource governance

El sistema deberá limitar:

- deadlock history;
- captured stack traces;
- SQL diagnostics;
- participant metadata;
- telemetry cardinality;
- repeated identical warnings.

---

# 125. Diagnostic history

Podrá mantenerse:

```text
last N deadlocks
```

en tooling local.

No:

```text
unbounded history in worker memory
```

---

# 126. Stack traces

En producción podrán:

```text
samplearse
```

o deshabilitarse para reducir costo.

---

# 127. Query fingerprints

El cálculo deberá reutilizar la infraestructura de:

```text
Query Metadata / Telemetry
```

en lugar de crear otro fingerprint incompatible.

---

# 128. Deduplication

Deadlocks repetidos con el mismo patrón podrán agruparse para diagnostics:

```text
PatternFingerprint
```

---

# 129. PatternFingerprint

Podría derivarse de:

```text
platform
query fingerprints
transaction phase
isolation
```

sin incluir IDs de negocio.

---

# 130. Deadlock storm detection

Conceptualmente:

```text
same pattern
+
high frequency
+
short interval
=
deadlock storm candidate
```

---

# 131. Storm detection does not alter truth

Una tormenta sigue siendo un conjunto de deadlocks individuales.

El agrupamiento es observacional.

---

# 132. Testing architecture

La suite deberá incluir:

```text
DeadlockClassifierTests
DeadlockNormalizationTests
DeadlockTransactionImpactTests
DeadlockRetryRecommendationTests
DeadlockMySqlTests
DeadlockMariaDbTests
DeadlockPostgreSqlTests
DeadlockSQLiteTests
DeadlockNestedTransactionTests
DeadlockSavepointTests
DeadlockORMTests
DeadlockTelemetryTests
DeadlockDiagnosticsTests
DeadlockPersistentRuntimeTests
DeadlockFailureInjectionTests
DeadlockIntegrationTests
```

---

# 133. Real deadlock integration test

Se deberán crear dos conexiones:

```text
Connection A
Connection B
```

y ejecutar locks en orden inverso.

---

# 134. Example test

```text
TX-A:
    lock row 1

TX-B:
    lock row 2

TX-A:
    request row 2

TX-B:
    request row 1
```

Esperado:

```text
one transaction becomes victim
```

y VoltStack clasifica correctamente el error.

---

# 135. Victim nondeterminism

Los tests no deberán asumir necesariamente:

```text
TX-A always victim
```

si el DBMS no garantiza eso.

---

# 136. Test assertion

Preferir:

```text
exactly one victim classified as deadlock
```

según comportamiento de la plataforma.

---

# 137. Lock timeout test

Construir:

```text
TX-A locks row
TX-B waits until timeout
```

Verificar:

```text
LOCK_TIMEOUT
```

no:

```text
DEADLOCK
```

---

# 138. Serialization failure test

Cuando la plataforma lo permita, generar un conflicto serializable y verificar:

```text
SERIALIZATION_FAILURE
```

como clasificación separada.

---

# 139. ORM deadlock test

Dos EntityManagers realizan updates en orden inverso.

Verificar:

```text
deadlock
transaction state updated
PersistenceContext tainted
retry uses fresh context
```

---

# 140. Nested REQUIRED test

```text
Outer
    Inner REQUIRED
        deadlock
```

Verificar que el physical owner recibe el error.

---

# 141. NESTED test

Si la plataforma no garantiza recuperación mediante savepoint:

```text
no local retry
```

---

# 142. REQUIRES_NEW test

Verificar que un deadlock inner no altera automáticamente el TransactionContext suspendido outer.

---

# 143. Retry test

```text
Attempt 1 → deadlock
Attempt 2 → commit
```

Verificar:

```text
Deadlock System classifications = 1
Retry System retries = 1
```

manteniendo responsabilidades separadas.

---

# 144. Unknown classification test

Un error ambiguo no deberá convertirse en deadlock solo porque contiene:

```text
lock
```

en el mensaje.

---

# 145. Persistent worker test

Después de completar una request con deadlock:

```text
DeadlockContext
```

deberá quedar liberado.

---

# 146. Platform conformance matrix

Cada plataforma deberá probar:

| Capacidad | MySQL | MariaDB | PostgreSQL | SQLite |
|---|---|---|---|---|
| Deadlock classification | Profile | Profile | Profile | Capability-dependent |
| Lock timeout distinction | Required | Required | Required | Capability-dependent |
| Serialization distinction | Required | Required | Required | Capability-dependent |
| Transaction impact | Explicit | Explicit | Explicit | Explicit |
| Retry recommendation | Canonical | Canonical | Canonical | Canonical |
| Savepoint recovery | Explicit | Explicit | Explicit | Explicit |

La tabla expresa requisitos de modelado, no afirma que todas las plataformas tengan las mismas capacidades.

---

# 147. Proposed directory structure

```text
src/Quantum/Database/Concurrency/
│
├── Deadlock/
│   ├── DeadlockClassification.php
│   ├── DeadlockConfidence.php
│   ├── DeadlockId.php
│   ├── DeadlockReport.php
│   ├── DeadlockSavepointRecovery.php
│   ├── DeadlockHandler.php
│   │
│   ├── Classification/
│   │   ├── ConcurrencyFailureType.php
│   │   ├── ConcurrencyFailureClassification.php
│   │   ├── TransactionImpact.php
│   │   └── RetryRecommendation.php
│   │
│   ├── Platform/
│   │   ├── ConcurrencyErrorProfile.php
│   │   ├── MySqlConcurrencyErrorProfile.php
│   │   ├── MariaDbConcurrencyErrorProfile.php
│   │   ├── PostgreSqlConcurrencyErrorProfile.php
│   │   └── SQLiteConcurrencyErrorProfile.php
│   │
│   ├── Error/
│   │   ├── DatabaseFailureDescriptor.php
│   │   ├── PlatformErrorIdentity.php
│   │   ├── PlatformErrorNormalizer.php
│   │   └── DriverErrorExtractor.php
│   │
│   ├── Diagnostics/
│   │   ├── DeadlockDiagnosticContext.php
│   │   ├── DeadlockInspector.php
│   │   ├── DeadlockPatternFingerprint.php
│   │   └── DeadlockRecommendation.php
│   │
│   ├── Telemetry/
│   │   ├── DeadlockTelemetry.php
│   │   └── DeadlockEvent.php
│   │
│   └── Exception/
│       ├── DatabaseConcurrencyException.php
│       ├── DeadlockDetectedException.php
│       ├── LockTimeoutException.php
│       ├── SerializationFailureException.php
│       └── UnknownConcurrencyException.php
```

---

# 148. Dependency model

Permitido:

```text
Deadlock Handling
      ↓
Driver Error Contracts
Platform Error Profiles
Transaction Context Contracts
Transaction State Model
Query Metadata
Telemetry Contracts
Diagnostics Contracts
```

Integración:

```text
Transaction Retry System
      ↓ consumes
Deadlock Classification
```

No permitido:

```text
Deadlock Handling
      ↓
direct application retry
business services
HTTP globals
static request state
raw vendor branching throughout core
```

---

# 149. Architectural invariants

## DB-DL-001
Deadlock será una categoría canónica de concurrencia.

## DB-DL-002
Deadlock no será lock timeout.

## DB-DL-003
Deadlock no será serialization failure.

## DB-DL-004
Deadlock no será optimistic lock conflict.

## DB-DL-005
Deadlock no será constraint violation.

## DB-DL-006
Deadlock detection del servidor no será reimplementada innecesariamente por VoltStack.

## DB-DL-007
VoltStack normalizará señales del driver/plataforma.

## DB-DL-008
Core no dependerá de mensajes vendor-specific.

## DB-DL-009
SQLSTATE podrá utilizarse como evidencia.

## DB-DL-010
Vendor code podrá utilizarse como evidencia.

## DB-DL-011
Failure phase podrá utilizarse como evidencia contextual.

## DB-DL-012
Transaction state podrá utilizarse como evidencia contextual.

## DB-DL-013
Deadlock classification tendrá confidence.

## DB-DL-014
UNKNOWN no será convertido a CERTAIN.

## DB-DL-015
Ambiguous lock errors no serán etiquetados automáticamente como deadlock.

## DB-DL-016
MySQL tendrá profile propio.

## DB-DL-017
MariaDB tendrá profile propio.

## DB-DL-018
PostgreSQL tendrá profile propio.

## DB-DL-019
SQLite tendrá profile propio.

## DB-DL-020
MariaDB no será alias semántico de MySQL.

## DB-DL-021
Platform version podrá afectar clasificación.

## DB-DL-022
Driver version podrá afectar extracción de error.

## DB-DL-023
Deadlock classification preservará error original.

## DB-DL-024
Aplicación podrá capturar DeadlockDetectedException portable.

## DB-DL-025
Deadlock exception no obligará a conocer PDO/vendor exceptions.

## DB-DL-026
Deadlock System no ejecutará automatic retry.

## DB-DL-027
Deadlock System producirá retry recommendation.

## DB-DL-028
Retry recommendation no será retry decision.

## DB-DL-029
Retry System será responsable de retry decision.

## DB-DL-030
Deadlock System determinará transaction impact cuando sea posible.

## DB-DL-031
Transaction impact será dimensión separada de failure type.

## DB-DL-032
Outcome certainty será dimensión separada de failure type.

## DB-DL-033
Transaction-aborting deadlock actualizará TransactionContext.

## DB-DL-034
Rollback-only deadlock actualizará TransactionContext.

## DB-DL-035
Unknown transaction impact podrá taintar TransactionContext.

## DB-DL-036
VoltStack no reseteará arbitrariamente transaction state a ACTIVE.

## DB-DL-037
Statement retry no será estrategia general ante deadlock.

## DB-DL-038
Full transaction retry será la recomendación típica para transaction-aborting deadlocks.

## DB-DL-039
Full transaction retry seguirá requiriendo RetryPolicy.

## DB-DL-040
Full transaction retry seguirá requiriendo owned boundary.

## DB-DL-041
Full transaction retry seguirá requiriendo safe outcome.

## DB-DL-042
Full transaction retry seguirá requiriendo budget.

## DB-DL-043
Full transaction retry seguirá respetando deadline.

## DB-DL-044
Full transaction retry seguirá respetando cancellation.

## DB-DL-045
Savepoint existence no implicará deadlock recovery.

## DB-DL-046
Savepoint recovery será platform-aware.

## DB-DL-047
Unknown savepoint recovery impedirá local retry por default.

## DB-DL-048
Nested REQUIRED compartirá impacto físico outer.

## DB-DL-049
NESTED no aislará transaction-aborting deadlock.

## DB-DL-050
REQUIRES_NEW tendrá impacto transaccional independiente.

## DB-DL-051
Outer suspendido no será automáticamente abortado por inner REQUIRES_NEW.

## DB-DL-052
Deadlock durante flush podrá abortar physical transaction.

## DB-DL-053
Flush failure no será commit failure por definición.

## DB-DL-054
Deadlock rollback no restaurará object graph.

## DB-DL-055
Deadlock rollback no restaurará UnitOfWork automáticamente.

## DB-DL-056
Deadlock rollback no restaurará IdentityMap automáticamente.

## DB-DL-057
Deadlock rollback no restaurará generated IDs automáticamente.

## DB-DL-058
PersistenceContext podrá quedar tainted.

## DB-DL-059
Retry ORM podrá utilizar fresh PersistenceContext.

## DB-DL-060
Deadlock System no implementará PersistenceContext reset directamente.

## DB-DL-061
Deadlock podrá correlacionarse con QueryExecutionId.

## DB-DL-062
Deadlock podrá correlacionarse con TransactionId.

## DB-DL-063
Deadlock podrá correlacionarse con RetryBoundaryId.

## DB-DL-064
QueryFingerprint será preferido para agregación.

## DB-DL-065
Raw SQL no será requerido para métricas.

## DB-DL-066
Bound parameters no serán incluidos por default.

## DB-DL-067
VoltStack no inventará deadlock participants.

## DB-DL-068
Full wait graph será UNKNOWN cuando no esté disponible.

## DB-DL-069
Hot path no requerirá consultas administrativas.

## DB-DL-070
Runtime DB credentials no requerirán permisos administrativos para clasificación normal.

## DB-DL-071
DeadlockId será identidad observacional local.

## DB-DL-072
DeadlockId no será asumido server ID.

## DB-DL-073
DeadlockReport será inmutable.

## DB-DL-074
Diagnostic recommendations tendrán confidence cuando sean heurísticas.

## DB-DL-075
Deterministic lock ordering podrá recomendarse.

## DB-DL-076
Deadlock System no reordenará business operations arbitrariamente.

## DB-DL-077
Persistence Planner podrá aplicar ordering únicamente cuando preserve semántica.

## DB-DL-078
Deadlock prevention será responsabilidad distribuida entre subsistemas.

## DB-DL-079
Long transaction duration será señal diagnóstica, no prueba de causa.

## DB-DL-080
External I/O dentro de transaction podrá advertirse.

## DB-DL-081
Large batch size podrá correlacionarse con contention.

## DB-DL-082
Correlation no implicará causalidad automática.

## DB-DL-083
Deadlock retry podrá usar backoff.

## DB-DL-084
Deadlock retry podrá usar jitter.

## DB-DL-085
Deadlock System no implementará backoff.

## DB-DL-086
Deadlock System no implementará jitter.

## DB-DL-087
Retry storm prevention pertenecerá a Retry/Resource Governance.

## DB-DL-088
Deadlock rate monitoring será observacional.

## DB-DL-089
Deadlock telemetry no alterará transaction semantics.

## DB-DL-090
Deadlock metrics usarán cardinalidad limitada.

## DB-DL-091
TransactionId no será metric label.

## DB-DL-092
DeadlockId no será metric label.

## DB-DL-093
Raw SQL no será metric label.

## DB-DL-094
Entity IDs no serán metric labels.

## DB-DL-095
Tracing podrá contener transaction correlation bajo política.

## DB-DL-096
Production logs aplicarán redaction.

## DB-DL-097
Developer diagnostics podrán ser más detallados.

## DB-DL-098
Credentials nunca se incluirán en deadlock diagnostics.

## DB-DL-099
Sensitive bound values no se incluirán por default.

## DB-DL-100
Deadlock history será bounded.

## DB-DL-101
Stack trace collection podrá ser configurable.

## DB-DL-102
Pattern fingerprint será estable dentro de su generación de metadata.

## DB-DL-103
Pattern fingerprint no incluirá IDs de negocio por default.

## DB-DL-104
Deadlock storm aggregation será observacional.

## DB-DL-105
Deadlock Handling será request/operation-safe.

## DB-DL-106
Mutable deadlock context no será global.

## DB-DL-107
FrankenPHP no compartirá deadlock state entre requests.

## DB-DL-108
RoadRunner no compartirá deadlock state entre operations.

## DB-DL-109
OpenSwoole no compartirá deadlock state entre coroutines.

## DB-DL-110
Immutable platform profiles podrán compartirse.

## DB-DL-111
Immutable error maps podrán compartirse.

## DB-DL-112
Mutable diagnostic context será scoped.

## DB-DL-113
Deadlock Handling no dependerá de HTTP.

## DB-DL-114
Deadlock Handling no dependerá de Jobs.

## DB-DL-115
Deadlock Handling no dependerá de Entity implementation.

## DB-DL-116
Deadlock Handling no dependerá de business services.

## DB-DL-117
Job retry será distinto de transaction retry.

## DB-DL-118
HTTP retry será distinto de transaction retry.

## DB-DL-119
Retry amplification será diagnosticable.

## DB-DL-120
Transaction retry count podrá correlacionarse con outer retry count.

## DB-DL-121
Deadlock Handling respetará cancellation indirectamente mediante Retry System.

## DB-DL-122
Deadlock Handling respetará deadlines indirectamente mediante Retry System.

## DB-DL-123
Deadlock classification no cambiará por haber expirado deadline.

## DB-DL-124
Retry decision sí podrá cambiar por deadline.

## DB-DL-125
Deadlock classification no cambiará por retry budget.

## DB-DL-126
Retry decision sí podrá cambiar por retry budget.

## DB-DL-127
Deadlock classification no será confundida con policy.

## DB-DL-128
Transaction impact no será inferido sin evidencia suficiente.

## DB-DL-129
UNKNOWN transaction impact permanecerá UNKNOWN.

## DB-DL-130
UNKNOWN outcome no será considerado safe retry automáticamente.

## DB-DL-131
Commit ambiguity tendrá prioridad sobre la etiqueta transient.

## DB-DL-132
Deadlock during COMMIT será evaluado según evidencia real, no solo failure class.

## DB-DL-133
Deadlock during EXECUTE será evaluado según platform transaction semantics.

## DB-DL-134
Rollback failure posterior a deadlock será preservado.

## DB-DL-135
Primary deadlock exception no será ocultado por cleanup errors.

## DB-DL-136
Composite failure information podrá conservarse.

## DB-DL-137
Deadlock diagnostics deberán ser explicables.

## DB-DL-138
Deadlock recommendations no deberán afirmar certeza causal sin evidencia.

## DB-DL-139
Testing incluirá deadlocks reales cuando la plataforma lo permita.

## DB-DL-140
Tests no asumirán víctima específica si la plataforma no lo garantiza.

## DB-DL-141
Testing distinguirá deadlock y timeout.

## DB-DL-142
Testing distinguirá deadlock y serialization failure.

## DB-DL-143
Testing cubrirá nested transactions.

## DB-DL-144
Testing cubrirá ORM flush failures.

## DB-DL-145
Testing cubrirá retry integration.

## DB-DL-146
Testing cubrirá persistent runtimes.

## DB-DL-147
Platform conformance será obligatoria para nuevos drivers.

## DB-DL-148
Custom drivers deberán proporcionar error normalization suficiente.

## DB-DL-149
Un custom driver que no pueda clasificar deadlocks deberá devolver UNKNOWN en vez de adivinar.

## DB-DL-150
VoltStack nunca convertirá un error ambiguo en deadlock únicamente para habilitar un retry.

---

# 150. Anti-patterns

## 150.1 Parsear mensajes en application code

```php
if (str_contains($e->getMessage(), 'deadlock')) {
    // retry
}
```

**Rechazado.**

---

## 150.2 Deadlock = retry

```text
DEADLOCK
→
always retry
```

**Incorrecto.**

Debe ser:

```text
DEADLOCK
→
classify impact
→
evaluate retry boundary
→
policy decision
```

---

## 150.3 Deadlock = lock timeout

```text
any lock error
→ DEADLOCK
```

**Prohibido.**

---

## 150.4 Retry del último statement

```text
UPDATE B deadlocked
→ retry UPDATE B
```

sin reiniciar una transacción abortada.

**Rechazado.**

---

## 150.5 Reutilizar EntityManager tainted

```text
deadlock
→ rollback
→ reuse same dirty UoW
```

**Rechazado como default.**

---

## 150.6 Savepoint mágico

```text
deadlock
→ rollback to latest savepoint
→ continue
```

sin verificar semántica de plataforma.

**Prohibido.**

---

## 150.7 Retry inmediato ilimitado

```text
deadlock
→ retry instantly forever
```

**Prohibido.**

---

## 150.8 SQL completo en métricas

```text
label.sql =
    "UPDATE users SET..."
```

**Rechazado.**

---

## 150.9 Inventar el grafo

```text
Deadlock detected
→ fabricate remote participant information
```

**Prohibido.**

---

## 150.10 Confundir victim selection con retry policy

```text
DB selected us as victim
→ VoltStack must retry
```

**Incorrecto.**

---

# 151. Master formulas

## Deadlock

```text
Deadlock(G)
⇔
Cycle(G)
```

para un wait-for graph conceptual `G`.

---

## Classification

```text
DeadlockClassification
=
f(
    SQLSTATE,
    VendorCode,
    DriverMetadata,
    PlatformProfile,
    FailurePhase,
    TransactionState
)
```

---

## Retry candidacy

```text
RetryCandidate
=
DeadlockClassification
+
TransactionImpact
+
OutcomeCertainty
```

---

## Actual retry

```text
RetryDecision
=
RetryCandidate
∧
BoundaryOwned
∧
PolicyAllows
∧
OutcomeSafe
∧
BudgetAvailable
∧
DeadlineAvailable
∧
ContextRecoverable
∧
¬Cancelled
```

---

## ORM consequence

```text
DeadlockRollback
≠
ObjectGraphRollback
```

---

## Savepoint consequence

```text
Deadlock
+
SavepointExists
≠
SavepointRecoverable
```

---

## Unknown

```text
TransactionImpact = UNKNOWN
→
SafeAutomaticRetry = FALSE
```

por default.

---

# 152. Modelo final

```text
                     Database Server
                           │
                    detects conflict
                           │
                           ▼
                      Driver Error
                           │
                           ▼
                  Driver Error Extractor
                           │
                           ▼
                 Platform Error Profile
                           │
                           ▼
                Canonical Classification
                           │
             ┌─────────────┼──────────────┐
             ▼             ▼              ▼
         DEADLOCK     LOCK_TIMEOUT   SERIALIZATION
             │
             ▼
      Transaction Impact
             │
             ▼
      Outcome Certainty
             │
             ▼
     TransactionContext
             │
             ▼
 DeadlockDetectedException
             │
      ┌──────┼───────────┐
      ▼      ▼           ▼
Telemetry Diagnostics Retry System
                         │
                         ▼
                   Retry Policy
                         │
                 ┌───────┴───────┐
                 ▼               ▼
             NO RETRY          RETRY
                                  │
                                  ▼
                        New Transaction Attempt
```

---

# 153. Decisiones arquitectónicas finales

VoltStack no implementará un detector de deadlocks paralelo al DBMS cuando el servidor ya disponga de uno.

La arquitectura se centrará en:

```text
Detection Evidence
        ↓
Normalization
        ↓
Classification
        ↓
Transaction Impact
        ↓
Recovery Recommendation
```

Las diferencias entre:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

se encapsularán mediante perfiles específicos de plataforma.

El core trabajará con categorías canónicas:

```text
DEADLOCK
LOCK_TIMEOUT
SERIALIZATION_FAILURE
OPTIMISTIC_LOCK_CONFLICT
LOCK_NOT_AVAILABLE
UNKNOWN
```

evitando dependencias directas de mensajes de error.

Se mantendrá estrictamente:

```text
Deadlock Detection
≠
Transaction Retry
```

El Deadlock Handling System podrá indicar:

```text
RETRY_TRANSACTION
```

pero únicamente:

```text
TransactionRetrySystem
```

decidirá si realmente puede repetirse el boundary.

También se conservará:

```text
Deadlock
≠
Lock Timeout
≠
Serialization Failure
```

porque cada condición puede tener diferente:

- causa;
- impacto transaccional;
- estrategia de recuperación;
- observabilidad;
- comportamiento por plataforma.

Los deadlocks durante ORM persistence respetarán:

```text
DatabaseRollback
≠
ObjectGraphRewind
```

por lo que un `PersistenceContext` afectado podrá quedar:

```text
TAINTED
```

y un retry completo deberá preferir un contexto nuevo o demostrablemente restaurado.

La existencia de un savepoint tampoco implicará recuperación local:

```text
SavepointExists
≠
DeadlockSavepointRecoverable
```

Finalmente, cualquier incertidumbre se conservará:

```text
UNKNOWN
```

en lugar de convertirla artificialmente en un error retryable.

La regla final será:

> **VoltStack tratará los deadlocks como información estructurada sobre un fallo de concurrencia y su efecto transaccional; la recuperación será una decisión posterior, explícita y verificable, nunca una reacción automática basada únicamente en haber encontrado la palabra “deadlock”.**

---

# 154. Relación con Transaction & Concurrency

```text
164_DATABASE_TRANSACTION_ARCHITECTURE
                ↓
165_DATABASE_TRANSACTION_MANAGER_SYSTEM
                ↓
166_DATABASE_TRANSACTION_CONTEXT_SYSTEM
                ↓
167_DATABASE_TRANSACTION_ISOLATION_SYSTEM
                ↓
168_DATABASE_NESTED_TRANSACTION_SYSTEM
                ↓
169_DATABASE_SAVEPOINT_SYSTEM
                ↓
170_DATABASE_TRANSACTION_RETRY_SYSTEM
                ↓
171_DATABASE_DEADLOCK_HANDLING_SYSTEM
                ↓
172_DATABASE_OPTIMISTIC_LOCKING_SYSTEM
                ↓
173_DATABASE_PESSIMISTIC_LOCKING_SYSTEM
                ↓
174_DATABASE_CONCURRENCY_CONTROL_SYSTEM
                ↓
175_DATABASE_TRANSACTION_EVENT_SYSTEM
```

---

# 155. Siguiente documento

```text
172_DATABASE_OPTIMISTIC_LOCKING_SYSTEM.md
```

El siguiente documento deberá definir en profundidad:

- optimistic concurrency architecture;
- version fields;
- version tokens;
- integer versioning;
- timestamp versioning;
- opaque version tokens;
- compare-and-swap semantics;
- expected vs actual version;
- update predicates;
- affected-row verification;
- stale entity detection;
- optimistic delete;
- optimistic relationship modifications;
- UnitOfWork integration;
- ChangeSet integration;
- Persistence Planner integration;
- SQL Compiler integration;
- entity hydration;
- snapshots;
- detached entities;
- merge/update semantics;
- bulk DML limitations;
- retry interaction;
- transaction isolation interaction;
- HTTP/API ETag integration boundaries;
- conflict reporting;
- conflict resolution;
- telemetry;
- diagnostics;
- persistent-runtime safety;
- testing;
- architectural invariants.