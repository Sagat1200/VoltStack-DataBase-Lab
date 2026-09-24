# 241_DATABASE_RESOURCE_EXHAUSTION_PROTECTION_SYSTEM.md

# VoltStack Quantum Database
## Database Resource Exhaustion Protection System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 241 — Database Resource Exhaustion Protection System  
**Bloque:** 23 — Resilience  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `240_DATABASE_FAILOVER_AND_RECOVERY_SYSTEM.md`  
**Siguiente documento:** `242_DATABASE_PERFORMANCE_ARCHITECTURE.md`

---

# 1. Propósito

Este documento define la arquitectura de **protección contra agotamiento de recursos** de VoltStack Database.

Su objetivo es impedir que una carga excesiva, una consulta costosa, una operación masiva, una fuga de recursos, un retry storm o una combinación de operaciones válidas terminen agotando los recursos de:

- la aplicación;
- el worker;
- el runtime;
- el Connection Pool;
- el servidor de base de datos;
- una réplica;
- un shard;
- un tenant;
- un proceso de importación/exportación;
- el ORM;
- el Query Engine;
- el sistema completo.

La regla central será:

> **VoltStack no deberá aceptar trabajo ilimitado esperando que la base de datos o el runtime sobrevivan; cada recurso finito deberá poseer límites, ownership, accounting y políticas de admisión explícitas, de modo que ante saturación el sistema degrade, difiera o rechace trabajo de forma controlada antes de convertir presión local en una falla generalizada.**

---

# 2. Problema fundamental

Una base de datos puede estar técnicamente saludable y aun así ser incapaz de aceptar más trabajo.

Ejemplo:

```text
Database
   │
   ├── connections: 100 / 100
   ├── running queries: 250
   ├── queued queries: 2,000
   ├── memory: critical
   └── latency: rapidly increasing
```

Aceptar otras 5,000 operaciones no aumenta necesariamente throughput.

Puede provocar:

```text
More Work
   ↓
More Contention
   ↓
Higher Latency
   ↓
More Timeouts
   ↓
More Retries
   ↓
More Work
   ↓
RESOURCE COLLAPSE
```

Este fenómeno deberá evitarse arquitectónicamente.

---

# 3. Regla de estabilidad

En términos generales:

```text
Incoming Work Rate > Sustainable Processing Rate
```

durante suficiente tiempo implica:

```text
Queue Growth
    ↓
Resource Saturation
    ↓
Latency Growth
    ↓
Timeouts
    ↓
Failure Amplification
```

VoltStack deberá intervenir antes del colapso.

---

# 4. Distinciones fundamentales

```text
Resource Exhaustion Protection
≠
Performance Optimization

Resource Exhaustion Protection
≠
Rate Limiting

Resource Exhaustion Protection
≠
Connection Pooling

Resource Exhaustion Protection
≠
Query Timeout

Resource Exhaustion Protection
≠
Circuit Breaker

Resource Exhaustion Protection
≠
Retry Policy

Resource Exhaustion Protection
≠
Load Balancing

Resource Exhaustion Protection
≠
Operating System OOM Protection
```

Todos pueden colaborar, pero representan responsabilidades distintas.

---

# 5. Performance vs protection

Performance pregunta:

> ¿Cómo hacemos una operación más eficiente?

Resource Protection pregunta:

> ¿Cuánto trabajo podemos permitir simultáneamente sin comprometer el sistema?

---

# 6. Filosofía

VoltStack utilizará:

```text
Bounded Resources
+
Admission Control
+
Resource Budgets
+
Resource Accounting
+
Backpressure
+
Bulkhead Isolation
+
Controlled Rejection
+
Telemetry
```

en lugar de:

```text
Accept Everything
+
Hope
```

---

# 7. Arquitectura general

```text
                DATABASE OPERATION
                       │
                       ▼
                Operation Context
                       │
                       ▼
               Resource Estimator
                       │
                       ▼
                 Resource Budget
                       │
                       ▼
                Admission Control
                       │
          ┌────────────┼─────────────┐
          │            │             │
        ADMIT         WAIT         REJECT
          │            │
          │       bounded queue
          │            │
          └──────┬─────┘
                 ▼
            Acquire Permit
                 │
                 ▼
          Resource Accounting
                 │
                 ▼
              Execute
                 │
                 ▼
          Runtime Monitoring
                 │
         ┌───────┼────────┐
         │       │        │
       NORMAL   SOFT     HARD
                LIMIT    LIMIT
                  │        │
                  ▼        ▼
            Backpressure  Abort /
                         Reject
                 │
                 ▼
           Release Permit
                 │
                 ▼
          Resource Accounting
```

---

# 8. Resource domains

VoltStack deberá modelar al menos:

```php
enum DatabaseResourceType
{
    case CONNECTION;
    case QUERY_SLOT;
    case TRANSACTION;
    case MEMORY;
    case RESULT_BUFFER;
    case RESULT_CURSOR;
    case STREAM;
    case PREPARED_STATEMENT;
    case HYDRATION;
    case IDENTITY_MAP;
    case UNIT_OF_WORK;
    case BATCH;
    case CHUNK;
    case BULK_OPERATION;
    case IMPORT;
    case EXPORT;
    case DISTRIBUTED_FANOUT;
    case RETRY;
    case FAILOVER;
    case TEMPORARY_STORAGE;
    case WORKER_CAPACITY;
}
```

---

# 9. Resource Scope

Un límite sin scope es ambiguo.

VoltStack distinguirá:

```php
enum ResourceScope
{
    case OPERATION;
    case REQUEST;
    case TRANSACTION;
    case WORKER;
    case PROCESS;
    case CONNECTION_POOL;
    case LOGICAL_DATABASE;
    case ENDPOINT;
    case REPLICA;
    case SHARD;
    case TENANT;
    case APPLICATION;
}
```

---

# 10. ResourceBudget

El concepto central será:

```php
final readonly class ResourceBudget
{
    public function __construct(
        public ResourceScope $scope,
        public ResourceLimits $limits,
        public ResourcePolicy $policy,
    ) {}
}
```

---

# 11. ResourceLimits

```php
final readonly class ResourceLimits
{
    public function __construct(
        public array $limits,
    ) {}
}
```

Ejemplo conceptual:

```php
[
    DatabaseResourceType::CONNECTION->name => 50,
    DatabaseResourceType::QUERY_SLOT->name => 100,
    DatabaseResourceType::RESULT_CURSOR->name => 20,
]
```

---

# 12. Límites jerárquicos

Una operación puede estar sometida simultáneamente a:

```text
Application Budget
       ↓
Database Budget
       ↓
Tenant Budget
       ↓
Worker Budget
       ↓
Request Budget
       ↓
Operation Budget
```

Debe cumplir todos.

---

# 13. Effective budget

Conceptualmente:

```text
EffectiveLimit(resource)
=
min(
    ApplicationLimit,
    DatabaseLimit,
    TenantLimit,
    WorkerLimit,
    RequestLimit,
    OperationLimit
)
```

cuando dichos límites sean comparables.

---

# 14. Budget inheritance

Un scope hijo no deberá aumentar silenciosamente un límite impuesto por su padre.

```text
Parent hard limit = 100
Child requested limit = 200
```

Resultado efectivo:

```text
100
```

salvo un mecanismo administrativo explícito.

---

# 15. Thresholds

Cada recurso podrá manejar:

```php
final readonly class ResourceThresholds
{
    public function __construct(
        public int|float|null $warning,
        public int|float|null $soft,
        public int|float|null $hard,
        public int|float|null $critical,
    ) {}
}
```

---

# 16. WARNING

Indica presión creciente.

No necesariamente cambia ejecución.

---

# 17. SOFT LIMIT

Puede activar:

```text
queueing
backpressure
reduced concurrency
warnings
lower priority rejection
```

---

# 18. HARD LIMIT

No se admitirán nuevas operaciones que excedan el límite.

---

# 19. CRITICAL

Puede activar mecanismos defensivos adicionales:

```text
load shedding
emergency rejection
resource quarantine
optional cancellation
```

según policy.

---

# 20. Hard limit ≠ crash threshold

El límite duro deberá colocarse antes del límite físico real.

Ejemplo:

```text
DB max_connections = 500

VoltStack usable ceiling = 420
```

dejando margen para:

```text
administration
health checks
recovery
migrations
emergency operations
```

---

# 21. Safety reserve

```php
final readonly class ResourceReserve
{
    public function __construct(
        public DatabaseResourceType $resource,
        public int|float $reserved,
    ) {}
}
```

---

# 22. Capacity ≠ Usable Capacity

```text
UsableCapacity
=
PhysicalCapacity
-
SafetyReserve
```

---

# 23. Resource accounting

VoltStack deberá saber conceptualmente:

```text
allocated
reserved
active
waiting
released
unknown
```

para los recursos administrados.

---

# 24. ResourceUsage

```php
final readonly class ResourceUsage
{
    public function __construct(
        public DatabaseResourceType $resource,
        public ResourceScope $scope,
        public int|float $used,
        public int|float $reserved,
        public int|float $limit,
    ) {}
}
```

---

# 25. Resource accounting ≠ physical measurement

VoltStack puede contabilizar:

```text
connections leased by VoltStack
```

sin conocer exactamente toda la memoria consumida por PostgreSQL/MySQL.

La diferencia deberá permanecer explícita.

---

# 26. Measurement confidence

```php
enum ResourceMeasurementConfidence
{
    case EXACT;
    case ESTIMATED;
    case OBSERVED;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 27. UNKNOWN usage

```text
UNKNOWN
≠
ZERO
```

---

# 28. Admission Control

Antes de ejecutar trabajo costoso:

```text
Can this operation enter the system?
```

---

# 29. AdmissionDecision

```php
enum AdmissionDecisionType
{
    case ADMIT;
    case WAIT;
    case THROTTLE;
    case REJECT;
}
```

---

# 30. AdmissionResult

```php
final readonly class AdmissionResult
{
    public function __construct(
        public AdmissionDecisionType $decision,
        public ?Duration $maxWait,
        public string $reason,
    ) {}
}
```

---

# 31. Admission controller

```php
interface DatabaseAdmissionController
{
    public function evaluate(
        DatabaseWorkRequest $request,
        ResourceSnapshot $resources,
    ): AdmissionResult;
}
```

---

# 32. DatabaseWorkRequest

Representará características semánticas de la operación:

```php
final readonly class DatabaseWorkRequest
{
    public function __construct(
        public DatabaseOperationType $type,
        public WorkPriority $priority,
        public ResourceEstimate $estimate,
        public DatabaseContext $context,
    ) {}
}
```

---

# 33. Admission ≠ authorization

Una operación puede estar:

```text
AUTHORIZED
```

pero ser:

```text
RESOURCE_REJECTED
```

---

# 34. Admission ≠ validation

Una consulta válida puede ser rechazada por saturación.

---

# 35. Resource Permit

Una operación admitida obtiene conceptualmente:

```php
interface ResourcePermit
{
    public function allocation(): ResourceAllocation;

    public function release(): void;
}
```

---

# 36. Permit ownership

Cada permit tendrá un owner definido.

```text
Request
Operation
Transaction
Stream
Chunk Runner
Import Job
```

---

# 37. Permit lifecycle

```text
REQUESTED
   ↓
GRANTED
   ↓
ACTIVE
   ↓
RELEASED
```

Con ramas:

```text
REQUESTED → REJECTED
ACTIVE → EXPIRED
ACTIVE → LEAK_SUSPECTED
```

---

# 38. Resource leak

Un recurso adquirido y nunca liberado puede ser más peligroso que una consulta lenta.

VoltStack deberá detectar cuando sea posible:

```text
connection lease leak
cursor leak
stream leak
transaction leak
permit leak
```

---

# 39. Destructor ≠ correctness mechanism

La liberación deberá ser explícita mediante:

```text
try/finally
scope lifecycle
RAII-like abstraction
```

Los destructores serán solo defensa adicional.

---

# 40. Connection Budget

Las conexiones serán uno de los recursos más estrictamente administrados.

---

# 41. Connection capacity

Ejemplo:

```text
Application
├── Worker 1
├── Worker 2
├── Worker 3
...
└── Worker 100
```

Si cada worker permite:

```text
20 connections
```

podrían existir:

```text
2,000 connections
```

aunque la DB solo soporte 500.

---

# 42. Worker-local pool problem

El tamaño del pool por worker no puede diseñarse ignorando:

```text
number of workers
database max connections
other applications
admin reserve
replicas
tenants
```

---

# 43. Connection capacity model

Conceptualmente:

```text
GlobalConnectionBudget
≥
Σ WorkerConnectionBudgets
```

deberá mantenerse razonablemente dentro de la capacidad asignada a VoltStack.

---

# 44. Persistent runtime implications

Esto será especialmente importante para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

porque los workers pueden mantener conexiones durante mucho tiempo.

---

# 45. Connection admission

Si el pool está lleno:

```text
acquire connection
      ↓
available?
 ┌────┴────┐
 yes       no
 │         │
lease    queue?
           │
       ┌───┴───┐
      yes      no
       │        │
 bounded     reject
 wait
```

---

# 46. Connection acquisition timeout

Nunca se esperará indefinidamente.

```php
final readonly class ConnectionAcquisitionPolicy
{
    public function __construct(
        public Duration $maxWait,
        public int $maxQueueDepth,
    ) {}
}
```

---

# 47. Pool exhaustion

Deberá generar un error específico:

```text
ConnectionPoolExhaustedException
```

no una excepción genérica de conexión.

---

# 48. Reserved connections

Podrán reservarse conexiones para:

```text
health
administration
recovery
high-priority control operations
```

---

# 49. Query concurrency budget

El hecho de disponer de 100 conexiones no significa que deban ejecutarse 100 consultas pesadas simultáneamente.

Por ello:

```text
Connection Limit
≠
Query Concurrency Limit
```

---

# 50. Query slots

VoltStack podrá manejar:

```text
QueryExecutionPermit
```

separado del Connection Permit.

---

# 51. Query concurrency limiter

```php
interface QueryConcurrencyLimiter
{
    public function acquire(
        QueryResourceRequest $request
    ): QueryExecutionPermit;
}
```

---

# 52. Lightweight vs heavyweight queries

Cuando exista evidencia suficiente, podrán existir clases:

```php
enum QueryWorkClass
{
    case LIGHT;
    case NORMAL;
    case HEAVY;
    case BULK;
    case STREAMING;
    case ADMINISTRATIVE;
    case UNKNOWN;
}
```

---

# 53. UNKNOWN work class

No deberá considerarse automáticamente:

```text
LIGHT
```

---

# 54. Query cost estimate

Puede utilizar:

```text
query metadata
planner information
historical telemetry
explicit hints
operation type
expected rows
fan-out
```

sin convertir estimaciones en garantías.

---

# 55. Cost estimate ≠ actual cost

```text
EstimatedCost
≠
ObservedCost
```

---

# 56. Transaction budget

Las transactions mantienen recursos por tiempo.

VoltStack limitará:

```text
concurrent transactions
transaction duration
transaction idle time
connections held by transactions
locks held indirectly
```

---

# 57. Long transaction risk

```text
BEGIN
   ↓
idle 5 minutes
   ↓
UPDATE
   ↓
idle
   ↓
COMMIT
```

puede mantener:

```text
connection
locks
MVCC state
replication pressure
```

durante demasiado tiempo.

---

# 58. Transaction duration budget

```php
final readonly class TransactionResourceBudget
{
    public function __construct(
        public Duration $maxDuration,
        public Duration $maxIdleDuration,
    ) {}
}
```

---

# 59. Transaction timeout ≠ query timeout

Ambos deberán ser independientes.

---

# 60. Existing transaction ownership

Resource Protection nunca hará:

```text
commit caller transaction
```

para liberar recursos.

---

# 61. Result Buffer Budget

Una consulta puede devolver:

```text
10,000,000 rows
```

aunque sea rápida en la DB.

Materializar todo puede destruir la memoria del worker.

---

# 62. Result materialization protection

Operaciones como:

```php
$query->get();
$query->all();
$lazy->collect();
$lazy->toArray();
```

podrán estar sujetas a:

```text
max rows
max estimated bytes
max materialization duration
```

---

# 63. Unknown cardinality

```text
Unknown Result Size
≠
Small Result
```

---

# 64. Streaming alternative

Ante resultados grandes podrá recomendarse:

```text
cursor
stream
lazy
chunk
```

pero VoltStack no deberá cambiar silenciosamente la semántica solicitada.

---

# 65. Result cursor budget

Cada cursor puede mantener:

```text
connection
server-side cursor
buffer
transaction
network resources
```

---

# 66. Cursor limit

Se limitarán:

```text
active cursors per request
active cursors per worker
active cursors per database
```

cuando corresponda.

---

# 67. Stream Budget

Streams de larga duración podrán poseer:

```text
max lifetime
max idle time
max rows
max bytes
deadline
```

---

# 68. Early stream termination

Debe liberar:

```text
cursor
statement
connection
permit
```

inmediatamente.

---

# 69. Prepared Statement Budget

El Prepared Statement System deberá evitar caches ilimitados.

---

# 70. Prepared statement cache

Podrá utilizar:

```text
bounded LRU
size limit
TTL
generation invalidation
```

según driver/runtime.

---

# 71. Unbounded statement cache forbidden

```text
Every unique SQL
→ permanently cached statement
```

estará prohibido.

---

# 72. ORM memory

Procesar muchas entidades puede hacer crecer:

```text
IdentityMap
UnitOfWork
Snapshots
Relationship State
Hydration State
```

---

# 73. Lazy ≠ constant memory

Aunque se utilice:

```php
User::query()->lazy();
```

el IdentityMap puede conservar entidades.

---

# 74. Chunk ≠ constant memory

Lo mismo ocurre con:

```php
->chunk(1000)
```

si el ORM conserva referencias.

---

# 75. IdentityMap Budget

El ORM podrá observar:

```text
managed entity count
estimated memory
entity type distribution
```

---

# 76. IdentityMap hard clearing

VoltStack no deberá ejecutar:

```php
$entityManager->clear();
```

silenciosamente sobre un EntityManager compartido para solucionar presión.

---

# 77. Explicit memory policy

Para workloads masivos podrán existir políticas:

```php
enum OrmMemoryPressurePolicy
{
    case CALLER_MANAGED;
    case DETACH_PROCESSED;
    case CLEAR_DEDICATED_CONTEXT;
    case READ_ONLY_HYDRATION;
    case REJECT;
}
```

---

# 78. Dedicated processing EntityManager

Para operaciones masivas podrá recomendarse:

```text
Dedicated EntityManager
```

con lifecycle independiente.

---

# 79. UnitOfWork Budget

Un UnitOfWork con:

```text
500,000 pending inserts
```

no deberá crecer indefinidamente.

---

# 80. Pending mutation threshold

Podrán definirse límites para:

```text
NEW
DIRTY
REMOVED
relationship changes
pending persistence operations
```

---

# 81. Flush recommendation

Al alcanzar un soft threshold podrá emitirse:

```text
FlushRecommended
```

pero no deberá realizarse un `flush()` inesperado si cambia semántica.

---

# 82. Flush ≠ commit

Sigue aplicando:

```text
flush()
≠
commit()
```

---

# 83. Batch Persistence

`DATABASE_BATCH_PERSISTENCE_SYSTEM` deberá integrarse con budgets.

---

# 84. Batch size

Un batch demasiado grande puede causar:

```text
memory pressure
packet limits
lock pressure
transaction growth
statement size growth
```

---

# 85. Batch maximum

La policy podrá imponer:

```text
max operations per batch
max parameters
max estimated bytes
```

---

# 86. Chunk processing protection

Chunk Processing utilizará:

```text
ChunkResourceBudget
```

para limitar:

```text
chunk size
rows
duration
memory
number of chunks
```

---

# 87. Chunk size ≠ memory bound

Porque cada elemento puede tener distinto tamaño.

---

# 88. Lazy Collection protection

Las operaciones terminales:

```text
collect
all
toArray
count-by-consumption
```

podrán activar warnings o limits.

---

# 89. Bulk Insert protection

Bulk Insert deberá respetar:

```text
max rows per statement
max parameters
max statement bytes
transaction budget
connection budget
```

---

# 90. Bulk Update protection

Se limitarán:

```text
affected scope
batch size
transaction duration
parameter count
```

---

# 91. Bulk Delete protection

Operaciones destructivas masivas podrán requerir:

```text
explicit confirmation policy
bounded batches
affected-row guard
```

según contexto.

---

# 92. Import protection

Import deberá controlar:

```text
input size
row count
batch size
parser memory
error accumulation
transaction duration
temporary storage
```

---

# 93. Export protection

Export deberá controlar:

```text
result size
stream lifetime
output buffering
temporary files
connection lifetime
```

---

# 94. Large Dataset Processing

El documento 208 deberá integrarse con este sistema como consumidor de budgets y permits.

---

# 95. Distributed fan-out

Una consulta distribuida puede convertirse en:

```text
1 logical query
→ 100 shards
→ 100 physical queries
```

---

# 96. Fan-out amplification

Por ello:

```text
LogicalQueryCount
≠
PhysicalWorkCount
```

---

# 97. FanoutBudget

```php
final readonly class FanoutBudget
{
    public function __construct(
        public int $maxTargets,
        public int $maxParallelTargets,
        public Duration $deadline,
    ) {}
}
```

---

# 98. Unbounded scatter/gather forbidden

Una query no podrá disparar trabajo ilimitado sobre shards.

---

# 99. Fan-out concurrency

Podrá procesarse:

```text
100 shards
```

con:

```text
10 concurrent shard operations
```

en lugar de 100 simultáneas.

---

# 100. Partial fan-out failure

Resource rejection en un shard no deberá ocultarse.

El resultado global deberá preservar:

```text
PARTIAL
FAILED
UNKNOWN
```

según semántica.

---

# 101. Tenant resource isolation

Un tenant no deberá agotar todos los recursos compartidos.

---

# 102. Tenant budget

```php
final readonly class TenantDatabaseResourceBudget
{
    public function __construct(
        public TenantId $tenant,
        public ResourceLimits $limits,
    ) {}
}
```

---

# 103. Tenant fairness

Ejemplo:

```text
Tenant A → 10,000 requests
Tenant B → 10 requests
```

Tenant A no deberá monopolizar necesariamente todo el Connection Pool.

---

# 104. Noisy Neighbor Protection

VoltStack podrá implementar:

```text
per-tenant concurrency
per-tenant queue
weighted fairness
resource quotas
```

---

# 105. Tenant quota ≠ authorization

Son capas distintas.

---

# 106. Shard isolation

Un shard saturado no deberá necesariamente bloquear otros shards.

---

# 107. Bulkhead pattern

VoltStack utilizará el concepto:

```text
Bulkhead
```

para aislar recursos.

---

# 108. Bulkhead examples

```text
Writer Pool
Replica Pool

Shard A Pool
Shard B Pool

Interactive Work
Batch Work

Tenant Group A
Tenant Group B

Normal Queries
Administrative Queries
```

---

# 109. Bulkhead objective

Evitar:

```text
Failure/Saturation Domain A
        ↓
Consumes everything
        ↓
Failure Domain B collapses
```

---

# 110. Bulkhead ≠ separate physical server

Puede ser solo aislamiento lógico de capacidad.

---

# 111. Priority

VoltStack podrá distinguir:

```php
enum WorkPriority
{
    case CRITICAL;
    case HIGH;
    case NORMAL;
    case LOW;
    case BACKGROUND;
}
```

---

# 112. Priority ≠ unlimited privilege

CRITICAL tampoco podrá consumir capacidad infinita.

---

# 113. Reserved capacity

Puede existir:

```text
Normal Capacity
+
Emergency Reserve
```

---

# 114. Starvation

Un sistema de prioridades debe impedir que:

```text
LOW
```

espere eternamente.

---

# 115. Fairness policies

Posibles:

```text
FIFO
Weighted Fair Queue
Tenant Fairness
Priority + Aging
Round Robin
Custom
```

---

# 116. Priority aging

Una operación que lleva demasiado tiempo esperando podrá aumentar prioridad dentro de límites.

---

# 117. Queueing

Esperar puede ser mejor que rechazar, pero una cola infinita solo mueve el problema.

---

# 118. Bounded queue

Toda cola de resource admission deberá tener:

```text
max depth
max wait time
deadline awareness
```

---

# 119. Queue saturation

Cuando:

```text
QueueDepth = MaxQueueDepth
```

nuevas operaciones deberán:

```text
REJECT
```

o aplicar policy explícita.

---

# 120. Queue ≠ capacity

Una cola de 100,000 elementos no crea capacidad de procesamiento.

---

# 121. Little's Law consideration

Conceptualmente:

```text
L = λW
```

donde:

- `L` = trabajo medio dentro del sistema;
- `λ` = throughput;
- `W` = tiempo medio en sistema.

Si la latencia aumenta, la cantidad de trabajo concurrente puede crecer incluso sin aumentar throughput.

Esto justifica limitar concurrencia y cola.

---

# 122. Backpressure

Cuando un downstream está saturado:

```text
Database
```

VoltStack deberá propagar presión hacia:

```text
Query Engine
Jobs
HTTP lifecycle
Importers
Background processors
```

en lugar de acumular trabajo ilimitadamente.

---

# 123. Backpressure signal

```php
enum BackpressureLevel
{
    case NONE;
    case LOW;
    case MODERATE;
    case HIGH;
    case CRITICAL;
}
```

---

# 124. Backpressure actions

Podrán incluir:

```text
delay
reduce concurrency
pause producer
reduce batch size
reject low-priority work
shed optional work
```

---

# 125. Database layer boundary

Database podrá emitir la señal.

No deberá controlar directamente:

```text
HTTP server
queue broker
frontend
```

---

# 126. Framework integration

Capas superiores decidirán cómo traducir:

```text
DatabaseOverloaded
```

a:

```text
HTTP 503
job retry later
consumer pause
CLI error
```

---

# 127. Load Shedding

Cuando el sistema está críticamente saturado:

> Es preferible rechazar una parte controlada del trabajo que permitir que todo el sistema falle.

---

# 128. Load shedding candidates

Por ejemplo:

```text
background analytics
optional telemetry DB writes
non-critical reports
large exports
expensive search
low-priority jobs
```

según policy.

---

# 129. Core operations

No se marcarán arbitrariamente como prescindibles.

La clasificación deberá ser explícita.

---

# 130. Load shedding ≠ silent data loss

Una escritura rechazada deberá retornar error explícito.

Nunca:

```text
drop write
return success
```

---

# 131. Retry amplification

Supongamos:

```text
100 requests fail
```

cada una reintenta 5 veces:

```text
500 additional attempts
```

---

# 132. Retry storm

```text
Saturation
  ↓
Timeout
  ↓
Retry
  ↓
More Saturation
```

---

# 133. Retry budget integration

`238_DATABASE_RETRY_POLICY_SYSTEM` deberá consultar:

```text
Resource Pressure
```

antes de permitir nuevos retries.

---

# 134. High pressure

Con:

```text
Backpressure = CRITICAL
```

puede ser preferible:

```text
fail fast
```

a reintentar.

---

# 135. Retry permit

Los retries podrán consumir:

```text
RetryResourcePermit
```

separado del trabajo inicial.

---

# 136. Retry fairness

Los retries no deberán desplazar indefinidamente a operaciones nuevas.

---

# 137. Failover amplification

Después de una caída:

```text
10,000 requests
```

pueden intentar usar simultáneamente el nuevo writer.

---

# 138. Failover storm protection

El documento 240 deberá integrarse con:

```text
connection ramp-up
probe limits
recovery concurrency
pool warm-up limits
```

---

# 139. Recovered endpoint protection

Un endpoint recién recuperado no recibirá necesariamente toda la carga inmediatamente.

---

# 140. Ramp-up

Conceptualmente:

```text
10%
25%
50%
75%
100%
```

cuando la infraestructura de routing/load balancing lo permita.

---

# 141. Health checks

Health checks deberán poseer capacidad reservada o límites propios para no convertirse en carga adicional durante incidentes.

---

# 142. Telemetry pressure

La telemetría tampoco deberá amplificar una falla.

---

# 143. Telemetry degradation

Bajo presión crítica podrán:

```text
sample traces
aggregate metrics
drop non-critical debug events
```

pero no falsear métricas esenciales.

---

# 144. Debug information

El Query Profiler y Debug Toolbar no deberán retener ilimitadamente:

```text
queries
bindings
stack traces
plans
results
```

---

# 145. Debug budget

En desarrollo:

```php
final readonly class DebugResourceBudget
{
    public function __construct(
        public int $maxQueries,
        public int $maxEvents,
        public int $maxBytes,
    ) {}
}
```

---

# 146. Query history

Deberá ser:

```text
bounded
```

en workers persistentes.

---

# 147. Telemetry history ≠ database history

No deberá convertirse accidentalmente en un memory leak.

---

# 148. Memory Pressure Monitor

VoltStack podrá observar memoria del proceso cuando el runtime lo permita.

---

# 149. Memory state

```php
enum MemoryPressureState
{
    case NORMAL;
    case ELEVATED;
    case HIGH;
    case CRITICAL;
    case UNKNOWN;
}
```

---

# 150. PHP memory_limit

Será una última barrera física, no el mecanismo principal de protección.

---

# 151. Soft memory ceiling

VoltStack podrá definir:

```text
soft ceiling < PHP memory_limit
```

para actuar antes del fatal OOM.

---

# 152. Memory estimates

No deberán tratarse como mediciones exactas si no lo son.

---

# 153. Memory response

Ante presión elevada:

```text
reject large materialization
reduce batch size
stop prefetching
avoid optional debug retention
request worker recycle
```

según runtime/policy.

---

# 154. Worker recycling

En persistent runtimes podrá recomendarse:

```text
worker recycle
```

después de:

```text
memory threshold
request count
suspected leak
resource corruption
```

---

# 155. Database core boundary

Database podrá emitir:

```text
WorkerRecycleRecommended
```

pero RuntimeManager será responsable de reciclar el worker.

---

# 156. Temporary storage

Imports, exports y grandes operaciones pueden utilizar:

```text
temporary files
spill files
intermediate buffers
```

---

# 157. Temp storage budget

Deberá controlar:

```text
bytes
file count
lifetime
directory quota
```

---

# 158. Disk exhaustion

No deberá permitirse que un export llene todo el filesystem.

---

# 159. Spill-to-disk

Mover memoria a disco no elimina el límite.

```text
Memory Pressure
→ Disk Pressure
```

no será considerado solución infinita.

---

# 160. Resource governance hierarchy

```text
Physical Infrastructure Limits
            │
            ▼
Application Resource Envelope
            │
            ▼
Logical Database Budget
            │
            ▼
Endpoint / Shard Budget
            │
            ▼
Tenant Budget
            │
            ▼
Worker Budget
            │
            ▼
Request / Job Budget
            │
            ▼
Operation Budget
```

---

# 161. Resource Envelope

```php
final readonly class DatabaseResourceEnvelope
{
    public function __construct(
        public ResourceBudgetSet $budgets,
        public ResourceReserveSet $reserves,
        public ResourcePolicySet $policies,
    ) {}
}
```

---

# 162. Budget composition

Budgets deberán ser:

```text
deterministic
explainable
bounded
scope-aware
```

---

# 163. Dynamic limits

Algunos límites podrán adaptarse mediante telemetry.

Pero:

```text
Dynamic
≠
Unbounded
```

---

# 164. Adaptive concurrency

VoltStack podrá soportar posteriormente:

```text
latency-aware concurrency
error-aware concurrency
database-load-aware concurrency
```

---

# 165. Adaptive controller

Nunca deberá oscilar agresivamente.

Necesitará:

```text
minimum
maximum
smoothing
hysteresis
cooldown
```

---

# 166. Static defaults first

La primera implementación deberá priorizar:

```text
predictable bounded limits
```

antes que algoritmos adaptativos complejos.

---

# 167. Configuration example

```php
'database' => [
    'resources' => [
        'connections' => [
            'max' => 40,
            'reserve' => 4,
            'acquire_timeout' => '2s',
            'queue' => 100,
        ],

        'queries' => [
            'max_concurrent' => 80,
            'heavy_max_concurrent' => 10,
        ],

        'transactions' => [
            'max_concurrent' => 30,
            'max_duration' => '30s',
            'max_idle' => '5s',
        ],

        'results' => [
            'max_materialized_rows' => 100_000,
        ],

        'cursors' => [
            'max_per_worker' => 20,
            'max_lifetime' => '60s',
        ],

        'fanout' => [
            'max_targets' => 64,
            'max_parallel' => 8,
        ],
    ],
];
```

Los valores son ilustrativos, no defaults normativos.

---

# 168. Secure defaults

VoltStack deberá preferir:

```text
bounded
```

sobre:

```text
unlimited
```

para recursos peligrosos.

---

# 169. Unlimited configuration

Cuando se permita:

```text
UNLIMITED
```

deberá ser una elección explícita.

---

# 170. Unlimited ≠ null

Se evitará ambigüedad entre:

```text
null = inherit
```

y:

```text
unlimited
```

---

# 171. LimitValue

```php
final readonly class LimitValue
{
    private function __construct(
        public LimitMode $mode,
        public int|float|null $value,
    ) {}
}
```

con:

```php
enum LimitMode
{
    case VALUE;
    case INHERIT;
    case UNLIMITED;
}
```

---

# 172. Configuration validation

Se rechazará:

```text
hard < soft
negative limits
negative timeout
reserve > total capacity
invalid scope
```

---

# 173. Runtime limit update

Cambiar configuración podrá afectar:

```text
new permits
```

sin necesariamente revocar inmediatamente permisos existentes.

---

# 174. Limit reduction

Si:

```text
current usage = 50
new limit = 30
```

no deberán eliminarse arbitrariamente 20 operaciones activas.

---

# 175. Drain mode

El sistema podrá:

```text
reject new work
allow active work to finish
```

hasta volver debajo del límite.

---

# 176. Emergency revocation

Cancelar trabajo activo será una policy separada y mucho más restrictiva.

---

# 177. Resource revocation

Solo recursos explícitamente revocables podrán ser cancelados.

---

# 178. Transaction revocation risk

Cancelar una transaction puede dejar:

```text
UNKNOWN outcome
```

dependiendo del punto de fallo.

No deberá utilizarse como mecanismo trivial de presión.

---

# 179. Graceful degradation

Orden preferido:

```text
1. Stop optional work
2. Reduce concurrency
3. Apply backpressure
4. Bound queue
5. Reject low-priority new work
6. Reject normal new work
7. Preserve critical reserve
```

---

# 180. Brownout mode

Podrá existir conceptualmente:

```php
enum DatabaseDegradationMode
{
    case NORMAL;
    case CONSERVATIVE;
    case DEGRADED;
    case BROWNOUT;
    case EMERGENCY;
}
```

---

# 181. Brownout

Puede deshabilitar temporalmente:

```text
expensive optional features
large reports
non-critical analytics
deep profiling
```

mediante integraciones superiores.

---

# 182. Database core does not choose business importance

La aplicación deberá declarar qué operaciones son opcionales.

---

# 183. Resilience integration

```text
Resource Pressure
       │
       ├── Retry Policy
       ├── Circuit Breaker
       ├── Failover
       ├── Connection Pool
       ├── Query Executor
       └── Runtime
```

---

# 184. Circuit Breaker distinction

Circuit Breaker responde principalmente a:

```text
failing dependency
```

Resource Protection responde a:

```text
saturated capacity
```

---

# 185. Healthy but saturated

Un servidor puede ser:

```text
HEALTHY
+
SATURATED
```

---

# 186. Saturated ≠ failed

No deberá abrirse necesariamente el Circuit Breaker por cada resource rejection local.

---

# 187. Resource rejection classification

```php
enum ResourceRejectionReason
{
    case CONNECTION_LIMIT;
    case QUERY_CONCURRENCY_LIMIT;
    case TRANSACTION_LIMIT;
    case MEMORY_LIMIT;
    case RESULT_LIMIT;
    case CURSOR_LIMIT;
    case QUEUE_LIMIT;
    case TENANT_QUOTA;
    case SHARD_QUOTA;
    case FANOUT_LIMIT;
    case DEADLINE;
    case BACKPRESSURE;
    case EMERGENCY_SHEDDING;
}
```

---

# 188. Retry classification

Algunos resource rejections serán:

```text
transient
```

pero eso no significa:

```text
retry immediately
```

---

# 189. Retry-after hint

El sistema podrá devolver conceptualmente:

```text
RetryAfter
```

si existe información suficiente.

---

# 190. RetryAfter ≠ guarantee

Indica una recomendación, no que habrá capacidad futura.

---

# 191. Failover interaction

Saturación de una replica puede permitir elegir otra replica.

Pero:

```text
Resource Saturation
≠
Writer Failure
```

---

# 192. Load balancing

El Load Balancer podrá considerar:

```text
resource pressure
```

como señal de elegibilidad/peso.

---

# 193. Pressure-aware endpoint selection

Ejemplo:

```text
Replica A → 95% saturated
Replica B → 40% saturated
```

B puede recibir mayor peso.

---

# 194. Pressure metric uncertainty

Si pressure de B es UNKNOWN, no se asumirá 0%.

---

# 195. Security

Resource Protection también es una defensa frente a:

```text
resource abuse
accidental denial of service
query amplification
tenant abuse
unbounded exports
unbounded imports
```

---

# 196. Security ≠ resource governance

Sin embargo, no se mezclarán permisos con cuotas.

---

# 197. Query input limits

El documento 228 puede limitar:

```text
IN list size
predicate depth
sort fields
CTE depth
query complexity
```

antes de llegar a Resource Protection.

---

# 198. Defense in depth

```text
Input Security
      ↓
Query Validation
      ↓
Resource Estimation
      ↓
Admission Control
      ↓
Runtime Budget
```

---

# 199. Raw SQL

Raw SQL tampoco estará exento de:

```text
timeout
connection limits
query slots
transaction limits
result limits
```

---

# 200. Administrative bypass

Si existe bypass deberá ser:

```text
explicit
authorized
audited
bounded where possible
```

---

# 201. Telemetry

Eventos conceptuales:

```text
DatabaseResourcePressureChanged
DatabaseResourceThresholdReached
DatabaseAdmissionRejected
DatabaseAdmissionQueued
DatabasePermitAcquired
DatabasePermitReleased
DatabaseResourceLeakSuspected
DatabaseBackpressureActivated
DatabaseLoadSheddingActivated
DatabaseTenantQuotaReached
DatabaseWorkerResourcePressure
DatabaseMemoryPressureChanged
```

---

# 202. Event volume

No se emitirá necesariamente:

```text
PermitAcquired
PermitReleased
```

como evento global costoso para cada query.

Podrán ser:

```text
internal counters
sampled telemetry
debug-only traces
```

---

# 203. Metrics

Ejemplos:

```text
db.resource.connections.active
db.resource.connections.waiting
db.resource.connections.rejected

db.resource.queries.active
db.resource.queries.waiting

db.resource.transactions.active

db.resource.cursors.active

db.resource.memory.pressure

db.resource.admission.rejected
db.resource.queue.depth
db.resource.queue.wait

db.resource.tenant.rejected

db.resource.fanout.active
```

---

# 204. Metrics cardinality

No utilizar:

```text
user_id
request_id
raw_sql
transaction_id
entity_id
```

como labels globales.

---

# 205. Resource snapshot

```php
final readonly class DatabaseResourceSnapshot
{
    public function __construct(
        public Instant $capturedAt,
        public array $usage,
        public BackpressureLevel $backpressure,
    ) {}
}
```

---

# 206. Diagnostics

Ejemplo:

```text
Database Resource State

Database:
    primary

Connections:
    active: 38
    waiting: 12
    usable_limit: 40
    reserve: 4
    pressure: HIGH

Queries:
    active: 74
    limit: 80

Heavy Queries:
    active: 10
    limit: 10
    pressure: HARD_LIMIT

Transactions:
    active: 18
    limit: 30

Result Cursors:
    active: 14
    limit: 20

Backpressure:
    HIGH

Admission:
    LOW_PRIORITY_REJECTED
```

---

# 207. Explain admission

```php
$db->resources()->explain($operation);
```

podrá producir:

```text
Operation:
    EXPORT

Priority:
    BACKGROUND

Required:
    1 connection
    1 streaming cursor
    1 query slot

Connection Capacity:
    AVAILABLE

Cursor Capacity:
    HARD LIMIT

Decision:
    REJECT

Reason:
    RESULT_CURSOR_LIMIT
```

---

# 208. Error hierarchy

```text
DatabaseException
└── DatabaseResourceException
    ├── DatabaseResourceExhaustedException
    ├── DatabaseAdmissionRejectedException
    ├── DatabaseResourceQueueFullException
    ├── DatabaseResourceWaitTimeoutException
    ├── ConnectionPoolExhaustedException
    ├── QueryConcurrencyLimitException
    ├── TransactionResourceLimitException
    ├── ResultMaterializationLimitException
    ├── CursorResourceLimitException
    ├── StreamResourceLimitException
    ├── OrmMemoryLimitException
    ├── UnitOfWorkLimitException
    ├── TenantResourceQuotaException
    ├── FanoutResourceLimitException
    ├── TemporaryStorageLimitException
    ├── ResourcePermitException
    └── ResourceAccountingException
```

---

# 209. Error information

Las excepciones podrán contener:

```text
resource type
scope
current usage
soft limit
hard limit
wait duration
priority
rejection reason
retryability hint
```

sin exponer información sensible.

---

# 210. Persistent runtime

En FrankenPHP:

```text
Worker
├── Request 1
├── Request 2
├── Request 3
└── ...
```

un leak puede acumularse entre requests.

---

# 211. Request cleanup

Al terminar cada request deberán liberarse:

```text
operation permits
request permits
unowned cursors
streams
temporary query state
request-scoped accounting
```

según ownership.

---

# 212. Connection reuse

Las conexiones intencionalmente pool-managed no deberán cerrarse como si fueran leaks.

---

# 213. Ownership is essential

```text
Resource exists after request
```

no implica leak si su owner es:

```text
Worker Connection Pool
```

---

# 214. Scoped resource registry

```php
interface ScopedResourceRegistry
{
    public function register(OwnedResource $resource): void;

    public function releaseScope(ResourceScopeId $scope): void;
}
```

---

# 215. Leak detection

Al finalizar scope:

```text
resource still active
+
owner scope ended
```

podrá producir:

```text
LEAK_SUSPECTED
```

---

# 216. Force cleanup

Solo se realizará cuando el recurso tenga semántica segura de cierre.

---

# 217. Unknown transaction state

Una transaction abandonada puede requerir:

```text
connection quarantine
rollback attempt
EntityManager taint
```

antes de volver al pool.

---

# 218. Resource quarantine

Recursos potencialmente corruptos no regresarán inmediatamente al pool.

---

# 219. Connection quarantine

Ejemplo:

```text
connection lost during transaction
```

no será tratada como conexión reutilizable.

---

# 220. Worker health integration

Resource leaks repetidos podrán degradar:

```text
WorkerHealth
```

---

# 221. RuntimeManager integration

```text
Database Resource System
       ↓
WorkerRecycleRecommended
       ↓
RuntimeManagerServer
```

La decisión final de lifecycle del worker corresponde al runtime.

---

# 222. FrankenPHP

Será el primer runtime objetivo de integración profunda.

---

# 223. RoadRunner

El diseño deberá permitir el mismo modelo de:

```text
scope cleanup
worker budgets
persistent pool limits
recycle recommendation
```

---

# 224. OpenSwoole

Además deberá considerar alta concurrencia/coroutines y evitar compartir accidentalmente permits o contexto entre coroutines.

---

# 225. Coroutine scope

```text
Worker Scope
≠
Coroutine Scope
≠
Request Scope
```

---

# 226. Atomic accounting

Contadores compartidos deberán actualizarse de forma segura bajo concurrencia.

---

# 227. Oversubscription race

No deberá ocurrir:

```text
limit = 10
current = 9

Coroutine A sees 9 → admits
Coroutine B sees 9 → admits
Coroutine C sees 9 → admits

current = 12
```

por accounting no atómico.

---

# 228. Reservation before execution

La capacidad deberá reservarse antes de iniciar el trabajo protegido.

---

# 229. Resource reservation

```text
Check
+
Reserve
```

deberán ser una operación lógicamente atómica.

---

# 230. Distributed budgets

Un límite global entre múltiples máquinas requiere coordinación externa.

---

# 231. Local vs global limits

VoltStack distinguirá:

```php
enum ResourceLimitScopeMode
{
    case LOCAL;
    case DISTRIBUTED;
    case EXTERNAL;
}
```

---

# 232. Local limit

Ejemplo:

```text
20 connections per worker
```

---

# 233. Distributed limit

Ejemplo:

```text
500 concurrent exports across cluster
```

requiere provider coordinado.

---

# 234. External limit

Puede ser impuesto por:

```text
database proxy
cloud service
orchestrator
queue
```

---

# 235. No fake global quota

Si solo existe accounting local, VoltStack no declarará que conoce el uso global exacto.

---

# 236. Distributed Resource Provider

```php
interface DistributedResourceCoordinator
{
    public function acquire(
        DistributedResourceRequest $request
    ): DistributedResourcePermit;
}
```

será opcional.

---

# 237. Provider failure

Si el coordinador distribuido falla, la policy deberá decidir:

```text
FAIL_CLOSED
FAIL_OPEN
LOCAL_FALLBACK
UNKNOWN
```

---

# 238. Critical quotas

Para límites de seguridad estrictos:

```text
FAIL_CLOSED
```

puede ser apropiado.

---

# 239. Availability-oriented limits

Otros podrán permitir fallback local.

Nunca será implícito.

---

# 240. Testing architecture

Deberán existir:

```text
FakeResourceGovernor
FakeAdmissionController
FakeResourceMeter
FakeResourceCoordinator
FakeClock
FakeMemoryMonitor
ResourcePressureSimulator
PermitLeakDetector
```

---

# 241. Connection exhaustion test

Config:

```text
max connections = 2
```

Abrir dos leases.

Solicitar tercero.

Esperado:

```text
WAIT
```

o:

```text
REJECT
```

según policy.

Nunca:

```text
unbounded connection creation
```

---

# 242. Queue bound test

```text
queue max = 10
```

Con 10 waiting:

```text
request 11
```

deberá ser rechazado.

---

# 243. Acquisition timeout test

Una operación esperando conexión supera:

```text
2 seconds
```

Esperado:

```text
DatabaseResourceWaitTimeoutException
```

---

# 244. Permit release test

Después de operación:

```text
active permits
```

deberán volver al valor anterior.

---

# 245. Exception cleanup test

Si execution lanza excepción:

```text
permit
```

también deberá liberarse.

---

# 246. Cursor early-break test

```php
foreach ($query->cursor() as $row) {
    break;
}
```

deberá liberar correctamente recursos asociados.

---

# 247. Lazy materialization test

Una Lazy Collection supera el límite de materialización.

Esperado:

```text
ResultMaterializationLimitException
```

o policy explícita.

---

# 248. IdentityMap pressure test

Procesar cientos de miles de entidades.

Esperado:

```text
pressure observable
```

sin ejecutar `clear()` silencioso sobre contexto compartido.

---

# 249. UnitOfWork pressure test

Registrar demasiadas entidades pending.

Esperado:

```text
warning / rejection
```

según threshold.

---

# 250. Fan-out test

Query intenta:

```text
100 shards
```

con:

```text
maxTargets = 32
```

Esperado:

```text
FanoutResourceLimitException
```

---

# 251. Parallel fan-out test

```text
targets = 32
parallel limit = 4
```

Nunca deberán ejecutarse más de cuatro simultáneamente dentro de ese scope.

---

# 252. Tenant isolation test

Tenant A satura su cuota.

Tenant B conserva capacidad.

---

# 253. Priority test

Con saturación moderada:

```text
BACKGROUND
```

puede ser rechazado mientras:

```text
CRITICAL
```

usa reserva disponible.

---

# 254. Critical limit test

Incluso CRITICAL deberá ser rechazado al consumir su límite absoluto.

---

# 255. Retry storm test

Con DB saturada:

```text
100 failed requests
```

no deberán convertirse inmediatamente en:

```text
500 concurrent retries
```

---

# 256. Failover storm test

Nuevo writer recibe tráfico de recuperación gradualmente según policy.

---

# 257. Worker leak test

Un request termina dejando un cursor request-owned.

Esperado:

```text
leak detection
cleanup/quarantine where safe
telemetry
```

---

# 258. Transaction leak test

Request termina con transaction activa.

Esperado:

```text
rollback attempt
connection state verification
possible quarantine
context taint
```

Nunca:

```text
return blindly to pool
```

---

# 259. Memory pressure test

Al superar soft ceiling:

```text
BackpressureLevel ↑
```

y operaciones grandes podrán ser rechazadas.

---

# 260. Persistent worker test

Después de miles de requests:

```text
request-scoped accounting
```

no deberá acumularse.

---

# 261. Concurrency race test

100 coroutines intentan adquirir 10 permits.

En ningún instante deberán existir más de 10 permits válidos.

---

# 262. Distributed accounting failure test

Si el provider falla:

```text
policy result
```

deberá ser determinista y explícito.

---

# 263. Directory structure

```text
src/Quantum/Database/Resource/
│
├── Contract/
│   ├── DatabaseResourceGovernor.php
│   ├── DatabaseAdmissionController.php
│   ├── ResourceMeter.php
│   ├── ResourcePermit.php
│   ├── QueryConcurrencyLimiter.php
│   ├── ScopedResourceRegistry.php
│   ├── MemoryPressureMonitor.php
│   └── DistributedResourceCoordinator.php
│
├── Model/
│   ├── DatabaseResourceType.php
│   ├── ResourceScope.php
│   ├── ResourceBudget.php
│   ├── ResourceBudgetSet.php
│   ├── ResourceLimits.php
│   ├── ResourceThresholds.php
│   ├── ResourceReserve.php
│   ├── ResourceUsage.php
│   ├── ResourceAllocation.php
│   ├── ResourceSnapshot.php
│   ├── ResourceEstimate.php
│   ├── ResourceMeasurementConfidence.php
│   ├── LimitValue.php
│   ├── LimitMode.php
│   └── ResourceLimitScopeMode.php
│
├── Admission/
│   ├── AdmissionDecisionType.php
│   ├── AdmissionResult.php
│   ├── DefaultAdmissionController.php
│   ├── DatabaseWorkRequest.php
│   └── WorkPriority.php
│
├── Governor/
│   ├── DefaultDatabaseResourceGovernor.php
│   ├── HierarchicalResourceGovernor.php
│   ├── ResourceBudgetResolver.php
│   └── ResourceEnvelopeResolver.php
│
├── Permit/
│   ├── DefaultResourcePermit.php
│   ├── ResourcePermitRegistry.php
│   ├── ResourcePermitState.php
│   └── ResourcePermitOwner.php
│
├── Connection/
│   ├── ConnectionResourceBudget.php
│   ├── ConnectionAcquisitionPolicy.php
│   ├── ConnectionPermit.php
│   └── ConnectionCapacityManager.php
│
├── Query/
│   ├── QueryResourceBudget.php
│   ├── QueryExecutionPermit.php
│   ├── QueryWorkClass.php
│   ├── QueryResourceEstimator.php
│   └── DefaultQueryConcurrencyLimiter.php
│
├── Transaction/
│   ├── TransactionResourceBudget.php
│   └── TransactionResourceMonitor.php
│
├── Result/
│   ├── ResultResourceBudget.php
│   ├── ResultMaterializationGuard.php
│   ├── CursorResourceBudget.php
│   └── StreamResourceBudget.php
│
├── ORM/
│   ├── OrmResourceBudget.php
│   ├── IdentityMapResourceMonitor.php
│   ├── UnitOfWorkResourceMonitor.php
│   └── OrmMemoryPressurePolicy.php
│
├── Bulk/
│   ├── BatchResourceBudget.php
│   ├── ChunkResourceBudget.php
│   ├── BulkResourceBudget.php
│   ├── ImportResourceBudget.php
│   └── ExportResourceBudget.php
│
├── Distribution/
│   ├── FanoutBudget.php
│   ├── FanoutPermit.php
│   └── DistributedResourceCoordinator.php
│
├── Tenant/
│   ├── TenantDatabaseResourceBudget.php
│   ├── TenantQuotaResolver.php
│   └── TenantFairnessPolicy.php
│
├── Backpressure/
│   ├── BackpressureLevel.php
│   ├── BackpressureController.php
│   ├── BackpressureSignal.php
│   └── LoadSheddingPolicy.php
│
├── Bulkhead/
│   ├── ResourceBulkhead.php
│   ├── BulkheadRegistry.php
│   └── BulkheadPolicy.php
│
├── Memory/
│   ├── MemoryPressureState.php
│   ├── ProcessMemoryMonitor.php
│   └── WorkerMemoryPolicy.php
│
├── Temporary/
│   ├── TemporaryStorageBudget.php
│   └── TemporaryStorageMonitor.php
│
├── Runtime/
│   ├── ScopedResourceRegistry.php
│   ├── ResourceScopeCleaner.php
│   ├── ResourceLeakDetector.php
│   ├── ResourceQuarantine.php
│   └── WorkerRecycleRecommendation.php
│
├── Telemetry/
│   └── DatabaseResourceTelemetry.php
│
├── Diagnostics/
│   ├── DatabaseResourceDiagnostics.php
│   ├── AdmissionExplainResult.php
│   └── ResourcePressureReport.php
│
├── Testing/
│   ├── FakeResourceGovernor.php
│   ├── FakeAdmissionController.php
│   ├── FakeResourceMeter.php
│   ├── FakeResourceCoordinator.php
│   ├── FakeMemoryMonitor.php
│   └── ResourcePressureSimulator.php
│
└── Exception/
    ├── DatabaseResourceException.php
    ├── DatabaseResourceExhaustedException.php
    ├── DatabaseAdmissionRejectedException.php
    ├── DatabaseResourceQueueFullException.php
    ├── DatabaseResourceWaitTimeoutException.php
    ├── ConnectionPoolExhaustedException.php
    ├── QueryConcurrencyLimitException.php
    ├── TransactionResourceLimitException.php
    ├── ResultMaterializationLimitException.php
    ├── CursorResourceLimitException.php
    ├── StreamResourceLimitException.php
    ├── OrmMemoryLimitException.php
    ├── UnitOfWorkLimitException.php
    ├── TenantResourceQuotaException.php
    ├── FanoutResourceLimitException.php
    ├── TemporaryStorageLimitException.php
    ├── ResourcePermitException.php
    └── ResourceAccountingException.php
```

---

# 264. Flujo de integración

```text
Application
     │
     ▼
Database Public API
     │
     ▼
Query / ORM / Bulk / Import / Export
     │
     ▼
Resource Requirement
     │
     ▼
┌─────────────────────────────────────┐
│       RESOURCE GOVERNANCE           │
│                                     │
│ Budget Resolver                     │
│ Admission Controller                │
│ Bulkhead                            │
│ Concurrency Limiter                 │
│ Backpressure                        │
│ Resource Accounting                 │
└──────────────────┬──────────────────┘
                   │
             Resource Permit
                   │
                   ▼
             Query Executor
                   │
                   ▼
           Connection Manager
                   │
                   ▼
             Connection Pool
                   │
                   ▼
                Driver
                   │
                   ▼
               Database
```

---

# 265. Resource lifecycle

```text
Operation Created
       │
       ▼
Estimate Resources
       │
       ▼
Resolve Effective Budget
       │
       ▼
Admission Evaluation
       │
 ┌─────┼──────────────┐
 │     │              │
ADMIT WAIT          REJECT
 │     │
 │   bounded
 │    queue
 │     │
 └─────┘
    │
    ▼
Reserve Resources
    │
    ▼
Acquire Permit
    │
    ▼
Execute
    │
    ▼
Observe Pressure
    │
    ├── NORMAL
    ├── WARNING
    ├── BACKPRESSURE
    └── HARD LIMIT
    │
    ▼
Finish / Fail / Cancel
    │
    ▼
Release Resource
    │
    ▼
Update Accounting
```

---

# 266. Invariantes arquitectónicas

## DB-RESOURCE-001
Todos los recursos finitos críticos deberán poder poseer límites.

## DB-RESOURCE-002
Resource Protection no será Performance Optimization.

## DB-RESOURCE-003
Resource Protection no será Circuit Breaker.

## DB-RESOURCE-004
Resource Protection no será Retry Policy.

## DB-RESOURCE-005
Resource Protection no será Connection Pooling.

## DB-RESOURCE-006
Resource Protection no será Authorization.

## DB-RESOURCE-007
Resource Protection no será Query Validation.

## DB-RESOURCE-008
Capacidad física no será capacidad utilizable.

## DB-RESOURCE-009
Podrá reservarse capacidad de seguridad.

## DB-RESOURCE-010
Hard limits deberán proteger antes del límite físico cuando sea posible.

## DB-RESOURCE-011
UNKNOWN usage no será zero usage.

## DB-RESOURCE-012
Estimated usage no será exact usage.

## DB-RESOURCE-013
Measurement confidence será explícita cuando importe.

## DB-RESOURCE-014
Resource budgets tendrán scope.

## DB-RESOURCE-015
Operation scope no será Worker scope.

## DB-RESOURCE-016
Worker scope no será Application scope.

## DB-RESOURCE-017
Tenant scope no será Database scope.

## DB-RESOURCE-018
Shard scope no será Global scope.

## DB-RESOURCE-019
Límites jerárquicos deberán componerse.

## DB-RESOURCE-020
Un hijo no elevará silenciosamente el hard limit del padre.

## DB-RESOURCE-021
UNLIMITED será explícito.

## DB-RESOURCE-022
INHERIT será distinto de UNLIMITED.

## DB-RESOURCE-023
Null no significará ambiguamente unlimited.

## DB-RESOURCE-024
Configuraciones inválidas serán rechazadas.

## DB-RESOURCE-025
Admission será previo al consumo protegido.

## DB-RESOURCE-026
Check y reserve serán lógicamente atómicos.

## DB-RESOURCE-027
Un permit tendrá owner.

## DB-RESOURCE-028
Un permit tendrá lifecycle.

## DB-RESOURCE-029
Todo permit adquirido deberá liberarse.

## DB-RESOURCE-030
Exception paths deberán liberar permits.

## DB-RESOURCE-031
Cancellation paths deberán liberar permits.

## DB-RESOURCE-032
Destructor no será mecanismo primario de correctness.

## DB-RESOURCE-033
Scoped cleanup deberá existir.

## DB-RESOURCE-034
Resource leak deberá ser observable cuando sea posible.

## DB-RESOURCE-035
Long-lived resource no será leak si posee owner válido.

## DB-RESOURCE-036
Connection Pool ownership será distinto de Request ownership.

## DB-RESOURCE-037
Pool capacity será bounded.

## DB-RESOURCE-038
Connection acquisition wait será bounded.

## DB-RESOURCE-039
Connection acquisition queue será bounded.

## DB-RESOURCE-040
Pool exhaustion tendrá error específico.

## DB-RESOURCE-041
Connection limit no será Query Concurrency limit.

## DB-RESOURCE-042
Query concurrency será limitable independientemente.

## DB-RESOURCE-043
Heavy query concurrency podrá ser limitada separadamente.

## DB-RESOURCE-044
UNKNOWN query cost no será LIGHT automáticamente.

## DB-RESOURCE-045
Estimated query cost no será actual cost.

## DB-RESOURCE-046
Transaction concurrency será bounded cuando se configure.

## DB-RESOURCE-047
Transaction duration será bounded cuando se configure.

## DB-RESOURCE-048
Transaction idle time podrá ser bounded.

## DB-RESOURCE-049
Transaction timeout no será Query timeout.

## DB-RESOURCE-050
Resource governor no hará commit de transaction ajena.

## DB-RESOURCE-051
Resource governor no hará rollback arbitrario para liberar capacidad.

## DB-RESOURCE-052
Active transaction cancellation preservará outcome uncertainty.

## DB-RESOURCE-053
Unknown transaction state podrá requerir connection quarantine.

## DB-RESOURCE-054
Result materialization será gobernable.

## DB-RESOURCE-055
Unknown cardinality no será small cardinality.

## DB-RESOURCE-056
Large result no será materializado ilimitadamente por default.

## DB-RESOURCE-057
Streaming será alternativa explícita.

## DB-RESOURCE-058
Chunking será alternativa explícita.

## DB-RESOURCE-059
Lazy Collection será alternativa explícita.

## DB-RESOURCE-060
Resource governor no cambiará silenciosamente get() por stream().

## DB-RESOURCE-061
Cursor count será bounded cuando se configure.

## DB-RESOURCE-062
Cursor lifetime será bounded cuando se configure.

## DB-RESOURCE-063
Stream lifetime será bounded cuando se configure.

## DB-RESOURCE-064
Early cursor termination liberará recursos.

## DB-RESOURCE-065
Early stream termination liberará recursos.

## DB-RESOURCE-066
Prepared statement caches serán bounded.

## DB-RESOURCE-067
Unique SQL no producirá cache infinito.

## DB-RESOURCE-068
Lazy no significará constant memory.

## DB-RESOURCE-069
Chunk no significará constant memory.

## DB-RESOURCE-070
Streaming no significará zero memory.

## DB-RESOURCE-071
IdentityMap growth será observable.

## DB-RESOURCE-072
UnitOfWork growth será observable.

## DB-RESOURCE-073
Resource pressure no ejecutará EntityManager clear silencioso.

## DB-RESOURCE-074
Dedicated EntityManager podrá utilizarse para procesamiento masivo.

## DB-RESOURCE-075
Read-only hydration podrá reducir presión cuando semánticamente válido.

## DB-RESOURCE-076
UnitOfWork podrá tener thresholds.

## DB-RESOURCE-077
Flush recommendation no será automatic flush.

## DB-RESOURCE-078
Flush no será Commit.

## DB-RESOURCE-079
Batch size será bounded.

## DB-RESOURCE-080
Chunk size será bounded.

## DB-RESOURCE-081
Chunk size no será memory bound.

## DB-RESOURCE-082
Bulk Insert respetará resource limits.

## DB-RESOURCE-083
Bulk Update respetará resource limits.

## DB-RESOURCE-084
Bulk Delete respetará resource limits.

## DB-RESOURCE-085
Import respetará resource limits.

## DB-RESOURCE-086
Export respetará resource limits.

## DB-RESOURCE-087
Large Dataset Processing respetará resource limits.

## DB-RESOURCE-088
Distributed fan-out será bounded.

## DB-RESOURCE-089
Logical query no será physical work count.

## DB-RESOURCE-090
Fan-out targets tendrán máximo configurable.

## DB-RESOURCE-091
Fan-out parallelism tendrá máximo configurable.

## DB-RESOURCE-092
Scatter/gather no será ilimitado.

## DB-RESOURCE-093
Partial distributed rejection será visible.

## DB-RESOURCE-094
Resource rejection en shard no será fake global success.

## DB-RESOURCE-095
Tenant podrá tener quota.

## DB-RESOURCE-096
Tenant quota no será authorization.

## DB-RESOURCE-097
Tenant A no deberá necesariamente consumir toda capacidad de Tenant B.

## DB-RESOURCE-098
Noisy-neighbor protection será soportable.

## DB-RESOURCE-099
Shard saturation podrá aislarse.

## DB-RESOURCE-100
Bulkheads podrán aislar capacity domains.

## DB-RESOURCE-101
Bulkhead no requerirá servidor físico separado.

## DB-RESOURCE-102
Work priority será explícita.

## DB-RESOURCE-103
Priority no otorgará capacidad infinita.

## DB-RESOURCE-104
Critical work podrá usar reserva explícita.

## DB-RESOURCE-105
Critical work también tendrá hard limit.

## DB-RESOURCE-106
Fairness será policy explícita.

## DB-RESOURCE-107
Priority scheduling deberá considerar starvation.

## DB-RESOURCE-108
Queues serán bounded.

## DB-RESOURCE-109
Queue depth tendrá máximo.

## DB-RESOURCE-110
Queue wait tendrá máximo.

## DB-RESOURCE-111
Queue no será capacity.

## DB-RESOURCE-112
Backpressure será explícita.

## DB-RESOURCE-113
Backpressure podrá propagarse a capas superiores.

## DB-RESOURCE-114
Database core no controlará directamente HTTP.

## DB-RESOURCE-115
Database core no controlará directamente queue broker.

## DB-RESOURCE-116
Load shedding será explícito.

## DB-RESOURCE-117
Load shedding no retornará fake success.

## DB-RESOURCE-118
Business importance será declarada por capas apropiadas.

## DB-RESOURCE-119
Retry deberá respetar resource pressure.

## DB-RESOURCE-120
Retry no será ilimitado bajo saturación.

## DB-RESOURCE-121
Retries podrán tener budget separado.

## DB-RESOURCE-122
Retry storm deberá mitigarse.

## DB-RESOURCE-123
Failover storm deberá mitigarse.

## DB-RESOURCE-124
Recovered endpoint podrá tener ramp-up.

## DB-RESOURCE-125
Health checks serán bounded.

## DB-RESOURCE-126
Telemetry será bounded.

## DB-RESOURCE-127
Debug query history será bounded.

## DB-RESOURCE-128
Debug Toolbar no retendrá información ilimitada.

## DB-RESOURCE-129
Profiler no será memory leak.

## DB-RESOURCE-130
Memory pressure será observable cuando sea posible.

## DB-RESOURCE-131
PHP memory_limit no será primera defensa.

## DB-RESOURCE-132
Soft memory ceiling podrá actuar antes del OOM.

## DB-RESOURCE-133
Memory estimates no serán tratadas como exactas.

## DB-RESOURCE-134
Database podrá recomendar worker recycling.

## DB-RESOURCE-135
Database no reciclará worker directamente.

## DB-RESOURCE-136
Temporary storage será bounded.

## DB-RESOURCE-137
Spill-to-disk no será unlimited.

## DB-RESOURCE-138
Export no podrá llenar filesystem sin límites.

## DB-RESOURCE-139
Import no podrá consumir temporary storage ilimitado.

## DB-RESOURCE-140
Adaptive limits siempre tendrán min/max.

## DB-RESOURCE-141
Adaptive limits tendrán hysteresis.

## DB-RESOURCE-142
Static predictable limits serán implementación inicial preferida.

## DB-RESOURCE-143
Runtime limit reduction no matará arbitrariamente trabajo activo.

## DB-RESOURCE-144
Limit reduction podrá activar drain mode.

## DB-RESOURCE-145
Emergency revocation será policy separada.

## DB-RESOURCE-146
Graceful degradation precederá emergency termination.

## DB-RESOURCE-147
Healthy no será Unsaturated.

## DB-RESOURCE-148
Saturated no será Failed.

## DB-RESOURCE-149
Resource rejection no abrirá necesariamente Circuit Breaker.

## DB-RESOURCE-150
Circuit Breaker no sustituirá resource governor.

## DB-RESOURCE-151
Load Balancer podrá consumir pressure information.

## DB-RESOURCE-152
UNKNOWN pressure no será zero pressure.

## DB-RESOURCE-153
Raw SQL respetará resource limits.

## DB-RESOURCE-154
Administrative bypass será explícito.

## DB-RESOURCE-155
Administrative bypass será autorizado.

## DB-RESOURCE-156
Administrative bypass será auditable cuando corresponda.

## DB-RESOURCE-157
Resource metrics tendrán cardinalidad controlada.

## DB-RESOURCE-158
Raw SQL no será metric label.

## DB-RESOURCE-159
User ID no será global metric label.

## DB-RESOURCE-160
Request ID no será global metric label.

## DB-RESOURCE-161
Resource diagnostics serán estructurados.

## DB-RESOURCE-162
Admission rejection será explicable.

## DB-RESOURCE-163
Resource scope será diagnosticable.

## DB-RESOURCE-164
Current usage será diagnosticable cuando sea conocido.

## DB-RESOURCE-165
Effective limit será diagnosticable.

## DB-RESOURCE-166
Resource pressure será diagnosticable.

## DB-RESOURCE-167
Persistent workers deberán limpiar request-scoped resources.

## DB-RESOURCE-168
Persistent workers no limpiarán worker-owned pools como request leaks.

## DB-RESOURCE-169
Request state no sobrevivirá accidentalmente a la siguiente request.

## DB-RESOURCE-170
Resource accounting no sobrevivirá accidentalmente a su scope.

## DB-RESOURCE-171
OpenSwoole coroutine context será aislado.

## DB-RESOURCE-172
Permit accounting compartido será concurrency-safe.

## DB-RESOURCE-173
Concurrent admission no podrá oversubscribe por race.

## DB-RESOURCE-174
Distributed budget no será fingido mediante local counters.

## DB-RESOURCE-175
Local limit será identificable como local.

## DB-RESOURCE-176
Distributed limit requerirá coordination provider.

## DB-RESOURCE-177
External limit será representable.

## DB-RESOURCE-178
Distributed coordinator failure tendrá policy explícita.

## DB-RESOURCE-179
Fail-open no será default implícito.

## DB-RESOURCE-180
Fail-closed no será default implícito para todos los recursos.

## DB-RESOURCE-181
Testing podrá simular exhaustion.

## DB-RESOURCE-182
Testing podrá simular queue saturation.

## DB-RESOURCE-183
Testing podrá simular permit leaks.

## DB-RESOURCE-184
Testing podrá simular memory pressure.

## DB-RESOURCE-185
Testing podrá simular retry storms.

## DB-RESOURCE-186
Testing podrá simular failover storms.

## DB-RESOURCE-187
Testing podrá verificar tenant isolation.

## DB-RESOURCE-188
Testing podrá verificar fan-out limits.

## DB-RESOURCE-189
Testing podrá verificar cleanup tras exceptions.

## DB-RESOURCE-190
Testing podrá verificar cleanup tras cancellation.

## DB-RESOURCE-191
Testing podrá verificar transaction quarantine.

## DB-RESOURCE-192
Testing podrá verificar persistent worker cleanup.

## DB-RESOURCE-193
Testing podrá verificar atomic admission.

## DB-RESOURCE-194
Resource exhaustion tendrá errores específicos.

## DB-RESOURCE-195
Resource exhaustion no será ocultado como generic query failure.

## DB-RESOURCE-196
Resource exhaustion no será ocultado como connection failure.

## DB-RESOURCE-197
Resource rejection podrá incluir retryability hint.

## DB-RESOURCE-198
Retryability hint no será retry guarantee.

## DB-RESOURCE-199
Correctness tendrá prioridad sobre throughput.

## DB-RESOURCE-200
Controlled degradation tendrá prioridad sobre uncontrolled collapse.

---

# 267. Modelo formal

Sea un recurso:

```text
r
```

con:

```text
C(r) = capacidad física
R(r) = reserva
L(r) = límite operativo
U(r,t) = uso en tiempo t
```

Deberá cumplirse idealmente:

```text
L(r) ≤ C(r) - R(r)
```

---

# 268. Admission

Para una operación `o` con demanda estimada:

```text
D(o,r)
```

una condición simplificada sería:

```text
U(r,t) + D(o,r) ≤ L(r)
```

para cada recurso obligatorio `r`.

---

# 269. Multiple budgets

Si existen scopes:

```text
S = {application, database, tenant, worker, request}
```

entonces:

```text
Admit(o)
=
∀ s ∈ S,
∀ r ∈ Resources(o):

U(s,r) + D(o,r) ≤ L(s,r)
```

cuando el uso sea conocido y el recurso requiera hard enforcement.

---

# 270. Queue condition

Si no puede admitirse inmediatamente:

```text
WAIT
```

solo será válido cuando:

```text
QueueDepth < QueueLimit
∧
RemainingDeadline > 0
∧
PolicyAllowsWait
```

---

# 271. Rejection condition

```text
REJECT
```

cuando:

```text
HardLimitExceeded
∨ QueueFull
∨ DeadlineInsufficient
∨ EmergencyShedding
∨ PolicyRejects
```

---

# 272. Saturation feedback loop

Sin protección:

```text
λ > μ
  ↓
Queue ↑
  ↓
Latency ↑
  ↓
Timeout ↑
  ↓
Retry ↑
  ↓
λ ↑↑
```

Con protección:

```text
λ > μ
  ↓
Admission Control
  ↓
Bounded Queue
  ↓
Backpressure
  ↓
Controlled Rejection
  ↓
System remains bounded
```

---

# 273. Principio de estabilidad

El objetivo no es:

```text
Never Reject Work
```

sino:

```text
Never Allow Unbounded Work
```

---

# 274. Regla maestra

> **Todo recurso finito utilizado por VoltStack Database deberá tener ownership y un modelo de capacidad. Cuando la demanda supere la capacidad segura, VoltStack deberá limitar concurrencia, aplicar backpressure, esperar dentro de límites o rechazar trabajo explícitamente; nunca deberá transformar saturación en crecimiento ilimitado de conexiones, memoria, colas, cursors, transactions, retries o fan-out.**

En forma compacta:

```text
Available
≠
Unlimited

Healthy
≠
Unsaturated

Queue
≠
Capacity

Connection Pool
≠
Infinite Connections

Connection Limit
≠
Query Limit

Query Timeout
≠
Resource Governance

Chunk
≠
Constant Memory

Lazy
≠
Constant Memory

Streaming
≠
Zero Resource Usage

Flush
≠
Commit

Retry
≠
Free Work

Failover
≠
Free Capacity

Logical Query
≠
Single Physical Operation

Local Counter
≠
Global Quota

UNKNOWN Usage
≠
Zero Usage

Priority
≠
Unlimited Privilege

Resource Rejection
≠
Database Failure

Saturation
≠
Endpoint Failure

Load Shedding
≠
Silent Data Loss
```

Prioridad arquitectónica:

```text
Correctness
    >
Resource Ownership
    >
Bounded Capacity
    >
Isolation
    >
System Stability
    >
Backpressure
    >
Controlled Degradation
    >
Fairness
    >
Throughput
    >
Convenience
```

---

# 275. Resultado del Bloque 23 — Resilience

Con este documento queda completa la arquitectura de resiliencia de VoltStack Database:

```text
BLOCK 23 — RESILIENCE

✓ 235_DATABASE_RESILIENCE_ARCHITECTURE.md
│
├── Failure classification
├── resilience policies
├── evidence model
└── resilience boundaries
│
✓ 236_DATABASE_CONNECTION_FAILURE_HANDLING_SYSTEM.md
│
├── connection failures
├── connection state
├── broken connections
└── pool reconciliation
│
✓ 237_DATABASE_QUERY_FAILURE_HANDLING_SYSTEM.md
│
├── query failure classification
├── execution failures
├── outcome evidence
└── query recovery boundaries
│
✓ 238_DATABASE_RETRY_POLICY_SYSTEM.md
│
├── retry eligibility
├── retry budgets
├── backoff
├── jitter
└── replay safety
│
✓ 239_DATABASE_CIRCUIT_BREAKER_INTEGRATION_SYSTEM.md
│
├── failure containment
├── CLOSED / OPEN / HALF_OPEN
├── probing
└── endpoint isolation
│
✓ 240_DATABASE_FAILOVER_AND_RECOVERY_SYSTEM.md
│
├── authority
├── topology
├── writer failover
├── replica recovery
├── fencing
└── reintegration
│
✓ 241_DATABASE_RESOURCE_EXHAUSTION_PROTECTION_SYSTEM.md
  │
  ├── resource budgets
  ├── admission control
  ├── concurrency limits
  ├── bulkheads
  ├── backpressure
  ├── load shedding
  ├── tenant isolation
  ├── persistent worker protection
  └── resource accounting
```

La cadena de resiliencia queda conceptualmente:

```text
                     DATABASE WORK
                           │
                           ▼
                  RESOURCE GOVERNANCE
                           │
                     admission?
                           │
                           ▼
                       EXECUTION
                           │
                  ┌────────┴────────┐
                  │                 │
               SUCCESS           FAILURE
                                    │
                                    ▼
                         FAILURE CLASSIFICATION
                                    │
                     ┌──────────────┼──────────────┐
                     │              │              │
                   RETRY        CIRCUIT        FAILOVER
                     │           BREAKER            │
                     └──────────────┼───────────────┘
                                    │
                                    ▼
                                RECOVERY
                                    │
                                    ▼
                        RESOURCE RECONCILIATION
                                    │
                                    ▼
                             NORMAL SERVICE
```

Esto establece una propiedad importante:

> **La resiliencia de VoltStack Database no consistirá en “reintentar hasta que funcione”. Consistirá en conocer el estado disponible, preservar resultados inciertos, limitar la amplificación de fallos, aislar recursos degradados, cambiar de infraestructura únicamente cuando exista autoridad válida y mantener siempre el trabajo dentro de límites operativos conocidos.**

---

# 276. Siguiente bloque

Con `241_DATABASE_RESOURCE_EXHAUSTION_PROTECTION_SYSTEM.md` queda terminado el **Bloque 23 — Resilience**.

El siguiente bloque será:

```text
BLOCK 24 — PERFORMANCE

242_DATABASE_PERFORMANCE_ARCHITECTURE.md
243_DATABASE_QUERY_PERFORMANCE_SYSTEM.md
244_DATABASE_ORM_PERFORMANCE_SYSTEM.md
245_DATABASE_HYDRATION_PERFORMANCE_SYSTEM.md
246_DATABASE_METADATA_COMPILATION_SYSTEM.md
247_DATABASE_QUERY_COMPILATION_OPTIMIZATION_SYSTEM.md
248_DATABASE_MEMORY_MANAGEMENT_SYSTEM.md
249_DATABASE_RESOURCE_GOVERNANCE_SYSTEM.md
250_DATABASE_PERFORMANCE_BENCHMARK_SYSTEM.md
```

---

# 277. Siguiente documento

```text
242_DATABASE_PERFORMANCE_ARCHITECTURE.md
```

Este documento establecerá la arquitectura global de rendimiento de VoltStack Database y deberá conectar:

```text
Query AST
    ↓
Semantic Analysis
    ↓
Optimizer
    ↓
Planner
    ↓
Compiler
    ↓
Execution Engine
    ↓
Connection
    ↓
Database
```

con:

```text
ORM
Hydration
IdentityMap
UnitOfWork
Metadata
Cache
Prepared Statements
Result Processing
Relationships
Batching
Chunking
Lazy Collections
Bulk Operations
Distribution
Persistent Runtime
Telemetry
Resource Governance
```

La arquitectura deberá distinguir explícitamente:

```text
Performance
≠
Correctness

Performance
≠
Resource Governance

Optimization
≠
Semantic Change

Fast Query
≠
Efficient Application

Low SQL Latency
≠
Low End-to-End Latency

Few Queries
≠
Efficient Queries

Cache Hit
≠
Correct Result

Low Memory
≠
High Throughput

Benchmark
≠
Production Performance
```

y establecerá como principio central:

> **VoltStack Database optimizará el rendimiento únicamente después de preservar semántica, consistencia, aislamiento y seguridad; toda optimización deberá ser medible, explicable y reversible, y ninguna mejora de velocidad justificará introducir resultados incorrectos, estado compartido inseguro o consumo ilimitado de recursos.**