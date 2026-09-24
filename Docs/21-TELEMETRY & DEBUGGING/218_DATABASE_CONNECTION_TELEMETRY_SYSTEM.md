# 218_DATABASE_CONNECTION_TELEMETRY_SYSTEM.md

# VoltStack Quantum Database
## Connection Telemetry System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 218 — Connection Telemetry System  
**Bloque:** 21 — Telemetry and Debugging  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `217_DATABASE_QUERY_TELEMETRY_SYSTEM.md`  
**Siguiente documento:** `219_DATABASE_TRANSACTION_TELEMETRY_SYSTEM.md`

---

# 1. Propósito

`Connection Telemetry System` define cómo VoltStack observará el ciclo completo de las conexiones de base de datos.

El sistema deberá permitir diagnosticar:

```text
Connection Request
        ↓
Connection Resolution
        ↓
Routing
        ↓
Pool Selection
        ↓
Pool Wait
        ↓
Physical Connection Acquisition
        ↓
Creation / Reuse
        ↓
Authentication / Initialization
        ↓
Validation
        ↓
Active Use
        ↓
Transaction / Query Activity
        ↓
Reset
        ↓
Return to Pool / Close
```

La regla central será:

> **Connection Telemetry observa la adquisición, creación, reutilización, estado, utilización, reset y liberación de conexiones; nunca decide qué conexión seleccionar, cuándo hacer failover, cómo balancear carga, cómo administrar el pool ni cómo ejecutar queries.**

Formalmente:

```text
ConnectionSystem(Request)
        ↓
Connection

Telemetry
        ↓
Observation(ConnectionLifecycle)
```

Nunca:

```text
Telemetry
    ↓
SelectEndpoint()
RouteQuery()
OpenTransaction()
ResetConnection()
Failover()
```

---

# 2. Relación con Database Telemetry Architecture

La arquitectura general permanece:

```text
Database Telemetry
│
├── Query Telemetry
├── Connection Telemetry     ← este documento
├── Transaction Telemetry
├── ORM Telemetry
├── Query Profiler
├── Slow Query Detection
├── N+1 Telemetry
├── Debug Information
└── Developer Debug Toolbar
```

Connection Telemetry será la fuente canónica para métricas relacionadas con:

```text
connection acquisition
pool waiting
physical connection creation
connection reuse
connection lifetime
connection validation
connection reset
connection close
pool utilization
connection failures
connection leaks
```

---

# 3. Distinciones fundamentales

VoltStack deberá preservar:

```text
Logical Connection
≠
Physical Connection

Connection Request
≠
Connection

Connection
≠
Endpoint

Endpoint
≠
Pool

Pool
≠
Driver

Connection Role
≠
Endpoint Role

Connection Acquisition
≠
Connection Creation

Connection Reuse
≠
Connection Creation

Connection Lifetime
≠
Request Lifetime

Connection Failure
≠
Query Failure

Connection Health
≠
Database Health

Pool Saturation
≠
Database Saturation

Connection Telemetry
≠
Connection Manager

Connection Telemetry
≠
Health Check System

Connection Telemetry
≠
Failover System

Connection Telemetry
≠
Load Balancer
```

---

# 4. Logical Connection

VoltStack puede exponer una abstracción:

```php
$connection = DB::connection('primary');
```

pero esa referencia no necesariamente representa una conexión física abierta.

Puede representar:

```text
LogicalConnection
    ↓
ConnectionDefinition
    ↓
ConnectionResolver
    ↓
PhysicalConnection
```

Por tanto:

> **Obtener una conexión lógica no implica adquirir inmediatamente un socket o sesión física con el servidor de base de datos.**

---

# 5. Physical Connection

Una conexión física representa una sesión real con un servidor:

```text
Application Worker
       │
       ▼
Driver
       │
       ▼
TCP / Unix Socket
       │
       ▼
Database Server
       │
       ▼
Database Session
```

Puede mantenerse viva durante múltiples:

```text
queries
transactions
requests
jobs
```

si el runtime y el pool lo permiten.

---

# 6. Connection identity model

Se utilizarán identidades distintas.

```text
LogicalConnectionId
PhysicalConnectionId
ConnectionLeaseId
EndpointId
PoolId
```

---

# 7. LogicalConnectionId

```php
final readonly class LogicalConnectionId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Representa la identidad lógica de configuración/resolución.

Ejemplo conceptual:

```text
primary
analytics
reporting
tenant-database
```

No identifica un socket.

---

# 8. PhysicalConnectionId

```php
final readonly class PhysicalConnectionId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Representa una sesión física concreta.

Ejemplo:

```text
pc_000043
```

---

# 9. ConnectionLeaseId

Cuando un pool entrega temporalmente una conexión:

```php
final readonly class ConnectionLeaseId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Ejemplo:

```text
PhysicalConnection pc_43

Lease #1 → Request A
Lease #2 → Request B
Lease #3 → Job C
```

Entonces:

```text
PhysicalConnectionId
=
pc_43
```

pero:

```text
LeaseId1 ≠ LeaseId2 ≠ LeaseId3
```

---

# 10. Por qué separar PhysicalConnection y Lease

En runtimes persistentes:

```text
FrankenPHP Worker
RoadRunner Worker
OpenSwoole Worker
```

una conexión física puede sobrevivir a múltiples operaciones.

Por tanto:

```text
Physical Connection Lifetime
>
Request Lifetime
```

puede ser completamente válido.

---

# 11. EndpointId

```php
final readonly class EndpointId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Representa un endpoint lógico conocido por la topología.

No deberá contener necesariamente:

```text
hostname
IP
credentials
```

---

# 12. PoolId

```php
final readonly class PoolId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Permite correlacionar:

```text
pool size
active leases
idle connections
waiters
acquisition latency
creation rate
close rate
```

---

# 13. Identity hierarchy

Conceptualmente:

```text
PersistenceDomain
      │
      ▼
LogicalConnection
      │
      ▼
Replication Group
      │
      ▼
Endpoint
      │
      ▼
Connection Pool
      │
      ▼
Physical Connection
      │
      ▼
Connection Lease
      │
      ▼
Query / Transaction
```

---

# 14. Connection role

```php
enum ConnectionRole
{
    case WRITER;
    case REPLICA;
    case ANALYTICS;
    case MAINTENANCE;
    case MIGRATION;
    case ADMINISTRATION;
    case UNKNOWN;
}
```

---

# 15. Role ≠ endpoint

Un endpoint puede cambiar de función después de:

```text
promotion
failover
topology reconfiguration
```

Por ello:

```text
EndpointIdentity
≠
EndpointRole
```

---

# 16. ConnectionTelemetryContext

```php
final readonly class ConnectionTelemetryContext
{
    public function __construct(
        public ?LogicalConnectionId $logicalConnectionId,
        public ?PhysicalConnectionId $physicalConnectionId,
        public ?ConnectionLeaseId $leaseId,
        public ?PoolId $poolId,
        public ?EndpointId $endpointId,
        public ?PersistenceDomainId $domainId,
        public ?TenantId $tenantId,
        public ?ShardId $shardId,
        public ConnectionRole $role,
        public ConnectionTelemetryPolicy $policy,
    ) {}
}
```

---

# 17. Connection request

Una solicitud de conexión deberá poder representarse independientemente de su resultado.

```php
final readonly class ConnectionRequestId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 18. Connection request lifecycle

```text
REQUESTED
   │
   ▼
RESOLVING
   │
   ▼
ROUTING
   │
   ▼
WAITING
   │
   ▼
ACQUIRED
```

o:

```text
FAILED
TIMEOUT
CANCELLED
```

---

# 19. Connection acquisition

Connection acquisition representa:

```text
request connection
        ↓
wait if necessary
        ↓
obtain usable connection
```

No necesariamente implica crearla.

---

# 20. Connection acquisition ≠ creation

Ejemplo A:

```text
Acquire
↓
Idle pooled connection found
↓
Reuse
```

Ejemplo B:

```text
Acquire
↓
No idle connection
↓
Create physical connection
↓
Authenticate
↓
Initialize
↓
Lease
```

Ambos son acquisition.

Solo B incluye creation.

---

# 21. Acquisition timing

```text
T_acquire =
T_usable_connection
-
T_connection_request
```

---

# 22. Acquisition phases

Podrá descomponerse:

```text
Resolution
Routing
Pool Wait
Physical Creation
Authentication
Initialization
Validation
Lease
```

---

# 23. ConnectionTelemetryPhase

```php
enum ConnectionTelemetryPhase
{
    case RESOLUTION;
    case ROUTING;
    case POOL_WAIT;
    case CREATION;
    case TRANSPORT_CONNECT;
    case AUTHENTICATION;
    case INITIALIZATION;
    case VALIDATION;
    case ACQUISITION;
    case ACTIVE_USE;
    case RESET;
    case RELEASE;
    case CLOSE;
}
```

---

# 24. Phase timing

```php
final readonly class ConnectionPhaseTiming
{
    public function __construct(
        public ConnectionTelemetryPhase $phase,
        public Duration $duration,
    ) {}
}
```

---

# 25. Monotonic timing

Todas las duraciones deberán utilizar un reloj monotónico cuando esté disponible.

---

# 26. Pool wait

Una de las métricas más importantes será:

```text
db.connection.pool.wait.duration
```

Representa:

```text
time waiting for available capacity
```

---

# 27. Pool wait ≠ connection creation

Si:

```text
Pool Max = 20
Active = 20
Idle = 0
```

una solicitud puede esperar 400 ms sin que la creación de conexión sea lenta.

---

# 28. Diagnóstico correcto

Telemetry debe permitir distinguir:

```text
Query slow
```

de:

```text
Connection acquisition slow
```

y de:

```text
Pool saturated
```

---

# 29. Connection creation duration

Métrica:

```text
db.connection.creation.duration
```

puede incluir:

```text
DNS
TCP
TLS
authentication
session initialization
```

según las capacidades del driver.

---

# 30. Detailed transport timing

Cuando exista soporte:

```text
dns.duration
transport_connect.duration
tls.duration
authentication.duration
initialization.duration
```

podrán registrarse.

---

# 31. Unsupported phase

Si PDO/driver no expone una fase:

```text
UNKNOWN / NOT_OBSERVED
```

Nunca inventar:

```text
0 ms
```

---

# 32. Observation coverage

Podrá modelarse:

```php
enum ConnectionTelemetryCoverage
{
    case COMPLETE;
    case PARTIAL;
    case MINIMAL;
    case UNKNOWN;
}
```

---

# 33. Connection creation outcome

```php
enum ConnectionCreationOutcome
{
    case SUCCESS;
    case FAILED;
    case TIMEOUT;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 34. Connection acquisition outcome

Separadamente:

```php
enum ConnectionAcquisitionOutcome
{
    case ACQUIRED_REUSED;
    case ACQUIRED_NEW;
    case FAILED;
    case TIMEOUT;
    case CANCELLED;
    case POOL_EXHAUSTED;
    case UNKNOWN;
}
```

---

# 35. UNKNOWN preservation

Nunca:

```text
UNKNOWN → FAILED
```

solo para simplificar métricas.

---

# 36. Connection state

Modelo conceptual:

```php
enum PhysicalConnectionState
{
    case CREATING;
    case IDLE;
    case LEASED;
    case ACTIVE;
    case RESETTING;
    case VALIDATING;
    case CLOSING;
    case CLOSED;
    case BROKEN;
    case UNKNOWN;
}
```

---

# 37. State transitions

Ejemplo normal:

```text
CREATING
   ↓
IDLE
   ↓
LEASED
   ↓
ACTIVE
   ↓
RESETTING
   ↓
IDLE
```

---

# 38. Close path

```text
ACTIVE
  ↓
RESETTING
  ↓
CLOSING
  ↓
CLOSED
```

---

# 39. Broken path

```text
ACTIVE
  ↓
BROKEN
  ↓
CLOSED
```

---

# 40. State telemetry ≠ state authority

Connection Manager mantiene el estado funcional.

Telemetry únicamente lo observa.

---

# 41. Pool state

```php
final readonly class ConnectionPoolSnapshot
{
    public function __construct(
        public int $max,
        public int $total,
        public int $active,
        public int $idle,
        public int $waiting,
        public int $creating,
    ) {}
}
```

---

# 42. Snapshot consistency

Un Pool Snapshot puede ser aproximado bajo concurrencia.

Por ello deberá poder marcarse:

```text
EXACT
APPROXIMATE
UNKNOWN
```

---

# 43. Core pool metrics

```text
db.connection.pool.size
db.connection.pool.active
db.connection.pool.idle
db.connection.pool.waiting
db.connection.pool.utilization
db.connection.pool.wait.duration
db.connection.pool.acquire.timeout
```

---

# 44. Pool utilization

Conceptualmente:

```text
Utilization =
ActiveConnections
/
MaxConnections
```

si `max` está definido.

---

# 45. Utilization ≠ saturation

Un pool al:

```text
100%
```

no necesariamente está saturado si no existen waiters.

Por tanto:

```text
Utilization
≠
Saturation
```

---

# 46. Saturation evidence

Indicadores:

```text
active ≈ max
+
waiting > 0
+
increasing wait duration
```

son evidencia más fuerte.

---

# 47. Saturation telemetry

Podrá existir:

```text
db.connection.pool.saturation
```

como señal derivada.

La derivación deberá ser explícita.

---

# 48. Physical connection count

```text
db.connection.physical.open
```

gauge/up-down counter según provider.

---

# 49. Connections created

```text
db.connection.created
```

counter.

---

# 50. Connections closed

```text
db.connection.closed
```

counter.

---

# 51. Connections reused

```text
db.connection.reused
```

counter.

---

# 52. Connection churn

Alta tasa:

```text
created
+
closed
```

puede indicar:

```text
poor pooling
network instability
server-side timeout
bad lifetime policy
worker churn
```

---

# 53. Churn ≠ failure

Alta rotación no implica automáticamente error.

Telemetry deberá presentarla como evidencia.

---

# 54. Connection lifetime

```text
T_lifetime =
T_closed
-
T_created
```

---

# 55. Lease duration

```text
T_lease =
T_released
-
T_acquired
```

---

# 56. Lifetime ≠ lease duration

Una conexión física puede tener:

```text
Lifetime = 4 hours
```

mientras cada lease dura:

```text
20 ms
100 ms
3 s
```

---

# 57. Lease telemetry

Métricas:

```text
db.connection.lease.duration
db.connection.lease.active
```

---

# 58. Long lease

Un lease anormalmente largo puede indicar:

```text
long transaction
streaming query
unclosed cursor
application stall
connection leak
```

pero Telemetry no deberá decidir automáticamente cuál.

---

# 59. Connection leak

Conceptualmente:

```text
Lease Acquired
      ↓
Expected Scope Ends
      ↓
No Release
```

puede generar:

```text
CONNECTION_LEAK_CANDIDATE
```

---

# 60. Leak candidate ≠ proven leak

Una conexión puede estar legítimamente viva por:

```text
stream
transaction
long-running job
```

Por tanto:

```text
LongLease
≠
Leak
```

---

# 61. Leak detector

```php
interface ConnectionLeakDetector
{
    public function inspect(
        ActiveConnectionLease $lease,
        ConnectionTelemetryContext $context,
    ): ConnectionLeakAssessment;
}
```

---

# 62. Leak confidence

```php
enum LeakConfidence
{
    case LOW;
    case MEDIUM;
    case HIGH;
    case UNKNOWN;
}
```

---

# 63. Connection reset

Antes de devolver una conexión al pool, VoltStack puede requerir:

```text
rollback unfinished transaction
clear session variables
reset isolation
reset read-only state
reset temporary state
clear tenant-specific state
clear framework markers
```

---

# 64. Reset telemetry

Podrá observar:

```text
reset duration
reset success
reset failure
connection discarded after reset
```

---

# 65. Reset failure

Una conexión cuyo reset falla deberá normalmente:

```text
not return to reusable idle pool
```

La decisión pertenece al Connection Lifecycle System.

Telemetry registra:

```text
RESET_FAILED
```

---

# 66. Reset ≠ close

```text
Reset
≠
Close
```

Reset puede preparar la conexión para reutilización.

---

# 67. Reset ≠ validation

```text
Reset
≠
Validation
```

Reset limpia estado.

Validation comprueba usabilidad.

---

# 68. Connection validation

Puede incluir:

```text
driver health check
ping
socket state
server session validation
```

dependiendo de capacidades.

---

# 69. Validation telemetry

```text
db.connection.validation.duration
db.connection.validation.failure
```

---

# 70. Validation query caveat

Si la validación ejecuta:

```sql
SELECT 1
```

deberá clasificarse como:

```text
FRAMEWORK_INTERNAL
CONNECTION_VALIDATION
```

para evitar contaminar query counts de aplicación.

---

# 71. Health ≠ validation

Una conexión válida no implica:

```text
whole database cluster healthy
```

---

# 72. Connection health state

```php
enum ConnectionHealthState
{
    case HEALTHY;
    case DEGRADED;
    case UNHEALTHY;
    case UNKNOWN;
}
```

---

# 73. UNKNOWN health

Nunca:

```text
UNKNOWN = HEALTHY
```

---

# 74. Endpoint health

Endpoint health será una capa superior:

```text
many connection observations
+
health checks
+
failure history
↓
EndpointHealth
```

---

# 75. Telemetry feeds health

Connection Telemetry podrá alimentar Health System.

Pero:

```text
Connection Telemetry
≠
Health System
```

---

# 76. Connection failures

Clasificación:

```php
enum ConnectionFailureCategory
{
    case DNS;
    case NETWORK;
    case CONNECT_TIMEOUT;
    case TLS;
    case AUTHENTICATION;
    case AUTHORIZATION;
    case SERVER_UNAVAILABLE;
    case TOO_MANY_CONNECTIONS;
    case POOL_EXHAUSTED;
    case VALIDATION;
    case RESET;
    case PROTOCOL;
    case DRIVER;
    case CLOSED_UNEXPECTEDLY;
    case RESOURCE_EXHAUSTION;
    case UNKNOWN;
}
```

---

# 77. Failure stage

Además de categoría:

```php
enum ConnectionFailureStage
{
    case RESOLUTION;
    case ROUTING;
    case POOL_WAIT;
    case CONNECT;
    case AUTHENTICATE;
    case INITIALIZE;
    case VALIDATE;
    case ACTIVE_USE;
    case RESET;
    case RELEASE;
    case CLOSE;
    case UNKNOWN;
}
```

---

# 78. Category + stage

Ejemplo:

```text
category = NETWORK
stage    = CONNECT
```

es más útil que:

```text
connection failed
```

---

# 79. Error messages

No serán metric labels.

---

# 80. Credentials

Nunca deberán aparecer:

```text
username?
```

puede requerir policy.

Nunca:

```text
password
DSN with password
access token
client certificate private key
```

---

# 81. DSN protection

Un DSN podrá contener:

```text
host
port
database
username
password
options
```

Por tanto:

> **El DSN completo no es una representación segura de telemetry.**

---

# 82. Safe endpoint representation

Utilizar:

```php
final readonly class SafeEndpointTelemetry
{
    public function __construct(
        public EndpointId $endpointId,
        public ConnectionRole $role,
        public DatabaseSystem $system,
        public ?string $region,
    ) {}
}
```

---

# 83. Hostname capture

Hostname real:

```text
db-primary.internal.example
```

solo bajo policy.

---

# 84. IP addresses

También deberán tratarse como información operacional potencialmente sensible.

---

# 85. Port

Puede ser bounded, pero no siempre aporta valor como metric dimension.

---

# 86. Database name

Nombre físico de DB puede contener información sensible.

Preferir:

```text
PersistenceDomainId
```

o alias lógico.

---

# 87. Tenant correlation

Connection Telemetry puede correlacionar:

```text
TenantId
```

cuando la arquitectura multitenant esté instalada.

Pero no deberá usarlo como label métrico por default.

---

# 88. Tenant isolation telemetry

Puede diagnosticar:

```text
requested tenant
resolved persistence domain
selected connection
selected shard
```

en una representación segura.

---

# 89. Telemetry must not leak tenant topology

Una respuesta o dashboard público no deberá revelar topología de otros tenants.

---

# 90. Shard correlation

Puede registrarse:

```text
ShardId
```

en traces/diagnostics.

Metric labels solo bajo cardinality policy.

---

# 91. Writer telemetry

Una conexión writer puede indicar:

```text
connection.role=writer
```

como dimensión bounded.

---

# 92. Replica telemetry

Igualmente:

```text
connection.role=replica
```

---

# 93. Replica endpoint identity

No será necesario exponer cada replica como metric label.

---

# 94. Routing correlation

Flujo:

```text
Read/Write Routing
       ↓
RoutingDecision
       ↓
Connection Acquisition
       ↓
Connection Telemetry
```

---

# 95. Routing authority

Telemetry nunca cambiará:

```text
writer → replica
replica → writer
```

---

# 96. Failover correlation

Una adquisición puede estar relacionada con:

```text
FailoverOperationId
```

---

# 97. Failover hierarchy

```text
FailoverOperation
     │
     ├── Connection Attempt A
     ├── Connection Attempt B
     └── Connection Attempt C
```

---

# 98. Connection attempt

Cada intento tendrá:

```php
final readonly class ConnectionAttemptId
{
    public function __construct(
        public ConnectionRequestId $requestId,
        public int $attempt,
    ) {}
}
```

---

# 99. Retry connection attempt

Un request puede intentar:

```text
Endpoint A → failure
Endpoint B → success
```

Telemetry deberá preservar ambos.

---

# 100. Successful fallback must not erase failure

Nunca representar:

```text
overall success
```

como si Endpoint A nunca hubiese fallado.

---

# 101. Overall acquisition result

Puede ser:

```text
SUCCESS_PRIMARY
SUCCESS_AFTER_RETRY
SUCCESS_AFTER_FAILOVER
FAILED
TIMEOUT
CANCELLED
UNKNOWN
```

---

# 102. Failover decision ownership

Pertenece a:

```text
Failover System
```

no a telemetry.

---

# 103. Load balancing correlation

Telemetry podrá observar:

```text
candidate count
selected endpoint role
selection policy
```

si se expone de forma segura.

---

# 104. Load balancing telemetry ≠ load balancer

Nunca utilizar telemetry synchronous callbacks para seleccionar endpoint.

---

# 105. Transaction pinning

Una transacción puede fijar:

```text
TransactionId
↓
PhysicalConnectionId
```

---

# 106. Transaction connection affinity

Telemetry podrá diagnosticar:

```text
transaction.connection_pinned=true
```

---

# 107. TransactionId cardinality

No será metric label.

---

# 108. Connection switch inside transaction

Si ocurre una violación arquitectónica:

```text
Transaction
Connection A
↓
Connection B
```

podrá producir:

```text
TRANSACTION_CONNECTION_AFFINITY_VIOLATION
```

---

# 109. Query correlation

Query Telemetry deberá poder referenciar:

```text
PhysicalConnectionId
ConnectionLeaseId
```

en traces/diagnostics.

---

# 110. Canonical acquisition timing

Aunque Query Telemetry muestre:

```text
connection wait
```

el canonical owner será Connection Telemetry.

---

# 111. No double measurement

Query Telemetry deberá:

```text
reference / correlate
```

en lugar de crear otra medición independiente incompatible.

---

# 112. Queries per connection

Podrá registrarse internamente:

```text
query count per physical connection
```

para diagnóstico.

No necesariamente como metric series por ConnectionId.

---

# 113. Transactions per connection

Igualmente:

```text
transaction count
```

puede formar parte del lifecycle record.

---

# 114. Connection usage record

```php
final readonly class PhysicalConnectionUsageSummary
{
    public function __construct(
        public int $leases,
        public int $queries,
        public int $transactions,
        public int $failures,
        public Duration $lifetime,
    ) {}
}
```

---

# 115. Lifetime record cardinality

No se exportará automáticamente una serie métrica por conexión.

---

# 116. Connection spans

Operaciones importantes podrán producir:

```text
db.connection.acquire
db.connection.create
db.connection.reset
db.connection.close
```

---

# 117. Pool wait span

Solo cuando sea suficientemente relevante/sampled.

Evitar span explosion.

---

# 118. Query span relationship

Ejemplo:

```text
HTTP Request
│
├── db.connection.acquire
│
└── db.query
```

o, si la conexión ya existe:

```text
HTTP Request
└── db.query
```

---

# 119. Connection lifetime span prohibited

No deberá mantenerse un span abierto durante horas solo porque una conexión física permanezca viva.

---

# 120. Why

Una conexión física puede sobrevivir:

```text
10,000 requests
```

y un span de ese tamaño sería operacionalmente inadecuado.

---

# 121. Lifecycle records instead of lifetime spans

Para conexiones persistentes:

```text
metrics
+
bounded lifecycle events
+
diagnostic records
```

serán preferibles.

---

# 122. Connection telemetry events

Signals posibles:

```text
ConnectionRequested
ConnectionAcquisitionStarted
ConnectionPoolWaitStarted
ConnectionPoolWaitCompleted
PhysicalConnectionCreating
PhysicalConnectionCreated
ConnectionAcquired
ConnectionValidated
ConnectionReset
ConnectionReleased
PhysicalConnectionClosed
ConnectionAcquisitionFailed
ConnectionLeakSuspected
```

---

# 123. Telemetry signals ≠ domain events

No deberán confundirse con los eventos públicos de Database Event System.

---

# 124. Event noise

No todos los signals deberán exportarse.

El provider podrá recibir una representación agregada.

---

# 125. Metrics architecture

Métricas principales:

```text
db.connection.requests
db.connection.acquisitions
db.connection.acquisition.duration

db.connection.created
db.connection.creation.duration

db.connection.reused
db.connection.closed

db.connection.active
db.connection.idle

db.connection.lease.active
db.connection.lease.duration

db.connection.pool.waiting
db.connection.pool.wait.duration
db.connection.pool.utilization

db.connection.validation.duration
db.connection.validation.failures

db.connection.reset.duration
db.connection.reset.failures

db.connection.failures
```

---

# 126. Recommended bounded dimensions

```text
db.system
connection.role
connection.outcome
failure.category
failure.stage
pool.class
runtime
```

---

# 127. Dangerous dimensions

Evitar por default:

```text
connection.id
lease.id
request.id
transaction.id
tenant.id
query.id
raw hostname
IP
database name
DSN
```

---

# 128. Pool identity

Si la aplicación tiene un número pequeño y estático de pools:

```text
pool.name
```

podría ser bounded.

Debe pasar por cardinality policy.

---

# 129. Database system

Valores bounded:

```text
mysql
mariadb
postgresql
sqlite
```

---

# 130. SQLite special case

SQLite puede no utilizar:

```text
network endpoint
TCP
authentication
traditional connection pool
```

Connection Telemetry deberá soportar:

```text
NOT_APPLICABLE
```

sin fingir semánticas cliente-servidor.

---

# 131. In-memory SQLite

```text
sqlite::memory:
```

tiene lifecycle diferente.

No deberá inventarse:

```text
network latency
endpoint health
replica role
```

---

# 132. MySQL and MariaDB

Aunque compartan conceptos:

```text
MySQL
≠
MariaDB
```

seguirán siendo plataformas distintas.

---

# 133. PostgreSQL

Podrá exponer capacidades adicionales de sesión.

Telemetry deberá usar:

```text
Capability System
```

y no vendor conditionals dispersos.

---

# 134. Driver capabilities

Ejemplo:

```php
interface ConnectionTelemetryCapabilities
{
    public function canMeasureTransportConnect(): bool;

    public function canMeasureAuthentication(): bool;

    public function canObserveServerSessionId(): bool;

    public function canPingConnection(): bool;
}
```

---

# 135. Version ≠ capability

Nunca:

```php
if ($postgresVersion >= ...)
```

en Connection Telemetry cuando el Capability System pueda resolverlo.

---

# 136. Server session identifier

Algunos motores pueden proporcionar:

```text
backend PID
connection/thread ID
session ID
```

---

# 137. Server session ID sensitivity

Podrá ser útil en diagnostics, pero no como metric label.

---

# 138. Server session ID ≠ PhysicalConnectionId

VoltStack deberá mantener su propia identidad.

---

# 139. Connection initialization

Una conexión nueva puede requerir:

```text
charset
timezone
isolation defaults
application name
session variables
SQL mode
```

---

# 140. Initialization telemetry

Podrá medir:

```text
initialization.duration
initialization.failure
```

---

# 141. Initialization statement telemetry

Queries internas de inicialización deberán clasificarse:

```text
FRAMEWORK_INTERNAL
CONNECTION_INITIALIZATION
```

---

# 142. Session state

Connection Telemetry podrá registrar un fingerprint de estado:

```text
SessionStateFingerprint
```

cuando sea útil.

---

# 143. No session value leakage

No deberán registrarse arbitrariamente:

```text
session variables
temporary values
tenant-specific variables
```

---

# 144. Connection reset verification

Opcionalmente:

```text
Reset
↓
Validation
↓
Reusable
```

---

# 145. Reusable state

Podrá modelarse:

```php
enum ConnectionReuseState
{
    case REUSABLE;
    case REQUIRES_RESET;
    case MUST_CLOSE;
    case UNKNOWN;
}
```

---

# 146. UNKNOWN reuse state

No deberá interpretarse automáticamente como:

```text
REUSABLE
```

---

# 147. Pool ownership

Un pool debe saber:

```text
which physical connections it owns
```

Telemetry solo observa esa relación.

---

# 148. Cross-pool reuse

Una conexión física no deberá aparecer simultáneamente como propiedad activa de múltiples pools salvo que exista una arquitectura explícita que lo permita.

---

# 149. Pool invariants diagnostics

Telemetry podrá detectar inconsistencias observables como:

```text
active + idle > total
```

pero no repararlas.

---

# 150. Pool snapshot equation

Idealmente:

```text
Total
=
Idle
+
Leased
+
Resetting
+
Validating
+
OtherOwnedStates
```

dependiendo del modelo exacto.

---

# 151. Wait queue

Podrá medirse:

```text
current waiters
max observed waiters
wait duration
wait timeout
```

---

# 152. Waiter identity

Nunca será metric label.

---

# 153. Queue discipline

Telemetry podrá observar:

```text
FIFO
LIFO
PRIORITY
CUSTOM
```

si el Pool System lo expone.

No decidirá la disciplina.

---

# 154. Resource exhaustion

Podrá distinguir:

```text
POOL_LIMIT
PROCESS_FILE_DESCRIPTOR_LIMIT
DATABASE_CONNECTION_LIMIT
MEMORY_PRESSURE
NETWORK_RESOURCE
UNKNOWN
```

cuando exista evidencia.

---

# 155. Database connection limit

`too many connections` deberá clasificarse distinto de:

```text
local pool exhausted
```

---

# 156. Local saturation ≠ server saturation

```text
Pool Exhausted
≠
Database Max Connections Exhausted
```

---

# 157. Persistent runtime architecture

FrankenPHP cambia significativamente la semántica tradicional PHP-FPM.

En FPM:

```text
Request
↓
PHP process lifecycle
↓
connection often dies relatively soon
```

En FrankenPHP:

```text
Worker
├── Request A
├── Request B
├── Request C
└── ...
```

y una conexión puede sobrevivir entre requests.

---

# 158. FrankenPHP connection telemetry

Deberá distinguir:

```text
worker lifetime
request lifetime
physical connection lifetime
lease lifetime
```

---

# 159. Request state isolation

Aunque una conexión física se reutilice:

```text
Tenant A request
↓
reset
↓
Tenant B request
```

no deberá existir contaminación de estado.

---

# 160. Telemetry for reset isolation

Podrá correlacionarse:

```text
lease released
reset performed
next lease acquired
```

sin registrar datos sensibles.

---

# 161. Missing reset diagnostic

Si una política exige reset y no existe evidencia de ejecución:

```text
RESET_REQUIRED_BUT_NOT_OBSERVED
```

podrá generarse.

---

# 162. RoadRunner

Cada job deberá obtener:

```text
fresh logical operation context
```

aunque reutilice conexiones físicas.

---

# 163. OpenSwoole

Con concurrencia por coroutines:

```text
Coroutine A → Lease A
Coroutine B → Lease B
```

el estado deberá ser coroutine-safe.

---

# 164. No singleton current connection

Prohibido:

```php
private ?Connection $currentConnection;
```

como estado mutable global de telemetry.

---

# 165. Concurrent leases

Un PhysicalConnection normalmente no deberá tener dos leases concurrentes salvo que el driver/protocolo lo soporte explícitamente.

---

# 166. Concurrency violation

Podrá diagnosticarse:

```text
CONCURRENT_LEASE_VIOLATION
```

---

# 167. Streaming query

Una query streaming puede retener:

```text
ConnectionLease
```

durante todo el consumo.

---

# 168. Streaming correlation

```text
QueryId
↓
ResultCursor
↓
ConnectionLeaseId
```

---

# 169. Early cursor close

Deberá liberar el lease cuando la arquitectura lo permita.

---

# 170. Cursor leak

Puede provocar:

```text
connection lease leak
```

por lo que Query Telemetry y Connection Telemetry deberán correlacionarse.

---

# 171. Long transactions

Igualmente:

```text
TransactionId
↓
ConnectionLeaseId
```

puede explicar leases prolongados.

---

# 172. Lease reason

Podrá registrarse:

```php
enum ConnectionLeasePurpose
{
    case QUERY;
    case TRANSACTION;
    case STREAM;
    case MIGRATION;
    case MAINTENANCE;
    case HEALTH_CHECK;
    case INTERNAL;
    case UNKNOWN;
}
```

---

# 173. Purpose bounded

`ConnectionLeasePurpose` es apto para dimensiones métricas por ser un conjunto controlado.

---

# 174. Connection affinity

Algunas operaciones pueden requerir:

```text
same physical session
```

por ejemplo:

```text
transaction
temporary tables
session locks
server-side cursor
```

---

# 175. Affinity telemetry

Podrá registrar:

```text
connection.affinity=true
```

y razón bounded.

---

# 176. Connection close reasons

```php
enum ConnectionCloseReason
{
    case NORMAL;
    case MAX_LIFETIME;
    case IDLE_TIMEOUT;
    case RESET_FAILURE;
    case VALIDATION_FAILURE;
    case NETWORK_FAILURE;
    case SERVER_CLOSED;
    case FAILOVER;
    case WORKER_SHUTDOWN;
    case POOL_SHUTDOWN;
    case RESOURCE_PRESSURE;
    case UNKNOWN;
}
```

---

# 177. Close reason metrics

```text
db.connection.closed{
    reason
}
```

es viable si `reason` es bounded.

---

# 178. Max lifetime rotation

Cerrar una conexión por:

```text
MAX_LIFETIME
```

no es error.

---

# 179. Idle timeout

Igualmente:

```text
IDLE_TIMEOUT
```

puede ser política normal.

---

# 180. Unexpected close

Deberá distinguirse de close normal.

---

# 181. Server-side close

Ejemplo:

```text
server idle timeout
network reset
database restart
```

puede aparecer como:

```text
CLOSED_UNEXPECTEDLY
```

si la causa exacta no se conoce.

---

# 182. UNKNOWN cause

No inventar:

```text
server restart
```

sin evidencia.

---

# 183. Connection diagnostics

API conceptual:

```php
DB::telemetry()
    ->connections()
    ->explain($connectionId);
```

---

# 184. Example diagnostic

```text
CONNECTION TELEMETRY
────────────────────────────────

Physical Connection:
  pc_0042

Logical Connection:
  primary

Database:
  PostgreSQL

Role:
  REPLICA

Endpoint:
  endpoint_replica_02

Pool:
  read_pool

State:
  IDLE

Created:
  18 minutes ago

Lifetime:
  18m 14s

Leases:
  1,284

Queries:
  1,307

Transactions:
  8

Creation Duration:
  31 ms

Validation Failures:
  0

Reset Failures:
  0

Unexpected Closures:
  0

Last Lease:
  82 ms

Health:
  HEALTHY

Telemetry Coverage:
  PARTIAL
```

---

# 185. Pool diagnostic

```text
CONNECTION POOL
────────────────────────────────

Pool:
  read_pool

Role:
  REPLICA

Maximum:
  32

Total:
  28

Active:
  24

Idle:
  4

Waiting:
  3

Utilization:
  75%

P95 Acquisition:
  17 ms

P95 Pool Wait:
  9 ms

Created / min:
  1.2

Closed / min:
  0.9

Acquisition Timeouts:
  2

Warning:
  intermittent queue pressure
```

---

# 186. Diagnostic language

Evitar conclusiones absolutas como:

```text
database overloaded
```

cuando la evidencia solo muestra:

```text
local pool wait increased
```

---

# 187. Explainability

Preferir:

```text
Observed:
  Pool wait increased.

Evidence:
  P95 = 280ms
  Active = Max
  Waiters = 14

Interpretation:
  Local connection pool saturation is likely.
```

---

# 188. ConnectionTelemetryRecord

```php
final readonly class ConnectionTelemetryRecord
{
    public function __construct(
        public PhysicalConnectionId $connectionId,
        public ?LogicalConnectionId $logicalId,
        public ?PoolId $poolId,
        public ?EndpointId $endpointId,
        public ConnectionRole $role,
        public DatabaseSystem $system,
        public ConnectionCreationOutcome $creationOutcome,
        public Duration $creationDuration,
        public PhysicalConnectionUsageSummary $usage,
        public ConnectionCloseReason $closeReason,
        public ConnectionTelemetryCoverage $coverage,
    ) {}
}
```

---

# 189. Record creation

No será necesario crear un lifetime record completo para cada conexión en producción.

Puede limitarse a:

```text
metrics only
sampled records
diagnostic mode
```

---

# 190. ActiveConnectionTelemetry

Mientras viva:

```php
interface ActiveConnectionTelemetry
{
    public function leaseStarted(ConnectionLeaseId $lease): void;

    public function leaseFinished(ConnectionLeaseId $lease): void;

    public function resetStarted(): void;

    public function resetFinished(ConnectionResetOutcome $outcome): void;

    public function close(ConnectionCloseReason $reason): void;
}
```

---

# 191. Instrumentation failure

Si telemetry falla:

```text
database connection lifecycle
```

deberá continuar normalmente por default.

---

# 192. Fail-open telemetry

Política normal:

```text
Telemetry Failure
↓
Record internal diagnostic
↓
Continue Connection Operation
```

---

# 193. Security exception

Una configuración de telemetry insegura podrá ser rechazada durante bootstrap.

Eso es distinto de una falla runtime del exporter.

---

# 194. Sampling

Connection creation/failure son relativamente poco frecuentes y pueden instrumentarse ampliamente.

Lease events pueden ser mucho más frecuentes.

---

# 195. Lease sampling

Podrá utilizarse:

```text
metrics always
trace sampled
diagnostics conditional
```

---

# 196. Tail promotion

Un lease inicialmente no trazado podrá generar diagnóstico si:

```text
lease duration > threshold
```

si existe contexto mínimo suficiente.

---

# 197. Connection churn detection

Será desarrollado por análisis/profiling, no por instrumentation básica.

Telemetry provee:

```text
created count
closed count
lifetime distribution
close reasons
```

---

# 198. Pool pressure analysis

Igualmente:

```text
waiters
wait duration
utilization
timeouts
```

son evidencia.

---

# 199. Connection telemetry configuration

```php
return [
    'database' => [
        'telemetry' => [
            'connection' => [
                'enabled' => true,

                'metrics' => true,
                'tracing' => true,

                'acquisition' => true,
                'creation' => true,
                'leases' => true,
                'reset' => true,
                'validation' => true,

                'endpoint_identity' => false,
                'server_session_id' => false,

                'diagnostics' => [
                    'enabled' => false,
                    'track_lifetime' => false,
                    'detect_leaks' => true,
                ],
            ],
        ],
    ],
];
```

---

# 200. ConnectionTelemetryPolicy

```php
final readonly class ConnectionTelemetryPolicy
{
    public function __construct(
        public bool $metrics,
        public bool $tracing,
        public bool $trackAcquisition,
        public bool $trackCreation,
        public bool $trackLeases,
        public bool $trackReset,
        public bool $trackValidation,
        public bool $captureEndpointIdentity,
        public bool $captureServerSessionId,
        public ConnectionTelemetryBudget $budget,
    ) {}
}
```

---

# 201. Connection telemetry budget

```php
final readonly class ConnectionTelemetryBudget
{
    public function __construct(
        public int $maxDiagnosticRecords,
        public int $maxTrackedConnections,
        public int $maxTrackedLeases,
        public int $maxAttributesPerRecord,
    ) {}
}
```

---

# 202. Bounded active registry

Si diagnostics necesita rastrear leases:

```text
ActiveLeaseRegistry
```

deberá ser bounded.

---

# 203. Registry overflow

Si se excede:

```text
records_dropped
```

deberá incrementarse.

No agotar memoria para observar agotamiento de recursos.

---

# 204. Telemetry paradox prevention

Regla:

> **El sistema de telemetría nunca deberá convertirse en una causa significativa de agotamiento del mismo recurso que intenta observar.**

---

# 205. Hot path

Connection lease acquire/release puede ser extremadamente frecuente.

Debe evitar:

```text
stack traces
string formatting
large allocations
DSN parsing
credential inspection
filesystem operations
network calls
```

en el hot path.

---

# 206. Export async/buffered

Cuando el provider lo permita:

```text
Connection Lifecycle
↓
Lightweight Recording
↓
Telemetry Buffer
↓
Exporter
```

---

# 207. Backpressure

Si el exporter está saturado:

```text
drop / sample / aggregate
```

según policy.

No bloquear indefinidamente adquisición de conexiones.

---

# 208. Connection Telemetry testing

Deberá soportar:

```php
ConnectionTelemetry::assertAcquisitionCount(2);
```

---

# 209. Reuse assertion

```php
ConnectionTelemetry::assertConnectionReused();
```

---

# 210. Pool wait assertion

```php
ConnectionTelemetry::assertNoPoolWaitExceeded(
    Duration::milliseconds(50)
);
```

---

# 211. Leak assertion

```php
ConnectionTelemetry::assertNoOpenLeases();
```

---

# 212. Reset assertion

```php
ConnectionTelemetry::assertConnectionReset(
    $physicalConnectionId
);
```

---

# 213. Transaction affinity assertion

```php
ConnectionTelemetry::assertTransactionPinnedToSingleConnection(
    $transactionId
);
```

---

# 214. Fake connection clock

Testing deberá permitir simular:

```text
creation delay
pool wait
lease duration
idle timeout
max lifetime
```

determinísticamente.

---

# 215. Pool testing

Debe probarse:

```text
empty pool
idle reuse
pool full
waiting
timeout
connection creation failure
reset failure
validation failure
shutdown
```

---

# 216. Persistent runtime tests

Obligatorios:

```text
Request A → acquire → release
Request B → reuse same physical connection
```

sin compartir:

```text
tenant context
transaction context
session state
telemetry scope
```

---

# 217. Concurrency tests

Para OpenSwoole/RoadRunner:

```text
N concurrent operations
↓
N independent lease scopes
```

sin corrupción de counters/context.

---

# 218. Failure injection

Testing deberá poder simular:

```text
network disconnect
authentication failure
server unavailable
pool timeout
unexpected close
reset failure
```

---

# 219. Directory structure

```text
src/Quantum/Database/Telemetry/Connection/
│
├── Contract/
│   ├── ConnectionTelemetryInstrumentation.php
│   ├── ConnectionTelemetryScope.php
│   ├── ConnectionTelemetrySampler.php
│   └── ConnectionLeakDetector.php
│
├── Identity/
│   ├── LogicalConnectionId.php
│   ├── PhysicalConnectionId.php
│   ├── ConnectionRequestId.php
│   ├── ConnectionAttemptId.php
│   ├── ConnectionLeaseId.php
│   ├── EndpointId.php
│   └── PoolId.php
│
├── Context/
│   ├── ConnectionTelemetryContext.php
│   ├── ConnectionTelemetryContextFactory.php
│   └── ConnectionTelemetryContextResolver.php
│
├── Lifecycle/
│   ├── PhysicalConnectionState.php
│   ├── ConnectionLifecycleTracker.php
│   ├── ActiveConnectionTelemetry.php
│   ├── ConnectionReuseState.php
│   └── ConnectionCloseReason.php
│
├── Acquisition/
│   ├── ConnectionAcquisitionScope.php
│   ├── ConnectionAcquisitionOutcome.php
│   ├── ConnectionCreationOutcome.php
│   └── ConnectionAcquisitionRecord.php
│
├── Lease/
│   ├── ConnectionLeaseScope.php
│   ├── ConnectionLeasePurpose.php
│   ├── ActiveConnectionLease.php
│   └── ConnectionLeaseRecord.php
│
├── Pool/
│   ├── ConnectionPoolSnapshot.php
│   ├── ConnectionPoolTelemetry.php
│   ├── ConnectionPoolPressure.php
│   └── ConnectionPoolSnapshotConsistency.php
│
├── Phase/
│   ├── ConnectionTelemetryPhase.php
│   ├── ConnectionPhaseScope.php
│   └── ConnectionPhaseTiming.php
│
├── Failure/
│   ├── ConnectionFailureCategory.php
│   ├── ConnectionFailureStage.php
│   └── ConnectionFailureRecord.php
│
├── Health/
│   ├── ConnectionHealthState.php
│   ├── ConnectionValidationTelemetry.php
│   └── ConnectionLeakAssessment.php
│
├── Record/
│   ├── ConnectionTelemetryRecord.php
│   ├── PhysicalConnectionUsageSummary.php
│   └── SafeEndpointTelemetry.php
│
├── Metric/
│   ├── ConnectionMetricRecorder.php
│   ├── ConnectionMetricRegistry.php
│   └── ConnectionMetricDescriptor.php
│
├── Trace/
│   ├── ConnectionSpanFactory.php
│   ├── ConnectionSpanEnricher.php
│   └── ConnectionTracePolicy.php
│
├── Diagnostics/
│   ├── ConnectionTelemetryInspector.php
│   ├── ConnectionTelemetryExplainer.php
│   ├── ConnectionDiagnosticBuffer.php
│   └── ConnectionTelemetryLeakDetector.php
│
├── Policy/
│   ├── ConnectionTelemetryPolicy.php
│   ├── CompiledConnectionTelemetryPolicy.php
│   └── ConnectionTelemetryBudget.php
│
├── Capability/
│   └── ConnectionTelemetryCapabilities.php
│
├── Testing/
│   ├── RecordingConnectionTelemetry.php
│   ├── ConnectionTelemetryAssertions.php
│   └── FakeConnectionTelemetryClock.php
│
└── Exception/
    ├── ConnectionTelemetryException.php
    ├── ConnectionTelemetryStateException.php
    ├── ConnectionTelemetrySecurityException.php
    └── ConnectionTelemetryBudgetException.php
```

---

# 220. Architectural invariants

## DB-CTEL-001
Connection Telemetry observará conexiones sin controlarlas.

## DB-CTEL-002
Connection Telemetry no seleccionará endpoints.

## DB-CTEL-003
Connection Telemetry no realizará routing.

## DB-CTEL-004
Connection Telemetry no controlará pooling.

## DB-CTEL-005
Connection Telemetry no ejecutará failover.

## DB-CTEL-006
Connection Telemetry no ejecutará queries.

## DB-CTEL-007
Logical Connection será distinta de Physical Connection.

## DB-CTEL-008
Connection Request será distinta de Connection.

## DB-CTEL-009
Endpoint será distinto de Connection.

## DB-CTEL-010
Pool será distinto de Endpoint.

## DB-CTEL-011
Connection Acquisition será distinta de Connection Creation.

## DB-CTEL-012
Connection Reuse será distinta de Creation.

## DB-CTEL-013
Connection Lifetime será distinta de Lease Lifetime.

## DB-CTEL-014
PhysicalConnectionId será distinto de LeaseId.

## DB-CTEL-015
EndpointId será distinto de endpoint address.

## DB-CTEL-016
Connection Role será distinta de Endpoint Identity.

## DB-CTEL-017
Una conexión lógica no implicará conexión física inmediata.

## DB-CTEL-018
Cada adquisición tendrá identidad observable.

## DB-CTEL-019
Cada physical connection podrá tener múltiples leases secuenciales.

## DB-CTEL-020
Un lease será scoped.

## DB-CTEL-021
Acquisition lifecycle será explícito.

## DB-CTEL-022
Pool wait será medido separadamente.

## DB-CTEL-023
Pool wait será distinto de connection creation time.

## DB-CTEL-024
Creation phases desconocidas no tendrán duración falsa cero.

## DB-CTEL-025
Telemetry coverage será representable.

## DB-CTEL-026
UNKNOWN outcome permanecerá UNKNOWN.

## DB-CTEL-027
Connection state funcional pertenecerá al Connection System.

## DB-CTEL-028
Telemetry no será autoridad del state machine.

## DB-CTEL-029
Pool snapshots podrán declarar precisión.

## DB-CTEL-030
Pool utilization será distinta de saturation.

## DB-CTEL-031
100% utilization sin waiters no probará saturation.

## DB-CTEL-032
Connection churn será evidencia, no failure automática.

## DB-CTEL-033
Physical lifetime será distinta de request lifetime.

## DB-CTEL-034
Lease duration será observable.

## DB-CTEL-035
Long lease no será automáticamente leak.

## DB-CTEL-036
Leak diagnostics tendrán confidence.

## DB-CTEL-037
Reset será distinto de Close.

## DB-CTEL-038
Reset será distinto de Validation.

## DB-CTEL-039
Reset failure será observable.

## DB-CTEL-040
Validation failure será observable.

## DB-CTEL-041
Connection validation query será internal query.

## DB-CTEL-042
Connection health será distinta de database health.

## DB-CTEL-043
UNKNOWN health no será HEALTHY.

## DB-CTEL-044
Endpoint Health podrá consumir Connection Telemetry.

## DB-CTEL-045
Connection Telemetry será distinta del Health System.

## DB-CTEL-046
Connection failure tendrá categoría.

## DB-CTEL-047
Connection failure tendrá stage.

## DB-CTEL-048
Connection failure será distinta de Query failure.

## DB-CTEL-049
Error message no será metric label.

## DB-CTEL-050
Password nunca será telemetry attribute.

## DB-CTEL-051
Access tokens nunca serán telemetry attributes.

## DB-CTEL-052
Private keys nunca serán telemetry attributes.

## DB-CTEL-053
DSN completo no será telemetry attribute.

## DB-CTEL-054
Endpoint telemetry usará representación segura.

## DB-CTEL-055
Hostname será protegido por policy.

## DB-CTEL-056
IP será tratado como información operacional sensible.

## DB-CTEL-057
Database physical name podrá ser ocultado.

## DB-CTEL-058
TenantId no será metric label por default.

## DB-CTEL-059
ShardId estará sujeto a cardinality policy.

## DB-CTEL-060
ConnectionRole será bounded.

## DB-CTEL-061
Routing decision será observada después de resolverse.

## DB-CTEL-062
Telemetry no cambiará writer/replica routing.

## DB-CTEL-063
Failover attempts serán preservados.

## DB-CTEL-064
Successful failover no ocultará intentos fallidos.

## DB-CTEL-065
Failover decision pertenecerá al Failover System.

## DB-CTEL-066
Load balancing decision pertenecerá al Load Balancing System.

## DB-CTEL-067
Transaction connection affinity será observable.

## DB-CTEL-068
TransactionId no será metric label.

## DB-CTEL-069
QueryId no será metric label.

## DB-CTEL-070
Query Telemetry podrá correlacionarse con LeaseId.

## DB-CTEL-071
Connection acquisition timing tendrá un único canonical owner.

## DB-CTEL-072
No habrá double recording incompatible de acquisition latency.

## DB-CTEL-073
Queries per connection no crearán series por connection ID.

## DB-CTEL-074
Transactions per connection no crearán series por connection ID.

## DB-CTEL-075
Connection lifetime spans de horas estarán prohibidos por default.

## DB-CTEL-076
Persistent connection lifecycle usará metrics/records.

## DB-CTEL-077
Telemetry signals serán distintos de domain events.

## DB-CTEL-078
Metric dimensions serán bounded.

## DB-CTEL-079
PhysicalConnectionId no será metric label.

## DB-CTEL-080
LeaseId no será metric label.

## DB-CTEL-081
RequestId no será metric label.

## DB-CTEL-082
Raw hostname no será metric label por default.

## DB-CTEL-083
DSN nunca será metric label.

## DB-CTEL-084
SQLite no fingirá network telemetry.

## DB-CTEL-085
SQLite in-memory no fingirá endpoint telemetry.

## DB-CTEL-086
MySQL y MariaDB serán plataformas distintas.

## DB-CTEL-087
Version será distinta de Capability.

## DB-CTEL-088
Telemetry utilizará Platform Capabilities.

## DB-CTEL-089
Server session ID será distinto de PhysicalConnectionId.

## DB-CTEL-090
Server session ID no será metric label.

## DB-CTEL-091
Initialization será observable.

## DB-CTEL-092
Initialization statements serán internal queries.

## DB-CTEL-093
Session variables no se exportarán arbitrariamente.

## DB-CTEL-094
Connection reuse state será explícito.

## DB-CTEL-095
UNKNOWN reuse state no será REUSABLE.

## DB-CTEL-096
Pool ownership permanecerá responsabilidad del Pool System.

## DB-CTEL-097
Telemetry podrá diagnosticar pool invariant violations.

## DB-CTEL-098
Telemetry no reparará pool invariants.

## DB-CTEL-099
Wait queue será observable.

## DB-CTEL-100
Waiter identity no será metric label.

## DB-CTEL-101
Pool exhaustion será distinta de server connection exhaustion.

## DB-CTEL-102
Local saturation será distinta de server saturation.

## DB-CTEL-103
FrankenPHP podrá reutilizar physical connections entre requests.

## DB-CTEL-104
Request context no sobrevivirá entre requests.

## DB-CTEL-105
Tenant context no sobrevivirá entre leases.

## DB-CTEL-106
Transaction context no sobrevivirá entre leases.

## DB-CTEL-107
Required reset será observable.

## DB-CTEL-108
RoadRunner tendrá operation scope fresco por job.

## DB-CTEL-109
OpenSwoole tendrá telemetry coroutine-safe.

## DB-CTEL-110
No existirá singleton mutable currentConnection.

## DB-CTEL-111
Concurrent leases tendrán contextos independientes.

## DB-CTEL-112
Unsupported concurrent lease será diagnosticable.

## DB-CTEL-113
Streaming podrá retener connection lease.

## DB-CTEL-114
Cursor y connection lease serán correlacionables.

## DB-CTEL-115
Cursor leak podrá correlacionarse con connection leak.

## DB-CTEL-116
Transaction lease podrá explicar long lease.

## DB-CTEL-117
Lease purpose será bounded.

## DB-CTEL-118
Connection affinity será explícita.

## DB-CTEL-119
Close reason será estructurada.

## DB-CTEL-120
MAX_LIFETIME close no será failure.

## DB-CTEL-121
IDLE_TIMEOUT close no será failure automáticamente.

## DB-CTEL-122
Unexpected close será distinguible.

## DB-CTEL-123
UNKNOWN close cause permanecerá UNKNOWN.

## DB-CTEL-124
Connection diagnostics serán explainable.

## DB-CTEL-125
Diagnostics distinguirán evidencia de interpretación.

## DB-CTEL-126
Telemetry record podrá ser immutable.

## DB-CTEL-127
Lifetime records completos serán opt-in/sampled.

## DB-CTEL-128
Runtime telemetry failure no romperá connection operation por default.

## DB-CTEL-129
Unsafe telemetry config podrá fallar durante bootstrap.

## DB-CTEL-130
Lease tracing podrá samplearse.

## DB-CTEL-131
Connection creation podrá instrumentarse ampliamente.

## DB-CTEL-132
Connection failures podrán instrumentarse ampliamente.

## DB-CTEL-133
Tail promotion será posible con contexto mínimo.

## DB-CTEL-134
Connection churn analysis consumirá telemetry.

## DB-CTEL-135
Pool pressure analysis consumirá telemetry.

## DB-CTEL-136
Connection telemetry policy será compilable.

## DB-CTEL-137
Telemetry budget será bounded.

## DB-CTEL-138
Active lease registry será bounded.

## DB-CTEL-139
Registry overflow será observable.

## DB-CTEL-140
Telemetry no agotará memoria para observar resource exhaustion.

## DB-CTEL-141
Hot path evitará stack inspection.

## DB-CTEL-142
Hot path evitará DSN formatting.

## DB-CTEL-143
Hot path evitará credential inspection.

## DB-CTEL-144
Hot path evitará network exporter calls.

## DB-CTEL-145
Exporter backpressure no bloqueará indefinidamente connection acquisition.

## DB-CTEL-146
Telemetry podrá degradarse mediante sampling/drop.

## DB-CTEL-147
Testing podrá inspeccionar acquisitions.

## DB-CTEL-148
Testing podrá inspeccionar reuse.

## DB-CTEL-149
Testing podrá inspeccionar pool waits.

## DB-CTEL-150
Testing podrá detectar open leases.

## DB-CTEL-151
Testing podrá verificar resets.

## DB-CTEL-152
Testing podrá verificar transaction affinity.

## DB-CTEL-153
Timing tests usarán fake monotonic clock.

## DB-CTEL-154
Failure injection será soportada.

## DB-CTEL-155
Persistent runtime isolation será testeada.

## DB-CTEL-156
Concurrency isolation será testeada.

## DB-CTEL-157
Connection Telemetry permanecerá provider-agnostic.

## DB-CTEL-158
Connection Telemetry permanecerá driver-aware mediante capabilities.

## DB-CTEL-159
Connection Telemetry nunca será Database truth.

## DB-CTEL-160
Connection Telemetry nunca sustituirá Connection Manager.

## DB-CTEL-161
Connection Telemetry nunca sustituirá Pool Manager.

## DB-CTEL-162
Connection Telemetry nunca sustituirá Failover System.

## DB-CTEL-163
Connection Telemetry nunca sustituirá Health Check System.

## DB-CTEL-164
Connection Telemetry nunca sustituirá Resource Governance.

## DB-CTEL-165
Observation coverage será explícita cuando sea incompleta.

## DB-CTEL-166
Missing observation no será equivalente a zero.

## DB-CTEL-167
Telemetry no inventará causas de failure.

## DB-CTEL-168
Telemetry no inventará endpoint health.

## DB-CTEL-169
Telemetry no inventará server saturation.

## DB-CTEL-170
Toda conexión reutilizada deberá conservar separación entre physical lifetime y operation scope.

---

# 221. Modelo formal de adquisición

Sea:

```text
R
```

una solicitud de conexión.

Sea:

```text
P
```

un pool.

La adquisición puede representarse:

```text
Acquire(R, P)
→
Lease(C)
```

donde:

```text
C ∈ PhysicalConnections(P)
```

o:

```text
C = CreateConnection(E)
```

si debe crearse una nueva conexión.

---

# 222. Tiempo de adquisición

Formalmente:

```text
T_acquire(R)
=
T_resolve
+
T_route
+
T_wait
+
T_create_if_needed
+
T_validate
+
T_lease
```

pero únicamente para fases realmente ejecutadas.

Por tanto:

```text
T_create_if_needed = NOT_APPLICABLE
```

cuando se reutiliza una conexión existente.

---

# 223. Modelo de reutilización

Para una conexión física `C`:

```text
C
├── Lease L1
├── Lease L2
├── Lease L3
└── Lease Ln
```

con:

```text
PhysicalConnectionId(L1)
=
PhysicalConnectionId(L2)
=
...
=
PhysicalConnectionId(Ln)
```

pero:

```text
L1 ≠ L2 ≠ ... ≠ Ln
```

---

# 224. Modelo de lifetime

```text
Lifetime(C)
=
CloseTime(C)
-
CreationTime(C)
```

Mientras:

```text
LeaseDuration(Li)
=
ReleaseTime(Li)
-
AcquireTime(Li)
```

Por tanto:

```text
Σ LeaseDuration(Li)
≤
Lifetime(C)
```

normalmente.

---

# 225. Modelo de pool

Sea:

```text
Pmax
```

el máximo configurado.

En un instante `t`:

```text
Ptotal(t)
≤
Pmax
```

salvo arquitecturas explícitas de burst/overflow.

Y:

```text
Ptotal
=
Pidle
+
Pleasd
+
Presetting
+
Pvalidating
+
Pother
```

según los estados implementados.

---

# 226. Modelo de presión

Una señal conceptual de presión podría derivarse de:

```text
Pressure(P)
=
f(
    Utilization,
    Waiters,
    WaitDuration,
    AcquisitionTimeouts
)
```

pero:

> **Connection Telemetry recopila las variables; un sistema de diagnóstico o profiling interpreta la presión.**

---

# 227. Modelo de seguridad

La información de conexión exportable deberá derivarse:

```text
Connection
      ↓
Minimize
      ↓
Redact
      ↓
Cardinality Policy
      ↓
Telemetry View
```

Nunca:

```text
Connection
↓
Dump DSN
↓
Exporter
```

---

# 228. Modelo persistent runtime

Para dos requests:

```text
Request A
    │
    ▼
Lease LA
    │
    ▼
Physical Connection C
    │
    ▼
Reset
    │
    ▼
Request B
    │
    ▼
Lease LB
    │
    ▼
Physical Connection C
```

deberá cumplirse:

```text
PhysicalConnection(LA)
=
PhysicalConnection(LB)
```

mientras:

```text
OperationContext(LA)
≠
OperationContext(LB)
```

y:

```text
TenantContext(LA)
```

no podrá contaminar:

```text
TenantContext(LB)
```

---

# 229. Arquitectura final

```text
                    Application / ORM / Query Engine
                                │
                                ▼
                        Connection Request
                                │
                                ▼
                       Connection Resolver
                                │
                                ▼
                         Routing System
                                │
                                ▼
                         Endpoint Decision
                                │
                                ▼
                      Connection Pool Manager
                                │
                   ┌────────────┴─────────────┐
                   │                          │
                   ▼                          ▼
             Idle Connection            Pool Capacity
                   │                          │
                   │                          ▼
                   │                  Create Connection
                   │                          │
                   │                   Authenticate
                   │                          │
                   │                    Initialize
                   │                          │
                   └────────────┬─────────────┘
                                ▼
                            Validate
                                │
                                ▼
                         Connection Lease
                                │
                  ┌─────────────┼─────────────┐
                  ▼             ▼             ▼
                Query       Transaction     Stream
                  │             │             │
                  └─────────────┼─────────────┘
                                ▼
                             Release
                                │
                                ▼
                              Reset
                                │
                   ┌────────────┴────────────┐
                   ▼                         ▼
               Reusable                  Must Close
                   │                         │
                   ▼                         ▼
               Pool Idle                  Closed
```

Connection Telemetry rodeará estos boundaries:

```text
Request
Acquire
Wait
Create
Validate
Lease
Use
Reset
Release
Close
```

sin asumir control funcional sobre ninguno.

---

# 230. Relación con Query Telemetry

La correlación será:

```text
QueryOperationId
       │
       ▼
QueryId
       │
       ▼
ConnectionLeaseId
       │
       ▼
PhysicalConnectionId
       │
       ▼
EndpointId
```

Esto permitirá responder:

```text
¿Qué query fue lenta?
        ↓
¿Cuánto esperó por conexión?
        ↓
¿Qué conexión utilizó?
        ↓
¿Era nueva o reutilizada?
        ↓
¿Writer o replica?
        ↓
¿Existía presión en el pool?
        ↓
¿La conexión tuvo problemas?
```

sin mezclar responsabilidades entre sistemas.

---

# 231. Filosofía arquitectónica

VoltStack seguirá:

```text
Logical connection
over
premature physical connection

Lease identity
over
ambiguous connection usage

Pool wait visibility
over
generic "database latency"

Creation vs reuse
over
single acquisition metric

Explicit lifecycle
over
opaque PDO handles

Safe endpoint identity
over
raw DSNs

Bounded metrics
over
per-connection series

Evidence
over
speculation

Persistent-runtime isolation
over
request-lifetime assumptions

Capabilities
over
vendor conditionals

Observation
over
control
```

---

# 232. Regla maestra

> **VoltStack deberá poder explicar de dónde provino una conexión, cuánto costó obtenerla, si fue creada o reutilizada, cuánto tiempo permaneció arrendada, qué operaciones utilizaron esa sesión, cómo fue limpiada y por qué terminó cerrándose, sin convertir la telemetría en participante del routing, pooling, failover o lifecycle funcional.**

En forma compacta:

```text
Request
↓
Resolve
↓
Route
↓
Wait
↓
Acquire
↓
Use
↓
Reset
↓
Release
↓
Observe
↓
Correlate
↓
Diagnose
```

Nunca:

```text
Telemetry
↓
Route / Pool / Failover / Reset
```

---

# 233. Estado del Bloque 21

```text
BLOCK 21 — TELEMETRY AND DEBUGGING

✓ 216_DATABASE_TELEMETRY_ARCHITECTURE.md
✓ 217_DATABASE_QUERY_TELEMETRY_SYSTEM.md
✓ 218_DATABASE_CONNECTION_TELEMETRY_SYSTEM.md
○ 219_DATABASE_TRANSACTION_TELEMETRY_SYSTEM.md
○ 220_DATABASE_ORM_TELEMETRY_SYSTEM.md
○ 221_DATABASE_QUERY_PROFILER_SYSTEM.md
○ 222_DATABASE_SLOW_QUERY_DETECTION_SYSTEM.md
○ 223_DATABASE_N_PLUS_ONE_TELEMETRY_SYSTEM.md
○ 224_DATABASE_DEBUG_INFORMATION_SYSTEM.md
○ 225_DATABASE_DEVELOPER_DEBUG_TOOLBAR_INTEGRATION.md
```

---

# 234. Siguiente documento

```text
219_DATABASE_TRANSACTION_TELEMETRY_SYSTEM.md
```

El siguiente documento deberá definir la observabilidad completa de:

```text
Transaction Request
↓
Begin
↓
Isolation Resolution
↓
Connection Pinning
↓
Statements / Queries
↓
Nested Scopes
↓
Savepoints
↓
Retries
↓
Commit / Rollback
↓
Outcome
```

incluyendo:

```text
TransactionId
TransactionOperationId
TransactionAttemptId
logical vs physical transaction
transaction depth
isolation requested vs effective
connection affinity
transaction duration
time to first statement
idle-in-transaction time
savepoint telemetry
nested transaction telemetry
deadlock correlation
lock wait correlation
retry attempts
rollback reason
commit duration
rollback duration
UNKNOWN commit outcome
long-running transaction detection
transaction leak detection
queries per transaction
transaction resource budgets
distributed database boundaries
tenant/shard context
persistent runtime isolation
FrankenPHP
RoadRunner
OpenSwoole
metrics cardinality
traces
diagnostics
security
testing
```

bajo la regla:

> **Transaction Telemetry deberá observar la frontera transaccional completa y preservar con precisión SUCCESS, ROLLBACK, FAILED y UNKNOWN, sin adquirir autoridad para iniciar, confirmar, revertir, reintentar o modificar una transacción.**