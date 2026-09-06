# 84_DATABASE_QUERY_TIMEOUT_AND_CANCELLATION_SYSTEM.md

# VoltStack Quantum Database
## Query Timeout and Cancellation System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 84 — Query Timeout and Cancellation System  
**Bloque:** 7 — Execution Engine  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Query Timeout and Cancellation System` define la arquitectura mediante la cual VoltStack puede interrumpir, limitar temporalmente y finalizar de forma segura operaciones de base de datos en ejecución.

El sistema cubre transversalmente:

```text
ExecutionInstance
├── QueryExecutor
├── StatementExecutor
├── PreparedStatement
├── Parameter Binding
├── ResultCursor
├── StreamingResult
├── Connection Acquisition
├── Driver Operation
└── Transaction Coordination
```

Su responsabilidad es proporcionar un modelo uniforme para:

- cancellation explícita;
- deadlines;
- query timeouts;
- statement timeouts;
- connection acquisition timeouts;
- fetch timeouts;
- stream deadlines;
- idle timeouts;
- propagación de cancellation;
- composición de deadlines;
- driver cancellation;
- server-side cancellation;
- client-side interruption;
- cleanup;
- clasificación del impacto sobre connection y transaction;
- partial-effect handling;
- race conditions;
- telemetry;
- persistent-runtime safety.

Principio central:

```text
Cancellation
≠
Timeout
≠
Failure
≠
Successful Completion
```

---

# 2. Principio arquitectónico

VoltStack deberá distinguir claramente:

```text
Cancellation
=
Explicit Termination Request
```

de:

```text
Timeout
=
Deadline Expiration
```

Un timeout puede utilizar internamente el mecanismo de cancellation:

```text
Deadline Expires
      ↓
Timeout Detected
      ↓
Cancellation Requested
      ↓
Active Operation Interrupted
```

pero la causa semántica seguirá siendo:

```text
TIMED_OUT
```

y no:

```text
CANCELLED
```

---

# 3. Problema

Una operación de base de datos puede quedar bloqueada o prolongarse debido a:

```text
slow query
lock contention
deadlock detection delay
network stall
database overload
connection acquisition delay
slow result consumption
driver stall
remote database failure
application cancellation
HTTP client disconnect
worker shutdown
resource pressure
```

Sin una arquitectura explícita, aparecen problemas como:

```text
request terminated
     ↓
query continues running
     ↓
connection remains occupied
     ↓
worker resources remain alive
```

o:

```text
timeout
   ↓
exception thrown
   ↓
driver still executing query
```

o incluso:

```text
connection returned to pool
while cancelled statement
still has unknown state
```

VoltStack deberá evitar estos escenarios.

---

# 4. Objetivos

El sistema deberá proporcionar:

1. cancellation first-class;
2. deadlines first-class;
3. propagación jerárquica;
4. timeout composition;
5. monotonic timing;
6. cancellation scopes;
7. driver-aware cancellation;
8. server-side timeout integration;
9. client-side timeout fallback;
10. connection impact classification;
11. transaction impact classification;
12. partial-effect awareness;
13. race-safe completion;
14. deterministic cleanup;
15. streaming integration;
16. resource-governance integration;
17. telemetry;
18. persistent-worker isolation;
19. runtime neutrality.

---

# 5. No objetivos

Este sistema no deberá:

```text
generate SQL
optimize queries
plan execution
retry queries automatically
commit transactions
rollback caller-owned transactions arbitrarily
hydrate ORM entities
authorize database access
change query semantics
invent idempotency
silently reconnect
silently rerun queries
```

---

# 6. Terminología fundamental

VoltStack deberá distinguir:

```text
CancellationToken
CancellationSource
CancellationRequest
CancellationReason
CancellationScope

Deadline
TimeoutPolicy
TimeoutEvent
TimeoutScope

DriverCancellation
ServerSideCancellation
ClientSideInterruption

ExecutionTermination
ExecutionFailure
```

---

# 7. CancellationToken

`CancellationToken` representa una señal observable de cancelación.

```php
namespace VoltStack\Quantum\Database\Execution\Cancellation\Contract;

interface CancellationToken
{
    public function isCancellationRequested(): bool;

    public function reason(): ?CancellationReason;

    public function throwIfCancellationRequested(): void;
}
```

---

# 8. CancellationSource

La fuente controla la transición:

```text
NOT_CANCELLED
      ↓
CANCEL_REQUESTED
```

Contrato:

```php
interface CancellationSource
{
    public function token(): CancellationToken;

    public function cancel(
        CancellationReason $reason
    ): CancellationRequestOutcome;
}
```

---

# 9. Separación Source / Token

El consumidor normal recibe:

```text
CancellationToken
```

no:

```text
CancellationSource
```

Principio:

```text
Observe Cancellation
≠
Authority To Cancel
```

---

# 10. CancellationReason

```php
enum CancellationReason: string
{
    case USER_REQUEST = 'user_request';
    case CALLER_REQUEST = 'caller_request';
    case PARENT_CANCELLED = 'parent_cancelled';
    case REQUEST_DISCONNECTED = 'request_disconnected';
    case REQUEST_TERMINATED = 'request_terminated';
    case WORKER_SHUTDOWN = 'worker_shutdown';
    case APPLICATION_SHUTDOWN = 'application_shutdown';
    case RESOURCE_PRESSURE = 'resource_pressure';
    case ADMINISTRATIVE = 'administrative';
    case TIMEOUT = 'timeout';
    case TRANSACTION_ABORT = 'transaction_abort';
    case CONNECTION_INVALIDATED = 'connection_invalidated';
    case DEPENDENCY_FAILURE = 'dependency_failure';
}
```

---

# 11. CancellationReason ≠ final status

Una razón:

```text
TIMEOUT
```

puede activar cancellation internamente.

Sin embargo, el resultado final será:

```text
TIMED_OUT
```

---

# 12. Cancellation state

Se propone:

```php
enum CancellationState: string
{
    case ACTIVE = 'active';
    case REQUESTED = 'requested';
    case PROPAGATING = 'propagating';
    case ACKNOWLEDGED = 'acknowledged';
    case COMPLETED = 'completed';
}
```

---

# 13. Cancellation state machine

```text
ACTIVE
  │
  │ cancel()
  ▼
REQUESTED
  │
  ▼
PROPAGATING
  │
  ▼
ACKNOWLEDGED
  │
  ▼
COMPLETED
```

---

# 14. Cancellation request idempotence

Múltiples solicitudes:

```text
cancel()
cancel()
cancel()
```

no deberán provocar múltiples interrupciones inconsistentes.

Idealmente:

```text
Cancel(Cancel(X)) = Cancel(X)
```

---

# 15. First cancellation reason

Por defecto, la primera causa efectiva deberá preservarse.

Ejemplo:

```text
USER_REQUEST
     ↓
cancelled
     ↓
WORKER_SHUTDOWN
```

La causa primaria continúa siendo:

```text
USER_REQUEST
```

aunque razones posteriores puedan registrarse como información secundaria.

---

# 16. Cancellation hierarchy

La cancelación deberá poder propagarse:

```text
Execution
   ↓
Stage
   ↓
ExecutionUnit
   ↓
Statement
   ↓
Cursor
   ↓
Stream
```

---

# 17. Parent-child cancellation

```text
Cancel(parent)
⇒
Cancel(children)
```

cuando exista dependencia de lifecycle.

Pero:

```text
Cancel(child)
⇏
Cancel(parent)
```

por defecto.

---

# 18. CancellationScope

Se propone:

```php
final readonly class CancellationScope
{
    public function __construct(
        public CancellationScopeId $id,
        public CancellationToken $token,
        public ?CancellationScopeId $parent,
        public CancellationPropagationPolicy $propagation,
    ) {}
}
```

---

# 19. Scope examples

```text
RequestCancellationScope
ExecutionCancellationScope
StageCancellationScope
UnitCancellationScope
StatementCancellationScope
CursorCancellationScope
StreamCancellationScope
```

---

# 20. No global cancellation token

Nunca:

```php
static $currentCancellationToken;
```

---

# 21. Deadline

Un deadline representa un instante máximo absoluto dentro de un dominio de reloj.

```php
final readonly class Deadline
{
    public function __construct(
        public MonotonicInstant $instant,
    ) {}
}
```

---

# 22. Timeout vs Deadline

Timeout:

```text
duration
```

Ejemplo:

```text
5 seconds
```

Deadline:

```text
absolute execution boundary
```

Conceptualmente:

```text
deadline = monotonicNow + timeout
```

---

# 23. Preferencia interna

Las APIs públicas pueden aceptar:

```text
timeout = 5 seconds
```

pero internamente deberán convertirse lo antes posible a:

```text
Deadline
```

---

# 24. Por qué

Si cada suboperación recibe nuevamente:

```text
5 seconds
```

podría ocurrir:

```text
connection acquire = 5s
prepare            = 5s
execute            = 5s
fetch              = 5s
```

resultando:

```text
total = 20s
```

aunque el usuario pidió:

```text
5s
```

---

# 25. Deadline propagation

Correctamente:

```text
Global Deadline = T
```

Cada operación calcula:

```text
RemainingTime = T - now
```

---

# 26. RemainingTime

```php
interface DeadlineClock
{
    public function remaining(
        Deadline $deadline
    ): Duration;
}
```

---

# 27. Expired deadline

```text
now ≥ deadline
⇒
deadline expired
```

---

# 28. Monotonic clock

Los deadlines de ejecución deberán utilizar reloj monotónico.

No:

```text
wall clock
```

para medir duración operacional.

---

# 29. Motivo

El reloj civil puede cambiar debido a:

```text
NTP adjustment
manual clock change
DST
timezone change
virtualization
```

---

# 30. Clock abstraction

```php
interface MonotonicClock
{
    public function now(): MonotonicInstant;
}
```

---

# 31. Wall clock

Podrá utilizarse para:

```text
human diagnostics
logs
timestamps
```

pero no como fuente principal para enforcement de duration.

---

# 32. Deadline composition

Si existen varios deadlines:

```text
RequestDeadline
ExecutionDeadline
StatementDeadline
TransactionDeadline
```

el deadline efectivo será normalmente:

```text
EffectiveDeadline
=
min(
    RequestDeadline,
    ExecutionDeadline,
    StatementDeadline,
    TransactionDeadline
)
```

---

# 33. Nested deadline invariant

Un child scope no deberá extender silenciosamente el parent deadline.

```text
ChildDeadline ≤ ParentDeadline
```

---

# 34. Local shorter deadline

Sí puede existir:

```text
Parent = 30s
Child  = 5s
```

---

# 35. Local longer deadline

Si:

```text
Parent = 5s
Child requested = 30s
```

el efectivo será:

```text
5s
```

---

# 36. DeadlineScope

```php
final readonly class DeadlineScope
{
    public function __construct(
        public DeadlineScopeId $id,
        public Deadline $deadline,
        public DeadlineOrigin $origin,
        public ?DeadlineScopeId $parent,
    ) {}
}
```

---

# 37. DeadlineOrigin

```php
enum DeadlineOrigin: string
{
    case REQUEST = 'request';
    case EXECUTION = 'execution';
    case QUERY = 'query';
    case STATEMENT = 'statement';
    case CONNECTION_ACQUISITION = 'connection_acquisition';
    case FETCH = 'fetch';
    case STREAM = 'stream';
    case TRANSACTION = 'transaction';
    case RESOURCE_POLICY = 'resource_policy';
}
```

---

# 38. Timeout scopes

VoltStack deberá distinguir:

```text
ConnectionAcquisitionTimeout
QueryTimeout
StatementTimeout
ExecutionTimeout
FetchTimeout
StreamTimeout
StreamIdleTimeout
TransactionTimeout
```

---

# 39. Connection acquisition timeout

Limita:

```text
waiting for ConnectionLease
```

No necesariamente:

```text
query execution
```

---

# 40. Statement timeout

Limita la operación:

```text
prepare/execute statement
```

según capabilities.

---

# 41. Query timeout

Representa la política de duración permitida para la query lógica/ejecución asociada.

Puede abarcar varias operaciones runtime si el `ExecutionPlan` lo requiere.

---

# 42. Fetch timeout

Puede limitar:

```text
single cursor fetch operation
```

sin extender el deadline global.

---

# 43. Stream timeout

Limita la vida completa del stream.

---

# 44. Stream idle timeout

Limita cuánto tiempo puede permanecer el stream sin actividad del consumidor.

---

# 45. Idle timeout ≠ execution timeout

```text
ExecutionTimeout = 60s
IdleTimeout = 10s
```

puede significar:

```text
stream may live up to 60s
but consumer cannot remain idle > 10s
```

---

# 46. TimeoutPolicy

```php
final readonly class TimeoutPolicy
{
    public function __construct(
        public ?Duration $executionTimeout,
        public ?Duration $connectionAcquisitionTimeout,
        public ?Duration $statementTimeout,
        public ?Duration $fetchTimeout,
        public ?Duration $streamIdleTimeout,
        public TimeoutEnforcementPolicy $enforcement,
    ) {}
}
```

---

# 47. TimeoutEnforcementPolicy

```php
enum TimeoutEnforcementPolicy: string
{
    case STRICT = 'strict';
    case BEST_EFFORT = 'best_effort';
    case DRIVER_NATIVE = 'driver_native';
    case FRAMEWORK_ONLY = 'framework_only';
    case HYBRID = 'hybrid';
}
```

---

# 48. STRICT

Si VoltStack no puede garantizar la política solicitada:

```text
fail before execution
```

cuando sea posible.

---

# 49. BEST_EFFORT

VoltStack aplica todos los mecanismos disponibles y reporta claramente las garantías reales.

---

# 50. DRIVER_NATIVE

Depende de mecanismos nativos del driver/database.

---

# 51. FRAMEWORK_ONLY

VoltStack controla el deadline desde el proceso cliente, sin afirmar que el servidor pueda detener inmediatamente la operación.

---

# 52. HYBRID

Combina:

```text
server-side timeout
+
client-side deadline
+
cancellation fallback
```

---

# 53. Timeout capability model

El sistema deberá conocer capacidades como:

```text
supportsServerStatementTimeout
supportsDriverQueryTimeout
supportsAsyncCancellation
supportsConnectionAbort
supportsCursorCancellation
supportsPerStatementTimeout
supportsSessionTimeout
supportsTransactionLocalTimeout
```

---

# 54. Capability ≠ vendor check

No:

```php
if ($driver === 'pgsql') {
}
```

Preferir:

```php
if ($capabilities->supportsStatementCancellation()) {
}
```

---

# 55. TimeoutGuarantee

```php
enum TimeoutGuarantee: string
{
    case SERVER_ENFORCED = 'server_enforced';
    case DRIVER_ENFORCED = 'driver_enforced';
    case FRAMEWORK_OBSERVED = 'framework_observed';
    case HYBRID = 'hybrid';
    case BEST_EFFORT = 'best_effort';
    case UNSUPPORTED = 'unsupported';
}
```

---

# 56. Server-side timeout

El servidor de base de datos detiene la operación.

Conceptualmente:

```text
Database Server
      ↓
deadline/timeout
      ↓
statement aborted
```

---

# 57. Driver-side timeout

El driver aplica un mecanismo propio.

No necesariamente equivale a server-side cancellation.

---

# 58. Framework-observed timeout

VoltStack detecta:

```text
deadline expired
```

pero el driver puede seguir bloqueado hasta recuperar control.

---

# 59. Critical honesty rule

VoltStack no deberá afirmar:

```text
query cancelled on server
```

si sólo sabe:

```text
client stopped waiting
```

---

# 60. Cancellation capabilities

```php
enum CancellationCapability: string
{
    case SERVER_STATEMENT_CANCEL = 'server_statement_cancel';
    case DRIVER_OPERATION_CANCEL = 'driver_operation_cancel';
    case CURSOR_CLOSE = 'cursor_close';
    case CONNECTION_ABORT = 'connection_abort';
    case COOPERATIVE_ONLY = 'cooperative_only';
}
```

---

# 61. Cancellation strategy

El sistema puede elegir una estrategia compatible:

```text
1. cooperative cancellation
2. cursor cancellation
3. statement cancellation
4. driver cancellation
5. server cancellation
6. connection abort
```

según capabilities y política.

---

# 62. Cancellation escalation

Puede existir:

```text
request cancellation
      ↓
graceful statement cancel
      ↓
wait bounded grace period
      ↓
connection abort
```

si está autorizado.

---

# 63. No automatic destructive escalation

Abortar una conexión puede afectar:

```text
transaction
session state
other resources
```

por tanto debe ser policy-driven.

---

# 64. CancellationCoordinator

```php
interface CancellationCoordinator
{
    public function requestCancellation(
        ExecutionCancellationRequest $request
    ): CancellationOutcome;
}
```

---

# 65. ExecutionCancellationRequest

```php
final readonly class ExecutionCancellationRequest
{
    public function __construct(
        public CancellationReason $reason,
        public CancellationScopeId $scope,
        public CancellationUrgency $urgency,
    ) {}
}
```

---

# 66. CancellationUrgency

```php
enum CancellationUrgency
{
    case GRACEFUL;
    case NORMAL;
    case IMMEDIATE;
}
```

---

# 67. Urgency ≠ permission

`IMMEDIATE` no significa que cualquier recurso pueda destruirse arbitrariamente.

La estrategia sigue limitada por:

```text
capabilities
ownership
transaction state
security policy
resource policy
```

---

# 68. CancellationOutcome

```php
final readonly class CancellationOutcome
{
    public function __construct(
        public CancellationStatus $status,
        public CancellationGuarantee $guarantee,
        public ConnectionImpact $connectionImpact,
        public TransactionImpact $transactionImpact,
    ) {}
}
```

---

# 69. CancellationStatus

```php
enum CancellationStatus: string
{
    case REQUESTED = 'requested';
    case ACKNOWLEDGED = 'acknowledged';
    case COMPLETED = 'completed';
    case TOO_LATE = 'too_late';
    case UNSUPPORTED = 'unsupported';
    case FAILED = 'failed';
}
```

---

# 70. TOO_LATE

Una carrera legítima:

```text
cancel requested
      ↓
query completed first
```

Resultado:

```text
TOO_LATE
```

No necesariamente error.

---

# 71. CancellationGuarantee

```php
enum CancellationGuarantee: string
{
    case SERVER_CONFIRMED = 'server_confirmed';
    case DRIVER_CONFIRMED = 'driver_confirmed';
    case LOCAL_RESOURCE_CLOSED = 'local_resource_closed';
    case REQUEST_ONLY = 'request_only';
    case UNKNOWN = 'unknown';
}
```

---

# 72. Race: completion vs cancellation

Considérese:

```text
Thread/Coroutine A:
query completes

Thread/Coroutine B:
cancel()
```

Debe existir una única transición terminal válida.

---

# 73. Terminal state arbitration

Estados terminales:

```text
COMPLETED
FAILED
CANCELLED
TIMED_OUT
UNKNOWN
```

La transición deberá ser atómica/lógicamente serializada.

---

# 74. TerminalStateCoordinator

```php
interface TerminalStateCoordinator
{
    public function tryTransition(
        ExecutionTerminalState $state,
        TerminationEvidence $evidence
    ): TerminalTransitionOutcome;
}
```

---

# 75. First terminal winner

Una vez establecida una transición terminal válida:

```text
terminal state
```

no podrá sobrescribirse arbitrariamente.

---

# 76. But cleanup continues

Terminal state:

```text
CANCELLED
```

no significa que todos los recursos ya estén cerrados.

---

# 77. Execution state distinction

```text
Execution Semantic Terminal State
≠
Resource Cleanup State
```

---

# 78. Example

```text
RUNNING
  ↓
CANCELLED
  ↓
CLEANING
  ↓
CLOSED
```

---

# 79. TimeoutCoordinator

```php
interface TimeoutCoordinator
{
    public function check(
        DeadlineScope $scope
    ): DeadlineCheckResult;
}
```

---

# 80. DeadlineCheckResult

```php
enum DeadlineCheckResult
{
    case ACTIVE;
    case EXPIRING;
    case EXPIRED;
}
```

---

# 81. EXPIRING

Puede utilizarse para:

```text
telemetry
resource preparation
avoid starting expensive optional work
```

pero no debe cambiar semántica.

---

# 82. DeadlineScheduler

En runtimes capaces de timers podrá existir:

```php
interface DeadlineScheduler
{
    public function schedule(
        Deadline $deadline,
        DeadlineCallback $callback
    ): DeadlineRegistration;
}
```

---

# 83. Polling fallback

En runtimes sin timers adecuados:

```text
check deadline at safe points
```

---

# 84. Safe points

Ejemplos:

```text
before connection acquisition
after connection acquisition
before prepare
after prepare
before bind
before execute
after execute
before fetch
after fetch
between execution units
before materialization
during framework-controlled loops
before returning result
```

---

# 85. Safe point limitation

Si el driver bloquea dentro de:

```text
execute()
```

un check antes y después no puede interrumpirlo.

Se requiere capability de driver/server para cancellation real.

---

# 86. Statement execution integration

```text
StatementExecutor
      ↓
check cancellation
      ↓
check deadline
      ↓
configure native timeout if supported
      ↓
execute
      ↓
classify outcome
```

---

# 87. Prepared statement integration

El `PreparedStatement` live handle podrá recibir configuración runtime de timeout sólo si el driver lo soporta.

La configuración:

```text
timeout value
```

no forma parte necesariamente del `PreparedStatementBlueprint`.

---

# 88. Why

El blueprint es reusable/cacheable.

El deadline:

```text
operation-scoped runtime state
```

---

# 89. Parameter binding integration

Cancellation podrá verificarse antes de binding costoso.

Pero:

```text
CancellationToken
```

no se convierte en un SQL parameter.

---

# 90. ResultCursor integration

Antes de cada fetch:

```text
check cancellation
check deadline
```

cuando sea apropiado.

---

# 91. Streaming integration

Del documento 83:

```text
StreamingResult
```

deberá heredar o vincularse al cancellation/deadline scope de la ejecución.

---

# 92. Streaming cancellation flow

```text
Cancellation requested
        ↓
StreamingResult stops demand
        ↓
ResultCursor cancellation
        ↓
Statement cancellation
        ↓
Driver cancellation
        ↓
Cleanup
```

según capabilities.

---

# 93. Stream timeout

Si el stream permanece abierto hasta su deadline:

```text
TIMED_OUT
```

aunque el SQL original haya terminado de producir datos server-side.

---

# 94. Idle timeout

Si:

```text
now - lastConsumerActivity
≥
IdleTimeout
```

el stream puede terminar:

```text
TIMED_OUT
```

con subtype:

```text
STREAM_IDLE_TIMEOUT
```

---

# 95. TimeoutKind

```php
enum TimeoutKind: string
{
    case CONNECTION_ACQUISITION = 'connection_acquisition';
    case EXECUTION = 'execution';
    case QUERY = 'query';
    case STATEMENT = 'statement';
    case FETCH = 'fetch';
    case STREAM = 'stream';
    case STREAM_IDLE = 'stream_idle';
    case TRANSACTION = 'transaction';
    case RESOURCE_WAIT = 'resource_wait';
}
```

---

# 96. TimeoutOutcome

```php
final readonly class TimeoutOutcome
{
    public function __construct(
        public TimeoutKind $kind,
        public Deadline $deadline,
        public TimeoutGuarantee $guarantee,
        public CancellationOutcome $cancellation,
        public ConnectionImpact $connectionImpact,
        public TransactionImpact $transactionImpact,
    ) {}
}
```

---

# 97. Connection impact

Después de cancellation/timeout, la connection puede quedar:

```text
HEALTHY
RESET_REQUIRED
INVALID
UNKNOWN
```

---

# 98. ConnectionImpact

```php
enum ConnectionImpact: string
{
    case NONE = 'none';
    case HEALTH_CHECK_REQUIRED = 'health_check_required';
    case RESET_REQUIRED = 'reset_required';
    case INVALIDATE = 'invalidate';
    case CONNECTION_LOST = 'connection_lost';
    case UNKNOWN = 'unknown';
}
```

---

# 99. Connection Manager integration

El sistema de timeout/cancellation no deberá devolver directamente una conexión al pool.

Deberá comunicar:

```text
ConnectionImpact
```

al `ConnectionLeaseManager`.

---

# 100. Example

```text
statement timeout
      ↓
driver says transaction aborted
      ↓
ConnectionImpact = RESET_REQUIRED
TransactionImpact = ABORTED
```

---

# 101. Unknown connection state

Cuando no pueda probarse que la connection es reusable:

```text
UNKNOWN
```

deberá tratarse conservadoramente.

---

# 102. Conservative rule

```text
Unknown Connection Safety
⇒
Do Not Return As Healthy
```

---

# 103. Transaction impact

Cancellation puede afectar una transaction activa.

---

# 104. TransactionImpact

```php
enum TransactionImpact: string
{
    case NONE = 'none';
    case STATEMENT_ABORTED_ONLY = 'statement_aborted_only';
    case TRANSACTION_MARKED_FAILED = 'transaction_marked_failed';
    case TRANSACTION_ABORTED = 'transaction_aborted';
    case COMMIT_OUTCOME_UNKNOWN = 'commit_outcome_unknown';
    case ROLLBACK_REQUIRED = 'rollback_required';
    case UNKNOWN = 'unknown';
}
```

---

# 105. Transaction ownership

El Cancellation System no deberá:

```text
commit
rollback
```

directamente salvo que actúe mediante el contrato explícito del `TransactionExecutionCoordinator`.

---

# 106. Caller-owned transaction

Si una query es cancelada dentro de una caller-owned transaction:

```text
Execution Engine
```

deberá reportar el impacto.

El caller/Transaction Manager mantiene autoridad sobre el lifecycle.

---

# 107. Engine-owned transaction

El Execution Engine podrá solicitar al Transaction Coordinator la acción definida por su policy.

---

# 108. Partial effects

Cancellation no implica necesariamente:

```text
no database effects
```

---

# 109. DML example

```sql
UPDATE accounts
SET status = ...
WHERE ...
```

Si ocurre timeout, el cliente puede no saber inmediatamente si:

```text
0 rows changed
all rows changed
transaction rolled back
server still processing
commit succeeded
```

---

# 110. Outcome certainty

Se necesita distinguir:

```text
SUCCESS
FAILURE
CANCELLED
TIMED_OUT
UNKNOWN
```

---

# 111. UNKNOWN

Debe utilizarse cuando:

```text
observable client state
```

no permite determinar con seguridad el resultado server-side.

---

# 112. Ambiguous commit

Caso crítico:

```text
COMMIT sent
    ↓
network lost
    ↓
response unavailable
```

No se puede asumir:

```text
rollback
```

ni:

```text
commit
```

Resultado:

```text
UNKNOWN
```

---

# 113. Cancellation during commit

Debe tratarse especialmente.

Una vez que `COMMIT` ha sido enviado:

```text
client cancellation
```

no implica que el commit no ocurrió.

---

# 114. Cancellation during rollback

Igualmente:

```text
rollback requested
+
connection lost
```

puede dejar outcome incierto.

---

# 115. Partial result delivery

Si un streaming result ya entregó:

```text
N > 0 rows
```

antes del timeout:

```text
partial observable effects exist
```

aunque sean efectos del lado del consumidor.

---

# 116. No fake atomicity

VoltStack no deberá presentar:

```text
timeout
```

como si ninguna fila hubiera sido observada cuando ya fueron entregadas.

---

# 117. Retry boundary

Cancellation/timeout no implica automáticamente retry.

---

# 118. Retry prerequisites

Cualquier retry futuro deberá considerar:

```text
failure classification
timeout kind
cancellation reason
idempotency
transaction state
partial effects
rows delivered
connection state
deadline remaining
retry policy
```

---

# 119. Expired deadline and retry

Si:

```text
RemainingTime ≤ 0
```

no deberá iniciarse retry.

---

# 120. Retry cannot reset user deadline

Nunca:

```text
query timeout = 5s
↓
fails after 4.9s
↓
retry gets another 5s
```

salvo que una política superior explícitamente defina un nuevo operation scope.

---

# 121. Deadline preservation across retry

Normalmente:

```text
RetryDeadline
=
OriginalDeadline
```

---

# 122. Cancellation and QueryExecutor

El `QueryExecutor` deberá verificar cancellation/deadline antes de despachar nuevas unidades.

---

# 123. Readiness formula

Extendiendo el documento 77:

```text
Ready(u)
=
HardDepsCompleted(u)
∧
InputsAvailable(u)
∧
AffinityPermits(u)
∧
ResourcesPermit(u)
∧
¬CancellationRequested
∧
DeadlineValid
```

---

# 124. Stop scheduling

Una vez solicitada cancellation:

```text
Pending Units
```

no deberán comenzar salvo unidades necesarias para:

```text
cleanup
rollback coordination
resource release
termination
```

---

# 125. Active units

Las unidades ya activas recibirán la señal de cancellation.

---

# 126. Dependent units

Podrán pasar a:

```text
CANCELLED
```

o:

```text
BLOCKED
```

según la causa.

---

# 127. Cleanup units

No deberán cancelarse simplemente porque el query fue cancelado.

---

# 128. Important principle

```text
Cancellation Stops Work.
Cancellation Must Not Stop Cleanup.
```

---

# 129. Cancellation and execution stages

Un stage:

```text
RUNNING
```

puede pasar a:

```text
CANCELLING
```

antes de:

```text
CANCELLED
```

si se necesita representar el proceso.

---

# 130. Suggested unit state extension

```php
enum UnitRuntimeState
{
    case PENDING;
    case READY;
    case RUNNING;
    case CANCELLING;
    case COMPLETED;
    case FAILED;
    case CANCELLED;
    case TIMED_OUT;
    case BLOCKED;
    case SKIPPED;
    case CLEANED;
}
```

---

# 131. Cancellation and framework execution

Operaciones framework-controlled deberán cooperar revisando tokens en puntos razonables.

Ejemplo:

```php
foreach ($items as $item) {
    $token->throwIfCancellationRequested();

    process($item);
}
```

---

# 132. Check frequency

No necesariamente:

```text
every machine instruction
```

pero sí suficientemente frecuente para cumplir la política de responsiveness.

---

# 133. Cancellation latency

Se puede modelar:

```text
CancellationLatency
=
CancellationObservedAt
-
CancellationRequestedAt
```

---

# 134. Server cancellation latency

Distinta de:

```text
LocalCancellationLatency
```

---

# 135. Cancellation responsiveness policy

```php
final readonly class CancellationResponsivenessPolicy
{
    public function __construct(
        public Duration $targetObservationLatency,
        public ?Duration $escalationAfter,
    ) {}
}
```

---

# 136. No hard guarantee without capability

Un target no deberá presentarse como garantía si el driver puede bloquear indefinidamente.

---

# 137. Driver cancellation adapter

```php
interface DriverCancellationAdapter
{
    public function capabilities(): DriverCancellationCapabilities;

    public function cancel(
        DriverOperationHandle $operation,
        DriverCancellationContext $context
    ): DriverCancellationResult;
}
```

---

# 138. DriverOperationHandle

Debe ser un handle runtime operation-scoped.

No debe formar parte de:

```text
CompiledDatabaseCommand
PreparedStatementBlueprint
ExecutionPlan
```

---

# 139. Driver cancellation context

Puede incluir:

```text
operation ID
connection lease
statement handle
deadline
cancellation reason
transaction context
security context
```

sin convertirse en service locator.

---

# 140. Server-side cancellation identifiers

Algunos motores pueden requerir:

```text
backend process ID
session ID
query ID
```

Estos identificadores serán:

```text
runtime-sensitive operational metadata
```

---

# 141. Security

No deberán exponerse indiscriminadamente.

---

# 142. Administrative cancellation

Cancelar queries de otra sesión puede requerir privilegios elevados.

VoltStack deberá tratarlo como una capability de seguridad explícita.

---

# 143. No arbitrary cross-session cancellation

Una aplicación no deberá poder cancelar operaciones ajenas simplemente conociendo un identificador.

---

# 144. Cancellation authority

```text
CancellationAuthority
```

deberá validarse antes de acciones administrativas.

---

# 145. Resource pressure cancellation

ResourceGovernor puede solicitar cancelación debido a:

```text
memory pressure
too many streams
connection exhaustion
worker shutdown
resource quota
```

---

# 146. Resource cancellation ≠ timeout

Aunque ambas terminen una operación:

```text
RESOURCE_PRESSURE
```

se mantiene como causa.

---

# 147. Request disconnect

En aplicaciones HTTP:

```text
client disconnect
```

puede convertirse en:

```text
REQUEST_DISCONNECTED
```

si el runtime puede detectarlo.

---

# 148. Runtime neutrality

El core no dependerá directamente de:

```text
FrankenPHP APIs
RoadRunner APIs
OpenSwoole APIs
```

---

# 149. RuntimeCancellationBridge

```php
interface RuntimeCancellationBridge
{
    public function bind(
        RuntimeRequestContext $runtime,
        CancellationSource $source
    ): RuntimeCancellationRegistration;
}
```

---

# 150. FrankenPHP

Adapter:

```text
FrankenPHP request lifecycle
        ↓
RuntimeCancellationBridge
        ↓
CancellationSource
```

---

# 151. RoadRunner

Mismo contrato:

```text
RoadRunner worker/request lifecycle
        ↓
CancellationSource
```

---

# 152. OpenSwoole

Podrá integrar:

```text
coroutine cancellation
request disconnect
timer
worker shutdown
```

mediante adapter.

---

# 153. No runtime-specific logic in QueryExecutor

No:

```php
if (FrankenPHP::requestDisconnected()) {
}
```

dentro del QueryExecutor.

---

# 154. Persistent runtime

Todo estado mutable de cancellation/deadline será:

```text
operation-scoped
```

---

# 155. Shared state allowed

Podrán compartirse:

```text
immutable policies
frozen registries
stateless coordinators
driver capability descriptors
```

---

# 156. Shared state forbidden

No compartir:

```text
current token
current deadline
current statement
current connection
current query
current transaction
current tenant
```

---

# 157. Request cleanup

Al terminar un request:

```text
Request Scope
    ↓
cancel owned executions
    ↓
cancel streams
    ↓
cleanup cursors
    ↓
cleanup statements
    ↓
classify connections
    ↓
release/invalidate connections
```

---

# 158. Worker shutdown

Debe existir un mecanismo para:

```text
stop accepting new database executions
↓
cancel/finish active operations according to policy
↓
cleanup resources
↓
shutdown worker
```

---

# 159. Graceful shutdown

Puede permitir:

```text
grace period
```

antes de escalación.

---

# 160. Hard shutdown

Si el proceso termina forzosamente, no se puede garantizar cleanup completo.

La arquitectura deberá distinguir:

```text
graceful termination
```

de:

```text
process destruction
```

---

# 161. Timeout precision

No se deberá prometer precisión temporal superior a la capacidad real del runtime/driver.

---

# 162. Timeout resolution

Puede modelarse:

```php
final readonly class TimeoutResolution
{
    public function __construct(
        public Duration $minimumResolution,
        public TimeoutGuarantee $guarantee,
    ) {}
}
```

---

# 163. Effective timeout

Si el driver sólo acepta segundos enteros y quedan:

```text
350ms
```

el sistema deberá aplicar una estrategia explícita.

No truncar silenciosamente a:

```text
0
```

si eso significa “sin timeout”.

---

# 164. Timeout conversion

La conversión de durations a driver units deberá ser:

```text
overflow-safe
underflow-safe
semantically explicit
```

---

# 165. Timeout zero semantics

Nunca asumir que:

```text
timeout = 0
```

significa universalmente lo mismo.

Puede significar:

```text
immediate timeout
no timeout
driver default
invalid
```

según driver.

La adaptación deberá resolverlo explícitamente.

---

# 166. Native timeout scope

Algunos motores aplican timeout a:

```text
session
transaction
statement
```

VoltStack deberá conocer el scope real.

---

# 167. Session mutation danger

Si para aplicar timeout el adapter modifica session state:

```text
SET ...
```

deberá restaurarse/resetearse correctamente antes de devolver la connection.

---

# 168. No hidden session leakage

```text
Request A timeout = 1s
↓
connection reused
↓
Request B unexpectedly inherits 1s
```

deberá ser imposible.

---

# 169. Prefer local/native scoped configuration

Cuando exista una opción:

```text
statement-local
```

será preferible a una modificación global de sesión.

---

# 170. Connection reset integration

Cualquier timeout/cancellation que modifique session state deberá registrarlo en:

```text
ConnectionState
```

para reset.

---

# 171. Cancellation and prepared statement cache

Un live prepared statement cancelado puede quedar:

```text
REUSABLE
RESET_REQUIRED
INVALID
UNKNOWN
```

---

# 172. StatementImpact

```php
enum StatementImpact: string
{
    case NONE = 'none';
    case REUSABLE = 'reusable';
    case RESET_REQUIRED = 'reset_required';
    case INVALIDATE = 'invalidate';
    case UNKNOWN = 'unknown';
}
```

---

# 173. Live statement cache rule

```text
UNKNOWN
⇒
do not return as healthy reusable statement
```

---

# 174. Cursor impact

Cancellation de cursor puede requerir:

```text
drain
close
discard
connection reset
```

según driver.

---

# 175. No universal drain

No asumir que consumir todas las filas pendientes siempre sea:

```text
safe
fast
necessary
```

---

# 176. Cancellation propagation graph

Conceptualmente:

```text
CancellationSource
       │
       ▼
ExecutionScope
       │
       ├────────► QueryExecutor
       │
       ├────────► Active Stage
       │
       ├────────► Active Unit
       │
       ├────────► StatementExecutor
       │
       ├────────► ResultCursor
       │
       └────────► StreamingResult
```

---

# 177. Dependency cancellation

Si una unidad `A` falla/cancela y `B` depende obligatoriamente de A:

```text
A CANCELLED
    ↓
B cannot execute
```

B puede clasificarse:

```text
BLOCKED_BY_CANCELLED_DEPENDENCY
```

en lugar de fingir que B fue activamente cancelada.

---

# 178. Cancellation propagation policy

```php
enum CancellationPropagationPolicy
{
    case CASCADE;
    case ISOLATED;
    case DEPENDENCY_AWARE;
}
```

---

# 179. CASCADE

Todos los children se cancelan.

---

# 180. ISOLATED

La cancellation local no se propaga fuera del scope.

---

# 181. DEPENDENCY_AWARE

La propagación sigue las relaciones del ExecutionPlan.

Éste deberá ser el modelo preferido para execution graphs complejos.

---

# 182. Recursive execution

Las regiones recursivas deberán revisar:

```text
cancellation
deadline
iteration budget
```

en cada boundary razonable.

---

# 183. Correlated execution

Una query correlacionada que ejecuta trabajo por outer row deberá respetar el deadline global.

---

# 184. No timeout reset per row

Nunca:

```text
for each outer row:
    give correlated subquery a fresh 5 seconds
```

si el scope superior sólo tenía cinco segundos totales.

---

# 185. Materialization

Operaciones de materialización framework-controlled deberán ser cancelables.

---

# 186. Spill operations

Si existe spill a disco:

```text
cancellation
```

deberá detener producción y limpiar temporary resources.

---

# 187. Cancellation during cleanup

Cleanup mismo puede encontrar cancellation global.

Sin embargo:

```text
cleanup-critical operations
```

deben disponer de una política especial.

---

# 188. Cleanup deadline

Puede existir un:

```text
CleanupDeadline
```

separado y acotado.

---

# 189. Why

Si el deadline principal ya expiró:

```text
remaining execution time = 0
```

pero todavía es necesario:

```text
close cursor
release statement
invalidate connection
```

---

# 190. Cleanup deadline ≠ execution extension

El cleanup deadline no permite continuar la query.

Sólo permite terminar recursos de forma segura.

---

# 191. Cleanup cancellation policy

```php
enum CleanupCancellationPolicy
{
    case BEST_EFFORT;
    case BOUNDED;
    case MUST_ATTEMPT;
}
```

---

# 192. Cleanup failure

No deberá reemplazar automáticamente:

```text
TimeoutException
```

por:

```text
CleanupException
```

---

# 193. Primary cause preservation

Ejemplo:

```text
Primary:
QueryTimedOutException

Suppressed:
CursorCloseException
ConnectionResetException
```

---

# 194. Error hierarchy

```text
QueryTerminationException
├── QueryCancellationException
│   ├── CancellationRequestedException
│   ├── DriverCancellationException
│   ├── ServerCancellationException
│   ├── CancellationPropagationException
│   └── CancellationInvariantException
│
├── QueryTimeoutException
│   ├── ConnectionAcquisitionTimeoutException
│   ├── ExecutionTimeoutException
│   ├── StatementTimeoutException
│   ├── FetchTimeoutException
│   ├── StreamTimeoutException
│   ├── StreamIdleTimeoutException
│   └── ResourceWaitTimeoutException
│
├── DeadlineException
│   ├── DeadlineExpiredException
│   ├── InvalidDeadlineException
│   ├── DeadlineCompositionException
│   └── DeadlineConversionException
│
├── CancellationCapabilityException
├── TimeoutCapabilityException
├── CancellationSecurityException
├── CancellationResourceException
├── CancellationTransactionException
├── CancellationConnectionException
├── AmbiguousTerminationException
└── QueryTerminationInvariantException
```

---

# 195. Normalized error context

Los errores podrán incluir:

```text
ExecutionId
StageId
ExecutionUnitId
StatementExecutionId
StreamId
TimeoutKind
CancellationReason
DeadlineOrigin
ConnectionImpact
TransactionImpact
StatementImpact
rows delivered
operation phase
guarantee level
```

---

# 196. Sensitive information

No incluir por defecto:

```text
parameter values
credentials
tokens
PII
full result rows
connection passwords
```

---

# 197. SQL diagnostics

Podrá incluirse SQL redacted/normalized según política de telemetry.

---

# 198. Timeout diagnostics

Ejemplo:

```text
execution_id      = exec-17
timeout_kind      = statement
configured        = 5s
elapsed           = 5.01s
guarantee         = server_enforced
connection_impact = reset_required
transaction       = aborted
```

---

# 199. Telemetry events

```text
CancellationRequested
CancellationPropagating
CancellationAcknowledged
CancellationCompleted
CancellationTooLate
CancellationFailed

DeadlineCreated
DeadlineApproaching
DeadlineExpired

TimeoutTriggered
TimeoutCancellationStarted
TimeoutCancellationCompleted

ConnectionInvalidatedAfterCancellation
TransactionAffectedByCancellation
StatementInvalidatedAfterCancellation

ExecutionCancelled
ExecutionTimedOut
```

---

# 200. Telemetry is observational

Telemetry no deberá decidir:

```text
whether query is cancelled
whether timeout applies
whether connection is reusable
```

---

# 201. Metrics

Ejemplos:

```text
db.execution.cancelled
db.execution.timeout
db.execution.cancellation_latency
db.execution.timeout_overrun
db.statement.timeout
db.connection.acquire_timeout
db.cursor.fetch_timeout
db.stream.timeout
db.stream.idle_timeout
db.cancellation.server_confirmed
db.cancellation.driver_confirmed
db.cancellation.connection_abort
db.cancellation.too_late
db.termination.unknown
```

---

# 202. Timeout overrun

Puede medirse:

```text
TimeoutOverrun
=
ActualTerminationTime
-
Deadline
```

---

# 203. Important interpretation

Un overrun no implica necesariamente bug.

Puede reflejar:

```text
driver cancellation latency
network delay
cleanup
runtime scheduling
```

---

# 204. Metrics cardinality

No utilizar:

```text
ExecutionId
QueryId
TenantId
UserId
raw SQL
```

como labels de métricas de alta cardinalidad.

---

# 205. Security model

Cancellation es una operación de control y potencialmente privilegiada.

---

# 206. Local cancellation

El owner de una ejecución puede cancelar su propia operación según política.

---

# 207. Cross-operation cancellation

Cancelar una ejecución ajena deberá requerir:

```text
explicit authority
```

---

# 208. Administrative cancellation

Puede integrarse posteriormente con:

```text
Authorization System
Administration System
Database Operations
```

sin hacerlos dependencias obligatorias del core.

---

# 209. Cancellation token confidentiality

Tokens/handles de cancellation no deberán convertirse en identificadores globales inseguros.

---

# 210. Tenant isolation

Un tenant nunca deberá cancelar accidentalmente una operación perteneciente a otro tenant.

---

# 211. Tenant scope invariant

```text
CancellationAuthority(Tenant A)
∩
Execution(Tenant B)
=
∅
```

salvo autoridad administrativa explícita.

---

# 212. Timeout policy security

El usuario no necesariamente podrá solicitar:

```text
timeout = infinite
```

si una política superior impone:

```text
max execution = 30s
```

---

# 213. Effective timeout policy

```text
EffectiveTimeout
=
min(
    ApplicationRequestedTimeout,
    FrameworkMaximum,
    TenantMaximum,
    ResourceGovernorMaximum,
    ParentDeadline
)
```

donde existan dichas políticas.

---

# 214. No timeout privilege escalation

Una capa inferior no podrá ampliar límites superiores.

---

# 215. Resource governance

Cancellation/timeout deberá integrarse con `ResourceGovernor`.

---

# 216. Resource wait

Esperar:

```text
connection
memory reservation
stream slot
temporary storage
```

deberá consumir el deadline.

---

# 217. No free waiting time

No:

```text
wait 10s for connection
+
then start 5s query timeout
```

si el contrato era:

```text
whole execution ≤ 5s
```

---

# 218. Execution deadline formula

```text
TotalExecutionTime
=
ResourceWait
+
ConnectionAcquire
+
Prepare
+
Bind
+
Execute
+
Fetch/Stream
+
FrameworkExecution
```

sujeto al mismo deadline global cuando corresponda.

---

# 219. Query-level vs operation-level timeout

Debe poder distinguirse un timeout sólo para:

```text
database statement
```

de uno para:

```text
whole execution plan
```

---

# 220. Default public semantics

Se recomienda que una API simple como:

```php
$query->timeout(seconds: 5);
```

se interprete como:

```text
query execution deadline
```

y no como un detalle específico de un driver.

---

# 221. Advanced APIs

Podrán configurar:

```text
connection acquisition timeout
statement timeout
fetch timeout
stream idle timeout
```

separadamente.

---

# 222. Query Builder boundary

El Query Builder puede expresar una política declarativa:

```text
timeout preference/requirement
```

pero no implementa timers ni cancellation.

---

# 223. Planner boundary

El Planner puede transportar requirements relacionados con execution constraints.

No ejecuta timeout.

---

# 224. Compiler boundary

El Compiler puede generar representación nativa de timeout sólo cuando forme parte legítima de la representación SQL/statement y ya haya sido decidida.

No inventa políticas.

---

# 225. Executor boundary

El Execution Engine aplica el runtime deadline/cancellation contract.

---

# 226. Driver boundary

El Driver implementa capacidades concretas.

---

# 227. Connection boundary

Connection Manager administra el impacto sobre conexión y reset/reuse.

---

# 228. Transaction boundary

Transaction Manager administra el impacto sobre transaction lifecycle.

---

# 229. Layering

```text
Application Policy
      ↓
Query/Execution Requirements
      ↓
Execution Context
      ↓
Timeout/Cancellation System
      ↓
Statement/Cursor/Stream
      ↓
Driver Capability Adapter
      ↓
Database
```

---

# 230. Cancellation registration

Los recursos activos pueden registrarse:

```php
interface CancellationRegistration
{
    public function unregister(): void;
}
```

---

# 231. Registration lifecycle

```text
resource becomes active
      ↓
register cancellation callback
      ↓
resource completes/closes
      ↓
unregister
```

---

# 232. Callback lifetime

No deberá sobrevivir al recurso asociado.

---

# 233. Memory leak prevention

Cancellation sources no deberán retener indefinidamente callbacks de operaciones completadas.

---

# 234. Callback execution

Los callbacks deberán ser:

```text
bounded
non-blocking where possible
exception-safe
```

---

# 235. Cancellation callback failure

Una callback que falle no deberá impedir intentar cancelar otros recursos.

---

# 236. Cancellation fan-out

```text
Cancel Execution
      ↓
├── Statement A
├── Cursor B
└── Framework Unit C
```

El fallo al cancelar A no deberá impedir intentar B y C.

---

# 237. Aggregated cancellation report

```php
final readonly class CancellationReport
{
    public function __construct(
        public array $attempts,
        public CancellationStatus $overallStatus,
        public ConnectionImpact $connectionImpact,
        public TransactionImpact $transactionImpact,
    ) {}
}
```

---

# 238. Deterministic cancellation ordering

Cuando sea relevante, cancellation deberá seguir orden consistente.

Ejemplo:

```text
consumer demand
↓
cursor
↓
statement
↓
connection abort fallback
```

---

# 239. Cleanup ordering remains separate

Cancellation ordering no sustituye el cleanup dependency graph.

---

# 240. Testing strategy

El sistema deberá cubrir al menos:

```text
cancellation before execution
cancellation during connection wait
cancellation during prepare
cancellation during bind
cancellation during execute
cancellation during fetch
cancellation during streaming
cancellation after completion
timeout before execution
timeout during execute
timeout during fetch
stream idle timeout
transaction cancellation
connection loss during cancellation
driver cancellation failure
server cancellation unsupported
worker shutdown
request disconnect
nested deadlines
deadline race
completion/cancellation race
cleanup after timeout
persistent worker isolation
```

---

# 241. Cancellation before start test

```text
token already cancelled
↓
execute()
```

deberá:

```text
acquire no unnecessary resources
execute no statement
return/throw cancellation outcome
```

---

# 242. Deadline already expired test

```text
deadline < now
```

deberá fallar antes de resource acquisition cuando sea posible.

---

# 243. Cancellation during connection wait test

Verificar:

```text
pool wait stops
no connection leaked
execution cancelled
```

---

# 244. Cancellation during execute test

Con driver cancellable:

```text
cancel
↓
driver cancellation
↓
statement outcome classified
↓
connection classified
↓
transaction classified
↓
cleanup
```

---

# 245. Unsupported cancellation test

Con driver no cancellable:

VoltStack deberá reportar:

```text
CancellationGuarantee = REQUEST_ONLY
```

o equivalente real.

---

# 246. Completion race test

Simular:

```text
query completion
||
cancellation request
```

y verificar una única terminal transition.

---

# 247. Timeout race test

Simular:

```text
query completes exactly near deadline
```

La resolución deberá ser determinista según evidencia/ordering definido.

---

# 248. Streaming partial timeout test

```text
100 rows delivered
↓
deadline expires
```

deberá producir:

```text
TIMED_OUT
rowsDelivered = 100
fullyConsumed = false
```

---

# 249. Transaction ambiguity test

```text
COMMIT
↓
network loss
```

deberá poder producir:

```text
UNKNOWN
```

---

# 250. Persistent worker test

```text
Request A timeout
↓
cleanup/reset
↓
Request B
```

Request B no deberá heredar:

```text
token
deadline
driver timeout
session timeout
transaction state
```

de A.

---

# 251. Nested deadline test

```text
Parent deadline = T+5s
Child requested = T+30s
```

resultado:

```text
Child effective deadline = T+5s
```

---

# 252. Driver timeout conversion test

Probar:

```text
milliseconds → seconds
overflow
underflow
zero semantics
maximum supported value
```

---

# 253. Cleanup timeout test

El deadline de ejecución puede expirar, pero cleanup deberá intentarse dentro de su política acotada.

---

# 254. Cancellation callback test

Una callback defectuosa no impedirá cancelar los demás recursos.

---

# 255. Performance requirements

En hot path:

```text
token->isCancellationRequested()
deadline check
```

deberán ser operaciones de muy bajo costo.

---

# 256. Complexity

Normalmente:

```text
CancellationCheck = O(1)
DeadlineCheck = O(1)
```

---

# 257. No heavy service lookup

No realizar en cada fetch:

```text
container->get(...)
config repository lookup
global tenant lookup
reflection
```

---

# 258. Deadline caching

El deadline efectivo puede calcularse una vez por scope.

No recalcular toda la jerarquía en cada row si no cambia.

---

# 259. Cancellation tree efficiency

La propagación deberá evitar recorrer estructuras globales no relacionadas.

---

# 260. Suggested namespace

```text
VoltStack\Quantum\Database\Execution\Termination
```

con subdominios:

```text
Cancellation
Timeout
Deadline
Impact
Runtime
Telemetry
Diagnostic
Exception
```

---

# 261. Directory structure

```text
VoltStack/
└── Quantum/
    └── Database/
        └── Execution/
            └── Termination/
                ├── Cancellation/
                │   ├── Contract/
                │   │   ├── CancellationToken.php
                │   │   ├── CancellationSource.php
                │   │   ├── CancellationCoordinator.php
                │   │   └── CancellationRegistration.php
                │   ├── CancellationReason.php
                │   ├── CancellationState.php
                │   ├── CancellationStatus.php
                │   ├── CancellationUrgency.php
                │   ├── CancellationGuarantee.php
                │   ├── CancellationCapability.php
                │   ├── CancellationScope.php
                │   ├── CancellationScopeId.php
                │   ├── CancellationPropagationPolicy.php
                │   ├── CancellationOutcome.php
                │   ├── CancellationReport.php
                │   └── DefaultCancellationCoordinator.php
                │
                ├── Deadline/
                │   ├── Deadline.php
                │   ├── DeadlineScope.php
                │   ├── DeadlineScopeId.php
                │   ├── DeadlineOrigin.php
                │   ├── DeadlineClock.php
                │   ├── DeadlineCheckResult.php
                │   ├── DeadlineScheduler.php
                │   └── DeadlineRegistration.php
                │
                ├── Timeout/
                │   ├── TimeoutPolicy.php
                │   ├── TimeoutKind.php
                │   ├── TimeoutGuarantee.php
                │   ├── TimeoutEnforcementPolicy.php
                │   ├── TimeoutOutcome.php
                │   ├── TimeoutCoordinator.php
                │   ├── TimeoutResolution.php
                │   └── DefaultTimeoutCoordinator.php
                │
                ├── Clock/
                │   ├── MonotonicClock.php
                │   ├── MonotonicInstant.php
                │   └── Duration.php
                │
                ├── Driver/
                │   ├── DriverCancellationAdapter.php
                │   ├── DriverCancellationCapabilities.php
                │   ├── DriverCancellationContext.php
                │   ├── DriverCancellationResult.php
                │   └── DriverOperationHandle.php
                │
                ├── Impact/
                │   ├── ConnectionImpact.php
                │   ├── TransactionImpact.php
                │   ├── StatementImpact.php
                │   └── TerminationEvidence.php
                │
                ├── State/
                │   ├── ExecutionTerminalState.php
                │   ├── TerminalStateCoordinator.php
                │   └── TerminalTransitionOutcome.php
                │
                ├── Cleanup/
                │   ├── CleanupDeadline.php
                │   └── CleanupCancellationPolicy.php
                │
                ├── Runtime/
                │   ├── RuntimeCancellationBridge.php
                │   └── RuntimeCancellationRegistration.php
                │
                ├── Security/
                │   ├── CancellationAuthority.php
                │   └── CancellationSecurityPolicy.php
                │
                ├── Telemetry/
                │   ├── TerminationTelemetry.php
                │   ├── CancellationTelemetry.php
                │   └── TimeoutTelemetry.php
                │
                ├── Diagnostic/
                │   ├── TerminationDiagnostic.php
                │   ├── CancellationDiagnostic.php
                │   └── TimeoutDiagnostic.php
                │
                └── Exception/
                    ├── QueryTerminationException.php
                    ├── QueryCancellationException.php
                    ├── QueryTimeoutException.php
                    ├── DeadlineException.php
                    ├── DeadlineExpiredException.php
                    ├── InvalidDeadlineException.php
                    ├── DeadlineCompositionException.php
                    ├── DeadlineConversionException.php
                    ├── ConnectionAcquisitionTimeoutException.php
                    ├── ExecutionTimeoutException.php
                    ├── StatementTimeoutException.php
                    ├── FetchTimeoutException.php
                    ├── StreamTimeoutException.php
                    ├── StreamIdleTimeoutException.php
                    ├── ResourceWaitTimeoutException.php
                    ├── DriverCancellationException.php
                    ├── ServerCancellationException.php
                    ├── CancellationPropagationException.php
                    ├── CancellationCapabilityException.php
                    ├── TimeoutCapabilityException.php
                    ├── CancellationSecurityException.php
                    ├── AmbiguousTerminationException.php
                    └── QueryTerminationInvariantException.php
```

---

# 262. Architectural invariants

## DB-TERM-001

Cancellation será distinta de timeout.

## DB-TERM-002

Cancellation será distinta de failure.

## DB-TERM-003

Timeout será distinto de failure.

## DB-TERM-004

Timeout será distinto de successful completion.

## DB-TERM-005

Cancellation será distinta de successful completion.

## DB-TERM-006

Timeout representará deadline expiration.

## DB-TERM-007

Cancellation representará una solicitud explícita de terminación.

## DB-TERM-008

Timeout podrá utilizar cancellation internamente sin perder su identidad.

## DB-TERM-009

CancellationToken será distinto de CancellationSource.

## DB-TERM-010

Observar cancellation no otorgará autoridad para cancelarla.

## DB-TERM-011

Cancellation request será idempotente.

## DB-TERM-012

La primera causa efectiva se preservará como causa primaria.

## DB-TERM-013

Cancellation será operation-scoped.

## DB-TERM-014

No existirá global current cancellation token.

## DB-TERM-015

Deadlines serán operation-scoped.

## DB-TERM-016

No existirá global current deadline.

## DB-TERM-017

Timeout durations se convertirán internamente a deadlines cuando sea posible.

## DB-TERM-018

Deadlines usarán monotonic time para enforcement.

## DB-TERM-019

Wall clock no será la fuente principal de execution duration.

## DB-TERM-020

Child deadline no extenderá parent deadline.

## DB-TERM-021

Effective deadline será el mínimo de los límites aplicables.

## DB-TERM-022

Resource waiting consumirá el deadline global cuando corresponda.

## DB-TERM-023

Retry no reiniciará silenciosamente el deadline.

## DB-TERM-024

Connection acquisition timeout será distinto de query timeout.

## DB-TERM-025

Statement timeout será distinto de fetch timeout.

## DB-TERM-026

Stream timeout será distinto de stream idle timeout.

## DB-TERM-027

Timeout capability será explícita.

## DB-TERM-028

Cancellation capability será explícita.

## DB-TERM-029

Capabilities no se inferirán mediante vendor conditionals dispersos.

## DB-TERM-030

Server-side cancellation no se afirmará sin evidencia.

## DB-TERM-031

Client-side interruption no se presentará como server-side cancellation.

## DB-TERM-032

Framework-observed timeout no implicará automáticamente server termination.

## DB-TERM-033

Cancellation strategy será capability-aware.

## DB-TERM-034

Cancellation escalation será policy-driven.

## DB-TERM-035

Connection abort no será fallback destructivo automático.

## DB-TERM-036

CancellationUrgency no sustituirá security/ownership rules.

## DB-TERM-037

Cancellation completion será distinta de cancellation request.

## DB-TERM-038

TOO_LATE será un resultado válido de cancellation race.

## DB-TERM-039

Sólo existirá una terminal transition efectiva.

## DB-TERM-040

Terminal state no implicará cleanup completion.

## DB-TERM-041

Cleanup continuará después de cancellation.

## DB-TERM-042

Cleanup continuará después de timeout.

## DB-TERM-043

Cancellation no cancelará cleanup crítico arbitrariamente.

## DB-TERM-044

Cleanup tendrá política temporal propia cuando sea necesario.

## DB-TERM-045

Cleanup deadline no permitirá continuar ejecución normal.

## DB-TERM-046

Primary termination cause será preservada.

## DB-TERM-047

Cleanup failures serán secundarios frente al primary failure/timeout/cancellation.

## DB-TERM-048

QueryExecutor verificará cancellation antes de iniciar nuevas unidades.

## DB-TERM-049

QueryExecutor verificará deadline antes de iniciar nuevas unidades.

## DB-TERM-050

Pending work no comenzará después de cancellation salvo termination/cleanup work.

## DB-TERM-051

Active work recibirá cancellation según capabilities.

## DB-TERM-052

Blocked dependency será distinguible de active cancellation.

## DB-TERM-053

Framework-controlled loops serán cooperativamente cancelables.

## DB-TERM-054

Driver-blocked operations no se declararán instantáneamente cancelables sin capability.

## DB-TERM-055

Cancellation latency será observable.

## DB-TERM-056

Timeout overrun será observable.

## DB-TERM-057

DriverOperationHandle será runtime-only.

## DB-TERM-058

DriverOperationHandle no formará parte del ExecutionPlan.

## DB-TERM-059

DriverOperationHandle no formará parte del CompiledDatabaseCommand.

## DB-TERM-060

DriverOperationHandle no formará parte del PreparedStatementBlueprint.

## DB-TERM-061

Server cancellation identifiers serán tratados como operational-sensitive metadata.

## DB-TERM-062

Cross-session cancellation requerirá authority.

## DB-TERM-063

Cross-tenant cancellation estará prohibida salvo authority explícita.

## DB-TERM-064

ResourceGovernor podrá solicitar cancellation.

## DB-TERM-065

Resource pressure cancellation será distinta de timeout.

## DB-TERM-066

Request disconnect podrá propagarse mediante runtime adapter.

## DB-TERM-067

Core no dependerá directamente de FrankenPHP.

## DB-TERM-068

Core no dependerá directamente de RoadRunner.

## DB-TERM-069

Core no dependerá directamente de OpenSwoole.

## DB-TERM-070

Runtime bridges implementarán integración específica.

## DB-TERM-071

Persistent workers no compartirán mutable cancellation state.

## DB-TERM-072

Persistent workers no compartirán mutable deadline state.

## DB-TERM-073

Request B no heredará timeout de Request A.

## DB-TERM-074

Request B no heredará cancellation token de Request A.

## DB-TERM-075

Request B no heredará driver session timeout de Request A.

## DB-TERM-076

Request termination cancelará owned executions según policy.

## DB-TERM-077

Worker shutdown tendrá termination policy explícita.

## DB-TERM-078

Graceful shutdown será distinto de forced process termination.

## DB-TERM-079

Timeout precision no excederá capacidades reales.

## DB-TERM-080

Timeout conversion será overflow-safe.

## DB-TERM-081

Timeout conversion será underflow-safe.

## DB-TERM-082

Timeout zero semantics serán driver-aware.

## DB-TERM-083

Session-level timeout mutations serán restauradas/reset.

## DB-TERM-084

Timeout session state no podrá filtrarse entre requests.

## DB-TERM-085

Statement impact será clasificado después de cancellation.

## DB-TERM-086

Unknown statement safety impedirá reuse optimista.

## DB-TERM-087

Connection impact será clasificado después de cancellation.

## DB-TERM-088

Unknown connection safety impedirá healthy reuse optimista.

## DB-TERM-089

Transaction impact será clasificado después de cancellation.

## DB-TERM-090

Cancellation System no hará commit arbitrario.

## DB-TERM-091

Cancellation System no hará rollback arbitrario.

## DB-TERM-092

Caller-owned transaction mantendrá lifecycle authority.

## DB-TERM-093

Partial database effects serán representables.

## DB-TERM-094

Ambiguous transaction outcome será representable.

## DB-TERM-095

Cancellation durante commit podrá producir UNKNOWN.

## DB-TERM-096

Timeout durante commit podrá producir UNKNOWN.

## DB-TERM-097

Cancellation no implicará ausencia de database effects.

## DB-TERM-098

Timeout no implicará ausencia de database effects.

## DB-TERM-099

Streaming partial delivery será preservada en termination metadata.

## DB-TERM-100

Timeout después de rows delivered no fingirá atomic failure.

## DB-TERM-101

Cancellation después de rows delivered no fingirá zero delivery.

## DB-TERM-102

Retry no será responsabilidad de este sistema.

## DB-TERM-103

Cancellation no implicará retry.

## DB-TERM-104

Timeout no implicará retry.

## DB-TERM-105

Retry eligibility deberá considerar partial effects.

## DB-TERM-106

Expired deadline impedirá retry normal.

## DB-TERM-107

Cancellation registration tendrá lifecycle explícito.

## DB-TERM-108

Completed resources eliminarán cancellation registrations.

## DB-TERM-109

Cancellation callbacks serán exception-safe.

## DB-TERM-110

Una callback fallida no impedirá cancelar otros recursos.

## DB-TERM-111

Cancellation fan-out será bounded al scope relevante.

## DB-TERM-112

Cancellation ordering será determinista cuando importe.

## DB-TERM-113

Cancellation ordering será distinto de cleanup ordering.

## DB-TERM-114

Recursive execution respetará cancellation.

## DB-TERM-115

Recursive execution respetará deadline.

## DB-TERM-116

Correlated execution no reiniciará deadline por outer row.

## DB-TERM-117

Materialization será cancelable cuando sea framework-controlled.

## DB-TERM-118

Temporary spill resources serán limpiados tras cancellation.

## DB-TERM-119

StreamingResult heredará cancellation/deadline scope apropiado.

## DB-TERM-120

Streaming timeout mantendrá TIMED_OUT como causa semántica.

## DB-TERM-121

Stream idle timeout será explícitamente distinguible.

## DB-TERM-122

Prepared statement blueprint no almacenará operation deadline.

## DB-TERM-123

Compiled artifacts no almacenarán mutable cancellation state.

## DB-TERM-124

Parameter bindings no contendrán cancellation tokens como SQL values.

## DB-TERM-125

Cancellation telemetry será observational.

## DB-TERM-126

Timeout telemetry será observational.

## DB-TERM-127

Telemetry no decidirá termination semantics.

## DB-TERM-128

Sensitive parameter values no serán incluidos por defecto en diagnostics.

## DB-TERM-129

Cancellation metrics tendrán cardinalidad controlada.

## DB-TERM-130

Execution IDs no serán metric labels de alta cardinalidad.

## DB-TERM-131

Cancellation authority será security-sensitive.

## DB-TERM-132

Administrative cancellation será explícitamente autorizada.

## DB-TERM-133

User timeout no podrá ampliar framework maximum.

## DB-TERM-134

Tenant timeout no podrá ampliar parent deadline.

## DB-TERM-135

Resource policy podrá reducir effective deadline.

## DB-TERM-136

Una capa inferior no ampliará límites superiores.

## DB-TERM-137

Query Builder sólo declarará timeout requirements/policy.

## DB-TERM-138

Planner no ejecutará timers.

## DB-TERM-139

Compiler no implementará runtime cancellation.

## DB-TERM-140

Executor aplicará runtime timeout/cancellation.

## DB-TERM-141

Driver implementará capacidades concretas.

## DB-TERM-142

Connection Manager decidirá reuse/reset/invalidation basado en impacto.

## DB-TERM-143

Transaction Manager conservará transaction lifecycle authority.

## DB-TERM-144

No habrá hidden reconnect después de cancellation.

## DB-TERM-145

No habrá hidden query rerun después de timeout.

## DB-TERM-146

No habrá hidden query rerun después de cancellation.

## DB-TERM-147

No habrá false server-cancellation claims.

## DB-TERM-148

No habrá false timeout precision claims.

## DB-TERM-149

No habrá false atomicity claims.

## DB-TERM-150

No habrá false successful-completion claims.

---

# 263. Invariante maestro de deadline

Para una operación `O` con deadline efectivo `D`:

```text
StartNewWork(O, t)
is permitted only if
t < D
```

salvo trabajo explícito de termination/cleanup.

---

# 264. Invariante maestro de composición

```text
EffectiveDeadline
=
min(ApplicableDeadlines)
```

---

# 265. Invariante maestro de propagación

```text
Cancel(ParentScope)
⇒
Cancel(DependentChildScopes)
```

según la política explícita del grafo de ejecución.

---

# 266. Invariante maestro de cleanup

```text
Cancellation ∨ Timeout ∨ Failure
⇒
CleanupAttempted
```

---

# 267. Invariante maestro de resource safety

```text
Termination
⇒
Every Live Resource
is eventually
Released ∨ Transferred ∨ Invalidated
```

---

# 268. Invariante maestro de connection safety

```text
ConnectionState = UNKNOWN
⇒
Connection cannot be returned as HEALTHY
```

---

# 269. Invariante maestro de transaction safety

```text
TransactionOutcome = UNKNOWN
⇒
VoltStack must not invent COMMITTED or ROLLED_BACK
```

---

# 270. Invariante maestro de cancellation honesty

```text
CancellationRequested
≠
CancellationConfirmed
```

---

# 271. Invariante maestro de timeout honesty

```text
DeadlineExpired
≠
ServerOperationConfirmedStopped
```

---

# 272. Invariante maestro de streaming

Si:

```text
RowsDelivered = N > 0
```

antes de timeout/cancellation:

```text
PartialDelivery = true
```

---

# 273. Invariante maestro de retry boundary

```text
Termination
+
UnknownEffects
⇒
NoBlindRetry
```

---

# 274. Flujo completo de timeout

```text
ExecutionRequest
      ↓
ExecutionContext
      ↓
Effective Deadline
      ↓
QueryExecutor
      ↓
StatementExecutor
      ↓
Driver Operation
      │
      ├─────────────── query completes
      │                       ↓
      │                    SUCCESS
      │
      └── deadline expires
               ↓
         TimeoutCoordinator
               ↓
       CancellationCoordinator
               ↓
        Driver Cancellation
               ↓
       classify statement
               ↓
       classify connection
               ↓
      classify transaction
               ↓
            cleanup
               ↓
          TIMED_OUT
```

---

# 275. Flujo completo de cancellation

```text
Caller / Runtime / ResourceGovernor
               ↓
        CancellationSource
               ↓
        CancellationToken
               ↓
      ExecutionCancellationScope
               ↓
      CancellationCoordinator
               ↓
      ┌────────┼──────────┐
      ▼        ▼          ▼
QueryExecutor Statement  Stream
      │        │          │
      └────────┼──────────┘
               ▼
          Driver Adapter
               ↓
       cancellation attempt
               ↓
      impact classification
               ↓
             cleanup
               ↓
           CANCELLED
```

---

# 276. Race model

```text
                     ┌── Completion
RUNNING ─────────────┼── Failure
                     ├── Cancellation
                     └── Timeout
```

Un `TerminalStateCoordinator` deberá resolver exactamente una transición semántica terminal válida.

---

# 277. Arquitectura final

```text
              QUERY TIMEOUT & CANCELLATION SYSTEM

                    ExecutionContext
                          │
            ┌─────────────┴──────────────┐
            │                            │
            ▼                            ▼
    Cancellation Scope              Deadline Scope
            │                            │
            ▼                            ▼
 CancellationCoordinator          TimeoutCoordinator
            │                            │
            └─────────────┬──────────────┘
                          ▼
                  Termination Control
                          │
          ┌───────────────┼────────────────┐
          ▼               ▼                ▼
     QueryExecutor   StatementExecutor   StreamingResult
          │               │                │
          └───────────────┼────────────────┘
                          ▼
                  Driver Cancellation
                          │
                          ▼
                       Database
                          │
                          ▼
               Termination Evidence
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
        Statement     Connection   Transaction
          Impact        Impact       Impact
             └────────────┼────────────┘
                          ▼
                        Cleanup
                          │
                          ▼
                  Terminal Outcome
```

---

# 278. Fórmula arquitectónica

```text
Query Timeout & Cancellation System
=
Cancellation Tokens
+
Cancellation Sources
+
Cancellation Scopes
+
Deadline Model
+
Monotonic Clock
+
Deadline Composition
+
Timeout Policies
+
Driver Capabilities
+
Server-side Cancellation
+
Client-side Interruption
+
Cancellation Propagation
+
Terminal State Arbitration
+
Connection Impact Classification
+
Transaction Impact Classification
+
Partial Effect Awareness
+
Streaming Termination
+
Deterministic Cleanup
+
Security
+
Telemetry
+
Persistent Runtime Isolation
```

---

# 279. Fórmula de corrección

```text
CorrectTermination
=
CausePreservation
∧
DeadlineCorrectness
∧
CancellationPropagation
∧
SingleTerminalOutcome
∧
ResourceCleanup
∧
ConnectionSafety
∧
TransactionSafety
∧
PartialEffectTransparency
∧
SecurityIsolation
∧
PersistentRuntimeIsolation
```

---

# 280. Principio final

> **A timeout tells VoltStack that time expired. A cancellation tells VoltStack that execution should stop. Neither one, by itself, proves what happened on the database server.**

Por tanto:

```text
Safe Query Termination
=
Explicit Cause
+
Explicit Scope
+
Explicit Deadline
+
Capability-aware Interruption
+
Impact Classification
+
Deterministic Cleanup
```

y nunca:

```text
throw TimeoutException
=
assume query stopped
=
assume transaction rolled back
=
return connection to pool
```

---

# 281. Siguiente documento

```text
85_DATABASE_EXECUTION_ERROR_SYSTEM.md
```

El siguiente documento deberá formalizar el sistema completo de errores de ejecución de VoltStack, incluyendo:

```text
Execution Error Taxonomy
Driver Error Normalization
SQLSTATE Preservation
Native Error Preservation
Connection Errors
Statement Errors
Binding Errors
Constraint Violations
Deadlocks
Serialization Failures
Timeout Errors
Cancellation Errors
Cursor Errors
Streaming Errors
Transaction Impact
Connection Impact
Unknown Outcome
Partial Effects
Primary vs Suppressed Errors
Error Context
Error Source Mapping
Compiled SQL Source Maps
Sensitive Data Redaction
Retry Classification
Error Recoverability
Error Severity
Error Codes
Framework Exception Hierarchy
Driver Exception Adapters
Telemetry
Diagnostics
Persistent Runtime Safety
```

manteniendo como principio:

```text
Normalize Error Semantics
≠
Destroy Native Error Information
```

y:

```text
Execution Error
=
Normalized Framework Meaning
+
Preserved Native Cause
+
Execution Context
+
Impact Classification
+
Recovery Metadata
```