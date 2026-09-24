# 236_DATABASE_CONNECTION_FAILURE_HANDLING_SYSTEM.md

# VoltStack Quantum Database
## Database Connection Failure Handling System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 236 — Database Connection Failure Handling System  
**Bloque:** 23 — Resilience  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `235_DATABASE_RESILIENCE_ARCHITECTURE.md`  
**Siguiente documento:** `237_DATABASE_QUERY_FAILURE_HANDLING_SYSTEM.md`

---

# 1. Propósito

Este documento define el sistema responsable de detectar, clasificar, contener y recuperar fallos relacionados con conexiones de base de datos dentro de `VoltStack/Quantum/Database`.

El sistema deberá cubrir:

- fallos de resolución DNS;
- conexión TCP/socket;
- negociación TLS;
- autenticación;
- selección de database;
- configuración inicial de sesión;
- adquisición desde pool;
- conexiones expiradas;
- conexiones stale;
- conexiones half-open;
- pérdida de conexión durante operaciones;
- reset remoto;
- server shutdown;
- network partitions;
- idle timeout;
- connection timeout;
- pool contamination;
- validación antes de reutilización;
- descarte de conexiones;
- reconnect;
- interacción con transactions;
- interacción con replicas;
- failover;
- circuit breaker;
- persistent runtimes;
- telemetría y diagnóstico.

La regla central será:

> **La recuperación de una conexión y la recuperación de una operación son problemas diferentes: VoltStack podrá reemplazar una conexión rota sin asumir que la operación que utilizaba esa conexión falló, tuvo éxito o puede repetirse.**

Formalmente:

```text
Connection Failure
≠
Operation Failure
```

y:

```text
Connection Recovery
≠
Operation Retry
```

---

# 2. Relación con documentos anteriores

Este sistema especializa principalmente:

```text
10_DATABASE_DRIVER_ARCHITECTURE.md
11_DATABASE_CONNECTION_SYSTEM.md
12_DATABASE_CONNECTION_MANAGER.md
13_DATABASE_CONNECTION_CONFIGURATION_AND_RESOLUTION.md
14_DATABASE_CONNECTION_POOLING_SYSTEM.md
15_DATABASE_CONNECTION_LIFECYCLE_SYSTEM.md
16_DATABASE_CONNECTION_STATE_AND_RESET_SYSTEM.md

84_DATABASE_QUERY_TIMEOUT_AND_CANCELLATION_SYSTEM.md
85_DATABASE_EXECUTION_ERROR_SYSTEM.md
86_DATABASE_EXECUTION_RETRY_SYSTEM.md

165_DATABASE_TRANSACTION_MANAGER_SYSTEM.md
166_DATABASE_TRANSACTION_CONTEXT_SYSTEM.md
170_DATABASE_TRANSACTION_RETRY_SYSTEM.md

176_DATABASE_READ_WRITE_CONNECTION_SYSTEM.md
177_DATABASE_READ_WRITE_ROUTING_SYSTEM.md
178_DATABASE_REPLICA_SYSTEM.md
179_DATABASE_REPLICA_LAG_AWARENESS_SYSTEM.md
180_DATABASE_STICKY_CONNECTION_SYSTEM.md
181_DATABASE_FAILOVER_SYSTEM.md
182_DATABASE_LOAD_BALANCING_SYSTEM.md

218_DATABASE_CONNECTION_TELEMETRY_SYSTEM.md

230_DATABASE_CONNECTION_SECURITY_SYSTEM.md

235_DATABASE_RESILIENCE_ARCHITECTURE.md
```

---

# 3. Responsabilidad

`Connection Failure Handling System` deberá responder:

```text
What happened to the connection?

Is the connection still usable?

What evidence exists?

Was an operation in flight?

What was the operation phase?

Can the connection return to the pool?

Should the endpoint health change?

Can another connection be acquired?

Does reconnect preserve required semantics?

Is the transaction still valid?

Is operation outcome known?

May execution continue?
```

---

# 4. Lo que no hará

Este sistema no deberá decidir por sí mismo:

```text
SQL generation
query semantics
ORM entity state
transaction replay
domain idempotency
authorization
shard ownership
business compensation
```

---

# 5. Separación fundamental

Debe distinguirse:

```text
Connection Health
Operation Outcome
Transaction Outcome
Endpoint Health
Topology Health
```

Son dimensiones diferentes.

---

# 6. Ejemplo crítico

```text
Application
    │
    │ UPDATE accounts ...
    ▼
Database
    │
    │ executes UPDATE
    ▼
Connection lost
```

El cliente observa:

```text
Connection = BROKEN
```

pero la operación puede ser:

```text
SUCCESS
FAILURE
UNKNOWN
```

Por tanto:

```text
Broken Connection
≠
Confirmed Failed Query
```

---

# 7. Arquitectura general

```text
Connection Request
       │
       ▼
Connection Manager
       │
       ▼
Connection Acquisition
       │
       ├── Existing pooled connection
       │
       └── New physical connection
       │
       ▼
Connection Validation
       │
       ▼
Connection Usage
       │
       ▼
Failure Observation
       │
       ▼
Connection Failure Classifier
       │
       ├── Establishment
       ├── DNS
       ├── Network
       ├── TLS
       ├── Authentication
       ├── Session
       ├── Timeout
       ├── Remote Reset
       ├── Server Shutdown
       ├── Pool
       └── Unknown
       │
       ▼
Connection State Evaluator
       │
       ├── HEALTHY
       ├── SUSPECT
       ├── BROKEN
       ├── CLOSED
       └── UNKNOWN
       │
       ▼
Containment
       │
       ├── KEEP
       ├── VALIDATE
       ├── QUARANTINE
       ├── DISCARD
       └── CLOSE
       │
       ▼
Recovery Decision
       │
       ├── NONE
       ├── RECONNECT
       ├── REACQUIRE
       ├── FAILOVER
       └── PROPAGATE
```

---

# 8. ConnectionFailure

Objeto canónico:

```php
final readonly class ConnectionFailure
{
    public function __construct(
        public ConnectionFailureId $id,
        public ConnectionFailureType $type,
        public ConnectionFailureStage $stage,
        public ConnectionHealthState $resultingState,
        public DatabaseFailureCategory $category,
        public DatabaseFailureEvidence $evidence,
        public ?Throwable $cause,
    ) {}
}
```

---

# 9. ConnectionFailureType

```php
enum ConnectionFailureType
{
    case DNS_RESOLUTION_FAILED;
    case NETWORK_UNREACHABLE;
    case CONNECTION_REFUSED;
    case CONNECTION_TIMEOUT;

    case TLS_NEGOTIATION_FAILED;
    case TLS_VERIFICATION_FAILED;

    case AUTHENTICATION_FAILED;
    case DATABASE_SELECTION_FAILED;

    case SESSION_INITIALIZATION_FAILED;

    case CONNECTION_RESET;
    case CONNECTION_ABORTED;
    case CONNECTION_CLOSED_BY_SERVER;
    case CONNECTION_LOST;

    case IDLE_CONNECTION_EXPIRED;
    case STALE_CONNECTION;

    case POOL_ACQUISITION_TIMEOUT;
    case POOL_EXHAUSTED;

    case SERVER_SHUTDOWN;
    case NETWORK_PARTITION;

    case PROTOCOL_ERROR;

    case UNKNOWN;
}
```

---

# 10. ConnectionFailureStage

```php
enum ConnectionFailureStage
{
    case RESOLUTION;
    case SOCKET_CONNECT;
    case TLS_HANDSHAKE;
    case AUTHENTICATION;
    case DATABASE_SELECTION;
    case SESSION_INITIALIZATION;

    case POOL_ACQUIRE;
    case VALIDATION;

    case PREPARE;
    case BIND;
    case SEND;
    case EXECUTE;
    case RECEIVE;

    case TRANSACTION_BEGIN;
    case TRANSACTION_COMMIT;
    case TRANSACTION_ROLLBACK;

    case RESET;
    case RELEASE;
    case CLOSE;

    case UNKNOWN;
}
```

---

# 11. Stage ≠ operation phase

`ConnectionFailureStage` describe dónde fue observado el problema de conexión.

`DatabaseOperationPhase` describe el progreso de la operación lógica.

Ambos pueden coexistir.

---

# 12. ConnectionHealthState

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

# 13. HEALTHY

Existe evidencia suficiente para considerar la conexión reutilizable bajo su contrato actual.

No significa:

```text
future operations cannot fail
```

---

# 14. SUSPECT

Existe evidencia de posible degradación, pero no necesariamente de inutilización definitiva.

Ejemplos:

```text
unexpected timeout
ambiguous protocol condition
failed lightweight validation
unusual socket condition
```

Una conexión `SUSPECT` no deberá regresar directamente al pool como healthy.

---

# 15. BROKEN

La conexión no deberá reutilizarse.

Ejemplos:

```text
socket reset
server closed connection
protocol desynchronization
failed transaction connection
unrecoverable TLS state
```

---

# 16. CLOSED

La conexión física ya fue cerrada.

---

# 17. UNKNOWN

No existe evidencia suficiente para demostrar que la conexión sigue siendo segura.

Default conservador:

```text
UNKNOWN
→
DO NOT RETURN TO NORMAL POOL
```

---

# 18. Connection state ≠ endpoint state

Una conexión rota puede deberse a:

```text
local socket problem
idle timeout
specific network path
server restart
endpoint failure
```

Por tanto:

```text
Broken Connection
≠
Unhealthy Endpoint
```

---

# 19. Endpoint health

Se integrará con:

```text
DatabaseEndpointHealth
```

del documento 235.

Estados:

```text
HEALTHY
DEGRADED
UNHEALTHY
UNKNOWN
```

---

# 20. Failure evidence

El sistema deberá capturar evidencia estructurada.

Ejemplo:

```php
final readonly class ConnectionFailureEvidence
{
    public function __construct(
        public ?string $sqlState,
        public ?int $vendorCode,
        public ?string $driverCode,
        public ?SocketFailureCode $socketCode,
        public ConnectionFailureStage $stage,
        public bool $operationPossiblySent,
        public bool $responsePossiblyReceived,
        public bool $transactionActive,
        public Instant $observedAt,
    ) {}
}
```

---

# 21. Sensitive evidence

Nunca incluir por default:

```text
password
TLS private key
raw DSN with credentials
authentication token
full sensitive query bindings
```

---

# 22. Connection establishment pipeline

```text
Connection Configuration
        │
        ▼
Resolve Endpoint
        │
        ▼
DNS / Address Resolution
        │
        ▼
Socket Connect
        │
        ▼
TLS Negotiation
        │
        ▼
Authentication
        │
        ▼
Database Selection
        │
        ▼
Session Initialization
        │
        ▼
Validation
        │
        ▼
READY
```

Cada etapa tendrá fallos semánticamente diferentes.

---

# 23. DNS failure

Ejemplos:

```text
hostname not found
temporary resolver failure
DNS timeout
invalid hostname
```

No todos son retryables.

---

# 24. DNS classification

Podrán distinguirse:

```text
TEMPORARY_RESOLUTION_FAILURE
PERMANENT_NAME_FAILURE
RESOLUTION_TIMEOUT
CONFIGURATION_ERROR
UNKNOWN
```

---

# 25. DNS retry

Un fallo temporal puede ser candidato a retry.

Un hostname inválido normalmente será:

```text
PERMANENT
```

---

# 26. TCP/socket connection failure

Ejemplos:

```text
connection refused
network unreachable
host unreachable
connect timeout
connection reset
broken pipe
```

---

# 27. Connection refused

Puede indicar:

```text
server down
server restarting
port closed
firewall
incorrect configuration
```

Por tanto:

```text
Connection Refused
≠
Always Transient
```

---

# 28. Connect timeout

Puede ser:

```text
network congestion
firewall drop
endpoint overload
network partition
incorrect route
```

---

# 29. TLS failures

Se distinguirán:

```text
TLS negotiation failure
certificate verification failure
certificate expired
hostname mismatch
unsupported protocol
cipher mismatch
```

---

# 30. Security-first TLS handling

Fallos como:

```text
certificate verification failed
hostname mismatch
```

no deberán convertirse automáticamente en:

```text
retry without TLS
```

---

# 31. No insecure fallback

Prohibido:

```text
TLS verification failed
→ reconnect with verify=false
```

---

# 32. Authentication failures

Ejemplos:

```text
invalid username
invalid password
expired credential
revoked credential
authentication plugin mismatch
```

---

# 33. Authentication retry

Credenciales inválidas no deberán provocar retries agresivos.

Esto evita:

```text
retry storm
account lockout
credential brute force behavior
```

---

# 34. Credential refresh

Cuando exista un proveedor dinámico de credenciales:

```text
Authentication Failed
        ↓
Credential Provider
        ↓
Refresh / Rotate
        ↓
New Connection Attempt
```

solo bajo política explícita.

---

# 35. Credential refresh ≠ blind retry

---

# 36. Session initialization

Después de establecer conexión, VoltStack puede necesitar configurar:

```text
timezone
charset
collation
isolation defaults
application name
search path
session variables
```

---

# 37. Initialization failure

Una conexión cuya inicialización requerida falla no deberá marcarse READY.

---

# 38. Partially initialized connection

```text
Physical Connection Established
+
Session Initialization Failed
=
Not Usable
```

---

# 39. Connection lifecycle

Estados conceptuales:

```text
NEW
 ↓
CONNECTING
 ↓
INITIALIZING
 ↓
READY
 ↓
IN_USE
 ↓
RELEASING
 ↓
IDLE
 ↓
CLOSED
```

Con fallos:

```text
SUSPECT
BROKEN
```

---

# 40. Lifecycle transitions

```text
READY → IN_USE
IN_USE → READY/IDLE
IN_USE → SUSPECT
IN_USE → BROKEN
SUSPECT → READY
SUSPECT → BROKEN
BROKEN → CLOSED
```

---

# 41. Invalid transition

```text
BROKEN → READY
```

no deberá ocurrir sobre el mismo recurso físico sin una reconstrucción explícita que conceptualmente produzca una nueva conexión.

---

# 42. Connection identity

Cada conexión física tendrá:

```text
ConnectionInstanceId
```

---

# 43. Reconnect creates new identity

```text
Connection A
    ↓ broken
Reconnect
    ↓
Connection B
```

Entonces:

```text
ConnectionInstanceId(A)
≠
ConnectionInstanceId(B)
```

---

# 44. Logical connection reference

La API superior puede mantener una referencia lógica, pero la identidad física debe permanecer distinguible.

---

# 45. Reconnect semantics

Reconnect significa:

```text
create another physical connection
```

No:

```text
restore exact previous server session
```

---

# 46. Session continuity

Después de reconnect no deberá asumirse preservado:

```text
transaction
temporary tables
session locks
prepared statements
server-side cursors
session variables changed dynamically
advisory locks
transaction isolation overrides
```

---

# 47. Safe initialization state

Solo configuración declarativa reconstruible podrá reaplicarse automáticamente.

Ejemplo:

```text
charset=utf8mb4
timezone=UTC
application_name=VoltStack
```

---

# 48. Dynamic session state

Estado introducido durante la operación deberá tratarse separadamente.

---

# 49. Connection reset

Antes de regresar una conexión al pool puede ser necesario:

```text
rollback active transaction
restore isolation
restore read-only state
clear session settings
release resources
```

---

# 50. Reset failure

Si reset falla:

```text
Connection
→ BROKEN
→ DISCARD
```

---

# 51. Reset failure ≠ ignore

Una conexión que no pudo regresar a baseline no deberá contaminar otro request.

---

# 52. Pool architecture

```text
Pool
├── IDLE HEALTHY
├── IN_USE
├── VALIDATING
├── SUSPECT/QUARANTINED
└── CLOSING
```

---

# 53. Broken connection and pool

Regla:

```text
BROKEN
→ never returned to reusable idle set
```

---

# 54. Pool contamination

Ocurre cuando una conexión con state residual o health incierta es reutilizada.

Ejemplos:

```text
open transaction
changed isolation
temporary session variable
broken socket
unconsumed protocol result
```

---

# 55. Pool contamination protection

Antes de release:

```text
Validate ownership
        ↓
Check transaction state
        ↓
Reset session
        ↓
Validate protocol
        ↓
Determine health
        ↓
Return or discard
```

---

# 56. Connection ownership

Una conexión en uso deberá tener ownership scoped.

```text
Request A
owns
Connection X
```

Otro request no podrá recibirla hasta release válido.

---

# 57. Leaked ownership

Persistent runtimes deberán detectar conexiones no liberadas al finalizar scope.

---

# 58. Request cleanup

Al finalizar request/job:

```text
active borrowed connections
        ↓
cleanup
        ↓
rollback if safely required
        ↓
reset
        ↓
return/discard
```

---

# 59. Cleanup failure

Nunca ocultará una conexión dañada dentro del pool.

---

# 60. Stale connection

Una conexión idle puede haber sido cerrada externamente.

```text
Pool
    │
    │ connection idle 10 min
    ▼
Firewall/server closes socket
```

El cliente puede no saberlo inmediatamente.

---

# 61. Stale ≠ broken until observed

Pero al adquirirla deberá existir una estrategia apropiada de validación.

---

# 62. Connection validation

Estrategias posibles:

```php
enum ConnectionValidationStrategy
{
    case NONE;
    case ON_ACQUIRE;
    case ON_RELEASE;
    case PERIODIC;
    case AFTER_IDLE_THRESHOLD;
    case DRIVER_NATIVE;
    case CUSTOM;
}
```

---

# 63. Validation cost

Validar cada conexión con una query puede agregar latencia.

Por tanto, la estrategia será configurable/capability-driven.

---

# 64. Validation query

Si se requiere una query de validación:

```text
SELECT 1
```

será una implementación de plataforma, no una regla universal hardcoded en capas superiores.

---

# 65. Driver-native validation

Si el driver proporciona una verificación fiable, podrá preferirse.

---

# 66. Validation success

Solo demuestra:

```text
connection appears usable at validation time
```

No:

```text
next operation is guaranteed to succeed
```

---

# 67. Validation failure

Produce:

```text
discard connection
```

y potencialmente:

```text
acquire another connection
```

---

# 68. Validation failure before operation

Si ninguna operación fue iniciada:

```text
Operation Outcome
=
NOT_STARTED
```

Por tanto, reacquisition puede ser segura.

---

# 69. Important distinction

```text
Connection replacement before operation
```

es mucho más sencillo que:

```text
connection replacement after operation was sent
```

---

# 70. Operation send boundary

El sistema deberá conocer, cuando sea posible:

```text
NOT_SENT
POSSIBLY_SENT
CONFIRMED_SENT
UNKNOWN
```

---

# 71. SendState

```php
enum DatabaseSendState
{
    case NOT_SENT;
    case POSSIBLY_SENT;
    case SENT;
    case UNKNOWN;
}
```

---

# 72. NOT_SENT

Si la conexión falla antes de enviar la operación:

```text
Database side effect
=
none attributable to this attempt
```

cuando la evidencia sea suficiente.

Esto puede permitir retry/reacquisition.

---

# 73. POSSIBLY_SENT

No deberá asumirse failure.

---

# 74. SENT

Aun sabiendo que fue enviado, puede desconocerse si fue ejecutado/committed.

---

# 75. UNKNOWN send state

Default conservador.

---

# 76. Receive boundary

También puede distinguirse:

```text
NO_RESPONSE
PARTIAL_RESPONSE
COMPLETE_RESPONSE
UNKNOWN
```

---

# 77. Partial result

Si una conexión se pierde durante lectura streaming:

```text
rows 1..500 received
connection lost
```

el resultado no deberá declararse completo.

---

# 78. Partial read handling

Dependiendo de la API:

```text
FAIL
PARTIAL
RESUMABLE
UNKNOWN
```

---

# 79. Ordinary query

Por default:

```text
partial result
→ failure
```

sin devolverlo como resultado completo.

---

# 80. Streaming API

Puede exponer que el consumidor ya recibió elementos antes del fallo.

Eso no vuelve el stream resumible.

---

# 81. Connection loss before query

```text
acquire connection
connection already stale
validation fails
```

Puede:

```text
discard
reacquire
execute
```

sin constituir query retry porque la query nunca comenzó.

---

# 82. Connection loss during prepare

Depende de si prepare es:

```text
client-side
server-side
```

y de capacidades del driver.

---

# 83. Prepared statement invalidation

Tras reconnect:

```text
server-side prepared statement
→ invalid
```

Debe prepararse nuevamente.

---

# 84. Prepared statement cache

El cache deberá estar ligado a:

```text
connection identity
```

cuando el statement sea server-side.

---

# 85. Connection loss during bind

Si binding ocurre localmente y nada fue enviado:

```text
NOT_SENT
```

puede determinarse.

No deberá asumirse universalmente.

---

# 86. Connection loss during send

Este es uno de los casos de mayor incertidumbre.

```text
send(query)
    │
    ├── some bytes sent
    └── socket failure
```

Outcome:

```text
UNKNOWN
```

salvo evidencia adicional.

---

# 87. Connection loss during receive

El servidor puede haber ejecutado completamente la operación.

Por tanto:

```text
receive failure
≠
execution failure
```

---

# 88. Read receive failure

Puede ser retryable como operación lógica bajo políticas específicas, pero la repetición puede observar datos diferentes.

---

# 89. Write receive failure

Especialmente peligroso:

```text
write executed
ACK lost
```

No blind retry.

---

# 90. Connection loss during BEGIN

Puede dejar incierto si la transacción fue iniciada en servidor.

La conexión rota deberá descartarse.

La nueva conexión no continúa ese transaction context.

---

# 91. Connection loss during active transaction

Default:

```text
Connection BROKEN
Transaction TAINTED/FAILED/UNKNOWN
```

según fase y evidencia.

---

# 92. No transaction migration

Prohibido:

```text
Transaction on Connection A
A lost
Acquire B
Continue statements
```

---

# 93. Transaction retry

Solo podrá ocurrir desde el boundary definido por:

```text
170_DATABASE_TRANSACTION_RETRY_SYSTEM.md
```

---

# 94. Connection loss during COMMIT

Caso crítico:

```text
COMMIT sent
server commits
ACK lost
```

Resultado:

```text
Transaction Outcome = UNKNOWN
```

---

# 95. UNKNOWN COMMIT rule

```text
UNKNOWN COMMIT
→
NO BLIND RETRY
```

---

# 96. Connection loss during ROLLBACK

También puede dejar outcome incierto.

Aunque una conexión cerrada normalmente provoque rollback del lado servidor, VoltStack no deberá generalizar esto sin garantías de plataforma/protocolo.

---

# 97. Server disconnect semantics

Las garantías del servidor podrán ser modeladas mediante capabilities/evidence.

---

# 98. Implicit rollback

Si una plataforma garantiza que cerrar una sesión activa revierte la transacción:

```text
connection definitely terminated
+
server guarantee
```

puede aportar evidencia.

Pero:

```text
network partition
```

no necesariamente demuestra que la sesión servidor terminó.

---

# 99. Half-open connection

Situación:

```text
client believes socket alive
server/network path no longer usable
```

Puede detectarse solo al siguiente I/O o mediante keepalive/validation.

---

# 100. TCP keepalive

Podrá ayudar a detectar conexiones muertas.

No constituye garantía de database-level health.

---

# 101. Database keepalive

Podrá existir como health/validation mechanism.

Debe ser bounded para no generar carga excesiva.

---

# 102. Idle timeout awareness

El pool podrá conocer:

```text
maxLifetime
idleTimeout
serverIdleTimeoutHint
```

---

# 103. Proactive retirement

Puede cerrar conexiones antes de alcanzar límites conocidos.

---

# 104. Connection lifetime

```php
final readonly class ConnectionLifetimePolicy
{
    public function __construct(
        public Duration $maxLifetime,
        public Duration $maxIdleTime,
        public Duration $validationAfterIdle,
    ) {}
}
```

---

# 105. Jittered retirement

En pools grandes, cerrar todas las conexiones al mismo tiempo puede crear reconnect storms.

Podrá aplicarse jitter al lifetime.

---

# 106. Connection storm

Después de restart:

```text
100 workers
×
20 connections
=
2000 simultaneous reconnects
```

Debe mitigarse.

---

# 107. Reconnect backoff

Nuevas conexiones podrán usar:

```text
exponential backoff
+
jitter
```

bajo políticas apropiadas.

---

# 108. Reconnect budget

Reconnect deberá estar acotado.

---

# 109. Connection acquisition budget

Una request no deberá esperar indefinidamente por una conexión.

---

# 110. PoolAcquireDeadline

```text
acquire timeout
≤
operation deadline
```

---

# 111. Pool exhaustion

Debe distinguirse de:

```text
database unavailable
```

---

# 112. PoolExhausted

Puede significar:

```text
too many concurrent operations
connection leak
slow queries
undersized pool
database saturation
```

---

# 113. Pool wait queue

Debe ser bounded.

---

# 114. Unbounded waiters prohibited

```text
Unlimited Requests
    ↓
Unlimited Pool Wait Queue
```

podría consumir memoria.

---

# 115. Backpressure

Ante saturación:

```text
WAIT bounded
REJECT
SHED
```

según policy.

---

# 116. Connection leak detection

El sistema podrá detectar:

```text
borrowed too long
scope ended while borrowed
ownership lost
```

---

# 117. Leak detection ≠ forced close always

Cerrar una conexión todavía usada puede causar corrupción lógica.

Debe respetarse ownership/context.

---

# 118. Persistent runtime risks

FrankenPHP, RoadRunner y OpenSwoole mantienen procesos vivos.

Una conexión dañada puede sobrevivir al request si no se limpia.

---

# 119. Request boundary cleanup

```text
Request End
    ↓
Inspect borrowed connections
    ↓
Resolve active transactions
    ↓
Reset
    ↓
Validate if required
    ↓
Return / Discard
```

---

# 120. Worker restart

Ante corrupción grave de infraestructura local, una policy superior puede recomendar reciclar worker.

Database core podrá reportar la condición, pero no deberá asumir control del runtime sin integración explícita.

---

# 121. Fork safety

Si algún runtime utiliza procesos/forks:

```text
physical DB connection
```

no deberá asumirse segura para compartir entre procesos.

---

# 122. Concurrent coroutine safety

En OpenSwoole:

```text
Connection
≠
Automatically coroutine-safe
```

Una conexión física no deberá ser utilizada simultáneamente por contextos incompatibles.

---

# 123. Connection lease

El pool podrá utilizar:

```php
final readonly class ConnectionLease
{
    public function __construct(
        public ConnectionInstanceId $connection,
        public ConnectionLeaseId $lease,
        public DatabaseScopeId $owner,
        public Instant $acquiredAt,
    ) {}
}
```

---

# 124. Lease prevents cross-request reuse

Mientras exista lease activo:

```text
connection unavailable to another owner
```

---

# 125. Lease invalidation

Si la conexión se rompe:

```text
lease
→ invalidated
```

---

# 126. Endpoint selection

Antes de abrir una conexión:

```text
Logical Connection
        ↓
Routing
        ↓
Topology
        ↓
Eligible Endpoint
        ↓
Circuit Breaker
        ↓
Physical Connect
```

---

# 127. Circuit OPEN

Si el endpoint está bajo circuit breaker OPEN:

```text
do not attempt normal connection
```

salvo probes HALF_OPEN.

---

# 128. Connection failure contribution to circuit

Podrán contribuir:

```text
connection refused
network unreachable
connect timeout
server shutdown
repeated resets
```

---

# 129. Failures that normally should not

Ejemplos:

```text
bad password
permission denied
invalid database name
application query error
```

No representan necesariamente endpoint health.

---

# 130. Circuit granularity

Preferentemente:

```text
EndpointId
+
ConnectionPurpose
```

cuando sea necesario.

---

# 131. Connection purpose

Puede ser:

```text
WRITER
REPLICA
MIGRATION
ADMIN
BACKGROUND
```

---

# 132. Endpoint failure

Si una replica falla:

```text
Replica A
→ unhealthy
```

podrá seleccionarse otra replica elegible.

---

# 133. Writer failure

Requiere autoridad topológica.

No basta:

```text
try another server
```

---

# 134. Failover integration

Se delegará al:

```text
240_DATABASE_FAILOVER_AND_RECOVERY_SYSTEM.md
```

---

# 135. Reconnect vs failover

```text
Reconnect:
Endpoint A → Endpoint A

Failover:
Endpoint A → Endpoint B
```

---

# 136. Reacquire

`Reacquire` significa obtener otra conexión mediante ConnectionManager.

Puede resultar en:

```text
same endpoint
or
different eligible endpoint
```

según routing/failover policy.

---

# 137. Reacquire ≠ retry

Puede ocurrir antes de que la operación haya iniciado.

---

# 138. Replica routing

Read operations podrán reacquire en otra replica si:

```text
read intent permits
consistency permits
transaction permits
sticky state permits
lag policy permits
```

---

# 139. Sticky connection

Después de write:

```text
read-your-writes
```

puede exigir writer.

No deberá violarse para recuperarse de una replica.

---

# 140. Transaction connection affinity

Una transaction activa impide routing arbitrario.

---

# 141. Shard affinity

La conexión también podrá estar ligada a:

```text
ShardId
```

---

# 142. Cross-shard recovery

No se deberá cambiar shard para recuperar una operación shard-bound.

---

# 143. Tenant context

La reconexión deberá preservar:

```text
TenantContext
PersistenceDomain
ShardContext
SecurityContext
```

---

# 144. Tenant connection resolution

Si cada tenant utiliza database diferente:

```text
Reconnect
```

debe volver a resolver únicamente dentro del contexto autorizado del mismo tenant.

---

# 145. No tenant drift

```text
Tenant A connection fails
→ Tenant B database
```

será arquitectónicamente imposible.

---

# 146. Connection security

Integración con:

```text
229_DATABASE_CREDENTIAL_SECURITY_SYSTEM.md
230_DATABASE_CONNECTION_SECURITY_SYSTEM.md
```

---

# 147. TLS downgrade prohibited

---

# 148. Credential logging prohibited

---

# 149. Connection string redaction

Diagnóstico deberá transformar:

```text
mysql://user:secret@db.internal/app
```

en representación segura, por ejemplo:

```text
mysql://***@db.internal/app
```

---

# 150. Error redaction

Los mensajes nativos del driver deberán pasar por sensitive-data protection antes de exposición.

---

# 151. Connection retry safety

Debe distinguirse:

```text
Retry Connecting
```

de:

```text
Retry Database Operation
```

---

# 152. Safe connect retry

Antes de establecer sesión:

```text
Connect attempt failed
```

puede repetirse dentro de budget.

No existe todavía query outcome que duplicar.

---

# 153. Unsafe operation retry

Después de:

```text
operation possibly sent
```

la nueva conexión no hace seguro el retry.

---

# 154. ConnectionRecoveryDecision

```php
enum ConnectionRecoveryAction
{
    case KEEP;
    case VALIDATE;
    case QUARANTINE;
    case DISCARD;
    case CLOSE;
    case RECONNECT;
    case REACQUIRE;
    case REQUEST_FAILOVER;
    case PROPAGATE;
}
```

---

# 155. ConnectionRecoveryContext

```php
final readonly class ConnectionRecoveryContext
{
    public function __construct(
        public ConnectionFailure $failure,
        public ConnectionHealthState $health,
        public DatabaseSendState $sendState,
        public DatabaseOperationPhase $operationPhase,
        public bool $transactionActive,
        public DatabaseConsistencyRequirement $consistency,
        public DatabaseDeadline $deadline,
        public RetryBudget $budget,
    ) {}
}
```

---

# 156. ConnectionRecoveryPolicy

```php
interface ConnectionRecoveryPolicy
{
    public function decide(
        ConnectionRecoveryContext $context
    ): ConnectionRecoveryDecision;
}
```

---

# 157. Decision example

```text
Failure:
    stale pooled connection

Stage:
    validation

Operation:
    NOT_STARTED

Transaction:
    none

Decision:
    DISCARD
    REACQUIRE
```

---

# 158. Another example

```text
Failure:
    connection reset

Stage:
    receive

Operation:
    WRITE

Send state:
    SENT

Outcome:
    UNKNOWN

Decision:
    DISCARD CONNECTION
    DO NOT RETRY OPERATION
    PROPAGATE UNKNOWN
```

---

# 159. Active transaction example

```text
Failure:
    network partition

Transaction:
    ACTIVE

Connection:
    BROKEN

Decision:
    DISCARD
    TAINT TRANSACTION
    NO MID-TX RECONNECT
```

---

# 160. ConnectionFailureClassifier

```php
interface ConnectionFailureClassifier
{
    public function classify(
        ConnectionFailureObservation $observation
    ): ConnectionFailure;
}
```

---

# 161. Platform classifiers

```text
MySqlConnectionFailureClassifier
MariaDbConnectionFailureClassifier
PostgreSqlConnectionFailureClassifier
SqliteConnectionFailureClassifier
```

---

# 162. SQLite special case

SQLite normalmente no utiliza una conexión de red remota como MySQL/PostgreSQL.

Sus fallos pueden involucrar:

```text
file unavailable
database locked
I/O failure
filesystem permissions
disk full
corrupt file
```

Por tanto, no se deberá forzar semántica TCP sobre SQLite.

---

# 163. Platform-specific evidence

El classifier deberá aprovechar:

```text
SQLSTATE
vendor error codes
driver-native errors
socket errors
```

---

# 164. Error string fallback

String matching podrá existir únicamente como fallback cuidadosamente encapsulado cuando el driver no exponga mejor estructura.

---

# 165. Unknown classification

Cuando no exista evidencia:

```text
type = UNKNOWN
health = UNKNOWN
```

---

# 166. Conservative containment

```text
UNKNOWN connection health
→ discard physical connection
```

puede ser preferible a reutilizar state posiblemente corrupto.

---

# 167. Discard cost

Descartar conexiones excesivamente puede producir:

```text
connection churn
latency
server load
```

Por tanto, classification debe ser precisa.

---

# 168. Quarantine

Una conexión `SUSPECT` podrá pasar a:

```text
QUARANTINE
```

antes de validation.

---

# 169. Quarantine pool

No formará parte del conjunto normal de conexiones disponibles.

---

# 170. Validation after suspect

```text
SUSPECT
   ↓
Validation
   ├── pass → HEALTHY
   └── fail → BROKEN
```

solo cuando la clase de fallo permita una validación fiable.

---

# 171. Protocol desynchronization

Si existe posibilidad de protocolo desincronizado:

```text
discard
```

será preferido.

No intentar:

```text
SELECT 1
```

sobre una conexión cuyo protocolo puede estar fuera de fase.

---

# 172. Unconsumed result

Algunos drivers requieren consumir/cerrar resultados antes de reutilizar conexión.

---

# 173. Result cleanup failure

Puede volver la conexión:

```text
BROKEN
```

o:

```text
SUSPECT
```

según driver.

---

# 174. Streaming result ownership

Mientras exista ResultCursor/StreamingResult activo:

```text
connection lease remains active
```

salvo que el driver materialice independientemente.

---

# 175. Early stream termination

Al romper un `foreach`:

```text
close result
release server cursor
restore connection state
return/discard
```

según capacidades.

---

# 176. Stream cleanup failure

La conexión no deberá regresar al pool sin validación apropiada.

---

# 177. Query cancellation impact

Una cancelación puede dejar:

```text
connection reusable
connection requires reset
connection broken
unknown
```

según plataforma/driver.

---

# 178. Cancellation classifier

La capa Connection deberá recibir evidencia del Execution Engine/Driver sobre el estado posterior.

---

# 179. Timeout impact

Timeout local no significa necesariamente que el servidor haya detenido la query.

---

# 180. Dangerous timeout reuse

Si el cliente deja de esperar pero la query continúa en servidor:

```text
connection
```

puede no estar disponible para nueva operación.

---

# 181. Timeout recovery

Podrá requerir:

```text
server cancellation
drain result
reset
or discard
```

---

# 182. Deadline expiration

No deberá iniciar nuevos reconnect attempts si no existe tiempo útil restante.

---

# 183. Connection attempt timeout

Cada connect attempt podrá tener un timeout, pero además estará limitado por el deadline lógico.

---

# 184. Health observations

Cada failure podrá producir:

```php
final readonly class ConnectionHealthObservation
{
    public function __construct(
        public EndpointId $endpoint,
        public ConnectionInstanceId $connection,
        public ConnectionHealthState $connectionHealth,
        public ?DatabaseEndpointHealth $endpointHealthHint,
        public Instant $observedAt,
        public Confidence $confidence,
    ) {}
}
```

---

# 185. Observation ≠ truth

Un único failure no siempre determina salud global del endpoint.

---

# 186. Failure aggregation

Endpoint health podrá considerar:

```text
multiple connections
multiple workers
time window
failure ratio
health probes
```

---

# 187. Local evidence

Un worker no deberá fingir conocimiento global si solo tiene evidencia local.

---

# 188. Telemetry

Integración con:

```text
218_DATABASE_CONNECTION_TELEMETRY_SYSTEM.md
```

---

# 189. Metrics

Ejemplos:

```text
db.connection.failures
db.connection.connect_failures
db.connection.resets
db.connection.stale_detected
db.connection.discarded
db.connection.reconnects
db.connection.validation_failures

db.pool.exhausted
db.pool.acquire_timeout
db.pool.wait_duration

db.connection.unknown_state
```

---

# 190. Dimensions

Cardinalidad limitada:

```text
platform
connection role
failure type
failure stage
recovery action
health state
```

---

# 191. No raw endpoint by default

Hosts dinámicos pueden generar cardinalidad/sensibilidad.

Usar:

```text
EndpointRole
EndpointAlias
```

cuando sea apropiado.

---

# 192. Trace lifecycle

```text
db.connection.acquire
db.connection.connect
db.connection.validate
db.connection.use
db.connection.reset
db.connection.release
```

---

# 193. Failure trace

Registrar:

```text
failure type
stage
connection state
operation phase
recovery action
```

---

# 194. Retry trace

Connect retries deberán distinguirse de query retries.

---

# 195. Example telemetry

```text
connection.connect
    attempt=2
    role=REPLICA
    failure=CONNECTION_REFUSED
    recovery=BACKOFF_RETRY
```

---

# 196. Debug information

Ejemplo:

```text
Connection Failure
------------------
Connection: conn_01H...
Role: REPLICA
Stage: VALIDATION
Failure: STALE_CONNECTION
Health: BROKEN

Operation
---------
Phase: NOT_STARTED
Send state: NOT_SENT
Transaction: NONE

Recovery
--------
Discard: YES
Reacquire: YES
Operation retry: NOT_APPLICABLE
```

---

# 197. Critical debug example

```text
Connection Failure
------------------
Stage: RECEIVE
Failure: CONNECTION_RESET
Health: BROKEN

Operation
---------
Type: WRITE
Send state: SENT
Transaction: ACTIVE

Outcome
-------
UNKNOWN

Recovery
--------
Discard connection: YES
Reconnect transaction: NO
Retry statement: NO
Transaction context: TAINTED
```

---

# 198. Developer exception hierarchy

```text
DatabaseConnectionException
├── ConnectionEstablishmentException
│   ├── DnsResolutionException
│   ├── NetworkUnreachableException
│   ├── ConnectionRefusedException
│   └── ConnectionTimeoutException
│
├── ConnectionSecurityException
│   ├── TlsNegotiationException
│   ├── TlsVerificationException
│   └── AuthenticationException
│
├── ConnectionInitializationException
├── ConnectionValidationException
├── ConnectionLostException
├── ConnectionResetException
├── StaleConnectionException
├── PoolExhaustedException
├── PoolAcquireTimeoutException
└── UnknownConnectionFailureException
```

---

# 199. Exception ≠ recovery decision

Lanzar una excepción tipada no determina automáticamente:

```text
retry
reconnect
failover
```

---

# 200. Public developer API

La mayor parte del sistema deberá funcionar transparentemente.

Ejemplo:

```php
DB::connection('primary')->select(...);
```

pero podrán existir APIs diagnósticas:

```php
DB::connection('primary')->health();
```

o:

```php
DB::diagnostics()->connections();
```

sin exponer detalles internos inseguros.

---

# 201. Configuration

Ejemplo conceptual:

```php
'database' => [
    'connections' => [
        'primary' => [
            'connect_timeout' => '3s',

            'pool' => [
                'max_connections' => 20,
                'acquire_timeout' => '2s',
                'max_idle_time' => '5m',
                'max_lifetime' => '30m',
            ],

            'validation' => [
                'strategy' => 'after_idle_threshold',
                'after_idle' => '30s',
            ],

            'reconnect' => [
                'max_attempts' => 3,
                'backoff' => 'exponential_jitter',
            ],
        ],
    ],
];
```

---

# 202. Configuration safety

No permitir configuraciones como:

```text
max_attempts = infinite
acquire_timeout = infinite
pool_wait_queue = unbounded
```

por default.

---

# 203. Connection profile compilation

La configuración deberá compilarse a objetos immutable:

```text
ConnectionFailurePolicy
ConnectionValidationPolicy
ConnectionLifetimePolicy
ConnectionRecoveryPolicy
```

---

# 204. Runtime mutation

Cambiar políticas globales durante una request estará desaconsejado/prohibido según scope.

---

# 205. Testing architecture

Deberán existir pruebas específicas para cada failure stage.

---

# 206. DNS tests

```text
temporary DNS failure
permanent DNS failure
DNS timeout
```

---

# 207. TCP tests

```text
refused
timeout
reset
broken pipe
unreachable
```

---

# 208. TLS tests

```text
invalid certificate
expired certificate
hostname mismatch
protocol mismatch
```

y comprobar:

```text
no insecure downgrade
```

---

# 209. Authentication tests

Comprobar:

```text
invalid credentials
→ no aggressive retry
```

---

# 210. Stale pool test

```text
connection enters pool
server closes it
connection acquired
validation detects stale
discard
new connection acquired
```

---

# 211. Reset contamination test

```text
Request A changes session
reset fails
connection discarded
Request B never receives it
```

---

# 212. Transaction loss test

```text
BEGIN
UPDATE
connection lost
```

Debe comprobar:

```text
transaction cannot continue on another connection
```

---

# 213. Unknown write test

```text
write sent
server executes
ACK lost
```

Debe producir:

```text
connection = BROKEN
operation = UNKNOWN
no blind retry
```

---

# 214. Unknown commit test

```text
COMMIT sent
server commits
ACK lost
```

Debe producir:

```text
transaction = UNKNOWN
```

---

# 215. Streaming failure test

```text
yield 500 rows
connection lost
```

Debe:

```text
close source
mark connection broken
surface iteration failure
not fake complete stream
```

---

# 216. Pool exhaustion test

Verificar:

```text
bounded waiters
bounded timeout
no unbounded memory growth
```

---

# 217. Reconnect storm test

Muchos workers deberán aplicar:

```text
backoff + jitter
```

cuando corresponda.

---

# 218. Persistent runtime isolation test

```text
Request A
    broken connection

Request B
    never receives broken connection
```

---

# 219. Tenant isolation test

```text
Tenant A connection fails
reconnect
→ still Tenant A persistence domain
```

---

# 220. Replica failover test

```text
Replica A fails
Replica B eligible
```

solo podrá utilizar B si se preservan las reglas de consistencia.

---

# 221. Writer authority test

```text
Writer A fails
Replica B reachable
```

no significa:

```text
write to B
```

sin promoción/authority evidence.

---

# 222. Failure injection hooks

Testing podrá definir:

```php
$failures->onConnection(
    ConnectionFailureStage::RECEIVE,
    ConnectionFailureType::CONNECTION_RESET,
);
```

---

# 223. Deterministic timing

Reconnect/backoff deberá usar:

```text
Clock abstraction
Random abstraction
```

para pruebas reproducibles.

---

# 224. Directorios propuestos

```text
src/Quantum/Database/Resilience/Connection/
│
├── Contract/
│   ├── ConnectionFailureClassifier.php
│   ├── ConnectionRecoveryPolicy.php
│   ├── ConnectionValidator.php
│   └── ConnectionHealthObserver.php
│
├── Failure/
│   ├── ConnectionFailure.php
│   ├── ConnectionFailureId.php
│   ├── ConnectionFailureType.php
│   ├── ConnectionFailureStage.php
│   ├── ConnectionFailureEvidence.php
│   └── ConnectionFailureObservation.php
│
├── Health/
│   ├── ConnectionHealthState.php
│   ├── ConnectionHealthObservation.php
│   ├── ConnectionHealthEvaluator.php
│   └── ConnectionHealthRegistry.php
│
├── Validation/
│   ├── ConnectionValidationStrategy.php
│   ├── ConnectionValidationPolicy.php
│   ├── ConnectionValidationResult.php
│   ├── DriverConnectionValidator.php
│   └── PlatformConnectionValidator.php
│
├── Recovery/
│   ├── ConnectionRecoveryAction.php
│   ├── ConnectionRecoveryContext.php
│   ├── ConnectionRecoveryDecision.php
│   ├── ConnectionRecoveryCoordinator.php
│   └── ConnectionReconnector.php
│
├── Lifecycle/
│   ├── ConnectionLifetimePolicy.php
│   ├── ConnectionRetirementPolicy.php
│   └── ConnectionLease.php
│
├── Pool/
│   ├── PoolFailureHandler.php
│   ├── PoolExhaustionPolicy.php
│   ├── PoolAcquireDeadline.php
│   ├── ConnectionQuarantine.php
│   └── ConnectionLeakDetector.php
│
├── State/
│   ├── DatabaseSendState.php
│   ├── DatabaseReceiveState.php
│   └── ConnectionSessionState.php
│
├── Classifier/
│   ├── CompositeConnectionFailureClassifier.php
│   ├── MySqlConnectionFailureClassifier.php
│   ├── MariaDbConnectionFailureClassifier.php
│   ├── PostgreSqlConnectionFailureClassifier.php
│   └── SqliteConnectionFailureClassifier.php
│
├── Telemetry/
│   └── ConnectionResilienceTelemetry.php
│
├── Testing/
│   ├── ConnectionFailureInjector.php
│   ├── FakeConnectionValidator.php
│   └── ConnectionFailureScenario.php
│
└── Exception/
    ├── DatabaseConnectionException.php
    ├── ConnectionEstablishmentException.php
    ├── DnsResolutionException.php
    ├── NetworkUnreachableException.php
    ├── ConnectionRefusedException.php
    ├── ConnectionTimeoutException.php
    ├── TlsNegotiationException.php
    ├── TlsVerificationException.php
    ├── AuthenticationException.php
    ├── ConnectionInitializationException.php
    ├── ConnectionValidationException.php
    ├── ConnectionLostException.php
    ├── ConnectionResetException.php
    ├── StaleConnectionException.php
    ├── PoolExhaustedException.php
    └── PoolAcquireTimeoutException.php
```

---

# 225. Flujo: stale pooled connection

```text
Acquire
  │
  ▼
Pooled Connection
  │
  ▼
Idle threshold exceeded?
  │
  ├── NO ─────► Use
  │
  ▼
Validate
  │
  ├── PASS ───► Use
  │
  ▼
FAIL
  │
  ▼
Mark BROKEN
  │
  ▼
Discard
  │
  ▼
Budget available?
  │
  ├── NO ─────► Acquisition Failure
  │
  ▼
Acquire another
```

La operación todavía no comenzó.

Por tanto:

```text
this is connection reacquisition,
not query retry
```

---

# 226. Flujo: connection lost during read

```text
READ
 │
 ▼
Send
 │
 ▼
Execute
 │
 ▼
Receive rows
 │
 ▼
Connection Lost
 │
 ├── Connection → BROKEN
 │
 ▼
Was result complete?
 │
 ├── YES ─► Result may complete
 │
 └── NO
      │
      ▼
 Partial/Unknown Result
      │
      ▼
 Query Recovery Policy
```

---

# 227. Flujo: connection lost during write

```text
WRITE
 │
 ▼
Send
 │
 ▼
Connection Lost
 │
 ▼
Send State?
 │
 ├── NOT_SENT
 │      │
 │      ▼
 │   retry analysis
 │
 ├── POSSIBLY_SENT
 │      │
 │      ▼
 │    UNKNOWN
 │
 ├── SENT
 │      │
 │      ▼
 │   Outcome known?
 │
 └── UNKNOWN
        │
        ▼
      UNKNOWN
```

---

# 228. Flujo: transaction connection failure

```text
Transaction ACTIVE
       │
       ▼
Connection Failure
       │
       ▼
Connection usable?
       │
       ├── YES ─► transaction-specific handling
       │
       └── NO
            │
            ▼
       Connection BROKEN
            │
            ▼
       Transaction outcome?
            │
      ┌─────┼─────┐
      │     │     │
   FAILURE SUCCESS UNKNOWN
      │     │     │
      └─────┼─────┘
            ▼
     Transaction Manager
            │
            ▼
     Retry whole boundary?
```

Nunca:

```text
Acquire another connection
→ continue old transaction
```

---

# 229. Flujo: release to pool

```text
Operation Ends
      │
      ▼
Connection Lease Release
      │
      ▼
Active Transaction?
      │
      ▼
Resolve/Reject
      │
      ▼
Reset Required State
      │
      ▼
Reset Successful?
      │
   ┌──┴──┐
  YES    NO
   │      │
   ▼      ▼
Health   BROKEN
Check     │
   │      ▼
   │    Discard
   ▼
Reusable?
 ┌─┴─┐
YES NO
 │   │
 ▼   ▼
Pool Discard
```

---

# 230. Invariantes arquitectónicas

## DB-CONN-FAIL-001
Connection Failure será distinto de Operation Failure.

## DB-CONN-FAIL-002
Connection Recovery será distinto de Operation Retry.

## DB-CONN-FAIL-003
Broken Connection no implicará Confirmed Failed Query.

## DB-CONN-FAIL-004
Connection health será distinta de endpoint health.

## DB-CONN-FAIL-005
Connection health será distinta de transaction outcome.

## DB-CONN-FAIL-006
Connection health será distinta de operation outcome.

## DB-CONN-FAIL-007
Connection failures serán normalizados.

## DB-CONN-FAIL-008
Raw driver evidence útil será preservada.

## DB-CONN-FAIL-009
Sensitive connection evidence será redactada.

## DB-CONN-FAIL-010
Failure stage será explícito.

## DB-CONN-FAIL-011
Operation phase será preservada independientemente.

## DB-CONN-FAIL-012
DNS failures serán distinguibles.

## DB-CONN-FAIL-013
Network failures serán distinguibles.

## DB-CONN-FAIL-014
TLS failures serán distinguibles.

## DB-CONN-FAIL-015
Authentication failures serán distinguibles.

## DB-CONN-FAIL-016
Session initialization failures serán distinguibles.

## DB-CONN-FAIL-017
Pool failures serán distinguibles.

## DB-CONN-FAIL-018
TLS verification failure no provocará insecure downgrade.

## DB-CONN-FAIL-019
Authentication failure no tendrá aggressive blind retry.

## DB-CONN-FAIL-020
Partially initialized connection no será READY.

## DB-CONN-FAIL-021
BROKEN connection no volverá a READY físicamente.

## DB-CONN-FAIL-022
Reconnect creará nueva physical connection identity.

## DB-CONN-FAIL-023
Reconnect no implicará session continuity.

## DB-CONN-FAIL-024
Reconnect no preservará transaction automáticamente.

## DB-CONN-FAIL-025
Reconnect no preservará temporary tables automáticamente.

## DB-CONN-FAIL-026
Reconnect no preservará session locks automáticamente.

## DB-CONN-FAIL-027
Reconnect no preservará server-side cursors automáticamente.

## DB-CONN-FAIL-028
Prepared statements podrán estar ligados a connection identity.

## DB-CONN-FAIL-029
Safe declarative session configuration podrá reconstruirse.

## DB-CONN-FAIL-030
Dynamic session state no será inventado tras reconnect.

## DB-CONN-FAIL-031
Reset failure descartará connection.

## DB-CONN-FAIL-032
Pool contamination será evitada.

## DB-CONN-FAIL-033
BROKEN connection nunca regresará al reusable pool.

## DB-CONN-FAIL-034
UNKNOWN connection health no será reusable por default.

## DB-CONN-FAIL-035
Connection ownership será scoped.

## DB-CONN-FAIL-036
Connection lease será explícito cuando exista pooling.

## DB-CONN-FAIL-037
Connection no podrá prestarse simultáneamente a owners incompatibles.

## DB-CONN-FAIL-038
Request cleanup verificará borrowed connections.

## DB-CONN-FAIL-039
Stale connection será detectable.

## DB-CONN-FAIL-040
Connection validation será configurable.

## DB-CONN-FAIL-041
Validation strategy será capability-driven.

## DB-CONN-FAIL-042
Validation success no garantizará next-operation success.

## DB-CONN-FAIL-043
Validation failure antes de operation permitirá reacquisition.

## DB-CONN-FAIL-044
Reacquisition before operation no será query retry.

## DB-CONN-FAIL-045
Send state será modelado cuando sea posible.

## DB-CONN-FAIL-046
NOT_SENT será distinto de POSSIBLY_SENT.

## DB-CONN-FAIL-047
POSSIBLY_SENT no será tratado como confirmed failure.

## DB-CONN-FAIL-048
SENT no implicará successful execution.

## DB-CONN-FAIL-049
Receive failure no implicará execution failure.

## DB-CONN-FAIL-050
Partial result no será complete result.

## DB-CONN-FAIL-051
Ordinary query no devolverá partial result como completo.

## DB-CONN-FAIL-052
Streaming failure podrá ocurrir después de yields previos.

## DB-CONN-FAIL-053
Streaming failure no implicará resumability.

## DB-CONN-FAIL-054
Server-side prepared statements se invalidarán tras reconnect cuando corresponda.

## DB-CONN-FAIL-055
Connection loss during send podrá producir UNKNOWN.

## DB-CONN-FAIL-056
Connection loss during receive podrá producir UNKNOWN.

## DB-CONN-FAIL-057
Write ACK loss no tendrá blind retry.

## DB-CONN-FAIL-058
Active transaction no migrará a otra connection.

## DB-CONN-FAIL-059
Connection loss podrá taint transaction.

## DB-CONN-FAIL-060
UNKNOWN COMMIT será preservado.

## DB-CONN-FAIL-061
UNKNOWN COMMIT no tendrá blind retry.

## DB-CONN-FAIL-062
Rollback connection loss podrá producir uncertainty.

## DB-CONN-FAIL-063
Network partition no implicará server session termination.

## DB-CONN-FAIL-064
Implicit rollback requerirá evidence/capability.

## DB-CONN-FAIL-065
Half-open connection será contemplada.

## DB-CONN-FAIL-066
TCP keepalive no equivaldrá a database health.

## DB-CONN-FAIL-067
Idle connection lifetime será gobernable.

## DB-CONN-FAIL-068
Proactive retirement será soportable.

## DB-CONN-FAIL-069
Connection retirement podrá usar jitter.

## DB-CONN-FAIL-070
Reconnect storms serán mitigables.

## DB-CONN-FAIL-071
Reconnect tendrá bounded budget.

## DB-CONN-FAIL-072
Acquire tendrá bounded timeout.

## DB-CONN-FAIL-073
Pool exhaustion será distinta de endpoint unavailable.

## DB-CONN-FAIL-074
Pool wait queue será bounded.

## DB-CONN-FAIL-075
Unbounded waiters estarán prohibidos por default.

## DB-CONN-FAIL-076
Backpressure será aplicable a pool saturation.

## DB-CONN-FAIL-077
Connection leak detection será soportable.

## DB-CONN-FAIL-078
Leak detection no cerrará arbitrariamente recursos aún owned.

## DB-CONN-FAIL-079
Persistent runtime no conservará broken connection para otro request.

## DB-CONN-FAIL-080
Request cleanup será obligatorio en persistent runtimes.

## DB-CONN-FAIL-081
Physical connections no se asumirán fork-safe.

## DB-CONN-FAIL-082
Physical connections no se asumirán coroutine-safe.

## DB-CONN-FAIL-083
Circuit breaker será consultado antes de normal connection attempt cuando aplique.

## DB-CONN-FAIL-084
Not every connection-related error contribuirá al circuit.

## DB-CONN-FAIL-085
Invalid credentials no implicarán unhealthy endpoint.

## DB-CONN-FAIL-086
Connection failure no autorizará writer failover por sí sola.

## DB-CONN-FAIL-087
Reconnect será distinto de failover.

## DB-CONN-FAIL-088
Reacquire será distinto de retry.

## DB-CONN-FAIL-089
Replica reacquisition respetará consistency.

## DB-CONN-FAIL-090
Sticky read rules sobrevivirán a recovery.

## DB-CONN-FAIL-091
Transaction affinity sobrevivirá a recovery decision.

## DB-CONN-FAIL-092
Shard affinity no será alterada arbitrariamente.

## DB-CONN-FAIL-093
Tenant context será preservado.

## DB-CONN-FAIL-094
Tenant drift estará prohibido.

## DB-CONN-FAIL-095
Security context será preservado.

## DB-CONN-FAIL-096
TLS policy será preservada durante retries/reconnect.

## DB-CONN-FAIL-097
Credentials no serán expuestas en telemetry.

## DB-CONN-FAIL-098
DSN será redactado.

## DB-CONN-FAIL-099
Connect retry será distinto de operation retry.

## DB-CONN-FAIL-100
Connect failure antes de query podrá ser retryable.

## DB-CONN-FAIL-101
New connection no hará seguro un uncertain write.

## DB-CONN-FAIL-102
Recovery action será explícita.

## DB-CONN-FAIL-103
Connection recovery policy no decidirá domain idempotency.

## DB-CONN-FAIL-104
Connection classifier no decidirá transaction replay.

## DB-CONN-FAIL-105
Platform classifiers podrán diferir.

## DB-CONN-FAIL-106
MySQL y MariaDB podrán tener classifiers independientes.

## DB-CONN-FAIL-107
SQLite no será modelado artificialmente como TCP database.

## DB-CONN-FAIL-108
Structured error codes serán preferidos a string matching.

## DB-CONN-FAIL-109
String matching será fallback encapsulado.

## DB-CONN-FAIL-110
Unknown classification permanecerá UNKNOWN.

## DB-CONN-FAIL-111
Conservative discard podrá aplicarse a unknown health.

## DB-CONN-FAIL-112
Discard policy evitará churn innecesario.

## DB-CONN-FAIL-113
SUSPECT podrá pasar por quarantine.

## DB-CONN-FAIL-114
Quarantined connection no estará disponible normalmente.

## DB-CONN-FAIL-115
Protocol desynchronization preferirá discard.

## DB-CONN-FAIL-116
Unconsumed results serán considerados antes de reuse.

## DB-CONN-FAIL-117
Streaming result mantendrá ownership apropiado.

## DB-CONN-FAIL-118
Early stream termination liberará recursos.

## DB-CONN-FAIL-119
Stream cleanup failure podrá invalidar connection.

## DB-CONN-FAIL-120
Cancellation podrá afectar connection health.

## DB-CONN-FAIL-121
Timeout podrá afectar connection health.

## DB-CONN-FAIL-122
Client timeout no significará server cancellation.

## DB-CONN-FAIL-123
Timed-out query no permitirá unsafe immediate connection reuse.

## DB-CONN-FAIL-124
Deadline limitará reconnect.

## DB-CONN-FAIL-125
Connection health observations serán timestamped.

## DB-CONN-FAIL-126
Health observation será distinta de global truth.

## DB-CONN-FAIL-127
Endpoint health podrá agregarse desde múltiples observations.

## DB-CONN-FAIL-128
Local evidence no será presentada como global evidence.

## DB-CONN-FAIL-129
Connection telemetry tendrá bounded cardinality.

## DB-CONN-FAIL-130
Connect retry será observable separadamente.

## DB-CONN-FAIL-131
Failure stage será observable.

## DB-CONN-FAIL-132
Connection state será observable.

## DB-CONN-FAIL-133
Operation outcome uncertainty será observable.

## DB-CONN-FAIL-134
Raw credentials nunca serán telemetry dimensions.

## DB-CONN-FAIL-135
Connection exceptions serán tipadas.

## DB-CONN-FAIL-136
Exception type no decidirá recovery automáticamente.

## DB-CONN-FAIL-137
Connection policies serán compilables a immutable objects.

## DB-CONN-FAIL-138
Infinite reconnect no será default válido.

## DB-CONN-FAIL-139
Infinite pool acquire no será default válido.

## DB-CONN-FAIL-140
Unbounded pool queue no será default válido.

## DB-CONN-FAIL-141
Failure injection será soportada.

## DB-CONN-FAIL-142
Unknown write tendrá prueba explícita.

## DB-CONN-FAIL-143
Unknown commit tendrá prueba explícita.

## DB-CONN-FAIL-144
Pool contamination tendrá prueba explícita.

## DB-CONN-FAIL-145
Persistent runtime isolation tendrá prueba explícita.

## DB-CONN-FAIL-146
Tenant reconnect isolation tendrá prueba explícita.

## DB-CONN-FAIL-147
Writer authority tendrá prueba explícita.

## DB-CONN-FAIL-148
Backoff tests serán deterministas.

## DB-CONN-FAIL-149
Connection system no generará SQL de aplicación.

## DB-CONN-FAIL-150
Connection system no interpretará entities.

## DB-CONN-FAIL-151
Connection system no realizará ORM recovery.

## DB-CONN-FAIL-152
Connection system no fabricará transaction continuity.

## DB-CONN-FAIL-153
Connection system no fabricará confirmed operation outcome.

## DB-CONN-FAIL-154
Connection system no fabricará failover authority.

## DB-CONN-FAIL-155
Connection replacement será explícitamente distinguido de operation replay.

## DB-CONN-FAIL-156
Connection Manager controlará adquisición, no Query Builder.

## DB-CONN-FAIL-157
Driver reportará evidencia, no política de negocio.

## DB-CONN-FAIL-158
Executor preservará operation phase al reportar connection failure.

## DB-CONN-FAIL-159
Transaction Manager determinará impacto transaccional.

## DB-CONN-FAIL-160
Resilience layer coordinará recovery.

## DB-CONN-FAIL-161
Failover layer determinará cambios de endpoint.

## DB-CONN-FAIL-162
Topology determinará authority elegible.

## DB-CONN-FAIL-163
Security layer determinará restricciones de conexión.

## DB-CONN-FAIL-164
Pool nunca convertirá broken connection en reusable por conveniencia.

## DB-CONN-FAIL-165
Una nueva conexión no heredará identity de la anterior.

## DB-CONN-FAIL-166
Una nueva conexión no heredará locks de la anterior.

## DB-CONN-FAIL-167
Una nueva conexión no heredará transaction de la anterior.

## DB-CONN-FAIL-168
Una nueva conexión no resolverá UNKNOWN outcome por sí misma.

## DB-CONN-FAIL-169
Connection availability no implicará operation replay safety.

## DB-CONN-FAIL-170
Ante evidencia insuficiente, VoltStack preferirá preservar uncertainty antes que inventar continuidad.

---

# 231. Modelo formal

Sea:

```text
C
```

una conexión física y:

```text
O
```

una operación.

Definimos:

```text
Health(C)
∈
{HEALTHY, SUSPECT, BROKEN, CLOSED, UNKNOWN}
```

y:

```text
Outcome(O)
∈
{SUCCESS, FAILURE, UNKNOWN}
```

No existe implicación general:

```text
Health(C) = BROKEN
⇒
Outcome(O) = FAILURE
```

---

# 232. Regla de descarte

Sea:

```text
Reusable(C)
```

la propiedad que permite regresar una conexión al pool.

Entonces:

```text
Health(C) = BROKEN
⇒
Reusable(C) = false
```

y por default:

```text
Health(C) = UNKNOWN
⇒
Reusable(C) = false
```

---

# 233. Regla de reconnect

Si:

```text
Reconnect(C₁) = C₂
```

entonces:

```text
C₁ ≠ C₂
```

y:

```text
SessionState(C₁)
≠
SessionState(C₂)
```

salvo el subconjunto explícitamente reconstruido.

---

# 234. Regla de transaction affinity

Sea una transacción:

```text
T
```

ligada a:

```text
C₁
```

Entonces:

```text
Connection(T) = C₁
```

Si:

```text
Health(C₁) = BROKEN
```

no es válido:

```text
Connection(T) := C₂
```

para continuar la misma transacción.

---

# 235. Regla de send uncertainty

Si:

```text
SendState(O)
∈
{POSSIBLY_SENT, UNKNOWN}
```

y no existe evidencia adicional:

```text
Outcome(O) = UNKNOWN
```

cuando la operación pudiera tener side effects.

---

# 236. Regla de reacquisition

Si:

```text
OperationPhase(O) = NOT_STARTED
```

y:

```text
C₁ = unusable
```

puede realizarse:

```text
C₂ = Acquire()
```

sin considerar esto un replay de `O`.

---

# 237. Regla de pool

Una conexión podrá regresar al pool solo si:

```text
Reusable(C)
=
OwnershipReleased(C)
∧
TransactionStateSafe(C)
∧
ResetSuccessful(C)
∧
ProtocolStateSafe(C)
∧
HealthAcceptable(C)
```

---

# 238. Regla de seguridad

Reconnect será válido únicamente si:

```text
SecurityPolicy(C₂)
≥
RequiredSecurityPolicy(O)
```

Nunca se degradará seguridad para recuperar disponibilidad.

---

# 239. Modelo final

```text
                CONNECTION
                    │
                    ▼
               Physical I/O
                    │
          ┌─────────┴─────────┐
          │                   │
       HEALTHY             FAILURE
          │                   │
          │                   ▼
          │             Classify Failure
          │                   │
          │                   ▼
          │            Evaluate Connection
          │                   │
          │        ┌──────────┼──────────┐
          │        │          │          │
          │     SUSPECT    BROKEN     UNKNOWN
          │        │          │          │
          │        ▼          └────┬─────┘
          │    Validate            │
          │        │               ▼
          │   ┌────┴────┐       Discard
          │   │         │           │
          │ PASS      FAIL          │
          │   │         │           │
          └───┘         └─────┬─────┘
                              ▼
                     Need another connection?
                              │
                       ┌──────┴──────┐
                       │             │
                      NO            YES
                       │             │
                       ▼             ▼
                   Propagate     Reacquire/
                                Reconnect/
                                Failover
                                     │
                                     ▼
                        Operation already started?
                              ┌──────┴──────┐
                              │             │
                             NO            YES
                              │             │
                              ▼             ▼
                           Execute      Analyze Outcome
                                            │
                                    ┌───────┼───────┐
                                    │       │       │
                                 SUCCESS FAILURE UNKNOWN
                                    │       │       │
                                    └───────┼───────┘
                                            ▼
                                    Recovery Policy
```

---

# 240. Regla maestra final

> **VoltStack tratará cada conexión física como un recurso descartable y reemplazable, pero nunca tratará el estado lógico de una operación o transacción como descartable. Una conexión rota puede cerrarse y sustituirse; el resultado de aquello que ya fue enviado a la base de datos deberá determinarse independientemente mediante evidencia.**

En forma compacta:

```text
Connection Failure
≠
Query Failure

Connection Failure
≠
Transaction Failure

Broken Connection
≠
Unhealthy Cluster

Reconnect
≠
Retry

Reconnect
≠
Failover

Reconnect
≠
Session Continuity

Reacquire
≠
Operation Replay

Validation
≠
Future Availability

Timeout
≠
Server Cancellation

Connection Lost
≠
Operation Failed

Connection Lost
≠
Transaction Rolled Back

New Connection
≠
Old Transaction

New Connection
≠
Known Outcome

BROKEN
→
DISCARD

UNKNOWN CONNECTION HEALTH
→
DO NOT REUSE BY DEFAULT

UNKNOWN OPERATION OUTCOME
→
DO NOT BLIND RETRY
```

La prioridad arquitectónica será siempre:

```text
Correctness
   >
Session Convenience
   >
Automatic Reconnect
```

---

# 241. Estado del Bloque 23

```text
BLOCK 23 — RESILIENCE

✓ 235_DATABASE_RESILIENCE_ARCHITECTURE.md
✓ 236_DATABASE_CONNECTION_FAILURE_HANDLING_SYSTEM.md
○ 237_DATABASE_QUERY_FAILURE_HANDLING_SYSTEM.md
○ 238_DATABASE_RETRY_POLICY_SYSTEM.md
○ 239_DATABASE_CIRCUIT_BREAKER_INTEGRATION_SYSTEM.md
○ 240_DATABASE_FAILOVER_AND_RECOVERY_SYSTEM.md
○ 241_DATABASE_RESOURCE_EXHAUSTION_PROTECTION_SYSTEM.md
```

---

# 242. Siguiente documento

```text
237_DATABASE_QUERY_FAILURE_HANDLING_SYSTEM.md
```

El siguiente documento definirá el tratamiento específico de fallos producidos durante la ejecución de consultas:

```text
query validation failures
compilation failures
statement preparation failures
parameter binding failures
execution failures
constraint violations
type failures
deadlocks
serialization failures
lock timeouts
query timeouts
cancellation
partial result failures
platform errors
unknown query outcomes
```

manteniendo la separación fundamental:

> **Que una query produzca una excepción no determina por sí mismo si la base de datos la ejecutó, si sus efectos fueron confirmados, si la transacción continúa siendo válida o si la operación puede repetirse.**