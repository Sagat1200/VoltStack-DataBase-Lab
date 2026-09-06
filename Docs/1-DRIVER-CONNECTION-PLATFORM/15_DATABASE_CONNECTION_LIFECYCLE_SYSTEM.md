# 15_DATABASE_CONNECTION_LIFECYCLE_SYSTEM.md

# VoltStack Quantum Database
## Connection Lifecycle System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 15 — Database Connection Lifecycle System  
**Estado:** Architecture Specification  
**Nivel:** Infrastructure Architecture  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura oficial del ciclo de vida de las conexiones de:

```text
VoltStack/Quantum/Database
```

El `Connection Lifecycle System` establece cómo nacen, se resuelven, adquieren, inicializan, utilizan, validan, resetean, liberan, retiran y destruyen los recursos relacionados con conexiones de base de datos.

Su objetivo principal es garantizar que:

```text
Connection Lifecycle
=
Deterministic
+
Explicit
+
Scoped
+
Recoverable
+
Observable
+
Persistent-Runtime Safe
```

Especialmente bajo:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 2. Problema arquitectónico

En un runtime PHP tradicional puede parecer suficiente:

```text
Request
   │
   ▼
Open Connection
   │
   ▼
Execute Queries
   │
   ▼
Request Ends
```

porque gran parte del estado desaparece al terminar la petición.

VoltStack no puede depender de ese comportamiento.

Con workers persistentes:

```text
Worker
 │
 ├── Request A
 ├── Request B
 ├── Request C
 ├── Job D
 └── ...
```

una conexión física puede sobrevivir a múltiples operaciones.

Por tanto:

> La terminación de una petición no equivale a la terminación de una conexión física.

---

# 3. Principio fundamental

La arquitectura distinguirá explícitamente:

```text
Connection Definition
        │
        ▼
Logical Connection
        │
        ▼
Connection Lease
        │
        ▼
Physical Connection
        │
        ▼
Native Connection
```

Cada nivel tiene un lifecycle diferente.

---

# 4. Los cinco lifecycles

VoltStack reconocerá al menos:

```text
Configuration Lifecycle
Logical Connection Lifecycle
Lease Lifecycle
Physical Connection Lifecycle
Native Resource Lifecycle
```

No deberán confundirse.

---

# 5. Configuration Lifecycle

Representa cuánto tiempo permanece válida una:

```text
ConnectionDefinition
```

o una generación compilada de configuración.

Ejemplo:

```text
Generation 42
     │
     ▼
ConnectionDefinition
```

Puede vivir durante todo un worker.

---

# 6. Logical Connection Lifecycle

Una `Connection` lógica representa una vista controlada sobre una definición/resolución de conexión.

No implica necesariamente que exista un socket abierto.

```text
Logical Connection
       │
       ├── no physical resource
       │
       └── acquire when required
```

---

# 7. Lease Lifecycle

El `ConnectionLease` representa ownership temporal de un recurso físico.

```text
Acquire
   │
   ▼
Lease Active
   │
   ▼
Use
   │
   ▼
Release / Discard
```

---

# 8. Physical Connection Lifecycle

Representa la vida del recurso físico reutilizable:

```text
Create
  │
  ▼
Initialize
  │
  ▼
Idle
  │
  ▼
Lease
  │
  ▼
Use
  │
  ▼
Reset
  │
  ├── Idle
  └── Retire
```

---

# 9. Native Resource Lifecycle

El Driver puede encapsular:

```text
PDO
PgSql\Connection
mysqli
native socket/client
future native driver
```

Su lifecycle queda contenido dentro de:

```text
PhysicalConnection
```

y no deberá propagarse hacia capas superiores.

---

# 10. Regla principal

```text
Logical Connection Lifetime
≠
Physical Connection Lifetime
≠
Lease Lifetime
```

---

# 11. Arquitectura general

```text
Application / Execution Scope
            │
            ▼
     ConnectionManager
            │
            ▼
     Logical Connection
            │
            ▼
  ConnectionResourceProvider
            │
            ▼
       Acquire Lease
            │
            ▼
    Physical Connection
            │
            ▼
          Driver
            │
            ▼
      Native Resource
```

Al finalizar:

```text
Operation Complete
        │
        ▼
Release Lease
        │
        ▼
Reset Resource
        │
        ├── reusable → Pool
        └── unsafe   → Close
```

---

# 12. Lifecycle ownership

Cada recurso deberá tener un owner claro.

| Recurso | Owner |
|---|---|
| `ConnectionDefinition` | Configuration System |
| `LogicalConnection` | Database/Execution Context |
| `ConnectionLease` | Operation / Transaction / Cursor |
| `PhysicalConnection` idle | Pool |
| `PhysicalConnection` leased | Lease |
| Native resource | Physical Connection |
| Transaction state | Transaction Context |
| Streaming cursor | Result/Cursor Context |

---

# 13. No ambiguous ownership

No deberá existir un recurso físico cuyo owner sea conceptualmente:

```text
"someone"
```

o:

```text
"global database service"
```

El ownership deberá ser determinista.

---

# 14. Lifecycle hierarchy

```text
Application
│
├── Worker
│   │
│   ├── ConnectionPoolManager
│   │
│   ├── ConnectionPool
│   │
│   ├── Physical Connections
│   │
│   └── Execution Scope
│       │
│       ├── DatabaseContext
│       ├── Logical Connections
│       ├── Connection Leases
│       ├── Transactions
│       └── Results / Cursors
```

---

# 15. Application lifecycle

Durante application bootstrap se podrán crear:

```text
compiled configuration
driver registry
platform registry
dialect registry
connection factories
connection resolvers
pool manager
lifecycle coordinators
```

Pero no deberán abrirse conexiones físicas automáticamente.

---

# 16. Bootstrap invariant

```text
Application Boot
≠
Database Connect
```

---

# 17. Lazy physical connections

El comportamiento predeterminado será:

```text
Boot
→ no database socket

First DB operation
→ acquire/create physical resource
```

---

# 18. Worker lifecycle

En runtime persistente:

```text
Worker Start
    │
    ▼
Database Worker Services
    │
    ├── PoolManager
    ├── Pools
    └── persistent-safe registries
```

podrán sobrevivir múltiples execution scopes.

---

# 19. Worker-owned state

Podrán sobrevivir:

```text
immutable configuration
compiled metadata
driver descriptors
dialect descriptors
platform descriptors
capability snapshots
connection pools
idle physical resources
```

---

# 20. Request-owned state

No deberá sobrevivir:

```text
EntityManager state
UnitOfWork
IdentityMap
active transaction
tenant context
query context
hydration context
leased connection ownership
open cursor
request-specific session state
```

---

# 21. Execution Scope

VoltStack utilizará una abstracción neutral:

```text
ExecutionScope
```

que podrá representar:

```text
HTTP request
queue job
CLI command
scheduled task
WebSocket operation
worker operation
```

---

# 22. Execution scope lifecycle

```text
ExecutionScope::begin()
        │
        ▼
DatabaseContext created
        │
        ▼
Database operations
        │
        ▼
DatabaseLifecycle::end()
        │
        ▼
DatabaseContext disposed
```

---

# 23. DatabaseContext

Cada execution scope tendrá su propio:

```text
DatabaseContext
```

que podrá contener referencias a:

```text
logical connections
lease tracker
transaction contexts
tenant context bridge
execution metadata
scope diagnostics
```

---

# 24. DatabaseContext isolation

```text
Request A
→ DatabaseContext A

Request B
→ DatabaseContext B
```

y:

```text
DatabaseContext A
≠
DatabaseContext B
```

aunque ambas peticiones utilicen el mismo worker.

---

# 25. Logical Connection states

Una conexión lógica podrá tener estados conceptuales:

```text
UNRESOLVED
RESOLVED
READY
IN_USE
DISPOSED
```

No necesariamente requiere una máquina de estados rígida pública.

---

# 26. UNRESOLVED

Representa una solicitud conceptual:

```text
"default"
```

que todavía no ha sido transformada en una definición concreta.

---

# 27. RESOLVED

Después de:

```text
ConnectionManager
      │
      ▼
ConnectionResolver
```

se obtiene una definición efectiva.

---

# 28. READY

La conexión lógica está preparada para solicitar recursos físicos.

Esto no implica conexión física abierta.

---

# 29. IN_USE

Existe una operación que utiliza un lease asociado.

---

# 30. DISPOSED

El execution scope terminó y la conexión lógica no deberá volver a utilizarse.

---

# 31. Logical connection disposal

Después de:

```text
ExecutionScope end
```

una referencia accidental a una conexión lógica scoped deberá fallar claramente.

---

# 32. Scope invalidation

Podrá utilizarse:

```text
ScopeToken
ScopeId
ScopeGeneration
```

para detectar referencias stale.

---

# 33. Stale logical connection

Ejemplo:

```text
Request A:
$conn = DB::connection()

Request A ends

Request B:
reuse $conn from A
```

deberá considerarse arquitectónicamente inválido.

---

# 34. No scoped connection caching

Prohibido:

```php
static $connection;
```

en Facades, Models o servicios singleton.

---

# 35. Physical Connection state machine

Estado conceptual principal:

```text
NEW
 │
 ▼
CONNECTING
 │
 ▼
INITIALIZING
 │
 ▼
READY
 │
 ▼
IDLE
 │
 ▼
LEASED
 │
 ▼
RESETTING
 │
 ├──► IDLE
 │
 └──► RETIRING
          │
          ▼
        CLOSED
```

Estados de fallo:

```text
BROKEN
TAINTED
```

podrán provocar:

```text
RETIRING
→ CLOSED
```

---

# 36. NEW

Objeto físico creado, pero sin recurso nativo operativo.

---

# 37. CONNECTING

El Driver está estableciendo comunicación con el servidor.

---

# 38. INITIALIZING

La conexión nativa existe, pero aún debe configurarse.

---

# 39. Initialization responsibilities

Puede incluir:

```text
TLS verification
authentication verification
server version discovery
platform capability discovery
encoding setup
timezone setup
session baseline
application name
connection attributes
```

---

# 40. READY

La conexión ha sido inicializada correctamente.

---

# 41. IDLE

El recurso pertenece al Pool y puede ser adquirido.

---

# 42. LEASED

El recurso está bajo ownership de un `ConnectionLease`.

---

# 43. RESETTING

Se está restaurando el baseline de sesión.

---

# 44. RETIRING

La conexión dejará de formar parte del conjunto reutilizable.

---

# 45. CLOSED

El recurso nativo ha sido cerrado o invalidado definitivamente.

---

# 46. BROKEN

El framework conoce una falla que impide reutilización normal.

Ejemplos:

```text
socket closed
server gone away
protocol error
native resource invalid
```

---

# 47. TAINTED

VoltStack no puede demostrar que el estado sea seguro.

Ejemplos:

```text
unknown transaction state
abandoned cursor
failed session reset
uncertain protocol state
```

---

# 48. BROKEN vs TAINTED

```text
BROKEN
=
known unusable

TAINTED
=
safety cannot be guaranteed
```

Ambos normalmente conducen a retiro.

---

# 49. State transition invariant

Las transiciones deberán realizarse mediante operaciones controladas.

No:

```php
$connection->state = 'idle';
```

desde componentes arbitrarios.

---

# 50. ConnectionStateMachine

Podrá existir internamente:

```text
PhysicalConnectionStateMachine
```

para validar transiciones.

---

# 51. Invalid transition

Ejemplo:

```text
CLOSED
→ LEASED
```

deberá fallar inmediatamente.

---

# 52. Creation lifecycle

```text
Connection requested
      │
      ▼
Resource provider
      │
      ▼
Pool has no idle resource
      │
      ▼
Reserve capacity
      │
      ▼
PhysicalConnectionFactory
      │
      ▼
Driver::connect()
      │
      ▼
Initialize
      │
      ▼
Validate
      │
      ▼
Lease
```

---

# 53. Capacity reservation before connect

La capacidad deberá reservarse antes de abrir la conexión para evitar carreras concurrentes.

---

# 54. Creation failure

```text
CONNECTING
→ failure
→ BROKEN/CLOSED
```

y la reserva de capacidad deberá liberarse.

---

# 55. Initialization failure

```text
INITIALIZING
→ failure
→ close native resource
→ release capacity
```

---

# 56. Partial initialization

Una conexión parcialmente inicializada nunca deberá regresar al Pool.

---

# 57. Connection initializer

Contrato conceptual:

```php
interface PhysicalConnectionInitializerInterface
{
    public function initialize(
        PhysicalConnectionInterface $connection,
        ConnectionInitializationContext $context
    ): void;
}
```

---

# 58. Initializer responsibility

El initializer coordina el baseline inicial.

No deberá contener lógica ORM o Query Builder.

---

# 59. Initialization idempotency

No deberá asumirse que toda inicialización puede repetirse arbitrariamente.

El lifecycle deberá conocer si la conexión está:

```text
NEW
INITIALIZED
```

---

# 60. Session baseline

Toda conexión reutilizable deberá tener un baseline conocido.

Ejemplo conceptual:

```text
autocommit       = expected
transaction      = none
role             = default
schema           = configured
timezone         = configured
encoding         = configured
open cursors     = none
```

---

# 61. SessionProfile

Podrá modelarse:

```php
final readonly class ConnectionSessionProfile
{
    // expected baseline
}
```

---

# 62. Initialization vs Reset

```text
Initialization
=
NEW → known baseline

Reset
=
USED → known baseline
```

---

# 63. Acquisition lifecycle

```text
Logical Connection
       │
       ▼
Resource Provider
       │
       ▼
Acquire Request
       │
       ▼
Pool / Direct Provider
       │
       ▼
Physical Resource
       │
       ▼
Lease
```

---

# 64. ConnectionAcquisitionContext

Podrá contener:

```text
execution scope
connection identity
role
deadline
cancellation token
transaction affinity
diagnostic metadata
```

sin incluir application-domain semantics.

---

# 65. Acquisition deadline

Deberá propagarse un deadline claro.

No deberán crearse múltiples timeouts independientes que puedan exceder el límite total de la operación.

---

# 66. Acquire timeout

```text
start acquisition
       │
       ▼
wait/create/validate
       │
       ▼
deadline reached
       │
       ▼
ConnectionAcquisitionTimeoutException
```

---

# 67. Cancellation

Una operación cancelada deberá detener:

```text
wait queue participation
connection creation when safely possible
future resource assignment
```

---

# 68. Lease creation

Una vez elegido el recurso:

```text
IDLE
→ LEASED
```

y se crea:

```text
ConnectionLease
```

---

# 69. Lease state machine

```text
CREATED
   │
   ▼
ACTIVE
   │
   ├──► RELEASED
   ├──► DISCARDED
   └──► INVALIDATED
```

---

# 70. ACTIVE

El lease puede utilizar su recurso físico.

---

# 71. RELEASED

El consumidor terminó correctamente y devolvió ownership.

---

# 72. DISCARDED

El consumidor determinó que el recurso no debe reutilizarse.

---

# 73. INVALIDATED

El lifecycle/pool cerró o invalidó el recurso externamente.

---

# 74. Lease terminal states

```text
RELEASED
DISCARDED
INVALIDATED
```

serán terminales.

---

# 75. Use-after-release

```php
$lease->release();

$lease->connection();
```

deberá fallar o impedir uso operativo.

---

# 76. Double release

La implementación deberá evitar devolver dos veces el mismo recurso.

---

# 77. Lease ownership types

El owner conceptual puede ser:

```text
single statement
execution operation
transaction
streaming result
cursor
explicit connection scope
```

---

# 78. Short-lived statement lease

Fuera de transaction:

```text
Query
→ acquire
→ execute
→ materialize result
→ release
```

cuando el resultado ya no dependa de la conexión.

---

# 79. Transaction lease

```text
BEGIN
→ acquire
→ pin lease
→ Query A
→ Query B
→ Query C
→ COMMIT
→ release
```

---

# 80. Streaming lease

```text
Query
→ acquire
→ execute streaming
→ cursor owns lease
→ iterate
→ close cursor
→ release
```

---

# 81. Explicit connection scope

Una API avanzada podría permitir:

```php
DB::withConnection(function ($connection) {
    // multiple operations using controlled affinity
});
```

sin convertir la conexión en global.

---

# 82. Lease pinning

Pinning significa:

> El lease no puede devolverse al proveedor hasta que finalice la operación que requiere afinidad física.

---

# 83. Pinning reasons

```text
transaction
streaming cursor
session-dependent operation
temporary-table workflow
advisory lock
platform-specific sequence
```

---

# 84. Pin count

Internamente podría existir ownership compuesto/ref-counting.

Pero deberá evitarse complejidad innecesaria.

Preferencia:

```text
single explicit owner
```

siempre que sea posible.

---

# 85. Transaction lifecycle integration

El `TransactionManager` deberá coordinar con el lifecycle.

```text
TransactionManager
        │
        ▼
Acquire/Pinned Lease
        │
        ▼
Physical Connection
```

---

# 86. Transaction start

```text
NO_TRANSACTION
      │
      ▼
acquire lease
      │
      ▼
driver begin
      │
      ▼
ACTIVE_TRANSACTION
```

---

# 87. Commit lifecycle

```text
ACTIVE_TRANSACTION
       │
       ▼
COMMIT
       │
       ▼
transaction state clear
       │
       ▼
lease eligible for release
```

---

# 88. Rollback lifecycle

```text
ACTIVE_TRANSACTION
       │
       ▼
ROLLBACK
       │
       ▼
transaction state clear
       │
       ▼
lease eligible for reset/release
```

---

# 89. Failed commit

Si el resultado de commit es incierto:

```text
transaction outcome unknown
```

la conexión deberá marcarse:

```text
TAINTED
```

o:

```text
BROKEN
```

según clasificación.

---

# 90. No retry ambiguity

El Pool no decidirá si repetir una transaction.

Eso pertenece a:

```text
TransactionManager
Resilience System
```

---

# 91. Orphan transaction

Al terminar un execution scope:

```text
active transaction detected
```

deberá ejecutarse:

```text
rollback
```

si es seguro.

---

# 92. Orphan rollback failure

```text
rollback fails
→ connection tainted/broken
→ discard
```

---

# 93. Savepoint lifecycle

Savepoints permanecen dentro del mismo:

```text
TransactionContext
ConnectionLease
PhysicalConnection
```

---

# 94. Cursor lifecycle

Un cursor podrá tener estados:

```text
OPEN
EXHAUSTED
CLOSED
FAILED
```

---

# 95. Cursor ownership

Mientras un cursor requiera conexión:

```text
Cursor
→ owns/pins lease
```

---

# 96. Exhausted cursor

Si el Driver garantiza que el resultado ya no requiere conexión:

```text
EXHAUSTED
→ release eligible
```

---

# 97. Cursor close

Cerrar explícitamente deberá:

```text
close native cursor
release statement resources
update lease ownership
```

---

# 98. Abandoned cursor

Al terminar scope:

```text
open cursor
→ forced close
→ connection state assessment
→ reset/release/discard
```

---

# 99. Cursor close failure

Puede marcar la conexión como:

```text
TAINTED
```

si el protocolo queda incierto.

---

# 100. Result lifecycle

Resultados materializados:

```text
array
collection
scalar
DTO
hydrated entities
```

no deberán retener leases innecesariamente.

---

# 101. Materialize then release

```text
execute
→ fetch all
→ close native result
→ release connection
→ hydrate/process
```

cuando sea semánticamente posible.

---

# 102. Performance benefit

Esto reduce:

```text
lease duration
pool contention
connection starvation
```

---

# 103. Hydration separation

ORM Hydration no deberá mantener la conexión ocupada sólo por conveniencia si los datos ya fueron materializados.

---

# 104. Statement lifecycle

```text
prepare
→ bind
→ execute
→ result
→ close/reset statement
```

deberá finalizar antes de que la conexión vuelva a idle, salvo caching explícitamente seguro.

---

# 105. Statement leak

Un statement abierto al release deberá ser:

```text
closed
reset
or cause discard
```

según Driver.

---

# 106. Session mutation lifecycle

Operaciones como:

```text
SET ROLE
SET search_path
SET timezone
SET sql_mode
USE database
```

alteran estado físico.

---

# 107. SessionStateTracker

Podrá existir:

```text
ConnectionSessionStateTracker
```

para registrar mutaciones conocidas.

---

# 108. Dirty session

Después de una mutación:

```text
SessionState
→ DIRTY
```

hasta reset.

---

# 109. Clean session

Después de reset verificado:

```text
SessionState
→ BASELINE
```

---

# 110. Unknown session

Si VoltStack no conoce todas las modificaciones:

```text
SessionState
→ UNKNOWN
```

---

# 111. Unknown state policy

Por defecto:

```text
UNKNOWN
→ strict reset
```

y si no puede demostrarse recuperación:

```text
discard
```

---

# 112. Native escape hatch impact

Si el usuario obtiene acceso nativo:

```php
$connection->native(/* ... */);
```

VoltStack puede perder visibilidad sobre mutaciones de sesión.

---

# 113. Native access taint policy

Una estrategia conservadora podrá marcar:

```text
native session access
→ potentially dirty
```

---

# 114. Explicit native access contract

Podrá requerir:

```text
read-only native access
```

o:

```text
unsafe mutable native access
```

como APIs diferentes.

---

# 115. Connection validation lifecycle

Validation podrá ocurrir:

```text
after creation
before acquisition
after long idle
after suspicious failure
during maintenance
```

---

# 116. Validation is contextual

No deberá ejecutarse indiscriminadamente en todos los transitions.

---

# 117. Health states

Podrán existir:

```text
UNKNOWN
HEALTHY
SUSPECT
UNHEALTHY
```

---

# 118. HEALTHY

La evidencia disponible indica que el recurso puede utilizarse.

---

# 119. SUSPECT

Existe evidencia de posible falla pero no confirmación.

---

# 120. UNHEALTHY

No deberá entregarse a consumidores.

---

# 121. Validation flow

```text
IDLE
 │
 ▼
validation required?
 │
 ├── no ─────► LEASED
 │
 └── yes
      │
      ▼
   VALIDATE
      │
      ├── healthy ─► LEASED
      └── failed ──► RETIRE
```

---

# 122. Passive failure detection

Errores durante query execution también podrán actualizar health state.

---

# 123. Failure classification

El Driver deberá ayudar a distinguir:

```text
query semantic error
constraint error
transaction error
connection error
network error
protocol error
authentication error
server shutdown
```

---

# 124. Connection-fatal failure

Un error clasificado como fatal para la conexión deberá producir:

```text
LEASED
→ BROKEN
→ DISCARD
```

---

# 125. Non-fatal SQL failure

Ejemplo:

```text
unique constraint violation
```

no deberá cerrar automáticamente la conexión.

---

# 126. Transaction-aborting failure

PostgreSQL, por ejemplo, puede dejar una transaction en estado abortado.

El lifecycle deberá delegar recuperación al Transaction/Platform layer.

---

# 127. Reset lifecycle

Cuando un lease se libera:

```text
LEASED
   │
   ▼
Release Requested
   │
   ▼
Outstanding Resource Cleanup
   │
   ▼
Transaction Check
   │
   ▼
Session Reset
   │
   ▼
Validation / Retirement Check
   │
   ├── reusable → IDLE
   └── unsafe   → CLOSED
```

---

# 128. Reset ordering

El orden importa.

Por ejemplo:

```text
close cursor
before
session reset
```

puede ser obligatorio.

---

# 129. Reset pipeline

Pipeline conceptual:

```text
1. stop new use
2. close result/cursor
3. close/reset statements
4. resolve transaction state
5. restore session state
6. validate native resource if required
7. apply retirement policy
8. return to pool or close
```

---

# 130. Reset context

Podrá contener:

```text
release reason
execution scope
dirty flags
transaction outcome
cursor state
native access state
configuration generation
credential generation
```

---

# 131. Reset strategy composition

Podrá componerse:

```text
CursorResetter
TransactionResetter
SessionResetter
DriverResetter
PlatformResetter
```

sin convertirlo en una cadena arbitraria de plugins.

---

# 132. Reset failure semantics

Si cualquier paso crítico falla:

```text
resource not reusable
```

---

# 133. Best-effort cleanup

Aun si un paso falla, se deberá intentar:

```text
close native resource
release accounting
remove ownership
```

---

# 134. Cleanup failure must not leak capacity

Incluso si:

```text
native close fails
```

el Pool deberá actualizar correctamente su estado interno.

---

# 135. Release lifecycle

```text
Lease ACTIVE
     │
     ▼
release()
     │
     ▼
Lease ownership transferred
     │
     ▼
Pool reset pipeline
     │
     ▼
Lease RELEASED
```

---

# 136. Discard lifecycle

```text
Lease ACTIVE
     │
     ▼
discard()
     │
     ▼
PhysicalConnection RETIRING
     │
     ▼
native close
     │
     ▼
Lease DISCARDED
```

---

# 137. Release must be atomic conceptually

No deberá ocurrir:

```text
resource visible as IDLE
```

antes de completar reset.

---

# 138. Race prevention

Otro consumidor no podrá adquirir una conexión en:

```text
RESETTING
```

---

# 139. Retirement lifecycle

```text
Retirement requested
       │
       ▼
currently leased?
       │
       ├── yes → mark retire-on-release
       │
       └── no  → close now
```

---

# 140. Retirement reasons

```text
max lifetime
max uses
idle timeout
configuration generation change
credential rotation
health failure
reset failure
pool shutdown
driver failure
administrative retirement
```

---

# 141. Retire-on-release

Una conexión activa no deberá cerrarse arbitrariamente por maintenance ordinario.

---

# 142. Immediate invalidation

Excepcionalmente puede ser necesario invalidar inmediatamente por:

```text
security compromise
credential revocation
fatal protocol condition
runtime shutdown
```

---

# 143. Graceful retirement

Preferencia:

```text
finish safe operation
→ retire
```

---

# 144. Connection invalidation

`invalidate()` significa:

> El recurso ya no debe ser considerado apto para nuevas operaciones.

No necesariamente implica cierre instantáneo.

---

# 145. Invalidation states

Puede producir:

```text
retire-on-release
immediate close
broken
tainted
```

según causa.

---

# 146. Configuration generation lifecycle

Cuando cambia configuración:

```text
Generation 1
      │
      ▼
Generation 2 published
```

los recursos antiguos deberán identificarse.

---

# 147. Old generation policy

```text
idle old resource
→ retire

leased old resource
→ retire on release
```

---

# 148. No in-place identity mutation

No deberá modificarse una conexión abierta para convertirla arbitrariamente a una nueva definición.

---

# 149. Credential generation lifecycle

```text
Credentials v1
     │
     ▼
rotate
     │
     ▼
Credentials v2
```

Nuevas conexiones usarán v2.

---

# 150. Existing credential sessions

Según security policy:

```text
finish existing lease
→ retire
```

o:

```text
immediate invalidate
```

---

# 151. Credential material lifecycle

Credenciales resueltas deberán existir durante el mínimo tiempo necesario.

---

# 152. No credential retention

Un `PhysicalConnectionMetadata` no deberá almacenar:

```text
plaintext password
token
private key
```

---

# 153. Pool lifecycle

Pool states:

```text
CREATED
RUNNING
DRAINING
CLOSING
CLOSED
```

---

# 154. CREATED

Pool configurado pero todavía no necesariamente utilizado.

---

# 155. RUNNING

Acepta acquisitions.

---

# 156. DRAINING

No acepta nuevas acquisitions.

Permite finalizar leases existentes.

---

# 157. CLOSING

Está cerrando recursos restantes.

---

# 158. CLOSED

No puede volver a utilizarse.

---

# 159. Pool lifecycle and connections

```text
Pool RUNNING
├── idle
├── leased
└── creating

Pool DRAINING
├── no new acquisitions
├── close idle
└── wait active leases

Pool CLOSED
└── no resources
```

---

# 160. Worker shutdown lifecycle

```text
Worker shutdown
      │
      ▼
Stop new execution scopes
      │
      ▼
Drain database pools
      │
      ▼
Finish/cancel active operations
      │
      ▼
Close physical resources
      │
      ▼
Dispose database worker services
```

---

# 161. Shutdown deadline

El runtime deberá proporcionar un deadline.

---

# 162. Deadline exceeded

Si leases permanecen activos:

```text
invalidate
→ close when possible
```

según shutdown policy.

---

# 163. Application shutdown

El shutdown del subsystem deberá ser idempotente.

---

# 164. Repeated shutdown

```text
shutdown()
shutdown()
```

no deberá corromper accounting.

---

# 165. Request termination lifecycle

Este es uno de los boundaries más importantes de VoltStack.

```text
HTTP Request Ends
       │
       ▼
Database Scope Cleanup
```

---

# 166. Request cleanup order

Orden conceptual:

```text
1. stop new database operations
2. close open cursors/results
3. resolve active transactions
4. release/discard outstanding leases
5. clear ORM state
6. clear connection logical state
7. clear tenant/query/hydration context
8. dispose DatabaseContext
```

---

# 167. Cleanup dependency ordering

El orden exacto deberá respetar dependencias.

Por ejemplo:

```text
EntityManager cleanup
```

no deberá intentar hacer lazy queries después de iniciar cierre de conexiones.

---

# 168. Freeze before cleanup

Podrá marcarse:

```text
DatabaseContext
→ CLOSING
```

para impedir nuevas operaciones durante cleanup.

---

# 169. Context states

```text
CREATED
ACTIVE
CLOSING
CLOSED
```

---

# 170. Operation during CLOSING

Deberá fallar claramente.

---

# 171. Scope cleanup coordinator

Podrá existir:

```text
DatabaseScopeLifecycle
```

o:

```text
DatabaseLifecycleCoordinator
```

---

# 172. Cleanup participants

Participantes posibles:

```text
ResultLifecycle
TransactionLifecycle
LeaseLifecycle
OrmLifecycle
ConnectionLifecycle
TenantContextLifecycle
```

---

# 173. Deterministic ordering

Los participantes no deberán depender del orden accidental de destrucción de objetos PHP.

---

# 174. Cleanup priority

El orden deberá ser explícito y testeable.

---

# 175. Cleanup exception handling

Un fallo limpiando un componente no deberá impedir limpiar los demás.

---

# 176. Example

```text
cursor close fails
→ record failure
→ continue transaction cleanup
→ continue lease discard
→ continue ORM cleanup
```

---

# 177. Aggregated cleanup error

En desarrollo podrá producirse:

```text
DatabaseLifecycleCleanupException
```

con múltiples causas.

---

# 178. Production cleanup

La policy podrá priorizar:

```text
resource safety
telemetry
worker survival
```

sin ocultar fallos críticos.

---

# 179. Leak detection

Al terminar scope deberá verificarse:

```text
open leases == 0
active transactions == 0
open cursors == 0
```

---

# 180. Leak diagnostics

Si no:

```text
DatabaseResourceLeakDetected
```

---

# 181. Development mode

Podrá incluir:

```text
resource ID
acquisition timestamp
scope ID
owner type
optional acquisition trace
```

---

# 182. Production mode

Deberá evitar stack traces costosos por cada adquisición.

---

# 183. Leak recovery

Detectar leak no es suficiente.

El lifecycle deberá intentar:

```text
close
rollback
discard
release accounting
```

---

# 184. Lease leak

```text
active lease
→ force lifecycle cleanup
```

---

# 185. Transaction leak

```text
active transaction
→ rollback
→ reset/discard
```

---

# 186. Cursor leak

```text
open cursor
→ close
→ assess connection
```

---

# 187. Logical connection leak

Las referencias PHP pueden seguir existiendo, pero deberán quedar inválidas mediante scope token.

---

# 188. Persistent worker invariant

Después de finalizar un scope:

```text
request-specific database state = 0
```

conceptualmente.

---

# 189. Persistent runtime model

```text
Worker
│
├── Persistent Safe State
│   ├── compiled config
│   ├── registries
│   ├── pools
│   └── idle physical resources
│
├── Scope A
│   └── temporary database state
│
├── RESET
│
├── Scope B
│   └── temporary database state
│
└── RESET
```

---

# 190. FrankenPHP lifecycle

FrankenPHP será el runtime de referencia inicial.

```text
FrankenPHP Worker Start
        │
        ▼
VoltStack Application Boot
        │
        ▼
Database Worker Services
        │
        ▼
Request A Begin
        │
        ▼
DatabaseContext A
        │
        ▼
Acquire / Use / Release
        │
        ▼
Request A Cleanup
        │
        ▼
Request B Begin
```

---

# 191. FrankenPHP invariant

`Request B` nunca deberá observar:

```text
transaction state from A
tenant from A
session mutation from A
cursor from A
EntityManager from A
IdentityMap from A
UnitOfWork from A
```

---

# 192. FrankenPHP pool reuse

Sí podrá reutilizar:

```text
clean physical connection
```

porque:

```text
physical resource reuse
≠
request state reuse
```

---

# 193. RoadRunner lifecycle

RoadRunner deberá mapear sus worker/job/request hooks hacia:

```text
ExecutionScope begin
ExecutionScope end
Worker shutdown
```

---

# 194. RoadRunner architecture

No deberán introducirse:

```text
if RoadRunner
```

dentro de Connection core.

---

# 195. OpenSwoole lifecycle

OpenSwoole requerirá especial atención a concurrencia.

```text
Worker
├── Coroutine A
│   └── DatabaseContext A
└── Coroutine B
    └── DatabaseContext B
```

---

# 196. OpenSwoole isolation

Nunca:

```text
global current DatabaseContext
```

---

# 197. Coroutine-safe scope storage

El runtime adapter deberá proporcionar almacenamiento contextual compatible.

---

# 198. Concurrent lease isolation

```text
Coroutine A
→ Lease A

Coroutine B
→ Lease B
```

salvo afinidad explícitamente compartida por una abstracción segura.

---

# 199. Physical connection concurrency

Por defecto:

```text
one leased physical connection
=
one exclusive owner
```

---

# 200. Concurrent multiplexing

Sólo podrá habilitarse si:

```text
Driver Capability
+
Protocol Capability
+
Implementation Guarantee
```

lo permiten explícitamente.

---

# 201. CLI lifecycle

Un comando CLI también tendrá:

```text
ExecutionScope
```

aunque sólo exista uno durante el proceso.

---

# 202. Long-running CLI

Para procesos como:

```text
import
consumer
daemon
migration orchestration
```

podrán existir múltiples sub-scopes.

---

# 203. Queue worker lifecycle

```text
Worker
├── Job A → DatabaseContext A
├── reset
├── Job B → DatabaseContext B
└── reset
```

---

# 204. Job failure

Aunque un Job lance excepción:

```text
Database cleanup
```

deberá ejecutarse.

---

# 205. Finally semantics

El runtime integration deberá comportarse conceptualmente como:

```php
$scope = $databaseLifecycle->begin($execution);

try {
    $application->handle($execution);
} finally {
    $databaseLifecycle->end($scope);
}
```

---

# 206. Scheduled tasks

Cada ejecución programada tendrá su propio scope.

---

# 207. WebSocket considerations

Una conexión WebSocket puede vivir mucho tiempo.

No deberá equivaler automáticamente a:

```text
one permanent database lease
```

---

# 208. WebSocket operation scopes

Preferencia:

```text
WebSocket connection
├── Message A DB scope
├── Message B DB scope
└── Message C DB scope
```

cuando sea apropiado.

---

# 209. Long-lived business operation

Si realmente requiere una conexión larga, deberá declararse explícitamente.

---

# 210. Connection timeout lifecycle

VoltStack distinguirá:

```text
acquisition timeout
connect timeout
statement timeout
transaction timeout
idle timeout
max lifetime
shutdown timeout
```

---

# 211. No generic timeout

Una sola propiedad:

```text
timeout
```

para todos estos conceptos deberá evitarse.

---

# 212. Deadlines

Cuando sea posible se preferirá:

```text
absolute operation deadline
```

internamente para evitar timeout inflation.

---

# 213. Example

```text
overall deadline = 5s

acquire consumes 2s
remaining query budget = 3s
```

---

# 214. Cancellation propagation

```text
Execution Cancellation
       │
       ├── wait queue
       ├── connect
       ├── statement
       ├── streaming
       └── transaction orchestration
```

cuando las capabilities lo permitan.

---

# 215. Cancellation does not imply reusable connection

Después de cancelar una operación deberá evaluarse:

```text
connection state
```

---

# 216. Protocol uncertainty

Si la cancelación deja estado incierto:

```text
TAINTED
→ discard
```

---

# 217. Connection lifecycle events

Eventos conceptuales:

```text
LogicalConnectionResolved
PhysicalConnectionCreating
PhysicalConnectionCreated
PhysicalConnectionInitializing
PhysicalConnectionInitialized
PhysicalConnectionValidationStarted
PhysicalConnectionValidated
ConnectionLeaseAcquired
ConnectionLeaseReleased
ConnectionLeaseDiscarded
PhysicalConnectionResetStarted
PhysicalConnectionReset
PhysicalConnectionTainted
PhysicalConnectionBroken
PhysicalConnectionRetiring
PhysicalConnectionClosed
ConnectionLifecycleCleanupStarted
ConnectionLifecycleCleanupCompleted
ConnectionResourceLeakDetected
```

---

# 218. Events vs lifecycle control

Los eventos son observacionales.

No deberán convertirse en el mecanismo principal que hace funcionar el lifecycle.

---

# 219. Event listener restrictions

Un listener no deberá:

```text
retain lease
change ownership
silently prevent reset
silently prevent close
```

---

# 220. Telemetry

Métricas sugeridas:

```text
database.connection.create.duration
database.connection.initialize.duration
database.connection.validation.duration
database.connection.reset.duration
database.connection.lease.duration
database.connection.lifetime
database.connection.closed
database.connection.broken
database.connection.tainted
database.connection.leak
```

---

# 221. Trace spans

Podrán existir:

```text
db.connection.acquire
db.connection.connect
db.connection.initialize
db.connection.reset
```

siempre evitando overhead excesivo.

---

# 222. Lease duration

Será una métrica especialmente útil para detectar:

```text
long transactions
long streams
resource leaks
pool contention
```

---

# 223. Connection age

Permite analizar:

```text
resource churn
excessive reconnection
overly long-lived resources
```

---

# 224. Sensitive telemetry

Nunca deberá incluir:

```text
password
access token
private key
credential payload
unsafe DSN
```

---

# 225. Connection identity telemetry

Preferir:

```text
logical connection name
driver family
role
pool identity
```

con cardinalidad controlada.

---

# 226. Tenant telemetry

Tenant IDs dinámicos no deberán convertirse en labels de alta cardinalidad por defecto.

---

# 227. Lifecycle diagnostics

Un snapshot podrá mostrar:

```text
Connection: primary
Logical state: READY
Pool: primary
Physical resources:
  idle: 4
  leased: 2
  resetting: 0
  broken: 0

Scope:
  leases: 1
  transactions: 0
  cursors: 1
```

---

# 228. Debug Toolbar

La integración futura podrá mostrar:

```text
connection acquisitions
physical reuse
lease durations
transaction pinning
pool waits
reset failures
```

---

# 229. Security lifecycle

El lifecycle deberá respetar:

```text
credential lifetime
TLS policy
session identity
role reset
tenant isolation
native resource isolation
```

---

# 230. Role reset

Una conexión usada con:

```text
SET ROLE privileged_role
```

no deberá entregarse al siguiente scope conservando ese role.

---

# 231. Schema/search-path reset

Igualmente:

```text
Tenant A schema
```

no deberá contaminar:

```text
Tenant B
```

---

# 232. Database selection reset

Cambiar database/session catalog mediante APIs nativas deberá considerarse mutación crítica.

---

# 233. Security-first retirement

Cuando la seguridad no pueda demostrarse:

```text
discard
```

es preferible a reuse.

---

# 234. Resilience lifecycle

Connection lifecycle reporta estados y fallos.

No decide por sí solo:

```text
retry query
retry transaction
switch replica
failover primary
```

---

# 235. Failure ownership

```text
Driver
→ classify native failure

Connection Lifecycle
→ update resource state

Pool
→ reuse/discard accounting

Resilience
→ retry/failover decision

Transaction
→ transaction recovery
```

---

# 236. Connection retry

Una conexión fallida puede reemplazarse por otra.

Pero eso no significa que la operación SQL pueda repetirse.

---

# 237. Example

```text
INSERT sent
network failure before response
```

VoltStack puede no saber si el INSERT ocurrió.

Por tanto:

```text
new connection
≠
safe query retry
```

---

# 238. Connection lifecycle must preserve ambiguity

El error deberá conservar información suficiente para que capas superiores tomen una decisión correcta.

---

# 239. Physical connection replacement

```text
P1 broken
→ close P1
→ capacity available
→ create P2
```

es responsabilidad legítima del resource layer.

---

# 240. Logical connection continuity

La conexión lógica:

```text
primary
```

puede continuar existiendo aunque P1 sea reemplazada por P2.

---

# 241. This is why logical and physical connection differ

```text
Logical Connection
     │
     ├── P1
     └── P2
```

durante su lifecycle.

---

# 242. Connection lifecycle contracts

Contratos conceptuales:

```text
ConnectionLifecycleInterface
PhysicalConnectionLifecycleInterface
ConnectionLeaseLifecycleInterface
ConnectionInitializerInterface
ConnectionResetterInterface
ConnectionHealthValidatorInterface
ConnectionRetirementPolicyInterface
ConnectionLifecycleObserverInterface
```

---

# 243. Avoid mega lifecycle interface

No deberá crearse:

```text
DatabaseConnectionLifecycleManager
```

con decenas de responsabilidades.

---

# 244. Coordinator vs specialized components

Preferir:

```text
LifecycleCoordinator
   │
   ├── Initializer
   ├── Validator
   ├── Resetter
   ├── RetirementPolicy
   └── ResourceCloser
```

---

# 245. Lifecycle coordinator

Coordina transitions.

No implementa toda la lógica especializada.

---

# 246. ResourceCloser

Podrá existir una abstracción dedicada a cierre seguro.

---

# 247. Close semantics

```text
close requested
→ stop new operations
→ close dependent native resources
→ close native connection
→ mark CLOSED
```

---

# 248. Close idempotency

El cierre deberá tolerar múltiples intentos controlados.

---

# 249. Close failure

Aunque el native close produzca error:

```text
VoltStack resource state
→ CLOSED/UNUSABLE
```

y no deberá volver al Pool.

---

# 250. Native resource resurrection

Prohibido:

```text
CLOSED
→ reconnect same PhysicalConnection object
```

como comportamiento implícito.

---

# 251. Reconnect semantics

Si se necesita reconectar:

```text
old PhysicalConnection
→ closed

new PhysicalConnection
→ new resource identity
```

---

# 252. Resource identity invariant

Un `PhysicalConnectionId` representará una sola vida física.

---

# 253. Why?

Mejora:

```text
telemetry
debugging
leak tracking
failure analysis
generation tracking
```

---

# 254. Connection metadata

Podrá incluir:

```text
physical connection ID
pool ID
driver ID
configuration generation
credential generation
createdAt
initializedAt
lastLeasedAt
lastReleasedAt
lastValidatedAt
leaseCount
state
health
```

---

# 255. Metadata mutability

Runtime metadata puede cambiar internamente.

No deberá exponerse como mutable public API.

---

# 256. Clock abstraction

Lifecycle y tests deberán utilizar:

```text
ClockInterface
```

para:

```text
idle timeout
max lifetime
lease duration
deadlines
```

---

# 257. No direct `time()` everywhere

Centralizar el reloj mejora determinismo de tests.

---

# 258. Runtime synchronization

Las transiciones concurrentes podrán requerir:

```text
mutex
atomic state
channel
semaphore
runtime-specific primitive
```

---

# 259. Core abstraction

Database core no deberá depender directamente de primitivas exclusivas de OpenSwoole.

---

# 260. Runtime adapter

Podrá proporcionar:

```text
SynchronizationStrategy
ExecutionContextStorage
CancellationPrimitive
```

---

# 261. Lifecycle concurrency invariant

Un mismo recurso físico no podrá atravesar dos transitions incompatibles simultáneamente.

---

# 262. Example race

```text
Coroutine A → release P1
Coroutine B → shutdown P1
```

deberá resolverse determinísticamente.

---

# 263. State transition ownership

La máquina de estados/lifecycle coordinator deberá serializar transitions relevantes.

---

# 264. Performance model

El lifecycle no deberá introducir overhead excesivo en:

```text
acquire
release
query execution
```

---

# 265. Hot path

Ideal:

```text
acquire idle
→ state transition
→ lease
```

sin:

```text
reflection
container lookup
config parsing
driver discovery
```

---

# 266. Slow-path operations

Pueden incluir:

```text
physical connection creation
platform discovery
strict reset
health validation
retirement
```

---

# 267. State tracking cost

Los estados deberán modelarse de manera suficientemente ligera para high-throughput workloads.

---

# 268. Debug vs production instrumentation

Development puede registrar más información.

Production deberá usar instrumentación optimizada.

---

# 269. Lifecycle extension points

Podrán permitirse extensiones controladas en:

```text
connection initialization
health validation
reset strategy
retirement policy
diagnostics
runtime integration
```

---

# 270. No arbitrary lifecycle hooks

No se permitirá que plugins alteren transitions críticas sin contrato explícito.

---

# 271. Extension lifecycle requirements

Una extensión deberá declarar:

```text
supported connection types
runtime safety
statefulness
ordering
failure behavior
```

---

# 272. Lifecycle extension failure

Una extensión crítica que falle durante reset deberá provocar:

```text
connection discard
```

antes que reuse inseguro.

---

# 273. Optional observer failure

Un fallo de telemetry observacional no deberá normalmente invalidar la conexión.

---

# 274. Security extension failure

Un fallo en una etapa requerida de seguridad deberá impedir reutilización.

---

# 275. Testing strategy

El lifecycle deberá probarse como máquina de estados y como integración completa.

---

# 276. State machine tests

Verificar:

```text
NEW → CONNECTING
CONNECTING → INITIALIZING
INITIALIZING → READY
READY → LEASED
LEASED → RESETTING
RESETTING → IDLE
IDLE → LEASED
RESETTING → RETIRING
RETIRING → CLOSED
```

---

# 277. Invalid transition tests

Ejemplos:

```text
CLOSED → LEASED
BROKEN → IDLE without recovery
LEASED → IDLE without release/reset
```

deberán fallar.

---

# 278. Lazy connection test

```text
boot application

assert:
physical connections = 0
```

---

# 279. First-use test

```text
execute first query

assert:
physical connection created lazily
```

---

# 280. Reuse test

```text
Scope A
→ acquire P1
→ release/reset

Scope B
→ acquire

assert:
P1 may be reused
```

---

# 281. Isolation test

```text
Scope A:
SET session variable

cleanup

Scope B:
acquire same P1

assert:
baseline restored
```

---

# 282. Transaction cleanup test

```text
Scope A:
BEGIN
throw exception
scope ends

assert:
rollback attempted
no active transaction remains
```

---

# 283. Rollback failure test

```text
rollback fails

assert:
connection discarded
```

---

# 284. Cursor leak test

```text
open cursor
scope ends without close

assert:
cursor cleanup attempted
connection reset or discarded
```

---

# 285. Lease use-after-release test

```text
lease.release()
lease.execute()

assert:
fails
```

---

# 286. Double release test

```text
lease.release()
lease.release()

assert:
no duplicate idle resource
```

---

# 287. Connection generation test

```text
P1 generation 1
configuration becomes generation 2

assert:
P1 retires safely
```

---

# 288. Credential rotation test

```text
P1 credentials v1
rotate to v2

assert:
new resources use v2
P1 retires according to policy
```

---

# 289. Broken connection test

```text
network fatal failure

assert:
P1 marked broken
P1 never returns idle
```

---

# 290. Non-fatal query failure test

```text
constraint violation

assert:
P1 not automatically broken
```

---

# 291. Cancellation test

```text
cancel query

assert:
connection state evaluated
unsafe resource discarded
```

---

# 292. Shutdown test

```text
worker shutdown
active + idle connections

assert:
new acquisitions stop
idle close
active drain
deadline enforced
```

---

# 293. Persistent runtime test

Ejecutar miles de scopes:

```text
Scope 1
reset
Scope 2
reset
...
Scope N
```

y comprobar:

```text
no transaction leakage
no tenant leakage
no cursor leakage
no lease leakage
stable pool accounting
```

---

# 294. Concurrent lifecycle test

Simular:

```text
concurrent acquire
release
shutdown
timeout
cancellation
```

y comprobar invariantes.

---

# 295. FrankenPHP integration test

Deberá comprobar específicamente:

```text
worker reused
application not rebooted
physical connections reused
request database state not reused
```

---

# 296. RoadRunner conformance test

Mismos invariantes con adapter RoadRunner.

---

# 297. OpenSwoole conformance test

Además:

```text
coroutine isolation
concurrent pool accounting
no shared current scope
```

---

# 298. CLI conformance test

```text
command scope begins
database used
command ends
cleanup executes
```

---

# 299. Queue conformance test

```text
Job A fails
cleanup
Job B starts

assert:
B clean
```

---

# 300. Architecture tests

Deberán impedir dependencias como:

```text
ConnectionLifecycle → ORM
ConnectionLifecycle → QueryBuilder
ConnectionLifecycle → HTTP Request
ConnectionLifecycle → Controller
ConnectionLifecycle → concrete FrankenPHP API
```

---

# 301. Suggested namespaces

```text
VoltStack\Quantum\Database\Connection\Lifecycle
│
├── Contract
│   ├── ConnectionLifecycleInterface.php
│   ├── PhysicalConnectionLifecycleInterface.php
│   ├── ConnectionInitializerInterface.php
│   ├── ConnectionResetterInterface.php
│   ├── ConnectionHealthValidatorInterface.php
│   ├── ConnectionRetirementPolicyInterface.php
│   └── ConnectionLifecycleObserverInterface.php
│
├── Coordinator
│   └── ConnectionLifecycleCoordinator.php
│
├── State
│   ├── LogicalConnectionState.php
│   ├── PhysicalConnectionState.php
│   ├── ConnectionLeaseState.php
│   ├── ConnectionHealthState.php
│   └── DatabaseContextState.php
│
├── Initialization
│   ├── ConnectionInitializer.php
│   ├── ConnectionInitializationContext.php
│   └── ConnectionInitializationResult.php
│
├── Reset
│   ├── ConnectionResetter.php
│   ├── ConnectionResetContext.php
│   ├── ConnectionResetResult.php
│   └── ConnectionSessionStateTracker.php
│
├── Validation
│   ├── ConnectionHealthValidator.php
│   ├── ConnectionValidationContext.php
│   └── ConnectionValidationResult.php
│
├── Retirement
│   ├── ConnectionRetirementPolicy.php
│   ├── ConnectionRetirementReason.php
│   └── ConnectionRetirementDecision.php
│
├── Scope
│   ├── DatabaseScopeLifecycle.php
│   ├── DatabaseContext.php
│   ├── DatabaseContextState.php
│   └── DatabaseResourceTracker.php
│
├── Lease
│   ├── ConnectionLeaseLifecycle.php
│   └── ConnectionLeaseTracker.php
│
├── Resource
│   ├── PhysicalConnectionMetadata.php
│   ├── PhysicalConnectionId.php
│   └── PhysicalConnectionCloser.php
│
├── Diagnostics
│   ├── ConnectionLifecycleDiagnostics.php
│   ├── ConnectionLifecycleSnapshot.php
│   └── DatabaseResourceLeakDetector.php
│
└── Exception
    ├── ConnectionLifecycleException.php
    ├── InvalidConnectionStateTransitionException.php
    ├── ConnectionInitializationException.php
    ├── ConnectionResetException.php
    ├── ConnectionValidationException.php
    ├── ConnectionAlreadyClosedException.php
    └── DatabaseResourceLeakException.php
```

---

# 302. Relationship with previous documents

```text
10_DATABASE_DRIVER_ARCHITECTURE
            │
            ▼
11_DATABASE_CONNECTION_SYSTEM
            │
            ▼
12_DATABASE_CONNECTION_MANAGER
            │
            ▼
13_DATABASE_CONNECTION_CONFIGURATION_AND_RESOLUTION
            │
            ▼
14_DATABASE_CONNECTION_POOLING_SYSTEM
            │
            ▼
15_DATABASE_CONNECTION_LIFECYCLE_SYSTEM
```

Cada documento responde una pregunta distinta:

```text
10 Driver
→ ¿cómo hablamos nativamente con DB?

11 Connection
→ ¿qué representa una conexión para VoltStack?

12 Manager
→ ¿cómo obtenemos/resolvemos conexiones?

13 Configuration/Resolution
→ ¿cómo sabemos cuál definición usar?

14 Pool
→ ¿cómo reutilizamos recursos físicos?

15 Lifecycle
→ ¿cómo controlamos de forma segura toda su vida?
```

---

# 303. Relationship with runtime architecture

Este documento implementa específicamente para Connection los principios definidos en:

```text
08_DATABASE_LIFECYCLE_AND_RUNTIME_MODEL.md
```

La relación es:

```text
Database Runtime Lifecycle
          │
          ▼
Execution Scope
          │
          ▼
DatabaseContext
          │
          ▼
Connection Lifecycle
          │
          ▼
Lease Lifecycle
          │
          ▼
Physical Resource Lifecycle
```

---

# 304. Architectural invariants

## DB-CONN-LIFE-001

La vida de una conexión lógica será independiente de la vida de una conexión física.

## DB-CONN-LIFE-002

La existencia de una conexión lógica no implicará una conexión física abierta.

## DB-CONN-LIFE-003

Las conexiones físicas se abrirán lazy por defecto.

## DB-CONN-LIFE-004

Todo uso de un recurso físico tendrá ownership explícito.

## DB-CONN-LIFE-005

Un recurso leased no podrá utilizarse simultáneamente por otro owner incompatible.

## DB-CONN-LIFE-006

Una conexión no podrá regresar a idle antes de completar reset.

## DB-CONN-LIFE-007

Una conexión broken nunca regresará directamente a idle.

## DB-CONN-LIFE-008

Una conexión tainted sólo podrá reutilizarse si existe una recuperación explícitamente demostrada como segura.

## DB-CONN-LIFE-009

Un reset fallido provocará retiro del recurso.

## DB-CONN-LIFE-010

Una conexión cerrada no será resucitada implícitamente.

## DB-CONN-LIFE-011

Reconnect creará una nueva identidad física.

## DB-CONN-LIFE-012

Una transaction mantendrá afinidad con su conexión física.

## DB-CONN-LIFE-013

Un cursor streaming mantendrá su lease mientras dependa de la conexión.

## DB-CONN-LIFE-014

Un scope no podrá terminar dejando una transaction activa sin cleanup.

## DB-CONN-LIFE-015

Un scope no podrá terminar dejando leases activos sin recovery.

## DB-CONN-LIFE-016

El lifecycle no dependerá del garbage collector como mecanismo principal de cleanup.

## DB-CONN-LIFE-017

El lifecycle será explícito e idempotente donde corresponda.

## DB-CONN-LIFE-018

Los recursos scoped serán invalidados al finalizar su execution scope.

## DB-CONN-LIFE-019

Una referencia scoped de un request anterior no podrá reutilizarse en otro request.

## DB-CONN-LIFE-020

El worker podrá conservar únicamente estado declarado persistent-safe.

## DB-CONN-LIFE-021

El estado de sesión deberá restaurarse antes de reuse.

## DB-CONN-LIFE-022

Un health check exitoso no implicará que la sesión esté limpia.

## DB-CONN-LIFE-023

Un reset exitoso y un health check serán conceptos distintos.

## DB-CONN-LIFE-024

Las credenciales plaintext no permanecerán en metadata de lifecycle.

## DB-CONN-LIFE-025

Cambios de configuración podrán invalidar generaciones físicas anteriores.

## DB-CONN-LIFE-026

Credential rotation podrá retirar recursos anteriores sin detener todo el subsystem.

## DB-CONN-LIFE-027

El Pool no decidirá retry de queries.

## DB-CONN-LIFE-028

El Connection Lifecycle no decidirá failover topology.

## DB-CONN-LIFE-029

Los errores nativos serán clasificados antes de decidir el estado del recurso.

## DB-CONN-LIFE-030

Un error SQL no fatal no destruirá automáticamente una conexión.

## DB-CONN-LIFE-031

Un error fatal de transporte/protocolo impedirá reutilización.

## DB-CONN-LIFE-032

El cleanup continuará de forma segura aunque uno de sus participantes falle.

## DB-CONN-LIFE-033

Los recursos no se publicarán como disponibles durante RESETTING.

## DB-CONN-LIFE-034

El lifecycle será seguro bajo FrankenPHP persistent workers.

## DB-CONN-LIFE-035

El lifecycle será adaptable a RoadRunner.

## DB-CONN-LIFE-036

El lifecycle permitirá aislamiento concurrente para OpenSwoole.

## DB-CONN-LIFE-037

El core no dependerá de APIs específicas de FrankenPHP, RoadRunner u OpenSwoole.

## DB-CONN-LIFE-038

HTTP no será el concepto base del lifecycle; `ExecutionScope` lo será.

## DB-CONN-LIFE-039

CLI, Queue, Scheduler y WebSocket utilizarán el mismo modelo conceptual.

## DB-CONN-LIFE-040

Toda transición crítica será determinista y testeable.

---

# 305. Anti-pattern — Connection equals PDO

```php
class Connection
{
    public PDO $pdo;
}
```

como arquitectura completa.

**Evitar.**

VoltStack necesita distinguir:

```text
Logical Connection
Physical Connection
Native Connection
Lease
```

---

# 306. Anti-pattern — Connect at bootstrap

```php
public function boot()
{
    $this->pdo = new PDO(...);
}
```

**Prohibido como comportamiento predeterminado.**

---

# 307. Anti-pattern — Request end means cleanup

```text
"PHP will clean it eventually"
```

**Inválido para VoltStack persistent runtime.**

---

# 308. Anti-pattern — Static current connection

```php
Connection::$current = $connection;
```

**Prohibido.**

---

# 309. Anti-pattern — Return dirty connection

```text
lease
→ SET ROLE admin
→ release
→ idle
```

sin reset.

**Prohibido.**

---

# 310. Anti-pattern — Release active transaction

```text
BEGIN
→ release connection
```

**Prohibido.**

---

# 311. Anti-pattern — Release active cursor

```text
open stream
→ release connection
```

mientras el cursor sigue dependiendo del recurso.

**Prohibido.**

---

# 312. Anti-pattern — Silent reconnect

```text
query fails
→ Connection silently reconnects
→ repeats query
```

**Prohibido como comportamiento general.**

Puede romper semántica transaccional e idempotencia.

---

# 313. Anti-pattern — Reset by ping

```text
SELECT 1
→ session considered clean
```

**Incorrecto.**

---

# 314. Anti-pattern — One lifecycle for everything

```text
ConnectionLifecycleManager
```

controlando:

```text
driver
pool
transactions
ORM
queries
tenant routing
telemetry
```

**God Object. Prohibido.**

---

# 315. Anti-pattern — Runtime conditionals

```php
if ($runtime === 'openswoole') {
    // connection lifecycle
}
```

en core.

**Prohibido.**

---

# 316. Anti-pattern — Request object dependency

```php
ConnectionLifecycle::__construct(
    Request $request
)
```

**Prohibido.**

El lifecycle depende de:

```text
ExecutionScope
```

---

# 317. Anti-pattern — Resource reuse without identity

Cada recurso deberá tener identidad y metadata suficientes para diagnostics.

---

# 318. Anti-pattern — Mutable generation

Una conexión física no deberá cambiar silenciosamente de:

```text
Generation 1
```

a:

```text
Generation 2
```

---

# 319. Anti-pattern — cleanup listener as correctness mechanism

No depender de un evento opcional para:

```text
rollback
release
reset
```

Estas operaciones pertenecen al lifecycle obligatorio.

---

# 320. Final lifecycle architecture

```text
                    APPLICATION / WORKER
                           │
                           ▼
                  ConnectionPoolManager
                           │
                           ▼
                     ConnectionPool
                           │
                    owns idle resources
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    EXECUTION SCOPE                          │
│                                                             │
│  DatabaseContext                                            │
│       │                                                     │
│       ▼                                                     │
│  LogicalConnection                                         │
│       │                                                     │
│       ▼                                                     │
│  acquire()                                                  │
│       │                                                     │
│       ▼                                                     │
│  ConnectionLease ───────────────┐                           │
│       │                         │                           │
│       ▼                         ▼                           │
│ PhysicalConnection       Transaction / Cursor               │
│       │                  may pin the Lease                  │
│       ▼                                                     │
│     Driver                                                  │
│       │                                                     │
│       ▼                                                     │
│ Native Resource                                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                    Scope Termination
                           │
                           ▼
                DatabaseLifecycle Cleanup
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
    Close Cursors      Rollback Tx     Release Leases
                                             │
                                             ▼
                                           Reset
                                             │
                                    ┌────────┴────────┐
                                    ▼                 ▼
                                  CLEAN             UNSAFE
                                    │                 │
                                    ▼                 ▼
                                  IDLE              RETIRE
                                    │                 │
                                    ▼                 ▼
                                  Pool              CLOSED
```

---

# 321. Lifecycle equation

```text
Safe Connection Lifecycle
=
Explicit Ownership
+
State Machine
+
Scoped Resources
+
Deterministic Cleanup
+
Session Reset
+
Failure Classification
+
Retirement
+
Runtime Isolation
```

---

# 322. Persistent runtime equation

```text
Persistent Worker Safety
=
Reusable Infrastructure State
-
Request State Leakage
```

En Database:

```text
Reusable:
    pools
    clean physical connections
    immutable configuration
    compiled metadata
    stateless services

Never reusable across scopes:
    active transactions
    UnitOfWork
    IdentityMap
    tenant state
    cursors
    leases
    dirty session state
```

---

# 323. Criterios de aceptación

El sistema será considerado arquitectónicamente correcto cuando:

```text
application boot does not require a database connection

logical connection can exist without physical connection

physical connections are created lazily

every physical use has explicit ownership

lease use-after-release is prevented

double release cannot corrupt pool state

transactions pin their physical resource

streaming results pin their physical resource

dirty session state is reset before reuse

unknown unsafe state causes discard

broken connections never return idle

configuration generations can retire old resources

credential rotation can retire old sessions

scope termination cleans outstanding database resources

cleanup does not depend on PHP GC

request A cannot leak state into request B

queue job A cannot leak state into job B

CLI and HTTP use the same lifecycle abstraction

FrankenPHP workers can safely reuse physical resources

RoadRunner can implement the same lifecycle

OpenSwoole can isolate concurrent execution scopes

runtime-specific code remains outside connection core

connection failures preserve enough information for resilience decisions

lifecycle transitions are deterministic

lifecycle transitions are observable

lifecycle transitions are testable
```

---

# 324. Decisión arquitectónica central

La regla más importante de este documento es:

> VoltStack reutiliza infraestructura; nunca reutiliza accidentalmente contexto de ejecución.

Esto significa:

```text
Physical Connection
```

puede sobrevivir.

Pero:

```text
Transaction
Lease
Cursor
Tenant Context
Request Session State
```

no.

---

# 325. Resultado arquitectónico

Con este modelo VoltStack podrá utilizar conexiones persistentes de forma segura sin acoplar el resto del framework a ellas.

```text
Laravel-like DX
       │
       ▼
Logical Connection API
       │
       ▼
Explicit Lifecycle
       │
       ▼
Lease / Pool
       │
       ▼
Physical Resource
       │
       ▼
Driver
```

La aplicación seguirá viendo una API sencilla.

La complejidad necesaria para:

```text
pooling
persistent workers
transactions
streaming
reset
failure recovery
concurrency
```

quedará encapsulada dentro de la infraestructura de Database.

---

# 326. Conclusión

`Connection Lifecycle System` establece el contrato temporal de toda la infraestructura de conexiones de VoltStack.

Los documentos anteriores definieron:

```text
Driver
Connection
ConnectionManager
Configuration/Resolution
Pooling
```

Este documento determina cómo todas esas piezas viven juntas en el tiempo.

La secuencia esencial será:

```text
Resolve
   │
   ▼
Acquire
   │
   ▼
Initialize / Validate
   │
   ▼
Lease
   │
   ▼
Use
   │
   ▼
Release
   │
   ▼
Reset
   │
   ├── Reuse
   └── Retire
```

y sobre runtimes persistentes:

```text
Worker Start
     │
     ├── Scope A
     │     └── Database lifecycle
     │
     ├── CLEANUP
     │
     ├── Scope B
     │     └── Database lifecycle
     │
     ├── CLEANUP
     │
     └── Worker Shutdown
```

Esta separación será una de las bases para que VoltStack pueda tratar FrankenPHP como runtime de primera clase desde su diseño inicial, manteniendo simultáneamente compatibilidad futura con RoadRunner y OpenSwoole.

---

# 327. Siguiente documento

El siguiente documento será:

```text
16_DATABASE_CONNECTION_STATE_AND_RESET_SYSTEM.md
```

y profundizará en un aspecto deliberadamente separado del lifecycle general:

```text
physical connection state
session state
dirty-state tracking
baseline state
transaction residue
cursor residue
prepared statement residue
session variables
roles
schemas
search paths
temporary objects
native escape hatch contamination
state snapshots
state mutation tracking
reset planning
reset strategies
fast reset
strict reset
native reset
platform-specific reset
reset verification
reset failure
tainted resources
connection discard
state isolation
persistent worker contamination prevention
tenant state isolation
reset telemetry
reset diagnostics
reset conformance testing
```

La relación será:

```text
15 Connection Lifecycle
        │
        ▼
"WHEN must a connection be cleaned?"
        │
        ▼
16 Connection State & Reset
        │
        ▼
"WHAT state exists and HOW is it safely cleaned?"
```

De esta forma:

```text
14 Pooling
→ ownership and reuse

15 Lifecycle
→ temporal orchestration

16 State & Reset
→ contamination control
```

quedan separados como tres responsabilidades distintas.