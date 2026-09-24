# 238_DATABASE_RETRY_POLICY_SYSTEM.md

# VoltStack Quantum Database
## Database Retry Policy System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 238 — Database Retry Policy System  
**Bloque:** 23 — Resilience  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `237_DATABASE_QUERY_FAILURE_HANDLING_SYSTEM.md`  
**Siguiente documento:** `239_DATABASE_CIRCUIT_BREAKER_INTEGRATION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura central mediante la cual VoltStack decide **si una operación de base de datos puede volver a intentarse, qué debe repetirse, cuándo hacerlo, cuántas veces, sobre qué recurso y bajo qué garantías semánticas**.

El sistema será compartido por:

```text
Connection recovery
Query execution
Transaction replay
Replica routing
Failover
Distributed execution
Bulk processing
Chunk processing
Import/Export
Large dataset processing
```

La regla fundamental será:

> **Un retry no es una reacción automática ante una excepción; es una nueva ejecución autorizada únicamente cuando VoltStack puede justificar que el intento es útil, acotado y semánticamente seguro.**

Formalmente:

```text
Failure
≠
Retry
```

y:

```text
Transient Failure
≠
Safe Retry
```

---

# 2. Problema

Una implementación ingenua suele hacer:

```php
try {
    return $operation();
} catch (Throwable $e) {
    sleep(1);

    return $operation();
}
```

Esto es insuficiente.

Una operación puede haber:

```text
no comenzado
comenzado
terminado con error
terminado exitosamente
producido efectos parciales
producido efectos cuyo resultado se desconoce
```

Por tanto:

```text
Retry Decision
```

requiere más información que:

```text
Exception Type
```

---

# 3. Objetivos

El Retry Policy System deberá responder:

```text
¿El error es transitorio?

¿Existe evidencia suficiente del resultado anterior?

¿La operación puede reproducirse?

¿Qué unidad debe reproducirse?

¿Statement?

¿Transaction?

¿Connection acquisition?

¿Chunk?

¿Shard operation?

¿Debe utilizarse otro endpoint?

¿Debe mantenerse el mismo endpoint?

¿Cuántos intentos quedan?

¿Cuánto tiempo queda?

¿Qué backoff corresponde?

¿Debe aplicarse jitter?

¿Existe un circuit breaker abierto?

¿La consistencia permite cambiar de réplica?

¿La operación está ligada a un shard?

¿Existe riesgo de duplicar efectos?

¿Existe UNKNOWN outcome?

¿Quién es propietario del retry?
```

---

# 4. Principio arquitectónico

VoltStack separará:

```text
Failure Detection
        ↓
Failure Classification
        ↓
Outcome Resolution
        ↓
Replay Safety Analysis
        ↓
Retry Eligibility
        ↓
Retry Policy
        ↓
Retry Scheduling
        ↓
Attempt Execution
```

Ninguna de estas fases deberá colapsarse innecesariamente.

---

# 5. Retry Policy ≠ Failure Classifier

El Failure Classifier responde:

```text
¿Qué ocurrió?
```

El Retry Policy responde:

```text
¿Qué puede hacerse después?
```

---

# 6. Retry Policy ≠ Retry Executor

La política decide.

El executor ejecuta.

```text
RetryPolicy
    ↓
RetryDecision
    ↓
RetryCoordinator
    ↓
Operation Executor
```

---

# 7. Retry Policy ≠ Backoff Strategy

Backoff solamente determina:

```text
cuándo realizar el siguiente intento
```

No determina:

```text
si el retry es correcto
```

---

# 8. Retry Policy ≠ Circuit Breaker

Retry intenta recuperar una operación.

Circuit Breaker evita insistir sobre un recurso que presenta fallos persistentes.

```text
Retry
→ local operation recovery

Circuit Breaker
→ systemic failure containment
```

---

# 9. Retry Policy ≠ Failover

Retry responde:

```text
¿debemos volver a intentar?
```

Failover responde:

```text
¿debemos cambiar el recurso responsable?
```

---

# 10. Arquitectura general

```text
Database Operation
       │
       ▼
    Attempt
       │
       ▼
   Execution
       │
       ▼
    Failure
       │
       ▼
Failure Classification
       │
       ▼
Outcome Resolution
       │
       ▼
Replay Safety Analyzer
       │
       ▼
Retry Eligibility Analyzer
       │
       ▼
Retry Policy
       │
       ▼
Retry Decision
       │
 ┌─────┼───────────┐
 │     │           │
STOP  RETRY     ESCALATE
       │
       ▼
Retry Scheduler
       │
       ▼
Backoff + Jitter
       │
       ▼
Retry Coordinator
       │
       ▼
Next Attempt
```

---

# 11. RetryOperationId

Cada operación lógica tendrá una identidad estable:

```php
final readonly class RetryOperationId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 12. AttemptId

Cada ejecución física será diferente:

```php
final readonly class RetryAttemptId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Relación:

```text
Operation A
├── Attempt 1
├── Attempt 2
└── Attempt 3
```

---

# 13. Logical operation ≠ physical attempt

Esto es crítico para:

```text
telemetry
audit
timeouts
budgets
metrics
idempotency
```

---

# 14. RetryContext

```php
final readonly class RetryContext
{
    public function __construct(
        public RetryOperationId $operationId,
        public RetryAttempt $attempt,
        public RetryScope $scope,
        public RetryBudget $budget,
        public Deadline $deadline,
        public ReplaySafety $replaySafety,
        public RetryExecutionContext $execution,
        public ?DatabaseFailure $failure,
    ) {}
}
```

---

# 15. RetryAttempt

```php
final readonly class RetryAttempt
{
    public function __construct(
        public RetryAttemptId $id,
        public int $number,
        public Instant $startedAt,
    ) {}
}
```

Convención:

```text
attempt 1 = ejecución inicial
attempt 2 = primer retry
attempt 3 = segundo retry
```

---

# 16. RetryScope

```php
enum RetryScope
{
    case CONNECTION_ACQUISITION;

    case STATEMENT;

    case QUERY;

    case TRANSACTION;

    case CHUNK;

    case BATCH;

    case SHARD_OPERATION;

    case DISTRIBUTED_OPERATION;

    case CUSTOM;
}
```

---

# 17. Retry scope es obligatorio

No será válido decir simplemente:

```text
retry
```

Debe conocerse:

```text
retry what?
```

---

# 18. Statement retry

Repite:

```text
one database statement
```

---

# 19. Query retry

Puede reconstruir:

```text
query plan
connection
prepared statement
execution
```

según contexto.

---

# 20. Transaction retry

Reproduce:

```text
entire transaction callback/unit
```

desde un boundary seguro.

---

# 21. Connection acquisition retry

Solamente intenta obtener una conexión válida.

No implica repetir ninguna operación previamente enviada.

---

# 22. Chunk retry

Repite una unidad delimitada de procesamiento.

Debe respetar las reglas de:

```text
201_DATABASE_CHUNK_PROCESSING_SYSTEM.md
```

---

# 23. Batch retry

Puede repetir:

```text
batch unit
```

solo si se conoce el progreso y la seguridad de replay.

---

# 24. Distributed retry

Debe modelarse por:

```text
shard
endpoint
operation
outcome
```

Nunca como retry global ciego.

---

# 25. Retry ownership

Toda operación tendrá:

```text
RetryOwner
```

---

# 26. RetryOwner

```php
enum RetryOwner
{
    case NONE;

    case CONNECTION_MANAGER;

    case QUERY_EXECUTOR;

    case TRANSACTION_MANAGER;

    case CHUNK_RUNNER;

    case BATCH_RUNNER;

    case DISTRIBUTED_COORDINATOR;

    case APPLICATION;

    case CUSTOM;
}
```

---

# 27. Single retry owner

Regla:

> **Para una misma failure boundary solo una capa deberá ser propietaria efectiva del retry.**

---

# 28. Problema de retries anidados

Sin ownership:

```text
HTTP middleware
  3 retries

Transaction
  3 retries

Query
  3 retries

Driver
  3 retries
```

Máximo potencial:

```text
3⁴ = 81 attempts
```

Esto será considerado:

```text
Retry Amplification
```

---

# 29. Retry amplification prevention

El contexto propagará:

```text
operation identity
attempt count
budget
deadline
retry ownership
```

a capas inferiores.

---

# 30. Retry delegation

Una capa podrá responder:

```text
DELEGATE_TO_TRANSACTION
```

en vez de reintentar localmente.

---

# 31. RetryDecision

```php
final readonly class RetryDecision
{
    public function __construct(
        public RetryDecisionType $type,
        public RetryScope $scope,
        public RetryReason $reason,
        public Duration $delay,
        public RetryTarget $target,
        public RetryBudgetState $budget,
    ) {}
}
```

---

# 32. RetryDecisionType

```php
enum RetryDecisionType
{
    case RETRY;

    case STOP;

    case DELEGATE;

    case FAILOVER;

    case ABORT;

    case MARK_UNKNOWN;
}
```

---

# 33. STOP

Significa:

```text
no more attempts at this layer
```

No significa necesariamente que toda la aplicación deba terminar.

---

# 34. DELEGATE

Ejemplo:

```text
Statement detects deadlock
        ↓
DELEGATE
        ↓
Transaction Manager
        ↓
retry entire transaction
```

---

# 35. FAILOVER

Solicita cambio de endpoint/topología.

La implementación concreta pertenece al documento:

```text
240_DATABASE_FAILOVER_AND_RECOVERY_SYSTEM.md
```

---

# 36. MARK_UNKNOWN

Se utiliza cuando no existe suficiente evidencia para una recuperación automática segura.

---

# 37. RetryReason

```php
enum RetryReason
{
    case CONNECTION_NOT_ESTABLISHED;

    case CONNECTION_LOST_BEFORE_SEND;

    case TRANSIENT_SERVER_FAILURE;

    case DEADLOCK;

    case SERIALIZATION_FAILURE;

    case LOCK_TIMEOUT;

    case RESOURCE_PRESSURE;

    case REPLICA_FAILURE;

    case FAILOVER_REQUIRED;

    case CUSTOM;

    case NONE;
}
```

---

# 38. Retry eligibility

Antes de consultar política deberá determinarse:

```text
RetryEligibility
```

---

# 39. RetryEligibility

```php
enum RetryEligibility
{
    case ELIGIBLE;

    case ELIGIBLE_WITH_CONDITIONS;

    case NOT_ELIGIBLE;

    case REQUIRES_HIGHER_SCOPE_REPLAY;

    case UNKNOWN;
}
```

---

# 40. UNKNOWN ≠ ELIGIBLE

Regla:

```text
RetryEligibility::UNKNOWN
```

no deberá convertirse silenciosamente en:

```text
ELIGIBLE
```

---

# 41. ReplaySafety

```php
enum ReplaySafety
{
    case SAFE;

    case CONDITIONALLY_SAFE;

    case UNSAFE;

    case UNKNOWN;
}
```

---

# 42. Replay safety ≠ failure transience

Ejemplo:

```text
Connection reset
```

puede ser transitorio.

Pero un:

```text
INSERT
```

con outcome desconocido puede ser unsafe de repetir.

---

# 43. Retry eligibility formula

Conceptualmente:

```text
Eligible =
    FailureRecoverable
    ∧ ReplaySafe
    ∧ OutcomeAllowsRetry
    ∧ ScopeAllowsRetry
    ∧ TransactionAllowsRetry
    ∧ ConsistencyAllowsRetry
    ∧ BudgetAvailable
    ∧ DeadlineAvailable
    ∧ CircuitAllowsAttempt
    ∧ TargetAvailable
```

---

# 44. Outcome gate

El resultado anterior es una entrada obligatoria.

---

# 45. NOT_EXECUTED

Si existe certeza:

```text
Outcome = NOT_EXECUTED
```

la seguridad de retry suele ser significativamente mayor.

---

# 46. FAILED

Un `FAILED` confirmado puede ser retryable si:

```text
failure is transient
```

y la unidad puede reproducirse.

---

# 47. SUCCEEDED

No debe reintentarse como recovery de failure.

---

# 48. UNKNOWN

Será el caso más restrictivo.

---

# 49. Unknown mutation

Regla:

```text
Mutation
+
UNKNOWN outcome
=
NO BLIND RETRY
```

---

# 50. Unknown read

Puede ser reintentable bajo ciertas condiciones, pero:

```text
same read definition
≠
same observed result
```

---

# 51. RetrySafetyAnalyzer

```php
interface RetrySafetyAnalyzer
{
    public function analyze(
        RetryCandidate $candidate
    ): RetrySafetyAssessment;
}
```

---

# 52. RetrySafetyAssessment

```php
final readonly class RetrySafetyAssessment
{
    public function __construct(
        public RetryEligibility $eligibility,
        public ReplaySafety $replaySafety,
        public Confidence $confidence,
        public array $constraints,
        public array $warnings,
    ) {}
}
```

---

# 53. Idempotency

El sistema distinguirá varios niveles.

```text
Database statement idempotency
Application operation idempotency
Transaction idempotency
External side-effect idempotency
```

---

# 54. Statement idempotency

Ejemplo potencial:

```sql
UPDATE users
SET active = 1
WHERE id = ?
```

Repetir puede producir el mismo estado final de esa columna.

Pero todavía pueden existir:

```text
triggers
audit
timestamps
version columns
events
external side effects
```

---

# 55. Non-idempotent mutation

Ejemplo:

```sql
UPDATE counters
SET value = value + 1
WHERE id = ?
```

Dos ejecuciones:

```text
≠
una ejecución
```

---

# 56. INSERT

Un INSERT no será considerado automáticamente replay-safe.

---

# 57. DELETE

Tampoco será automáticamente replay-safe.

---

# 58. SELECT

Un SELECT tampoco será automáticamente replay-equivalent.

Puede contener:

```text
volatile functions
sequences
locking
side-effect functions
temporary state
```

---

# 59. Query semantics

Replay safety deberá utilizar:

```text
Query Model
AST
semantic metadata
operation metadata
transaction context
platform capabilities
```

No simplemente:

```text
SQL string prefix
```

---

# 60. Idempotency declaration

Capas superiores podrán proporcionar:

```php
final readonly class IdempotencyDeclaration
{
    public function __construct(
        public IdempotencyLevel $level,
        public ?IdempotencyKey $key,
        public Confidence $confidence,
    ) {}
}
```

---

# 61. IdempotencyLevel

```php
enum IdempotencyLevel
{
    case NONE;

    case STATEMENT;

    case TRANSACTION;

    case APPLICATION_OPERATION;

    case EXTERNALLY_DEDUPLICATED;
}
```

---

# 62. Declaración ≠ prueba absoluta

Una declaración incorrecta del desarrollador no modifica la realidad.

VoltStack podrá tratarla como:

```text
explicit application contract
```

y hacerla visible en diagnostics.

---

# 63. Idempotency keys

Podrán ayudar con operaciones donde:

```text
same logical operation
```

debe producir un único efecto.

Pero su implementación de negocio no pertenecerá al Query Engine.

---

# 64. RetryPolicy

Contrato:

```php
interface RetryPolicy
{
    public function decide(
        RetryContext $context
    ): RetryDecision;
}
```

---

# 65. DefaultRetryPolicy

VoltStack proporcionará una política conservadora:

```text
DefaultDatabaseRetryPolicy
```

Principio:

> Cuando la seguridad de replay no pueda demostrarse, el framework preferirá propagar el fallo antes que duplicar efectos.

---

# 66. Policy composition

Podrán componerse:

```text
FailurePolicy
OutcomePolicy
ReplaySafetyPolicy
BudgetPolicy
DeadlinePolicy
BackoffPolicy
CircuitPolicy
TopologyPolicy
```

---

# 67. Policy pipeline

```text
Retry Candidate
      │
      ▼
Failure Gate
      │
      ▼
Outcome Gate
      │
      ▼
Replay Safety Gate
      │
      ▼
Transaction Gate
      │
      ▼
Consistency Gate
      │
      ▼
Circuit Gate
      │
      ▼
Budget Gate
      │
      ▼
Deadline Gate
      │
      ▼
Target Resolution
      │
      ▼
Backoff Strategy
      │
      ▼
Retry Decision
```

---

# 68. Failure gate

Errores típicamente no retryable:

```text
invalid query
syntax error
constraint violation
permission denied
invalid parameter
unsupported feature
semantic error
```

---

# 69. Potentially transient failures

Ejemplos:

```text
deadlock
serialization conflict
temporary connection failure
server unavailable
temporary resource pressure
lock timeout
replica unavailable
```

Pero cada uno requiere reglas adicionales.

---

# 70. Retry classification

```php
enum RetryFailureClass
{
    case NON_RETRYABLE;

    case TRANSIENT;

    case TRANSACTION_REPLAY;

    case CONNECTION_RECOVERY;

    case FAILOVER_CANDIDATE;

    case RESOURCE_PRESSURE;

    case UNKNOWN;
}
```

---

# 71. Deadlock

Por default:

```text
RetryScope = TRANSACTION
```

cuando ocurra dentro de una unidad transaccional replayable.

---

# 72. Deadlock statement retry

Normalmente prohibido.

---

# 73. Serialization conflict

Generalmente:

```text
Transaction Replay Candidate
```

---

# 74. Lock timeout

Podrá ser:

```text
STOP
RETRY
TRANSACTION_REPLAY
```

según:

```text
platform semantics
transaction state
policy
deadline
```

---

# 75. Connection failure before send

Si:

```text
send state = NOT_SENT
```

podrá:

```text
reacquire connection
retry operation
```

si demás condiciones son válidas.

---

# 76. Connection failure after send

Si:

```text
send = SENT
outcome = UNKNOWN
```

una mutation no se reintentará ciegamente.

---

# 77. Connection acquisition

Una conexión que nunca pudo establecerse no implica riesgo de duplicar query side effects.

Por ello puede utilizar una política de retry diferente.

---

# 78. RetryBudget

El número de intentos será un recurso.

```php
final readonly class RetryBudget
{
    public function __construct(
        public int $maxAttempts,
        public Duration $maxElapsedTime,
        public Duration $maxCumulativeDelay,
    ) {}
}
```

---

# 79. Attempts vs retries

Si:

```text
maxAttempts = 3
```

significa:

```text
initial attempt
+
up to 2 retries
```

---

# 80. RetryBudgetState

```php
final readonly class RetryBudgetState
{
    public function __construct(
        public int $attemptsUsed,
        public int $attemptsRemaining,
        public Duration $elapsed,
        public Duration $delayConsumed,
    ) {}
}
```

---

# 81. Budget exhaustion

Cuando:

```text
attemptsRemaining = 0
```

la decisión será:

```text
STOP
```

---

# 82. Hierarchical budgets

VoltStack podrá manejar:

```text
Request Budget
    ↓
Database Operation Budget
        ↓
Transaction Budget
            ↓
Query Budget
                ↓
Connection Budget
```

---

# 83. Child budget

Un subsistema no podrá crear tiempo/intentos que el padre ya no posee.

---

# 84. Budget propagation

Formalmente:

```text
ChildRemaining
≤
ParentRemaining
```

---

# 85. Global retry amplification control

Incluso si cada capa individualmente permite 3 retries, el budget superior limitará el total.

---

# 86. RetryDeadline

```php
final readonly class RetryDeadline
{
    public function __construct(
        public Instant $expiresAt,
    ) {}
}
```

---

# 87. Remaining time

Antes de cada retry:

```text
remaining =
deadline - now
```

---

# 88. Retry after deadline

Prohibido.

---

# 89. Delay larger than remaining time

Si:

```text
nextDelay >= remainingDeadline
```

el retry deberá cancelarse.

---

# 90. Query timeout interaction

Cada nuevo attempt deberá recibir:

```text
min(
    configuredQueryTimeout,
    remainingOperationDeadline
)
```

---

# 91. BackoffStrategy

```php
interface BackoffStrategy
{
    public function delay(
        RetryAttempt $attempt,
        RetryContext $context
    ): Duration;
}
```

---

# 92. Immediate backoff

```text
delay = 0
```

Solo apropiado para casos específicos.

---

# 93. Constant backoff

```text
delayₙ = c
```

---

# 94. Linear backoff

```text
delayₙ = base × n
```

---

# 95. Exponential backoff

```text
delayₙ =
min(
    maxDelay,
    base × 2^(n-1)
)
```

---

# 96. Exponential backoff example

Con:

```text
base = 100 ms
```

se obtiene:

```text
100 ms
200 ms
400 ms
800 ms
1600 ms
...
```

hasta:

```text
maxDelay
```

---

# 97. Exponential overflow

El cálculo deberá estar protegido contra overflow.

---

# 98. Maximum delay

Siempre deberá existir una cota cuando se utilice exponential backoff.

---

# 99. Jitter

Backoff determinista puede sincronizar clientes.

Ejemplo:

```text
1000 workers fail simultaneously
```

y todos esperan:

```text
1 second
```

Entonces vuelven simultáneamente.

Esto produce:

```text
Thundering Herd
```

---

# 100. JitterStrategy

```php
interface JitterStrategy
{
    public function apply(
        Duration $delay,
        RetryAttempt $attempt,
        RandomSource $random
    ): Duration;
}
```

---

# 101. No jitter

```text
delay = original delay
```

---

# 102. Full jitter

Conceptualmente:

```text
actualDelay = random(0, calculatedDelay)
```

---

# 103. Equal jitter

Conceptualmente:

```text
half = calculatedDelay / 2

actualDelay =
half + random(0, half)
```

---

# 104. Decorrelated jitter

Podrá utilizar:

```text
previous delay
base delay
maximum delay
random source
```

para reducir sincronización.

---

# 105. RandomSource

No se utilizará directamente:

```php
mt_rand();
```

en componentes centrales.

Se inyectará:

```php
interface RandomSource
{
    public function integer(int $min, int $max): int;
}
```

Esto permite testing determinista.

---

# 106. Clock

Tampoco se utilizará directamente:

```php
time();
microtime();
```

en política central.

Se utilizará:

```php
interface Clock
{
    public function now(): Instant;
}
```

---

# 107. RetryScheduler

```php
interface RetryScheduler
{
    public function wait(
        Duration $duration,
        CancellationToken $cancellation
    ): void;
}
```

---

# 108. Runtime-aware waiting

En:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

el scheduler podrá integrarse con el runtime.

---

# 109. Blocking sleep

La política no deberá imponer:

```php
sleep();
```

como única implementación.

---

# 110. Cancellation

Todo delay deberá ser cancelable.

---

# 111. Cancellation during backoff

Si el request/job es cancelado:

```text
STOP
```

inmediatamente.

---

# 112. Deadline during backoff

También deberá interrumpir el retry.

---

# 113. RetryCoordinator

```php
interface RetryCoordinator
{
    public function execute(
        RetryableOperation $operation,
        RetryPolicy $policy,
        RetryContext $context
    ): mixed;
}
```

---

# 114. Coordinator responsibility

Será responsable de:

```text
attempt lifecycle
budget accounting
deadline
policy invocation
backoff
jitter
cancellation
telemetry
```

No deberá decidir semántica del query.

---

# 115. RetryableOperation

```php
interface RetryableOperation
{
    public function execute(
        RetryAttemptContext $context
    ): mixed;
}
```

---

# 116. Reconstruct operation

Cada attempt deberá reconstruir los recursos necesarios.

No deberá reutilizar automáticamente:

```text
failed statement
broken connection
invalid cursor
tainted transaction
```

---

# 117. Prepared statements

Si la conexión fue reemplazada:

```text
prepared statement
```

también deberá reconstruirse.

---

# 118. Query compilation

Una query compilada puede reutilizarse si:

```text
platform
dialect
capability generation
schema assumptions
```

siguen siendo compatibles.

---

# 119. Failover compilation

Cambiar endpoint dentro de la misma plataforma puede permitir reutilización.

Cambiar semántica/plataforma no.

---

# 120. Transaction replay

Una transaction retry deberá reconstruir:

```text
transaction
queries
reads
writes
application decisions
```

desde el boundary autorizado.

---

# 121. Transaction callback

Ejemplo conceptual:

```php
$db->transaction(
    callback: function (TransactionContext $tx) {
        // complete unit of work
    },
    retry: RetryPolicy::deadlocks(maxAttempts: 3),
);
```

---

# 122. Callback replay warning

El callback puede ejecutarse varias veces.

Por tanto:

```text
Transaction callback
```

no deberá contener side effects externos no idempotentes sin protección.

---

# 123. Dangerous transaction callback

```php
$db->transaction(function () {
    $order = createOrder();

    sendEmail($order);

    chargeExternalApi($order);
});
```

Si ocurre deadlock después del correo:

```text
transaction replay
```

podría duplicar side effects.

---

# 124. Outbox integration

Para efectos externos se preferirá:

```text
Database Transaction
    ↓
Outbox Record
    ↓
Commit
    ↓
External Delivery
```

---

# 125. Transaction retry declaration

El Transaction Manager podrá exigir:

```text
ReplayableTransaction
```

o policy explícita.

---

# 126. ORM interaction

Una transaction replay con ORM no puede simplemente reutilizar:

```text
dirty UnitOfWork
old IdentityMap
partially mutated entities
```

sin una estrategia.

---

# 127. ORM replay

Podrá requerir:

```text
new EntityManager scope
reloaded state
new UnitOfWork
new transaction
```

---

# 128. Object graph rewind

Regla:

```text
Database rollback
≠
PHP object graph rewind
```

---

# 129. ORM Retry Strategy

Podrá definirse:

```php
enum OrmRetryStrategy
{
    case RECREATE_CONTEXT;

    case CALLER_MANAGED;

    case REJECT;

    case CUSTOM;
}
```

---

# 130. Default ORM safety

VoltStack no reejecutará automáticamente una operación ORM compleja sobre un EntityManager tainted.

---

# 131. Connection retry

Integración con:

```text
236_DATABASE_CONNECTION_FAILURE_HANDLING_SYSTEM.md
```

---

# 132. Broken connection

Una conexión marcada:

```text
BROKEN
```

no se reutilizará.

---

# 133. Reacquire

Retry podrá solicitar:

```text
new connection lease
```

---

# 134. Same endpoint

Algunos failures podrán reintentar contra el mismo endpoint.

---

# 135. Different endpoint

Otros podrán solicitar:

```text
alternate replica
```

o:

```text
new writer after failover
```

---

# 136. RetryTarget

```php
enum RetryTargetMode
{
    case SAME_CONNECTION;

    case NEW_CONNECTION_SAME_ENDPOINT;

    case SAME_ROLE_DIFFERENT_ENDPOINT;

    case WRITER;

    case FAILOVER_TARGET;

    case SAME_SHARD;

    case CUSTOM;
}
```

---

# 137. SAME_CONNECTION

Solo si:

```text
connection health
```

lo permite.

---

# 138. NEW_CONNECTION_SAME_ENDPOINT

Útil para una conexión individual rota sin evidencia de endpoint failure.

---

# 139. SAME_ROLE_DIFFERENT_ENDPOINT

Puede utilizarse para replicas.

---

# 140. Writer fallback

No será automático.

---

# 141. Retry + read/write routing

El retry deberá respetar:

```text
read intent
write intent
locking intent
transaction pinning
sticky connection
read-your-writes
```

---

# 142. Replica retry

Si una replica falla:

```text
Replica A
   ↓
failure
```

podrá considerarse:

```text
Replica B
```

solo si cumple:

```text
eligibility
freshness
consistency
health
```

---

# 143. Replica retry ≠ failover

Cambiar entre replicas saludables puede ser routing.

Failover implica cambio de autoridad/topología más significativo.

---

# 144. Sticky connection

Un request sticky al writer no deberá saltar a replica durante retry.

---

# 145. Transaction pinning

Una transaction física no podrá saltar transparentemente de conexión.

---

# 146. Transaction connection failure

Normalmente:

```text
current transaction lost
```

y requiere:

```text
whole transaction replay
```

si es seguro.

---

# 147. Sharding

Retry deberá conservar:

```text
ShardId
ShardMapGeneration
RoutingContext
```

---

# 148. Same shard invariant

```text
Retry(Q on Shard A)
```

no podrá convertirse en:

```text
Q on Shard B
```

---

# 149. Shard replica

Dentro del mismo shard sí podrá existir:

```text
Replica A
Replica B
Writer
```

sujeto a routing policy.

---

# 150. Shard map change

Si durante retry cambia:

```text
ShardMapGeneration
```

deberá reevaluarse routing.

---

# 151. Resharding

No deberá utilizarse un boundary viejo si:

```text
ownership changed
```

sin validación.

---

# 152. Distributed operation

Una operación distribuida deberá mantener:

```text
per-shard attempt state
```

---

# 153. Per-shard retry

Ejemplo:

```text
Shard A SUCCESS
Shard B TRANSIENT FAILURE
Shard C SUCCESS
```

No siempre será correcto repetir:

```text
A + B + C
```

---

# 154. Distributed replay

Dependerá de:

```text
operation semantics
atomicity model
idempotency
per-shard outcomes
```

---

# 155. No fake distributed ACID

VoltStack no asumirá que repetir una operación distribuida corrige automáticamente partial commits.

---

# 156. Bulk Insert

Retry deberá integrarse con:

```text
203_DATABASE_BULK_INSERT_SYSTEM.md
```

y considerar:

```text
batch progress
idempotency
generated identifiers
partial execution
```

---

# 157. Bulk Update

Debe considerar:

```text
affected ranges
mutation semantics
version predicates
```

---

# 158. Bulk Delete

Debe considerar:

```text
already deleted rows
triggers
cascades
side effects
```

---

# 159. Import

Retry de import deberá realizarse sobre:

```text
checkpointed unit
```

no necesariamente sobre todo el archivo.

---

# 160. Export

Un export puede ser replayable desde cierto checkpoint, pero:

```text
new execution
≠
same snapshot
```

sin consistencia explícita.

---

# 161. Chunk processing

Integración con:

```text
201_DATABASE_CHUNK_PROCESSING_SYSTEM.md
```

---

# 162. Chunk retry

Debe distinguir:

```text
fetch failure
processing failure
commit failure
checkpoint failure
```

---

# 163. Checkpoint

Un checkpoint no demuestra exactamente-once.

```text
Checkpoint
+
Retry
≠
Exactly Once
```

---

# 164. Lazy Collection

Una Lazy Collection activa no deberá reintentarse arbitrariamente desde el inicio si ya produjo elementos al consumidor.

---

# 165. Observable output

Si:

```text
items already yielded
```

un retry puede producir duplicados visibles.

---

# 166. Streaming result

Un streaming query que falla después de entregar filas no puede reiniciarse transparentemente como si nada hubiera ocurrido.

---

# 167. Result cursor

```text
Result Cursor
```

es stateful.

Una nueva ejecución crea un nuevo cursor.

---

# 168. Partial result retry

Debe ser explícito.

---

# 169. Query Cache

Retry no deberá cachear:

```text
failed result
unknown result
partial result
```

como resultado exitoso.

---

# 170. Result Cache

Solo resultados confirmados y permitidos por policy podrán publicarse.

---

# 171. Cache invalidation

Para mutation con UNKNOWN outcome:

```text
conservative invalidation
```

podrá ser necesaria.

---

# 172. Retry + cache

Un retry exitoso no demuestra que el intento anterior no produjo efectos.

---

# 173. Circuit breaker integration

Antes de ejecutar un retry:

```text
CircuitBreaker
```

podrá rechazar el attempt.

---

# 174. Open circuit

```text
OPEN
```

significa:

```text
do not send normal attempt
```

---

# 175. Circuit breaker ≠ retry budget

Son mecanismos independientes.

---

# 176. Half-open

Un retry normal no deberá apropiarse automáticamente de un:

```text
half-open probe
```

sin coordinación.

---

# 177. Resource pressure

Ante:

```text
database overloaded
connection slots exhausted
temporary memory pressure
```

el retry deberá ser conservador.

---

# 178. Immediate retry under overload

Normalmente empeora:

```text
load
```

---

# 179. Adaptive policy

Futuras extensiones podrán ajustar:

```text
backoff
attempt count
target selection
```

según señales de salud.

Pero:

```text
Adaptive Retry
```

no deberá sacrificar correctness.

---

# 180. Retry storm

Muchos workers pueden fallar simultáneamente.

VoltStack deberá reducir:

```text
synchronized retries
```

mediante:

```text
jitter
budgets
circuit breaker
resource governance
```

---

# 181. Retry quota

Podrá existir un límite global/local de retries por:

```text
endpoint
pool
request
worker
operation class
```

---

# 182. Retry token

Arquitectura futura podrá utilizar:

```text
RetryTokenBucket
```

para limitar presión.

---

# 183. Retry budget ≠ rate limiter

Retry Budget controla una operación.

Rate Limiter controla presión agregada.

---

# 184. Runtime model

En persistent runtimes:

```text
RetryContext
RetryBudgetState
Attempt state
Deadline
```

serán operation-scoped.

---

# 185. Shared immutable policies

Podrán compartirse:

```text
DefaultRetryPolicy
BackoffStrategy
JitterStrategy
FailureClassifier
```

si son inmutables/stateless.

---

# 186. No static attempt

Prohibido:

```php
static int $attempt = 0;
```

---

# 187. No static retry budget

Prohibido almacenar budget mutable global por worker.

---

# 188. Request isolation

```text
Request A retry state
≠
Request B retry state
```

---

# 189. Worker reset

Al terminar operación:

```text
attempt context
cancellation registrations
temporary retry state
```

deberán liberarse.

---

# 190. FrankenPHP

Será runtime principal soportado inicialmente.

---

# 191. RoadRunner

Deberá preservar exactamente las mismas invariantes de scope.

---

# 192. OpenSwoole

Deberá soportar scheduler no bloqueante cuando la integración lo permita.

---

# 193. Telemetry

El sistema emitirá señales como:

```text
DatabaseRetryEvaluated
DatabaseRetryScheduled
DatabaseRetryStarted
DatabaseRetrySucceeded
DatabaseRetryFailed
DatabaseRetryExhausted
DatabaseRetryRejected
DatabaseRetryDelegated
```

---

# 194. Event cardinality

No deberá generarse telemetry excesiva por cada microdecisión interna.

---

# 195. Retry telemetry context

Podrá incluir:

```text
operation id
attempt
scope
owner
failure category
outcome
replay safety
decision
delay
budget remaining
deadline remaining
target type
```

---

# 196. Sensitive information

No incluir:

```text
raw parameters
credentials
tokens
PII
```

---

# 197. Metrics

Ejemplos:

```text
db.retry.operations
db.retry.attempts
db.retry.success
db.retry.exhausted
db.retry.rejected
db.retry.unknown_blocked
db.retry.transaction_replays
db.retry.connection_retries
db.retry.failover_requests
```

---

# 198. Retry success metric

Debe distinguir:

```text
operation succeeded after retry
```

de:

```text
retry attempt succeeded
```

---

# 199. Attempts histogram

Podrá medirse:

```text
attempts per logical operation
```

---

# 200. Delay histogram

```text
cumulative retry delay
```

---

# 201. Debug diagnostics

Ejemplo:

```text
Retry analysis

Operation:
    Query

Failure:
    ConnectionLost

Outcome:
    NOT_EXECUTED

Replay safety:
    SAFE

Attempt:
    1 / 3

Budget:
    2 attempts remaining

Deadline:
    1.8s remaining

Decision:
    RETRY

Target:
    New connection / same endpoint

Delay:
    83ms

Reason:
    Connection lost before query transmission
```

---

# 202. Unknown diagnostic

Ejemplo:

```text
Retry rejected

Operation:
    INSERT

Failure:
    ConnectionLost

Send state:
    SENT

Outcome:
    UNKNOWN

Replay safety:
    UNKNOWN

Decision:
    STOP

Reason:
    Database mutation may already have been applied.
```

---

# 203. Explain API

Podrá existir:

```php
$decision = $db->retries()->explain($context);
```

para testing/debugging.

---

# 204. Configuration

Ejemplo conceptual:

```php
'database' => [

    'retry' => [

        'enabled' => true,

        'max_attempts' => 3,

        'max_elapsed' => '5s',

        'backoff' => [
            'strategy' => 'exponential',
            'base' => '50ms',
            'max' => '1s',
        ],

        'jitter' => 'full',

        'unknown_outcome' => 'never_retry',

    ],

];
```

---

# 205. Configuration ≠ semantic override

No será válido:

```php
'unknown_outcome' => 'always_retry'
```

como forma de eliminar invariantes fundamentales.

---

# 206. Safe defaults

Defaults sugeridos:

```text
bounded attempts
bounded duration
jitter enabled
unknown mutations not retried
deadlocks delegated to transaction replay
connection-before-send failures retryable
constraint errors not retryable
permission errors not retryable
syntax errors not retryable
```

---

# 207. Retry profiles

Podrán existir perfiles:

```text
NONE
CONSERVATIVE
STANDARD
AGGRESSIVE_READS
TRANSACTION_CONFLICTS
CUSTOM
```

---

# 208. NONE

```text
maxAttempts = 1
```

---

# 209. CONSERVATIVE

Solo casos con evidencia fuerte.

Será apropiado como default general.

---

# 210. AGGRESSIVE_READS

Podrá aumentar recuperación para workloads read-only, pero respetando:

```text
consistency
deadline
volatile query semantics
```

---

# 211. TRANSACTION_CONFLICTS

Optimizado para:

```text
deadlocks
serialization conflicts
```

con transaction replay.

---

# 212. Custom policy

```php
final class BillingRetryPolicy implements RetryPolicy
{
    public function decide(
        RetryContext $context
    ): RetryDecision {
        // application-specific contract
    }
}
```

---

# 213. Custom policy boundaries

Una policy custom no deberá poder:

```text
change tenant
change shard arbitrarily
revive broken transaction
reinterpret UNKNOWN as FAILED
```

sin pasar por APIs explícitas autorizadas.

---

# 214. Policy registry

```php
interface RetryPolicyRegistry
{
    public function get(
        RetryPolicyId $id
    ): RetryPolicy;
}
```

---

# 215. Registry freeze

En persistent runtime podrá congelarse después de bootstrap.

---

# 216. Extension model

Plugins podrán registrar:

```text
RetryPolicy
BackoffStrategy
JitterStrategy
RetrySafetyAnalyzer
RetryTargetResolver
```

---

# 217. Extension isolation

Una extensión no deberá modificar directamente:

```text
Connection internals
Transaction internals
Driver state
```

---

# 218. Error hierarchy

```text
DatabaseRetryException
│
├── RetryPolicyException
├── RetryBudgetExceededException
├── RetryDeadlineExceededException
├── RetryNotSafeException
├── RetryUnknownOutcomeException
├── RetryOwnershipException
├── RetryTargetUnavailableException
├── RetrySchedulingException
├── RetryCancelledException
└── RetryConfigurationException
```

---

# 219. Retry exhaustion

No deberá reemplazar la causa original.

---

# 220. Failure chain

Ejemplo:

```text
Primary:
    ConnectionUnavailable

Retry history:
    Attempt 1 → Connection refused
    Attempt 2 → Connection timeout
    Attempt 3 → Connection refused

Final:
    RetryExhausted
```

La causa original y cada attempt deberán permanecer accesibles.

---

# 221. RetryHistory

```php
final readonly class RetryHistory
{
    /** @param list<RetryAttemptRecord> $attempts */
    public function __construct(
        public array $attempts,
    ) {}
}
```

---

# 222. RetryAttemptRecord

```php
final readonly class RetryAttemptRecord
{
    public function __construct(
        public RetryAttemptId $id,
        public int $number,
        public Instant $startedAt,
        public Duration $duration,
        public ?DatabaseFailure $failure,
        public QueryOutcome $outcome,
    ) {}
}
```

---

# 223. Memory bounds

RetryHistory deberá ser:

```text
bounded
```

No deberá almacenar payloads completos o traces ilimitados.

---

# 224. Testing architecture

El sistema deberá ser completamente determinista bajo:

```text
FakeClock
DeterministicRandomSource
FakeScheduler
FailureInjector
```

---

# 225. Retry sequence test

Ejemplo:

```text
Attempt 1 → transient failure
Attempt 2 → transient failure
Attempt 3 → success
```

Verificar:

```text
3 attempts
2 retries
1 logical operation
```

---

# 226. Non-retryable test

```text
UniqueConstraintViolation
```

Esperado:

```text
attempts = 1
```

---

# 227. Unknown write test

```text
INSERT
sent
connection lost
```

Esperado:

```text
outcome = UNKNOWN
retry = rejected
```

---

# 228. Before-send failure test

```text
connection lost
before send
```

Esperado:

```text
outcome = NOT_EXECUTED
retry potentially allowed
```

---

# 229. Deadlock test

Verificar:

```text
statement retry = no
transaction replay = evaluated
```

---

# 230. Deadline test

Si:

```text
remaining = 30ms
next delay = 100ms
```

Esperado:

```text
STOP
```

---

# 231. Budget test

```text
maxAttempts = 3
```

Nunca ejecutar attempt 4.

---

# 232. Jitter test

Con RandomSource determinista, delays deberán ser reproducibles.

---

# 233. Cancellation test

Cancelar durante backoff deberá evitar siguiente attempt.

---

# 234. Nested retry test

```text
Transaction owns retry
Query layer encounters failure
```

Query layer deberá:

```text
DELEGATE
```

---

# 235. Retry amplification test

Verificar que nested policies respeten budget compartido.

---

# 236. Replica retry test

Replica A falla.

Replica B:

```text
healthy
but too stale
```

No deberá seleccionarse si consistency policy la rechaza.

---

# 237. Sticky test

Después de write:

```text
sticky writer
```

un read retry no deberá saltar a replica.

---

# 238. Shard test

Retry de Shard A nunca deberá ejecutarse en Shard B.

---

# 239. Transaction pinning test

Una transaction activa nunca migrará silenciosamente de connection.

---

# 240. ORM taint test

EntityManager tainted deberá impedir replay inseguro.

---

# 241. Streaming test

Si ya se entregaron filas al consumidor:

```text
transparent restart
```

deberá rechazarse por default.

---

# 242. Persistent runtime test

```text
Request A
  retries = 2

Request B
  starts
```

Request B deberá comenzar:

```text
attempt = 1
```

---

# 243. Directory structure

```text
src/Quantum/Database/Resilience/Retry/
│
├── Contract/
│   ├── RetryPolicy.php
│   ├── RetryCoordinator.php
│   ├── RetrySafetyAnalyzer.php
│   ├── RetryScheduler.php
│   ├── BackoffStrategy.php
│   ├── JitterStrategy.php
│   └── RetryTargetResolver.php
│
├── Model/
│   ├── RetryOperationId.php
│   ├── RetryAttemptId.php
│   ├── RetryAttempt.php
│   ├── RetryContext.php
│   ├── RetryDecision.php
│   ├── RetryDecisionType.php
│   ├── RetryReason.php
│   ├── RetryScope.php
│   ├── RetryOwner.php
│   ├── RetryEligibility.php
│   ├── ReplaySafety.php
│   └── RetryFailureClass.php
│
├── Safety/
│   ├── DefaultRetrySafetyAnalyzer.php
│   ├── RetrySafetyAssessment.php
│   ├── IdempotencyDeclaration.php
│   ├── IdempotencyLevel.php
│   ├── QueryReplaySafetyAnalyzer.php
│   ├── TransactionReplaySafetyAnalyzer.php
│   └── DistributedReplaySafetyAnalyzer.php
│
├── Budget/
│   ├── RetryBudget.php
│   ├── RetryBudgetState.php
│   ├── RetryBudgetManager.php
│   └── HierarchicalRetryBudget.php
│
├── Deadline/
│   ├── RetryDeadline.php
│   └── RetryDeadlinePolicy.php
│
├── Backoff/
│   ├── ImmediateBackoff.php
│   ├── ConstantBackoff.php
│   ├── LinearBackoff.php
│   ├── ExponentialBackoff.php
│   └── DecorrelatedBackoff.php
│
├── Jitter/
│   ├── NoJitter.php
│   ├── FullJitter.php
│   ├── EqualJitter.php
│   └── DecorrelatedJitter.php
│
├── Policy/
│   ├── NoRetryPolicy.php
│   ├── ConservativeRetryPolicy.php
│   ├── StandardRetryPolicy.php
│   ├── TransactionConflictRetryPolicy.php
│   ├── ReadRetryPolicy.php
│   ├── CompositeRetryPolicy.php
│   └── RetryPolicyRegistry.php
│
├── Coordinator/
│   └── DefaultRetryCoordinator.php
│
├── Target/
│   ├── RetryTarget.php
│   ├── RetryTargetMode.php
│   └── DefaultRetryTargetResolver.php
│
├── History/
│   ├── RetryHistory.php
│   └── RetryAttemptRecord.php
│
├── Runtime/
│   ├── RetryOperationScope.php
│   └── RetryStateResetter.php
│
├── Telemetry/
│   └── RetryTelemetry.php
│
├── Testing/
│   ├── FakeRetryScheduler.php
│   ├── FakeRetryPolicy.php
│   ├── DeterministicRandomSource.php
│   └── RetryScenarioBuilder.php
│
└── Exception/
    ├── DatabaseRetryException.php
    ├── RetryPolicyException.php
    ├── RetryBudgetExceededException.php
    ├── RetryDeadlineExceededException.php
    ├── RetryNotSafeException.php
    ├── RetryUnknownOutcomeException.php
    ├── RetryOwnershipException.php
    ├── RetryTargetUnavailableException.php
    ├── RetrySchedulingException.php
    ├── RetryCancelledException.php
    └── RetryConfigurationException.php
```

---

# 244. Dependencias

Dirección permitida:

```text
Failure Handling
      ↓
Retry Safety
      ↓
Retry Policy
      ↓
Retry Coordinator
      ↓
Execution abstraction
```

Integraciones:

```text
Retry
├── Connection
├── Transaction
├── Query Executor
├── Routing
├── Replica
├── Failover
├── Circuit Breaker
├── Telemetry
├── Runtime
└── Testing
```

---

# 245. Dependencias prohibidas

Retry core no deberá depender directamente de:

```text
HTTP
Controllers
Livewire-like runtime
application models
specific tenant package
specific cache provider
specific queue provider
```

---

# 246. Invariantes arquitectónicas

## DB-RETRY-001
Failure no será equivalente a Retry.

## DB-RETRY-002
Transient Failure no será equivalente a Safe Retry.

## DB-RETRY-003
Retry Policy no será Failure Classifier.

## DB-RETRY-004
Retry Policy no será Retry Executor.

## DB-RETRY-005
Retry Policy no será Backoff Strategy.

## DB-RETRY-006
Retry Policy no será Circuit Breaker.

## DB-RETRY-007
Retry Policy no será Failover System.

## DB-RETRY-008
Cada retry tendrá scope explícito.

## DB-RETRY-009
Cada retry tendrá owner explícito.

## DB-RETRY-010
Solo una capa será owner efectivo por failure boundary.

## DB-RETRY-011
Logical operation será distinta de physical attempt.

## DB-RETRY-012
Attempt inicial será attempt 1.

## DB-RETRY-013
Retry count será distinto de attempt count.

## DB-RETRY-014
Retry amplification será controlada.

## DB-RETRY-015
Nested retries compartirán budgets cuando corresponda.

## DB-RETRY-016
Child budget no excederá parent budget.

## DB-RETRY-017
Retry tendrá maximum attempts.

## DB-RETRY-018
Retry podrá tener maximum elapsed time.

## DB-RETRY-019
Retry podrá tener maximum cumulative delay.

## DB-RETRY-020
No habrá retry después de deadline.

## DB-RETRY-021
Delay no podrá consumir tiempo inexistente.

## DB-RETRY-022
Cada attempt respetará remaining deadline.

## DB-RETRY-023
Backoff no decidirá retry eligibility.

## DB-RETRY-024
Jitter no decidirá retry eligibility.

## DB-RETRY-025
Jitter podrá reducir thundering herd.

## DB-RETRY-026
RandomSource será inyectable.

## DB-RETRY-027
Clock será inyectable.

## DB-RETRY-028
Scheduler será abstraído.

## DB-RETRY-029
Core no dependerá de sleep().

## DB-RETRY-030
Backoff será cancelable.

## DB-RETRY-031
Cancellation detendrá nuevos attempts.

## DB-RETRY-032
UNKNOWN eligibility no será ELIGIBLE.

## DB-RETRY-033
UNKNOWN replay safety no será SAFE.

## DB-RETRY-034
UNKNOWN mutation outcome no tendrá blind retry.

## DB-RETRY-035
Reconnect no convertirá UNKNOWN en FAILED.

## DB-RETRY-036
Retry success no demostrará que previous attempt failed.

## DB-RETRY-037
Previous attempt podrá haber tenido éxito aunque ACK se perdiera.

## DB-RETRY-038
Idempotency será explícita cuando se use para justificar retry.

## DB-RETRY-039
Statement idempotency será distinta de operation idempotency.

## DB-RETRY-040
Operation idempotency será distinta de transaction idempotency.

## DB-RETRY-041
External side-effect idempotency será distinta de DB idempotency.

## DB-RETRY-042
SELECT no será automáticamente replay-safe.

## DB-RETRY-043
INSERT no será automáticamente replay-safe.

## DB-RETRY-044
UPDATE no será automáticamente replay-safe.

## DB-RETRY-045
DELETE no será automáticamente replay-safe.

## DB-RETRY-046
SQL prefix no determinará replay safety.

## DB-RETRY-047
Query semantics podrán determinar replay safety.

## DB-RETRY-048
Volatile functions serán consideradas.

## DB-RETRY-049
Locking reads serán consideradas.

## DB-RETRY-050
Database side-effect functions serán consideradas.

## DB-RETRY-051
Constraint violations no serán retryable por default.

## DB-RETRY-052
Syntax errors no serán retryable por default.

## DB-RETRY-053
Semantic errors no serán retryable por default.

## DB-RETRY-054
Permission errors no serán retryable por default.

## DB-RETRY-055
Invalid parameters no serán retryable por default.

## DB-RETRY-056
Unsupported features no serán retryable por default.

## DB-RETRY-057
Deadlocks podrán requerir transaction replay.

## DB-RETRY-058
Deadlocks no tendrán blind statement retry.

## DB-RETRY-059
Serialization failures podrán requerir transaction replay.

## DB-RETRY-060
Lock timeout tendrá platform-aware policy.

## DB-RETRY-061
Connection failure before send podrá ser retryable.

## DB-RETRY-062
Connection failure after unknown write no tendrá blind retry.

## DB-RETRY-063
Connection acquisition retry será distinto de query retry.

## DB-RETRY-064
Broken connection no será reutilizada.

## DB-RETRY-065
Prepared statement ligado a broken connection no será reutilizado.

## DB-RETRY-066
Transaction no migrará transparentemente entre conexiones.

## DB-RETRY-067
Lost transaction requerirá higher-scope analysis.

## DB-RETRY-068
Database rollback no rebobinará object graph.

## DB-RETRY-069
Tainted EntityManager no será replayado ciegamente.

## DB-RETRY-070
ORM retry podrá recrear context.

## DB-RETRY-071
Transaction callback podrá ejecutarse múltiples veces bajo replay.

## DB-RETRY-072
External side effects dentro de replayable transaction requerirán protección.

## DB-RETRY-073
Outbox será preferible para side effects transaccionales.

## DB-RETRY-074
Retry target será explícito.

## DB-RETRY-075
Same connection requerirá connection health válido.

## DB-RETRY-076
New connection no significará new logical operation.

## DB-RETRY-077
Replica retry respetará freshness.

## DB-RETRY-078
Replica retry respetará consistency.

## DB-RETRY-079
Replica failure no implicará writer fallback.

## DB-RETRY-080
Writer fallback requerirá policy.

## DB-RETRY-081
Sticky writer será preservado.

## DB-RETRY-082
Transaction pinning será preservado.

## DB-RETRY-083
Shard affinity será preservada.

## DB-RETRY-084
Retry no cambiará arbitrariamente de shard.

## DB-RETRY-085
Shard map generation podrá formar parte del context.

## DB-RETRY-086
Resharding requerirá reevaluación.

## DB-RETRY-087
Distributed retry será per-outcome aware.

## DB-RETRY-088
Distributed partial success no será complete failure ni success automáticamente.

## DB-RETRY-089
Distributed partial commit no será ocultado.

## DB-RETRY-090
Cross-shard ACID no será inventado.

## DB-RETRY-091
Bulk retry preservará progress semantics.

## DB-RETRY-092
Import retry podrá usar checkpoint.

## DB-RETRY-093
Export retry no garantizará same snapshot.

## DB-RETRY-094
Chunk retry respetará checkpoint semantics.

## DB-RETRY-095
Checkpoint + Retry no significará Exactly Once.

## DB-RETRY-096
Lazy Collection no reiniciará transparentemente tras producir items.

## DB-RETRY-097
Streaming Result no reiniciará transparentemente tras producir filas.

## DB-RETRY-098
Partial result será explícito.

## DB-RETRY-099
Failed result no será publicado como cache success.

## DB-RETRY-100
Unknown result no será publicado como cache success.

## DB-RETRY-101
Retry success no eliminará necesidad de conservative invalidation tras unknown write.

## DB-RETRY-102
Circuit Breaker podrá bloquear attempts.

## DB-RETRY-103
Circuit Breaker no será Retry Budget.

## DB-RETRY-104
Half-open probes serán coordinados.

## DB-RETRY-105
Resource pressure no tendrá retry storm ilimitado.

## DB-RETRY-106
Jitter podrá utilizarse ante fallos masivos.

## DB-RETRY-107
Rate limiting será distinto de per-operation budget.

## DB-RETRY-108
RetryContext será operation-scoped.

## DB-RETRY-109
Attempt state no será static mutable state.

## DB-RETRY-110
Budget state no será static mutable state.

## DB-RETRY-111
Request A no heredará retry state de Request B.

## DB-RETRY-112
Immutable policies podrán compartirse entre workers.

## DB-RETRY-113
Runtime-specific scheduler no alterará semántica.

## DB-RETRY-114
FrankenPHP respetará scope isolation.

## DB-RETRY-115
RoadRunner respetará scope isolation.

## DB-RETRY-116
OpenSwoole respetará scope isolation.

## DB-RETRY-117
Telemetry distinguirá operation de attempt.

## DB-RETRY-118
Telemetry distinguirá scheduled retry de executed retry.

## DB-RETRY-119
Telemetry no expondrá bindings sensibles.

## DB-RETRY-120
Metrics tendrán cardinalidad acotada.

## DB-RETRY-121
Retry history será bounded.

## DB-RETRY-122
Retry exhaustion conservará causas anteriores.

## DB-RETRY-123
Retry exhaustion no reemplazará primary failure.

## DB-RETRY-124
Custom policy no cambiará tenant.

## DB-RETRY-125
Custom policy no cambiará shard arbitrariamente.

## DB-RETRY-126
Custom policy no revivirá transaction abortada.

## DB-RETRY-127
Custom policy no convertirá UNKNOWN en FAILED.

## DB-RETRY-128
Custom policy no convertirá UNKNOWN en SUCCESS.

## DB-RETRY-129
Retry registry podrá congelarse tras bootstrap.

## DB-RETRY-130
Retry extensions respetarán component boundaries.

## DB-RETRY-131
Failure classifier no ejecutará retry.

## DB-RETRY-132
Query Compiler no ejecutará retry.

## DB-RETRY-133
Query Builder no ejecutará retry.

## DB-RETRY-134
Driver no decidirá business retry.

## DB-RETRY-135
Transaction Manager será owner de transaction replay.

## DB-RETRY-136
Connection Manager podrá ser owner de connection acquisition retry.

## DB-RETRY-137
Query Executor podrá ser owner solo cuando scope sea query/statement seguro.

## DB-RETRY-138
Distributed Coordinator será owner de distributed retry.

## DB-RETRY-139
Chunk Runner será owner de chunk retry.

## DB-RETRY-140
Retry delegation será explícita.

## DB-RETRY-141
Retry scheduling será separado de policy.

## DB-RETRY-142
Target resolution será separado de policy cuando corresponda.

## DB-RETRY-143
Backoff tendrá maximum delay.

## DB-RETRY-144
Exponential backoff protegerá overflow.

## DB-RETRY-145
Cumulative delay será contabilizable.

## DB-RETRY-146
Elapsed execution time será contabilizable.

## DB-RETRY-147
Waiting time consumirá deadline.

## DB-RETRY-148
Connection acquisition consumirá deadline.

## DB-RETRY-149
Query execution consumirá deadline.

## DB-RETRY-150
Transaction replay consumirá parent deadline.

## DB-RETRY-151
Retry policy podrá ser explicable.

## DB-RETRY-152
Retry decision tendrá reason estructurado.

## DB-RETRY-153
Retry rejection tendrá reason estructurado.

## DB-RETRY-154
Retry safety tendrá confidence explícita.

## DB-RETRY-155
Policy no inferirá certainty inexistente.

## DB-RETRY-156
Correctness tendrá prioridad sobre availability.

## DB-RETRY-157
Consistency tendrá prioridad sobre transparent replica retry.

## DB-RETRY-158
Transaction integrity tendrá prioridad sobre statement convenience.

## DB-RETRY-159
Bounded recovery tendrá prioridad sobre infinite retry.

## DB-RETRY-160
Automatic recovery nunca deberá ocultar UNKNOWN database reality.

## DB-RETRY-161
A new endpoint no demostrará outcome del endpoint anterior.

## DB-RETRY-162
A new connection no demostrará outcome del attempt anterior.

## DB-RETRY-163
A successful retry no demostrará ausencia de duplicate side effects.

## DB-RETRY-164
Retry policy deberá considerar connection impact.

## DB-RETRY-165
Retry policy deberá considerar transaction impact.

## DB-RETRY-166
Retry policy deberá considerar query outcome.

## DB-RETRY-167
Retry policy deberá considerar replay safety.

## DB-RETRY-168
Retry policy deberá considerar operation scope.

## DB-RETRY-169
Retry policy deberá considerar owner.

## DB-RETRY-170
Retry policy deberá considerar target eligibility.

## DB-RETRY-171
Retry policy deberá considerar circuit state.

## DB-RETRY-172
Retry policy deberá considerar resource pressure.

## DB-RETRY-173
Retry policy deberá considerar cancellation.

## DB-RETRY-174
Retry policy deberá considerar deadline.

## DB-RETRY-175
Retry policy deberá considerar budget.

## DB-RETRY-176
Retry policy deberá considerar consistency.

## DB-RETRY-177
Retry policy deberá considerar tenant context.

## DB-RETRY-178
Retry policy deberá considerar shard context.

## DB-RETRY-179
Retry policy deberá considerar transaction ownership.

## DB-RETRY-180
Retry policy deberá considerar externally visible progress.

---

# 247. Modelo formal

Sea:

```text
O
```

una operación lógica y:

```text
Aₙ
```

su intento número `n`.

Definimos:

```text
Failure(Aₙ)
Outcome(Aₙ)
ReplaySafety(O)
Budget(O)
Deadline(O)
```

---

# 248. Condición mínima de retry

Un siguiente intento:

```text
Aₙ₊₁
```

solo podrá crearse si:

```text
RetryAllowed(Aₙ) = true
```

---

# 249. Función general

```text
RetryAllowed(Aₙ)
=
Recoverable(Failure(Aₙ))
∧
OutcomeCompatible(Outcome(Aₙ))
∧
ReplaySafe(O)
∧
ScopeValid(O)
∧
TransactionValid(O)
∧
ConsistencyValid(O)
∧
TargetAvailable(O)
∧
CircuitAllows(O)
∧
BudgetRemaining(O)
∧
DeadlineRemaining(O)
∧
¬Cancelled(O)
```

---

# 250. Unknown mutation rule

Si:

```text
Mutation(O) = true
```

y:

```text
Outcome(Aₙ) = UNKNOWN
```

entonces por default:

```text
RetryAllowed(Aₙ) = false
```

---

# 251. Budget rule

Para:

```text
MaxAttempts = M
```

debe cumplirse:

```text
n < M
```

antes de crear:

```text
Aₙ₊₁
```

---

# 252. Deadline rule

Sea:

```text
D = deadline
t = current time
b = next backoff
```

El retry solo podrá programarse si:

```text
t + b < D
```

y existe tiempo razonable para ejecutar el attempt.

---

# 253. Hierarchical budget rule

Para un child scope:

```text
BudgetChild
```

y parent:

```text
BudgetParent
```

se cumplirá:

```text
Remaining(Child)
≤
Remaining(Parent)
```

---

# 254. Backoff formal

Exponential:

```text
Bₙ =
min(
    Bmax,
    Bbase × 2^(n-1)
)
```

Con full jitter:

```text
Jₙ ~ Uniform(0, Bₙ)
```

El delay efectivo será:

```text
Delayₙ = Jₙ
```

---

# 255. Replay rule

```text
Retryable
≠
Transient
```

sino:

```text
Retryable
=
Transient
∧
ReplaySafe
∧
PolicyAllowed
```

más las restricciones operativas correspondientes.

---

# 256. Retry state machine

```text
             ┌─────────────┐
             │   CREATED   │
             └──────┬──────┘
                    │
                    ▼
             ┌─────────────┐
             │  ATTEMPTING │
             └──────┬──────┘
                    │
          ┌─────────┴─────────┐
          │                   │
       SUCCESS              FAILURE
          │                   │
          ▼                   ▼
     ┌─────────┐       ┌─────────────┐
     │COMPLETE │       │  EVALUATING │
     └─────────┘       └──────┬──────┘
                              │
                 ┌────────────┼─────────────┐
                 │            │             │
               STOP         RETRY        DELEGATE
                 │            │             │
                 ▼            ▼             ▼
              FAILED       WAITING      ESCALATED
                              │
                              ▼
                          ATTEMPTING
```

Estados adicionales:

```text
CANCELLED
EXHAUSTED
UNKNOWN
```

---

# 257. Flujo completo

```text
Logical Database Operation
          │
          ▼
      Attempt #1
          │
          ▼
       Execute
          │
     ┌────┴─────┐
     │          │
 SUCCESS      FAILURE
     │          │
     ▼          ▼
 COMPLETE   Normalize Failure
                │
                ▼
          Resolve Outcome
                │
                ▼
          Analyze Replay
                │
                ▼
          Check Ownership
                │
                ▼
          Check Transaction
                │
                ▼
          Check Consistency
                │
                ▼
          Check Circuit
                │
                ▼
          Check Budget
                │
                ▼
          Check Deadline
                │
                ▼
          Resolve Target
                │
                ▼
          Retry Policy
                │
       ┌────────┼─────────┐
       │        │         │
      STOP   DELEGATE   RETRY
       │        │         │
       ▼        ▼         ▼
     FAIL    Parent     Backoff
             Scope        │
                          ▼
                        Jitter
                          │
                          ▼
                       Scheduler
                          │
                          ▼
                      Attempt #2
```

---

# 258. Regla maestra

> **VoltStack solo realizará un nuevo intento cuando conozca suficientemente el fallo anterior, pueda identificar la unidad exacta que debe repetirse, disponga de un boundary de replay válido, tenga presupuesto y tiempo disponibles, preserve las garantías de transacción, consistencia, tenant y shard, y no exista evidencia de que repetir la operación pueda duplicar efectos de manera insegura.**

En forma compacta:

```text
Exception
≠
Retry

Transient
≠
Retryable

Reconnect
≠
Replay Permission

Backoff
≠
Retry Policy

Jitter
≠
Retry Safety

Circuit Breaker
≠
Retry

Failover
≠
Retry

Statement Retry
≠
Transaction Replay

Idempotent Statement
≠
Idempotent Operation

Rollback
≠
Object Graph Rewind

Checkpoint
+
Retry
≠
Exactly Once

Successful Retry
≠
Previous Attempt Failed

UNKNOWN WRITE
→
NO BLIND RETRY
```

La prioridad será:

```text
Correctness
    >
Outcome Certainty
    >
Transaction Integrity
    >
Consistency
    >
Replay Safety
    >
Resource Protection
    >
Availability
    >
Transparent Recovery
    >
Convenience
```

---

# 259. Estado del Bloque 23

```text
BLOCK 23 — RESILIENCE

✓ 235_DATABASE_RESILIENCE_ARCHITECTURE.md
✓ 236_DATABASE_CONNECTION_FAILURE_HANDLING_SYSTEM.md
✓ 237_DATABASE_QUERY_FAILURE_HANDLING_SYSTEM.md
✓ 238_DATABASE_RETRY_POLICY_SYSTEM.md
○ 239_DATABASE_CIRCUIT_BREAKER_INTEGRATION_SYSTEM.md
○ 240_DATABASE_FAILOVER_AND_RECOVERY_SYSTEM.md
○ 241_DATABASE_RESOURCE_EXHAUSTION_PROTECTION_SYSTEM.md
```

---

# 260. Siguiente documento

```text
239_DATABASE_CIRCUIT_BREAKER_INTEGRATION_SYSTEM.md
```

El siguiente documento definirá cómo VoltStack Database podrá detectar fallos sistémicos repetidos y **dejar temporalmente de enviar operaciones hacia recursos degradados**, evitando que retry, pooling y alta concurrencia conviertan una falla parcial de infraestructura en una falla general del sistema.

Se desarrollarán:

```text
CircuitBreaker
CircuitState
CLOSED
OPEN
HALF_OPEN

Failure Window
Failure Threshold
Failure Rate
Minimum Throughput
Slow Call Threshold
Probe System
Half-Open Probe Budget
Recovery Threshold
Open Duration
Adaptive Cooldown

Circuit Scope
Endpoint Circuit
Replica Circuit
Writer Circuit
Shard Circuit
Connection Pool Circuit

Failure Classification
Circuit-Relevant Failures
Circuit-Irrelevant Failures

Retry Integration
Failover Integration
Load Balancer Integration
Replica Health Integration
Connection Pool Integration
Resource Governance

Distributed Circuit State
Local vs Shared State
Persistent Runtime Safety
Telemetry
Diagnostics
Testing
```

La regla central será:

> **El Circuit Breaker no intenta reparar una operación fallida; protege al sistema evitando que nuevas operaciones continúen ejerciendo presión sobre un recurso que presenta evidencia suficiente de degradación.**