# 211_DATABASE_CONNECTION_EVENT_SYSTEM.md

# VoltStack Quantum Database
## Database Connection Event System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 211 — Database Connection Event System  
**Bloque:** 20 — Events  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `210_DATABASE_QUERY_EVENT_SYSTEM.md`  
**Siguiente documento:** `212_DATABASE_TRANSACTION_EVENT_PIPELINE.md`

---

# 1. Propósito

`Database Connection Event System` define el modelo de eventos mediante el cual VoltStack podrá observar de forma segura, tipada y desacoplada el ciclo de vida de conexiones lógicas y físicas dentro de `Quantum/Database`.

El sistema cubrirá:

```text
Connection Resolution
↓
Endpoint Selection
↓
Acquisition
↓
Pool Checkout
↓
Physical Open / Reuse
↓
Configuration
↓
Usage
↓
Reset
↓
Release
↓
Pool Return / Discard / Physical Close
```

así como:

```text
connection failure
timeout
pool exhaustion
tainting
reset failure
failover
endpoint degradation
physical close
```

La regla central será:

> **Un Connection Event describe un hecho observable del ciclo de vida de una conexión; no representa la conexión misma, no concede acceso al recurso físico y no sustituye al Connection Manager, Pool, Driver, Router ni a sus contratos de ownership y reset.**

---

# 2. Objetivos

El sistema deberá proporcionar:

```text
typed connection events
logical/physical connection distinction
pool awareness
acquisition observability
reuse observability
reset observability
release observability
taint/discard observability
failover correlation
endpoint-role awareness
transaction awareness
query correlation
safe endpoint metadata
connection-wait timing
pool-pressure diagnostics
persistent-runtime safety
telemetry integration
testing support
```

---

# 3. Distinciones fundamentales

VoltStack deberá preservar:

```text
Connection Event
≠
Connection
≠
Physical Connection
≠
Logical Connection
≠
Connection Lease
≠
Connection Pool
≠
Endpoint
≠
Driver
≠
Query Event
≠
Transaction Event
≠
Telemetry Event
```

---

# 4. Logical Connection ≠ Physical Connection

Una conexión lógica representa:

```text
application/database access context
```

Una conexión física representa:

```text
actual driver/session/socket/protocol connection
```

Por tanto:

```text
Logical Connection
≠
Physical Connection
```

---

# 5. Connection Acquired ≠ Connection Opened

Esto es fundamental.

En un pool:

```text
Physical Connection P1 already open
↓
checkout
↓
ConnectionAcquired
```

sin:

```text
ConnectionOpened
```

durante esa operación.

---

# 6. Connection Released ≠ Connection Closed

En persistent runtimes:

```text
operation ends
↓
ConnectionReleased
↓
returned to pool
```

pero la conexión física puede permanecer abierta.

---

# 7. Connection Returned ≠ Connection Reset

Una conexión no deberá volver al pool reusable antes de que:

```text
session state
transaction state
temporary state
tenant state
driver state
```

hayan sido restaurados según contrato.

---

# 8. Connection Failure ≠ Query Failure

Ejemplo:

```text
Pool Acquisition Timeout
```

puede hacer que una Query falle.

Pero son perspectivas distintas:

```text
ConnectionAcquisitionFailed
=
connection subsystem fact

QueryFailed
=
query operation fact
```

---

# 9. Connection Event ≠ Connection Hook

Los listeners no deberán modificar arbitrariamente:

```text
DSN
credentials
endpoint
pool ownership
session state
transaction state
```

Los cambios deberán ocurrir mediante:

```text
Connection Resolver
Routing Policy
Connection Factory
Pool Policy
Driver Extension
Session Configurator
```

---

# 10. Arquitectura general

```text
Application / Query / Transaction
             │
             ▼
      Connection Manager
             │
             ▼
      Connection Resolver
             │
             ▼
       Endpoint Router
             │
             ▼
      Connection Pool
             │
      ┌──────┴───────┐
      ▼              ▼
  Reuse Existing   Open New
      │              │
      └──────┬───────┘
             ▼
      Connection Lease
             │
             ▼
           Usage
             │
             ▼
           Reset
             │
      ┌──────┴───────┐
      ▼              ▼
 Return to Pool    Discard
```

Los eventos observarán boundaries dentro de este flujo.

---

# 11. ConnectionEvent contract

```php
interface ConnectionEvent extends DatabaseEvent
{
    public function connectionOperationId(): ConnectionOperationId;

    public function connectionContext(): ConnectionEventContext;
}
```

---

# 12. ConnectionOperationId

Cada ciclo lógico de adquisición/uso/release podrá recibir:

```text
ConnectionOperationId
```

---

# 13. ConnectionOperationId ≠ PhysicalConnectionId

Una misma conexión física puede servir múltiples operaciones:

```text
Physical P1
├── Operation C1
├── Operation C2
└── Operation C3
```

---

# 14. PhysicalConnectionId

Podrá existir una identidad interna segura:

```php
final readonly class PhysicalConnectionId
{
    public function __construct(
        public string $value,
    ) {}
}
```

No deberá contener:

```text
host
username
password
raw socket details
```

---

# 15. ConnectionLeaseId

Para pools:

```text
PhysicalConnectionId
≠
ConnectionLeaseId
```

Cada checkout puede tener un lease distinto.

---

# 16. Connection event context

```php
final readonly class ConnectionEventContext
{
    public function __construct(
        public ConnectionOperationId $operationId,
        public ?PhysicalConnectionId $physicalId,
        public ?ConnectionLeaseId $leaseId,
        public LogicalConnectionName $logicalName,
        public EndpointRole $role,
        public PersistenceDomain $domain,
        public ?TenantContextReference $tenant,
        public ?ShardId $shard,
        public ?TransactionId $transactionId,
        public ConnectionEventMetadata $metadata,
    ) {}
}
```

---

# 17. EndpointRole

```php
enum EndpointRole
{
    case WRITER;
    case REPLICA;
    case UNKNOWN;
}
```

Podrán añadirse roles especializados en extensiones.

---

# 18. Connection lifecycle stages

```php
enum ConnectionLifecycleStage
{
    case RESOLUTION;
    case ROUTING;
    case ACQUISITION;
    case OPEN;
    case CONFIGURATION;
    case CHECKOUT;
    case USAGE;
    case RESET;
    case RELEASE;
    case CLOSE;
    case FAILOVER;
}
```

---

# 19. Core event set

Eventos base recomendados:

```text
ConnectionResolutionStarted
ConnectionResolved

ConnectionAcquisitionStarted
ConnectionAcquired
ConnectionAcquisitionFailed

ConnectionOpened
ConnectionReused

ConnectionResetStarted
ConnectionReset
ConnectionResetFailed

ConnectionReleased
ConnectionDiscarded
ConnectionClosed

ConnectionTainted
ConnectionFailed
```

---

# 20. Diagnostic events

Opcionales:

```text
ConnectionPoolWaitStarted
ConnectionPoolWaitCompleted
ConnectionPoolExhausted

ConnectionConfigurationStarted
ConnectionConfigured

ConnectionFailoverStarted
ConnectionFailoverCompleted
ConnectionFailoverFailed
```

---

# 21. Resolution lifecycle

```text
Logical Connection Request
↓
ConnectionResolutionStarted
↓
resolve domain/config/tenant/shard
↓
ConnectionResolved
```

---

# 22. ConnectionResolved

Podrá describir:

```text
logical connection name
persistence domain
writer/replica requirement
shard
routing constraints
consistency requirement
```

sin revelar secretos.

---

# 23. Resolution ≠ Routing

Resolver:

> ¿Qué configuración lógica aplica?

Router:

> ¿Qué endpoint elegible debe usarse?

Por tanto:

```text
Connection Resolution
≠
Endpoint Routing
```

---

# 24. Routing events

La decisión exacta puede pertenecer al Read/Write Routing System.

El Connection Event System podrá observar el resultado mediante contexto, sin duplicar `QueryRouteResolved`.

---

# 25. ConnectionAcquisitionStarted

Representa el comienzo de la obtención del recurso necesario.

Puede iniciar antes de:

```text
pool wait
new physical connection
checkout
```

---

# 26. Acquisition timing

Deberá poder medir:

```text
poolWait
physicalOpen
configuration
totalAcquisition
```

por separado cuando exista evidencia.

---

# 27. ConnectionPoolWaitStarted

Solo se emite cuando realmente existe espera.

No deberá emitirse en fast-path inmediato.

---

# 28. ConnectionPoolWaitCompleted

Podrá incluir:

```text
wait duration
queue depth snapshot
pool occupancy snapshot
```

con cardinalidad controlada.

---

# 29. Pool occupancy snapshot

Ejemplo:

```php
final readonly class ConnectionPoolSummary
{
    public function __construct(
        public int $capacity,
        public int $inUse,
        public int $idle,
        public int $waiting,
    ) {}
}
```

---

# 30. Pool metrics ≠ Event payload obligatorio

No será necesario calcular estos valores para cada acquisition si nadie los observa.

---

# 31. ConnectionPoolExhausted

Representa:

```text
no lease available within policy/budget
```

No necesariamente:

```text
database endpoint unavailable
```

---

# 32. Pool exhaustion ≠ DB outage

La base puede estar sana mientras la aplicación agotó su pool.

---

# 33. ConnectionAcquired

Se emite cuando el caller recibe un lease utilizable.

---

# 34. Acquired prerequisites

Antes de `ConnectionAcquired` deberá haberse satisfecho:

```text
endpoint selected
physical resource valid
required reset/configuration complete
lease established
```

según la arquitectura.

---

# 35. Acquired ≠ Opened

Se reafirma:

```text
ConnectionAcquired
≠
ConnectionOpened
```

---

# 36. ConnectionOpened

Se emite únicamente cuando se crea una nueva conexión física.

---

# 37. Opened payload

Podrá incluir:

```text
platform ID
driver ID
endpoint role
open duration
TLS mode classification
physical connection ID
```

sin:

```text
password
raw DSN
client secret
```

---

# 38. ConnectionReused

Se emite cuando un recurso físico existente es reutilizado.

---

# 39. Reuse source

Podrá indicar:

```text
POOL_IDLE
PERSISTENT_RUNTIME_REUSE
DRIVER_REUSE
CUSTOM
```

---

# 40. Reused ≠ Safe Yet

Una conexión física existente solo deberá considerarse reusable después de validar/resetear el estado requerido.

---

# 41. Connection configuration

Una conexión recién abierta puede requerir:

```text
timezone
encoding
isolation defaults
SQL mode
application name
session variables
role
schema/search_path
```

---

# 42. Configuration ownership

La configuración pertenece a:

```text
Connection Configurator
Platform
Driver
Tenant/Shard Integration
```

no a listeners.

---

# 43. ConnectionConfigured

Evento diagnóstico.

Podrá indicar:

```text
configuration profile
generation
capability profile
```

sin exponer secretos.

---

# 44. Session state problem

En persistent runtimes una conexión puede conservar:

```text
open transaction
changed isolation
temporary tables
session variables
search_path
tenant state
role changes
prepared statements
locks
```

---

# 45. Reset is correctness-critical

Por tanto:

> **Connection reset no será un evento opcional de observabilidad; el reset real será un contrato directo del Connection System.**

El evento solo lo describe.

---

# 46. ConnectionResetStarted

Se emite cuando comienza el protocolo de saneamiento antes de reuse/release según policy.

---

# 47. Reset responsibilities

Podrán incluir:

```text
rollback open transaction
restore isolation
clear session variables
restore schema/search_path
clear tenant-specific state
release locks
clear temporary execution state
reset driver attributes
```

según plataforma.

---

# 48. Reset profile

```php
final readonly class ConnectionResetProfile
{
    public function __construct(
        public bool $transactionReset,
        public bool $sessionReset,
        public bool $tenantReset,
        public bool $schemaReset,
        public bool $driverReset,
    ) {}
}
```

---

# 49. ConnectionReset

Se emite solo tras reset confirmado.

---

# 50. Reset completed ≠ connection released

Puede existir:

```text
Reset
↓
more internal validation
↓
Release
```

---

# 51. ConnectionResetFailed

Si el estado no puede garantizarse limpio:

```text
connection MUST NOT be returned as reusable
```

por default.

---

# 52. Reset failure policy

Default recomendado:

```text
Reset Failure
↓
TAINT
↓
DISCARD
↓
PHYSICAL CLOSE
```

---

# 53. ConnectionTainted

Representa que la conexión ya no puede considerarse segura para reutilización normal.

---

# 54. Taint causes

Ejemplos:

```text
unknown transaction outcome
failed rollback
failed reset
protocol desynchronization
driver fatal error
session state uncertainty
timeout with unknown server state
```

---

# 55. TAINTED ≠ CLOSED

Una conexión tainted puede seguir físicamente abierta durante un corto periodo antes de ser descartada/cerrada.

---

# 56. TAINTED ≠ FAILED operation

Puede marcarse preventivamente por uncertainty aunque la operación previa haya terminado con algún outcome conocido.

---

# 57. ConnectionDiscarded

Indica:

```text
connection removed from reusable pool
```

No implica necesariamente que `close()` físico ya terminó.

---

# 58. ConnectionClosed

Representa el cierre físico confirmado hasta donde el driver permite afirmarlo.

---

# 59. Closed ≠ Released

Una conexión puede ser cerrada directamente en lugar de volver al pool.

---

# 60. Release lifecycle

```text
Caller done
↓
ResetStarted
↓
Reset
↓
ConnectionReleased
↓
Returned to Pool
```

o:

```text
Caller done
↓
ResetFailed
↓
Tainted
↓
Discarded
↓
Closed
```

---

# 61. ConnectionReleased

Representa el fin del lease lógico del caller.

---

# 62. Release must clear ownership

Después de release, el caller no deberá continuar usando el connection lease.

---

# 63. Use-after-release

Deberá ser tratado como error de lifecycle.

---

# 64. Connection lease state

```php
enum ConnectionLeaseState
{
    case ACQUIRED;
    case IN_USE;
    case RELEASING;
    case RELEASED;
    case INVALID;
}
```

---

# 65. Connection lifecycle state

Para recurso físico:

```php
enum PhysicalConnectionState
{
    case OPENING;
    case OPEN;
    case IDLE;
    case LEASED;
    case RESETTING;
    case TAINTED;
    case CLOSING;
    case CLOSED;
    case UNKNOWN;
}
```

---

# 66. Event state ≠ canonical state

Los eventos describen transiciones.

La autoridad continúa siendo:

```text
Connection Manager / Pool / Lease state
```

---

# 67. ConnectionFailed

Será evento genérico para fallas del subsystem.

Pero deberá incluir una categoría.

---

# 68. Failure categories

```php
enum ConnectionFailureCategory
{
    case RESOLUTION;
    case ROUTING;
    case POOL_EXHAUSTED;
    case OPEN_FAILED;
    case AUTHENTICATION_FAILED;
    case TLS_FAILED;
    case CONFIGURATION_FAILED;
    case RESET_FAILED;
    case NETWORK_FAILURE;
    case PROTOCOL_FAILURE;
    case CLOSED_UNEXPECTEDLY;
    case DRIVER_FAILURE;
    case UNKNOWN;
}
```

---

# 69. Authentication failure security

Nunca incluir:

```text
password
secret
full connection string
```

en payload.

---

# 70. Safe endpoint identity

```php
final readonly class SafeEndpointIdentity
{
    public function __construct(
        public EndpointId $id,
        public EndpointRole $role,
        public PlatformId $platform,
        public ?string $region,
        public ?string $availabilityZone,
    ) {}
}
```

---

# 71. Hostname exposure

Podrá ser configurable.

En algunos entornos incluso:

```text
db-internal-prod-01
```

puede ser sensible.

---

# 72. Endpoint fingerprint

Podrá utilizarse en producción en lugar del hostname.

---

# 73. Driver messages

Al igual que Query Events:

```text
driver exception text
```

deberá sanitizarse.

---

# 74. Connection wait timeout

Debe distinguirse de:

```text
network connect timeout
query timeout
transaction timeout
```

---

# 75. Timeout categories

```php
enum ConnectionTimeoutKind
{
    case POOL_WAIT;
    case TCP_CONNECT;
    case TLS_HANDSHAKE;
    case AUTHENTICATION;
    case DRIVER_OPEN;
    case RESET;
    case CUSTOM;
}
```

---

# 76. Connection timing summary

```php
final readonly class ConnectionTimingSummary
{
    public function __construct(
        public ?Duration $resolution,
        public ?Duration $poolWait,
        public ?Duration $open,
        public ?Duration $configuration,
        public ?Duration $reset,
        public ?Duration $totalAcquisition,
    ) {}
}
```

---

# 77. Monotonic timing

Las duraciones deberán usar reloj monotónico cuando sea posible.

---

# 78. Connection events and queries

Correlación:

```text
QueryId
↓
ConnectionOperationId
↓
ConnectionLeaseId
↓
PhysicalConnectionId
```

---

# 79. One connection, multiple queries

Dentro de una transacción:

```text
Connection Lease L1
├── Query Q1
├── Query Q2
└── Query Q3
```

No emitir `ConnectionAcquired` por cada query si comparten el mismo lease.

---

# 80. Query without explicit transaction

Puede:

```text
acquire
execute
release
```

por operación.

---

# 81. Connection pinning

Transactions, streaming results o explicit pinning pueden prolongar el lease.

---

# 82. Streaming result

Una conexión puede permanecer leased hasta:

```text
ResultCursor closed/exhausted
```

---

# 83. QueryExecuted ≠ ConnectionReleased

En streaming:

```text
QueryExecuted/open cursor
↓
consume rows
↓
cursor closed
↓
ConnectionReleased
```

---

# 84. Transaction correlation

Dentro de transacción:

```text
TransactionStarted
↓
ConnectionAcquired
↓
Queries
↓
Commit/Rollback
↓
Reset
↓
ConnectionReleased
```

---

# 85. Transaction owns connection affinity

La transaction policy podrá requerir que todas las queries del scope utilicen la misma conexión.

---

# 86. Connection Event System does not enforce affinity

La afinidad pertenece al Transaction/Connection Manager.

Los eventos solo la hacen observable.

---

# 87. Transaction taint

Si commit outcome es UNKNOWN:

```text
Connection
→ TAINTED
```

por default conservador.

---

# 88. Why

El estado protocol/session puede ser incierto y no debe volver al pool normal.

---

# 89. Rollback failure

También puede taint la conexión.

---

# 90. Nested transaction/savepoint

No debería cambiar automáticamente ownership del connection lease.

---

# 91. Savepoint event ≠ Connection event

Pertenece a Transaction Event Pipeline.

---

# 92. Read/write routing

Conexiones podrán estar asociadas a:

```text
WRITER
REPLICA
```

---

# 93. Replica selection

La decisión pertenece a:

```text
ReadWriteRouter / Replica Selector
```

El evento podrá incluir el resultado.

---

# 94. Replica lag

No deberá asumirse que una conexión a replica implica freshness suficiente.

---

# 95. Endpoint eligibility

Eventos podrán registrar:

```text
selected endpoint was eligible under policy X
```

sin redefinir la policy.

---

# 96. Failover

El sistema deberá observar cambios de endpoint.

---

# 97. ConnectionFailoverStarted

Podrá emitirse cuando un componente de resiliencia intenta cambiar authority/endpoint.

---

# 98. Failover ≠ retry

```text
Failover
=
topology/endpoint transition

Retry
=
operation replay
```

Pueden coexistir, pero son distintas.

---

# 99. Failover within transaction

Por default:

```text
active transaction
+
connection failure
```

no deberá transparentemente migrarse a otra conexión como si nada.

---

# 100. Transaction continuity

Una nueva conexión normalmente no comparte el transaction state de la anterior.

---

# 101. ConnectionFailoverCompleted

Debe indicar:

```text
previous endpoint identity
new endpoint identity
authority epoch / topology generation if relevant
reason
```

---

# 102. FailoverCompleted ≠ operation recovered

El cambio de endpoint puede completarse aunque la query/transaction original haya fallado.

---

# 103. ConnectionFailoverFailed

Representa incapacidad de encontrar/establecer un endpoint compatible.

---

# 104. Authority epoch

Podrá incluirse para:

```text
writer failover
```

cuando la topología lo maneje.

---

# 105. Sharding

Una conexión debe estar vinculada al shard efectivo cuando corresponda.

---

# 106. ShardContext

No deberá cambiar a mitad de lease salvo protocolo explícito.

---

# 107. Cross-shard query

Puede adquirir múltiples conexiones.

```text
Logical Query
├── Connection C1 → Shard A
├── Connection C2 → Shard B
└── Connection C3 → Shard C
```

---

# 108. Parent operation correlation

Los Connection Events podrán correlacionarse con:

```text
ParentQueryId
LargeDatasetOperationId
BulkOperationId
```

mediante metadata bounded.

---

# 109. Multitenancy

Una conexión tenant-aware podrá configurarse por:

```text
database
schema
session variable
role
search_path
```

según estrategia.

---

# 110. Tenant state is dangerous in pooled connections

Ejemplo:

```text
Request A:
tenant = 10
↓
connection session configured

connection returned without reset

Request B:
tenant = 20
↓
same connection
```

Esto sería una fuga crítica.

---

# 111. Tenant reset invariant

Antes de reuse:

```text
Tenant State Previous
→ RESET
→ Neutral/Base State
→ New Tenant State
```

cuando la estrategia use stateful session configuration.

---

# 112. Tenant reset events

Podrán reflejarse dentro de:

```text
ConnectionResetStarted
ConnectionReset
```

sin crear necesariamente un evento por cada variable.

---

# 113. Connection event payload must not leak tenant secrets

Tenant identity podrá ser:

```text
reference/fingerprint
```

según policy.

---

# 114. Persistent runtime importance

Este sistema será especialmente crítico en:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 115. Request reuse

El worker puede permanecer vivo mientras las conexiones físicas también.

Por tanto:

```text
worker lifetime
≠
request lifetime
≠
connection lifetime
```

---

# 116. Lifecycle matrix

```text
Application Request
    short

Connection Lease
    short/medium

Physical Connection
    long

Worker
    very long
```

---

# 117. State isolation

No deberán filtrarse:

```text
transaction
tenant
schema
session variables
temporary tables
locks
query timeout
isolation overrides
role
```

entre leases.

---

# 118. Reset event observability

Deberá ser posible comprobar:

```text
ConnectionReleased
was preceded by
successful reset
```

cuando pool policy lo exija.

---

# 119. OpenSwoole concurrency

Una misma conexión física no deberá ser entregada simultáneamente a dos coroutines salvo que Driver/connection contract lo permita explícitamente.

---

# 120. Lease exclusivity

Default:

```text
1 physical connection
→ at most 1 active lease
```

---

# 121. Lease exclusivity event

Violaciones deberán tratarse como:

```text
ConnectionLifecycleException
```

más que como evento funcional normal.

---

# 122. Persistent connection reuse

El evento `ConnectionReused` deberá poder distinguir:

```text
same worker reuse
pool reuse
driver persistent reuse
```

sin exponer detalles inseguros.

---

# 123. Connection pool architecture

Eventos del pool podrán observar:

```text
checkout
return
discard
capacity pressure
wait
exhaustion
```

---

# 124. Pool Event ≠ Connection Event

Conceptualmente pueden ser subfamilias.

```text
ConnectionEvent
├── Lease Events
├── Physical Events
└── Pool Events
```

---

# 125. ConnectionPoolEvent contract

```php
interface ConnectionPoolEvent extends ConnectionEvent
{
    public function poolId(): ConnectionPoolId;
}
```

---

# 126. Pool ID

Debe ser lógico/stable.

No deberá contener credenciales.

---

# 127. Pool created/destroyed

Podrán existir eventos de bajo volumen:

```text
ConnectionPoolStarted
ConnectionPoolStopped
```

principalmente para runtime lifecycle.

---

# 128. Per-request pool creation

No es recomendable en persistent runtimes salvo design específico.

---

# 129. Pool pressure telemetry

Podrá derivarse de eventos:

```text
wait duration
exhaustion
discard rate
open rate
reuse rate
```

---

# 130. Reuse ratio

Métrica conceptual:

```text
reuseRatio
=
reused acquisitions
/
total successful acquisitions
```

---

# 131. Open rate

Alta frecuencia de `ConnectionOpened` puede indicar:

```text
pool misconfiguration
connection churn
reset failures
low pool capacity
```

---

# 132. Discard rate

Alta tasa de discard puede indicar:

```text
tainted sessions
protocol failures
reset failures
connection instability
```

---

# 133. Telemetry bridge

```text
Connection Events
↓
Connection Telemetry Bridge
↓
metrics / spans / logs
```

---

# 134. Telemetry document

La especialización profunda será:

```text
218_DATABASE_CONNECTION_TELEMETRY_SYSTEM.md
```

más adelante.

---

# 135. Events remain source facts

Telemetry podrá:

```text
aggregate
sample
summarize
```

sin alterar lifecycle.

---

# 136. Event volume

`ConnectionAcquired` puede ser high-volume.

`ConnectionOpened` normalmente será menos frecuente con pooling.

---

# 137. Volume classifications

Ejemplo:

```text
ConnectionAcquired      HIGH
ConnectionReleased      HIGH
ConnectionOpened        MEDIUM
ConnectionClosed        MEDIUM
ConnectionPoolExhausted LOW
ConnectionTainted       LOW
```

---

# 138. Listener performance

Listeners sobre acquire/release deberán ser extremadamente ligeros.

---

# 139. No network I/O in hot listeners

No recomendado:

```text
ConnectionAcquired
↓
synchronous HTTP request
```

---

# 140. Security exposure levels

```php
enum ConnectionEventExposureLevel
{
    case MINIMAL;
    case SAFE_DIAGNOSTIC;
    case DEVELOPMENT;
}
```

---

# 141. MINIMAL

Podrá incluir:

```text
logical connection
role
platform
outcome
duration
```

---

# 142. SAFE_DIAGNOSTIC

Podrá añadir:

```text
endpoint fingerprint
pool ID
shard classification
region
reuse/open
```

---

# 143. DEVELOPMENT

Podrá permitir más detalles no secretos.

Nunca passwords.

---

# 144. DSN security

Por default:

```text
DSN
=
NOT EXPOSED
```

---

# 145. Username security

También podrá ocultarse por default.

---

# 146. TLS diagnostics

Podrá reportarse:

```text
TLS_ENABLED
TLS_DISABLED
TLS_UNKNOWN
```

sin claves/certificados privados.

---

# 147. Connection errors

Error summary:

```php
final readonly class ConnectionFailureSummary
{
    public function __construct(
        public ConnectionFailureCategory $category,
        public ConnectionLifecycleStage $stage,
        public bool $retryable,
        public bool $tainted,
        public SafeDatabaseError $error,
    ) {}
}
```

---

# 148. Retryable field

Será evidencia/policy hint.

No implica que el Connection Event System decida retry.

---

# 149. Connection retry ownership

Pertenece a:

```text
Connection Failure Handling
Retry Policy
Failover System
```

---

# 150. ConnectionRetryScheduled

Podrá existir como evento especializado si el subsystem de resiliencia decide retry.

---

# 151. Attempt model

```text
ConnectionOperationId C1
├── Attempt A1 → open fail
├── Attempt A2 → endpoint failover
└── Attempt A3 → success
```

---

# 152. ConnectionAttemptId

Podrá modelarse igual que QueryAttemptId.

---

# 153. Attempt failure ≠ operation failure

Un open failure intermedio no implica que acquisition final falle.

---

# 154. Final acquisition

Solo después de retry exhaustion podrá emitirse el outcome lógico final correspondiente.

---

# 155. Pool checkout race

Podría ocurrir:

```text
idle connection selected
↓
health check fails
↓
discard
↓
open/reuse another
↓
acquisition succeeds
```

Los eventos deberán permitir explicar este flujo.

---

# 156. Connection health checks

Podrán generar eventos diagnósticos:

```text
ConnectionHealthCheckStarted
ConnectionHealthCheckSucceeded
ConnectionHealthCheckFailed
```

pero no necesariamente estarán habilitados por default.

---

# 157. Health check ≠ Query business event

Si usa:

```text
SELECT 1
```

el sistema podrá marcar origin:

```text
INTERNAL_HEALTH_CHECK
```

para evitar contaminar analytics de queries de negocio.

---

# 158. Connection origin

```php
enum ConnectionOrigin
{
    case QUERY;
    case TRANSACTION;
    case MIGRATION;
    case HEALTH_CHECK;
    case LARGE_DATASET;
    case IMPORT;
    case EXPORT;
    case ADMINISTRATION;
    case INTERNAL;
}
```

---

# 159. Origin semantics

Sirve para:

```text
diagnostics
telemetry
recursion filtering
```

No para authorization.

---

# 160. Connection events and migrations

Migrations pueden requerir:

```text
writer-only
special timeout
long-running connection
```

y los eventos permitirán observarlo.

---

# 161. Connection events and backup/admin

Operations administrativas podrán usar conexiones dedicadas.

No deberán confundirse con application pool.

---

# 162. Dedicated connection

Podrá marcarse:

```text
leaseType = DEDICATED
```

---

# 163. Lease type

```php
enum ConnectionLeaseType
{
    case POOLED;
    case DEDICATED;
    case TRANSACTION_PINNED;
    case STREAM_PINNED;
    case ADMINISTRATIVE;
}
```

---

# 164. Dedicated ≠ untracked

Las conexiones dedicadas también deberán tener lifecycle events.

---

# 165. Connection lifecycle validation

En development/testing podrá verificarse:

```text
AcquisitionStarted
↓
Acquired
↓
Released
```

o:

```text
AcquisitionStarted
↓
Failed
```

---

# 166. Physical lifecycle validation

```text
Opened
↓
Idle/Leased cycles
↓
Closed
```

---

# 167. Invalid lifecycle

Ejemplos:

```text
Released before Acquired
Closed then Reused
Tainted then ReturnedReusable
Lease used after Release
```

---

# 168. ConnectionLifecycleValidator

```php
final class ConnectionLifecycleValidator
{
    public function observe(ConnectionEvent $event): void
    {
        // dev/testing invariant checks
    }
}
```

---

# 169. Production overhead

Lifecycle validation completa podrá deshabilitarse en producción.

---

# 170. Testing support

Deberá poder afirmarse:

```php
DatabaseEvents::assertSequence([
    ConnectionAcquisitionStarted::class,
    ConnectionAcquired::class,
    ConnectionResetStarted::class,
    ConnectionReset::class,
    ConnectionReleased::class,
]);
```

---

# 171. Pool reuse test

```text
Request A:
Opened
Acquired
Released

Request B:
Reused
Acquired
Released
```

sin segundo `Opened`.

---

# 172. Reset failure test

```text
Acquired
ResetStarted
ResetFailed
Tainted
Discarded
Closed
```

---

# 173. Unknown transaction outcome test

```text
ConnectionAcquired
Transaction Commit Attempt
Connection Lost
TransactionOutcomeUnknown
ConnectionTainted
ConnectionDiscarded
```

---

# 174. Tenant isolation test

Asegurar:

```text
tenant A session state
↓
reset
↓
tenant B lease
```

sin leakage.

---

# 175. Coroutine test

Dos coroutines no deberán recibir el mismo lease exclusivo simultáneamente.

---

# 176. Pool exhaustion test

Con capacity agotada:

```text
PoolWaitStarted
↓
timeout
↓
ConnectionPoolExhausted
↓
ConnectionAcquisitionFailed
```

---

# 177. Failover test

```text
Writer A fails
↓
ConnectionFailoverStarted
↓
Writer B elected/available
↓
ConnectionFailoverCompleted
↓
new acquisition
```

---

# 178. Event payload boundedness

No incluir:

```text
full session variable dump
full driver attributes
all prepared statements
full connection object
```

---

# 179. Session diagnostics summary

Podrá usar:

```php
final readonly class ConnectionSessionSummary
{
    public function __construct(
        public bool $transactionOpen,
        public bool $tenantStatePresent,
        public bool $schemaOverridePresent,
        public bool $temporaryStatePresent,
    ) {}
}
```

---

# 180. Summary ≠ reset mechanism

Solo observación.

---

# 181. Externalization

La mayoría de Connection Events serán:

```text
IN_PROCESS
```

---

# 182. External connection events

Si se externalizan:

```text
ConnectionPoolExhausted
ConnectionFailoverCompleted
ConnectionTainted
```

pueden ser útiles operacionalmente.

---

# 183. External payload safety

Nunca publicar:

```text
credentials
raw DSN
internal socket
private certificate data
```

---

# 184. Stable event names

Ejemplos:

```text
database.connection.resolution.started
database.connection.resolved

database.connection.acquisition.started
database.connection.acquired
database.connection.acquisition.failed

database.connection.opened
database.connection.reused

database.connection.reset.started
database.connection.reset.completed
database.connection.reset.failed

database.connection.released
database.connection.tainted
database.connection.discarded
database.connection.closed

database.connection.failover.started
database.connection.failover.completed
database.connection.failover.failed
```

---

# 185. Event names ≠ PHP FQCN

Permite refactor interno sin romper integraciones.

---

# 186. Directory structure

```text
src/Quantum/Database/Event/Connection/
│
├── Contract/
│   ├── ConnectionEvent.php
│   ├── ConnectionPoolEvent.php
│   └── ConnectionAttemptEvent.php
│
├── Event/
│   ├── ConnectionResolutionStarted.php
│   ├── ConnectionResolved.php
│   ├── ConnectionAcquisitionStarted.php
│   ├── ConnectionAcquired.php
│   ├── ConnectionAcquisitionFailed.php
│   ├── ConnectionPoolWaitStarted.php
│   ├── ConnectionPoolWaitCompleted.php
│   ├── ConnectionPoolExhausted.php
│   ├── ConnectionOpened.php
│   ├── ConnectionReused.php
│   ├── ConnectionConfigurationStarted.php
│   ├── ConnectionConfigured.php
│   ├── ConnectionResetStarted.php
│   ├── ConnectionReset.php
│   ├── ConnectionResetFailed.php
│   ├── ConnectionReleased.php
│   ├── ConnectionTainted.php
│   ├── ConnectionDiscarded.php
│   ├── ConnectionClosed.php
│   ├── ConnectionFailed.php
│   ├── ConnectionFailoverStarted.php
│   ├── ConnectionFailoverCompleted.php
│   └── ConnectionFailoverFailed.php
│
├── Model/
│   ├── ConnectionOperationId.php
│   ├── ConnectionAttemptId.php
│   ├── PhysicalConnectionId.php
│   ├── ConnectionLeaseId.php
│   ├── ConnectionEventContext.php
│   ├── ConnectionEventMetadata.php
│   ├── ConnectionLifecycleStage.php
│   ├── ConnectionFailureCategory.php
│   ├── ConnectionTimeoutKind.php
│   ├── ConnectionOrigin.php
│   ├── ConnectionLeaseType.php
│   ├── PhysicalConnectionState.php
│   └── ConnectionLeaseState.php
│
├── Summary/
│   ├── ConnectionTimingSummary.php
│   ├── ConnectionPoolSummary.php
│   ├── ConnectionFailureSummary.php
│   ├── ConnectionResetProfile.php
│   └── ConnectionSessionSummary.php
│
├── Security/
│   ├── ConnectionEventSanitizer.php
│   ├── ConnectionEventExposurePolicy.php
│   ├── ConnectionEventExposureLevel.php
│   └── SafeEndpointIdentity.php
│
├── Correlation/
│   ├── ConnectionEventCorrelator.php
│   └── ConnectionEventCorrelationContext.php
│
├── Diagnostics/
│   ├── ConnectionEventInspector.php
│   ├── ConnectionLifecycleValidator.php
│   └── ConnectionEventExplain.php
│
├── Testing/
│   ├── ConnectionEventRecorder.php
│   └── ConnectionEventAssertions.php
│
└── Exception/
    ├── ConnectionEventException.php
    ├── ConnectionEventLifecycleException.php
    ├── ConnectionEventSecurityException.php
    ├── ConnectionEventCorrelationException.php
    └── ConnectionEventCompatibilityException.php
```

---

# 187. Connection subsystem integration

```text
Connection Manager
      │
      ├── Resolution Events
      │
      ▼
Endpoint Router
      │
      ├── routing context
      │
      ▼
Connection Pool
      │
      ├── wait/exhaustion events
      │
      ▼
Connection Factory
      │
      ├── opened/reused events
      │
      ▼
Connection Lease
      │
      ├── acquired/released events
      │
      ▼
Reset Coordinator
      │
      ├── reset/taint/discard events
      │
      ▼
Physical Connection
```

---

# 188. Dependency rule

`Connection System` podrá emitir mediante:

```text
DatabaseEventDispatcher
```

pero no dependerá del Event System concreto de la aplicación.

---

# 189. Query integration

```text
QueryExecutionStarted
↓
ConnectionAcquired
↓
Query executes
↓
QueryExecuted
↓
ConnectionReleased
```

solo cuando no exista pinning más amplio.

---

# 190. Transaction integration

```text
TransactionStarted
↓
ConnectionAcquired
↓
Q1
Q2
Q3
↓
Commit/Rollback
↓
Reset
↓
Release
```

---

# 191. Streaming integration

```text
ConnectionAcquired
↓
Statement Executed
↓
Streaming Result Open
↓
consume
↓
Result Closed
↓
Reset
↓
ConnectionReleased
```

---

# 192. Persistent runtime integration

```text
Request A
↓
Acquire P1
↓
Use
↓
Reset
↓
Release

Request B
↓
Reuse P1
↓
Use
↓
Reset
↓
Release
```

Nunca:

```text
Request A state
↓
Request B
```

---

# 193. Architectural invariants

## DB-CEVENT-001
Connection Event será distinto de Connection.

## DB-CEVENT-002
Connection Event será distinto de Physical Connection.

## DB-CEVENT-003
Logical Connection será distinta de Physical Connection.

## DB-CEVENT-004
Connection Lease será distinto de Physical Connection.

## DB-CEVENT-005
Connection Pool será distinto de Connection.

## DB-CEVENT-006
Endpoint será distinto de Connection.

## DB-CEVENT-007
Connection Event será distinto de Query Event.

## DB-CEVENT-008
Connection Event será distinto de Transaction Event.

## DB-CEVENT-009
Connection Event será distinto de Telemetry Event.

## DB-CEVENT-010
ConnectionAcquired será distinto de ConnectionOpened.

## DB-CEVENT-011
ConnectionReleased será distinto de ConnectionClosed.

## DB-CEVENT-012
ConnectionReturned será distinto de ConnectionReset.

## DB-CEVENT-013
ConnectionFailure será distinto de QueryFailure.

## DB-CEVENT-014
Connection listeners no modificarán endpoint arbitrariamente.

## DB-CEVENT-015
Connection listeners no modificarán credentials.

## DB-CEVENT-016
Connection listeners no modificarán lease ownership.

## DB-CEVENT-017
Connection listeners no modificarán transaction state arbitrariamente.

## DB-CEVENT-018
ConnectionOperationId será distinto de PhysicalConnectionId.

## DB-CEVENT-019
PhysicalConnectionId será distinto de ConnectionLeaseId.

## DB-CEVENT-020
Una physical connection podrá tener múltiples leases secuenciales.

## DB-CEVENT-021
Connection context será scope-local.

## DB-CEVENT-022
Connection context no será static global state.

## DB-CEVENT-023
EndpointRole será explícito.

## DB-CEVENT-024
Resolution será distinta de Routing.

## DB-CEVENT-025
Acquisition será distinta de Open.

## DB-CEVENT-026
Pool wait será distinta de physical open.

## DB-CEVENT-027
Pool exhaustion será distinta de database outage.

## DB-CEVENT-028
ConnectionAcquired se emitirá solo con lease usable.

## DB-CEVENT-029
ConnectionOpened se emitirá solo para nueva physical connection.

## DB-CEVENT-030
ConnectionReused se emitirá para physical reuse.

## DB-CEVENT-031
Reused no implicará clean state sin reset/validation.

## DB-CEVENT-032
Connection configuration será responsibility directa del Connection System.

## DB-CEVENT-033
Reset será correctness-critical.

## DB-CEVENT-034
Reset correctness no dependerá de optional listeners.

## DB-CEVENT-035
ConnectionReset se emitirá solo tras reset confirmado.

## DB-CEVENT-036
Reset failure impedirá reuse normal por default.

## DB-CEVENT-037
Reset failure podrá taint connection.

## DB-CEVENT-038
Tainted será distinto de Closed.

## DB-CEVENT-039
Tainted será distinto de Failed Operation.

## DB-CEVENT-040
Discarded será distinto de Closed.

## DB-CEVENT-041
Released será distinto de Returned Reusable.

## DB-CEVENT-042
Release terminará caller ownership.

## DB-CEVENT-043
Use-after-release será error.

## DB-CEVENT-044
Physical lifecycle state será authority del Connection System.

## DB-CEVENT-045
Event state no reemplazará canonical connection state.

## DB-CEVENT-046
Connection failure category será explícita.

## DB-CEVENT-047
Authentication errors serán sanitizados.

## DB-CEVENT-048
TLS errors serán sanitizados.

## DB-CEVENT-049
Raw DSN no será expuesto por default.

## DB-CEVENT-050
Password nunca será event payload.

## DB-CEVENT-051
Endpoint fingerprint podrá usarse en lugar de hostname.

## DB-CEVENT-052
Connection timeout kind será explícito.

## DB-CEVENT-053
Pool wait timeout será distinto de query timeout.

## DB-CEVENT-054
Connect timeout será distinto de reset timeout.

## DB-CEVENT-055
Connection timing usará monotonic clock cuando sea posible.

## DB-CEVENT-056
QueryId podrá correlacionarse con ConnectionOperationId.

## DB-CEVENT-057
Una connection lease podrá servir múltiples queries.

## DB-CEVENT-058
ConnectionAcquired no se emitirá necesariamente por cada query.

## DB-CEVENT-059
Streaming podrá prolongar lease.

## DB-CEVENT-060
QueryExecuted será distinto de ConnectionReleased.

## DB-CEVENT-061
Transaction podrá fijar connection affinity.

## DB-CEVENT-062
Event System no impondrá transaction affinity.

## DB-CEVENT-063
Unknown transaction outcome podrá taint connection.

## DB-CEVENT-064
Rollback failure podrá taint connection.

## DB-CEVENT-065
Savepoint no cambiará ownership automáticamente.

## DB-CEVENT-066
Read/write routing será observable.

## DB-CEVENT-067
Replica selection no implicará freshness.

## DB-CEVENT-068
Failover será distinto de Retry.

## DB-CEVENT-069
Failover dentro de active transaction no será transparente por default.

## DB-CEVENT-070
New connection no heredará transaction state anterior.

## DB-CEVENT-071
FailoverCompleted será distinto de operation recovered.

## DB-CEVENT-072
Authority epoch podrá ser observable.

## DB-CEVENT-073
Shard context será connection-bound cuando corresponda.

## DB-CEVENT-074
Cross-shard query podrá adquirir múltiples connections.

## DB-CEVENT-075
Parent operation correlation será bounded.

## DB-CEVENT-076
Tenant state será reset obligatorio cuando sea session-stateful.

## DB-CEVENT-077
Tenant state no se filtrará entre leases.

## DB-CEVENT-078
Tenant identity podrá sanitizarse.

## DB-CEVENT-079
Worker lifetime será distinto de connection lifetime.

## DB-CEVENT-080
Connection lifetime será distinta de request lifetime.

## DB-CEVENT-081
Persistent workers no conservarán request-scoped DB state.

## DB-CEVENT-082
Reset deberá limpiar state requerido.

## DB-CEVENT-083
OpenSwoole coroutine leases serán aislados.

## DB-CEVENT-084
Physical connection exclusivity será default.

## DB-CEVENT-085
Shared connection concurrency requerirá capability explícita.

## DB-CEVENT-086
Pool events podrán ser subfamilia de Connection Events.

## DB-CEVENT-087
Pool ID será stable logical identity.

## DB-CEVENT-088
Pool metrics no serán obligatorios en cada event.

## DB-CEVENT-089
Reuse rate será derivable sin alterar lifecycle.

## DB-CEVENT-090
High open rate será observable.

## DB-CEVENT-091
High discard rate será observable.

## DB-CEVENT-092
Telemetry derivará de Connection Events.

## DB-CEVENT-093
Telemetry no alterará connection state.

## DB-CEVENT-094
Hot-path listeners serán ligeros.

## DB-CEVENT-095
Synchronous network I/O en acquire/release listeners será desaconsejado.

## DB-CEVENT-096
Exposure level será configurable.

## DB-CEVENT-097
Production default será seguro.

## DB-CEVENT-098
Username podrá ocultarse.

## DB-CEVENT-099
TLS state podrá resumirse sin private material.

## DB-CEVENT-100
Retryable hint no decidirá retry.

## DB-CEVENT-101
Connection retry ownership pertenecerá al resilience subsystem.

## DB-CEVENT-102
Connection attempts serán correlacionables.

## DB-CEVENT-103
Attempt failure será distinto de final acquisition failure.

## DB-CEVENT-104
Pool checkout fallback será observable.

## DB-CEVENT-105
Health checks podrán tener origin INTERNAL.

## DB-CEVENT-106
Health-check queries no contaminarán business query analytics cuando sea posible.

## DB-CEVENT-107
Connection origin será semantic category.

## DB-CEVENT-108
Migrations podrán usar dedicated connection.

## DB-CEVENT-109
Administrative connections serán distinguibles.

## DB-CEVENT-110
Dedicated connection seguirá siendo observable.

## DB-CEVENT-111
Lease type será explícito.

## DB-CEVENT-112
Lifecycle validation será soportable.

## DB-CEVENT-113
Released before Acquired será lifecycle violation.

## DB-CEVENT-114
Closed then Reused será lifecycle violation.

## DB-CEVENT-115
Tainted then ReturnedReusable será lifecycle violation.

## DB-CEVENT-116
Use-after-release será lifecycle violation.

## DB-CEVENT-117
Lifecycle validation completa podrá ser development/testing only.

## DB-CEVENT-118
Pool reuse será testeable.

## DB-CEVENT-119
Reset failure path será testeable.

## DB-CEVENT-120
Unknown transaction taint path será testeable.

## DB-CEVENT-121
Tenant leakage será testeable.

## DB-CEVENT-122
Coroutine lease isolation será testeable.

## DB-CEVENT-123
Pool exhaustion será testeable.

## DB-CEVENT-124
Failover lifecycle será testeable.

## DB-CEVENT-125
Event payload será bounded.

## DB-CEVENT-126
Full session state dump no será event payload.

## DB-CEVENT-127
Session summary será distinta de reset mechanism.

## DB-CEVENT-128
Connection Events serán in-process por default.

## DB-CEVENT-129
Externalization será opt-in.

## DB-CEVENT-130
External connection events no expondrán secrets.

## DB-CEVENT-131
Stable event name será distinto de FQCN.

## DB-CEVENT-132
Connection System dependerá de DatabaseEventDispatcher contract.

## DB-CEVENT-133
Connection System no dependerá del global Event System concreto.

## DB-CEVENT-134
Query integration no cambiará connection ownership semantics.

## DB-CEVENT-135
Transaction integration no cambiará connection ownership semantics.

## DB-CEVENT-136
Streaming integration preservará lease hasta release real.

## DB-CEVENT-137
Persistent runtime reuse requerirá reset.

## DB-CEVENT-138
Request A state no podrá aparecer en Request B.

## DB-CEVENT-139
FrankenPHP reuse será seguro.

## DB-CEVENT-140
RoadRunner reuse será seguro.

## DB-CEVENT-141
OpenSwoole reuse será coroutine-safe.

## DB-CEVENT-142
Connection Event System no abrirá connections.

## DB-CEVENT-143
Connection Event System no cerrará connections.

## DB-CEVENT-144
Connection Event System no resolverá endpoints.

## DB-CEVENT-145
Connection Event System no seleccionará replicas.

## DB-CEVENT-146
Connection Event System no administrará pools.

## DB-CEVENT-147
Connection Event System no realizará reset.

## DB-CEVENT-148
Connection Event System no decidirá failover.

## DB-CEVENT-149
Connection Event System no decidirá retry.

## DB-CEVENT-150
Connection Event System no administrará transactions.

## DB-CEVENT-151
Connection Event System no administrará tenant state.

## DB-CEVENT-152
Connection Event System no reemplazará Connection Manager.

## DB-CEVENT-153
Connection Event System no reemplazará Pool.

## DB-CEVENT-154
Connection Event System no reemplazará Driver.

## DB-CEVENT-155
Connection Event System preservará resource ownership boundaries.

## DB-CEVENT-156
Connection Event System será explainable.

## DB-CEVENT-157
Acquisition wait será explainable.

## DB-CEVENT-158
Reuse/open distinction será explainable.

## DB-CEVENT-159
Reset outcome será explainable.

## DB-CEVENT-160
Taint/discard cause será explainable.

## DB-CEVENT-161
Failover transition será explainable.

## DB-CEVENT-162
Security tendrá prioridad sobre diagnostics richness.

## DB-CEVENT-163
Reset correctness tendrá prioridad sobre reuse performance.

## DB-CEVENT-164
Connection safety tendrá prioridad sobre pool retention.

## DB-CEVENT-165
UNKNOWN connection/session state no será tratado como reusable.

---

# 194. Modelo formal

Sea:

```text
P
=
physical connection
```

y una secuencia de leases:

```text
L(P)
=
{L1, L2, ..., Ln}
```

Para cada lease:

```text
Acquire(Li)
→
Use(Li)
→
Reset(P)
→
Release(Li)
```

cuando la policy requiera reset antes de retorno reusable.

---

# 195. Reuse condition

Una conexión física solo podrá volver al conjunto reusable si:

```text
Healthy(P)
∧
ResetConfirmed(P)
∧
¬Tainted(P)
```

Formalmente:

```text
Reusable(P)
=
Healthy(P)
∧
ResetConfirmed(P)
∧
NotTainted(P)
```

---

# 196. Reset failure

Si:

```text
Reset(P) = FAILED
```

entonces por default:

```text
Reusable(P) = false
```

y:

```text
Discard(P)
```

deberá ocurrir.

---

# 197. Taint model

```text
Tainted(P)
=
UnknownProtocolState
∨
UnknownTransactionState
∨
ResetFailure
∨
FatalDriverError
∨
PolicyDefinedUnsafeState
```

---

# 198. Acquisition model

Para una acquisition lógica `A`:

```text
Acquire(A)
=
Resolve
+
Route
+
PoolWait?
+
OpenOrReuse
+
Configure/Validate
+
Lease
```

---

# 199. Timing model

```text
T_acquire
=
T_resolution
+
T_pool_wait
+
T_open_or_reuse
+
T_configuration
+
T_checkout
```

según fases aplicables.

---

# 200. Pool model

Sea:

```text
C
=
pool capacity
```

```text
I
=
in-use connections
```

```text
D
=
idle reusable connections
```

Entonces idealmente:

```text
I + D <= C
```

ignorando conexiones en transición temporal según implementación.

---

# 201. Connection reuse

Si una acquisition usa una conexión idle:

```text
OpenedDuringAcquisition = false
ReusedDuringAcquisition = true
```

---

# 202. New connection

Si el pool necesita crecer:

```text
OpenedDuringAcquisition = true
ReusedDuringAcquisition = false
```

---

# 203. Failover relation

Sea endpoint:

```text
E1
```

que deja de ser usable.

El failover puede resolver:

```text
E1 → E2
```

pero:

```text
OperationOutcome(E1)
```

continúa siendo independiente de:

```text
EndpointTransition(E1,E2)
```

---

# 204. Arquitectura final

```text
                      Connection Request
                             │
                             ▼
                     Connection Resolver
                             │
                    Resolution Events
                             │
                             ▼
                       Endpoint Router
                             │
                             ▼
                       Connection Pool
                    ┌────────┴────────┐
                    ▼                 ▼
                Idle Reuse         Open New
                    │                 │
          ConnectionReused    ConnectionOpened
                    │                 │
                    └────────┬────────┘
                             ▼
                     Configuration
                             │
                             ▼
                   ConnectionAcquired
                             │
                             ▼
                         Lease Use
                             │
                             ▼
                    ConnectionReset
                      ┌──────┴───────┐
                      ▼              ▼
                   Success         Failure
                      │              │
                      ▼              ▼
              ConnectionReleased   Tainted
                      │              │
                      ▼              ▼
                Return to Pool   Discarded
                                     │
                                     ▼
                                   Closed
```

---

# 205. Persistent runtime lifecycle

```text
Worker Starts
    ↓
Pool Created
    ↓
Request A
    ↓
Acquire P1
    ↓
Use Tenant A
    ↓
Reset
    ↓
Release P1
    ↓
Request B
    ↓
Reuse P1
    ↓
Configure Tenant B
    ↓
Use
    ↓
Reset
    ↓
Release
    ↓
Worker Continues
```

El evento system deberá permitir auditar que:

```text
Reset(A→neutral)
```

ocurrió antes de:

```text
Configure(B)
```

---

# 206. Regla maestra final

El sistema deberá preservar siempre:

```text
Connection Event
≠
Connection
```

```text
Logical Connection
≠
Physical Connection
```

```text
ConnectionAcquired
≠
ConnectionOpened
```

```text
ConnectionReleased
≠
ConnectionClosed
```

```text
ConnectionReset
≠
ConnectionReleased
```

```text
ConnectionFailure
≠
QueryFailure
```

```text
Pool Exhausted
≠
Database Down
```

```text
Tainted
≠
Closed
```

```text
Discarded
≠
Closed
```

```text
Lease
≠
Physical Connection
```

```text
Failover
≠
Retry
```

```text
Failover Completed
≠
Original Operation Recovered
```

```text
Replica Selected
≠
Replica Fresh Enough
```

```text
Worker Lifetime
≠
Connection Lifetime
```

```text
Request Lifetime
≠
Connection Lifetime
```

```text
Connection Reused
≠
Session State Safe
```

y, especialmente:

```text
Unknown Session State
≠
Reusable Connection
```

---

# 207. Resultado arquitectónico

Con `Database Connection Event System`, VoltStack podrá observar de forma precisa:

```text
connection resolution
pool wait
pool exhaustion
physical opens
physical reuse
lease acquisition
session configuration
reset
release
taint
discard
close
failover
```

sin mezclar:

```text
resource lifecycle
```

con:

```text
query lifecycle
transaction lifecycle
telemetry
routing policies
```

La arquitectura conservará:

```text
Connection Manager
=
authority over connection lifecycle

Pool
=
authority over reusable physical resources

Driver
=
authority over protocol connection

Event System
=
observer
```

---

# 208. Bloque 20 — Estado

```text
BLOCK 20 — EVENTS

✓ 209_DATABASE_EVENT_ARCHITECTURE.md
✓ 210_DATABASE_QUERY_EVENT_SYSTEM.md
✓ 211_DATABASE_CONNECTION_EVENT_SYSTEM.md
○ 212_DATABASE_TRANSACTION_EVENT_PIPELINE.md
○ 213_DATABASE_ENTITY_LIFECYCLE_EVENT_SYSTEM.md
○ 214_DATABASE_PERSISTENCE_EVENT_SYSTEM.md
○ 215_DATABASE_EVENT_EXTENSION_SYSTEM.md
```

---

# 209. Siguiente documento

```text
212_DATABASE_TRANSACTION_EVENT_PIPELINE.md
```

El siguiente documento definirá el pipeline de eventos de transacciones:

```text
Transaction Begin
↓
Logical/Nested Scope
↓
Savepoints
↓
Statements
↓
Commit Attempt / Rollback Attempt
↓
Confirmed Outcome
↓
Deferred Events
↓
AfterCommit / AfterRollback / AfterCompletion
```

incluyendo:

```text
TransactionStarted
TransactionNestedScopeStarted
SavepointCreated
SavepointReleased
SavepointRolledBack
TransactionCommitStarted
TransactionCommitted
TransactionCommitFailed
TransactionRollbackStarted
TransactionRolledBack
TransactionRollbackFailed
TransactionRetryScheduled
TransactionRetryStarted
TransactionOutcomeUnknown
AfterCommit delivery
AfterRollback delivery
```

bajo la regla central:

> **El Transaction Event Pipeline deberá describir con precisión los intentos y outcomes de una transacción sin confundir `flush`, statement success, savepoint release, nested scope completion o listener execution con un commit físico confirmado.**