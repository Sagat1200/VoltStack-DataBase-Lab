# 170_DATABASE_TRANSACTION_RETRY_SYSTEM.md

# VoltStack Quantum Database
## Database Transaction Retry System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 170 — Database Transaction Retry System  
**Bloque:** 15 — Transactions & Concurrency  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `169_DATABASE_SAVEPOINT_SYSTEM.md`  
**Siguiente documento:** `171_DATABASE_DEADLOCK_HANDLING_SYSTEM.md`

---

# 1. Propósito

`Database Transaction Retry System` define cómo VoltStack podrá repetir de forma controlada una transacción completa cuando esta falle por una condición temporal y recuperable.

Ejemplo:

```text
Attempt 1
    BEGIN
    operation A
    operation B
    DEADLOCK
    ROLLBACK

        ↓ retry

Attempt 2
    BEGIN
    operation A
    operation B
    COMMIT
```

La arquitectura deberá evitar un error especialmente peligroso:

```text
Transaction Retry
≠
Failed Statement Retry
```

Si una transacción contiene:

```text
BEGIN

INSERT order
UPDATE inventory
INSERT payment

COMMIT
```

y falla:

```text
UPDATE inventory
```

no necesariamente es correcto ejecutar únicamente otra vez ese `UPDATE`.

Dependiendo del motor, el error puede haber:

- invalidado la transacción;
- provocado rollback completo;
- alterado locks;
- cambiado el snapshot;
- cambiado el resultado de lecturas;
- dejado el estado local del ORM desincronizado.

Por ello la regla principal será:

> **VoltStack reintentará una unidad transaccional desde un boundary conocido y reproducible; nunca asumirá que repetir el statement que produjo el error equivale a repetir correctamente la transacción.**

---

# 2. Objetivos

El sistema deberá proporcionar:

1. retry boundaries explícitos;
2. clasificación de errores;
3. transient failure detection;
4. retry eligibility;
5. retry policies;
6. attempt tracking;
7. retry budgets;
8. exponential backoff;
9. jitter;
10. deadlines;
11. cancellation;
12. retry-safe callback execution;
13. transaction replay;
14. rollback verification;
15. UNKNOWN outcome protection;
16. EntityManager reset;
17. UnitOfWork reset;
18. IdentityMap reconciliation;
19. nested transaction awareness;
20. `REQUIRES_NEW` awareness;
21. savepoint retry integration;
22. deadlock integration;
23. isolation failure integration;
24. connection failure classification;
25. external side-effect protection;
26. observability;
27. telemetry;
28. persistent-runtime isolation;
29. resource governance;
30. deterministic testing.

---

# 3. No objetivos

Transaction Retry System no deberá:

- detectar deadlocks directamente desde vendor strings;
- implementar locking;
- implementar isolation;
- ejecutar SQL directamente;
- reemplazar Transaction Manager;
- convertir errores permanentes en temporales;
- ocultar UNKNOWN outcomes;
- garantizar idempotencia de código de aplicación;
- revertir side effects externos;
- implementar distributed transactions;
- implementar sagas;
- implementar two-phase commit.

---

# 4. Distinciones fundamentales

```text
Transaction Retry
≠
Query Retry
≠
Statement Retry
≠
Connection Retry
≠
HTTP Retry
≠
Job Retry
≠
Savepoint Retry
≠
Deadlock Detection
≠
Idempotency
```

---

# 5. Problema principal

Considérese:

```php
DB::transaction(function () {
    $order = Order::create(...);

    Inventory::reserve(...);

    Payment::record(...);
});
```

Si `Inventory::reserve()` genera un deadlock:

```text
Order INSERT
    ↓
Inventory UPDATE
    ↓
DEADLOCK
```

el sistema no deberá hacer:

```text
retry Inventory UPDATE only
```

como comportamiento transaccional general.

Deberá determinar si:

```text
entire transaction
```

puede ser reejecutada.

---

# 6. Retry Boundary

Concepto central:

```text
RetryBoundary
```

representa la unidad completa que puede repetirse.

Ejemplo:

```text
Retry Boundary
│
├── BEGIN
├── callback
│   ├── operation A
│   ├── operation B
│   └── operation C
├── COMMIT
└── completion
```

---

# 7. Retry boundary invariant

Debe cumplirse:

```text
Retry
→
restart from beginning of RetryBoundary
```

No:

```text
Retry
→
continue from failed statement
```

salvo una estrategia explícita especializada.

---

# 8. API conceptual

```php
$result = DB::transaction(
    callback: function () {
        // transactional work
    },
    retry: RetryPolicy::transient(
        maxAttempts: 3,
    ),
);
```

---

# 9. Alternative API

También podrá existir:

```php
DB::transaction()
    ->retry(3)
    ->run(function () {
        // ...
    });
```

Ambas APIs deberán converger sobre el mismo:

```text
TransactionRetryEngine
```

---

# 10. TransactionRetryEngine

Contrato conceptual:

```php
interface TransactionRetryEngine
{
    public function execute(
        TransactionDefinition $transaction,
        TransactionRetryPolicy $policy,
        callable $callback,
    ): mixed;
}
```

---

# 11. Arquitectura general

```text
Transaction Request
        ↓
Retry Engine
        ↓
Attempt #1
        ↓
Transaction Manager
        ↓
BEGIN
        ↓
Callback
        ↓
COMMIT
        ↓
Success
```

Ante error:

```text
Failure
   ↓
Failure Classifier
   ↓
Retry Eligibility
   ↓
Rollback / Cleanup
   ↓
Context Reset
   ↓
Backoff
   ↓
Attempt #2
```

---

# 12. Attempt model

Cada ejecución tendrá:

```php
final readonly class TransactionRetryAttempt
{
    public function __construct(
        public RetryAttemptNumber $number,
        public TransactionId $transactionId,
        public RetryBoundaryId $boundaryId,
        public RetryAttemptStartedAt $startedAt,
    ) {}
}
```

---

# 13. Transaction identity per attempt

Cada intento deberá utilizar una nueva:

```text
PhysicalTransactionId
```

Por tanto:

```text
Attempt 1 → TX-101
Attempt 2 → TX-102
Attempt 3 → TX-103
```

---

# 14. RetryBoundaryId

Todos los intentos compartirán:

```text
RetryBoundaryId
```

Ejemplo:

```text
Boundary:
    retry-boundary-8

Attempt 1:
    TX-101

Attempt 2:
    TX-102
```

Esto permite correlación sin afirmar que ambas son la misma transacción física.

---

# 15. Attempt state

```php
enum RetryAttemptState
{
    case CREATED;
    case RUNNING;
    case SUCCEEDED;
    case FAILED_RETRYABLE;
    case FAILED_FINAL;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 16. Retry lifecycle

```text
CREATED
   ↓
RUNNING
  /   \
 ▼     ▼
SUCCESS FAILURE
          ↓
    Classification
      /       \
     ▼         ▼
RETRYABLE   TERMINAL
    │
    ▼
BACKOFF
    │
    ▼
NEXT ATTEMPT
```

---

# 17. Failure classification

El Retry System consumirá una clasificación canónica:

```php
enum DatabaseFailureClass
{
    case DEADLOCK;
    case SERIALIZATION_FAILURE;
    case LOCK_TIMEOUT;
    case CONNECTION_TRANSIENT;
    case CONNECTION_FATAL;
    case RESOURCE_TRANSIENT;
    case CONSTRAINT_VIOLATION;
    case SYNTAX_ERROR;
    case AUTHENTICATION_FAILURE;
    case PERMISSION_FAILURE;
    case TRANSACTION_ABORTED;
    case UNKNOWN;
}
```

---

# 18. Failure Classifier

```php
interface DatabaseFailureClassifier
{
    public function classify(
        Throwable $error,
        FailureContext $context,
    ): DatabaseFailureClassification;
}
```

---

# 19. Classifier architecture

```text
Driver Exception
      ↓
Platform/Driver Error Adapter
      ↓
Canonical Failure Classification
      ↓
Transaction Retry System
```

No:

```text
Retry System
    ↓
parse random vendor message strings
```

---

# 20. Retryability is not failure class alone

Una clasificación:

```text
DEADLOCK
```

no implica automáticamente:

```text
retry = yes
```

Debe evaluarse:

```text
Failure Class
+
Transaction State
+
Outcome Certainty
+
Retry Policy
+
Attempt Number
+
Deadline
+
Side Effect Safety
+
Context Recoverability
=
Retry Eligibility
```

---

# 21. RetryEligibility

```php
enum RetryEligibility
{
    case ELIGIBLE;
    case NOT_ELIGIBLE;
    case EXHAUSTED;
    case UNSAFE;
    case UNKNOWN;
}
```

---

# 22. RetryDecision

```php
final readonly class RetryDecision
{
    public function __construct(
        public RetryEligibility $eligibility,
        public RetryReason $reason,
        public ?Duration $delay,
    ) {}
}
```

---

# 23. Typical retryable failures

Podrán incluir:

```text
deadlock
serialization failure
selected lock timeout conditions
temporary connection acquisition failure
selected transient infrastructure failures
```

siempre dependiendo de contexto y plataforma.

---

# 24. Typically non-retryable failures

Ejemplos:

```text
unique constraint violation
foreign key violation
SQL syntax error
invalid column
authentication failure
permission denied
invalid data type
business validation failure
```

Por default:

```text
NO RETRY
```

---

# 25. UNKNOWN is special

Regla crítica:

```text
UNKNOWN transaction outcome
≠
retryable transient failure
```

---

# 26. Commit ambiguity

Caso:

```text
COMMIT sent
    ↓
connection lost
    ↓
did commit happen?
```

Resultado:

```text
UNKNOWN
```

Reintentar automáticamente podría producir:

```text
duplicate order
duplicate payment
duplicate transfer
duplicate event
```

---

# 27. UNKNOWN default

Por default:

```text
TransactionOutcome::UNKNOWN
→
NO AUTOMATIC RETRY
```

---

# 28. Explicit idempotent recovery

Solo una estrategia especializada podrá manejar UNKNOWN mediante:

- idempotency key;
- transaction marker;
- business operation ID;
- reconciliation query;
- external durable ledger;
- application-specific recovery protocol.

Esto no formará parte del retry genérico.

---

# 29. Rollback certainty

Antes de repetir una transacción, VoltStack deberá determinar si el intento anterior está suficientemente terminado.

Idealmente:

```text
Attempt N
    ↓
ROLLBACK CONFIRMED
    ↓
cleanup
    ↓
Attempt N+1
```

---

# 30. Connection-lost rollback

Si la conexión se pierde antes de rollback, podrá saberse que el servidor eventualmente abortará la transacción, pero VoltStack no deberá inferir más de lo que permita el driver/platform evidence.

---

# 31. Retry policy

```php
interface TransactionRetryPolicy
{
    public function decide(
        TransactionRetryContext $context,
        DatabaseFailureClassification $failure,
    ): RetryDecision;
}
```

---

# 32. Default policy

VoltStack podrá proporcionar:

```text
NoRetryPolicy
TransientRetryPolicy
DeadlockRetryPolicy
SerializationRetryPolicy
CompositeRetryPolicy
```

---

# 33. Secure default

El default general deberá ser conservador:

```text
No automatic retry
```

o un retry muy limitado exclusivamente cuando la API indique explícitamente que el boundary es retryable.

Esto evita repetir callbacks que el desarrollador nunca declaró seguros.

---

# 34. Explicit retry declaration

Ejemplo:

```php
DB::transaction(
    callback: fn () => $service->execute(),
    retry: 3,
);
```

Esto comunica que el callback puede potencialmente repetirse.

---

# 35. Retry-safe callback

Un callback retryable debe ser considerado:

```text
potentially executed N times
```

aunque solo una transacción termine committed.

---

# 36. Important developer contract

Si:

```text
maxAttempts = 3
```

entonces:

```text
callback invocation count
∈ [1, 3]
```

---

# 37. External side effects

Esto es peligroso:

```php
DB::transaction(function () {
    $order->save();

    $mailer->sendInvoice();

    updateInventory();
}, retry: 3);
```

Si ocurre deadlock después del email:

```text
email sent
DB transaction rolled back
callback retried
email sent again
```

---

# 38. Side-effect invariant

```text
Database Rollback
≠
External Side Effect Rollback
```

---

# 39. Recommended pattern

Preferir:

```text
transaction
    ↓
database changes
    ↓
outbox record
    ↓
COMMIT
    ↓
outbox delivery
```

---

# 40. afterCommit

También podrá utilizarse:

```php
DB::afterCommit(function () {
    // external effect
});
```

pero debe entenderse que:

```text
afterCommit callback failure
≠
database rollback
```

---

# 41. Attempt-local callbacks

Callbacks registrados durante un intento fallido no deberán sobrevivir accidentalmente al siguiente intento.

---

# 42. Example

Attempt 1 registra:

```text
afterCommit(A)
```

y hace rollback.

Attempt 2 registra:

```text
afterCommit(A)
```

y hace commit.

Solo el callback perteneciente al intento 2 deberá ejecutarse.

---

# 43. Callback registry

Por tanto:

```text
Transaction Callback Registry
```

será:

```text
PhysicalTransactionContext-local
```

y se destruirá entre intentos.

---

# 44. ORM integration

El mayor problema de retry con ORM es:

```text
Database rollback
≠
Object graph rollback
```

---

# 45. Example ORM hazard

Attempt 1:

```text
Entity:
    status = "pending"

callback:
    status = "paid"
    flush()
    deadlock
    rollback
```

Después del rollback, el objeto PHP puede seguir:

```text
status = "paid"
```

---

# 46. Unsafe replay

Si Attempt 2 reutiliza exactamente el mismo:

```text
EntityManager
UnitOfWork
IdentityMap
entities
snapshots
```

el callback podría observar un estado distinto al del Attempt 1.

---

# 47. Persistence retry boundary

Para ORM, VoltStack deberá definir:

```text
Retryable Persistence Boundary
```

que pueda producir un estado limpio antes de cada attempt.

---

# 48. Recommended ORM strategy

Por default:

```text
Attempt fails
    ↓
rollback
    ↓
discard/clear attempt PersistenceContext
    ↓
create fresh PersistenceContext
    ↓
retry callback
```

---

# 49. Fresh EntityManager

En retry ORM seguro:

```text
Attempt 1
    EM-1
    UoW-1
    IdentityMap-1

Attempt 2
    EM-2
    UoW-2
    IdentityMap-2
```

es el modelo más fácil de razonar.

---

# 50. Entity references across attempts

El código no deberá asumir:

```text
$entityFromAttempt1 === $entityFromAttempt2
```

---

# 51. Return values from failed attempts

Los resultados de un attempt fallido no deberán escapar del retry boundary.

---

# 52. Captured mutable objects

Un problema más difícil:

```php
$user = $repository->find(1);

DB::transaction(function () use ($user) {
    $user->incrementBalance();
}, retry: 3);
```

El mismo objeto capturado puede mutarse varias veces.

---

# 53. Retry callback guidance

Preferir:

```php
DB::transaction(function () use ($userId) {
    $user = User::findOrFail($userId);

    $user->incrementBalance();
}, retry: 3);
```

Es decir:

```text
capture stable identifiers
```

en lugar de:

```text
capture mutable managed entities
```

---

# 54. Strict retry mode

VoltStack podrá proporcionar una política que detecte o advierta sobre:

- managed entities capturadas;
- active EntityManager externo;
- pending UoW state antes del retry boundary;
- external transaction participation.

---

# 55. Existing transaction problem

Considérese:

```text
Outer TX
    ↓
Inner REQUIRED with retry=3
```

El inner participa en la transacción física outer.

No puede simplemente:

```text
rollback outer
restart inner
```

porque no es owner.

---

# 56. Retry ownership invariant

> **Solo el owner de un retryable physical transaction boundary podrá reiniciar dicha transacción física.**

---

# 57. Joined scope behavior

Para:

```text
REQUIRED
+
existing physical transaction
```

el inner retry policy normalmente quedará:

```text
DEFERRED_TO_OUTER_BOUNDARY
```

---

# 58. Joined failure

```text
Inner fails with deadlock
    ↓
physical transaction aborted/rollback-only
    ↓
failure propagates
    ↓
outer retry owner decides
```

---

# 59. Nested retry policy

Podrá existir:

```php
enum NestedRetryBehavior
{
    case DEFER_TO_OWNER;
    case SAVEPOINT_IF_SAFE;
    case REJECT;
}
```

---

# 60. Default nested behavior

Para joined transactions:

```text
DEFER_TO_OWNER
```

será la opción segura.

---

# 61. NESTED + savepoint retry

Una transacción:

```text
Outer TX
    ↓
NESTED
    ↓
Savepoint
```

podría reintentar solo el inner scope si:

```text
error is savepoint recoverable
+
rollback to savepoint confirmed
+
ORM checkpoint recoverable
+
side effects safe
```

---

# 62. Savepoint retry is specialized

No será el mismo mecanismo que:

```text
full transaction retry
```

---

# 63. REQUIRES_NEW

Un inner:

```text
REQUIRES_NEW
```

posee su propia transacción física.

Por tanto podrá tener:

```text
its own retry boundary
```

---

# 64. REQUIRES_NEW retry example

```text
Outer TX-A
    ↓ suspend

Inner TX-B attempt 1
    deadlock
    rollback

Inner TX-C attempt 2
    commit

    ↓ resume

Outer TX-A
```

---

# 65. Outer retry hazard

Si inner `REQUIRES_NEW` ya hizo commit y posteriormente outer falla:

```text
Outer attempt 1
    Inner REQUIRES_NEW COMMIT
    Outer deadlock
    Outer rollback

Outer attempt 2
    Inner REQUIRES_NEW executes again
```

Esto puede duplicar el efecto inner.

---

# 66. Retry graph

Por tanto VoltStack deberá modelar:

```text
RetryBoundary
    │
    └── contains independently committed boundary
```

como un riesgo observable.

---

# 67. Diagnostic warning

Podrá generarse:

```text
Independent transaction committed inside retryable outer boundary.
Replay may duplicate independently committed effects.
```

---

# 68. Retry context

```php
final readonly class TransactionRetryContext
{
    public function __construct(
        public RetryBoundaryId $boundaryId,
        public RetryAttemptNumber $attempt,
        public int $maxAttempts,
        public Instant $startedAt,
        public Deadline $deadline,
        public ?DatabaseFailureClassification $lastFailure,
    ) {}
}
```

---

# 69. Attempt number

Se utilizará:

```text
1-based numbering
```

Ejemplo:

```text
Attempt 1
Attempt 2
Attempt 3
```

---

# 70. maxAttempts semantics

```text
maxAttempts = 3
```

significa:

```text
1 initial execution
+
up to 2 retries
```

No:

```text
1 initial + 3 retries
```

---

# 71. Alternative configuration

Para evitar ambigüedad interna se usarán conceptos distintos:

```text
maxAttempts
```

y:

```text
maxRetries
```

pero la API deberá documentar claramente cuál recibe.

Preferencia:

```text
maxAttempts
```

---

# 72. Retry budget

El sistema no deberá depender únicamente de:

```text
attempt count
```

También:

```text
time budget
```

---

# 73. RetryBudget

```php
final readonly class RetryBudget
{
    public function __construct(
        public int $maxAttempts,
        public Duration $maxElapsedTime,
    ) {}
}
```

---

# 74. Deadline

Debe cumplirse:

```text
NextAttemptStart
<
RetryDeadline
```

---

# 75. No retry beyond transaction/request deadline

El retry no podrá extender silenciosamente:

```text
request deadline
job deadline
transaction deadline
operation deadline
```

---

# 76. Effective retry deadline

Conceptualmente:

```text
EffectiveRetryDeadline
=
min(
    RetryPolicyDeadline,
    OperationDeadline,
    RequestOrJobDeadline
)
```

cuando existan.

---

# 77. Backoff

VoltStack deberá soportar:

```text
constant
linear
exponential
decorrelated
```

como políticas posibles.

---

# 78. Exponential backoff

Ejemplo:

```text
delay(n)
=
base × 2^(n-1)
```

con límite máximo.

---

# 79. Capped backoff

```text
delay(n)
=
min(
    maxDelay,
    base × 2^(n-1)
)
```

---

# 80. Jitter

Para evitar:

```text
many workers deadlock
    ↓
all retry at same moment
    ↓
collision again
```

se utilizará jitter.

---

# 81. Full jitter

Ejemplo:

```text
delay
=
random(0, exponentialCap)
```

---

# 82. Jitter source

La aleatoriedad deberá provenir de un componente inyectable:

```php
interface RetryJitterSource
{
    public function apply(Duration $delay): Duration;
}
```

Esto facilita tests deterministas.

---

# 83. BackoffPolicy

```php
interface RetryBackoffPolicy
{
    public function delay(
        RetryAttemptNumber $attempt,
        DatabaseFailureClassification $failure,
    ): Duration;
}
```

---

# 84. Cancellation

Durante backoff, el sistema deberá observar:

```text
CancellationToken
```

---

# 85. Cancel-aware sleep

No deberá hacer simplemente:

```php
sleep(30);
```

ignorando:

- request cancellation;
- worker shutdown;
- job cancellation;
- deadline expiration.

---

# 86. RetrySleeper

```php
interface RetrySleeper
{
    public function sleep(
        Duration $duration,
        CancellationToken $cancellation,
    ): void;
}
```

---

# 87. Runtime-aware sleeper

La implementación podrá variar entre:

```text
traditional PHP
FrankenPHP
RoadRunner
OpenSwoole
```

sin cambiar el Retry Engine.

---

# 88. OpenSwoole

En coroutine runtime:

```text
backoff
```

no deberá bloquear innecesariamente todo el worker.

---

# 89. Retry loop

Pseudocódigo:

```php
for ($attempt = 1; $attempt <= $policy->maxAttempts(); $attempt++) {
    try {
        return $this->runAttempt(
            attempt: $attempt,
            callback: $callback,
        );
    } catch (Throwable $e) {
        $failure = $this->classifier->classify($e);

        $decision = $policy->decide(
            $context,
            $failure,
        );

        if (!$decision->isRetryable()) {
            throw $e;
        }

        $this->prepareNextAttempt();

        $this->sleeper->sleep(
            $decision->delay,
            $cancellation,
        );
    }
}
```

La implementación real deberá manejar explícitamente rollback, UNKNOWN, cleanup y composite failures.

---

# 90. Correct attempt pipeline

```text
Create Attempt Context
        ↓
Create Fresh Transaction Context
        ↓
Create Fresh Persistence Context if required
        ↓
BEGIN
        ↓
Execute Callback
        ↓
COMMIT
        ↓
Confirmed?
   /           \
 YES           NO
  │             │
SUCCESS      classify
```

---

# 91. Failure pipeline

```text
Failure
   ↓
Capture Primary Failure
   ↓
Determine Transaction State
   ↓
Rollback If Required/Possible
   ↓
Determine Rollback Outcome
   ↓
Classify Failure
   ↓
Check Outcome Certainty
   ↓
Cleanup Attempt Context
   ↓
Evaluate Retry
```

---

# 92. Cleanup before next attempt

Antes de Attempt N+1:

```text
Previous TransactionContext
    → terminal

Previous connection lease
    → released/discarded appropriately

Previous SavepointStack
    → cleared

Previous callbacks
    → discarded

Previous PersistenceContext
    → reset/discarded

Previous retry-attempt state
    → archived observationally
```

---

# 93. Connection reuse

Una nueva tentativa podrá utilizar la misma conexión física únicamente si:

- la anterior terminó limpiamente;
- la conexión fue correctamente reseteada;
- no está tainted;
- pool/connection policy lo permite.

No existe requisito de usar una conexión distinta.

---

# 94. Tainted connection

```text
Connection TAINTED
→
do not reuse for retry
```

---

# 95. Connection replacement

El Connection Manager podrá descartar la conexión y adquirir otra antes del siguiente attempt.

---

# 96. Isolation preservation

Todos los attempts deberán conservar la misma:

```text
Requested Isolation Semantics
```

salvo que una política explícita permita adaptación.

---

# 97. No silent isolation weakening

Prohibido:

```text
Attempt 1 SERIALIZABLE fails
Attempt 2 READ_COMMITTED
```

sin una política explícita.

---

# 98. No silent isolation strengthening

Tampoco deberá cambiarse arbitrariamente en sentido contrario si altera semántica/performance.

---

# 99. Transaction definition stability

Todos los attempts compartirán una definición lógica estable:

```text
isolation
read mode
timeout policy
routing intent
tenant
database
shard
propagation boundary
```

---

# 100. Tenant stability

```text
Tenant(Attempt1)
=
Tenant(Attempt2)
```

para el mismo retry boundary.

---

# 101. Shard stability

Igualmente:

```text
Shard(Attempt1)
=
Shard(Attempt2)
```

salvo una estrategia distribuida explícita fuera del core retry.

---

# 102. Read/write routing

Un nuevo attempt podrá resolver una nueva writer connection si la anterior falló, siempre que:

```text
previous outcome is known safe to retry
```

---

# 103. Replica reads

Una retryable write transaction no deberá saltar arbitrariamente a replicas entre attempts si eso cambia garantías de consistencia.

---

# 104. Data may change between attempts

Importante:

```text
Attempt 1 reads X = 10
Attempt 2 reads X = 12
```

puede ser correcto.

Retry:

```text
≠ deterministic replay of database snapshot
```

---

# 105. Business logic implication

El callback debe soportar que:

```text
queries return different values
```

entre attempts.

---

# 106. Generated IDs

Attempt 1 puede obtener:

```text
id = 100
```

y hacer rollback.

Attempt 2 puede obtener:

```text
id = 101
```

Esto no es error.

---

# 107. ID invariant

La aplicación no deberá depender de que generated IDs sean iguales entre attempts.

---

# 108. Random values

Si el callback genera:

```text
UUID
random token
timestamp
```

cada attempt puede producir valores distintos.

Esto puede o no ser deseable.

---

# 109. Retry-stable values

VoltStack podrá ofrecer un:

```text
RetryScope
```

para generar ciertos valores una vez por retry boundary.

Ejemplo conceptual:

```php
$orderId = $retryContext->stable(
    'order-id',
    fn () => Uuid::v7(),
);
```

---

# 110. Stable value store

Este store deberá contener únicamente valores explícitamente registrados y seguros.

No deberá convertirse en:

```text
global mutable application state
```

---

# 111. Retry stable timestamp

Para determinados casos:

```text
created_at
```

podría conservarse entre attempts si la aplicación así lo solicita.

Pero no será automático para todas las llamadas al clock.

---

# 112. Database-generated values

No pueden garantizarse estables entre attempts.

---

# 113. Query result cache

Los caches request-local derivados del intento anterior no deberán contaminar automáticamente el siguiente attempt.

---

# 114. Transaction-local cache

Todo:

```text
transaction-local query/result cache
```

deberá reiniciarse entre attempts.

---

# 115. Entity cache

Un second-level entity cache requerirá invalidation/consistency policies separadas.

Retry System no deberá modificarlo arbitrariamente.

---

# 116. Retry policy DSL

Ejemplo futuro:

```php
RetryPolicy::transaction()
    ->maxAttempts(4)
    ->onDeadlock()
    ->onSerializationFailure()
    ->backoff(
        Backoff::exponential(
            base: 10,
            max: 500,
            unit: 'ms',
        )
    )
    ->jitter();
```

---

# 117. Policy immutability

Las policies deberán ser:

```text
immutable
```

y reutilizables entre requests.

---

# 118. Runtime context mutable

En cambio:

```text
attempt number
last failure
elapsed time
current delay
```

serán operation-local.

---

# 119. Policy composition

Podrá existir:

```text
BaseRetryPolicy
    +
FailureFilter
    +
BackoffPolicy
    +
JitterPolicy
    +
Budget
```

---

# 120. Configuration

Ejemplo:

```php
'database' => [
    'transactions' => [
        'retry' => [
            'default' => [
                'enabled' => false,
                'max_attempts' => 3,
                'max_elapsed_ms' => 3000,
            ],
        ],
    ],
];
```

---

# 121. Why disabled by default

Porque VoltStack no puede demostrar automáticamente que:

```text
arbitrary PHP callback
```

sea:

```text
safe to execute repeatedly
```

---

# 122. Framework-owned internal retries

VoltStack podrá habilitar retry internamente para operaciones que controle completamente y cuya idempotencia/replay semantics conozca.

---

# 123. Application retry opt-in

Para callbacks arbitrarios de usuario:

```text
explicit opt-in
```

será preferido.

---

# 124. Error propagation

Si el retry budget se agota:

```text
last transactional failure
```

deberá preservarse como causa.

---

# 125. RetryExhaustedException

```php
final class TransactionRetryExhaustedException
    extends DatabaseTransactionException
{
    public function attempts(): int;

    public function lastFailure(): Throwable;
}
```

---

# 126. Preserve original errors

No reemplazar:

```text
deadlock
```

por un genérico:

```text
retry failed
```

sin conservar la causa.

---

# 127. Attempt history

Diagnostics podrá conservar:

```text
Attempt 1 → DEADLOCK
Attempt 2 → SERIALIZATION_FAILURE
Attempt 3 → SUCCESS
```

---

# 128. Bounded history

La historia deberá ser:

```text
bounded
```

para evitar crecimiento de memoria.

---

# 129. RetryOutcome

```php
enum TransactionRetryOutcome
{
    case SUCCEEDED_FIRST_ATTEMPT;
    case SUCCEEDED_AFTER_RETRY;
    case EXHAUSTED;
    case NOT_RETRYABLE;
    case CANCELLED;
    case DEADLINE_EXCEEDED;
    case UNKNOWN;
}
```

---

# 130. Cancellation outcome

Si se cancela durante backoff:

```text
CANCELLED
```

No deberá iniciar otro attempt.

---

# 131. Deadline outcome

Si no existe tiempo suficiente:

```text
DEADLINE_EXCEEDED
```

---

# 132. Minimum attempt budget

Podrá existir una política que evite iniciar un nuevo attempt cuando:

```text
remaining time
<
minimum estimated attempt budget
```

aunque la estimación deberá ser conservadora.

---

# 133. Deadlock integration

El documento siguiente definirá:

```text
171_DATABASE_DEADLOCK_HANDLING_SYSTEM.md
```

Deadlock System deberá producir información como:

```text
FailureClass:
    DEADLOCK

TransactionEffect:
    ABORTED

RetryRecommendation:
    RETRY_TRANSACTION
```

sin ejecutar por sí mismo el retry.

---

# 134. Separation

```text
Deadlock System
→ identifies/classifies

Retry System
→ decides/replays
```

---

# 135. Serialization failures

Isolation System podrá identificar:

```text
SERIALIZATION_FAILURE
```

y Retry System decidir si reinicia toda la transacción.

---

# 136. Lock timeout

Un lock timeout podrá ser:

```text
retryable
```

o no dependiendo de:

- plataforma;
- transaction effect;
- policy;
- deadline;
- number of attempts.

---

# 137. Connection transient failure

Una conexión que falla antes de:

```text
BEGIN
```

puede ser mucho más segura de reintentar que una que falla durante:

```text
COMMIT
```

---

# 138. Failure phase

La clasificación deberá incluir:

```php
enum TransactionFailurePhase
{
    case ACQUIRE_CONNECTION;
    case BEGIN;
    case EXECUTE;
    case SAVEPOINT;
    case COMMIT;
    case ROLLBACK;
    case CLEANUP;
}
```

---

# 139. Phase matters

Ejemplo:

```text
connection failure during ACQUIRE_CONNECTION
```

puede ser retryable.

Pero:

```text
connection failure after COMMIT sent
```

puede producir UNKNOWN.

---

# 140. TransactionOutcomeEvidence

```php
enum TransactionOutcomeEvidence
{
    case NOT_STARTED;
    case ACTIVE_ABORTED;
    case ROLLBACK_CONFIRMED;
    case COMMIT_CONFIRMED;
    case COMMIT_NOT_PERFORMED;
    case UNKNOWN;
}
```

---

# 141. Retry eligibility formula

```text
RetryEligible
=
FailureIsTransient
∧ BoundaryOwned
∧ OutcomeSafe
∧ PolicyAllows
∧ BudgetAvailable
∧ DeadlineAvailable
∧ ContextRecoverable
∧ NotCancelled
```

---

# 142. OutcomeSafe

Conceptualmente:

```text
OutcomeSafe
=
NOT_STARTED
∨
ROLLBACK_CONFIRMED
∨
PLATFORM_PROVEN_ABORTED
```

No:

```text
COMMIT_UNKNOWN
```

---

# 143. Retry after constraint violation

No deberá hacerse:

```text
UNIQUE constraint violation
→ sleep
→ retry same transaction
```

por default.

---

# 144. Retry after syntax error

Nunca resolverá:

```text
SQL syntax error
```

repetir exactamente la misma operación.

---

# 145. Retry storm prevention

En sistemas con muchos workers:

```text
failure
→ thousands of retries
```

puede empeorar el incidente.

Por ello se requieren:

```text
jitter
attempt limits
time budgets
resource governance
```

---

# 146. Global retry pressure

VoltStack podrá integrar un:

```text
RetryPressureMonitor
```

que observe tasas de retry.

No deberá introducir estado global mutable inseguro en el core transaccional.

---

# 147. Circuit breaker integration

En futuras integraciones:

```text
CircuitBreaker
```

podrá impedir nuevos attempts ante un fallo sistémico.

Pero:

```text
Circuit Breaker
≠
Transaction Retry
```

---

# 148. Resource governance

Podrán existir límites:

```text
max attempts
max elapsed retry time
max backoff
max concurrent retrying operations
```

---

# 149. Retry concurrency limit

Una implementación de runtime podrá imponer:

```text
retry concurrency budget
```

para evitar saturación.

---

# 150. Telemetry

Métricas:

```text
db.transaction.retry.attempts
db.transaction.retry.success
db.transaction.retry.exhausted
db.transaction.retry.cancelled
db.transaction.retry.delay
db.transaction.retry.failure_class
```

---

# 151. Metric labels

Etiquetas razonables:

```text
platform
failure_class
outcome
attempt_bucket
```

Evitar:

```text
TransactionId
RetryBoundaryId
UserId
SQL
EntityId
```

---

# 152. Tracing

Estructura:

```text
transaction.retry
│
├── attempt 1
│   └── transaction TX-1
│
├── backoff
│
└── attempt 2
    └── transaction TX-2
```

---

# 153. Retry span attributes

Ejemplo:

```text
db.retry.attempt=2
db.retry.max_attempts=3
db.retry.reason=deadlock
db.retry.delay_ms=47
```

---

# 154. Diagnostics

Ejemplo:

```text
TRANSACTION RETRY

Boundary:
    retry-boundary-14

Attempt:
    2 / 3

Previous Failure:
    DEADLOCK

Previous Transaction:
    ROLLED_BACK

Outcome Certainty:
    CONFIRMED

Backoff:
    42 ms

Retry:
    ELIGIBLE
```

---

# 155. Unsafe diagnostic

```text
TRANSACTION RETRY REJECTED

Failure:
    CONNECTION_LOST_DURING_COMMIT

Transaction Outcome:
    UNKNOWN

Automatic Retry:
    DISABLED

Reason:
    Replaying may duplicate committed effects.
```

---

# 156. ORM diagnostic

```text
TRANSACTION RETRY

Attempt:
    2

Previous Persistence Context:
    DISCARDED

New Persistence Context:
    CREATED

Captured Managed Entities:
    warning

Recommendation:
    Capture identifiers and reload inside retry callback.
```

---

# 157. Explain API

Podrá existir:

```php
DB::transactions()
    ->retries()
    ->explain($exception);
```

Resultado:

```text
Failure:
    DEADLOCK

Transaction Effect:
    ABORTED

Policy:
    transient-retry

Attempts:
    1 / 3

Deadline:
    sufficient

Outcome:
    known

Decision:
    RETRY

Delay:
    38 ms
```

---

# 158. Exception hierarchy

```text
DatabaseTransactionRetryException
│
├── TransactionRetryExhaustedException
├── TransactionRetryNotAllowedException
├── TransactionRetryUnsafeException
├── TransactionRetryUnknownOutcomeException
├── TransactionRetryCancelledException
├── TransactionRetryDeadlineExceededException
├── TransactionRetryContextException
├── TransactionRetryOwnershipException
├── TransactionRetryCleanupException
├── TransactionRetryPolicyException
├── TransactionRetryBudgetException
└── TransactionRetryInvariantViolationException
```

---

# 159. Retry unsafe exception

Se utilizará cuando:

```text
policy requests retry
```

pero el sistema detecta:

```text
unsafe transaction outcome
```

---

# 160. Unknown outcome exception

Debe preservar información suficiente para que capas superiores puedan realizar reconciliación.

---

# 161. Cleanup exception

Si rollback fue exitoso pero:

```text
PersistenceContext cleanup
```

falla, el attempt no deberá reutilizar dicho contexto.

---

# 162. Composite failures

Puede ocurrir:

```text
Primary:
    DEADLOCK

Rollback:
    success

Cleanup:
    failure
```

o:

```text
Primary:
    DEADLOCK

Rollback:
    failure
```

Todos los fallos relevantes deberán preservarse.

---

# 163. Retry after rollback failure

Por default:

```text
Rollback Failure
→
NO RETRY
```

hasta determinar el estado físico.

---

# 164. Testing architecture

La suite deberá incluir:

```text
TransactionRetryUnitTests
TransactionRetryPolicyTests
RetryEligibilityTests
RetryBudgetTests
RetryBackoffTests
RetryJitterTests
RetryCancellationTests
RetryDeadlineTests
RetryFailureClassificationTests
RetryUnknownOutcomeTests
RetryConnectionFailureTests
RetryDeadlockTests
RetrySerializationTests
RetryNestedTransactionTests
RetryRequiresNewTests
RetrySavepointTests
RetryORMTests
RetryPersistentRuntimeTests
RetryTelemetryTests
RetryFailureInjectionTests
```

---

# 165. Basic retry test

Simular:

```text
Attempt 1:
    deadlock
    rollback confirmed

Attempt 2:
    commit
```

Esperado:

```text
callback executions = 2
committed transactions = 1
```

---

# 166. Exhaustion test

```text
Attempt 1 → deadlock
Attempt 2 → deadlock
Attempt 3 → deadlock
```

Esperado:

```text
TransactionRetryExhaustedException
```

con último deadlock como causa.

---

# 167. Permanent failure test

```text
Attempt 1:
    unique violation
```

Esperado:

```text
no second attempt
```

---

# 168. UNKNOWN test

```text
Attempt 1:
    COMMIT sent
    connection lost
```

Esperado:

```text
no automatic retry
```

---

# 169. Backoff test

Con clock y jitter deterministas verificar:

```text
attempt delays
```

sin utilizar sleeps reales.

---

# 170. Cancellation test

Cancelar durante backoff.

Esperado:

```text
no next transaction
```

---

# 171. Deadline test

Si el delay excede el tiempo restante:

```text
deadline exceeded
```

sin nuevo attempt.

---

# 172. ORM test

Attempt 1:

```text
EM-1
deadlock
```

Attempt 2:

```text
EM-2
```

Verificar:

```text
EM-1 != EM-2
UoW-1 != UoW-2
IdentityMap-1 != IdentityMap-2
```

---

# 173. Captured entity test

Probar una entidad managed capturada desde fuera del retry boundary y verificar:

```text
warning / rejection
```

en strict mode.

---

# 174. Generated ID test

```text
Attempt 1 → generated 100 → rollback
Attempt 2 → generated 101 → commit
```

El resultado correcto deberá usar:

```text
101
```

---

# 175. Nested REQUIRED test

```text
Outer retry owner
    Inner REQUIRED deadlocks
```

El inner no deberá iniciar su propio full transaction retry.

---

# 176. REQUIRES_NEW test

Verificar:

```text
outer suspended
inner attempt 1 rollback
inner attempt 2 commit
outer resumed
```

---

# 177. Independent commit hazard test

```text
outer retryable
    inner REQUIRES_NEW commit
outer fails
outer retries
```

Verificar que telemetry/diagnostics detecten el riesgo.

---

# 178. Savepoint retry test

Solo habilitar si:

```text
failure recoverable
rollback-to confirmed
persistence state restored
```

---

# 179. Persistent runtime test

Request A:

```text
retry attempt = 3
```

Request B deberá comenzar:

```text
attempt context = none
```

---

# 180. Coroutine test

Dos coroutines con retries simultáneos deberán mantener:

```text
independent attempt counters
independent deadlines
independent transaction contexts
```

---

# 181. Failure injection matrix

Inyectar fallos en:

```text
connection acquisition
BEGIN
callback before SQL
statement execution
savepoint
flush
COMMIT before send
COMMIT after send
ROLLBACK
cleanup
backoff
context reset
```

---

# 182. Proposed directory structure

```text
src/Quantum/Database/Transaction/
│
├── Retry/
│   ├── TransactionRetryEngine.php
│   ├── TransactionRetryContext.php
│   ├── TransactionRetryAttempt.php
│   ├── RetryAttemptNumber.php
│   ├── RetryBoundaryId.php
│   ├── RetryAttemptState.php
│   ├── TransactionRetryOutcome.php
│   │
│   ├── Policy/
│   │   ├── TransactionRetryPolicy.php
│   │   ├── NoRetryPolicy.php
│   │   ├── TransientRetryPolicy.php
│   │   ├── CompositeRetryPolicy.php
│   │   ├── RetryDecision.php
│   │   └── RetryEligibility.php
│   │
│   ├── Budget/
│   │   ├── RetryBudget.php
│   │   └── RetryBudgetEvaluator.php
│   │
│   ├── Backoff/
│   │   ├── RetryBackoffPolicy.php
│   │   ├── ConstantBackoff.php
│   │   ├── ExponentialBackoff.php
│   │   └── RetryJitterSource.php
│   │
│   ├── Failure/
│   │   ├── DatabaseFailureClassifier.php
│   │   ├── DatabaseFailureClassification.php
│   │   ├── DatabaseFailureClass.php
│   │   ├── TransactionFailurePhase.php
│   │   └── TransactionOutcomeEvidence.php
│   │
│   ├── Runtime/
│   │   ├── RetrySleeper.php
│   │   ├── RetryScope.php
│   │   └── RetryStableValueStore.php
│   │
│   ├── Persistence/
│   │   ├── PersistenceRetryCoordinator.php
│   │   └── PersistenceContextResetPolicy.php
│   │
│   ├── Diagnostics/
│   │   ├── TransactionRetryInspector.php
│   │   ├── RetryAttemptHistory.php
│   │   └── RetryDiagnostic.php
│   │
│   └── Exception/
│       └── ...
```

---

# 183. Dependency rules

Permitido:

```text
Retry System
    ↓
Transaction Manager contracts
Transaction Context contracts
Failure Classification
Connection contracts
Runtime cancellation
Clock
Telemetry
Persistence reset contracts
```

No permitido:

```text
Retry System
    ↓
PDO directly
vendor SQL parsing in core
HTTP globals
business services
static current attempt
static retry counter
```

---

# 184. Architectural invariants

## DB-TR-001
Transaction retry operará sobre un retry boundary explícito.

## DB-TR-002
Transaction retry no será statement retry.

## DB-TR-003
Transaction retry no será query retry.

## DB-TR-004
Transaction retry no será connection retry.

## DB-TR-005
Cada attempt tendrá una nueva physical transaction.

## DB-TR-006
Cada attempt tendrá un nuevo PhysicalTransactionId.

## DB-TR-007
Todos los attempts compartirán RetryBoundaryId.

## DB-TR-008
Attempt numbering será determinista.

## DB-TR-009
`maxAttempts` incluirá el intento inicial.

## DB-TR-010
Retry policy será explícita.

## DB-TR-011
Callbacks arbitrarios no se asumirán retry-safe.

## DB-TR-012
Automatic application retry será opt-in por default.

## DB-TR-013
Framework-owned operations podrán usar policies explícitas propias.

## DB-TR-014
Retryability no dependerá solo del error.

## DB-TR-015
Retryability considerará transaction outcome.

## DB-TR-016
Retryability considerará ownership.

## DB-TR-017
Retryability considerará budget.

## DB-TR-018
Retryability considerará deadline.

## DB-TR-019
Retryability considerará cancellation.

## DB-TR-020
Retryability considerará context recoverability.

## DB-TR-021
UNKNOWN no será automáticamente retryable.

## DB-TR-022
COMMIT UNKNOWN no será automáticamente retryable.

## DB-TR-023
Rollback confirmed podrá habilitar retry.

## DB-TR-024
Platform-proven transaction abort podrá habilitar retry.

## DB-TR-025
Rollback failure bloqueará retry por default.

## DB-TR-026
Failure classification será canónica.

## DB-TR-027
Vendor error parsing no estará disperso en Retry System.

## DB-TR-028
Deadlock detection será responsabilidad separada.

## DB-TR-029
Serialization failure classification será responsabilidad separada.

## DB-TR-030
Constraint violation no será retryable por default.

## DB-TR-031
Syntax error no será retryable.

## DB-TR-032
Authentication failure no será retryable por default.

## DB-TR-033
Permission failure no será retryable por default.

## DB-TR-034
Failure phase será considerada.

## DB-TR-035
Connection failure before BEGIN será distinta de failure during COMMIT.

## DB-TR-036
Commit ambiguity conservará UNKNOWN.

## DB-TR-037
Retry no fabricará rollback certainty.

## DB-TR-038
Retry no fabricará commit certainty.

## DB-TR-039
Previous attempt deberá alcanzar estado terminal antes del siguiente cuando sea demostrable.

## DB-TR-040
Attempt contexts serán independientes.

## DB-TR-041
Savepoint stacks serán independientes entre physical attempts.

## DB-TR-042
Transaction callbacks serán attempt-local.

## DB-TR-043
afterCommit de failed attempt será descartado.

## DB-TR-044
afterRollback callbacks seguirán semántica del attempt correspondiente.

## DB-TR-045
PersistenceContext podrá ser recreado entre attempts.

## DB-TR-046
UnitOfWork no se asumirá restaurado por rollback.

## DB-TR-047
IdentityMap no se asumirá restaurado por rollback.

## DB-TR-048
PHP object graph no se asumirá restaurado por rollback.

## DB-TR-049
Fresh PersistenceContext será default seguro para ORM retry.

## DB-TR-050
Entities managed de attempts anteriores no se asumirán válidas.

## DB-TR-051
Retry callback debería capturar IDs antes que managed entities.

## DB-TR-052
Strict mode podrá detectar managed entities externas.

## DB-TR-053
Generated IDs podrán cambiar entre attempts.

## DB-TR-054
Database sequence values podrán cambiar entre attempts.

## DB-TR-055
Query results podrán cambiar entre attempts.

## DB-TR-056
Retry no prometerá snapshot replay idéntico.

## DB-TR-057
Random application values podrán cambiar entre attempts.

## DB-TR-058
Retry-stable values requerirán declaración explícita.

## DB-TR-059
RetryStableValueStore será boundary-local.

## DB-TR-060
RetryStableValueStore no será global.

## DB-TR-061
Transaction definition permanecerá semánticamente estable entre attempts.

## DB-TR-062
Isolation no se debilitará silenciosamente.

## DB-TR-063
Isolation no cambiará silenciosamente.

## DB-TR-064
Tenant permanecerá estable entre attempts.

## DB-TR-065
Database context permanecerá estable entre attempts.

## DB-TR-066
Shard permanecerá estable entre attempts.

## DB-TR-067
Routing conservará consistency requirements.

## DB-TR-068
Tainted connection no se reutilizará.

## DB-TR-069
Connection Manager podrá adquirir nueva conexión.

## DB-TR-070
Retry System no gestionará directamente pools.

## DB-TR-071
Joined inner transaction no poseerá retry boundary físico.

## DB-TR-072
Joined retry será delegado al physical owner por default.

## DB-TR-073
Inner REQUIRED no reiniciará outer TX arbitrariamente.

## DB-TR-074
REQUIRES_NEW podrá tener retry independiente.

## DB-TR-075
REQUIRES_NEW commit podrá sobrevivir al outer attempt.

## DB-TR-076
Retryable outer con inner committed independiente será diagnosticable.

## DB-TR-077
NESTED savepoint retry será estrategia especializada.

## DB-TR-078
Savepoint retry requerirá rollback-to confirmado.

## DB-TR-079
Savepoint retry requerirá error recuperable.

## DB-TR-080
Savepoint retry requerirá persistence state recuperable.

## DB-TR-081
Savepoint retry requerirá side effects seguros.

## DB-TR-082
Savepoint retry no sustituirá full transaction retry.

## DB-TR-083
External side effects no serán revertidos por DB rollback.

## DB-TR-084
Retry puede ejecutar callback múltiples veces.

## DB-TR-085
El contrato público documentará callback replay.

## DB-TR-086
Email sending dentro de retry callback no se asumirá idempotente.

## DB-TR-087
Queue dispatch dentro de retry callback no se asumirá reversible.

## DB-TR-088
Filesystem writes no se asumirán reversibles.

## DB-TR-089
Remote API calls no se asumirán reversibles.

## DB-TR-090
Outbox será integración recomendada para durable side effects.

## DB-TR-091
afterCommit no ejecutará side effect antes del commit confirmado.

## DB-TR-092
afterCommit failure no revertirá DB commit.

## DB-TR-093
Retry tendrá max attempts.

## DB-TR-094
Retry tendrá time budget.

## DB-TR-095
Retry respetará operation deadline.

## DB-TR-096
Retry respetará cancellation.

## DB-TR-097
Backoff será policy-driven.

## DB-TR-098
Jitter será soportado.

## DB-TR-099
Jitter source será testable.

## DB-TR-100
Retry sleep será runtime-aware.

## DB-TR-101
OpenSwoole backoff no bloqueará globalmente el worker cuando exista soporte coroutine-aware.

## DB-TR-102
No se iniciará un attempt después de cancellation.

## DB-TR-103
No se iniciará un attempt después del deadline.

## DB-TR-104
Retry budget exhaustion será explícito.

## DB-TR-105
RetryExhaustedException preservará last failure.

## DB-TR-106
Primary failure no será ocultado por cleanup failure.

## DB-TR-107
Composite failures podrán conservarse.

## DB-TR-108
Retry history será bounded.

## DB-TR-109
Retry telemetry será observational.

## DB-TR-110
Telemetry no decidirá retry semantics.

## DB-TR-111
Metric labels evitarán RetryBoundaryId.

## DB-TR-112
Metric labels evitarán TransactionId.

## DB-TR-113
Metric labels evitarán raw SQL.

## DB-TR-114
Diagnostics podrán mostrar attempt history.

## DB-TR-115
Diagnostics mostrarán failure class.

## DB-TR-116
Diagnostics mostrarán outcome certainty.

## DB-TR-117
Diagnostics mostrarán retry decision.

## DB-TR-118
Diagnostics mostrarán backoff.

## DB-TR-119
Persistent workers no compartirán retry contexts.

## DB-TR-120
FrankenPHP requests tendrán retry state aislado.

## DB-TR-121
RoadRunner operations tendrán retry state aislado.

## DB-TR-122
OpenSwoole coroutines tendrán retry state aislado.

## DB-TR-123
Retry counters no serán static globals.

## DB-TR-124
Current attempt no será static global.

## DB-TR-125
Policies inmutables podrán compartirse.

## DB-TR-126
Backoff definitions inmutables podrán compartirse.

## DB-TR-127
Attempt state mutable será operation-local.

## DB-TR-128
Retry System no ejecutará SQL directamente.

## DB-TR-129
Retry System no realizará vendor-specific locking.

## DB-TR-130
Retry System no será un segundo TransactionManager.

## DB-TR-131
Retry System utilizará TransactionManager para cada attempt.

## DB-TR-132
Retry System consumirá Failure Classifier.

## DB-TR-133
Retry System consumirá Cancellation abstractions.

## DB-TR-134
Retry System consumirá Clock abstractions.

## DB-TR-135
Retry System podrá consumir Persistence Reset contracts.

## DB-TR-136
Retry System no dependerá de Entity implementation.

## DB-TR-137
Retry System no dependerá de HTTP globals.

## DB-TR-138
Retry System no dependerá de business services.

## DB-TR-139
Retry storms serán limitados mediante budgets/backoff/jitter.

## DB-TR-140
Retries infinitos estarán prohibidos por default.

## DB-TR-141
Failure UNKNOWN no será convertido a TRANSIENT.

## DB-TR-142
Un error transient con outcome UNKNOWN seguirá siendo unsafe por default.

## DB-TR-143
Error class y outcome certainty serán dimensiones separadas.

## DB-TR-144
Transaction phase y failure class serán dimensiones separadas.

## DB-TR-145
Attempt success requerirá commit confirmado cuando exista commit.

## DB-TR-146
Callback success no será transaction success.

## DB-TR-147
Statement success no será transaction success.

## DB-TR-148
Flush success no será transaction success.

## DB-TR-149
Commit sent no será commit confirmed.

## DB-TR-150
Cuando VoltStack no pueda demostrar que repetir una transacción es seguro, no la repetirá automáticamente.

---

# 185. Anti-patterns

## 185.1 Retry del statement fallido

```text
UPDATE fails
→ retry UPDATE
```

sin analizar la transacción completa.

**Rechazado.**

---

## 185.2 Retry infinito

```php
while (true) {
    tryTransaction();
}
```

**Prohibido como política framework.**

---

## 185.3 Retry de COMMIT UNKNOWN

```text
COMMIT sent
connection lost
→ retry transaction
```

**Prohibido por default.**

---

## 185.4 Reutilizar UoW sucio

```text
deadlock
rollback
retry with same dirty UnitOfWork
```

**Rechazado como default.**

---

## 185.5 Reutilizar callbacks afterCommit

```text
Attempt 1 registers callback
rollback
Attempt 2 registers callback
commit
→ execute both
```

**Prohibido.**

---

## 185.6 Retry de unique violation

```text
duplicate key
→ sleep
→ same insert
```

**Rechazado por default.**

---

## 185.7 External side effects sin idempotencia

```text
send payment
DB deadlock
retry
send payment again
```

**Peligroso y rechazado como supuesto de seguridad.**

---

## 185.8 Inner joined retry independiente

```text
Outer TX
    Inner REQUIRED
        deadlock
        rollback outer
        retry inner only
```

**Rechazado.**

---

## 185.9 Cambiar isolation para que funcione

```text
SERIALIZABLE fails
→ silently use READ_COMMITTED
```

**Prohibido.**

---

## 185.10 Bloquear worker durante backoff

```text
coroutine runtime
→ blocking sleep
```

cuando existe primitive runtime-aware.

**Rechazado.**

---

# 186. Master formulas

## Eligibility

```text
RetryEligible
=
TransientFailure
∧ SafeOutcome
∧ BoundaryOwned
∧ PolicyAllows
∧ BudgetAvailable
∧ DeadlineAvailable
∧ ContextRecoverable
∧ ¬Cancelled
```

## Attempts

```text
TotalCallbackExecutions
≤
MaxAttempts
```

## Exponential backoff

```text
Backoff(n)
=
min(
    MaxDelay,
    BaseDelay × 2^(n-1)
)
```

## Full jitter

```text
Delay(n)
∈
[0, Backoff(n)]
```

## Effective deadline

```text
RetryDeadline
=
min(
    PolicyDeadline,
    TransactionDeadline,
    OperationDeadline
)
```

## Transaction identity

```text
PhysicalTransactionId(Attempt N)
≠
PhysicalTransactionId(Attempt N+1)
```

## Boundary identity

```text
RetryBoundaryId(Attempt N)
=
RetryBoundaryId(Attempt N+1)
```

## ORM safety

```text
Rollback
≠
ObjectGraphRewind
```

Therefore:

```text
FailedAttempt
+
ORM
→
FreshOrProvablyRestoredPersistenceContext
```

## UNKNOWN

```text
CommitOutcome = UNKNOWN
→
AutomaticRetry = FALSE
```

por default.

---

# 187. Modelo final

```text
                 Retryable Transaction Request
                            │
                            ▼
                      Retry Policy
                            │
                            ▼
                    Retry Boundary
                            │
                            ▼
                       Attempt N
                            │
                            ▼
                   Transaction Manager
                            │
                           BEGIN
                            │
                            ▼
                         Callback
                            │
                     ┌──────┴──────┐
                     ▼             ▼
                  Success        Failure
                     │             │
                     ▼             ▼
                   COMMIT      Classifier
                     │             │
              ┌──────┴─────┐       ▼
              ▼            ▼   Failure Class
          CONFIRMED     UNKNOWN      │
              │            │         ▼
              ▼            ▼   Outcome Safety
           SUCCESS      TERMINAL      │
                                      ▼
                               Retry Eligibility
                                /            \
                               ▼              ▼
                           NO RETRY         RETRY
                                              │
                                              ▼
                                          Rollback
                                              │
                                              ▼
                                      Cleanup Attempt
                                              │
                                              ▼
                                    Reset Persistence State
                                              │
                                              ▼
                                           Backoff
                                              │
                                              ▼
                                          Attempt N+1
```

---

# 188. Decisiones arquitectónicas finales

VoltStack implementará retries transaccionales sobre:

```text
RetryBoundary
```

y no sobre statements individuales.

Cada attempt será una nueva:

```text
Physical Transaction
```

mientras todos los attempts pertenecerán al mismo:

```text
RetryBoundaryId
```

La decisión de retry utilizará:

```text
Failure Classification
+
Transaction Outcome Evidence
+
Ownership
+
Policy
+
Budget
+
Deadline
+
Cancellation
+
Context Recoverability
```

La política general será conservadora.

Un error:

```text
transient
```

no será suficiente por sí mismo.

Especialmente:

```text
COMMIT UNKNOWN
```

bloqueará automatic retry por default.

VoltStack soportará:

```text
max attempts
time budgets
backoff
jitter
cancellation
deadlines
```

para evitar retry loops y retry storms.

La integración ORM reconocerá:

```text
Database Rollback
≠
PHP Object Rollback
```

por lo que el modelo recomendado utilizará un PersistenceContext limpio para cada nuevo attempt.

Los callbacks retryable deberán considerarse:

```text
multi-execution callbacks
```

y los side effects externos deberán utilizar mecanismos explícitos como:

```text
idempotency
afterCommit
outbox
```

según corresponda.

Nested transactions respetarán ownership:

```text
joined scope
→ defer retry to physical owner
```

mientras:

```text
REQUIRES_NEW
```

podrá poseer su propio retry boundary.

Savepoint retry será una estrategia especializada y únicamente se permitirá cuando pueda demostrarse recuperación suficiente tanto del estado físico como del estado de persistencia.

La regla final será:

> **VoltStack solo reintentará automáticamente una transacción cuando pueda demostrar que el intento anterior no produjo un commit incierto, que el boundary puede reproducirse desde un estado limpio y que la política explícita dispone todavía de presupuesto para hacerlo.**

---

# 189. Relación con el bloque Transaction & Concurrency

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

# 190. Siguiente documento

```text
171_DATABASE_DEADLOCK_HANDLING_SYSTEM.md
```

El siguiente documento deberá definir:

- deadlock architecture;
- deadlock classification;
- deadlock victim semantics;
- transaction-aborted detection;
- platform error normalization;
- SQLSTATE/error-code mapping;
- MySQL behavior;
- MariaDB behavior;
- PostgreSQL behavior;
- SQLite locking/conflict distinctions;
- deadlock vs lock timeout;
- deadlock vs serialization failure;
- deadlock graph concepts;
- transaction impact;
- retry recommendations;
- integration with Transaction Retry System;
- rollback handling;
- ORM state implications;
- diagnostics;
- query correlation;
- telemetry;
- observability;
- resource governance;
- testing;
- persistent-runtime safety;
- architectural invariants.