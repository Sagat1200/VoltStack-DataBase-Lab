# 86_DATABASE_EXECUTION_RETRY_SYSTEM.md

# VoltStack Quantum Database
## Execution Retry System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 86 — Execution Retry System  
**Bloque:** 7 — Execution Engine  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Execution Retry System` define la arquitectura mediante la cual VoltStack determina si una operación de base de datos fallida puede volver a ejecutarse de manera segura.

Su responsabilidad principal no es simplemente:

```text
catch error
→ sleep
→ retry
```

sino demostrar que el reintento:

```text
preserva semántica
+
no duplica efectos
+
respeta transacciones
+
respeta deadlines
+
puede reproducir inputs
+
parte desde recursos válidos
```

La frontera principal será:

```text
Execution Failure
        ↓
Retry Eligibility Analysis
        ↓
Retry Scope Resolution
        ↓
Safety Verification
        ↓
Retry Policy
        ↓
Retry Decision
        ↓
Backoff / Budget
        ↓
Runtime State Recovery
        ↓
Retry Attempt
```

Principio central:

```text
Retryable Failure
≠
Safe Retry
```

---

# 2. Objetivo fundamental

El sistema deberá responder, antes de cada reintento:

1. ¿El error es potencialmente recuperable?
2. ¿La operación llegó al servidor?
3. ¿Su resultado es conocido?
4. ¿Pudieron producirse efectos parciales?
5. ¿La operación es idempotente?
6. ¿El scope correcto es statement, transaction o execution region?
7. ¿Los parámetros pueden reproducirse?
8. ¿Los streams de entrada son replayable?
9. ¿La conexión sigue siendo válida?
10. ¿La transacción debe reiniciarse?
11. ¿El resultado ya fue parcialmente entregado?
12. ¿Existe tiempo restante?
13. ¿Queda presupuesto de reintentos?
14. ¿La política permite el retry?
15. ¿El retry preservará la semántica observable?

Si cualquiera de los requisitos críticos permanece incierto:

```text
UNKNOWN
```

deberá tratarse conservadoramente.

---

# 3. Posición arquitectónica

```text
Execution Engine
├── Query Executor
├── Statement Execution
├── Prepared Statement
├── Parameter Binding
├── Result
├── Cursor
├── Streaming
├── Timeout / Cancellation
├── Execution Error
└── Execution Retry
```

Flujo:

```text
ExecutionPlan
    ↓
QueryExecutor
    ↓
StatementExecutor
    ↓
Failure
    ↓
Execution Error System
    ↓
ExecutionFailureReport
    ↓
RetryCoordinator
    ↓
RetryEligibilityAnalyzer
    ↓
RetryDecision
    ├── DENY
    ├── RETRY_STATEMENT
    ├── RETRY_TRANSACTION
    ├── RETRY_REGION
    └── RETRY_EXECUTION
```

---

# 4. Responsabilidad

`Execution Retry System` deberá encargarse de:

- analizar retry eligibility;
- resolver retry scope;
- verificar idempotencia;
- verificar efectos parciales;
- verificar outcome certainty;
- comprobar replayability;
- comprobar estado de conexión;
- comprobar estado transaccional;
- respetar deadline;
- aplicar retry budgets;
- calcular backoff;
- aplicar jitter;
- coordinar reset de recursos;
- coordinar transaction restart;
- crear nuevos execution attempts;
- registrar retry history;
- emitir telemetry;
- preservar seguridad;
- evitar retry storms.

---

# 5. No responsabilidades

No deberá:

```text
classify native DB errors
compile SQL
modify ExecutionPlan
invent transaction boundaries
invent idempotency
buffer streams automatically
convert unsafe writes into safe writes
hide ambiguous outcomes
retry cancelled operations automatically
reset arbitrary global state
perform ORM hydration
```

La clasificación inicial pertenece a:

```text
85_DATABASE_EXECUTION_ERROR_SYSTEM
```

---

# 6. Retry architecture

```text
                  ExecutionFailureReport
                           │
                           ▼
                 RetryEligibilityAnalyzer
                           │
                           ▼
                  RetryEligibility
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         Failure      Operation      Runtime
         Evidence     Semantics       State
              │            │            │
              └────────────┼────────────┘
                           ▼
                  RetryScopeResolver
                           │
                           ▼
                    RetrySafetyGate
                           │
                           ▼
                    RetryPolicyEngine
                           │
                           ▼
                     RetryDecision
                           │
                 ┌─────────┴─────────┐
                 │                   │
               DENY                ALLOW
                                     │
                                     ▼
                              RetryBudget
                                     │
                                     ▼
                              BackoffPolicy
                                     │
                                     ▼
                             RecoveryCoordinator
                                     │
                                     ▼
                               Retry Attempt
```

---

# 7. Fórmula principal

```text
SafeRetry
=
RetryableFailure
∧
ValidRetryScope
∧
SafeOperationSemantics
∧
AcceptableOutcomeCertainty
∧
NoForbiddenPartialEffects
∧
ReplayableInputs
∧
RecoverableRuntimeState
∧
TransactionSafety
∧
DeadlineRemaining
∧
RetryBudgetAvailable
∧
PolicyAllows
```

---

# 8. Retryability ≠ eligibility

El documento 85 puede clasificar:

```text
DEADLOCK
→ RETRYABLE_AFTER_TRANSACTION_RESTART
```

Esto sólo constituye evidencia.

No constituye todavía:

```text
RetryDecision::ALLOW
```

---

# 9. RetryEligibility

```php
final readonly class RetryEligibility
{
    public function __construct(
        public RetryEligibilityStatus $status,
        public RetryScope $scope,
        public RetrySafety $safety,
        public IdempotencyClassification $idempotency,
        public ReplayabilityClassification $replayability,
        public OutcomeCertainty $outcome,
        public PartialEffectClassification $partialEffects,
        public array $reasons,
    ) {}
}
```

---

# 10. RetryEligibilityStatus

```php
enum RetryEligibilityStatus: string
{
    case ELIGIBLE = 'eligible';
    case CONDITIONALLY_ELIGIBLE = 'conditionally_eligible';
    case INELIGIBLE = 'ineligible';
    case UNKNOWN = 'unknown';
}
```

---

# 11. UNKNOWN

`UNKNOWN` nunca deberá convertirse silenciosamente en:

```text
ELIGIBLE
```

Para operaciones con efectos:

```text
UNKNOWN
→ conservative denial
```

será la regla predeterminada.

---

# 12. RetryDecision

```php
final readonly class RetryDecision
{
    public function __construct(
        public RetryDecisionType $decision,
        public RetryScope $scope,
        public RetryAttemptNumber $nextAttempt,
        public ?Duration $delay,
        public RetryDecisionReasonSet $reasons,
    ) {}
}
```

---

# 13. RetryDecisionType

```php
enum RetryDecisionType: string
{
    case DENY = 'deny';
    case RETRY_IMMEDIATELY = 'retry_immediately';
    case RETRY_AFTER_DELAY = 'retry_after_delay';
    case RESTART_TRANSACTION = 'restart_transaction';
    case RESTART_REGION = 'restart_region';
    case RESTART_EXECUTION = 'restart_execution';
}
```

---

# 14. Retry scope

Uno de los elementos más importantes es determinar:

```text
qué debe volver a ejecutarse
```

No todo error debe reintentar el mismo statement.

---

# 15. RetryScope

```php
enum RetryScope: string
{
    case NONE = 'none';
    case STATEMENT = 'statement';
    case TRANSACTION = 'transaction';
    case EXECUTION_REGION = 'execution_region';
    case EXECUTION_PLAN = 'execution_plan';
}
```

---

# 16. Statement retry

Sólo será válido cuando volver a ejecutar el statement aislado preserve la semántica.

Ejemplo potencial:

```text
SELECT
↓
temporary connection failure before dispatch
↓
new connection
↓
same SELECT
```

---

# 17. Transaction retry

Un deadlock o serialization failure puede requerir:

```text
rollback
↓
new transaction
↓
replay complete transaction body
```

No:

```text
retry only failed statement
```

---

# 18. Execution region retry

Un `ExecutionPlan` podrá declarar regiones replayable:

```text
ExecutionPlan
├── Region A
├── Region B [retryable]
└── Region C
```

Una región puede representar varias unidades que deben repetirse juntas.

---

# 19. Whole execution retry

Será la opción más amplia:

```text
restart entire ExecutionPlan
```

Sólo será válida si todos los efectos previos pueden reproducirse con seguridad.

---

# 20. Retry scope must be planned

El executor no deberá inventar regiones de retry a partir de observación runtime.

Preferiblemente:

```text
ExecutionPlan
→ RetryRegionDescriptor
```

---

# 21. RetryRegionDescriptor

```php
final readonly class RetryRegionDescriptor
{
    public function __construct(
        public RetryRegionId $id,
        public array $units,
        public RetryScope $scope,
        public ReplayContract $replay,
        public EffectContract $effects,
        public TransactionRequirement $transaction,
    ) {}
}
```

---

# 22. Retry boundary

```text
Retry Boundary
≠
Stage
≠
Transaction
≠
Thread
≠
Connection
```

Aunque en algunos casos puedan coincidir.

---

# 23. Idempotency

La seguridad de reintento depende fuertemente de la idempotencia.

---

# 24. IdempotencyClassification

```php
enum IdempotencyClassification: string
{
    case PROVEN_IDEMPOTENT = 'proven_idempotent';
    case CONDITIONALLY_IDEMPOTENT = 'conditionally_idempotent';
    case NON_IDEMPOTENT = 'non_idempotent';
    case TRANSACTIONALLY_REPLAYABLE = 'transactionally_replayable';
    case UNKNOWN = 'unknown';
}
```

---

# 25. Idempotency must not be guessed from HTTP concepts

La base de datos no deberá asumir:

```text
SELECT = always safe
INSERT = always unsafe
UPDATE = always unsafe
```

La semántica real importa.

---

# 26. SELECT

Un SELECT puro generalmente es replayable, pero existen excepciones:

```text
volatile functions
locking reads
sequences
stored functions with side effects
temporary/session state
vendor extensions
```

Por tanto:

```text
SELECT syntax
≠
proof of purity
```

---

# 27. INSERT

```sql
INSERT INTO users (...) VALUES (...)
```

puede duplicar efectos.

No deberá reintentarse después de outcome incierto.

---

# 28. Idempotent insert patterns

Un plan podría demostrar seguridad mediante mecanismos como:

```text
application idempotency key
deterministic primary key
conflict semantics
transaction replay
```

Pero la prueba deberá ser explícita.

---

# 29. UPDATE

Ejemplo:

```sql
UPDATE counters
SET value = value + 1
```

no es idempotente.

Ejecutarlo dos veces cambia el resultado.

---

# 30. Idempotent update

```sql
UPDATE users
SET status = 'active'
WHERE id = ?
```

puede ser idempotente respecto al estado final.

Pero aún deben considerarse:

```text
triggers
timestamps
audit tables
side effects
RETURNING semantics
```

---

# 31. DELETE

Un DELETE puede parecer idempotente respecto al estado final:

```text
row absent
```

pero puede disparar:

```text
triggers
cascades
audit
external side effects
```

No se asumirá automáticamente.

---

# 32. Stored procedures/functions

Por defecto:

```text
UNKNOWN
```

salvo metadata explícita.

---

# 33. Volatility metadata

El Query Semantic System puede aportar:

```text
IMMUTABLE
STABLE
VOLATILE
SIDE_EFFECTING
UNKNOWN
```

como hechos.

Retry los consume; no los inventa.

---

# 34. EffectContract

```php
final readonly class EffectContract
{
    public function __construct(
        public EffectClassification $classification,
        public IdempotencyClassification $idempotency,
        public bool $externalEffectsPossible,
        public bool $sessionEffectsPossible,
        public bool $transactional,
    ) {}
}
```

---

# 35. EffectClassification

```php
enum EffectClassification
{
    case READ_ONLY;
    case DATABASE_MUTATION;
    case SESSION_MUTATION;
    case EXTERNAL_SIDE_EFFECT;
    case MIXED;
    case UNKNOWN;
}
```

---

# 36. External effects

Si una stored procedure envía:

```text
message
email
webhook
external queue event
```

el retry puede duplicar efectos que una transacción DB no revierte.

Por defecto:

```text
external side effect
→ retry unsafe
```

salvo protocolo explícito de idempotencia.

---

# 37. Outcome certainty

El documento 85 definió:

```text
NOT_EXECUTED
EXECUTED_SUCCESSFULLY
EXECUTED_FAILED
PARTIAL_EFFECTS_POSSIBLE
UNKNOWN
```

Retry deberá consumir esta evidencia.

---

# 38. NOT_EXECUTED

Es el caso más favorable:

```text
failure before dispatch
```

Puede permitir retry incluso para una operación no idempotente, siempre que se pueda demostrar:

```text
operation never reached effect boundary
```

---

# 39. UNKNOWN outcome

Ejemplo:

```text
INSERT dispatched
↓
connection lost
↓
no response
```

No se sabe si el INSERT ocurrió.

Por defecto:

```text
UNKNOWN + non-idempotent mutation
→ DENY RETRY
```

---

# 40. Commit ambiguity

```text
COMMIT sent
↓
connection lost
```

produce:

```text
COMMIT_OUTCOME_UNKNOWN
```

Nunca deberá generar automáticamente:

```text
restart transaction
```

porque la primera transacción podría haberse confirmado.

---

# 41. Partial effects

```php
enum PartialEffectClassification
{
    case NONE;
    case POSSIBLE;
    case OBSERVED;
    case EXTERNALLY_VISIBLE;
    case UNKNOWN;
}
```

---

# 42. Partial delivery

Si un stream entregó:

```text
500 rows
```

volver a ejecutar desde cero puede duplicar datos para el consumidor.

---

# 43. Streaming retry

Por defecto:

```text
rowsDelivered > 0
→ transparent retry denied
```

---

# 44. Resume ≠ retry

Continuar un stream desde cierto punto sería:

```text
resume protocol
```

no un simple retry.

Debe diseñarse separadamente.

---

# 45. Cursor retry

Igualmente:

```text
cursor consumed partially
```

no permite volver a ejecutar transparentemente sin demostrar equivalencia.

---

# 46. Replayability

Todos los inputs necesarios deben poder reproducirse.

---

# 47. ReplayabilityClassification

```php
enum ReplayabilityClassification: string
{
    case REPLAYABLE = 'replayable';
    case REPLAYABLE_WITH_RESET = 'replayable_with_reset';
    case ONE_SHOT = 'one_shot';
    case CONSUMED = 'consumed';
    case UNKNOWN = 'unknown';
}
```

---

# 48. Parameter binding integration

El documento 80 establece que cada binding puede aportar replayability.

Ejemplo:

```text
integer
→ replayable
```

```text
string
→ replayable
```

```text
one-shot stream
→ ONE_SHOT
```

---

# 49. Stream parameter

```php
$database->execute(
    query: $insertBlob,
    bindings: [
        'content' => $nonSeekableStream,
    ],
);
```

Si el primer attempt consume el stream:

```text
retry
→ impossible
```

sin una fuente replayable.

---

# 50. ReplayContract

```php
final readonly class ReplayContract
{
    public function __construct(
        public ReplayabilityClassification $classification,
        public array $requirements,
        public array $resources,
    ) {}
}
```

---

# 51. No hidden buffering

VoltStack no deberá convertir automáticamente:

```text
500 MB one-shot stream
```

en:

```text
500 MB memory buffer
```

sólo para habilitar retry.

---

# 52. Explicit replay buffer

Una aplicación/policy podrá solicitar un mecanismo de replay buffer.

Debe ser:

```text
explicit
bounded
resource-governed
security-aware
```

---

# 53. Runtime values

Runtime bindings replayables podrán conservarse durante la retry region.

Pero sólo dentro de su scope.

---

# 54. Sensitive values

El retry history no deberá copiar valores sensibles a:

```text
logs
telemetry
diagnostics
retry descriptors
```

---

# 55. Connection recovery

Un retry puede requerir:

```text
same connection
reset connection
new connection
```

---

# 56. ConnectionRetryRequirement

```php
enum ConnectionRetryRequirement
{
    case SAME_CONNECTION;
    case RESET_CONNECTION;
    case NEW_CONNECTION;
    case ANY_HEALTHY_CONNECTION;
}
```

---

# 57. Lost connection

Si:

```text
ConnectionImpact = LOST
```

el retry no reutilizará esa conexión.

---

# 58. Reset-required connection

Si:

```text
ConnectionImpact = RESET_REQUIRED
```

deberá completarse reset antes de considerarla reusable.

---

# 59. Session affinity

Si el plan depende de:

```text
temporary tables
session variables
advisory locks
connection-local prepared state
```

una nueva conexión puede no ser semánticamente equivalente.

---

# 60. Connection replacement

Por tanto:

```text
ConnectionLost
≠
always retry on another connection
```

---

# 61. Transaction integration

Retry deberá distinguir:

```text
no transaction
engine-owned transaction
caller-owned transaction
nested transaction/savepoint
external transaction context
```

---

# 62. Engine-owned transaction

VoltStack puede reiniciar una transacción únicamente si el contrato permite:

```text
rollback old attempt
↓
new transaction
↓
replay transaction region
```

---

# 63. Caller-owned transaction

Si el usuario abrió la transacción:

```php
$database->transaction(function () {
    // ...
});
```

el retry deberá respetar la política definida por Transaction Manager.

Execution Retry no deberá apropiarse silenciosamente de ella.

---

# 64. Transaction body replay

Un transaction retry correcto suele ser:

```text
BEGIN
A
B
C → serialization failure
ROLLBACK

BEGIN
A
B
C
COMMIT
```

No:

```text
A
B
retry C
```

---

# 65. Transaction replayability

Toda la región deberá ser replayable.

---

# 66. Transaction-local application state

Problema:

```php
$balance = $service->calculate();
$db->insert(...);
$externalService->notify();
```

Repetir la closure completa podría repetir:

```text
externalService->notify()
```

Por ello los transaction retries de application closures requieren reglas explícitas.

---

# 67. Database-internal retry vs application closure retry

Deben distinguirse:

```text
ExecutionPlan retry
```

de:

```text
application callback retry
```

El segundo pertenece a una API superior del Transaction System.

---

# 68. Savepoints

Un savepoint no implica automáticamente que:

```text
failed statement can be retried safely
```

La plataforma y el tipo de error determinan si la transacción sigue siendo válida.

---

# 69. PostgreSQL-style aborted transaction

Si la plataforma marca la transacción como fallida:

```text
statement retry in same transaction
→ invalid
```

Debe realizarse el recovery adecuado.

---

# 70. Deadlock retry

Un deadlock normalmente requiere:

```text
rollback/restart transaction
```

según plataforma.

---

# 71. Serialization retry

Una serialization failure suele requerir:

```text
replay transaction
```

desde un boundary válido.

---

# 72. Lock timeout

Puede permitir diferentes estrategias dependiendo de:

```text
platform
transaction impact
operation semantics
policy
```

---

# 73. Connection acquisition failure

Ejemplo:

```text
pool temporarily exhausted
```

puede permitir retry sin que ninguna operación DB haya sido ejecutada.

---

# 74. Connection failure before dispatch

Si se demuestra:

```text
NOT_EXECUTED
```

un retry puede ser seguro aunque el statement sea mutation.

---

# 75. Connection failure after dispatch

Si:

```text
OutcomeCertainty = UNKNOWN
```

se necesita idempotencia fuerte o protocolo explícito para retry.

---

# 76. Preparation failure

Errores transitorios de prepare pueden ser retryable sólo si:

```text
cause is transient
+
connection can recover
+
deadline permits
```

Errores de SQL inválido no lo son.

---

# 77. Binding failure

Generalmente:

```text
conversion error
invalid type
missing parameter
```

no son transitorios.

Por defecto:

```text
DENY
```

---

# 78. Constraint violation

Errores como:

```text
unique violation
foreign key violation
not-null violation
```

normalmente no son transitorios.

No deberán repetirse ciegamente.

---

# 79. Resource exhaustion

Puede ser retryable dependiendo de la causa:

```text
pool exhausted temporarily
→ maybe
```

```text
query exceeds fixed memory policy
→ no
```

---

# 80. Timeout retry

Un timeout no significa automáticamente:

```text
retry with same timeout
```

---

# 81. Deadline vs attempt timeout

Debe distinguirse:

```text
Global Execution Deadline
```

de:

```text
Per-Attempt Timeout
```

---

# 82. Global deadline

Ejemplo:

```text
deadline = 5 seconds
attempt 1 = 3 seconds
backoff = 1 second
```

Quedan aproximadamente:

```text
1 second
```

No se reinicia a 5 segundos.

---

# 83. Deadline invariant

```text
Retry
```

no deberá extender silenciosamente el deadline original.

---

# 84. Cancellation

Una operación cancelada explícitamente:

```text
CancellationRequested = true
```

no deberá reintentarse automáticamente.

---

# 85. Cancellation invariant

```text
Cancelled
⇒
NoFurtherRetry
```

salvo una API superior explícitamente distinta.

---

# 86. Retry budget

Los retries deben estar limitados.

---

# 87. RetryBudget

```php
final class RetryBudget
{
    public function __construct(
        public readonly int $maxAttempts,
        public readonly Duration $maxTotalDelay,
        public readonly Duration $maxElapsedTime,
    ) {}
}
```

---

# 88. Attempt count

Convención recomendada:

```text
attempt 1 = initial execution
attempt 2 = first retry
attempt 3 = second retry
```

Por tanto:

```text
maxAttempts = 3
```

significa:

```text
1 initial + 2 retries
```

---

# 89. RetryBudgetState

```php
final readonly class RetryBudgetState
{
    public function __construct(
        public int $attemptsUsed,
        public Duration $delayUsed,
        public Duration $elapsed,
        public bool $exhausted,
    ) {}
}
```

---

# 90. Multiple budgets

Podrán existir:

```text
per-operation budget
per-transaction budget
per-request budget
system/global retry pressure budget
```

---

# 91. Retry storm protection

En una caída general del DB:

```text
10,000 workers
×
3 retries
```

pueden empeorar la situación.

---

# 92. Retry amplification

El sistema deberá considerar:

```text
retry amplification
```

como riesgo operacional.

---

# 93. Backoff

Políticas posibles:

```text
NONE
FIXED
LINEAR
EXPONENTIAL
SERVER_HINTED
```

---

# 94. BackoffPolicy

```php
interface BackoffPolicy
{
    public function delay(
        RetryAttemptContext $context
    ): Duration;
}
```

---

# 95. Exponential backoff

Modelo:

```text
delay(n)
=
min(
    maxDelay,
    baseDelay × multiplier^(n - 1)
)
```

---

# 96. Jitter

Los retries concurrentes no deberán sincronizarse.

---

# 97. Jitter strategies

Podrán soportarse:

```text
NONE
FULL
EQUAL
DECORRELATED
```

---

# 98. Randomness

El jitter puede usar aleatoriedad runtime.

Pero:

```text
retry semantics
```

no deberán depender de la aleatoriedad.

Sólo el timing.

---

# 99. Deterministic testing

El jitter deberá permitir:

```text
injectable random source
```

para pruebas reproducibles.

---

# 100. Server hints

Si el driver/server aporta:

```text
retry-after
backoff hint
```

podrá considerarse.

Pero no deberá superar:

```text
global deadline
policy maximum
```

---

# 101. Retry policy

```php
interface RetryPolicy
{
    public function decide(
        RetryEligibility $eligibility,
        RetryAttemptContext $attempt
    ): RetryDecision;
}
```

---

# 102. Default policy

VoltStack deberá ser conservador.

Propuesta:

```text
reads:
    safe transient failures → limited retry

writes:
    retry only with proven safety

transactions:
    retry only from explicit replayable transaction boundary

unknown outcomes:
    deny unless explicit idempotency protocol proves safety

cancelled:
    deny

partial result delivered:
    deny transparent retry
```

---

# 103. RetryProfile

Puede existir:

```php
enum RetryProfile
{
    case DISABLED;
    case CONSERVATIVE;
    case STANDARD;
    case RESILIENT;
    case CUSTOM;
}
```

---

# 104. Profile semantics

Un perfil más agresivo:

```text
may increase retries
```

pero jamás:

```text
weaken correctness requirements
```

---

# 105. Retry policy ≠ correctness gate

La arquitectura deberá separar:

```text
RetrySafetyGate
```

de:

```text
RetryPolicy
```

La policy no podrá autorizar algo declarado semánticamente inseguro.

---

# 106. Safety precedence

```text
Correctness
>
Policy Preference
>
Performance
```

---

# 107. RetrySafety

```php
enum RetrySafety
{
    case PROVEN_SAFE;
    case SAFE_IF_REQUIREMENTS_MET;
    case UNSAFE;
    case UNKNOWN;
}
```

---

# 108. Retry reason model

Las decisiones deberán ser explicables.

---

# 109. RetryReason

```php
enum RetryReason: string
{
    case TRANSIENT_CONNECTION_FAILURE = 'transient_connection_failure';
    case DEADLOCK = 'deadlock';
    case SERIALIZATION_FAILURE = 'serialization_failure';
    case LOCK_TIMEOUT = 'lock_timeout';
    case RESOURCE_TEMPORARILY_UNAVAILABLE = 'resource_temporarily_unavailable';

    case NON_IDEMPOTENT = 'non_idempotent';
    case UNKNOWN_OUTCOME = 'unknown_outcome';
    case PARTIAL_EFFECTS = 'partial_effects';
    case INPUT_NOT_REPLAYABLE = 'input_not_replayable';
    case RESULT_ALREADY_DELIVERED = 'result_already_delivered';
    case TRANSACTION_NOT_REPLAYABLE = 'transaction_not_replayable';
    case DEADLINE_EXHAUSTED = 'deadline_exhausted';
    case RETRY_BUDGET_EXHAUSTED = 'retry_budget_exhausted';
    case CANCELLED = 'cancelled';
    case POLICY_DENIED = 'policy_denied';
}
```

---

# 110. Multiple reasons

Una decisión DENY puede contener:

```text
UNKNOWN_OUTCOME
+
NON_IDEMPOTENT
+
INPUT_NOT_REPLAYABLE
```

---

# 111. Retry attempt

Cada intento deberá ser first-class.

---

# 112. RetryAttempt

```php
final readonly class RetryAttempt
{
    public function __construct(
        public RetryAttemptId $id,
        public int $number,
        public Instant $startedAt,
        public RetryScope $scope,
        public RetryAttemptTrigger $trigger,
    ) {}
}
```

---

# 113. ExecutionId vs AttemptId

```text
ExecutionId
≠
RetryAttemptId
```

Una ejecución lógica puede contener múltiples attempts.

---

# 114. Correlation

```text
ExecutionId
    ├── Attempt 1
    ├── Attempt 2
    └── Attempt 3
```

---

# 115. New statement handles

Un nuevo attempt no deberá reutilizar un statement invalidado.

---

# 116. Resource reset

Antes de retry:

```text
cancel old work
↓
close cursor
↓
close/return statement
↓
rollback if required
↓
invalidate/reset connection
↓
release temporary resources
↓
reset execution-local outputs
↓
prepare new attempt
```

según scope.

---

# 117. RetryRecoveryCoordinator

```php
interface RetryRecoveryCoordinator
{
    public function recover(
        RetryDecision $decision,
        ExecutionInstance $execution
    ): RetryRecoveryResult;
}
```

---

# 118. Recovery must complete before retry

```text
OldAttempt resources
```

no deberán mezclarse con:

```text
NewAttempt resources
```

salvo recursos explícitamente declarados reusable.

---

# 119. Resource conservation

Para cada attempt:

```text
AcquiredResources
=
ReleasedResources
∪
TransferredReusableResources
```

---

# 120. Statement state

Un statement que falló parcialmente durante binding/execution deberá:

```text
reset
or
invalidate
```

antes de cualquier reuse.

---

# 121. Cursor state

Un cursor activo del attempt anterior no puede sobrevivir a un retry de su statement.

---

# 122. Materialization state

Materializaciones intermedias deben declarar si:

```text
reusable across retry
```

o:

```text
must be discarded
```

---

# 123. Correlation state

Los correlation frames del attempt anterior deben resetearse.

---

# 124. Output slots

Los output slots producidos por unidades dentro del retry scope deberán invalidarse.

---

# 125. Retry and QueryExecutor

`RetryCoordinator` deberá colaborar con:

```text
QueryExecutor
```

para volver a ejecutar el scope correcto.

No deberá llamar arbitrariamente a:

```text
StatementExecutor
```

ignorando dependencias del plan.

---

# 126. Retry and scheduler

El scheduler deberá recibir:

```text
retry region
+
new attempt state
```

y reconstruir únicamente el runtime state necesario.

---

# 127. No ExecutionPlan mutation

```text
Retry
```

crea nuevos runtime attempts.

No modifica:

```text
ExecutionPlan
```

---

# 128. Retry graph

Podrá representarse:

```text
Execution
│
├── Attempt 1
│   └── Failure
│
├── Recovery
│
└── Attempt 2
    └── Success
```

---

# 129. Nested retries

Deben evitarse retry loops independientes:

```text
ConnectionManager retries 3×
StatementExecutor retries 3×
Transaction retries 3×
Application retries 3×
```

porque producirían:

```text
3 × 3 × 3 × 3 = 81 attempts
```

---

# 130. Single retry authority

Dentro del Database Execution Engine deberá existir una autoridad coordinadora:

```text
RetryCoordinator
```

---

# 131. Lower-level retries

Drivers/connections pueden realizar internamente ciertas operaciones técnicas, pero deberán declararse y no cambiar la semántica observable.

---

# 132. Hidden driver retries

Si un driver realiza retry internamente y puede duplicar writes:

```text
unsafe
```

VoltStack deberá deshabilitarlo cuando sea posible o modelar explícitamente la capacidad.

---

# 133. Retry capabilities

```php
final readonly class DriverRetryCapabilities
{
    public function __construct(
        public bool $transparentReconnect,
        public bool $statementReplay,
        public bool $transactionReplay,
        public bool $serverRetryHints,
    ) {}
}
```

---

# 134. Transparent reconnect

Reconectar automáticamente puede ser peligroso si existía:

```text
transaction
temporary table
session variable
lock
connection-local state
```

---

# 135. Retry and prepared statements

Una nueva conexión implica:

```text
old PreparedStatementLease invalid
```

aunque el:

```text
PreparedStatementBlueprint
```

siga siendo reusable.

---

# 136. Correct reuse

```text
PreparedStatementBlueprint
        ↓
new connection
        ↓
new live PreparedStatementLease
```

---

# 137. Retry and compiled query cache

Los compiled artifacts pueden reutilizarse si siguen siendo compatibles.

No necesitan recompilarse sólo porque hubo un retry.

---

# 138. But capability change

Si failover lleva a un target con capacidades incompatibles:

```text
cached compiled artifact
```

deberá validarse nuevamente.

---

# 139. Failover boundary

```text
retry
```

y:

```text
failover
```

son conceptos diferentes.

Un retry puede utilizar failover, pero no son equivalentes.

---

# 140. Replica considerations

Un retry de lectura sobre otra réplica puede observar un estado diferente.

Esto deberá ser considerado por los futuros sistemas de:

```text
Read/Write Routing
Replica Lag
Sticky Connections
```

---

# 141. Consistency requirement

El plan podrá declarar:

```text
read consistency requirement
```

que limita los destinos válidos para retry.

---

# 142. Retry after write

Un read retry después de una escritura puede requerir:

```text
primary
sticky replica
session consistency
```

No se decidirá únicamente por disponibilidad.

---

# 143. Security

Retry no deberá:

```text
duplicate credentials in diagnostics
log bindings
weaken tenant context
switch tenant
bypass security predicates
change authorization context
```

---

# 144. Tenant context

El mismo retry lógico deberá conservar el mismo:

```text
TenantExecutionContext
```

salvo una transición explícitamente diseñada por otro sistema.

---

# 145. Security predicates

Nunca:

```text
retry simpler query without security filter
```

como fallback.

---

# 146. Raw SQL

Raw SQL tendrá:

```text
idempotency = UNKNOWN
```

por defecto, salvo metadata explícita.

---

# 147. Retry hints from user

Podrá existir API:

```php
$query->retryPolicy(...);
```

pero:

```text
user says retry
```

no podrá violar invariantes de seguridad/correctness.

---

# 148. User-declared idempotency

Una API avanzada podría permitir:

```php
->idempotency(Idempotency::Guaranteed)
```

pero deberá considerarse una afirmación explícita del desarrollador, no una prueba generada por VoltStack.

---

# 149. Trust boundary

Debe distinguirse:

```text
PROVEN_BY_FRAMEWORK
DECLARED_BY_APPLICATION
DECLARED_BY_EXTENSION
UNKNOWN
```

---

# 150. IdempotencyEvidence

```php
enum IdempotencyEvidenceSource
{
    case SEMANTIC_ANALYSIS;
    case EXECUTION_PLAN;
    case APPLICATION_DECLARATION;
    case EXTENSION_DECLARATION;
    case PLATFORM_CONTRACT;
    case UNKNOWN;
}
```

---

# 151. Retry policy configuration

Ejemplo conceptual:

```php
return [
    'database' => [
        'retry' => [
            'profile' => 'standard',
            'max_attempts' => 3,
            'base_delay_ms' => 50,
            'max_delay_ms' => 1000,
            'jitter' => 'full',
        ],
    ],
];
```

---

# 152. Configuration ≠ semantics

Config podrá ajustar:

```text
attempts
delays
profiles
```

pero no convertir:

```text
unsafe
```

en:

```text
safe
```

---

# 153. Retry extension system

Extensiones podrán aportar:

```text
retry policies
failure eligibility evaluators
backoff strategies
idempotency evidence providers
replayability providers
```

---

# 154. Extension restrictions

No podrán:

```text
mutate SQL
change query meaning
ignore partial effects
declare unknown outcome safe without evidence
access unrelated tenant state
execute hidden queries
```

---

# 155. RetryExtension

```php
interface ExecutionRetryExtension
{
    public function descriptor(): RetryExtensionDescriptor;

    public function eligibilityProviders(): iterable;

    public function policies(): iterable;

    public function backoffStrategies(): iterable;
}
```

---

# 156. Registry

```text
discover
↓
validate
↓
dependency resolution
↓
conflict detection
↓
deterministic ordering
↓
freeze
```

---

# 157. No last-wins

Dos policies con el mismo ID:

```text
bootstrap failure
```

No:

```text
last registered wins
```

---

# 158. Persistent runtime safety

Servicios compartibles:

```text
immutable policies
frozen registries
stateless eligibility analyzers
immutable configuration
```

---

# 159. Operation-scoped retry state

```text
RetryBudgetState
RetryHistory
AttemptState
RuntimeBindings
ReplayBuffers
TransactionState
ConnectionLease
StatementLease
CursorState
Deadline
CancellationToken
TenantContext
```

---

# 160. No static retry counter

Nunca:

```php
static $retryCount;
```

---

# 161. Request isolation

En FrankenPHP:

```text
Request A retry budget
∩
Request B retry budget
=
∅
```

---

# 162. Coroutine isolation

En OpenSwoole:

```text
Coroutine A RetryContext
≠
Coroutine B RetryContext
```

---

# 163. RetryContext

```php
final class RetryContext
{
    public function __construct(
        public readonly ExecutionId $executionId,
        public readonly RetryPolicyId $policy,
        public readonly RetryBudget $budget,
        public readonly Deadline $deadline,
        public readonly CancellationToken $cancellation,
        public RetryHistory $history,
    ) {}
}
```

---

# 164. RetryHistory

```php
final class RetryHistory
{
    /**
     * @var list<RetryAttemptRecord>
     */
    private array $attempts = [];
}
```

---

# 165. RetryAttemptRecord

Debe contener metadata, no datos sensibles:

```text
attempt number
start/end time
failure category
decision
delay
scope
connection replacement
transaction restart
outcome
```

---

# 166. Telemetry

Eventos sugeridos:

```text
DatabaseRetryEvaluated
DatabaseRetryAllowed
DatabaseRetryDenied
DatabaseRetryScheduled
DatabaseRetryAttemptStarted
DatabaseRetryAttemptSucceeded
DatabaseRetryAttemptFailed
DatabaseRetryBudgetExhausted
DatabaseTransactionRetryStarted
DatabaseRetryRecoveryFailed
```

---

# 167. Metrics

Ejemplos:

```text
db.retry.evaluations
db.retry.allowed
db.retry.denied
db.retry.attempts
db.retry.successes
db.retry.exhausted
db.retry.transaction_restarts
db.retry.recovery_failures
```

---

# 168. Metric labels

Cardinalidad controlada:

```text
failure_category
retry_scope
decision
platform
driver
attempt_bucket
```

---

# 169. Forbidden labels

No:

```text
SQL
parameter value
tenant ID
ExecutionId
transaction ID
email
user ID
native message
```

---

# 170. Retry telemetry semantics

Debe diferenciar:

```text
initial success
```

de:

```text
success after retry
```

---

# 171. Retry latency

La latencia total debe incluir:

```text
attempt execution
+
recovery
+
backoff
```

---

# 172. Query telemetry

Podrá reportar:

```text
attempt_count
retry_delay_total
recovery_time
```

sin convertir cada attempt necesariamente en una métrica high-cardinality.

---

# 173. Tracing

Conceptualmente:

```text
Database Execution Span
├── Attempt 1
│   └── Deadlock
├── Retry Backoff
└── Attempt 2
    └── Success
```

---

# 174. Retry success does not erase failure

Si el segundo intento funciona:

```text
overall outcome = success
```

pero telemetry deberá conservar que existió:

```text
attempt 1 failure
```

---

# 175. Diagnostics

Mensaje útil:

```text
Database operation succeeded after 2 attempts.
Initial failure: serialization failure.
Transaction was restarted.
```

---

# 176. Error after budget exhaustion

Si se agota:

```text
RetryBudget
```

el error final deberá preservar:

```text
last failure
+
retry history
+
budget exhaustion reason
```

---

# 177. RetryExhaustedException

Puede existir:

```php
final class RetryExhaustedException extends DatabaseExecutionException
{
    public function __construct(
        public readonly RetryHistory $history,
        ExecutionFailure $lastFailure,
    ) {
        // ...
    }
}
```

---

# 178. Recovery failure

Si el retry era válido pero falla:

```text
connection reset
rollback
resource cleanup
```

se deberá detener el retry.

---

# 179. Recovery failure ≠ original failure

Ambos deberán preservarse.

Ejemplo:

```text
Primary failure:
Deadlock

Recovery failure:
Rollback failed
```

---

# 180. Retry failure hierarchy

Propuesta:

```text
ExecutionRetryException
├── RetryEligibilityException
├── RetrySafetyException
├── RetryRecoveryException
├── RetryBudgetExceededException
├── RetryReplayException
├── RetryTransactionRestartException
├── RetryResourceResetException
├── RetryPolicyException
├── RetryExtensionException
└── RetryInvariantException
```

---

# 181. Retry decision must be deterministic

Dados:

```text
same failure evidence
same operation semantics
same runtime state snapshot
same retry policy
same budget state
```

la decisión:

```text
ALLOW / DENY / scope
```

deberá ser determinista.

El jitter sólo modifica:

```text
delay timing
```

---

# 182. Time source

Retry deberá utilizar un:

```text
MonotonicClock
```

para:

```text
elapsed time
deadline accounting
backoff budget
```

cuando sea posible.

---

# 183. Wall clock

No deberá usarse para medir elapsed time si puede sufrir:

```text
NTP adjustment
DST
manual clock change
```

---

# 184. Sleep abstraction

No:

```php
usleep(...)
```

disperso por el sistema.

Utilizar:

```php
interface RetryDelayScheduler
{
    public function wait(
        Duration $delay,
        CancellationToken $cancellation,
        Deadline $deadline
    ): void;
}
```

---

# 185. Runtime neutrality

Esto permite adaptadores para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

sin acoplar el Retry System a un scheduler concreto.

---

# 186. Non-blocking runtimes

OpenSwoole u otros runtimes pueden proporcionar espera cooperativa.

La política de retry no deberá conocer esos detalles.

---

# 187. Cancellation during backoff

Si se cancela durante:

```text
backoff
```

el retry deberá abortarse inmediatamente cuando el runtime lo permita.

---

# 188. Deadline during backoff

Si el deadline vence durante el delay:

```text
no next attempt
```

---

# 189. Minimum useful attempt time

Opcionalmente una policy podrá requerir:

```text
remaining deadline >= minimumAttemptBudget
```

antes de iniciar otro attempt.

---

# 190. Retry fairness

Un execution que reintenta repetidamente no deberá monopolizar:

```text
connection pool
worker
memory
scheduler
```

---

# 191. Release during backoff

Cuando sea seguro:

```text
do not hold scarce connection during retry sleep
```

---

# 192. But affinity matters

Si liberar la conexión destruye semántica requerida:

```text
retry may be impossible
```

o requerir otra estrategia.

---

# 193. Retry with locks

No deberá retener locks innecesariamente durante backoff.

---

# 194. Retry with transaction

El transaction retry normalmente deberá:

```text
rollback
release/reset state
backoff
begin new transaction
```

según política.

---

# 195. Retry with savepoints

La policy deberá respetar las capacidades reales de la plataforma.

---

# 196. Batch execution

Un batch puede tener:

```text
operations 1..N
```

Si falla la operación 5:

```text
retry entire batch?
retry item 5?
retry remaining?
```

depende del plan y del transaction contract.

No deberá adivinarse.

---

# 197. Batch progress

Debe existir metadata:

```text
completed operations
failed operation
committed effects
unknown effects
```

---

# 198. Bulk operations

Bulk INSERT/UPDATE/DELETE tendrán las mismas reglas:

```text
partial effects
idempotency
transaction scope
```

---

# 199. Multi-statement operations

Como regla general:

```text
hidden multi-statement
= forbidden
```

Cuando sean explícitas, su retry scope deberá estar modelado.

---

# 200. Multiple result sets

Si una operación produce varios result sets y algunos ya fueron consumidos:

```text
transparent replay
```

normalmente será inseguro.

---

# 201. Result consumers

El Retry System no deberá asumir que puede retirar valores ya observados por el consumidor.

---

# 202. Observable semantics

Una vez que un valor salió del boundary de execution:

```text
externally observable
```

debe considerarse para retry safety.

---

# 203. Observable effect formula

```text
ObservableEffects
=
DatabaseMutations
+
DeliveredResults
+
SessionStateChanges
+
ExternalSideEffects
```

---

# 204. Retry correctness

```text
Retry is safe
```

sólo si la secuencia observable después del retry es compatible con una ejecución válida del plan original.

---

# 205. Exactly-once misconception

El Retry System no deberá prometer:

```text
exactly once
```

en presencia de:

```text
network ambiguity
external side effects
unknown commit outcome
```

sin protocolo adicional.

---

# 206. At-least-once risk

Un retry de write con outcome desconocido puede convertir la operación en:

```text
at least once
```

y duplicar efectos.

---

# 207. At-most-once strategy

Negar retry después de outcome desconocido favorece:

```text
at most one dispatch
```

pero puede dejar el resultado desconocido.

Esto deberá ser explícito.

---

# 208. Idempotency keys

Para ciertos sistemas superiores, una clave de idempotencia puede convertir:

```text
retry after ambiguous dispatch
```

en una operación recuperable.

Pero requiere soporte explícito del modelo de datos/aplicación.

---

# 209. Retry tokens

Podrá existir conceptualmente:

```php
final readonly class IdempotencyToken
{
    public function __construct(
        public string $value,
    ) {}
}
```

No deberá generarse automáticamente si cambia la semántica de negocio.

---

# 210. Idempotency token security

No debe aparecer como metric label si contiene identidad sensible.

---

# 211. Suggested namespace

```text
VoltStack\Quantum\Database\Execution\Retry
```

---

# 212. Directory structure

```text
VoltStack/
└── Quantum/
    └── Database/
        └── Execution/
            └── Retry/
                ├── Contract/
                │   ├── RetryPolicy.php
                │   ├── BackoffPolicy.php
                │   ├── RetryEligibilityProvider.php
                │   ├── RetryDelayScheduler.php
                │   └── ExecutionRetryExtension.php
                │
                ├── Coordinator/
                │   ├── RetryCoordinator.php
                │   ├── DefaultRetryCoordinator.php
                │   └── RetryRecoveryCoordinator.php
                │
                ├── Eligibility/
                │   ├── RetryEligibility.php
                │   ├── RetryEligibilityStatus.php
                │   ├── RetryEligibilityAnalyzer.php
                │   └── DefaultRetryEligibilityAnalyzer.php
                │
                ├── Safety/
                │   ├── RetrySafety.php
                │   ├── RetrySafetyGate.php
                │   ├── IdempotencyClassification.php
                │   ├── IdempotencyEvidence.php
                │   ├── IdempotencyEvidenceSource.php
                │   └── PartialEffectClassification.php
                │
                ├── Scope/
                │   ├── RetryScope.php
                │   ├── RetryScopeResolver.php
                │   ├── RetryRegionId.php
                │   └── RetryRegionDescriptor.php
                │
                ├── Replay/
                │   ├── ReplayContract.php
                │   ├── ReplayabilityClassification.php
                │   ├── ReplayabilityAnalyzer.php
                │   └── ReplayBuffer.php
                │
                ├── Effect/
                │   ├── EffectContract.php
                │   └── EffectClassification.php
                │
                ├── Decision/
                │   ├── RetryDecision.php
                │   ├── RetryDecisionType.php
                │   ├── RetryReason.php
                │   └── RetryDecisionReasonSet.php
                │
                ├── Policy/
                │   ├── DefaultRetryPolicy.php
                │   ├── RetryProfile.php
                │   └── RetryPolicyRegistry.php
                │
                ├── Budget/
                │   ├── RetryBudget.php
                │   ├── RetryBudgetState.php
                │   └── RetryBudgetManager.php
                │
                ├── Backoff/
                │   ├── FixedBackoffPolicy.php
                │   ├── LinearBackoffPolicy.php
                │   ├── ExponentialBackoffPolicy.php
                │   ├── ServerHintedBackoffPolicy.php
                │   ├── JitterStrategy.php
                │   └── RetryRandomSource.php
                │
                ├── Attempt/
                │   ├── RetryAttempt.php
                │   ├── RetryAttemptId.php
                │   ├── RetryAttemptContext.php
                │   ├── RetryAttemptRecord.php
                │   └── RetryHistory.php
                │
                ├── Recovery/
                │   ├── RetryRecoveryResult.php
                │   ├── ConnectionRetryRequirement.php
                │   ├── ConnectionRecovery.php
                │   ├── StatementRecovery.php
                │   ├── TransactionRecovery.php
                │   └── ResourceRecovery.php
                │
                ├── Runtime/
                │   └── RetryContext.php
                │
                ├── Extension/
                │   ├── RetryExtensionDescriptor.php
                │   └── RetryExtensionRegistry.php
                │
                ├── Telemetry/
                │   ├── RetryTelemetry.php
                │   ├── RetryMetrics.php
                │   └── RetryObserver.php
                │
                └── Exception/
                    ├── ExecutionRetryException.php
                    ├── RetryEligibilityException.php
                    ├── RetrySafetyException.php
                    ├── RetryRecoveryException.php
                    ├── RetryBudgetExceededException.php
                    ├── RetryReplayException.php
                    ├── RetryTransactionRestartException.php
                    ├── RetryResourceResetException.php
                    ├── RetryPolicyException.php
                    ├── RetryExtensionException.php
                    └── RetryInvariantException.php
```

---

# 213. Dependency model

```text
Execution Error System
        ↓
Execution Retry System
        ↓
Query Executor
        ↓
Statement Execution
```

También consume:

```text
ExecutionPlan
Parameter Replayability
Transaction Context
Connection Impact
Deadline
Cancellation
Resource Governance
```

---

# 214. Forbidden dependencies

No:

```text
Retry System
→ ORM EntityManager
→ Repository
→ Controller
→ HTTP Request global
→ global current tenant
```

---

# 215. Testing architecture

La suite deberá incluir:

```text
eligibility tests
idempotency tests
scope tests
transaction replay tests
connection recovery tests
stream replay tests
deadline tests
cancellation tests
budget tests
backoff tests
jitter tests
persistent-runtime tests
driver conformance tests
failure-injection tests
```

---

# 216. Essential scenarios

Debe probarse al menos:

```text
connection acquisition transient failure
connection loss before dispatch
connection loss after dispatch
deadlock
serialization failure
lock timeout
prepare transient failure
binding deterministic failure
constraint violation
timeout
explicit cancellation
unknown commit outcome
one-shot stream binding
partial result delivery
cursor partial consumption
retry budget exhaustion
recovery failure
transaction restart
connection replacement
persistent worker isolation
```

---

# 217. Deadlock test

```text
Attempt 1
→ deadlock
→ transaction rollback

Attempt 2
→ replay transaction
→ success
```

Verificar:

```text
no statement-only retry
```

---

# 218. Serialization test

Verificar que:

```text
whole retry region
```

se repite desde el boundary correcto.

---

# 219. Unknown write test

```text
INSERT dispatched
connection lost
outcome unknown
```

Esperado:

```text
DENY
```

si no existe idempotency evidence.

---

# 220. Idempotency-key test

Mismo escenario, pero con protocolo de idempotencia explícito:

```text
may become eligible
```

según policy.

---

# 221. Stream input test

```text
ONE_SHOT stream consumed
```

Esperado:

```text
INPUT_NOT_REPLAYABLE
→ DENY
```

---

# 222. Partial output test

```text
100 rows delivered
connection lost
```

Esperado:

```text
RESULT_ALREADY_DELIVERED
→ transparent retry DENY
```

---

# 223. Deadline test

```text
attempt 1 consumes most deadline
```

Retry no reinicia el reloj.

---

# 224. Cancellation during backoff test

```text
backoff
↓
cancel
```

Esperado:

```text
no attempt 2
```

---

# 225. Resource cleanup test

Verificar:

```text
old statement
old cursor
old connection lease
```

no contaminan el nuevo attempt.

---

# 226. Persistent worker test

```text
Request A
→ retry count 2

Request B
→ retry count 0
```

---

# 227. Concurrency test

Dos executions concurrentes deben mantener:

```text
separate RetryContext
separate RetryBudget
separate RetryHistory
```

---

# 228. Deterministic decision test

La decisión deberá ser idéntica con el mismo snapshot.

---

# 229. Driver conformance

Drivers oficiales deberán declarar comportamiento sobre:

```text
deadlocks
serialization
lock timeout
connection loss
transaction abort state
reconnect semantics
statement invalidation
```

---

# 230. Architectural invariants

## DB-RETRY-001

Retryable failure no implicará safe retry.

## DB-RETRY-002

Safe retry requerirá eligibility analysis.

## DB-RETRY-003

Retry decision será distinta de error classification.

## DB-RETRY-004

Retry scope será explícito.

## DB-RETRY-005

Statement retry será distinto de transaction retry.

## DB-RETRY-006

Transaction retry será distinto de execution-plan retry.

## DB-RETRY-007

Retry region será distinta de stage.

## DB-RETRY-008

Retry region será distinta de connection.

## DB-RETRY-009

Retry region será distinta de transaction aunque puedan coincidir.

## DB-RETRY-010

Executor no inventará retry regions.

## DB-RETRY-011

ExecutionPlan podrá declarar retry boundaries.

## DB-RETRY-012

Idempotency será explícita.

## DB-RETRY-013

UNKNOWN idempotency será conservadora.

## DB-RETRY-014

SELECT no implicará automáticamente idempotency.

## DB-RETRY-015

INSERT no será reintentado ciegamente.

## DB-RETRY-016

UPDATE no será considerado idempotente sólo por ser UPDATE.

## DB-RETRY-017

DELETE no será considerado idempotente automáticamente.

## DB-RETRY-018

Stored procedures serán UNKNOWN por defecto.

## DB-RETRY-019

External side effects deberán considerarse.

## DB-RETRY-020

Outcome certainty será considerada antes de retry.

## DB-RETRY-021

NOT_EXECUTED podrá habilitar retries que UNKNOWN no permite.

## DB-RETRY-022

UNKNOWN write outcome será conservador.

## DB-RETRY-023

Commit outcome unknown no será reintentado ciegamente.

## DB-RETRY-024

Partial effects serán considerados.

## DB-RETRY-025

Partial result delivery impedirá transparent replay por defecto.

## DB-RETRY-026

Resume será distinto de retry.

## DB-RETRY-027

Todos los inputs del retry scope deberán ser replayable.

## DB-RETRY-028

One-shot consumed input impedirá retry.

## DB-RETRY-029

Retry System no realizará hidden buffering.

## DB-RETRY-030

Replay buffers serán explícitos y bounded.

## DB-RETRY-031

Sensitive bindings no se copiarán a telemetry.

## DB-RETRY-032

Lost connection no será reutilizada.

## DB-RETRY-033

Reset-required connection deberá resetearse antes de reuse.

## DB-RETRY-034

Connection replacement deberá respetar session affinity.

## DB-RETRY-035

Retry no inventará transaction boundaries.

## DB-RETRY-036

Transaction retry repetirá el scope transaccional requerido.

## DB-RETRY-037

Deadlock no implicará statement-only retry.

## DB-RETRY-038

Serialization failure podrá requerir transaction restart.

## DB-RETRY-039

Caller-owned transaction no será apropiada silenciosamente.

## DB-RETRY-040

Application callback retry será distinto de ExecutionPlan retry.

## DB-RETRY-041

Savepoint no implicará retry safety.

## DB-RETRY-042

Platform transaction-abort semantics serán respetadas.

## DB-RETRY-043

Connection acquisition failure podrá ser retryable sin DB effects.

## DB-RETRY-044

Connection loss before dispatch será distinta de after dispatch.

## DB-RETRY-045

Prepare syntax errors no serán tratados como transient.

## DB-RETRY-046

Binding validation failures no serán reintentados por defecto.

## DB-RETRY-047

Constraint violations no serán reintentadas ciegamente.

## DB-RETRY-048

Resource exhaustion será clasificada por causa.

## DB-RETRY-049

Timeout no implicará retry automático.

## DB-RETRY-050

Global deadline no se reiniciará por retry.

## DB-RETRY-051

Cancellation detendrá futuros retries.

## DB-RETRY-052

Retry tendrá budget.

## DB-RETRY-053

Attempt 1 será la ejecución inicial.

## DB-RETRY-054

maxAttempts incluirá initial attempt.

## DB-RETRY-055

Retry budget podrá limitar tiempo y attempts.

## DB-RETRY-056

Retry storm será considerado.

## DB-RETRY-057

Backoff será configurable.

## DB-RETRY-058

Jitter no cambiará semántica.

## DB-RETRY-059

Random source será injectable para testing.

## DB-RETRY-060

Server backoff hint no podrá superar deadline.

## DB-RETRY-061

Retry policy no podrá violar safety gate.

## DB-RETRY-062

Correctness tendrá prioridad sobre policy preference.

## DB-RETRY-063

Retry decisions serán explicables.

## DB-RETRY-064

ExecutionId será distinto de RetryAttemptId.

## DB-RETRY-065

Cada attempt tendrá runtime state separado.

## DB-RETRY-066

Invalid statement no será reutilizado.

## DB-RETRY-067

Old cursor no sobrevivirá a statement retry.

## DB-RETRY-068

Retry invalidará outputs del retry scope.

## DB-RETRY-069

Correlation state será reset por scope.

## DB-RETRY-070

QueryExecutor coordinará replay del plan.

## DB-RETRY-071

Retry no mutará ExecutionPlan.

## DB-RETRY-072

Old-attempt resources deberán liberarse o transferirse explícitamente.

## DB-RETRY-073

Nested independent retry loops serán evitados.

## DB-RETRY-074

RetryCoordinator será autoridad central del execution retry.

## DB-RETRY-075

Hidden driver retries que cambien semantics serán prohibidos.

## DB-RETRY-076

Transparent reconnect será capability-aware.

## DB-RETRY-077

New connection requerirá new live prepared statement.

## DB-RETRY-078

PreparedStatementBlueprint podrá reutilizarse si compatible.

## DB-RETRY-079

Compiled artifact podrá reutilizarse si sigue compatible.

## DB-RETRY-080

Failover será distinto de retry.

## DB-RETRY-081

Retry destination respetará consistency requirements.

## DB-RETRY-082

Retry no cambiará tenant context.

## DB-RETRY-083

Retry no eliminará security predicates.

## DB-RETRY-084

Raw SQL tendrá UNKNOWN idempotency por defecto.

## DB-RETRY-085

Application-declared idempotency será distinguible de framework-proven idempotency.

## DB-RETRY-086

Configuration no convertirá unsafe en safe.

## DB-RETRY-087

Retry extensions serán typed.

## DB-RETRY-088

Retry registry será frozen.

## DB-RETRY-089

No habrá last-wins policy registration.

## DB-RETRY-090

Persistent runtime no compartirá RetryContext mutable.

## DB-RETRY-091

No existirá static retry counter.

## DB-RETRY-092

Concurrent executions tendrán budgets separados.

## DB-RETRY-093

Retry history será operation-scoped.

## DB-RETRY-094

Retry telemetry no contendrá binding values.

## DB-RETRY-095

Retry success no borrará telemetry de failed attempts.

## DB-RETRY-096

Budget exhaustion preservará last failure.

## DB-RETRY-097

Recovery failure preservará original failure.

## DB-RETRY-098

Retry decision será determinista dado el mismo state snapshot.

## DB-RETRY-099

Jitter podrá ser no determinista sin afectar decision semantics.

## DB-RETRY-100

Elapsed time usará monotonic clock cuando sea posible.

## DB-RETRY-101

Backoff será cancellation-aware.

## DB-RETRY-102

Backoff será deadline-aware.

## DB-RETRY-103

Retry no iniciará un attempt sin tiempo útil según policy.

## DB-RETRY-104

Retries deberán respetar resource fairness.

## DB-RETRY-105

Scarce resources deberán liberarse durante backoff cuando sea seguro.

## DB-RETRY-106

Retry no mantendrá locks innecesarios durante delay.

## DB-RETRY-107

Batch retry scope será explícito.

## DB-RETRY-108

Bulk operation partial effects serán explícitos.

## DB-RETRY-109

Hidden multi-statement retry será prohibido.

## DB-RETRY-110

Consumed result sets impedirán transparent replay por defecto.

## DB-RETRY-111

Observable delivered values contarán como effects.

## DB-RETRY-112

Database mutation será sólo una categoría de observable effect.

## DB-RETRY-113

Execution Retry no prometerá exactly-once sin protocolo que lo garantice.

## DB-RETRY-114

Unknown write retry reconocerá riesgo at-least-once.

## DB-RETRY-115

Retry denial podrá preservar at-most-dispatch semantics.

## DB-RETRY-116

Idempotency keys serán explícitas.

## DB-RETRY-117

Idempotency tokens no serán generados si alteran business semantics.

## DB-RETRY-118

Idempotency evidence tendrá provenance.

## DB-RETRY-119

Retry System no clasificará native driver errors por sí mismo.

## DB-RETRY-120

Retry System consumirá ExecutionFailureReport.

## DB-RETRY-121

Retry System no compilará SQL.

## DB-RETRY-122

Retry System no optimizará queries.

## DB-RETRY-123

Retry System no realizará ORM hydration.

## DB-RETRY-124

Retry System no alterará authorization.

## DB-RETRY-125

Retry System no ocultará unknown outcomes.

## DB-RETRY-126

Retry System no inventará idempotency.

## DB-RETRY-127

Retry System no inventará replayability.

## DB-RETRY-128

Retry System no inventará transaction recoverability.

## DB-RETRY-129

Retry System no confundirá recoverable error con safe retry.

## DB-RETRY-130

Safety gate dominará sobre retry profile.

## DB-RETRY-131

Retries deberán preservar query ordering semantics.

## DB-RETRY-132

Retries deberán preservar duplicate semantics.

## DB-RETRY-133

Retries deberán preservar transaction semantics.

## DB-RETRY-134

Retries deberán preservar result contract.

## DB-RETRY-135

Retries deberán preservar security semantics.

## DB-RETRY-136

Retries deberán preservar tenant isolation.

## DB-RETRY-137

Retries deberán respetar resource budgets.

## DB-RETRY-138

Retries deberán ser bounded.

## DB-RETRY-139

Retry history no será un global singleton.

## DB-RETRY-140

Retry recovery será completado antes del siguiente attempt.

## DB-RETRY-141

Failed recovery impedirá nuevo attempt.

## DB-RETRY-142

Old transaction state no contaminará new transaction attempt.

## DB-RETRY-143

Old statement bindings no contaminarán new attempt.

## DB-RETRY-144

Old result state no contaminará new attempt.

## DB-RETRY-145

Old cancellation state no será ignorado.

## DB-RETRY-146

Retry budget no será reset por internal attempt.

## DB-RETRY-147

Global deadline no será reset por connection replacement.

## DB-RETRY-148

Global deadline no será reset por transaction restart.

## DB-RETRY-149

Global deadline no será reset por failover.

## DB-RETRY-150

Execution Retry System será conservative-by-default.

---

# 231. Matriz básica de decisión

| Failure | Outcome | Operation | Default |
|---|---|---|---|
| Connection acquisition failure | NOT_EXECUTED | Read | Retry candidate |
| Connection acquisition failure | NOT_EXECUTED | Write | Retry candidate |
| Connection lost before dispatch | NOT_EXECUTED | Write | Retry candidate |
| Connection lost after dispatch | UNKNOWN | Non-idempotent write | Deny |
| Connection lost after dispatch | UNKNOWN | Proven idempotent operation | Conditional |
| Deadlock | FAILED | Transaction | Restart transaction |
| Serialization failure | FAILED | Transaction | Restart transaction |
| Constraint violation | FAILED | Write | Deny |
| Binding validation error | NOT_EXECUTED | Any | Deny |
| Explicit cancellation | — | Any | Deny |
| Timeout | Depends | Read | Conditional |
| Timeout | UNKNOWN | Write | Deny by default |
| Stream failure after rows delivered | PARTIAL | Read | Deny transparent retry |
| Commit acknowledgement lost | UNKNOWN | Transaction | Deny automatic replay |
| Pool temporarily exhausted | NOT_EXECUTED | Any | Conditional |

Esta tabla es sólo una política base.

La decisión real utiliza:

```text
full execution evidence
```

---

# 232. State machine

```text
INITIAL_ATTEMPT
      │
      ▼
   RUNNING
      │
      ├───────────────► SUCCESS
      │
      ▼
    FAILED
      │
      ▼
EVALUATING_RETRY
      │
      ├───────────────► RETRY_DENIED
      │
      ├───────────────► BUDGET_EXHAUSTED
      │
      ├───────────────► CANCELLED
      │
      ▼
   RECOVERING
      │
      ├───────────────► RECOVERY_FAILED
      │
      ▼
    BACKOFF
      │
      ├───────────────► CANCELLED
      │
      ├───────────────► DEADLINE_EXCEEDED
      │
      ▼
 NEXT_ATTEMPT
      │
      └───────────────► RUNNING
```

---

# 233. Retry attempt state machine

```text
CREATED
   ↓
WAITING
   ↓
PREPARING
   ↓
RUNNING
   ↓
┌─────────────┐
│             │
▼             ▼
SUCCEEDED    FAILED
              ↓
         RECOVERING
              ↓
           CLOSED
```

---

# 234. Core services

Arquitectura recomendada:

```text
RetryCoordinator
├── RetryEligibilityAnalyzer
├── RetryScopeResolver
├── RetrySafetyGate
├── IdempotencyAnalyzer
├── ReplayabilityAnalyzer
├── PartialEffectAnalyzer
├── RetryPolicy
├── RetryBudgetManager
├── BackoffPolicy
├── RetryDelayScheduler
├── RetryRecoveryCoordinator
├── RetryHistory
└── RetryTelemetry
```

---

# 235. Avoid God RetryCoordinator

`RetryCoordinator` orquesta.

No deberá implementar directamente:

```text
driver mappings
idempotency inference
resource reset
transaction rollback
connection replacement
backoff algorithms
telemetry exporters
```

---

# 236. Integration with Execution Error System

```text
85 Execution Error System
        │
        │ ExecutionFailureReport
        ▼
86 Execution Retry System
```

El Error System responde:

```text
What failed?
What was affected?
What is known?
```

Retry responde:

```text
Can this operation safely run again?
From where?
Under what conditions?
```

---

# 237. Integration with Parameter Binding

```text
80 Parameter Binding
        │
        │ Replayability metadata
        ▼
86 Retry
```

---

# 238. Integration with Streaming

```text
83 Streaming Result
        │
        │ delivered rows / stream state
        ▼
86 Retry
```

---

# 239. Integration with Timeout/Cancellation

```text
84 Timeout/Cancellation
        │
        ├── Deadline
        └── CancellationToken
                 │
                 ▼
              Retry
```

---

# 240. Integration with Transaction System

Posteriormente:

```text
164+ Transaction Architecture
```

deberá proporcionar los mecanismos concretos para:

```text
rollback
restart
savepoint recovery
transaction ownership
transaction replay boundaries
```

El documento actual define el contrato requerido por Execution Engine.

---

# 241. Fórmula de idempotencia

Conceptualmente:

```text
Idempotent(O)
⇔
ObservableState(O(O(S)))
=
ObservableState(O(S))
```

dentro del dominio de efectos relevante.

---

# 242. Idempotency caveat

Esta fórmula deberá incluir no sólo estado de tablas, sino:

```text
triggers
sequences
audit
session state
external effects
observable results
```

cuando sean parte de la operación.

---

# 243. Fórmula de replayability

```text
Replayable(Operation)
=
ReplayableInputs
∧
ReplayableDependencies
∧
ReplayableTransactionContext
∧
ReplayableRuntimeState
```

---

# 244. Fórmula de retry scope

```text
ValidRetryScope
=
SmallestPlannedScope
that
RestoresRequiredPreconditions
∧
PreservesObservableSemantics
```

No necesariamente:

```text
smallest failed statement
```

---

# 245. Fórmula de deadline

Para attempt `n`:

```text
RemainingDeadline(n)
=
OriginalDeadline
-
ElapsedExecution
-
ElapsedRecovery
-
ElapsedBackoff
```

Nunca:

```text
OriginalDeadline
```

de nuevo.

---

# 246. Fórmula de retry budget

```text
CanAttempt(n)
=
n ≤ MaxAttempts
∧
TotalDelay ≤ MaxDelay
∧
Elapsed ≤ MaxElapsed
∧
DeadlineRemaining > 0
```

---

# 247. Fórmula de ambiguous write

```text
Mutation
∧
DispatchStarted
∧
OutcomeUnknown
∧
¬ProvenIdempotent
⇒
RetryDenied
```

---

# 248. Fórmula de partial delivery

```text
DeliveredResultCount > 0
∧
Failure
⇒
TransparentReplayDenied
```

salvo protocolo explícito de resume/replay que garantice equivalencia.

---

# 249. Fórmula de cancellation

```text
CancellationRequested
⇒
NoNewRetryAttempt
```

---

# 250. Fórmula de resource recovery

Antes del siguiente attempt:

```text
InvalidOldResources = ∅
```

dentro del conjunto de recursos que el nuevo attempt pueda observar.

---

# 251. Fórmula maestra

```text
Execution Retry System
=
Failure Evidence
+
Retry Eligibility
+
Retry Scope
+
Idempotency
+
Outcome Certainty
+
Partial Effect Analysis
+
Input Replayability
+
Transaction Replayability
+
Connection Recovery
+
Resource Recovery
+
Retry Policy
+
Retry Budget
+
Backoff
+
Jitter
+
Deadline Preservation
+
Cancellation
+
Attempt Isolation
+
Telemetry
+
Security
+
Persistent Runtime Isolation
```

---

# 252. Principio maestro

```text
Retry is a semantic operation,
not an exception-handling trick.
```

---

# 253. Regla de seguridad

Ante la elección entre:

```text
possibly duplicate a database effect
```

y:

```text
report an ambiguous failure
```

VoltStack deberá preferir por defecto:

```text
report the ambiguity
```

antes que inventar seguridad.

---

# 254. Resultado arquitectónico

Con los documentos `76–86`, el Execution Engine queda estructurado de la siguiente manera:

```text
DATABASE EXECUTION ENGINE

ExecutionPlan
     │
     ▼
QueryExecutor
     │
     ▼
Statement Execution
     │
     ├── Prepared Statement
     │
     ├── Parameter Binding
     │
     └── Driver Execution
     │
     ▼
Result System
     │
     ├── Result Cursor
     └── Streaming Result
     │
     ▼
Execution Outcome
     │
     ├── Timeout / Cancellation
     ├── Execution Error
     └── Execution Retry
```

Con esto queda cerrado el **Bloque 7 — Execution Engine**.

---

# 255. Siguiente documento

El siguiente documento del plan maestro es:

```text
87_DATABASE_SCHEMA_ARCHITECTURE.md
```

Con él comienza:

```text
BLOCK 8 — SCHEMA
```

La secuencia será:

```text
87_DATABASE_SCHEMA_ARCHITECTURE.md
88_DATABASE_SCHEMA_MODEL.md
89_DATABASE_SCHEMA_AST_SYSTEM.md
90_DATABASE_SCHEMA_BUILDER_SYSTEM.md
91_DATABASE_TABLE_DEFINITION_SYSTEM.md
92_DATABASE_COLUMN_DEFINITION_SYSTEM.md
93_DATABASE_INDEX_SYSTEM.md
94_DATABASE_FOREIGN_KEY_SYSTEM.md
95_DATABASE_CONSTRAINT_SYSTEM.md
96_DATABASE_SCHEMA_INTROSPECTION_SYSTEM.md
97_DATABASE_SCHEMA_METADATA_SYSTEM.md
98_DATABASE_SCHEMA_DIFF_SYSTEM.md
99_DATABASE_SCHEMA_COMPILER_SYSTEM.md
100_DATABASE_SCHEMA_PLATFORM_COMPATIBILITY_SYSTEM.md
```

El documento `87_DATABASE_SCHEMA_ARCHITECTURE.md` deberá establecer la frontera fundamental:

```text
Schema Model
≠
Schema Builder
≠
Schema AST
≠
Schema Introspection
≠
Schema Diff
≠
Schema Compiler
≠
Migration
```

y conectar la nueva arquitectura con:

```text
Database Platform
        ↓
Schema Capability System
        ↓
Schema Model / AST
        ↓
Schema Compiler
        ↓
Execution Engine
```

sin permitir que el Schema System genere SQL directamente fuera del Compiler.