# 11_DATABASE_CONNECTION_SYSTEM.md

# VoltStack Quantum Database
## Connection System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 11 — Database Connection System  
**Estado:** Architecture Specification  
**Nivel:** Infrastructure Architecture  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura oficial del sistema de conexiones de:

```text
VoltStack/Quantum/Database
```

El Connection System será responsable de representar y administrar el acceso lógico de VoltStack a una base de datos sin exponer directamente los recursos nativos utilizados por el Driver.

Su posición arquitectónica será:

```text
Execution
    │
    ▼
Logical Connection
    │
    ▼
Connection Infrastructure
    │
    ▼
Driver
    │
    ▼
Native / Physical Connection
    │
    ▼
Database
```

El sistema deberá funcionar correctamente tanto en aplicaciones PHP tradicionales como en runtimes persistentes:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 2. Definición de Connection

Dentro de VoltStack:

> Una Connection representa un punto lógico y administrado de acceso a una base de datos.

No deberá confundirse con el socket, `PDO` o recurso físico subyacente.

Por tanto:

```text
Connection
≠
PDO
≠
NativeConnection
```

---

# 3. Logical Connection

La `Logical Connection` representa:

```text
connection identity
configuration reference
database role
driver association
platform association
dialect association
lifecycle policy
transaction affinity
runtime context
```

Puede existir sin que todavía exista una conexión física abierta.

---

# 4. Physical Connection

Una `Physical Connection` representa el recurso realmente conectado al motor.

Ejemplos:

```text
PDO
mysqli connection
pgsql connection
SQLite3 connection
future async client connection
```

En VoltStack estará encapsulada mediante:

```text
NativeConnectionInterface
```

---

# 5. Separación fundamental

```text
Logical Connection
        │
        ▼
Connection Infrastructure
        │
        ▼
Driver
        │
        ▼
NativeConnection
```

La aplicación no deberá trabajar normalmente con `NativeConnection`.

---

# 6. Connection no es Driver

```text
Connection
≠
Driver
```

Connection responde:

> ¿Qué acceso lógico estoy utilizando y cuál es su lifecycle?

Driver responde:

> ¿Cómo creo y utilizo el canal nativo?

---

# 7. Connection no es Platform

```text
Connection
≠
Platform
```

Connection representa acceso.

Platform representa:

```text
database semantics
capabilities
server behavior
version-specific behavior
```

---

# 8. Connection no es Dialect

```text
Connection
≠
Dialect
```

Connection no genera SQL.

Dialect define sintaxis.

---

# 9. Connection no es Transaction

Una Connection puede participar en una Transaction, pero no representa por sí misma el modelo transaccional.

```text
Transaction Manager
        │
        ▼
Transaction Context
        │
        ▼
Connection
```

---

# 10. Objetivos principales

El Connection System deberá proporcionar:

```text
logical connection abstraction
lazy physical acquisition
connection lifecycle
connection roles
connection state
connection leasing
connection reset
connection release
transaction affinity
read/write compatibility
persistent runtime safety
failure handling
```

---

# 11. No objetivos

Connection no deberá encargarse de:

```text
Query Builder
AST
Semantic Analysis
Optimization
SQL compilation
ORM
Entity hydration
UnitOfWork
IdentityMap
Schema modeling
Migration planning
business validation
authorization
```

---

# 12. Arquitectura general

```text
Connection
├── Contract
├── Identity
├── Configuration
├── Factory
├── Resolver
├── Manager
├── Handle
├── Lease
├── State
├── Role
├── Lifecycle
├── Reset
├── Pool Integration
├── Transaction Affinity
├── Failure
└── Diagnostics
```

---

# 13. ConnectionInterface

Contrato conceptual:

```php
interface ConnectionInterface
{
    public function identity(): ConnectionIdentity;

    public function role(): ConnectionRole;

    public function state(): ConnectionState;

    public function isConnected(): bool;
}
```

La API final podrá exponer operaciones adicionales mediante interfaces especializadas.

---

# 14. Interface segregation

Se evitará convertir `ConnectionInterface` en:

```text
God Interface
```

con métodos para:

```text
queries
transactions
schema
ORM
pool
telemetry
migration
```

Podrán existir contratos especializados.

---

# 15. Connection identity

Cada conexión lógica deberá poseer una identidad estable dentro de la configuración.

Ejemplos:

```text
default
primary
analytics
reporting
legacy
tenant
```

---

# 16. ConnectionIdentity

Conceptualmente:

```php
final readonly class ConnectionIdentity
{
    public function __construct(
        public string $name,
    ) {}
}
```

---

# 17. Identity no contiene secretos

No deberá contener:

```text
password
DSN credentials
tokens
```

---

# 18. Connection definition

Una conexión configurada podrá representarse mediante:

```text
ConnectionDefinition
```

que describe:

```text
identity
driver
database target
role
pool policy
timeouts
platform hints
runtime policy
```

---

# 19. Definition vs runtime Connection

Separación:

```text
ConnectionDefinition
      │
      ▼
ConnectionFactory
      │
      ▼
Runtime Connection
```

La definición es configuración.

La Connection es runtime infrastructure.

---

# 20. Connection role

VoltStack deberá modelar explícitamente roles como:

```text
PRIMARY
REPLICA
READ_ONLY
WRITE
READ_WRITE
ADMIN
MIGRATION
```

La lista exacta podrá evolucionar.

---

# 21. Role is not server name

Incorrecto:

```text
connection name = mysql2
therefore replica
```

Correcto:

```text
ConnectionRole::REPLICA
```

---

# 22. Connection capabilities

Una conexión efectiva puede restringir capabilities.

Ejemplo:

```text
Platform:
WRITE supported

Replica Connection:
WRITE disabled
```

---

# 23. Connection state model

Estados conceptuales:

```text
DEFINED
   │
   ▼
IDLE
   │
   ▼
ACQUIRING
   │
   ▼
READY
   │
   ▼
LEASED
   │
   ├── RESETTING
   │       │
   │       ▼
   │      IDLE
   │
   └── DISCARDING
           │
           ▼
         CLOSED
```

---

# 24. State machine

Los cambios de estado deberán ser explícitos.

No deberá inferirse estado únicamente con:

```php
$pdo !== null
```

---

# 25. ConnectionState

Podrá modelarse mediante:

```php
enum ConnectionState
{
    case DEFINED;
    case IDLE;
    case ACQUIRING;
    case READY;
    case LEASED;
    case RESETTING;
    case BROKEN;
    case CLOSED;
}
```

---

# 26. Broken state

Una conexión podrá entrar en:

```text
BROKEN
```

cuando el recurso físico ya no sea seguro para reutilización.

Ejemplos:

```text
network loss
protocol corruption
failed reset
uncertain transaction state
server disconnect
```

---

# 27. Lazy connection acquisition

VoltStack utilizará adquisición diferida por defecto.

```text
Application boot
      │
      ▼
Connection configured
      │
      ▼
No physical connection
      │
      ▼
First operation
      │
      ▼
Acquire physical connection
```

---

# 28. No eager connection at bootstrap

Bootstrap no deberá abrir todas las conexiones configuradas.

Esto es especialmente importante para:

```text
CLI
tests
workers
applications with multiple databases
optional integrations
```

---

# 29. Lazy connection benefits

```text
faster bootstrap
lower resource usage
unused connections remain unopened
better worker scalability
better failure isolation
```

---

# 30. Connection acquisition

La adquisición conceptual será:

```text
Logical Connection
      │
      ▼
Connection Resource Provider
      │
      ├── Pool
      │
      └── Driver connect()
      │
      ▼
NativeConnection
```

---

# 31. Resource provider

Connection no deberá saber necesariamente si el recurso procede de:

```text
new connection
pool
persistent transport
specialized runtime pool
```

---

# 32. ConnectionHandle

Se recomienda introducir un objeto interno:

```text
ConnectionHandle
```

que represente la asociación temporal entre una conexión lógica y un recurso físico.

---

# 33. Handle architecture

```text
Logical Connection
       │
       ▼
ConnectionHandle
       │
       ▼
NativeConnection
```

El Handle podrá contener:

```text
resource identity
lease information
state metadata
timestamps
role
pool origin
```

---

# 34. ConnectionLease

Cuando un recurso físico sea adquirido temporalmente podrá existir:

```text
ConnectionLease
```

---

# 35. Lease semantics

Un Lease significa:

> Este contexto posee temporalmente el derecho de utilizar este recurso físico.

---

# 36. Lease ownership

```text
ExecutionScope
      │
      ▼
ConnectionLease
      │
      ▼
Physical Connection
```

---

# 37. Lease release

Al terminar:

```text
Lease
  │
  ▼
reset if required
  │
  ├── reusable
  │      ▼
  │     Pool
  │
  └── unsafe
         ▼
       Close
```

---

# 38. Lease is not Connection

Una Logical Connection puede existir durante toda la aplicación.

Un Lease puede durar:

```text
one query
one transaction
one request
one streaming operation
```

según contexto.

---

# 39. Lease duration policy

La duración deberá determinarse mediante una política explícita.

No deberá quedar accidentalmente ligada a la vida de un objeto Facade.

---

# 40. Query-scoped lease

Una operación simple puede usar:

```text
Acquire
   │
Execute
   │
Reset
   │
Release
```

---

# 41. Transaction-scoped lease

Una Transaction requiere afinidad:

```text
BEGIN
  │
  ▼
Physical Connection A
  │
  ├── Query 1
  ├── Query 2
  ├── Query 3
  │
COMMIT
  │
  ▼
Release A
```

No puede cambiar silenciosamente a Connection B.

---

# 42. Transaction affinity

Regla crítica:

> Una transacción estará fijada a un único recurso físico compatible durante toda su vida.

---

# 43. Transaction pinning

```text
TransactionContext
       │
       ▼
ConnectionLease A
       │
       ▼
NativeConnection A
```

Mientras exista la transacción:

```text
queries
savepoints
commit
rollback
```

deberán utilizar A.

---

# 44. Nested transaction affinity

Nested transactions/savepoints también deberán permanecer sobre el mismo recurso.

---

# 45. Connection switching during transaction

Prohibido:

```text
BEGIN on Primary A
Query on Primary B
COMMIT on Primary A
```

---

# 46. Read/write routing and transactions

Fuera de transaction:

```text
SELECT → replica
WRITE  → primary
```

puede ser válido.

Dentro de una transaction:

```text
all compatible operations
→ pinned connection
```

---

# 47. Read-after-write consistency

Connection System deberá permitir que capas superiores implementen:

```text
sticky primary
```

después de writes cuando sea necesario.

La política completa se documentará posteriormente.

---

# 48. ConnectionManager

`ConnectionManager` será el coordinador principal del dominio Connection.

Responsabilidades:

```text
resolve logical connections
provide configured connection instances
coordinate acquisition
coordinate release
coordinate connection definitions
expose connection identities
```

---

# 49. ConnectionManager no es Executor

No deberá tener API principal como:

```php
$manager->select(...)
$manager->insert(...)
```

Eso pertenece a Execution/Query API.

---

# 50. ConnectionManager no es DriverRegistry

Podrá usar DriverResolver, pero no deberá absorber su responsabilidad.

---

# 51. ConnectionManager no es Pool

Pool administra recursos físicos reutilizables.

Manager administra conexiones lógicas.

---

# 52. Manager architecture

```text
ConnectionManager
      │
      ├── ConnectionDefinitionRegistry
      ├── ConnectionFactory
      ├── ConnectionResolver
      └── Resource Provider
```

---

# 53. ConnectionFactory

Responsable de construir la representación lógica de una Connection.

Conceptualmente:

```php
interface ConnectionFactoryInterface
{
    public function create(
        ConnectionDefinition $definition
    ): ConnectionInterface;
}
```

---

# 54. Factory does not necessarily connect

`create()` no implica:

```text
open socket
```

Puede crear una conexión lógica lazy.

---

# 55. PhysicalConnectionFactory

Si se requiere, deberá ser otro concepto.

```text
ConnectionFactory
→ logical connection

Driver / ResourceFactory
→ physical connection
```

---

# 56. ConnectionResolver

Resolverá cuál conexión lógica utilizar.

Ejemplo:

```php
$connections->connection('analytics');
```

---

# 57. Default connection

Podrá existir:

```text
default connection identity
```

resuelta por configuración.

---

# 58. Named connections

Ejemplo:

```yaml
database:
  default: primary

  connections:
    primary:
      driver: pgsql

    analytics:
      driver: mysql
```

---

# 59. No configuration lookup in hot path

La configuración deberá normalizarse durante bootstrap.

Runtime deberá trabajar con:

```text
ConnectionDefinition
```

compiladas/tipadas.

---

# 60. Connection registry

Podrá existir:

```text
ConnectionDefinitionRegistry
```

para definiciones.

No deberá almacenar conexiones físicas globales sin lifecycle explícito.

---

# 61. Registry freeze

Después del bootstrap:

```text
ConnectionDefinitionRegistry
→ frozen
```

por defecto.

---

# 62. Dynamic tenant connections

Multitenancy puede necesitar definiciones dinámicas.

Esto no deberá obligar a hacer mutable el registry global.

---

# 63. Dynamic connection scope

Podrá utilizarse:

```text
TenantConnectionResolver
      │
      ▼
Scoped ConnectionDefinition
```

sin registrar permanentemente cada tenant en el global registry.

---

# 64. Connection scope

Una Logical Connection podrá tener distintos scopes de infraestructura.

Ejemplos:

```text
application definition
worker resource
request handle
transaction lease
operation statement
```

---

# 65. Scope separation

```text
Application
├── ConnectionDefinition
├── DriverRegistry
└── Platform descriptors

Worker
└── Connection Pool

Request
└── DatabaseContext

Transaction
└── pinned ConnectionLease

Operation
├── Statement
└── Result/Cursor
```

---

# 66. DatabaseContext

El `DatabaseContext` deberá contener referencias al estado Database correspondiente al execution scope.

Podrá conocer:

```text
active connection leases
transaction context
tenant database context
sticky routing state
resource cleanup registrations
```

---

# 67. DatabaseContext no es Service Locator

No deberá convertirse en:

```php
$context->getAnything();
```

Sólo contendrá estado contextual claramente definido.

---

# 68. Connection context

Puede existir un componente específico:

```text
ConnectionContext
```

para ownership de leases y state contextual.

---

# 69. Persistent runtime model

Con FrankenPHP:

```text
Worker
  │
  ├── DriverRegistry
  ├── Connection Definitions
  ├── Pool
  │
  ├── Request A
  │      └── DatabaseContext A
  │
  ├── RESET
  │
  └── Request B
         └── DatabaseContext B
```

---

# 70. State leakage prohibition

Request B nunca deberá heredar accidentalmente de A:

```text
transaction
selected tenant
session variables
temporary tables
locks
connection role
unconsumed cursor
```

---

# 71. Connection reset

Antes de reutilizar una conexión física deberá restaurarse a un estado seguro.

---

# 72. Reset pipeline

Conceptualmente:

```text
Operation ends
     │
     ▼
Inspect Connection State
     │
     ├── active transaction?
     ├── open cursor?
     ├── changed session?
     ├── temporary state?
     ├── driver reset available?
     │
     ▼
Reset
     │
     ├── success → reusable
     └── failure → discard
```

---

# 73. Reset ownership

`ConnectionResetSystem` coordina.

Driver ejecuta primitivas nativas.

---

# 74. Reset strategies

Podrán existir:

```text
LIGHT
STANDARD
FULL
DISCARD
```

conceptualmente.

La política definitiva dependerá de Driver/Platform/Pool.

---

# 75. Rollback before release

Una conexión con transaction activa no deberá regresar al pool.

Deberá:

```text
rollback
```

o ser descartada si el estado es incierto.

---

# 76. Open cursors

Los cursores pendientes deberán:

```text
close
consume safely
or force discard
```

según Driver capability.

---

# 77. Session variables

Cambios como:

```text
SET search_path
SET timezone
SET role
SET sql_mode
```

deberán formar parte del state tracking/reset cuando VoltStack los gestione.

---

# 78. Tenant state

Si el tenant modifica:

```text
schema
database
role
session variable
```

deberá resetearse antes de reutilización.

---

# 79. Temporary tables

Las temporary tables son connection-scoped en muchos motores.

Su existencia puede afectar reutilización.

La Platform deberá informar comportamiento relevante.

---

# 80. Advisory locks

Locks ligados a la sesión también deberán considerarse en reset/lifecycle.

---

# 81. Connection tainting

Podrá existir el concepto:

```text
TAINTED
```

para un recurso cuyo estado ya no puede garantizarse como limpio.

---

# 82. Tainted resource

```text
NativeConnection
      │
      ▼
TAINTED
      │
      ▼
DISCARD
```

No deberá volver al pool.

---

# 83. Connection health

Se distinguirá:

```text
logical state
physical health
```

Una Connection lógica puede estar disponible aunque su recurso anterior haya fallado.

---

# 84. Physical health checks

Podrán incluir:

```text
driver ping
lightweight query
native status
connection age
idle timeout knowledge
```

---

# 85. Health check cost

No deberá ejecutarse necesariamente un:

```text
SELECT 1
```

antes de cada query.

La estrategia deberá considerar coste y runtime.

---

# 86. Stale connections

Un pool podrá detectar conexiones potencialmente expiradas por:

```text
age
idle duration
server timeout
previous failure
```

---

# 87. Connection acquisition failure

Ejemplos:

```text
driver unavailable
invalid credentials
network failure
TLS failure
pool exhausted
server unavailable
timeout
```

---

# 88. Failure normalization

Flujo:

```text
Driver Native Error
       │
       ▼
Normalized Driver Failure
       │
       ▼
Connection Failure Classifier
       │
       ▼
Connection Exception
```

---

# 89. Connection exception hierarchy

Conceptualmente:

```text
DatabaseConnectionException
├── ConnectionNotConfiguredException
├── ConnectionUnavailableException
├── ConnectionAcquisitionException
├── ConnectionAuthenticationException
├── ConnectionTimeoutException
├── ConnectionLostException
├── ConnectionResetException
├── ConnectionPoolExhaustedException
├── ConnectionRoleViolationException
└── ConnectionStateException
```

---

# 90. Connection loss during query

Si se pierde la conexión:

```text
Executor
   │
   ▼
ConnectionLostException
```

Resilience decide si puede reintentarse.

Connection no debe repetir automáticamente una operación cuya semántica desconoce.

---

# 91. No implicit write retry

Una Connection nunca deberá asumir:

```text
connection lost
→ repeat INSERT
```

porque podría duplicar efectos.

---

# 92. Connection recovery

Connection infrastructure podrá:

```text
discard broken resource
acquire replacement
```

pero la repetición de la operación pertenece a Resilience/Execution policy.

---

# 93. Connection role enforcement

Una conexión `READ_ONLY` deberá rechazar operaciones de escritura cuando el sistema pueda detectarlas estructuralmente.

---

# 94. Enforcement layers

La validación puede ocurrir en:

```text
Planner
Routing
Executor
Connection capability check
```

según información disponible.

---

# 95. Defense in depth

Aunque Planner haya elegido una replica, Connection puede mantener una restricción:

```text
WRITE = DISABLED
```

como protección adicional.

---

# 96. Connection selection

La selección futura podrá considerar:

```text
query intent
transaction state
read/write role
replica health
consistency requirement
tenant
shard
partition
```

pero Connection System no deberá absorber toda la lógica de Topology.

---

# 97. Topology boundary

```text
Topology
   │
   ▼
select target
   │
   ▼
Connection Resolver
   │
   ▼
Connection
```

Topology decide dónde.

Connection administra acceso al target seleccionado.

---

# 98. Read/write boundary

`ReadWriteRouter` pertenece al dominio de routing/topology.

Connection representa los targets resultantes.

---

# 99. Connection affinity

Además de transactions podrán existir afinidades para:

```text
temporary tables
session locks
streaming cursor
session-specific state
```

---

# 100. AffinityToken

Podrá modelarse conceptualmente:

```text
ConnectionAffinityToken
```

para garantizar que operaciones relacionadas utilicen el mismo recurso.

---

# 101. Transaction affinity is mandatory

Para transactions será obligatoria.

Para otros casos será capability/policy driven.

---

# 102. Streaming affinity

Mientras exista un cursor streaming:

```text
ResultCursor
      │
      ▼
ConnectionLease
```

el recurso puede permanecer ocupado.

---

# 103. Streaming and pool capacity

Una operación streaming larga puede reducir capacidad del pool.

Telemetry deberá poder hacerlo visible.

---

# 104. Connection release timing

No liberar:

```text
before ResultCursor closes
```

si el resultado depende del recurso físico.

---

# 105. Buffered result

Si el resultado está completamente materializado:

```text
execute
→ fetch all native data
→ release connection
→ consume result
```

puede ser posible.

---

# 106. Result ownership contract

El Result System deberá indicar explícitamente si conserva:

```text
connection lease
```

---

# 107. Connection lease reference counting

Se evitará reference counting complejo salvo necesidad real.

Preferir ownership estructurado.

---

# 108. Structured resource ownership

Ejemplo:

```text
ExecutionScope
   │
   └── owns Lease
           │
           └── owns NativeConnection access
```

---

# 109. Resource cleanup stack

`DatabaseContext` podrá mantener:

```text
ResourceCleanupStack
```

para asegurar liberación en orden inverso.

---

# 110. Cleanup order

Ejemplo:

```text
close result
close statement
rollback orphan transaction
reset connection
release lease
```

---

# 111. Exception-safe cleanup

Cleanup deberá ejecutarse incluso ante:

```text
exception
timeout
cancellation
early return
```

---

# 112. Destructor is not lifecycle strategy

No deberá dependerse principalmente de:

```php
__destruct()
```

para liberar conexiones críticas.

---

# 113. Explicit lifecycle

El framework deberá cerrar scopes explícitamente.

---

# 114. Request termination

Al finalizar una petición:

```text
DatabaseLifecycleManager
      │
      ▼
close open results
      │
      ▼
rollback orphan transactions
      │
      ▼
reset leased connections
      │
      ▼
release/discard resources
      │
      ▼
clear DatabaseContext
```

---

# 115. Worker termination

Al terminar el worker:

```text
close pool
close physical resources
flush diagnostics if required
```

---

# 116. CLI lifecycle

CLI commands deberán usar el mismo modelo de:

```text
ExecutionScope
```

aunque no exista HTTP request.

---

# 117. Queue jobs

Cada Job deberá obtener su propio:

```text
DatabaseContext
```

aunque varios jobs se ejecuten dentro del mismo worker.

---

# 118. Scheduled tasks

Mismo principio.

---

# 119. WebSocket/messages

Cada unidad lógica de ejecución deberá tener aislamiento definido.

---

# 120. Connection configuration relationship

La configuración completa será desarrollada en documentos posteriores, pero Connection deberá consumir únicamente objetos tipados.

---

# 121. Example definition

```php
new ConnectionDefinition(
    identity: new ConnectionIdentity('primary'),
    driver: DriverId::from('pdo.pgsql'),
    role: ConnectionRole::PRIMARY,
    database: new DatabaseTarget(...),
    pool: new PoolPolicy(...),
);
```

---

# 122. Credentials

Idealmente:

```text
ConnectionDefinition
      │
      ▼
CredentialReference
      │
      ▼
CredentialResolver
      │
      ▼
Secret
      │
      ▼
Driver connect()
```

El secreto vive el menor tiempo posible.

---

# 123. Credential rotation

La arquitectura deberá permitir rotación sin reconstruir Query/ORM.

Un pool podrá necesitar invalidar conexiones antiguas.

---

# 124. Connection generation

Podrá existir:

```text
ConnectionGeneration
```

o fingerprint de configuración para detectar recursos creados con configuración obsoleta.

---

# 125. Configuration fingerprint

Ejemplo:

```text
ConnectionDefinitionFingerprint
```

podrá incluir aspectos no secretos relevantes.

---

# 126. Pool key

La clave de pool no deberá depender únicamente de:

```text
connection name
```

si distintas configuraciones pueden compartir nombre en contextos distintos.

---

# 127. Pool identity

Conceptualmente:

```text
PoolKey =
Driver
+ Endpoint
+ Database
+ Role
+ Security Context
+ Relevant Session Profile
```

sin exponer secretos.

---

# 128. Connection reuse safety

Dos logical connections sólo podrán compartir recursos si sus requisitos son compatibles.

---

# 129. Tenant isolation and pooling

No deberá asumirse:

```text
same server
→ same reusable connection
```

si la conexión mantiene estado tenant-specific que no puede resetearse con seguridad.

---

# 130. Session profile

Podrá existir:

```text
ConnectionSessionProfile
```

describiendo estado esperado:

```text
timezone
charset
schema/search path
role
application name
SQL mode
```

---

# 131. Session initialization

Al adquirir un recurso:

```text
Physical Connection
      │
      ▼
Apply/Verify Session Profile
      │
      ▼
READY
```

---

# 132. Session profile fingerprint

Puede utilizarse para optimizar reset/reconfiguration.

---

# 133. Connection state tracker

Podrá existir:

```text
ConnectionStateTracker
```

para registrar cambios controlados por VoltStack.

---

# 134. State tracker limitation

No puede conocer automáticamente cualquier SQL raw ejecutado por el usuario.

Por ello raw/native escape hatches pueden:

```text
mark connection dirty
```

---

# 135. Dirty connection

Un recurso `DIRTY` requiere reset más fuerte antes de reuse.

---

# 136. Native handle access

Si el desarrollador solicita acceso nativo:

```php
$connection->unwrap(PDO::class);
```

VoltStack podrá marcar el recurso:

```text
DIRTY
```

por seguridad.

---

# 137. Unsafe native access policy

Podrá configurarse:

```text
allow
allow-and-reset
allow-and-discard
deny
```

según entorno.

---

# 138. Connection decorations

Podrán utilizarse decorators para:

```text
telemetry
debugging
security enforcement
diagnostics
```

sin alterar el Driver.

---

# 139. Decorator boundaries

Un decorator de Connection no deberá convertirse en ORM middleware.

---

# 140. Telemetry

Eventos/métricas posibles:

```text
connection.resolve
connection.acquire
connection.open
connection.reuse
connection.release
connection.reset
connection.discard
connection.failure
connection.wait
```

---

# 141. Sensitive telemetry

Nunca registrar:

```text
password
secret token
complete credential DSN
```

---

# 142. Useful dimensions

Podrán incluirse:

```text
connection identity
driver ID
platform family
role
pool name
reuse/new
duration
outcome
```

evitando cardinalidad excesiva.

---

# 143. Connection diagnostics

Developer tooling podrá mostrar:

```text
Connection: primary
Role: PRIMARY
Driver: pdo.pgsql
State: READY
Physical: acquired
Transaction: none
Pool: default
Platform: PostgreSQL
```

---

# 144. Diagnostics and tenant IDs

Tenant identifiers deberán tratarse según política de privacidad/cardinalidad.

---

# 145. Connection events

Podrán emitirse eventos neutrales:

```text
ConnectionAcquiring
ConnectionAcquired
ConnectionOpened
ConnectionReset
ConnectionReleased
ConnectionDiscarded
ConnectionFailed
```

---

# 146. Event failure policy

Un listener observacional no deberá romper una conexión exitosa salvo política explícita.

---

# 147. Security

Connection System deberá aplicar principios de:

```text
credential minimization
TLS verification
safe reset
role restriction
secret masking
resource isolation
```

---

# 148. TLS configuration

Deberá formar parte de configuración tipada.

No de flags dispersos en Driver-specific application code.

---

# 149. TLS capability

Driver/Platform podrán informar soporte y restricciones.

---

# 150. Production security defaults

En producción podrán aplicarse defaults estrictos:

```text
TLS verification when remote
secret masking
native access restrictions
safe pooling/reset
```

según configuración y entorno.

---

# 151. Timeouts

Se distinguirán:

```text
connection timeout
pool acquisition timeout
query timeout
transaction timeout
```

No deberán representarse todos mediante una única opción `timeout`.

---

# 152. Connection timeout

Controla cuánto esperar al abrir el recurso físico.

---

# 153. Pool acquisition timeout

Controla cuánto esperar por un recurso disponible.

---

# 154. Query timeout

Pertenece principalmente a Execution.

Puede utilizar primitivas del Driver/Connection.

---

# 155. Transaction timeout

Pertenece al Transaction System.

---

# 156. Cancellation

Una cancelación durante adquisición deberá dejar el sistema en estado consistente.

---

# 157. Cancelled acquisition

Si una conexión termina abriéndose después de que el consumidor canceló:

```text
resource
→ safely return to pool or close
```

según implementación.

---

# 158. Pooling relationship

El Connection System debe funcionar:

```text
with pooling
without pooling
```

---

# 159. No-pool mode

```text
Acquire
  │
  ▼
Driver connect
  │
  ▼
Use
  │
  ▼
Close
```

---

# 160. Pool mode

```text
Acquire
  │
  ▼
Pool
 ├── reusable available → lease
 └── none → Driver connect
  │
  ▼
Use
  │
  ▼
Reset
  │
  ▼
Return
```

---

# 161. Pool is optional infrastructure

Query/ORM no deberán conocer si existe.

---

# 162. Pool exhaustion

Cuando no hay recursos disponibles:

```text
wait
timeout
fail
```

según policy.

---

# 163. Backpressure

Connection pooling deberá poder producir backpressure.

No crear conexiones ilimitadas silenciosamente.

---

# 164. Connection limits

Podrán definirse:

```text
max connections
max idle
max lifetime
max idle time
acquisition timeout
```

---

# 165. Server capacity awareness

El framework no puede conocer automáticamente toda la capacidad del servidor.

Los límites deberán configurarse/observarse.

---

# 166. Connection fairness

En alta concurrencia podrá definirse policy de espera.

No es necesario prometer fairness estricta en v1 salvo requerimiento.

---

# 167. Connection reuse

La reutilización debe ser una optimización.

Nunca una violación de aislamiento.

---

# 168. Safety before reuse

Regla:

```text
reuse only if known safe
```

No:

```text
reuse unless known broken
```

cuando el estado sea incierto.

---

# 169. Unknown state

Si el estado es:

```text
UNKNOWN
```

la política segura puede ser:

```text
DISCARD
```

---

# 170. Connection ownership in ORM

EntityManager no deberá poseer directamente un PDO.

Podrá trabajar con:

```text
ConnectionResolver
TransactionManager
QueryExecutor
```

según operación.

---

# 171. IdentityMap independence

IdentityMap no pertenece a Connection.

---

# 172. UnitOfWork independence

UnitOfWork tampoco.

---

# 173. EntityManager and transaction pinning

Cuando ORM ejecuta `flush()` dentro de transaction:

```text
EntityManager
    │
    ▼
Persistence Engine
    │
    ▼
Query Executor
    │
    ▼
Transaction Context
    │
    ▼
Pinned Connection
```

---

# 174. Connection and Query Executor

Executor podrá solicitar una conexión basada en:

```text
ExecutionPlan
ConnectionRequirement
ExecutionContext
```

---

# 175. ConnectionRequirement

Value object conceptual:

```php
final readonly class ConnectionRequirement
{
    public function __construct(
        public ConnectionIntent $intent,
        public ConsistencyRequirement $consistency,
        public ?ConnectionAffinityToken $affinity = null,
    ) {}
}
```

---

# 176. ConnectionIntent

Podrá incluir:

```text
READ
WRITE
SCHEMA
MIGRATION
ADMIN
```

sin necesidad de mezclarlo con nombres físicos.

---

# 177. Requirement resolution

```text
Execution Plan
      │
      ▼
ConnectionRequirement
      │
      ▼
Topology/Connection Resolver
      │
      ▼
Logical Connection
      │
      ▼
Lease
```

---

# 178. Connection policy

Policies podrán decidir:

```text
lazy/eager physical acquisition
pool behavior
reset strength
native access behavior
health validation
```

---

# 179. Policy objects

Preferir objetos tipados como:

```text
ConnectionLifecyclePolicy
ConnectionResetPolicy
ConnectionHealthPolicy
```

a arrays arbitrarios.

---

# 180. No policy in Driver

Driver implementa mecanismos.

Connection policies deciden comportamiento.

---

# 181. Runtime-specific adaptation

FrankenPHP/RoadRunner/OpenSwoole deberán conectarse mediante lifecycle adapters.

No:

```php
if ($runtime === 'frankenphp') {}
```

disperso en Connection.

---

# 182. FrankenPHP default

FrankenPHP será el runtime de referencia para diseñar lifecycle y pooling.

---

# 183. FrankenPHP request model

```text
Worker
   │
   ├── Request A
   │     ├── acquire
   │     ├── use
   │     └── release/reset
   │
   ├── Request B
   │
   └── Request C
```

---

# 184. RoadRunner adapter

Deberá mapear:

```text
worker request lifecycle
```

al mismo `ExecutionScope`.

---

# 185. OpenSwoole adapter

Deberá considerar:

```text
concurrent coroutines
```

y evitar compartir una misma physical connection simultáneamente salvo que el Driver lo soporte explícitamente.

---

# 186. Exclusive lease

Por defecto:

> Una physical connection leased será exclusiva para una unidad de uso incompatible con multiplexing.

---

# 187. Multiplexing

Sólo se permitirá si:

```text
Driver Capability
+
Protocol Capability
+
Connection implementation
```

lo garantizan.

No será supuesto base.

---

# 188. Connection concurrency invariant

Una conexión física no deberá ser utilizada simultáneamente por contexts incompatibles.

---

# 189. Worker-safe vs coroutine-safe

No son equivalentes.

Un recurso puede ser reutilizable entre requests secuenciales y no ser seguro concurrentemente.

---

# 190. Connection capability metadata

Deberá expresar estas diferencias cuando sean relevantes.

---

# 191. Testing architecture

El Connection System necesitará:

```text
unit tests
integration tests
driver-backed tests
pool tests
lifecycle tests
failure tests
persistent runtime tests
concurrency tests
```

---

# 192. Connection contract tests

Todo implementation deberá cumplir una suite común.

---

# 193. Lazy acquisition test

```text
create logical connection
assert no physical connection

execute first operation
assert physical connection acquired
```

---

# 194. Transaction affinity test

```text
begin transaction
query A
query B
commit

assert same physical connection
```

---

# 195. Release test

```text
acquire
use
release

assert resource returned/closed
```

---

# 196. Reset test

```text
acquire
modify session
release
reacquire

assert clean session
```

---

# 197. Orphan transaction test

```text
begin
write
end execution scope unexpectedly

assert rollback before reuse
```

---

# 198. Broken connection test

```text
acquire
simulate network failure
release

assert not returned as healthy
```

---

# 199. Pool isolation test

```text
Request A
  set scoped state

release

Request B
  acquire same physical resource

assert state absent
```

---

# 200. Tenant isolation test

Cuando Multitenancy esté instalado:

```text
Tenant A
→ acquire/use/release

Tenant B
→ same resource if allowed

assert no Tenant A state
```

---

# 201. Concurrent lease test

Para runtimes concurrentes:

```text
Context A → Lease A
Context B → Lease B

assert no unsafe simultaneous sharing
```

---

# 202. Read-only role test

```text
Replica Connection
      │
      ▼
attempt write
      │
      ▼
reject
```

cuando la restricción pueda aplicarse.

---

# 203. Architecture tests

Deberán impedir:

```text
Connection → ORM
Connection → Entity
Connection → Repository
Connection → QueryBuilder
Connection → Controller
```

---

# 204. Allowed dependencies

Connection podrá depender de:

```text
Driver contracts
Platform contracts
Dialect descriptors
Configuration value objects
Capability model
Lifecycle contracts
Error model
low-level Support
```

---

# 205. Dependency direction

```text
Execution
   │
   ▼
Connection
   │
   ▼
Driver
```

No:

```text
Driver
   │
   ▼
Connection Manager
```

salvo contracts estrictamente inferiores requeridos por composición.

---

# 206. Suggested namespace structure

```text
VoltStack\Quantum\Database\Connection
│
├── Contract
│   ├── ConnectionInterface.php
│   ├── ConnectionFactoryInterface.php
│   ├── ConnectionResolverInterface.php
│   ├── ConnectionResourceProviderInterface.php
│   └── ConnectionResetterInterface.php
│
├── Configuration
│   ├── ConnectionDefinition.php
│   ├── ConnectionIdentity.php
│   └── ConnectionSessionProfile.php
│
├── Manager
│   └── ConnectionManager.php
│
├── Factory
│   └── ConnectionFactory.php
│
├── Resolver
│   └── ConnectionResolver.php
│
├── Resource
│   ├── ConnectionHandle.php
│   └── ConnectionLease.php
│
├── State
│   ├── ConnectionState.php
│   └── ConnectionStateTracker.php
│
├── Role
│   └── ConnectionRole.php
│
├── Lifecycle
├── Reset
├── Health
├── Failure
├── Diagnostics
└── Exception
```

---

# 207. Manager API example

Conceptualmente:

```php
$connection = $manager->connection('primary');
```

Esto obtiene:

```text
Logical Connection
```

No necesariamente abre inmediatamente el recurso.

---

# 208. Application API

Para uso normal:

```php
DB::connection('analytics')
    ->table('events')
    ->where(...)
    ->get();
```

pero internamente:

```text
Facade
  │
  ▼
Query API
  │
  ▼
Execution
  │
  ▼
Connection Manager
```

La Connection no implementa Query Builder internamente.

---

# 209. Facade separation

Aunque la sintaxis pública parezca:

```php
DB::connection()->table()
```

puede utilizar un wrapper/context API.

No significa:

```text
Connection owns QueryBuilder
```

---

# 210. Fluent DX vs internal architecture

Regla:

> La ergonomía de la API pública no deberá dictar dependencias internas incorrectas.

---

# 211. Raw SQL API

Podrá ofrecerse:

```php
DB::connection('primary')
    ->statement(...);
```

mediante una API de ejecución asociada al contexto de conexión.

Internamente deberá seguir:

```text
Execution Engine
→ Connection
→ Driver
```

---

# 212. Native connection escape hatch

El acceso nativo será avanzado.

Ejemplo conceptual:

```php
DB::connection()
    ->native(function (PDO $pdo) {
        // advanced native operation
    });
```

---

# 213. Scoped native access

Preferir callback/scoped access sobre devolver indefinidamente el handle nativo.

---

# 214. Native access lifecycle

```text
acquire
  │
  ▼
callback
  │
  ▼
mark dirty if necessary
  │
  ▼
reset/discard
  │
  ▼
release
```

---

# 215. Native handle must not escape

El callback deberá documentar que el handle no debe conservarse después.

---

# 216. Connection invariants

## DB-CONN-001

Connection no es Driver.

## DB-CONN-002

Connection no es Dialect.

## DB-CONN-003

Connection no es Platform.

## DB-CONN-004

Logical Connection no equivale a Physical Connection.

## DB-CONN-005

Una Logical Connection podrá existir sin abrir recurso físico.

## DB-CONN-006

La adquisición física será lazy por defecto.

## DB-CONN-007

Connection no conocerá ORM.

## DB-CONN-008

Connection no conocerá Query Builder.

## DB-CONN-009

Connection no generará SQL.

## DB-CONN-010

Una transaction deberá permanecer fijada a una physical connection compatible.

## DB-CONN-011

Una conexión con transaction activa no volverá al pool.

## DB-CONN-012

Un recurso de estado incierto deberá descartarse.

## DB-CONN-013

Toda reutilización requerirá estado seguro.

## DB-CONN-014

Request-scoped state no podrá sobrevivir entre execution scopes.

## DB-CONN-015

Una replica/read-only connection no deberá adquirir capabilities de escritura.

## DB-CONN-016

Connection Manager no será Pool.

## DB-CONN-017

Connection Manager no será Driver Registry.

## DB-CONN-018

Driver ejecutará primitivas; Connection coordinará lifecycle.

## DB-CONN-019

Los secretos no deberán formar parte de identities ni diagnostics públicos.

## DB-CONN-020

Native handles permanecerán encapsulados salvo escape hatch explícito.

## DB-CONN-021

Native access podrá marcar el recurso dirty.

## DB-CONN-022

Los cursores streaming conservarán el lease mientras lo requieran.

## DB-CONN-023

Connection cleanup será explícito y exception-safe.

## DB-CONN-024

No se dependerá de destructores como mecanismo principal de cleanup.

## DB-CONN-025

Una physical connection no será compartida concurrentemente sin capability explícita.

## DB-CONN-026

Pooling será opcional para consumidores superiores.

## DB-CONN-027

El sistema deberá funcionar correctamente sin pooling.

## DB-CONN-028

Connection state deberá modelarse explícitamente.

## DB-CONN-029

Retry de operaciones no será responsabilidad automática de Connection.

## DB-CONN-030

Cambiar de Driver no deberá alterar el contrato lógico de Connection mientras se mantengan capabilities requeridas.

---

# 217. Anti-pattern — PDO como Connection pública

```php
function connection(): PDO
```

como API arquitectónica principal.

**Prohibido.**

---

# 218. Anti-pattern — conexión global mutable

```php
static $currentConnection;
```

compartida entre requests.

**Prohibido.**

---

# 219. Anti-pattern — current tenant en singleton

```php
final class ConnectionManager
{
    private ?Tenant $currentTenant;
}
```

si `ConnectionManager` es persistente.

**Prohibido.**

---

# 220. Anti-pattern — transaction state in ConnectionManager singleton

```php
private bool $inTransaction;
```

global al worker.

**Prohibido.**

---

# 221. Anti-pattern — eager connect all

```text
bootstrap
→ connect primary
→ connect replica1
→ connect replica2
→ connect analytics
→ connect legacy
```

sin necesidad.

**Prohibido como default.**

---

# 222. Anti-pattern — unsafe pooling

```text
Request A
→ transaction left open
→ return connection

Request B
→ receives same connection
```

**Crítico y prohibido.**

---

# 223. Anti-pattern — role inferred by name

```php
str_contains($connectionName, 'replica')
```

**Prohibido.**

---

# 224. Anti-pattern — query retry inside Connection

```php
try {
    execute();
} catch (ConnectionLost $e) {
    execute();
}
```

sin conocer idempotencia.

**Prohibido.**

---

# 225. Anti-pattern — ORM state in Connection

```text
Connection
├── IdentityMap
├── UnitOfWork
└── entities
```

**Prohibido.**

---

# 226. Anti-pattern — vendor routing

```php
if ($connection->vendor() === 'pgsql') {
    // application behavior
}
```

en capas superiores.

Debe utilizarse Capability System.

---

# 227. Reference architecture

```text
                   Query / ORM / Schema
                           │
                           ▼
                       Execution
                           │
                           ▼
                 Connection Requirement
                           │
                           ▼
                 Topology / Resolution
                           │
                           ▼
                 Logical Connection
                           │
                           ▼
                 Connection Manager
                           │
                           ▼
                  Resource Provider
                    │            │
                    ▼            ▼
                  Pool         Driver
                    │            │
                    └─────┬──────┘
                          ▼
                  Connection Lease
                          │
                          ▼
                  Native Connection
                          │
                          ▼
                     Database
```

---

# 228. Lifecycle architecture

```text
Execution Scope
      │
      ▼
DatabaseContext
      │
      ▼
Acquire Connection Lease
      │
      ▼
Use Resource
      │
      ▼
Finish Operation
      │
      ▼
Close Results/Statements
      │
      ▼
Rollback Orphan Transaction
      │
      ▼
Reset Connection
      │
      ├── safe
      │     ▼
      │   release/pool
      │
      └── unsafe
            ▼
          discard
```

---

# 229. Persistent runtime architecture

```text
                   Worker
                     │
       ┌─────────────┼─────────────┐
       │             │             │
       ▼             ▼             ▼
Driver Registry   Pool      Connection Definitions
                     │
                     ▼
              Physical Resources
                     │
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
   Request A      Request B      Request C
       │             │             │
       ▼             ▼             ▼
 DatabaseCtx A  DatabaseCtx B  DatabaseCtx C
       │             │             │
       ▼             ▼             ▼
    Leases A       Leases B      Leases C
```

Los recursos físicos pueden reutilizarse.

El estado lógico de cada request no.

---

# 230. Design objective

El Connection System será correcto si puede responder de forma inequívoca:

```text
Which logical database access is requested?
Which physical resource owns the operation?
Who may use that resource?
For how long?
Is it transaction-pinned?
Is it safe to reuse?
How must it be reset?
What happens if it fails?
```

---

# 231. Ecuación arquitectónica

```text
Connection System
=
Logical Connection Model
+
Resource Acquisition
+
Explicit Ownership
+
Lifecycle
+
Reset
+
Isolation
+
Transaction Affinity
```

No:

```text
Connection System
=
PDO Wrapper
```

---

# 232. Principio final

> Una Connection representa acceso lógico; un Lease representa ownership temporal; el Driver proporciona el mecanismo; la NativeConnection representa el recurso físico.

Formalmente:

```text
Logical Connection
      │
      ▼
Lease
      │
      ▼
Driver-managed Native Resource
```

Esta separación será fundamental para:

```text
pooling
transactions
replicas
multitenancy
persistent workers
concurrency
failover
streaming
```

---

# 233. Conclusión

VoltStack Database utilizará un Connection System explícitamente orientado al lifecycle.

La arquitectura no tratará una conexión simplemente como:

```text
new PDO(...)
```

sino como un conjunto coordinado de conceptos:

```text
ConnectionDefinition
        │
        ▼
Logical Connection
        │
        ▼
ConnectionRequirement
        │
        ▼
Connection Resolution
        │
        ▼
Connection Lease
        │
        ▼
Native Connection
        │
        ▼
Reset / Release / Discard
```

Esto permitirá que una misma arquitectura funcione desde aplicaciones pequeñas con SQLite hasta aplicaciones persistentes y distribuidas con:

```text
FrankenPHP
PostgreSQL
connection pools
read replicas
multitenancy
sharding
```

sin contaminar Query Engine u ORM con lifecycle de infraestructura.

---

# 234. Siguiente documento

El siguiente documento será:

```text
12_DATABASE_CONNECTION_MANAGER.md
```

y deberá profundizar específicamente en:

```text
ConnectionManager responsibilities
ConnectionDefinitionRegistry
named connection resolution
default connection resolution
ConnectionFactory coordination
ConnectionResolver coordination
resource provider coordination

connection acquisition orchestration
connection lease ownership
scoped connection access
transaction-pinned resolution
connection aliases
dynamic connection definitions
tenant-aware integration boundaries

manager lifetime
manager state
persistent worker safety
manager API
manager extension points
manager diagnostics
manager testing
```

manteniendo una regla esencial:

```text
ConnectionManager
≠
Connection Pool
≠
Driver Registry
≠
Query Executor
≠
Transaction Manager
```

La relación deberá permanecer:

```text
Execution / Infrastructure Consumer
              │
              ▼
       ConnectionManager
              │
       ┌──────┼───────┐
       ▼      ▼       ▼
 Definitions Factory Resolver
              │
              ▼
      Logical Connection
              │
              ▼
     Resource Acquisition
              │
              ▼
        Driver / Pool
```