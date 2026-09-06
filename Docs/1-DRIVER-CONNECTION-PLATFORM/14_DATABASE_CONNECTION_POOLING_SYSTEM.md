# 14_DATABASE_CONNECTION_POOLING_SYSTEM.md

# VoltStack Quantum Database
## Connection Pooling System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 14 — Database Connection Pooling System  
**Estado:** Architecture Specification  
**Nivel:** Infrastructure Architecture  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura oficial del sistema de pooling de conexiones de:

```text
VoltStack/Quantum/Database
```

El `Connection Pooling System` será responsable de administrar recursos físicos reutilizables de conexión a bases de datos.

Su responsabilidad principal será responder:

> ¿Cómo adquirir, reutilizar, validar, resetear, retirar y cerrar conexiones físicas de forma segura y eficiente?

El Pool no decidirá qué conexión lógica necesita una operación.

Esa responsabilidad pertenece a:

```text
ConnectionManager
ConnectionResolver
Topology
```

La separación fundamental será:

```text
ConnectionManager
        │
        ▼
Logical Connection
        │
        ▼
Resource Provider
        │
        ▼
Connection Pool
        │
        ▼
ConnectionLease
        │
        ▼
Physical Connection
        │
        ▼
Driver
```

---

# 2. Principio fundamental

> El Pool administra recursos físicos; no administra semántica de Database.

Formalmente:

```text
ConnectionPool
=
Physical Resource Lifecycle
+
Resource Reuse
+
Capacity Control
+
Acquisition Coordination
+
Reset
+
Retirement
```

No:

```text
ConnectionPool
=
ConnectionManager
+
Query Router
+
Transaction Manager
+
ORM
```

---

# 3. Objetivos

El sistema deberá proporcionar:

```text
controlled physical connection reuse
bounded resource consumption
safe connection leasing
connection reset before reuse
connection health validation
broken resource detection
resource retirement
acquisition timeout
backpressure
persistent worker safety
transaction pinning compatibility
streaming compatibility
configuration generation awareness
credential rotation compatibility
telemetry
diagnostics
graceful shutdown
```

---

# 4. No objetivos

El Pool no deberá encargarse de:

```text
SQL generation
SQL parsing
Query AST
Query optimization
entity hydration
ORM persistence
connection selection
replica routing
shard selection
transaction semantics
tenant selection
migration planning
```

---

# 5. Arquitectura general

```text
Consumer
   │
   ▼
ConnectionResourceProvider
   │
   ▼
ConnectionPoolManager
   │
   ▼
ConnectionPool
   │
   ├── Idle Resources
   ├── Leased Resources
   ├── Wait Queue
   ├── Resource Factory
   ├── Health Validator
   ├── Reset Strategy
   └── Retirement Policy
           │
           ▼
    PhysicalConnection
           │
           ▼
         Driver
```

---

# 6. Pool vs ConnectionManager

La distinción será obligatoria:

| Componente | Responsabilidad |
|---|---|
| `ConnectionManager` | Resolver conexiones lógicas |
| `ConnectionPoolManager` | Resolver pools físicos |
| `ConnectionPool` | Administrar recursos físicos |
| `ConnectionLease` | Representar ownership temporal |
| `Driver` | Crear/comunicar con recurso nativo |
| `TransactionManager` | Administrar semántica transaccional |

---

# 7. Logical Connection vs Physical Connection

Una conexión lógica representa:

```text
"primary"
```

Una conexión física representa:

```text
una sesión real abierta contra PostgreSQL/MySQL/etc.
```

Por tanto:

```text
LogicalConnection
≠
PhysicalConnection
```

---

# 8. Una conexión lógica puede usar múltiples recursos físicos

Durante la vida de un worker:

```text
Logical "primary"
       │
       ├── Physical #1
       ├── Physical #2
       ├── Physical #3
       └── Physical #4
```

dependiendo de concurrencia y capacidad.

---

# 9. Un recurso físico tiene ownership

Toda conexión física deberá estar en exactamente uno de los siguientes estados conceptuales:

```text
CREATING
IDLE
LEASED
RESETTING
VALIDATING
RETIRING
CLOSED
BROKEN
```

---

# 10. Estado imposible

Nunca deberá existir una conexión simultáneamente:

```text
IDLE
+
LEASED
```

---

# 11. PhysicalConnection

VoltStack deberá encapsular el recurso nativo mediante:

```text
PhysicalConnectionInterface
```

Ejemplo conceptual:

```php
interface PhysicalConnectionInterface
{
    public function isOpen(): bool;

    public function close(): void;
}
```

Las capacidades específicas podrán estar en contratos especializados.

---

# 12. No PDO leakage

El Pool no deberá obligar al resto de Database a conocer:

```text
PDO
mysqli
PgSql\Connection
```

---

# 13. Native handle

El Driver Adapter podrá mantener internamente el recurso nativo.

```text
PhysicalConnection
        │
        ▼
Driver-specific native handle
```

---

# 14. ConnectionPool

Contrato conceptual:

```php
interface ConnectionPoolInterface
{
    public function acquire(
        ConnectionAcquisitionContext $context
    ): ConnectionLease;

    public function stats(): ConnectionPoolStatistics;

    public function shutdown(): void;
}
```

---

# 15. API deliberadamente pequeña

El consumidor ordinario no deberá llamar:

```text
create()
destroy()
reset()
validate()
```

directamente.

Estas son operaciones internas del Pool.

---

# 16. ConnectionLease

`ConnectionLease` será una pieza central.

Representa:

> Ownership temporal y exclusivo de un recurso físico.

---

# 17. Modelo

```text
Pool
 │
 ▼
acquire()
 │
 ▼
ConnectionLease
 │
 ▼
PhysicalConnection
 │
 ▼
use
 │
 ▼
release()
 │
 ▼
Pool
```

---

# 18. Lease invariant

Mientras exista un lease exclusivo:

```text
PhysicalConnection
```

no deberá entregarse simultáneamente a otro consumidor incompatible.

---

# 19. Lease is not the pool resource itself

```text
ConnectionLease
≠
PhysicalConnection
```

El Lease contiene:

```text
ownership
lifecycle
release semantics
resource reference
diagnostic metadata
```

---

# 20. Lease API conceptual

```php
interface ConnectionLeaseInterface
{
    public function connection(): PhysicalConnectionInterface;

    public function release(): void;

    public function discard(): void;
}
```

---

# 21. Idempotent release

Idealmente:

```text
release()
release()
```

no deberá devolver dos veces el recurso al Pool.

La segunda llamada deberá ser segura o producir un error controlado de desarrollo.

---

# 22. Automatic release

Podrán existir mecanismos auxiliares para garantizar liberación.

Pero la arquitectura no deberá depender únicamente de destructores PHP.

---

# 23. Why not `__destruct()` only?

En workers persistentes:

```text
GC timing
≠
execution scope termination
```

Por tanto la liberación deberá estar integrada con lifecycle explícito.

---

# 24. Scope cleanup

Al finalizar un execution scope:

```text
DatabaseLifecycle
       │
       ▼
Outstanding Lease Registry
       │
       ▼
release / discard
```

---

# 25. Lease ownership registry

Cada `DatabaseContext` podrá conocer los leases que posee.

```text
DatabaseContext
└── LeaseTracker
```

---

# 26. Pool identity

Cada Pool deberá tener:

```text
ConnectionPoolIdentity
```

independiente del nombre de Connection.

---

# 27. Why separate identity?

Varias conexiones lógicas podrían compartir una misma política/pool compatible.

O una misma conexión lógica podría requerir pools distintos por generación/runtime.

---

# 28. Pool compatibility key

La selección/reutilización deberá considerar:

```text
driver
endpoint
database
credentials generation
security policy
session baseline
native options
configuration generation
```

---

# 29. No unsafe sharing

Dos ConnectionDefinitions no deberán compartir recursos si difieren en propiedades que puedan cambiar:

```text
authentication
session semantics
security
target database
driver behavior
```

---

# 30. PoolKey

Podrá existir:

```php
final readonly class ConnectionPoolKey
{
    public function __construct(
        public ConnectionDefinitionFingerprint $definition,
        public CredentialGeneration $credentials,
    ) {}
}
```

---

# 31. Secrets in PoolKey

Nunca deberán incluirse secretos plaintext.

---

# 32. ConnectionPoolManager

Podrá existir:

```text
ConnectionPoolManager
```

como coordinador de pools.

---

# 33. PoolManager responsibilities

```text
pool lookup
pool creation
pool registry
pool generation management
pool shutdown
pool diagnostics
```

---

# 34. PoolManager does not resolve logical connections

No deberá implementar:

```text
default connection
aliases
tenant resolution
replica routing
```

---

# 35. PoolRegistry

Podrá mantener:

```text
PoolKey → ConnectionPool
```

---

# 36. Persistent PoolRegistry

En runtimes persistentes podrá vivir a nivel:

```text
worker
```

o aplicación, dependiendo del runtime.

---

# 37. Pool configuration

Ejemplo:

```yaml
database:
  pools:

    default:
      min: 0
      max: 20

      acquire_timeout: 2s
      connect_timeout: 5s

      idle_timeout: 60s
      max_lifetime: 30m
      max_uses: 10000

      health_check:
        enabled: true
        idle_threshold: 30s

      reset:
        mode: strict
```

---

# 38. Minimum capacity

```text
min
```

representa la cantidad mínima deseada de conexiones disponibles o mantenidas según policy.

No necesariamente deberán abrirse todas durante bootstrap.

---

# 39. Recommended default

Para VoltStack v1:

```text
min = 0
```

es un default razonable.

Esto preserva lazy connection.

---

# 40. Maximum capacity

```text
max
```

establece el límite máximo de recursos físicos simultáneos administrados por el Pool.

---

# 41. Capacity invariant

```text
total physical resources
<=
max
```

salvo recursos en transición estrictamente controlada durante reemplazo.

---

# 42. Why bounded capacity?

Evita:

```text
database overload
worker explosion
connection storms
resource exhaustion
```

---

# 43. Multi-worker consideration

Si existen:

```text
10 workers
×
20 connections per pool
```

el servidor podría recibir:

```text
200 connections
```

---

# 44. Global capacity awareness

VoltStack deberá documentar que:

```text
pool max
```

normalmente es un límite por proceso/worker/pool local, no necesariamente global al cluster.

---

# 45. Runtime-specific pool semantics

FrankenPHP, RoadRunner y OpenSwoole podrán implementar diferentes estrategias físicas.

Pero deberán respetar los mismos contratos conceptuales.

---

# 46. Acquisition flow

```text
acquire()
   │
   ▼
Idle healthy resource?
   │
   ├── yes ──► lease
   │
   └── no
        │
        ▼
capacity available?
   │
   ├── yes ──► create resource ──► lease
   │
   └── no
        │
        ▼
wait / backpressure
```

---

# 47. Idle resource selection

El Pool podrá seleccionar recursos idle mediante:

```text
FIFO
LIFO
least recently used
most recently validated
```

según estrategia.

---

# 48. Default strategy

Una estrategia simple y eficiente deberá utilizarse inicialmente.

La política exacta no deberá formar parte del contrato público.

---

# 49. Resource creation

Si existe capacidad:

```text
ConnectionResourceFactory
        │
        ▼
CredentialResolver
        │
        ▼
Driver
        │
        ▼
PhysicalConnection
```

---

# 50. ResourceFactory

Contrato conceptual:

```php
interface PhysicalConnectionFactoryInterface
{
    public function create(
        ConnectionDefinition $definition,
        ConnectionCreationContext $context
    ): PhysicalConnectionInterface;
}
```

---

# 51. Driver owns native creation

El Factory coordina.

El Driver realmente conoce cómo abrir el recurso nativo.

---

# 52. Connect timeout

La creación deberá respetar:

```text
connect timeout
```

separado de:

```text
acquire timeout
```

---

# 53. Acquire timeout

El acquire timeout mide:

> cuánto puede esperar el consumidor para obtener un lease.

---

# 54. Example

```text
Acquire timeout: 2s
Connect timeout: 5s
```

No son equivalentes.

---

# 55. Wait queue

Si:

```text
active = max
idle = 0
```

nuevos consumidores podrán entrar en:

```text
Wait Queue
```

---

# 56. Waiter

Podrá modelarse internamente:

```text
ConnectionAcquisitionWaiter
```

---

# 57. Wait queue requirements

Deberá soportar:

```text
timeout
cancellation
fairness policy
shutdown wakeup
```

---

# 58. Backpressure

El Pool deberá ejercer backpressure en lugar de crear conexiones ilimitadas.

---

# 59. Backpressure principle

```text
capacity exhausted
→ wait/reject
```

No:

```text
capacity exhausted
→ ignore max
→ create more connections
```

---

# 60. Acquisition failure

Si expira el timeout:

```text
ConnectionAcquisitionTimeoutException
```

---

# 61. Cancellation

Si la operación se cancela mientras espera:

```text
waiter removed
```

y no deberá recibir posteriormente un recurso abandonado.

---

# 62. Fairness

Podrán existir estrategias:

```text
FIFO
priority
weighted
```

pero v1 deberá priorizar comportamiento predecible.

---

# 63. Starvation prevention

Una política avanzada no deberá dejar consumidores esperando indefinidamente.

---

# 64. Resource validation

Antes de reutilizar una conexión idle, puede requerirse:

```text
ConnectionHealthValidator
```

---

# 65. Health validation strategies

```text
none
passive
on-acquire
idle-threshold
periodic
adaptive
```

---

# 66. Passive validation

Se basa en estado conocido:

```text
connection not marked broken
socket presumed valid
```

sin query extra.

---

# 67. Active validation

Puede ejecutar una operación ligera específica del Driver/Platform.

Ejemplo conceptual:

```text
SELECT 1
```

pero no deberá hardcodearse universalmente en Pool core.

---

# 68. HealthCheckStrategy

El Driver/Platform deberá proporcionar la estrategia correcta.

---

# 69. Avoid query per acquire

Ejecutar health check en cada adquisición puede ser costoso.

Por ello puede usarse:

```text
validate if idle > threshold
```

---

# 70. Idle validation example

```text
last_used = 45s ago
validation threshold = 30s
→ validate before lease
```

---

# 71. Broken resource

Si validation falla:

```text
IDLE
→ BROKEN
→ CLOSED
```

y el Pool intentará otro recurso según policy.

---

# 72. Validation failure is not application query failure

Deberá distinguirse en telemetry.

---

# 73. Connection reset

Antes de reutilizar un recurso deberá restaurarse un estado seguro.

---

# 74. Reset goal

```text
unknown previous session state
        │
        ▼
ConnectionResetter
        │
        ▼
known baseline state
```

---

# 75. Reset responsibilities

Puede incluir:

```text
rollback orphan transaction
clear transaction state
restore autocommit
restore isolation
restore schema/search path
restore timezone
restore session variables
clear temporary state when possible
close cursors
release prepared resources if required
restore role
```

dependiendo de plataforma.

---

# 76. Reset strategy

Contrato conceptual:

```php
interface ConnectionResetterInterface
{
    public function reset(
        PhysicalConnectionInterface $connection,
        ConnectionResetContext $context
    ): ConnectionResetResult;
}
```

---

# 77. Reset result

Podrá ser:

```text
REUSABLE
DISCARD
FAILED
```

---

# 78. Strict reset

VoltStack deberá favorecer:

```text
strict reset
```

en persistent runtimes.

---

# 79. Unknown session state

Si el framework no puede garantizar el estado:

```text
discard connection
```

es preferible a reutilizarla inseguramente.

---

# 80. Reset failure

```text
reset failed
→ do not return to idle
→ retire/discard
```

---

# 81. Reset is mandatory lifecycle boundary

El Pool no deberá asumir:

```text
request ended
=
connection automatically clean
```

---

# 82. Session baseline

La ConnectionDefinition puede proporcionar:

```text
ConnectionSessionProfile
```

como estado esperado.

---

# 83. Reset pipeline

```text
Lease Release
     │
     ▼
Check Taint
     │
     ▼
Close outstanding resources
     │
     ▼
Rollback orphan transaction
     │
     ▼
Reset session
     │
     ▼
Validate if required
     │
     ▼
Retirement policy
     │
     ├── reusable → IDLE
     └── retire   → CLOSE
```

---

# 84. Tainted connection

Una conexión podrá marcarse:

```text
TAINTED
```

cuando su estado no pueda garantizarse.

---

# 85. Taint causes

Ejemplos:

```text
protocol error
network interruption
failed transaction state
unknown native state
reset failure
driver-reported corruption
unclosed streaming state
```

---

# 86. Tainted resource rule

```text
tainted
→ never return directly to idle
```

---

# 87. Discard

El Lease podrá ejecutar:

```php
$lease->discard();
```

cuando el consumidor detecta que el recurso no es reutilizable.

---

# 88. Pool release

Conceptualmente:

```text
release(lease)
     │
     ▼
usable?
     │
     ├── no → close
     └── yes
           │
           ▼
         reset
           │
           ▼
       retirement check
           │
           ├── retire
           └── idle
```

---

# 89. Resource lifetime

Cada recurso deberá registrar metadata como:

```text
createdAt
lastAcquiredAt
lastReleasedAt
lastValidatedAt
useCount
generation
healthState
```

---

# 90. Max lifetime

Configuración:

```text
max_lifetime
```

permite retirar conexiones antiguas.

---

# 91. Max lifetime behavior

Al alcanzar el límite:

```text
active resource
→ finish current lease
→ retire on release
```

No deberá cerrarse en mitad de una operación normal.

---

# 92. Max idle time

```text
idle_timeout
```

permite cerrar recursos inactivos.

---

# 93. Idle eviction

```text
IDLE
+
idle > threshold
→ retire
```

respetando `min` si aplica.

---

# 94. Max uses

Una conexión podrá retirarse después de:

```text
max_uses
```

adquisiciones/usos.

---

# 95. Why max uses?

Puede ayudar frente a:

```text
driver leaks
server-side session accumulation
long-lived native state
credential rotation
```

---

# 96. Max queries

Opcionalmente podría distinguirse:

```text
lease count
```

de:

```text
statement/query count
```

---

# 97. Avoid unnecessary complexity in v1

Se recomienda iniciar con:

```text
max lifetime
idle timeout
lease/use count
```

y añadir métricas más finas sólo si son necesarias.

---

# 98. RetirementPolicy

Podrá existir:

```php
interface ConnectionRetirementPolicyInterface
{
    public function shouldRetire(
        PhysicalConnectionMetadata $metadata
    ): bool;
}
```

---

# 99. Retirement reasons

```text
MAX_LIFETIME
MAX_IDLE
MAX_USES
CONFIGURATION_CHANGED
CREDENTIALS_CHANGED
HEALTH_FAILURE
RESET_FAILURE
DRIVER_FAILURE
POOL_SHUTDOWN
MANUAL_RETIREMENT
```

---

# 100. Retirement diagnostics

La razón deberá registrarse sin exponer secretos.

---

# 101. Connection generation

Cada recurso físico deberá asociarse a:

```text
ConfigurationGeneration
CredentialGeneration
```

cuando corresponda.

---

# 102. Configuration change

```text
Generation 10 resource
+
current generation 11
→ retire on release
```

---

# 103. No unsafe mutation

No se intentará convertir una conexión física existente de:

```text
database A
```

a:

```text
database B
```

sólo porque cambió configuración.

---

# 104. Credential rotation

Cuando cambien credenciales:

```text
new acquisitions
→ new credential generation

old resources
→ retire safely
```

---

# 105. Credential rotation without downtime

Idealmente:

```text
old active leases finish
new resources use new credentials
old idle resources close
```

---

# 106. Pool warming

VoltStack podrá soportar:

```text
pool warming
```

opcionalmente.

---

# 107. Default behavior

No deberá requerirse warming para usar Database.

---

# 108. Lazy pool

Por defecto:

```text
Pool exists
Physical resources = 0
```

hasta la primera adquisición.

---

# 109. Warmup use cases

Puede ser útil en:

```text
high-throughput workers
latency-sensitive services
known traffic bursts
```

---

# 110. Warmup failure

No necesariamente deberá impedir application boot salvo que la configuración lo declare requerido.

---

# 111. Readiness distinction

```text
Database subsystem ready
≠
Pool warmed
≠
Database reachable
```

---

# 112. Pool shutdown

El Pool deberá soportar:

```text
shutdown
```

explícito.

---

# 113. Shutdown phases

```text
RUNNING
   │
   ▼
DRAINING
   │
   ▼
CLOSING
   │
   ▼
CLOSED
```

---

# 114. Draining

Durante draining:

```text
new acquisitions
→ rejected
```

mientras leases existentes pueden finalizar.

---

# 115. Graceful shutdown

```text
stop new leases
wait bounded time
close idle resources
allow active leases to return
force close if deadline exceeded
```

---

# 116. Shutdown timeout

Deberá existir una policy para evitar shutdown infinito.

---

# 117. Worker shutdown

Runtime lifecycle deberá invocar shutdown al terminar el worker.

---

# 118. Process crash

No puede garantizarse cleanup completo ante kill/crash.

El servidor Database deberá manejar desconexión del socket.

---

# 119. Pool and transactions

Una transaction requiere afinidad a un recurso físico.

---

# 120. Transaction pinning

```text
Transaction Start
      │
      ▼
Acquire Lease A
      │
      ▼
TransactionContext
      │
      └── pinned Lease A
```

---

# 121. Queries inside transaction

```text
Query 1 ─┐
Query 2 ─┼──► Lease A
Query 3 ─┘
```

---

# 122. No release between transactional queries

Mientras la transaction esté activa:

```text
Lease A
```

no regresa al Pool.

---

# 123. Transaction completion

```text
COMMIT / ROLLBACK
        │
        ▼
transaction state cleanup
        │
        ▼
lease release
        │
        ▼
reset
        │
        ▼
pool
```

---

# 124. Orphan transaction

Si un execution scope termina con transaction activa:

```text
rollback
→ reset
→ release/discard
```

según resultado.

---

# 125. Failed transaction

Algunos motores dejan la sesión en estado abortado.

El resetter deberá conocer cómo recuperarla.

---

# 126. Unrecoverable transaction state

```text
discard physical connection
```

---

# 127. Savepoints

No alteran la ownership principal:

```text
Transaction
→ same Lease
→ same PhysicalConnection
```

---

# 128. Pool and streaming results

Un cursor streaming puede requerir mantener el lease durante toda su iteración.

---

# 129. Streaming pinning

```text
Streaming Result
      │
      ▼
ConnectionLease
      │
      ▼
PhysicalConnection
```

---

# 130. Lease release after stream

Sólo cuando:

```text
stream exhausted
or
stream explicitly closed
```

podrá liberarse el lease.

---

# 131. Abandoned streams

El lifecycle deberá detectar streams abiertos al finalizar scope.

---

# 132. Stream cleanup

```text
close cursor
→ determine connection health
→ reset
→ release/discard
```

---

# 133. Pool exhaustion by streams

Streams de larga duración pueden consumir todos los recursos.

Telemetry deberá permitir detectar este escenario.

---

# 134. Pool and prepared statements

La reutilización de prepared statements puede ser:

```text
driver-level
physical-connection-level
execution-level
```

---

# 135. Pool core does not own query statement cache

El Pool sólo deberá garantizar lifecycle correcto del recurso físico.

---

# 136. Statement cleanup

Antes de regresar un recurso:

```text
open statements/cursors
```

deberán estar cerrados o invalidados según Driver.

---

# 137. Pool and temporary tables

Las tablas temporales pueden sobrevivir dentro de la sesión.

---

# 138. Session contamination

Ejemplo:

```text
Request A
→ CREATE TEMP TABLE

connection returned

Request B
→ receives same session
```

podría provocar leakage.

---

# 139. Temporary object policy

La Platform/ResetStrategy deberá indicar si puede:

```text
clean safely
```

o si debe:

```text
discard connection
```

---

# 140. Session variables

Lo mismo aplica a:

```text
SET ROLE
SET search_path
SET timezone
SET sql_mode
custom session variables
```

---

# 141. Persistent runtime safety

Este sistema será especialmente crítico bajo:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 142. Traditional PHP lifecycle

En PHP-FPM tradicional:

```text
Request
→ open DB connection
→ process request
→ request ends
```

mucho estado desaparece naturalmente.

---

# 143. Persistent worker lifecycle

En VoltStack:

```text
Worker
  │
  ├── Request A
  ├── Request B
  ├── Request C
  └── ...
```

los recursos pueden sobrevivir.

---

# 144. Therefore

VoltStack no podrá depender de:

```text
request termination
```

para limpiar implícitamente las conexiones.

---

# 145. FrankenPHP reference model

```text
FrankenPHP Worker
│
├── ConnectionPoolManager
│
├── Pool primary
│   ├── idle P1
│   ├── idle P2
│   └── leased P3 → Request A
│
├── Request A
│   └── DatabaseContext A
│       └── Lease P3
│
└── Request B
    └── DatabaseContext B
        └── Lease P1
```

---

# 146. End Request A

```text
Request A ends
      │
      ▼
DatabaseLifecycle
      │
      ▼
close cursors
      │
      ▼
rollback orphan transaction
      │
      ▼
reset P3
      │
      ▼
return P3 to pool
```

---

# 147. Request B must never inherit A state

Esto es una invariante crítica.

---

# 148. RoadRunner compatibility

La misma arquitectura deberá funcionar con un adapter de lifecycle diferente.

---

# 149. OpenSwoole concurrency

OpenSwoole puede tener operaciones concurrentes dentro del mismo proceso.

Por ello el Pool deberá soportar:

```text
concurrent acquire
concurrent release
waiter coordination
atomic capacity accounting
```

---

# 150. No global mutable current lease

Prohibido:

```php
static ?ConnectionLease $currentLease;
```

---

# 151. Coroutine isolation

```text
Coroutine A
→ Lease 1

Coroutine B
→ Lease 2
```

aunque ambas compartan el mismo Pool.

---

# 152. Fiber safety

La implementación deberá evitar assumptions basadas exclusivamente en thread-local semantics tradicionales.

---

# 153. Synchronization abstraction

VoltStack podrá definir:

```text
PoolSynchronizationStrategy
```

para adaptar locking/coordinación al runtime.

---

# 154. Sync vs async acquisition

La API arquitectónica deberá permitir evolución futura hacia drivers/runtimes async.

---

# 155. Future-friendly result

Una implementación futura podría proporcionar:

```text
acquireAsync()
```

sin alterar el modelo conceptual de Lease.

---

# 156. v1 focus

VoltStack v1 puede implementar adquisición síncrona.

No deberá introducir complejidad async prematuramente.

---

# 157. Connection storms

Cuando muchos workers arrancan simultáneamente:

```text
Worker 1 ─┐
Worker 2 ─┤
Worker 3 ─┼──► Database
Worker N ─┘
```

puede ocurrir una tormenta de conexiones.

---

# 158. Mitigation strategies

Futuras policies pueden incluir:

```text
lazy creation
jittered warming
creation rate limits
backoff
global proxy integration
```

---

# 159. Default mitigation

La principal defensa inicial será:

```text
lazy pools
+
bounded max
```

---

# 160. Resource creation concurrency

El Pool deberá evitar superar `max` por race condition.

---

# 161. Example race

```text
current = 19
max = 20

Coroutine A sees 19
Coroutine B sees 19

both create
→ 21
```

Debe impedirse mediante coordinación correcta.

---

# 162. Reserved creation slot

El contador deberá considerar recursos:

```text
CREATING
```

como parte de capacidad reservada.

---

# 163. Capacity accounting

Conceptualmente:

```text
capacity_used =
idle
+
leased
+
creating
+
resetting
+
validating
+
retiring_if_still_open
```

según implementación.

---

# 164. Pool state machine

```text
                 ┌─────────────┐
                 │  CREATING   │
                 └──────┬──────┘
                        ▼
                    VALIDATING
                        │
                        ▼
                      IDLE
                        │
                  acquire()
                        ▼
                     LEASED
                        │
                   release()
                        ▼
                    RESETTING
                   ┌────┴────┐
                   ▼         ▼
                 IDLE      RETIRING
                             │
                             ▼
                           CLOSED
```

Failures pueden enviar cualquier estado relevante a:

```text
BROKEN
→ RETIRING
→ CLOSED
```

---

# 165. State transitions

Las transiciones deberán estar centralizadas y testeadas.

---

# 166. No arbitrary state mutation

Evitar:

```php
$resource->state = 'idle';
```

desde componentes externos.

---

# 167. PhysicalConnectionMetadata

Podrá almacenar:

```text
resource ID
pool ID
created at
last used
use count
generation
state
health
```

---

# 168. Resource IDs

Los IDs diagnósticos no deberán exponer:

```text
password
DSN with secret
```

---

# 169. Pool statistics

`ConnectionPoolStatistics` podrá incluir:

```text
capacity max
resources total
idle
leased
creating
waiting
retiring
acquisition count
acquisition timeout count
creation failure count
reset failure count
health failure count
```

---

# 170. Metrics

Métricas sugeridas:

```text
database.pool.size
database.pool.idle
database.pool.leased
database.pool.waiters
database.pool.acquire.duration
database.pool.acquire.timeout
database.pool.connection.created
database.pool.connection.closed
database.pool.connection.reset.failure
database.pool.connection.health.failure
```

---

# 171. Low-cardinality labels

Labels aceptables:

```text
pool
driver family
role
```

cuando el número sea controlado.

---

# 172. Dangerous labels

Evitar por defecto:

```text
tenant_id
full database name
dynamic shard id
host generated per tenant
```

---

# 173. Pool telemetry integration

El Pool emitirá información mediante:

```text
Database Telemetry Port
```

sin depender obligatoriamente de `Quantum/Telemetry`.

---

# 174. Null telemetry

Sin Telemetry instalado:

```text
NullDatabaseTelemetry
```

---

# 175. Events

Eventos posibles:

```text
ConnectionPoolCreated
ConnectionAcquisitionStarted
ConnectionAcquired
ConnectionAcquisitionTimedOut
PhysicalConnectionCreated
PhysicalConnectionValidated
PhysicalConnectionReset
PhysicalConnectionRetired
PhysicalConnectionClosed
ConnectionPoolDraining
ConnectionPoolClosed
```

---

# 176. Event safety

Los eventos deberán evitar credenciales y SQL sensible.

---

# 177. Event listeners cannot own resources

Un listener no deberá retener referencias a:

```text
PhysicalConnection
ConnectionLease
```

más allá de su lifecycle.

---

# 178. Pool diagnostics

Un comando futuro:

```text
php voltstack db:pool status
```

podrá mostrar:

```text
Pool: primary

Max:      20
Total:     8
Idle:      5
Leased:    3
Waiting:   0
Creating:  0
```

---

# 179. Diagnostics should not mutate

Consultar stats no deberá:

```text
open connections
run health checks
change pool state
```

salvo comando explícito.

---

# 180. Health command

Separadamente:

```text
db:pool health
```

podrá ejecutar checks activos.

---

# 181. Leak detection

Development mode podrá detectar:

```text
lease acquired
but never released
```

---

# 182. Lease trace

Opcionalmente podrá registrar:

```text
acquisition origin
scope ID
acquisition timestamp
```

---

# 183. Production overhead

Stack traces completos no deberán capturarse por cada lease en producción normal.

---

# 184. Long-held lease detection

Podrá detectarse:

```text
lease held > threshold
```

---

# 185. Possible causes

```text
slow query
long transaction
forgotten cursor
application leak
```

---

# 186. Pool exhaustion diagnostics

Cuando se agota el pool, diagnostics deberán distinguir:

```text
max capacity reached
long transactions
long streams
creation failures
database unreachable
wait queue overload
```

---

# 187. Failure taxonomy

Errores principales:

```text
PoolClosedException
PoolDrainingException
ConnectionAcquisitionTimeoutException
PhysicalConnectionCreationException
ConnectionHealthCheckException
ConnectionResetException
ConnectionLeaseException
ConnectionPoolExhaustedException
```

---

# 188. Native errors

Los errores nativos deberán ser traducidos por Driver/Connection layers antes de cruzar boundaries públicos.

---

# 189. Creation failure

Si abrir una conexión falla:

```text
reserved creation slot
→ released
```

para no reducir capacidad permanentemente.

---

# 190. Waiter wakeup

Después de una creation failure:

```text
waiters
```

deberán ser notificados/coordinados según policy.

---

# 191. Retry ownership

El Pool puede reintentar:

```text
resource acquisition mechanics
```

de forma limitada.

Pero no deberá reintentar automáticamente:

```text
application SQL operation
```

---

# 192. Connection creation retry

Podrá delegarse a:

```text
ConnectionRetryPolicy
```

con backoff controlado.

---

# 193. Circuit breaker

Podrá integrarse mediante port.

---

# 194. Circuit open

Si un target está marcado como unavailable:

```text
acquire
→ fail fast
```

según Resilience policy.

---

# 195. Pool does not choose failover target

Si el target falla:

```text
Pool
→ reports failure
```

Topology/Resilience decide si utilizar otro target.

---

# 196. Security model

El Pool deberá proteger:

```text
credentials
TLS state
native handles
session isolation
tenant isolation
diagnostic data
```

---

# 197. No credentials in Pool stats

Nunca:

```text
username/password
```

en métricas ordinarias.

---

# 198. TLS connection reuse

Una conexión creada bajo una policy TLS antigua no deberá seguir reutilizándose si una nueva generación exige una policy incompatible.

---

# 199. Credential isolation

Pools de credenciales incompatibles no deberán compartir recursos.

---

# 200. Tenant isolation

Si tenants usan credenciales o databases diferentes:

```text
Tenant A resource
```

no deberá entregarse a:

```text
Tenant B
```

---

# 201. Pool cardinality problem

Database-per-tenant puede producir:

```text
thousands of tenant pools
```

---

# 202. Important future consideration

El sistema deberá permitir estrategias como:

```text
lazy tenant pools
bounded pool registry
pool eviction
tenant pool TTL
external pool/proxy
```

sin convertir Multitenancy en dependencia core.

---

# 203. Dynamic pool registry

Cuando existan pools dinámicos deberá ser:

```text
bounded
evictable
observable
```

---

# 204. No infinite tenant pool registry

Prohibido:

```text
tenant request
→ create permanent pool forever
```

---

# 205. Pool eviction

Un pool dinámico sin:

```text
active leases
```

podrá retirarse después de cierto periodo.

---

# 206. External pooling

VoltStack deberá permitir delegar pooling a infraestructura externa.

Ejemplos conceptuales:

```text
database proxy
server-side pooler
cloud database proxy
runtime-specific pool
```

---

# 207. Pool mode

Podrá existir:

```text
DISABLED
INTERNAL
EXTERNAL
```

---

# 208. Disabled pooling

```text
acquire
→ create physical connection

release
→ close physical connection
```

---

# 209. Internal pooling

```text
acquire
→ reuse/create

release
→ reset/idle
```

---

# 210. External pooling

VoltStack puede mantener una capa lógica mínima mientras el proveedor externo gestiona reutilización real.

---

# 211. Same resource provider contract

Las tres estrategias deberán poder ocultarse detrás de:

```text
ConnectionResourceProviderInterface
```

---

# 212. ResourceProvider architecture

```text
Logical Connection
       │
       ▼
ConnectionResourceProvider
       │
       ├── Direct Provider
       ├── Internal Pool Provider
       └── External Pool Provider
```

---

# 213. This protects Query Engine

Query/Execution no necesita saber qué estrategia está activa.

---

# 214. Pool configuration validation

Deberá verificar:

```text
min >= 0
max >= 1
min <= max
timeouts valid
lifetime positive
idle timeout positive
max uses positive
reset strategy registered
health strategy registered
```

---

# 215. Runtime capability validation

Algunas estrategias podrán requerir capacidades específicas del runtime.

---

# 216. Unsupported pool mode

Deberá fallar claramente.

No degradarse silenciosamente si cambia semántica importante.

---

# 217. Pool sizing

VoltStack no deberá pretender elegir automáticamente un número perfecto.

---

# 218. Defaults

Deberán ser conservadores.

---

# 219. Pool sizing diagnostics

Podrán ofrecer recomendaciones basadas en:

```text
worker count
peak concurrency
database connection limit
observed wait time
observed utilization
```

sin modificar configuración automáticamente.

---

# 220. Resource governance

Database podrá integrarse con:

```text
Quantum resource governance
```

en el futuro.

---

# 221. Global worker budget

Podrá existir un presupuesto superior a varios pools:

```text
Worker Database Connection Budget
```

---

# 222. Example

```text
primary max 20
analytics max 10
audit max 10
```

pero:

```text
global worker budget = 25
```

requeriría coordinación avanzada.

---

# 223. v1 boundary

No es necesario implementar global pool arbitration en v1.

La arquitectura deberá permitirlo posteriormente.

---

# 224. Pool and fork/process model

Los recursos abiertos antes de un process fork no deberán asumirse seguros después del fork.

---

# 225. Runtime adapter responsibility

Si un runtime utiliza fork:

```text
pre-fork resources
→ close/invalidate

post-fork worker
→ own pool
```

---

# 226. Pool serialization

Un Pool no deberá serializarse con recursos activos.

---

# 227. Container lifetime

Recomendación:

```text
ConnectionPoolManager
→ worker/application scope

ConnectionPool
→ worker/application scope

ConnectionLease
→ execution/operation scope

PhysicalConnection
→ pool-owned resource
```

---

# 228. Lifetime rule

Un singleton no deberá capturar:

```text
ConnectionLease
```

como dependencia permanente.

---

# 229. Pool ownership

El Pool posee el recurso cuando:

```text
IDLE
```

El Lease posee temporalmente el derecho exclusivo de uso cuando:

```text
LEASED
```

---

# 230. Ownership transfer

```text
Pool
→ Lease
→ Pool
```

deberá ser explícito.

---

# 231. Lease cannot outlive Pool

Si ocurre shutdown mientras existe un lease:

```text
pool remains in draining state
```

hasta devolución o deadline.

---

# 232. Forced shutdown

Después del deadline:

```text
active leases
→ invalidated/closed
```

según capabilities.

---

# 233. Consumer behavior after invalidation

Intentar usar un lease invalidado deberá producir error claro.

---

# 234. Pool maintenance

Podrá existir:

```text
ConnectionPoolMaintainer
```

para:

```text
idle eviction
health checks
lifetime retirement
dynamic pool eviction
```

---

# 235. Maintenance scheduling

Podrá ejecutarse:

```text
periodically
on acquire
on release
runtime hook
```

dependiendo del runtime.

---

# 236. Avoid mandatory background thread

El core no deberá asumir disponibilidad de un thread/background loop.

---

# 237. FrankenPHP maintenance

El adapter podrá utilizar mecanismos adecuados de worker/runtime sin contaminar Pool core.

---

# 238. Opportunistic maintenance

Una estrategia inicial puede ejecutar mantenimiento ligero durante:

```text
acquire/release
```

para evitar infraestructura background obligatoria.

---

# 239. Maintenance budget

El trabajo de mantenimiento deberá limitarse para no introducir latencia impredecible.

---

# 240. Pool warming and maintenance separation

```text
warming
≠
maintenance
```

---

# 241. Resource reset and health check separation

```text
reset
=
restore known state

health check
=
determine whether resource works
```

Ambos conceptos deberán permanecer separados.

---

# 242. Reset success does not always imply health

Un reset puede completar y aun así existir una conexión muerta detectada posteriormente.

---

# 243. Health success does not imply clean session

Una conexión puede responder a ping pero conservar:

```text
wrong role
wrong schema
open transaction
```

Por eso health y reset no son equivalentes.

---

# 244. Connection initialization

Una conexión recién creada deberá pasar por:

```text
native connect
→ security verification
→ platform discovery if needed
→ session initialization
→ health/readiness
→ available
```

---

# 245. Initializer

Podrá existir:

```text
PhysicalConnectionInitializer
```

---

# 246. Initializer vs Resetter

```text
Initializer
=
prepare new resource

Resetter
=
restore reused resource
```

---

# 247. Platform discovery

El primer recurso de una generación puede descubrir:

```text
server version
capabilities
```

y publicar un snapshot seguro.

---

# 248. Discovery concurrency

Si varias conexiones se crean simultáneamente, deberá evitarse corrupción/races en el capability snapshot.

---

# 249. Capability mismatch

Si un recurso descubre una plataforma incompatible con la definición:

```text
close resource
→ mark initialization failure
```

---

# 250. Connection validation after server restart

Una conexión idle puede quedar inválida después de reinicio del servidor.

Health validation deberá detectarlo.

---

# 251. Stale connection

Estado conceptual:

```text
STALE
```

puede representarse como metadata/health reason sin necesidad de añadir otro estado principal.

---

# 252. Network partitions

Una conexión puede parecer viva hasta el siguiente uso.

El Pool no puede garantizar detección perfecta previa.

---

# 253. Query failure feedback

Execution deberá poder informar:

```text
this lease/resource is broken
```

al Pool.

---

# 254. Failure feedback API

Podrá existir:

```php
$lease->markBroken($failure);
```

antes de release/discard.

---

# 255. Failure classifier

La decisión de si un error invalida la conexión deberá apoyarse en:

```text
Driver Failure Classification
```

---

# 256. SQL error does not imply broken connection

Ejemplo:

```text
unique constraint violation
```

no debería destruir la conexión.

---

# 257. Network/protocol failure

En cambio:

```text
server has gone away
connection reset
protocol desynchronization
```

puede requerir discard.

---

# 258. Transaction state failure

Puede requerir rollback/reset antes de reutilización.

---

# 259. Pool error boundaries

```text
SQL semantic failure
→ Execution

Connection failure
→ Connection/Driver

Resource lifecycle failure
→ Pool

Retry/failover decision
→ Resilience
```

---

# 260. Testing architecture

El Pool deberá contar con tests específicos para:

```text
acquisition
release
capacity
timeouts
waiters
reset
health
retirement
shutdown
transactions
streaming
configuration generations
concurrency
persistent workers
```

---

# 261. Basic acquisition test

```text
pool max = 2

acquire A
acquire B

assert:
leased = 2
```

---

# 262. Capacity test

```text
pool max = 2

acquire A
acquire B
attempt acquire C

assert:
C waits or times out
```

---

# 263. Release test

```text
acquire A
release A

assert:
A resource becomes reusable
```

si reset tiene éxito.

---

# 264. Reset failure test

```text
acquire A
release A
reset fails

assert:
resource closed
not idle
```

---

# 265. Broken resource test

```text
acquire A
mark broken
release

assert:
resource closed
```

---

# 266. Idle timeout test

```text
idle > configured timeout

maintenance

assert:
resource retired
```

---

# 267. Max lifetime test

```text
resource older than max lifetime
currently leased

assert:
not closed during operation

release

assert:
retired
```

---

# 268. Transaction pinning test

```text
begin transaction
acquire A

execute multiple operations

assert:
all use A
```

---

# 269. Streaming test

```text
open streaming result
assert lease held

close stream
assert lease released
```

---

# 270. Scope leak test

```text
Request A acquires lease
forgets release

Request A ends

assert lifecycle recovers/discards lease
```

---

# 271. Persistent request isolation test

```text
Request A:
SET session state

release

Request B:
acquire same physical resource

assert:
baseline state restored
```

---

# 272. Concurrency capacity test

```text
100 concurrent acquires
max = 10

assert:
physical resources never exceed 10
```

---

# 273. Waiter timeout test

```text
max = 1
A holds lease
B waits
B timeout reached

assert:
B fails
waiter removed
```

---

# 274. Cancellation test

```text
B waits
B operation cancelled

assert:
waiter removed
future released resource not assigned to cancelled B
```

---

# 275. Configuration generation test

```text
Pool generation 1 has resources

publish generation 2

assert:
generation 1 resources retire
generation 2 resources created
```

---

# 276. Credential rotation test

```text
old credential generation
→ existing resource

rotate credentials

new acquisition
→ new generation

old release
→ retire
```

---

# 277. Shutdown test

```text
Pool running
A leased
B idle

shutdown

assert:
B closes
A allowed to finish within deadline
new acquire rejected
```

---

# 278. Architecture tests

Deberán impedir:

```text
Pool → ORM
Pool → QueryBuilder
Pool → EntityManager
Pool → Migration
Pool → HTTP
```

---

# 279. Suggested namespaces

```text
VoltStack\Quantum\Database\Connection\Pool
│
├── Contract
│   ├── ConnectionPoolInterface.php
│   ├── ConnectionPoolManagerInterface.php
│   ├── ConnectionResourceProviderInterface.php
│   ├── PhysicalConnectionFactoryInterface.php
│   ├── ConnectionResetterInterface.php
│   ├── ConnectionHealthValidatorInterface.php
│   └── ConnectionRetirementPolicyInterface.php
│
├── Manager
│   └── ConnectionPoolManager.php
│
├── Pool
│   ├── ConnectionPool.php
│   ├── ConnectionPoolIdentity.php
│   ├── ConnectionPoolKey.php
│   ├── ConnectionPoolState.php
│   └── ConnectionPoolStatistics.php
│
├── Resource
│   ├── PooledConnectionResource.php
│   ├── PhysicalConnectionMetadata.php
│   ├── PhysicalConnectionState.php
│   └── PhysicalConnectionHealth.php
│
├── Lease
│   ├── ConnectionLease.php
│   ├── ConnectionLeaseTracker.php
│   └── ConnectionLeaseState.php
│
├── Acquisition
│   ├── ConnectionAcquisitionContext.php
│   ├── ConnectionAcquisitionWaiter.php
│   ├── ConnectionAcquisitionQueue.php
│   └── ConnectionAcquisitionPolicy.php
│
├── Reset
│   ├── ConnectionResetter.php
│   ├── ConnectionResetContext.php
│   └── ConnectionResetResult.php
│
├── Health
│   ├── ConnectionHealthValidator.php
│   ├── ConnectionHealthStrategy.php
│   └── ConnectionHealthResult.php
│
├── Retirement
│   ├── ConnectionRetirementPolicy.php
│   └── ConnectionRetirementReason.php
│
├── Maintenance
│   └── ConnectionPoolMaintainer.php
│
├── Provider
│   ├── DirectConnectionResourceProvider.php
│   ├── InternalPoolResourceProvider.php
│   └── ExternalPoolResourceProvider.php
│
├── Diagnostics
│   ├── ConnectionPoolDiagnostics.php
│   └── ConnectionPoolSnapshot.php
│
└── Exception
    ├── ConnectionPoolException.php
    ├── ConnectionPoolClosedException.php
    ├── ConnectionPoolDrainingException.php
    ├── ConnectionAcquisitionTimeoutException.php
    ├── ConnectionPoolExhaustedException.php
    ├── ConnectionResetException.php
    └── ConnectionLeaseException.php
```

---

# 280. Component dependency model

```text
ConnectionManager
       │
       ▼
Logical Connection
       │
       ▼
ConnectionResourceProvider
       │
       ▼
ConnectionPoolManager
       │
       ▼
ConnectionPool
       │
       ├── ResourceFactory
       │       │
       │       ▼
       │     Driver
       │
       ├── HealthValidator
       ├── Resetter
       ├── RetirementPolicy
       └── Lease
```

---

# 281. Dependency rules

El Pool podrá depender de:

```text
ConnectionDefinition
Driver contracts
PhysicalConnection contracts
Credential resolution contracts
Runtime synchronization abstractions
Telemetry ports
Clock abstractions
```

No deberá depender de:

```text
ORM
EntityManager
UnitOfWork
QueryBuilder
SchemaBuilder
MigrationPlanner
Controller
HTTP Request
```

---

# 282. Architectural invariants

## DB-POOL-001

El Pool administra recursos físicos, no conexiones lógicas.

## DB-POOL-002

`ConnectionManager` y `ConnectionPoolManager` permanecerán separados.

## DB-POOL-003

Toda conexión física reutilizable tendrá ownership explícito.

## DB-POOL-004

Un recurso no podrá estar `IDLE` y `LEASED` simultáneamente.

## DB-POOL-005

Toda adquisición producirá un Lease.

## DB-POOL-006

Los leases serán exclusivos salvo que un futuro Driver declare explícitamente otra semántica segura.

## DB-POOL-007

La capacidad máxima será respetada bajo concurrencia.

## DB-POOL-008

Los recursos `CREATING` contarán para capacity reservation.

## DB-POOL-009

El Pool aplicará backpressure en vez de crecer ilimitadamente.

## DB-POOL-010

Acquire timeout y connect timeout serán conceptos distintos.

## DB-POOL-011

Una conexión broken no regresará a idle.

## DB-POOL-012

Una conexión tainted no será reutilizada sin recuperación demostrablemente segura.

## DB-POOL-013

Reset y health check serán operaciones distintas.

## DB-POOL-014

Una conexión deberá regresar a un baseline conocido antes de reutilización.

## DB-POOL-015

Reset failure causará retiro/discard.

## DB-POOL-016

Una transaction mantendrá afinidad al mismo lease físico.

## DB-POOL-017

Un streaming cursor mantendrá el lease mientras necesite la sesión.

## DB-POOL-018

El lifecycle deberá recuperar leases abandonados al terminar el execution scope.

## DB-POOL-019

El Pool no dependerá de destructores PHP como único mecanismo de cleanup.

## DB-POOL-020

Configuraciones incompatibles no compartirán recursos físicos.

## DB-POOL-021

Los secretos no formarán parte visible del PoolKey.

## DB-POOL-022

Cambios de configuración podrán retirar recursos de generaciones anteriores.

## DB-POOL-023

Credential rotation no requerirá reutilizar recursos autenticados con credenciales obsoletas.

## DB-POOL-024

Pool shutdown impedirá nuevas adquisiciones.

## DB-POOL-025

Pool shutdown deberá soportar draining.

## DB-POOL-026

El Pool no decidirá failover topology.

## DB-POOL-027

El Pool no reintentará SQL application operations.

## DB-POOL-028

El Pool no conocerá ORM.

## DB-POOL-029

El Pool no conocerá Query AST.

## DB-POOL-030

El Pool será seguro bajo workers persistentes.

## DB-POOL-031

El Pool será compatible con aislamiento concurrente.

## DB-POOL-032

Los pools dinámicos deberán poder ser bounded/evictable.

## DB-POOL-033

Database-per-tenant no producirá obligatoriamente pools globales permanentes.

## DB-POOL-034

Internal, external y disabled pooling compartirán el mismo Resource Provider boundary.

## DB-POOL-035

Diagnostics no abrirán conexiones de forma implícita.

## DB-POOL-036

Health checks activos serán explícitos o gobernados por policy.

## DB-POOL-037

Pool metrics evitarán cardinalidad dinámica descontrolada.

## DB-POOL-038

Un SQL error ordinario no marcará automáticamente la conexión como broken.

## DB-POOL-039

La clasificación de errores de conexión pertenecerá al Driver/Resilience boundary.

## DB-POOL-040

El Pool no dependerá de un runtime PHP específico.

---

# 283. Anti-pattern — Pool as array of PDOs

```php
final class Pool
{
    private array $connections = [];
}
```

sin:

```text
lease ownership
capacity accounting
reset
health
retirement
```

no constituye un Pool seguro.

---

# 284. Anti-pattern — get and forget

```php
$pdo = $pool->get();
```

sin ownership explícito.

**Evitar.**

Preferir:

```php
$lease = $pool->acquire();
```

---

# 285. Anti-pattern — returning dirty connection

```text
Request A
→ transaction/session mutation
→ release directly to idle
→ Request B
```

**Prohibido.**

---

# 286. Anti-pattern — health check equals reset

```text
SELECT 1 succeeded
→ connection considered clean
```

Incorrecto.

Una sesión puede estar viva y contaminada.

---

# 287. Anti-pattern — unbounded pool

```text
if no idle:
    create new
```

sin `max`.

**Prohibido.**

---

# 288. Anti-pattern — global tenant pools forever

```text
tenant ID
→ permanent pool
```

sin eviction.

**Prohibido para arquitectura dinámica.**

---

# 289. Anti-pattern — release during transaction

```text
BEGIN on A
release A
acquire B
COMMIT on B
```

**Arquitectónicamente inválido.**

---

# 290. Anti-pattern — release while streaming

```text
open cursor on A
release A
continue cursor
```

**Inválido.**

---

# 291. Anti-pattern — closing active resource because max lifetime elapsed

El retiro deberá ocurrir en un boundary seguro.

---

# 292. Anti-pattern — pool decides replica

```text
Pool primary failed
→ Pool chooses replica
```

**Prohibido.**

Eso pertenece a Topology/Resilience.

---

# 293. Anti-pattern — static current connection

```php
Pool::$current = $connection;
```

**Prohibido.**

---

# 294. Anti-pattern — worker-specific logic in core

```php
if ($runtime === 'frankenphp') {
    // pool logic
}
```

dentro del Pool core.

**Prohibido.**

---

# 295. Anti-pattern — container lookup per acquire

```text
acquire
→ service container
→ find driver
→ find resetter
→ find config
```

en cada hot path.

Las dependencias deberán estar precompuestas.

---

# 296. Performance requirements

El hot path ideal:

```text
acquire
→ select idle
→ cheap validation decision
→ mark leased
→ return lease
```

deberá requerir mínima:

```text
allocation
reflection
configuration parsing
container lookup
locking
```

---

# 297. Release performance

El release podrá ser más costoso si requiere reset, pero deberá minimizar round trips innecesarios.

---

# 298. Reset optimization

Cuando el framework pueda demostrar que una sesión no cambió, ciertas operaciones de reset podrán omitirse de forma segura.

---

# 299. Dirty-state tracking

Podrá existir:

```text
ConnectionSessionStateTracker
```

para registrar cambios conocidos.

---

# 300. Conservative fallback

Si no se sabe si hubo mutación:

```text
strict reset
```

o discard.

Nunca asumir limpieza.

---

# 301. Driver-assisted reset

Drivers podrán proporcionar operaciones eficientes.

Ejemplo conceptual:

```text
reset session
```

si la plataforma ofrece una primitiva nativa segura.

---

# 302. Platform-specific reset

La estrategia podrá diferir para:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

sin condicionales dispersos en Pool core.

---

# 303. SQLite considerations

SQLite puede no necesitar pooling tradicional de la misma forma que un servidor remoto.

---

# 304. Pool capability

El sistema deberá poder determinar:

```text
pooling supported
pooling useful
pooling restricted
```

mediante capabilities/policy.

---

# 305. SQLite memory warning

Especial cuidado con:

```text
:memory:
```

porque distintas conexiones físicas pueden representar bases de datos diferentes.

---

# 306. Therefore

Una configuración SQLite memory deberá poder exigir:

```text
single physical connection
```

o una estrategia compatible con la semántica deseada.

---

# 307. Platform-specific semantics matter

El Pool no deberá asumir que:

```text
more connections
=
same logical database state
```

para todos los targets.

---

# 308. MySQL/MariaDB considerations

Session state como:

```text
sql_mode
time_zone
character_set
temporary tables
user variables
```

deberá contemplarse en reset.

---

# 309. PostgreSQL considerations

Session state como:

```text
search_path
role
timezone
prepared state
temporary objects
transaction failure state
```

deberá contemplarse.

---

# 310. Platform abstraction

Estos detalles pertenecerán a:

```text
Platform-specific ConnectionResetStrategy
```

no al Pool genérico.

---

# 311. Pool modes recommendation

Para VoltStack:

```text
Traditional request runtime
→ pooling optional

FrankenPHP persistent worker
→ internal/external pooling recommended where appropriate

RoadRunner
→ runtime adapter compatible

OpenSwoole
→ concurrency-aware pool implementation required
```

---

# 312. No forced pooling

Database deberá funcionar correctamente con:

```text
pooling disabled
```

---

# 313. Direct provider flow

```text
Acquire
  │
  ▼
Driver connect
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
Close
```

---

# 314. Internal pool flow

```text
Acquire
  │
  ▼
Idle/Create
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
  ▼
Idle
```

---

# 315. External pool flow

```text
VoltStack
   │
   ▼
ExternalPoolResourceProvider
   │
   ▼
Proxy / Runtime Pool
   │
   ▼
Database
```

---

# 316. Reference lifecycle

```text
Application/Worker Boot
        │
        ▼
PoolManager Ready
        │
        ▼
Pools Lazy
        │
        ▼
Execution Scope Begins
        │
        ▼
Acquire Lease
        │
        ▼
Execute Operations
        │
        ▼
Release Lease
        │
        ▼
Reset
        │
        ├── Reusable → Idle
        └── Unsafe   → Close
        │
        ▼
Execution Scope Ends
        │
        ▼
Leak Cleanup
        │
        ▼
Next Execution Scope
```

---

# 317. Final architecture

```text
                     ConnectionManager
                            │
                            ▼
                    Logical Connection
                            │
                            ▼
                ConnectionResourceProvider
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
          Direct         Internal        External
          Provider        Pool           Provider
                            │
                            ▼
                    ConnectionPool
                            │
          ┌─────────────────┼──────────────────┐
          │                 │                  │
          ▼                 ▼                  ▼
     Idle Resources    Wait Queue      Resource Factory
          │                                    │
          ▼                                    ▼
       Acquire                               Driver
          │                                    │
          └──────────────┬─────────────────────┘
                         ▼
                  ConnectionLease
                         │
                         ▼
                PhysicalConnection
                         │
                         ▼
                    Database
```

Release:

```text
ConnectionLease
      │
      ▼
Close outstanding state
      │
      ▼
Reset
      │
      ▼
Health / Retirement
      │
      ├── reusable ──► Idle
      │
      └── unsafe ────► Close
```

---

# 318. Architectural equation

```text
Safe Pooling
=
Bounded Capacity
+
Explicit Ownership
+
Lease Lifecycle
+
State Reset
+
Health Validation
+
Resource Retirement
+
Scope Cleanup
+
Concurrency Safety
```

---

# 319. Criterios de aceptación

La implementación será considerada correcta cuando:

```text
pooling can be completely disabled
logical connections remain separate from physical resources
all physical resource use has explicit lease ownership
max capacity survives concurrent acquisition
acquisition can time out
cancelled waiters cannot receive resources
broken resources never return to idle
reset failures retire resources
orphan transactions are cleaned
session state cannot leak across requests
streaming results retain their lease
transactions retain their lease
old configuration generations retire safely
credential rotation can retire old resources
shutdown drains safely
dynamic pools can be evicted
tenant pools cannot grow unbounded by design
FrankenPHP workers can reuse resources safely
RoadRunner can use the same abstractions
OpenSwoole can implement concurrency-safe acquisition
Query Engine does not know pool internals
ORM does not know pool internals
```

---

# 320. Principio final

> Una conexión física reutilizada sólo es una optimización si VoltStack puede demostrar que su ownership y su estado son seguros.

Por tanto:

```text
Reuse
without
Reset + Ownership + Isolation
=
Bug
```

mientras:

```text
Lease
+
Reset
+
Validation
+
Retirement
+
Scope Isolation
=
Safe Reuse
```

---

# 321. Conclusión

El `Connection Pooling System` de VoltStack no será simplemente una colección de conexiones abiertas.

Será un sistema explícito de administración de recursos:

```text
Physical Connection
        │
        ▼
Ownership
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
        ▼
Validation
        │
        ▼
Reuse or Retirement
```

Esta arquitectura es especialmente importante porque VoltStack tendrá a FrankenPHP como runtime de referencia.

En ese entorno:

```text
Worker lifetime
>>
Request lifetime
```

y por tanto:

```text
Physical connection lifetime
```

puede superar ampliamente la vida de una petición.

El Pool deberá garantizar que esa optimización nunca convierta una conexión reutilizada en un canal accidental para transferir:

```text
transactions
session variables
tenant state
temporary state
cursors
roles
schema selection
```

entre operaciones independientes.

La arquitectura resultante mantiene la separación:

```text
ConnectionManager
=
logical resolution

ConnectionPool
=
physical resource reuse

ConnectionLease
=
temporary ownership

Driver
=
native communication

DatabaseLifecycle
=
scope cleanup
```

permitiendo que VoltStack evolucione desde un uso tradicional de conexiones hasta aplicaciones PHP persistentes y altamente concurrentes sin rediseñar el Query Engine ni el ORM.

---

# 322. Siguiente documento

El siguiente documento será:

```text
15_DATABASE_CONNECTION_LIFECYCLE_SYSTEM.md
```

y deberá profundizar específicamente en:

```text
logical connection lifecycle
physical connection lifecycle
connection states
connection acquisition lifecycle
lease lifecycle
initialization
session setup
connection validation
connection reset
connection release
connection retirement
connection invalidation
broken and tainted states
transaction interaction
cursor and streaming interaction
execution scope ownership
request termination
worker lifecycle
shutdown
failure cleanup
scope leak detection
persistent runtime isolation
FrankenPHP lifecycle
RoadRunner lifecycle
OpenSwoole lifecycle
CLI and queue workers
connection lifecycle events
telemetry
diagnostics
testing
```

La relación principal será:

```text
Connection Definition
        │
        ▼
Logical Connection
        │
        ▼
Acquire
        │
        ▼
ConnectionLease
        │
        ▼
Physical Connection
        │
        ▼
Initialize / Validate
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

El documento `15` deberá establecer el **estado y lifecycle completo de una conexión**, mientras que este documento `14` queda concentrado específicamente en la administración y reutilización del conjunto de recursos físicos.