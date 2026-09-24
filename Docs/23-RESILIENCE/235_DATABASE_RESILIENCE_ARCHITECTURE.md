# 235_DATABASE_RESILIENCE_ARCHITECTURE.md

# VoltStack Quantum Database
## Database Resilience Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 235 — Database Resilience Architecture  
**Bloque:** 23 — Resilience  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `234_DATABASE_DATABASE_PERMISSION_MODEL.md`  
**Siguiente documento:** `236_DATABASE_CONNECTION_FAILURE_HANDLING_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura general de resiliencia de `VoltStack/Quantum/Database`.

Su objetivo es establecer cómo el subsistema Database deberá:

- detectar fallos;
- clasificarlos;
- determinar su impacto;
- contenerlos;
- decidir si una operación puede repetirse;
- aplicar backoff;
- cancelar operaciones;
- reaccionar ante timeouts;
- utilizar circuit breakers;
- realizar failover;
- recuperar conexiones;
- preservar semántica transaccional;
- proteger recursos;
- manejar resultados inciertos;
- producir evidencia diagnóstica;
- evitar cascadas de fallos.

La regla central será:

> **VoltStack Database solo deberá intentar recuperación automática cuando pueda demostrar que dicha recuperación preserva la semántica, consistencia y seguridad de la operación.**

Por tanto:

```text
Failure
≠
Retry
```

y:

```text
Unknown Outcome
≠
Safe Retry
```

---

# 2. Objetivo arquitectónico

La resiliencia no será implementada como:

```text
catch Throwable
    ↓
retry()
```

La arquitectura deberá distinguir:

```text
Failure Detection
        ↓
Failure Classification
        ↓
Outcome Classification
        ↓
Retry Safety Analysis
        ↓
Recovery Policy
        ↓
Recovery Action
        ↓
Result / Failure / Unknown
```

---

# 3. Problema fundamental

Las bases de datos son sistemas remotos.

Entre aplicación y servidor existen:

```text
Application
    │
    ▼
Driver
    │
    ▼
Socket
    │
    ▼
Network
    │
    ▼
Proxy / Load Balancer
    │
    ▼
Database Server
    │
    ▼
Storage / Replication
```

Un fallo observado por la aplicación no necesariamente revela qué ocurrió realmente dentro del servidor.

Ejemplo:

```text
COMMIT
   ↓
Database commits
   ↓
connection lost
   ↓
client receives error
```

Desde la aplicación:

```text
Did commit happen?
=
UNKNOWN
```

Este caso será fundamental para toda la arquitectura.

---

# 4. Resilience ≠ Retry

Resilience incluye:

```text
Detection
Classification
Containment
Timeouts
Cancellation
Retry
Backoff
Circuit Breaking
Failover
Recovery
Resource Protection
Observability
```

Por tanto:

```text
Resilience
⊃
Retry
```

---

# 5. Resilience ≠ High Availability

High Availability es una propiedad del sistema completo.

Puede involucrar:

```text
replication
clustering
failover
DNS
load balancing
storage
network
orchestration
```

Database Resilience es la capacidad del framework de reaccionar correctamente a esas condiciones.

```text
Database Resilience
≠
Database High Availability
```

---

# 6. Resilience ≠ Distributed Transaction

Failover o retry no crean atomicidad distribuida.

```text
Failover
≠
2PC
```

y:

```text
Retry
≠
Distributed Commit
```

---

# 7. Resilience ≠ Error Suppression

Nunca:

```php
try {
    $query->execute();
} catch (\Throwable) {
    return [];
}
```

como estrategia general.

Esto convertiría:

```text
Failure
→ Fake Success
```

---

# 8. Principio de evidencia

VoltStack deberá preservar únicamente aquello que pueda afirmar con evidencia suficiente.

Los outcomes fundamentales serán:

```text
SUCCESS
FAILURE
UNKNOWN
```

---

# 9. UNKNOWN como estado de primera clase

`UNKNOWN` no es un error de modelado.

Es una realidad de sistemas distribuidos.

Ejemplo:

```text
UPDATE sent
connection lost
```

puede significar:

```text
A) server never received UPDATE
B) server received but did not execute
C) server executed and rolled back
D) server executed and committed
E) server outcome cannot be observed
```

Por tanto:

```text
ConnectionLost
≠
OperationFailed
```

---

# 10. Regla de incertidumbre

> **VoltStack nunca deberá convertir automáticamente un resultado UNKNOWN en FAILURE o SUCCESS.**

---

# 11. Arquitectura general

```text
Database Operation
       │
       ▼
Execution Boundary
       │
       ▼
Failure Detector
       │
       ▼
Failure Classifier
       │
       ├── Connection
       ├── Query
       ├── Transaction
       ├── Timeout
       ├── Cancellation
       ├── Deadlock
       ├── Constraint
       ├── Resource
       ├── Topology
       └── Unknown
       │
       ▼
Outcome Analyzer
       │
       ├── FAILURE_CONFIRMED
       ├── SUCCESS_CONFIRMED
       └── UNKNOWN
       │
       ▼
Recovery Decision Engine
       │
       ├── PROPAGATE
       ├── RETRY
       ├── RECONNECT
       ├── FAILOVER
       ├── BACKOFF
       ├── OPEN_CIRCUIT
       ├── CANCEL
       └── ABORT
       │
       ▼
Recovery Executor
       │
       ▼
Final Outcome
```

---

# 12. Capas de resiliencia

La arquitectura se dividirá conceptualmente en:

```text
┌───────────────────────────────┐
│ Operation Resilience          │
├───────────────────────────────┤
│ Transaction Resilience        │
├───────────────────────────────┤
│ Query Resilience              │
├───────────────────────────────┤
│ Connection Resilience         │
├───────────────────────────────┤
│ Topology Resilience           │
├───────────────────────────────┤
│ Resource Protection           │
├───────────────────────────────┤
│ Driver / Platform             │
└───────────────────────────────┘
```

---

# 13. Failure Domain

Todo fallo deberá asociarse, cuando sea posible, con un dominio.

```php
enum DatabaseFailureDomain
{
    case CONNECTION;
    case QUERY;
    case TRANSACTION;
    case DRIVER;
    case PLATFORM;
    case NETWORK;
    case TOPOLOGY;
    case REPLICA;
    case SHARD;
    case RESOURCE;
    case TIMEOUT;
    case CANCELLATION;
    case SECURITY;
    case APPLICATION;
    case UNKNOWN;
}
```

---

# 14. Failure Domain ≠ Exception Class

Una excepción de driver puede representar:

```text
connection loss
deadlock
constraint violation
timeout
server shutdown
```

Por tanto:

```text
Exception Class
≠
Semantic Failure Domain
```

---

# 15. FailureCategory

```php
enum DatabaseFailureCategory
{
    case TRANSIENT;
    case PERMANENT;
    case CONCURRENCY;
    case CAPACITY;
    case CONFIGURATION;
    case SECURITY;
    case DATA;
    case TOPOLOGY;
    case CANCELLATION;
    case UNKNOWN;
}
```

---

# 16. Transient failure

Un fallo transitorio puede desaparecer sin cambiar la operación.

Ejemplos:

```text
temporary network interruption
temporary replica unavailability
connection refused during restart
deadlock
short-lived overload
```

Pero:

```text
Transient
≠
Automatically Retryable
```

---

# 17. Permanent failure

Ejemplos:

```text
invalid SQL semantics
missing required table
invalid credentials
unsupported feature
constraint violation
invalid data type
permission denied
```

Normalmente:

```text
Permanent Failure
→ No Retry
```

---

# 18. Capacity failure

Ejemplos:

```text
connection pool exhausted
memory budget exceeded
server too busy
queue saturated
disk full
resource quota exceeded
```

---

# 19. Configuration failure

Ejemplos:

```text
invalid host
invalid database
invalid TLS configuration
missing driver
invalid connection option
```

---

# 20. Security failure

Ejemplos:

```text
authentication rejected
permission denied
credential expired
TLS verification failure
security policy rejection
```

No deberán tratarse como fallos transitorios ordinarios.

---

# 21. Data failure

Ejemplos:

```text
unique constraint
foreign key constraint
not-null constraint
invalid cast
domain constraint
```

---

# 22. Concurrency failure

Ejemplos:

```text
deadlock
serialization failure
optimistic lock conflict
lock timeout
```

Cada uno tendrá semántica distinta.

---

# 23. FailureSeverity

Podrá clasificarse:

```php
enum DatabaseFailureSeverity
{
    case INFO;
    case WARNING;
    case ERROR;
    case CRITICAL;
}
```

Severity no decidirá retry por sí sola.

---

# 24. Failure scope

Un fallo puede afectar:

```text
STATEMENT
TRANSACTION
CONNECTION
ENDPOINT
REPLICA
SHARD
DATABASE
CLUSTER
WORKER
APPLICATION
```

---

# 25. Scope matters

Ejemplo:

```text
syntax error
```

afecta normalmente:

```text
statement
```

mientras:

```text
server shutdown
```

puede afectar:

```text
endpoint
```

---

# 26. DatabaseFailure

Objeto canónico:

```php
final readonly class DatabaseFailure
{
    public function __construct(
        public DatabaseFailureId $id,
        public DatabaseFailureDomain $domain,
        public DatabaseFailureCategory $category,
        public DatabaseFailureScope $scope,
        public DatabaseFailureSeverity $severity,
        public DatabaseFailureCode $code,
        public DatabaseFailureEvidence $evidence,
        public ?Throwable $cause,
    ) {}
}
```

---

# 27. Raw driver exception

Las excepciones del driver deberán normalizarse.

```text
PDOException
DriverException
NativeError
        ↓
Failure Normalizer
        ↓
DatabaseFailure
```

---

# 28. Driver information preservation

La normalización no deberá destruir evidencia útil.

Podrá conservar:

```text
SQLSTATE
vendor code
driver code
server message
connection state
platform
operation phase
```

de forma segura.

---

# 29. Error message ≠ classification

No depender de:

```php
str_contains($message, 'server has gone away')
```

como mecanismo primario.

Preferir:

```text
structured driver information
SQLSTATE
vendor error codes
driver capabilities
platform classifier
```

---

# 30. Platform-specific classifiers

Podrán existir:

```text
MySqlFailureClassifier
MariaDbFailureClassifier
PostgreSqlFailureClassifier
SqliteFailureClassifier
```

---

# 31. MySQL ≠ MariaDB

Aunque compartan protocolos/códigos en muchas áreas:

```text
MySQL failure semantics
≠
MariaDB failure semantics
```

cuando existan diferencias.

---

# 32. Capability-driven resilience

El sistema deberá preguntar:

```text
supportsSavepoints()
supportsCancellation()
supportsReconnect()
supportsReadOnlyTransaction()
supportsLockTimeout()
```

en vez de introducir vendor conditionals por todas partes.

---

# 33. Failure phase

Toda operación podrá encontrarse en:

```php
enum DatabaseOperationPhase
{
    case NOT_STARTED;
    case PREPARING;
    case BINDING;
    case SENDING;
    case EXECUTING;
    case RECEIVING;
    case HYDRATING;
    case FLUSHING;
    case COMMITTING;
    case ROLLING_BACK;
    case COMPLETED;
}
```

---

# 34. Phase matters

Una conexión perdida en:

```text
PREPARING
```

no tiene la misma semántica que una conexión perdida durante:

```text
COMMITTING
```

---

# 35. Outcome model

```php
enum DatabaseOutcomeStatus
{
    case SUCCESS;
    case FAILURE;
    case UNKNOWN;
}
```

---

# 36. DatabaseOperationOutcome

```php
final readonly class DatabaseOperationOutcome
{
    public function __construct(
        public DatabaseOutcomeStatus $status,
        public DatabaseOperationPhase $phase,
        public DatabaseOutcomeEvidence $evidence,
        public ?DatabaseFailure $failure,
    ) {}
}
```

---

# 37. Statement outcome ≠ transaction outcome

```text
Statement SUCCESS
```

no implica:

```text
Transaction COMMITTED
```

---

# 38. Transaction outcome ≠ ORM state

Incluso:

```text
Transaction ROLLED_BACK
```

no implica:

```text
Object Graph Automatically Rewound
```

---

# 39. Existing transaction architecture

Esta arquitectura deberá respetar:

```text
164_DATABASE_TRANSACTION_ARCHITECTURE.md
165_DATABASE_TRANSACTION_MANAGER_SYSTEM.md
166_DATABASE_TRANSACTION_CONTEXT_SYSTEM.md
170_DATABASE_TRANSACTION_RETRY_SYSTEM.md
171_DATABASE_DEADLOCK_HANDLING_SYSTEM.md
```

---

# 40. Failure Detection

La detección podrá originarse en:

```text
Driver
Connection
Executor
Transaction Manager
Timeout Manager
Cancellation Token
Pool
Replica Monitor
Health Check
Resource Governor
```

---

# 41. Detector ≠ policy

Un detector informa:

```text
something happened
```

No decide:

```text
retry
failover
ignore
```

---

# 42. FailureClassifier

```php
interface DatabaseFailureClassifier
{
    public function classify(
        DatabaseFailureObservation $observation
    ): DatabaseFailure;
}
```

---

# 43. Recovery Policy

Determina qué acciones están permitidas.

```php
interface DatabaseRecoveryPolicy
{
    public function decide(
        DatabaseRecoveryContext $context
    ): DatabaseRecoveryDecision;
}
```

---

# 44. RecoveryDecision

```php
enum DatabaseRecoveryAction
{
    case PROPAGATE;
    case RETRY;
    case RECONNECT;
    case FAILOVER;
    case BACKOFF;
    case OPEN_CIRCUIT;
    case CANCEL;
    case ABORT;
}
```

---

# 45. Recovery action composition

Una decisión puede producir:

```text
BACKOFF
    ↓
RECONNECT
    ↓
RETRY
```

o:

```text
MARK_ENDPOINT_UNHEALTHY
    ↓
FAILOVER
    ↓
RETRY
```

si es seguro.

---

# 46. Retry safety

La pregunta no será:

```text
Can this error disappear?
```

sino:

```text
Can this logical operation be replayed
without violating correctness?
```

---

# 47. RetrySafety

```php
enum RetrySafety
{
    case SAFE;
    case CONDITIONALLY_SAFE;
    case UNSAFE;
    case UNKNOWN;
}
```

---

# 48. UNKNOWN retry safety

Default:

```text
UNKNOWN
→ DO NOT RETRY
```

---

# 49. Retryable error ≠ retryable operation

Ejemplo:

```text
network timeout = transient
```

pero:

```text
INSERT payment
```

puede haber sido ejecutado.

Por tanto:

```text
Transient Failure
+
Unknown Outcome
=
Unsafe Blind Retry
```

---

# 50. Idempotency

La idempotencia será una fuente importante de retry safety.

```text
f(f(x)) = f(x)
```

conceptualmente.

Pero:

```text
Idempotent SQL Shape
≠
Guaranteed Safe Retry
```

sin considerar transacción, side effects y outcome.

---

# 51. Idempotency key

Operaciones de aplicación podrán utilizar:

```text
IdempotencyKey
```

cuando el dominio lo soporte.

Database core no deberá inventar idempotencia.

---

# 52. Read retries

Reads pueden parecer seguros, pero existen excepciones:

```text
SELECT ... FOR UPDATE
volatile functions
temporary state
session state
side-effecting functions
transaction semantics
```

Por tanto:

```text
SELECT
≠
Automatically Retryable
```

---

# 53. Write retries

Writes requieren análisis más estricto.

---

# 54. Transaction retries

Cuando un deadlock/serialization failure invalida la transacción:

```text
retry statement
```

normalmente será incorrecto.

Debe repetirse:

```text
whole transaction boundary
```

cuando sea seguro.

---

# 55. Retry boundary

```text
Retry Boundary
=
smallest semantic operation
that can be safely replayed
```

---

# 56. Retry scopes

```text
STATEMENT
QUERY_OPERATION
TRANSACTION
CHUNK
IMPORT_BATCH
BACKGROUND_OPERATION
CUSTOM
```

---

# 57. Retry attempt

Cada attempt tendrá identidad propia.

```text
Logical Operation
├── Attempt 1
├── Attempt 2
└── Attempt 3
```

---

# 58. Logical operation identity

Todos los attempts compartirán:

```text
OperationId
```

y tendrán:

```text
AttemptId
```

individual.

---

# 59. Retry budget

No deberá existir retry infinito.

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

# 60. Retry storm

Un fallo masivo puede provocar:

```text
1000 requests
    ↓
all fail
    ↓
all retry immediately
    ↓
database overload increases
```

Esto deberá evitarse.

---

# 61. Backoff

Estrategias:

```text
NONE
FIXED
LINEAR
EXPONENTIAL
EXPONENTIAL_WITH_JITTER
CUSTOM
```

---

# 62. Default distributed strategy

Para fallos transitorios de infraestructura, una política común será:

```text
Exponential Backoff + Jitter
```

cuando retry sea seguro.

---

# 63. Backoff ≠ retry safety

Añadir espera no vuelve segura una operación insegura.

---

# 64. Jitter

Evita sincronización masiva de retries.

Conceptualmente:

```text
delay_n
=
min(
    maxDelay,
    baseDelay × 2^n
)
+
jitter
```

---

# 65. Retry deadline

El retry budget deberá respetar:

```text
request deadline
query timeout
transaction timeout
job deadline
```

---

# 66. Deadline

```text
RemainingDeadline
<
RequiredRetryBudget
```

puede producir:

```text
DO_NOT_RETRY
```

---

# 67. Timeout architecture

Timeout es un límite temporal.

```text
Timeout
≠
Cancellation
```

---

# 68. Timeout types

```text
CONNECTION_TIMEOUT
ACQUIRE_TIMEOUT
QUERY_TIMEOUT
LOCK_TIMEOUT
TRANSACTION_TIMEOUT
OPERATION_TIMEOUT
POOL_WAIT_TIMEOUT
```

---

# 69. Timeout ≠ confirmed failure

Un client timeout puede ocurrir mientras el servidor continúa ejecutando.

---

# 70. Critical rule

```text
Client Timeout
≠
Database Rollback
```

---

# 71. Cancellation

Cancellation representa una solicitud de detener trabajo.

---

# 72. CancellationToken

```php
interface DatabaseCancellationToken
{
    public function isCancellationRequested(): bool;

    public function throwIfCancellationRequested(): void;
}
```

---

# 73. Cancellation propagation

```text
HTTP disconnect
Job cancellation
Deadline
Manual cancellation
       ↓
CancellationToken
       ↓
Query Executor
       ↓
Driver / Platform cancellation
```

cuando sea soportado.

---

# 74. Cancellation capability

No todos los drivers/plataformas pueden cancelar una query remota de forma fiable.

---

# 75. Cancellation request ≠ cancellation confirmed

```text
CancellationRequested
≠
DatabaseStoppedExecution
```

---

# 76. Circuit Breaker

Circuit Breaker evita bombardear un recurso que muestra fallos persistentes.

---

# 77. Circuit states

```php
enum CircuitState
{
    case CLOSED;
    case OPEN;
    case HALF_OPEN;
}
```

---

# 78. CLOSED

Operaciones permitidas normalmente.

---

# 79. OPEN

Operaciones elegibles son rechazadas rápidamente.

---

# 80. HALF_OPEN

Se permiten probes controlados para verificar recuperación.

---

# 81. Circuit breaker ≠ health check

Health check observa.

Circuit breaker controla tráfico.

---

# 82. Circuit breaker ≠ failover

Abrir circuito no selecciona automáticamente otro endpoint.

---

# 83. Circuit scope

Circuit breakers podrán asociarse a:

```text
endpoint
replica
connection target
external dependency
```

No necesariamente a toda la logical database.

---

# 84. Per-query circuit breaker

Evitar crear circuitos de alta cardinalidad por SQL/query fingerprint salvo casos especializados.

---

# 85. Failure threshold

El circuit breaker podrá considerar:

```text
consecutive failures
failure ratio
minimum request volume
rolling window
latency
capacity failures
```

---

# 86. Not every failure opens circuit

Errores como:

```text
unique constraint
syntax error
permission denied
```

normalmente no significan que el endpoint esté enfermo.

---

# 87. Circuit failure contribution

Solo fallos relevantes de infraestructura/capacidad deberán contribuir según policy.

---

# 88. Failover

Failover significa cambiar la autoridad/ruta operacional hacia otro endpoint/topology owner válido.

---

# 89. Failover ≠ retry

```text
Failover
→ change target

Retry
→ repeat operation
```

Pueden combinarse, pero son distintos.

---

# 90. Failover ≠ reconnect

Reconnect:

```text
same logical target
new connection
```

Failover:

```text
different eligible authority target
```

---

# 91. Failover safety

Debe considerar:

```text
transaction state
write authority
replication state
topology generation
read consistency
sticky state
unknown outcomes
```

---

# 92. No transparent mid-transaction failover

Default:

```text
active transaction
+
connection lost
→
transaction failed/unknown
```

No:

```text
connect another server
→ continue transaction
```

---

# 93. Transaction pinning

Una transacción permanecerá ligada a:

```text
connection
endpoint
shard
writer authority
```

según arquitectura existente.

---

# 94. Writer failover

Después de writer failover:

```text
new transaction
```

podrá utilizar el nuevo writer.

Una transacción previa no continúa mágicamente.

---

# 95. Split brain

La arquitectura deberá evitar aceptar múltiples writers no verificados.

```text
Multiple Possible Writers
→
UNKNOWN AUTHORITY
→
do not blindly write
```

---

# 96. Authority epoch

La topología podrá utilizar:

```text
AuthorityEpoch
```

para identificar generaciones de ownership.

---

# 97. Stale topology

Una conexión basada en una topology generation antigua puede ser rechazada.

---

# 98. Replica failover

Para reads elegibles:

```text
Replica A unavailable
→ Replica B
```

puede ser seguro si:

```text
consistency requirements
lag requirements
transaction state
sticky requirements
```

se preservan.

---

# 99. Replica switch ≠ same snapshot

Cambiar replica puede observar un estado distinto.

---

# 100. Read consistency

Failover deberá preservar:

```text
BEST_EFFORT
READ_YOUR_WRITES
MONOTONIC
SNAPSHOT
CUSTOM
```

según contexto.

---

# 101. Shard failure

Un shard caído no significa que una query distribuida pueda ignorarlo.

---

# 102. Partial distributed result

```text
Shard A success
Shard B success
Shard C unavailable
```

no deberá presentarse como:

```text
complete result
```

---

# 103. Distributed outcome

Estados posibles:

```text
COMPLETE
PARTIAL
FAILED
UNKNOWN
```

cuando una API soporte resultados parciales.

---

# 104. Ordinary query default

Para una query que promete resultado completo:

```text
missing shard
→ failure
```

---

# 105. Recovery

Recovery será el proceso de devolver un componente a estado utilizable.

---

# 106. Recovery ≠ retry

Ejemplo:

```text
Connection broken
→ discard connection
→ create new connection
```

es recovery.

No necesariamente retry.

---

# 107. Connection recovery

Podrá implicar:

```text
mark broken
remove from pool
close socket
recreate connection
reapply safe initialization
verify endpoint
```

---

# 108. Session state problem

Una conexión puede contener:

```text
session variables
temporary tables
transaction state
isolation level
timezone
search path
locks
prepared statements
```

Por tanto:

```text
Reconnect
≠
Continue Same Session
```

---

# 109. Safe connection initialization

Solo state explícitamente reconstruible deberá reaplicarse.

---

# 110. Temporary tables

No pueden asumirse presentes después de reconnect.

---

# 111. Session locks

No pueden asumirse preservados.

---

# 112. Prepared statements

Podrán requerir recompilación/repreparación.

---

# 113. Transaction recovery

Una transacción rota normalmente no se “repara”.

```text
Broken Transaction
→ abort
→ optionally replay whole logical transaction
```

si retry safety lo permite.

---

# 114. Tainted transaction

Estados posibles:

```text
ACTIVE
COMMITTED
ROLLED_BACK
FAILED
TAINTED
UNKNOWN
```

---

# 115. TAINTED

Significa:

```text
context can no longer be trusted for ordinary continuation
```

---

# 116. EntityManager impact

Si el resultado de persistencia es incierto:

```text
EntityManager
→ TAINTED
```

según arquitectura ORM.

---

# 117. IdentityMap impact

No deberá modificarse para fingir rollback/success.

---

# 118. Persistence consistency

Integración con:

```text
134_DATABASE_PERSISTENCE_CONSISTENCY_SYSTEM.md
```

Estados relevantes:

```text
CONSISTENT
STALE
UNCERTAIN
INCONSISTENT
TAINTED
UNKNOWN
```

---

# 119. Recovery does not fabricate ORM truth

---

# 120. Resource Exhaustion

El subsistema deberá protegerse de:

```text
connection exhaustion
memory exhaustion
cursor exhaustion
statement exhaustion
worker saturation
query queue growth
result size explosion
retry amplification
```

---

# 121. Resource protection

Principios:

```text
Bound
Reject
Backpressure
Cancel
Degrade safely
```

---

# 122. Backpressure

Cuando consumidores no pueden seguir el ritmo:

```text
Producer
   ↓
Bounded Buffer
   ↓
Consumer
```

No permitir crecimiento ilimitado.

---

# 123. Connection pool saturation

Podrá producir:

```text
WAIT
TIMEOUT
REJECT
```

según policy.

---

# 124. Pool saturation ≠ endpoint failure

Un pool local agotado no significa necesariamente que Database esté caído.

---

# 125. Memory budget

Large Dataset, Hydration, ORM, Lazy Collection y Result systems deberán respetar budgets.

---

# 126. Query result limit

Resource governance podrá imponer:

```text
maxRows
maxBytes
maxDuration
```

---

# 127. Retry amplification protection

Cada retry consume recursos.

Por tanto:

```text
OriginalLoad × RetryFactor
```

deberá permanecer acotado.

---

# 128. Retry budget hierarchy

Podrán existir:

```text
per-operation budget
per-request budget
per-worker budget
global dependency budget
```

---

# 129. Retry token budget

Una arquitectura avanzada podrá utilizar tokens para impedir retry storms.

---

# 130. Bulkhead pattern

Podrá aislar recursos:

```text
Primary Queries
Migration
Exports
Analytics
Background Jobs
```

para que una carga no agote todo el sistema.

---

# 131. Bulkhead ≠ transaction

---

# 132. Resource partitions

Ejemplo:

```text
Interactive Pool
Background Pool
Migration Pool
```

cuando infraestructura lo permita.

---

# 133. Load shedding

Ante saturación crítica:

```text
reject low-priority work
```

puede ser más seguro que aceptar trabajo imposible de completar.

---

# 134. Priority

Podrá existir:

```php
enum DatabaseOperationPriority
{
    case CRITICAL;
    case HIGH;
    case NORMAL;
    case LOW;
    case BACKGROUND;
}
```

---

# 135. Priority ≠ permission

Tener mayor priority no concede mayor authority.

---

# 136. Graceful degradation

Solo será permitida cuando la API tenga semántica explícita para ello.

---

# 137. Dangerous degradation

Nunca:

```text
database unavailable
→ return empty array
```

si vacío significa “no existen registros”.

---

# 138. Cache fallback

Un Result Cache podría actuar como fallback solo bajo una política explícita.

---

# 139. Stale cache

Deberá marcarse:

```text
STALE
```

si la API permite servir stale data.

---

# 140. Stale ≠ fresh

---

# 141. Cache fallback ≠ database truth

---

# 142. Fallback policy

Ejemplo:

```php
enum DatabaseFallbackPolicy
{
    case NONE;
    case STALE_CACHE_IF_ALLOWED;
    case READ_REPLICA_IF_CONSISTENT;
    case CUSTOM;
}
```

---

# 143. Fallback safety

Fallback no deberá violar:

```text
tenant isolation
authorization
read-your-writes
transaction semantics
sensitive-data policy
```

---

# 144. Resilience Context

```php
final readonly class DatabaseResilienceContext
{
    public function __construct(
        public DatabaseOperationId $operation,
        public DatabaseOperationType $type,
        public DatabaseOperationPhase $phase,
        public DatabaseExecutionContext $execution,
        public DatabaseConsistencyRequirement $consistency,
        public DatabaseTransactionContextReference $transaction,
        public DatabaseDeadline $deadline,
        public DatabaseCancellationToken $cancellation,
        public RetryBudget $retryBudget,
        public DatabaseResourceBudget $resourceBudget,
    ) {}
}
```

---

# 145. Context is scoped

Nunca global mutable.

---

# 146. Operation classification

Operaciones podrán clasificarse:

```text
READ
WRITE
TRANSACTION
SCHEMA
MIGRATION
IMPORT
EXPORT
BACKUP
RESTORE
ADMIN
```

---

# 147. Resilience profile

```php
enum DatabaseResilienceProfile
{
    case STRICT;
    case INTERACTIVE;
    case BACKGROUND;
    case BULK;
    case MIGRATION;
    case CUSTOM;
}
```

---

# 148. Profile ≠ hidden semantics

El profile solo agrupa políticas explícitas.

No deberá cambiar silenciosamente garantías fundamentales.

---

# 149. Strict profile

Ejemplo conceptual:

```text
low retries
no stale fallback
strict timeout
unknown → abort
```

---

# 150. Background profile

Podrá permitir:

```text
larger retry window
longer backoff
checkpoint recovery
```

si la operación es replay-safe.

---

# 151. Bulk profile

Podrá utilizar:

```text
chunk retry
checkpoint
bounded parallelism
```

---

# 152. Migration profile

Deberá ser extremadamente conservador con retries.

---

# 153. DDL retry

DDL no deberá repetirse ciegamente tras outcome incierto.

---

# 154. Import resilience

Integración con:

```text
206_DATABASE_IMPORT_SYSTEM.md
```

Puede utilizar:

```text
batch checkpoints
idempotency
per-batch transaction
resume
```

---

# 155. Export resilience

Integración con:

```text
207_DATABASE_EXPORT_SYSTEM.md
```

Puede utilizar:

```text
checkpoint
stream restart
destination resume
```

solo si el formato/destino lo permiten.

---

# 156. Large Dataset resilience

Integración con:

```text
208_DATABASE_LARGE_DATASET_PROCESSING_SYSTEM.md
```

---

# 157. Chunk resilience

Integración con:

```text
201_DATABASE_CHUNK_PROCESSING_SYSTEM.md
```

Retry podrá ocurrir por chunk cuando:

```text
chunk processing is replay-safe
```

---

# 158. Checkpoint ≠ exactly once

```text
Checkpoint + Retry
≠
Exactly Once
```

---

# 159. Lazy Collection resilience

Una excepción puede ocurrir durante iteración.

La Lazy Collection deberá liberar recursos.

---

# 160. Streaming resilience

Una conexión rota durante stream no implica que el stream pueda continuar automáticamente.

---

# 161. Result cursor recovery

```text
Result Cursor Lost
≠
Resume Possible
```

salvo estrategia explícita.

---

# 162. Pagination resilience

Offset page puede simplemente repetirse, pero el dataset puede haber cambiado.

---

# 163. Cursor pagination resilience

Un cursor lógico puede permitir repetir una página, sujeto a:

```text
cursor validity
topology
consistency
dataset mutation
```

---

# 164. Event system

Resilience podrá emitir eventos.

---

# 165. Resilience events

Ejemplos:

```text
DatabaseFailureDetected
DatabaseFailureClassified
DatabaseRetryScheduled
DatabaseRetryStarted
DatabaseRetrySucceeded
DatabaseRetryExhausted
DatabaseCircuitOpened
DatabaseCircuitHalfOpened
DatabaseCircuitClosed
DatabaseFailoverStarted
DatabaseFailoverCompleted
DatabaseRecoveryStarted
DatabaseRecoveryCompleted
DatabaseResourceLimitReached
```

---

# 166. Events ≠ control plane

Listeners no deberán redefinir silenciosamente el outcome.

---

# 167. Event listener failure

Un listener de observabilidad fallando no deberá convertir una operación DB exitosa en failure salvo contrato explícito.

---

# 168. Transaction event distinction

Después de COMMIT:

```text
afterCommit listener fails
```

no significa:

```text
transaction failed
```

---

# 169. Telemetry

Resilience será altamente observable.

---

# 170. Metrics

Ejemplos:

```text
db.failures.total
db.failures.transient
db.failures.permanent
db.failures.unknown

db.retries.total
db.retries.exhausted

db.circuit.open
db.circuit.half_open

db.failover.total
db.recovery.total

db.resource.exhaustion
db.pool.wait_timeout
```

---

# 171. Metric dimensions

Podrán incluir:

```text
failure domain
failure category
operation type
platform
endpoint role
recovery action
outcome
```

---

# 172. High-cardinality labels

Evitar:

```text
SQL
query parameters
user IDs
tenant IDs
exception messages
```

como labels por defecto.

---

# 173. Tracing

Un logical operation deberá mantener el mismo trace context a través de retries.

---

# 174. Retry spans

Conceptualmente:

```text
db.operation
├── attempt.1
├── backoff
├── attempt.2
└── attempt.3
```

---

# 175. Retry telemetry

Registrar:

```text
attempt number
reason
delay
failure category
retry safety
remaining budget
```

---

# 176. Unknown outcome telemetry

Deberá ser visible explícitamente.

Nunca registrarlo simplemente como:

```text
error
```

sin preservar incertidumbre.

---

# 177. Logging

Los logs deberán incluir:

```text
operation ID
attempt ID
failure code
phase
outcome
recovery action
```

con redacción de datos sensibles.

---

# 178. Query safety

No registrar bindings sensibles sin aplicar:

```text
232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM.md
```

---

# 179. Audit ≠ telemetry

Audit responde:

```text
what security-relevant operation occurred?
```

Telemetry responde:

```text
how did the system behave?
```

---

# 180. Debug information

El sistema podrá producir:

```text
Failure:
    domain: CONNECTION
    category: TRANSIENT
    scope: ENDPOINT
    phase: RECEIVING

Outcome:
    UNKNOWN

Retry:
    UNSAFE

Recovery:
    connection discarded

Circuit:
    failure recorded
```

---

# 181. Explain resilience

Podrá existir:

```php
$query->explainResilience();
```

conceptualmente.

---

# 182. Example explain

```text
Operation: WRITE
Transaction: ACTIVE
Retry boundary: TRANSACTION
Retry safety: CONDITIONALLY_SAFE
Max attempts: 3
Backoff: EXPONENTIAL_WITH_JITTER
Failover: DISABLED_DURING_TRANSACTION
Unknown outcome policy: ABORT
```

---

# 183. Resilience middleware

Podrá existir un pipeline:

```text
Deadline
    ↓
Cancellation
    ↓
Circuit Breaker
    ↓
Resource Budget
    ↓
Execution
    ↓
Failure Classification
    ↓
Retry Decision
    ↓
Recovery
```

---

# 184. Middleware ordering matters

Ejemplo:

```text
Circuit OPEN
```

debe poder rechazar antes de adquirir una conexión.

---

# 185. Retry should not bypass circuit

Un retry también deberá pasar por las reglas apropiadas del circuit breaker.

---

# 186. Resource budget across retries

Retry no reinicia automáticamente el budget.

---

# 187. Deadline across retries

El deadline pertenece a la operación lógica.

No a cada attempt.

---

# 188. Cancellation across retries

Una cancelación detendrá attempts futuros.

---

# 189. Security across retries

Retry deberá preservar:

```text
principal
permission context
tenant
authorization scope
```

---

# 190. Tenant across retries

Nunca cambiar tenant para “encontrar” una ejecución exitosa.

---

# 191. Shard across retries

Una operación shard-bound no podrá migrar arbitrariamente a otro shard.

---

# 192. Replica retries

Un read podrá probar otra replica únicamente si la política de consistencia lo permite.

---

# 193. Writer retries

No cambiar writer sin autoridad topológica verificada.

---

# 194. Connection state

Una conexión tendrá health state.

```php
enum ConnectionHealthState
{
    case HEALTHY;
    case SUSPECT;
    case BROKEN;
    case CLOSED;
    case UNKNOWN;
}
```

---

# 195. SUSPECT

Un error puede indicar que la conexión debe verificarse antes de reutilizarla.

---

# 196. BROKEN

No deberá regresar al pool.

---

# 197. UNKNOWN connection state

Default conservador:

```text
do not reuse
```

cuando no pueda verificarse.

---

# 198. Pool contamination

Una conexión rota no deberá contaminar futuras requests.

---

# 199. Persistent runtimes

Esto es especialmente crítico para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 200. Worker-local resilience state

Algunos componentes pueden ser worker-local:

```text
connection pools
endpoint health observations
circuit breakers
compiled failure classifiers
```

---

# 201. Request-local resilience state

Debe ser scoped:

```text
operation ID
attempt counter
deadline
cancellation
retry budget
transaction context
tenant
query context
resource budget
```

---

# 202. No static attempt counter

Prohibido:

```php
DatabaseRetry::$attempt++;
```

---

# 203. No request leakage

Un request no deberá heredar:

```text
retry count
transaction failure
cancellation
deadline
tenant
operation state
```

del request anterior.

---

# 204. Circuit breaker sharing

Un circuit breaker de endpoint sí puede ser compartido entre requests dentro del worker.

---

# 205. Distributed circuit state

Compartir circuit state entre múltiples workers/nodos será una integración opcional.

---

# 206. Local circuit limitations

Un circuit breaker local representa:

```text
this worker's evidence
```

no necesariamente el estado global del cluster.

---

# 207. Health evidence

Podrá tener:

```text
source
observedAt
confidence
generation
```

---

# 208. Stale health evidence

No deberá considerarse eternamente válida.

---

# 209. Health state

```php
enum DatabaseEndpointHealth
{
    case HEALTHY;
    case DEGRADED;
    case UNHEALTHY;
    case UNKNOWN;
}
```

---

# 210. UNKNOWN health ≠ HEALTHY

---

# 211. Health check success ≠ operation guarantee

Un ping exitoso no garantiza que la siguiente query funcione.

---

# 212. Health check failure ≠ transaction outcome

---

# 213. Resilience registry

Podrá centralizar políticas registradas.

```php
interface DatabaseResiliencePolicyRegistry
{
    public function resolve(
        DatabaseOperationType $operation
    ): DatabaseResiliencePolicy;
}
```

---

# 214. Policy immutability

Policies compiladas deberán ser immutable.

---

# 215. Policy precedence

Conceptualmente:

```text
Operation Override
        ↓
Connection Policy
        ↓
Database Policy
        ↓
Framework Default
```

---

# 216. No hidden unsafe override

Una policy local no deberá poder convertir:

```text
UNKNOWN COMMIT
```

en:

```text
safe retry
```

sin una estrategia de idempotencia/verification explícita.

---

# 217. Verification

Algunas operaciones pueden verificar outcome.

Ejemplo conceptual:

```text
INSERT with application-generated unique operation ID
connection lost
        ↓
query by operation ID
        ↓
determine whether insert exists
```

---

# 218. Verification ≠ universal recovery

Solo funciona si el dominio dispone de evidencia verificable.

---

# 219. OutcomeVerifier

```php
interface DatabaseOutcomeVerifier
{
    public function verify(
        DatabaseUnknownOutcome $outcome
    ): DatabaseOutcomeVerification;
}
```

---

# 220. Verification statuses

```text
CONFIRMED_SUCCESS
CONFIRMED_FAILURE
STILL_UNKNOWN
```

---

# 221. STILL_UNKNOWN

Debe preservarse.

---

# 222. Compensation

Algunos workflows pueden utilizar acciones compensatorias.

---

# 223. Compensation ≠ rollback

```text
Compensation
≠
Transaction Rollback
```

Una compensación es una nueva operación semántica.

---

# 224. Database core

No deberá inventar compensaciones de dominio.

---

# 225. Saga integration

Podrá existir en paquetes superiores.

No será responsabilidad central de Quantum Database.

---

# 226. Resilience and Events

Los eventos externos/side effects complican retries.

Ejemplo:

```text
DB write
→ email
→ transaction deadlock
→ retry
→ email sent twice
```

---

# 227. Outbox

Para side effects transaccionales:

```text
Outbox Pattern
```

será la integración recomendada cuando aplique.

---

# 228. Retry callback purity

Callbacks retryables deberían evitar side effects no idempotentes fuera de la transacción.

---

# 229. Transaction Retry rule

```text
Retryable Transaction Callback
+
External Non-idempotent Side Effect
=
Unsafe
```

salvo coordinación explícita.

---

# 230. Testing

La arquitectura deberá tener una suite específica de resiliencia.

---

# 231. Failure injection

Tests deberán poder simular:

```text
connection lost before send
connection lost after send
connection lost during receive
deadlock
serialization failure
timeout
cancellation
pool exhaustion
replica unavailable
writer unavailable
failover
unknown commit
```

---

# 232. Deterministic retry tests

Clock y jitter deberán poder sustituirse.

---

# 233. FakeClock

Permite probar backoff sin esperar tiempo real.

---

# 234. FakeRandom

Permite jitter reproducible.

---

# 235. Fake failure injector

```php
$injector->failOn(
    attempt: 2,
    with: FailureScenario::CONNECTION_LOST,
);
```

---

# 236. Unknown commit test

Debe existir explícitamente:

```text
COMMIT accepted by server
ACK lost
```

y comprobar:

```text
outcome = UNKNOWN
no blind retry
```

---

# 237. Retry exhaustion test

```text
attempts <= configured budget
```

siempre.

---

# 238. Cancellation test

Cancellation deberá impedir nuevos attempts.

---

# 239. Deadline test

Retries no deberán superar deadline.

---

# 240. Circuit breaker test

Validar:

```text
CLOSED
→ OPEN
→ HALF_OPEN
→ CLOSED
```

---

# 241. Resource exhaustion test

Validar que:

```text
bounded resource
```

no crece ilimitadamente.

---

# 242. Persistent runtime test

```text
Request A:
    failure
    retry count = 3

Request B:
    retry count = 0
```

---

# 243. Directory structure

```text
src/Quantum/Database/Resilience/
│
├── Contract/
│   ├── DatabaseFailureClassifier.php
│   ├── DatabaseRecoveryPolicy.php
│   ├── DatabaseOutcomeVerifier.php
│   ├── DatabaseCancellationToken.php
│   └── DatabaseResiliencePolicyRegistry.php
│
├── Failure/
│   ├── DatabaseFailure.php
│   ├── DatabaseFailureId.php
│   ├── DatabaseFailureDomain.php
│   ├── DatabaseFailureCategory.php
│   ├── DatabaseFailureScope.php
│   ├── DatabaseFailureSeverity.php
│   ├── DatabaseFailureCode.php
│   ├── DatabaseFailureEvidence.php
│   ├── DatabaseFailureObservation.php
│   └── DatabaseFailureNormalizer.php
│
├── Classifier/
│   ├── CompositeFailureClassifier.php
│   ├── MySqlFailureClassifier.php
│   ├── MariaDbFailureClassifier.php
│   ├── PostgreSqlFailureClassifier.php
│   └── SqliteFailureClassifier.php
│
├── Outcome/
│   ├── DatabaseOutcomeStatus.php
│   ├── DatabaseOperationOutcome.php
│   ├── DatabaseOutcomeEvidence.php
│   ├── DatabaseUnknownOutcome.php
│   └── DatabaseOutcomeVerification.php
│
├── Operation/
│   ├── DatabaseOperationId.php
│   ├── DatabaseAttemptId.php
│   ├── DatabaseOperationType.php
│   ├── DatabaseOperationPhase.php
│   ├── DatabaseOperationPriority.php
│   └── DatabaseResilienceContext.php
│
├── Policy/
│   ├── DatabaseResiliencePolicy.php
│   ├── DatabaseResilienceProfile.php
│   ├── DefaultResiliencePolicyRegistry.php
│   └── CompositeResiliencePolicy.php
│
├── Recovery/
│   ├── DatabaseRecoveryAction.php
│   ├── DatabaseRecoveryContext.php
│   ├── DatabaseRecoveryDecision.php
│   ├── DatabaseRecoveryCoordinator.php
│   └── DatabaseRecoveryExecutor.php
│
├── Retry/
│   ├── RetrySafety.php
│   ├── RetryBoundary.php
│   ├── RetryBudget.php
│   ├── RetryContext.php
│   ├── RetryDecision.php
│   ├── RetryCoordinator.php
│   ├── RetryAttempt.php
│   └── RetryBudgetTracker.php
│
├── Backoff/
│   ├── BackoffStrategy.php
│   ├── FixedBackoff.php
│   ├── LinearBackoff.php
│   ├── ExponentialBackoff.php
│   ├── ExponentialJitterBackoff.php
│   └── BackoffDelay.php
│
├── Timeout/
│   ├── DatabaseDeadline.php
│   ├── DatabaseTimeout.php
│   ├── DatabaseTimeoutType.php
│   └── DatabaseTimeoutPolicy.php
│
├── Cancellation/
│   ├── CancellationSource.php
│   ├── CancellationToken.php
│   └── DatabaseOperationCancelledException.php
│
├── CircuitBreaker/
│   ├── CircuitBreaker.php
│   ├── CircuitState.php
│   ├── CircuitBreakerPolicy.php
│   ├── CircuitBreakerRegistry.php
│   ├── CircuitFailureWindow.php
│   └── CircuitProbe.php
│
├── Health/
│   ├── DatabaseEndpointHealth.php
│   ├── DatabaseHealthEvidence.php
│   ├── DatabaseHealthObservation.php
│   └── DatabaseEndpointHealthRegistry.php
│
├── Failover/
│   ├── DatabaseFailoverCoordinator.php
│   ├── DatabaseFailoverContext.php
│   ├── DatabaseFailoverDecision.php
│   ├── DatabaseAuthorityEpoch.php
│   └── DatabaseTopologyGeneration.php
│
├── Resource/
│   ├── DatabaseResourceBudget.php
│   ├── DatabaseResourceGovernor.php
│   ├── DatabaseBackpressurePolicy.php
│   ├── DatabaseLoadSheddingPolicy.php
│   └── DatabaseBulkhead.php
│
├── Fallback/
│   ├── DatabaseFallbackPolicy.php
│   ├── DatabaseFallbackDecision.php
│   └── DatabaseFallbackCoordinator.php
│
├── Event/
│   ├── DatabaseFailureDetected.php
│   ├── DatabaseRetryScheduled.php
│   ├── DatabaseRetryExhausted.php
│   ├── DatabaseCircuitOpened.php
│   ├── DatabaseFailoverStarted.php
│   └── DatabaseRecoveryCompleted.php
│
├── Telemetry/
│   ├── DatabaseResilienceTelemetry.php
│   ├── DatabaseRetryTelemetry.php
│   └── DatabaseFailureTelemetry.php
│
├── Testing/
│   ├── DatabaseFailureInjector.php
│   ├── FailureScenario.php
│   ├── FakeDatabaseFailureClassifier.php
│   ├── FakeBackoffStrategy.php
│   ├── FakeCircuitBreaker.php
│   └── DatabaseResilienceAssertions.php
│
└── Exception/
    ├── DatabaseResilienceException.php
    ├── DatabaseTransientFailureException.php
    ├── DatabasePermanentFailureException.php
    ├── DatabaseUnknownOutcomeException.php
    ├── DatabaseRetryExhaustedException.php
    ├── DatabaseCircuitOpenException.php
    ├── DatabaseFailoverException.php
    ├── DatabaseRecoveryException.php
    ├── DatabaseResourceExhaustedException.php
    └── DatabaseDeadlineExceededException.php
```

---

# 244. Flujo de read resiliente

```text
READ
 │
 ▼
Check Cancellation
 │
 ▼
Check Deadline
 │
 ▼
Check Circuit
 │
 ▼
Acquire Connection
 │
 ▼
Execute
 │
 ├──────── SUCCESS ───────► Result
 │
 ▼
Failure
 │
 ▼
Classify
 │
 ▼
Determine Outcome
 │
 ▼
Retry Safety
 │
 ├── UNSAFE/UNKNOWN ──────► Propagate
 │
 └── SAFE
       │
       ▼
  Retry Budget?
       │
       ├── NO ────────────► Exhausted
       │
       ▼
    Backoff
       │
       ▼
 Connection/Replica Policy
       │
       ▼
     Retry
```

---

# 245. Flujo de write resiliente

```text
WRITE
 │
 ▼
Execute
 │
 ├──────── Confirmed Success ───► Success
 │
 ▼
Failure
 │
 ▼
Was write possibly sent?
 │
 ├── NO
 │    │
 │    ▼
 │  Analyze Retry Safety
 │
 └── YES
      │
      ▼
 Determine Server Outcome
      │
      ├── Confirmed Failure
      │       │
      │       ▼
      │   Analyze Retry
      │
      ├── Confirmed Success
      │       │
      │       ▼
      │      Success
      │
      └── UNKNOWN
              │
              ▼
       Outcome Verification?
              │
        ┌─────┴─────┐
        │           │
       YES          NO
        │           │
        ▼           ▼
    Verify       Abort/
        │        Propagate
        ▼
 Success / Failure / UNKNOWN
```

---

# 246. Flujo de transaction retry

```text
Transaction Callback
        │
        ▼
BEGIN
        │
        ▼
Statements
        │
        ▼
Failure
        │
        ▼
Classify
        │
        ▼
Transaction invalid?
        │
        ├── NO ──► normal handling
        │
        └── YES
             │
             ▼
       Rollback if possible
             │
             ▼
       Outcome known?
             │
        ┌────┴────┐
       YES        NO
        │          │
        ▼          ▼
 Retry-safe?     UNKNOWN
        │          │
        │          └────► Abort
        ▼
 Retry whole callback
```

---

# 247. Flujo de failover

```text
Endpoint Failure
      │
      ▼
Classify Scope
      │
      ▼
Endpoint Unhealthy?
      │
      ▼
Mark/Observe Health
      │
      ▼
Active Transaction?
      │
 ┌────┴────┐
YES        NO
 │          │
 ▼          ▼
Abort     Failover Eligible?
Txn          │
             ▼
       Resolve Topology
             │
             ▼
       Verify Authority
             │
             ▼
       Select Endpoint
             │
             ▼
       Re-evaluate Consistency
             │
             ▼
       Retry if Safe
```

---

# 248. Invariantes arquitectónicas

## DB-RES-001
Failure será distinto de Retry.

## DB-RES-002
Resilience será más amplia que Retry.

## DB-RES-003
Resilience será distinta de High Availability.

## DB-RES-004
Resilience será distinta de error suppression.

## DB-RES-005
UNKNOWN será outcome de primera clase.

## DB-RES-006
UNKNOWN no se convertirá automáticamente en FAILURE.

## DB-RES-007
UNKNOWN no se convertirá automáticamente en SUCCESS.

## DB-RES-008
ConnectionLost no implicará OperationFailed.

## DB-RES-009
Failure classification será semántica.

## DB-RES-010
Exception class será distinta de failure domain.

## DB-RES-011
Raw driver evidence será preservada cuando sea segura.

## DB-RES-012
Error strings no serán mecanismo primario de clasificación.

## DB-RES-013
SQLSTATE/vendor codes podrán alimentar clasificación.

## DB-RES-014
MySQL y MariaDB podrán tener clasificadores separados.

## DB-RES-015
Resilience será capability-driven.

## DB-RES-016
Operation phase será preservada.

## DB-RES-017
Failure during COMMIT tendrá tratamiento especial.

## DB-RES-018
Statement outcome será distinto de transaction outcome.

## DB-RES-019
Transaction outcome será distinto de ORM state.

## DB-RES-020
Detector será distinto de policy.

## DB-RES-021
Classifier será distinto de recovery coordinator.

## DB-RES-022
Transient failure no implicará retry automático.

## DB-RES-023
Permanent failure no será reintentado por default.

## DB-RES-024
Capacity failure será distinguido.

## DB-RES-025
Security failure será distinguido.

## DB-RES-026
Concurrency failure será distinguido.

## DB-RES-027
Failure scope será explícito.

## DB-RES-028
Retry safety será explícita.

## DB-RES-029
UNKNOWN retry safety implicará no retry por default.

## DB-RES-030
Retryable failure será distinto de retryable operation.

## DB-RES-031
Transient network failure no hará seguro un write incierto.

## DB-RES-032
Idempotency será distinta de retry policy.

## DB-RES-033
Database core no inventará domain idempotency.

## DB-RES-034
SELECT no será automáticamente retryable.

## DB-RES-035
Write retries serán conservadores.

## DB-RES-036
Transaction-invalidating failures requerirán transaction-level retry.

## DB-RES-037
Statement retry será distinto de transaction retry.

## DB-RES-038
Retry boundary será explícito.

## DB-RES-039
Logical operation tendrá identidad estable.

## DB-RES-040
Cada attempt tendrá identidad independiente.

## DB-RES-041
Retries estarán acotados.

## DB-RES-042
Infinite retry estará prohibido.

## DB-RES-043
Backoff será distinto de retry safety.

## DB-RES-044
Jitter podrá prevenir retry storms.

## DB-RES-045
Retry respetará deadline.

## DB-RES-046
Retry respetará cancellation.

## DB-RES-047
Timeout será distinto de cancellation.

## DB-RES-048
Client timeout no implicará database rollback.

## DB-RES-049
Cancellation request será distinta de cancellation confirmed.

## DB-RES-050
Platform cancellation será capability-driven.

## DB-RES-051
Circuit breaker será distinto de health check.

## DB-RES-052
Circuit breaker será distinto de failover.

## DB-RES-053
Circuit breaker será distinto de retry.

## DB-RES-054
No todo failure contribuirá al circuit.

## DB-RES-055
Application/data errors no abrirán circuit por default.

## DB-RES-056
Circuit scope será bounded.

## DB-RES-057
Failover será distinto de retry.

## DB-RES-058
Failover será distinto de reconnect.

## DB-RES-059
Mid-transaction transparent failover estará prohibido por default.

## DB-RES-060
Transaction permanecerá pinned.

## DB-RES-061
Writer failover no continuará una transaction anterior.

## DB-RES-062
Split-brain uncertainty impedirá blind writes.

## DB-RES-063
Topology generation será explícita.

## DB-RES-064
Authority epoch podrá invalidar stale ownership.

## DB-RES-065
Replica failover respetará consistency.

## DB-RES-066
Replica switch no garantizará same snapshot.

## DB-RES-067
Shard failure no será ignorado silenciosamente.

## DB-RES-068
Partial distributed result no será complete result.

## DB-RES-069
Recovery será distinta de retry.

## DB-RES-070
Reconnect será distinto de same session.

## DB-RES-071
Session state no se asumirá preservado tras reconnect.

## DB-RES-072
Temporary tables no se asumirán preservadas.

## DB-RES-073
Session locks no se asumirán preservados.

## DB-RES-074
Prepared statements podrán requerir reprepare.

## DB-RES-075
Broken transaction no será reparada transparentemente.

## DB-RES-076
TAINTED será estado explícito.

## DB-RES-077
Unknown persistence podrá taint EntityManager.

## DB-RES-078
Recovery no fabricará ORM truth.

## DB-RES-079
Resource exhaustion será failure category especial.

## DB-RES-080
Connection usage será bounded.

## DB-RES-081
Memory usage será gobernable.

## DB-RES-082
Buffers serán bounded.

## DB-RES-083
Pool saturation será distinta de endpoint failure.

## DB-RES-084
Retry amplification será limitada.

## DB-RES-085
Backpressure será soportada.

## DB-RES-086
Load shedding será explícito.

## DB-RES-087
Priority será distinta de permission.

## DB-RES-088
Graceful degradation requerirá semantic contract.

## DB-RES-089
Database failure no devolverá empty result silenciosamente.

## DB-RES-090
Stale cache será distinto de fresh data.

## DB-RES-091
Cache fallback requerirá policy explícita.

## DB-RES-092
Fallback respetará tenant isolation.

## DB-RES-093
Fallback respetará authorization.

## DB-RES-094
Fallback respetará transaction semantics.

## DB-RES-095
ResilienceContext será scoped.

## DB-RES-096
Resilience profile no ocultará cambios semánticos.

## DB-RES-097
Migration retry será conservador.

## DB-RES-098
DDL unknown outcome no será blind retry.

## DB-RES-099
Import resume requerirá checkpoint/idempotency apropiados.

## DB-RES-100
Export resume dependerá de formato y destino.

## DB-RES-101
Chunk retry requerirá replay safety.

## DB-RES-102
Checkpoint no implicará exactly-once.

## DB-RES-103
Lazy iteration failure liberará recursos.

## DB-RES-104
Streaming connection failure no implicará resumability.

## DB-RES-105
Result cursor perdido no será automáticamente resumible.

## DB-RES-106
Events no serán control plane implícito.

## DB-RES-107
Telemetry listener failure no redefinirá DB outcome.

## DB-RES-108
afterCommit failure no convertirá commit en rollback.

## DB-RES-109
Resilience tendrá bounded telemetry.

## DB-RES-110
Retries conservarán trace context.

## DB-RES-111
Attempts serán observables separadamente.

## DB-RES-112
UNKNOWN outcome será observable explícitamente.

## DB-RES-113
Logs no expondrán sensitive bindings.

## DB-RES-114
Audit será distinto de telemetry.

## DB-RES-115
Middleware ordering será determinista.

## DB-RES-116
Retry no bypassará circuit breaker.

## DB-RES-117
Retry no reiniciará resource budget automáticamente.

## DB-RES-118
Retry no reiniciará deadline.

## DB-RES-119
Cancellation detendrá attempts futuros.

## DB-RES-120
Retry preservará security context.

## DB-RES-121
Retry preservará tenant context.

## DB-RES-122
Retry no cambiará tenant.

## DB-RES-123
Shard-bound operation no cambiará shard arbitrariamente.

## DB-RES-124
Replica retry respetará read consistency.

## DB-RES-125
Writer retry requerirá verified authority.

## DB-RES-126
Broken connection no regresará al pool.

## DB-RES-127
UNKNOWN connection state no se reutilizará por default.

## DB-RES-128
Pool contamination será evitada.

## DB-RES-129
Request retry state no será global.

## DB-RES-130
Attempt counters no serán static.

## DB-RES-131
Request state será limpiado en persistent runtimes.

## DB-RES-132
Circuit state podrá ser worker-local.

## DB-RES-133
Local circuit state no se fingirá global.

## DB-RES-134
Health evidence tendrá freshness.

## DB-RES-135
UNKNOWN health será distinto de HEALTHY.

## DB-RES-136
Successful health check no garantizará successful query.

## DB-RES-137
Health failure no determinará transaction outcome.

## DB-RES-138
Resilience policies serán immutable.

## DB-RES-139
Unsafe local override no podrá ocultar UNKNOWN commit.

## DB-RES-140
Outcome verification será explícita.

## DB-RES-141
Outcome verification podrá permanecer UNKNOWN.

## DB-RES-142
Compensation será distinta de rollback.

## DB-RES-143
Database core no inventará domain compensation.

## DB-RES-144
Saga no será responsabilidad central de Database.

## DB-RES-145
External side effects serán considerados al determinar retry safety.

## DB-RES-146
Outbox será integración recomendada para side effects transaccionales.

## DB-RES-147
Retry callback con side effects no idempotentes podrá ser UNSAFE.

## DB-RES-148
Failure injection será soportada en testing.

## DB-RES-149
Backoff tests serán deterministas.

## DB-RES-150
Jitter tests serán reproducibles.

## DB-RES-151
Unknown commit tendrá pruebas explícitas.

## DB-RES-152
Unknown commit no tendrá blind retry.

## DB-RES-153
Retry exhaustion será verificable.

## DB-RES-154
Cancellation será verificable.

## DB-RES-155
Deadlines serán verificables.

## DB-RES-156
Circuit transitions serán verificables.

## DB-RES-157
Resource exhaustion será verificable.

## DB-RES-158
Persistent runtime isolation será verificable.

## DB-RES-159
Resilience no generará SQL.

## DB-RES-160
Resilience no interpretará entities.

## DB-RES-161
Compiler no decidirá retry.

## DB-RES-162
Driver no decidirá transaction replay.

## DB-RES-163
Connection no decidirá ORM recovery.

## DB-RES-164
ORM no decidirá endpoint failover.

## DB-RES-165
Topology no decidirá domain idempotency.

## DB-RES-166
Retry no fabricará exactly-once.

## DB-RES-167
Failover no fabricará distributed atomicity.

## DB-RES-168
Reconnect no fabricará session continuity.

## DB-RES-169
Recovery no fabricará confirmed outcome.

## DB-RES-170
Cuando la evidencia sea insuficiente, VoltStack preservará UNKNOWN.

---

# 249. Modelo formal

Sea una operación lógica:

```text
O
```

con attempts:

```text
A₁, A₂, ..., Aₙ
```

y un fallo:

```text
Fᵢ
```

en el attempt `Aᵢ`.

Definimos:

```text
Classify(Fᵢ)
=
(Domain, Category, Scope)
```

y:

```text
Outcome(Aᵢ)
∈
{SUCCESS, FAILURE, UNKNOWN}
```

La función de retry será:

```text
RetryAllowed(O, Aᵢ)
=
RetrySafety(O, Aᵢ) = SAFE
∧
BudgetRemaining(O)
∧
¬Cancelled(O)
∧
DeadlineRemaining(O)
∧
PolicyAllows(O)
```

---

# 250. Regla formal de UNKNOWN

Si:

```text
Outcome(Aᵢ) = UNKNOWN
```

entonces:

```text
RetryAllowed(O, Aᵢ) = false
```

por default.

Solo podrá cambiar cuando exista una estrategia explícita capaz de demostrar seguridad, por ejemplo:

```text
OutcomeVerification
IdempotencyProtocol
Application-defined Recovery Contract
```

---

# 251. Regla formal de transaction retry

Sea:

```text
T = transaction callback
```

Si un fallo invalida `T`:

```text
Retry(statement)
=
false
```

y:

```text
Retry(T)
=
RetrySafe(T)
∧
RollbackConfirmed
∧
BudgetRemaining
```

salvo semántica específica demostrable.

---

# 252. Regla formal de failover

Sea:

```text
E₁ = current endpoint
E₂ = candidate endpoint
```

Failover será válido únicamente si:

```text
Eligible(E₂)
∧
AuthorityValid(E₂)
∧
ConsistencySatisfied(E₂)
∧
TopologyCurrent(E₂)
∧
OperationCanMove(O)
```

---

# 253. Regla formal de recursos

Sea:

```text
B = ResourceBudget
U = CurrentUsage
```

La operación podrá continuar si:

```text
U ≤ B
```

Cuando:

```text
U > B
```

deberá aplicarse una acción explícita:

```text
WAIT
REJECT
CANCEL
SHED
```

nunca crecimiento ilimitado por default.

---

# 254. Modelo final

La arquitectura completa puede resumirse:

```text
                 DATABASE OPERATION
                         │
                         ▼
                Resilience Context
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
    Deadline         Cancellation     Resource Budget
        │                │                │
        └────────────────┼────────────────┘
                         ▼
                  Circuit Breaker
                         │
                         ▼
                 Connection/Route
                         │
                         ▼
                     Execute
                         │
              ┌──────────┴──────────┐
              │                     │
           SUCCESS                FAILURE
              │                     │
              ▼                     ▼
            Result          Failure Detection
                                    │
                                    ▼
                           Failure Classification
                                    │
                                    ▼
                            Outcome Analysis
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
                 SUCCESS         FAILURE         UNKNOWN
                    │               │               │
                    ▼               ▼               ▼
                  Result       Retry Safety     Verification?
                                    │               │
                                    ▼               ▼
                             Recovery Policy    Preserve
                                    │            UNKNOWN
                       ┌────────────┼─────────────┐
                       │            │             │
                     Retry      Reconnect      Failover
                       │            │             │
                       └────────────┼─────────────┘
                                    ▼
                              New Attempt
```

---

# 255. Regla maestra final

La regla maestra del sistema será:

> **VoltStack Database deberá diferenciar siempre entre detectar un fallo, conocer el resultado de una operación y demostrar que dicha operación puede repetirse. Ningún error transitorio, timeout, pérdida de conexión, failover o reconnect constituirá por sí mismo evidencia suficiente para repetir una operación cuyo resultado o seguridad de replay sean inciertos.**

En forma compacta:

```text
Failure
≠
Outcome

Outcome
≠
Retry Safety

Retry Safety
≠
Retry Policy

Retry
≠
Recovery

Recovery
≠
Failover

Failover
≠
Reconnect

Timeout
≠
Cancellation

Connection Lost
≠
Operation Failed

Statement Success
≠
Transaction Commit

Rollback
≠
Object Graph Rewind

Checkpoint
≠
Exactly Once

Reconnect
≠
Session Continuity

Failover
≠
Distributed Atomicity

UNKNOWN
≠
FAILURE

UNKNOWN
≠
SUCCESS
```

La propiedad final buscada será:

```text
Database Failure
      ↓
Evidence
      ↓
Classification
      ↓
Known / Unknown Outcome
      ↓
Safety Analysis
      ↓
Bounded Recovery
      ↓
Observable Result
```

con el principio:

```text
Correctness
>
Automatic Recovery
```

---

# 256. Estado del Bloque 23

```text
BLOCK 23 — RESILIENCE

✓ 235_DATABASE_RESILIENCE_ARCHITECTURE.md
○ 236_DATABASE_CONNECTION_FAILURE_HANDLING_SYSTEM.md
○ 237_DATABASE_QUERY_FAILURE_HANDLING_SYSTEM.md
○ 238_DATABASE_RETRY_POLICY_SYSTEM.md
○ 239_DATABASE_CIRCUIT_BREAKER_INTEGRATION_SYSTEM.md
○ 240_DATABASE_FAILOVER_AND_RECOVERY_SYSTEM.md
○ 241_DATABASE_RESOURCE_EXHAUSTION_PROTECTION_SYSTEM.md
```

---

# 257. Siguiente documento

```text
236_DATABASE_CONNECTION_FAILURE_HANDLING_SYSTEM.md
```

El siguiente documento definirá específicamente cómo VoltStack deberá detectar, clasificar y manejar fallos asociados con:

```text
connection establishment
DNS
TCP/socket
TLS
authentication
handshake
connection acquisition
connection loss
broken connections
stale connections
idle connections
half-open connections
connection reset
server shutdown
network partitions
pool contamination
connection validation
reconnection
```

manteniendo como regla fundamental:

> **Una conexión puede recuperarse o reemplazarse; una operación cuyo resultado quedó incierto por la pérdida de esa conexión no puede declararse fallida ni repetirse automáticamente solo porque exista una nueva conexión.**