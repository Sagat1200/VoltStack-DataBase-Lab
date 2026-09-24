# 258_DATABASE_FRANKENPHP_INTEGRATION_SYSTEM.md

# VoltStack Quantum Database
## Database FrankenPHP Integration System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 258 — Database FrankenPHP Integration System  
**Bloque:** 25 — Persistent Runtime  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `257_DATABASE_WORKER_LIFECYCLE_SYSTEM.md`  
**Siguiente documento:** `259_DATABASE_ROADRUNNER_INTEGRATION_SYSTEM.md`

---

# 1. Propósito

Este documento define la integración oficial entre:

```text
VoltStack/Quantum/Database
```

y:

```text
FrankenPHP
```

FrankenPHP será considerado el **runtime HTTP predeterminado de VoltStack**, pero la arquitectura Database continuará siendo independiente del runtime.

La regla central será:

> **FrankenPHP podrá mantener procesos, workers, conexiones e infraestructura Database durante múltiples requests, pero ningún estado mutable perteneciente a un request podrá convertirse implícitamente en estado del siguiente request.**

Formalmente:

```text
PersistentInfrastructure
≠
PersistentRequestState
```

y:

```text
ReusableConnection
≠
ReusableTransaction
≠
ReusableEntityManager
≠
ReusableIdentityMap
≠
ReusableUnitOfWork
```

---

# 2. Objetivo arquitectónico

La integración deberá permitir:

```text
FrankenPHP
    │
    ▼
VoltStack Runtime
    │
    ▼
Database Runtime Adapter
    │
    ├── persistent infrastructure
    │
    ├── connection reuse
    │
    ├── compiled metadata
    │
    └── bounded caches
    │
    ├── Request A
    │    └── Database Scope A
    │
    ├── Request B
    │    └── Database Scope B
    │
    └── Request C
         └── Database Scope C
```

sin contaminación entre scopes.

---

# 3. Contexto

En un modelo PHP clásico:

```text
Process Start
     ↓
Framework Boot
     ↓
Request
     ↓
Database Operations
     ↓
Response
     ↓
Process End
```

el sistema operativo y PHP destruyen naturalmente la mayor parte del estado.

Con un runtime persistente:

```text
Worker Boot
    ↓
Request A
    ↓
Reset
    ↓
Request B
    ↓
Reset
    ↓
Request C
    ↓
...
```

la destrucción automática del proceso deja de ser la principal barrera de aislamiento.

VoltStack deberá crear esa barrera explícitamente.

---

# 4. Principio de integración

FrankenPHP será:

```text
Runtime Provider
```

y no:

```text
Database Architecture Owner
```

Por tanto:

```text
Database Core
        ↑
Database Runtime Contracts
        ↑
FrankenPHP Adapter
        ↑
FrankenPHP Runtime
```

---

# 5. Dependency direction

Permitido:

```text
FrankenPHP Adapter
        ↓
Database Runtime Contracts
```

Prohibido:

```text
Database ORM
        ↓
FrankenPHP
```

o:

```text
Query Builder
        ↓
FrankenPHP
```

---

# 6. Objetivos

La integración deberá resolver:

1. runtime detection;
2. worker boot;
3. framework boot;
4. request admission;
5. request scope creation;
6. DatabaseContext binding;
7. EntityManager lifecycle;
8. IdentityMap lifecycle;
9. UnitOfWork lifecycle;
10. TransactionContext lifecycle;
11. connection pooling;
12. physical connection reuse;
13. request finalization;
14. state reset;
15. isolation verification;
16. request cancellation;
17. worker health;
18. worker taint;
19. worker drain;
20. worker recycling;
21. graceful shutdown;
22. configuration generations;
23. credential rotation;
24. topology changes;
25. concurrency;
26. Fibers;
27. telemetry;
28. diagnostics;
29. development mode;
30. production defaults.

---

# 7. No objetivos

Este sistema no redefinirá:

```text
FrankenPHP server internals
Caddy configuration model
HTTP routing
HTTP middleware semantics
Query AST
SQL compilation
ORM mapping
Transaction semantics
Connection protocol
```

Sólo definirá la frontera de integración.

---

# 8. Dos modelos de ejecución

VoltStack deberá contemplar al menos conceptualmente:

```text
FrankenPHP
├── classic request lifecycle
└── persistent worker lifecycle
```

La semántica Database deberá ser consistente en ambos.

---

# 9. Classic lifecycle

Conceptualmente:

```text
BOOT
 ↓
REQUEST
 ↓
DATABASE SCOPE
 ↓
FINALIZE
 ↓
SHUTDOWN
```

---

# 10. Persistent lifecycle

```text
WORKER BOOT
    ↓
READY
    ↓
REQUEST A
    ↓
FINALIZE A
    ↓
RESET A
    ↓
VERIFY A
    ↓
REQUEST B
    ↓
...
```

---

# 11. Mismo Database contract

La aplicación no deberá cambiar:

```php
$user = User::find($id);
```

según esté ejecutándose en:

```text
classic PHP
FrankenPHP persistent worker
RoadRunner
OpenSwoole
```

---

# 12. Runtime adapter

Se propone:

```php
final class FrankenPhpDatabaseRuntimeAdapter
    implements DatabaseRuntimeAdapter
{
    public function runtime(): DatabaseRuntime
    {
        return DatabaseRuntime::FRANKENPHP;
    }
}
```

---

# 13. Responsabilidad del adapter

El adapter traduce eventos del runtime hacia eventos del lifecycle Database.

Conceptualmente:

```text
FrankenPHP Event
      ↓
Runtime Adapter
      ↓
Database Lifecycle Event
```

---

# 14. Eventos fundamentales

El adapter deberá representar:

```text
worker boot
request accepted
request started
request completed
request failed
request cancelled
worker drain
worker shutdown
```

---

# 15. FrankenPHP ≠ DatabaseContext

FrankenPHP podrá identificar una request.

Pero:

```text
HTTP Request Context
≠
DatabaseContext
```

El segundo será construido por VoltStack.

---

# 16. Worker boot

Pipeline:

```text
FrankenPHP Worker Start
          ↓
VoltStack Bootstrap
          ↓
Database Worker Bootstrap
          ↓
Load Database Config
          ↓
Initialize Drivers
          ↓
Initialize Platforms
          ↓
Compile/Load Metadata
          ↓
Initialize Connection Infrastructure
          ↓
Initialize Runtime Adapter
          ↓
Register Reset Participants
          ↓
Register Isolation Verifiers
          ↓
READY
```

---

# 17. Infraestructura persistente

Podrá mantenerse:

```text
DriverRegistry
DialectRegistry
PlatformRegistry
TypeRegistry
compiled metadata
metadata cache
compiled query cache
connection pools
telemetry instruments
immutable configuration generations
```

---

# 18. Estado no persistente

Nunca deberá compartirse entre requests:

```text
DatabaseContext
EntityManager
IdentityMap
UnitOfWork
TransactionContext
ConnectionLease
QueryContext
HydrationSession
RelationshipLoadSession
SecurityContext
TenantContext
ResultCursor
LazyIterator
ChunkTraversalState
```

---

# 19. Request start

Cada request creará:

```text
FrankenPHP Request
       ↓
ExecutionId
       ↓
ExecutionScope
       ↓
DatabaseContext
```

---

# 20. Request scope

Conceptualmente:

```php
$scope = $scopeFactory->create(
    kind: DatabaseExecutionKind::HTTP_REQUEST,
    runtime: DatabaseRuntime::FRANKENPHP,
);
```

---

# 21. Request identity

Se deberá distinguir:

```text
HTTP request ID
Database execution ID
trace ID
```

Pueden correlacionarse, pero no son necesariamente el mismo identificador.

---

# 22. DatabaseContext

El request tendrá un:

```text
DatabaseContext
```

propio.

Contendrá referencias/contexto como:

```text
logical database
tenant
read/write intent
consistency requirements
transaction context
security context
execution deadline
cancellation token
resource budget
```

---

# 23. Context binding

El runtime adapter deberá permitir:

```text
current FrankenPHP execution
            ↓
DatabaseExecutionScopeResolver
            ↓
current DatabaseContext
```

---

# 24. Prohibición de global current context

Nunca:

```php
Database::$currentContext = $context;
```

como estado global mutable.

---

# 25. Scope locality

El mecanismo real podrá usar una abstracción compatible con:

```text
execution-local storage
Fiber-local storage
runtime request context
scoped container
```

---

# 26. Fiber awareness

Aunque un worker pertenezca a un proceso:

```text
Process
≠
Execution Context
```

Si existen múltiples ejecuciones/Fibers:

```text
Fiber A → Scope A
Fiber B → Scope B
```

---

# 27. Context switching

Cuando el runtime cambia:

```text
A → B → A
```

Database deberá resolver:

```text
Context A → Context B → Context A
```

correctamente.

---

# 28. Request-scoped container

VoltStack podrá utilizar un container jerárquico:

```text
Application Container
       │
       ├── Worker Scope
       │
       ├── Request Scope A
       │
       └── Request Scope B
```

---

# 29. Worker services

Ejemplos:

```text
ConnectionManager
DriverRegistry
PlatformRegistry
MetadataFactory
QueryCompilerRegistry
TelemetryProvider
```

siempre que su mutable state sea seguro.

---

# 30. Request services

Ejemplos:

```text
DatabaseContext
EntityManager
IdentityMap
UnitOfWork
TransactionContext
ExecutionResourceRegistry
```

---

# 31. EntityManager

Por default:

```text
1 Database Execution Scope
→ 1 primary EntityManager context
```

aunque puedan existir EntityManagers adicionales explícitos.

---

# 32. EntityManager reuse

Prohibido:

```text
Request A EntityManager
        ↓
Request B EntityManager
```

como misma instancia mutable.

---

# 33. IdentityMap

Siempre nueva por execution scope.

---

# 34. Example

Request A:

```text
User#10
→ PHP Object A
```

Request B:

```text
User#10
→ PHP Object B
```

No deberá reutilizarse `Object A`.

---

# 35. UnitOfWork

Cada request tendrá su propia:

```text
UnitOfWork
```

---

# 36. Pending changes

Cambios no flushed del request A deberán desaparecer durante finalization.

Nunca:

```text
Request A
$entity->name = 'X';

Request B
flush();

→ persists A changes
```

---

# 37. TransactionContext

Cada request tendrá transaction state aislado.

---

# 38. Transaction start

```text
Request A
    ↓
begin
    ↓
Connection Lease
    ↓
TransactionContext A
```

---

# 39. Request end with transaction

Si termina sin resolución:

```text
rollback
```

por default.

---

# 40. Rollback failure

Si rollback no puede verificarse:

```text
connection → discard
transaction → UNKNOWN
execution → failure/taint
```

---

# 41. Unknown commit

Caso:

```text
COMMIT sent
connection lost
```

Resultado:

```text
UNKNOWN
```

Nunca asumir:

```text
rolled back
```

ni:

```text
committed
```

sin evidencia.

---

# 42. Connection architecture

La integración deberá separar:

```text
ConnectionManager
ConnectionPool
PhysicalConnection
ConnectionLease
```

---

# 43. Worker connection pool

El pool podrá sobrevivir múltiples requests.

```text
FrankenPHP Worker
       │
       └── Connection Pool
              ├── C1
              ├── C2
              ├── C3
              └── C4
```

---

# 44. Request lease

Request A:

```text
Pool
 ↓
lease C2
 ↓
Request A
```

Finaliza:

```text
Request A
 ↓
reset C2
 ↓
verify C2
 ↓
Pool
```

Después:

```text
Pool
 ↓
lease C2
 ↓
Request B
```

---

# 45. Connection reuse invariant

```text
Reuse(C)
⇒
Released(C)
∧ Reset(C)
∧ Verified(C)
∧ Healthy(C)
```

---

# 46. Session state

Antes de reutilizar conexión deberán considerarse estados como:

```text
active transaction
savepoints
session variables
temporary settings
database role
schema/search path
timezone
isolation overrides
prepared statement state
temporary tables
locks
driver-specific state
```

según plataforma/capabilities.

---

# 47. ConnectionResetPlan

La plataforma deberá ayudar a construir:

```text
ConnectionResetPlan
```

---

# 48. Reset capability

No se asumirán comandos universales.

Ejemplo:

```text
MySQL
PostgreSQL
SQLite
MariaDB
```

pueden requerir estrategias distintas.

---

# 49. Capability-driven reset

Nunca:

```php
if ($driver === 'pgsql') {
    // hardcoded everywhere
}
```

Preferir:

```php
$platform->connectionResetCapabilities();
```

---

# 50. Failed reset

Si una conexión no puede limpiarse con certeza:

```text
DISCARD
```

---

# 51. Discard over unsafe reuse

Regla:

> **Crear una nueva conexión es preferible a reutilizar una conexión cuyo estado no pueda demostrarse limpio.**

---

# 52. Pool exhaustion

Persistent workers hacen importante limitar:

```text
max connections
max pending acquisitions
acquisition timeout
lease duration
idle lifetime
connection lifetime
```

---

# 53. Pool per worker vs shared

Database core no deberá asumir que todos los workers comparten el mismo pool físico.

Conceptualmente:

```text
Worker 1 → Pool 1
Worker 2 → Pool 2
Worker 3 → Pool 3
```

---

# 54. Total connection budgeting

Si existen:

```text
W workers
```

y:

```text
P max connections per worker
```

el máximo potencial será aproximadamente:

```text
Cmax = W × P
```

antes de considerar otros procesos.

---

# 55. Capacity planning

Por tanto:

```text
FrankenPHP worker count
```

y:

```text
database pool size
```

no podrán configurarse independientemente.

---

# 56. Connection budget

VoltStack deberá permitir:

```text
database.pool.max_connections_per_worker
```

o equivalente.

---

# 57. Global DB capacity

Si la DB soporta:

```text
D
```

conexiones y se reserva:

```text
R
```

para administración/otros servicios:

```text
Usable = D - R
```

Una configuración segura requiere aproximadamente:

```text
W × P ≤ Usable
```

con margen operacional.

---

# 58. Concurrency

La concurrencia real del runtime deberá reflejarse en el pool.

No asumir:

```text
one worker = one active DB request
```

---

# 59. Concurrent lease isolation

```text
Execution A → Connection C1
Execution B → Connection C2
```

o sharing controlado sólo si el driver/protocol lo permite explícitamente.

---

# 60. No concurrent transaction sharing

Dos requests no compartirán:

```text
same TransactionContext
```

---

# 61. Request finalization

Pipeline recomendado:

```text
Application Completed
        ↓
Freeze New DB Operations
        ↓
Cancel Pending Operations
        ↓
Close Streaming Results
        ↓
Close Result Cursors
        ↓
Close Lazy Iterators
        ↓
Finalize Chunk Traversals
        ↓
Resolve Transactions
        ↓
Release Connection Leases
        ↓
Clear UnitOfWork
        ↓
Clear IdentityMap
        ↓
Close EntityManager
        ↓
Clear DatabaseContext
        ↓
Reset Scope Services
        ↓
Verify Isolation
        ↓
Destroy Request Scope
```

---

# 62. Response sent ≠ cleanup complete

FrankenPHP podría haber producido la respuesta mientras todavía existe cleanup interno.

VoltStack deberá distinguir:

```text
ResponseCompletion
```

de:

```text
DatabaseExecutionFinalization
```

---

# 63. New request admission

No deberá reutilizar un scope que todavía esté finalizando.

---

# 64. Cleanup ownership

El runtime adapter será responsable de garantizar que el lifecycle manager reciba la señal de finalización.

---

# 65. Finally semantics

Conceptualmente:

```php
$scope = $lifecycle->begin($request);

try {
    return $application->handle($request);
} finally {
    $lifecycle->finalize($scope);
}
```

---

# 66. Exceptions

Application exception no evitará cleanup.

---

# 67. HTTP 500

Incluso:

```text
500 Internal Server Error
```

deberá ejecutar Database finalization.

---

# 68. Cancellation

Si request es cancelado:

```text
cancel operations
↓
finalize
↓
reset
↓
verify
```

---

# 69. Client disconnect

Client disconnect y Database cancellation serán conceptos distintos.

Policy podrá decidir si cancelar operaciones.

---

# 70. Query cancellation

Cuando sea soportado:

```text
CancellationToken
→ QueryExecutor
→ Driver
```

---

# 71. Write uncertainty

Cancelar una operación write no garantiza que no haya ocurrido.

Preservar:

```text
UNKNOWN
```

cuando corresponda.

---

# 72. ResultCursor

Todo cursor deberá estar registrado en:

```text
ExecutionResourceRegistry
```

---

# 73. StreamingResult

Igual.

---

# 74. LazyCollection

Una LazyCollection activa deberá cerrar su fuente durante finalization.

---

# 75. Chunk processing

Chunk traversal activo deberá:

```text
stop
release resources
preserve valid checkpoint if policy permits
```

---

# 76. Request-local temporary state

Cualquier estado de:

```text
pagination
cursor pagination
chunk
lazy collection
bulk processing
import/export
```

deberá estar asociado al scope correcto.

---

# 77. Import/export

Operaciones largas pueden exceder lifecycle HTTP normal.

VoltStack deberá recomendar:

```text
HTTP request
   ↓
dispatch job
```

para trabajos de larga duración.

---

# 78. Long-running HTTP operations

Si se permiten, deberán respetar:

```text
deadline
memory budget
connection budget
cancellation
```

---

# 79. Worker memory

Persistent workers hacen visible la acumulación de memoria.

---

# 80. ORM memory risk

Aunque request termine:

```text
EntityManager
IdentityMap
UnitOfWork
```

deben quedar inaccesibles desde infraestructura compartida.

---

# 81. Shared reference leak

Ejemplo crítico:

```php
final class BadService
{
    private ?EntityManager $manager = null;
}
```

si `BadService` es worker-shared y conserva el EntityManager de un request.

---

# 82. Scope safety validation

Development mode podrá detectar:

```text
worker-scoped service
→ references request-scoped service
```

---

# 83. Lifetime dependency rule

Permitido:

```text
REQUEST → WORKER
```

Prohibido normalmente:

```text
WORKER → REQUEST
```

como referencia persistente.

---

# 84. Lifetime graph

```text
PROCESS
  ↓
WORKER
  ↓
EXECUTION
  ↓
TRANSACTION
  ↓
OPERATION
```

Un lifetime largo no deberá capturar accidentalmente uno más corto.

---

# 85. Safe provider pattern

Si un worker service necesita acceso contextual:

```php
final class SomeService
{
    public function __construct(
        private DatabaseExecutionScopeResolver $resolver,
    ) {}
}
```

y resuelve el scope actual durante la llamada.

---

# 86. Unsafe provider pattern

No:

```php
final class SomeWorkerService
{
    public function __construct(
        private EntityManager $entityManager,
    ) {}
}
```

si el servicio sobrevive requests.

---

# 87. Container validation

VoltStack Container podrá detectar lifetime inversions durante compilación/boot.

---

# 88. Memory baseline

Después de cada finalization podrá registrarse:

```text
memory_before
memory_peak
memory_after_reset
```

con sampling.

---

# 89. Leak suspicion

Si:

```text
postResetMemory
```

crece sostenidamente fuera de budgets:

```text
worker health → DEGRADED
```

y eventualmente:

```text
recycle requested
```

---

# 90. Worker recycle

FrankenPHP adapter deberá poder traducir una solicitud abstracta:

```text
DatabaseWorkerRecycleRequested
```

al mecanismo soportado por el runtime/orquestador.

---

# 91. Core independence

Database core no ejecutará directamente comandos de servidor.

---

# 92. Recycle triggers

Ejemplos:

```text
max requests
max lifetime
memory threshold
repeated reset failures
shared-state taint
configuration incompatibility
credential policy
operator request
deployment
```

---

# 93. Request count recycling

Configuración conceptual:

```php
'worker' => [
    'max_executions' => 10_000,
];
```

---

# 94. Memory recycling

```php
'worker' => [
    'max_memory' => '512M',
];
```

La sintaxis final dependerá del Config System.

---

# 95. Tainted worker

Cuando:

```text
WorkerState = TAINTED
```

el adapter deberá:

```text
stop accepting new DB executions
↓
request drain
↓
request recycle
```

---

# 96. Resource-local failure

No todo fallo recicla worker.

Ejemplo:

```text
bad connection
→ discard connection
```

---

# 97. Scope-local failure

Ejemplo:

```text
broken EntityManager
→ close request scope
```

---

# 98. Shared failure

Ejemplo:

```text
corrupted shared registry
→ worker taint
```

---

# 99. Health model

FrankenPHP integration expondrá internamente:

```text
DatabaseWorkerHealth
DatabaseWorkerReadiness
DatabaseWorkerLiveness
```

---

# 100. Readiness

Un worker no estará Database-ready cuando:

```text
tainted
draining
critical resources exhausted
required DB authority unavailable
isolation verification failed globally
```

---

# 101. Database outage

No necesariamente implica matar worker.

Puede:

```text
health = DEGRADED/UNHEALTHY
ready = false
```

y esperar recuperación.

---

# 102. Circuit breaker

Integración:

```text
Endpoint Failure
      ↓
Circuit Breaker
      ↓
Routing Eligibility
      ↓
Worker Health
```

sin mezclar responsabilidades.

---

# 103. Read/write routing

Cada request conserva:

```text
read/write intent
sticky state
transaction affinity
consistency requirements
```

en su propio DatabaseContext.

---

# 104. Sticky connection state

No deberá almacenarse como:

```text
static bool $sticky = true;
```

porque contaminaría otros requests.

---

# 105. Sticky state scope

Será:

```text
execution-scoped
```

o dominio explícitamente superior si alguna feature futura lo requiere.

---

# 106. Replica state

La salud de replicas sí puede ser worker-shared porque representa infraestructura.

---

# 107. Distinción

```text
ReplicaHealth
→ worker/shared
```

```text
RequestRequiresWriter
→ execution
```

---

# 108. Multitenancy

Tenant deberá resolverse para cada request.

---

# 109. Tenant resolution

Conceptualmente:

```text
HTTP Request
     ↓
Tenant Resolver
     ↓
TenantContext
     ↓
DatabaseContext
     ↓
Connection Routing
```

---

# 110. Tenant context cleanup

Al finalizar:

```text
TenantContext = destroyed
```

---

# 111. Cross-request tenant contamination

Caso crítico:

```text
Request A → Tenant ACME
Request B → Tenant BETA
```

B nunca deberá recibir:

```text
connection
schema
role
query scope
cache key
IdentityMap
```

configurado para ACME.

---

# 112. Tenant-aware connection reset

Si tenancy usa:

```text
schema
database role
session variable
```

la conexión deberá restaurarse antes de regresar al pool.

---

# 113. Tenant-aware pool partitioning

Algunas estrategias podrán usar pools separados:

```text
Pool(Tenant A)
Pool(Tenant B)
```

pero esto será política del módulo Multitenancy, no requisito del core.

---

# 114. SecurityContext

Cada request tendrá contexto de seguridad propio.

---

# 115. Authorization query scope

No deberá sobrevivir al request.

---

# 116. Privileged DB role

Si temporalmente se eleva:

```text
SET ROLE
```

deberá restaurarse.

---

# 117. Sensitive values

No deberán conservarse accidentalmente en:

```text
shared diagnostics
long-lived event objects
static variables
unbounded telemetry
```

---

# 118. Query events

Query Event System podrá ser worker-shared en infraestructura, pero los eventos deberán llevar contexto correcto.

---

# 119. Event listener isolation

Listeners globales no capturarán:

```text
Request
EntityManager
TenantContext
TransactionContext
```

---

# 120. Telemetry

Cada execution tendrá:

```text
execution.id
runtime = frankenphp
worker.id
worker.generation
```

---

# 121. Query telemetry

Podrá correlacionarse:

```text
HTTP request
→ DB execution
→ transaction
→ query
→ connection
```

---

# 122. Cardinality control

No usar tenant/user/query raw como labels métricos sin control.

---

# 123. Worker metrics

```text
database_frankenphp_active_executions
database_frankenphp_completed_executions_total
database_frankenphp_reset_failures_total
database_frankenphp_isolation_failures_total
database_frankenphp_connection_reuse_total
database_frankenphp_connection_discard_total
database_frankenphp_worker_recycles_total
database_frankenphp_worker_memory_bytes
```

---

# 124. Runtime-neutral metrics

Cuando sea posible preferir:

```text
database_worker_*
```

con runtime como dimensión bounded.

---

# 125. Request finalization metrics

```text
database_execution_finalization_duration
database_execution_reset_duration
database_execution_verification_duration
```

---

# 126. Connection reuse metrics

Medir:

```text
acquired
new
reused
discarded
reset_failed
validation_failed
```

---

# 127. Diagnostics command

VoltStack podrá ofrecer:

```text
php volt database:runtime
```

Salida conceptual:

```text
Database Runtime
────────────────────────────────
Runtime: FrankenPHP
Mode: persistent

Worker
  ID:          worker-04
  Generation:  18
  State:       READY
  Health:      HEALTHY

Executions
  Active:      2
  Completed:   18,431

Connections
  Pools:       3
  Physical:    12
  Leased:      2
  Idle:        10

ORM
  Active EntityManagers: 2
  Active IdentityMaps:    2
  Active UnitOfWorks:     2

Transactions
  Active:      1

Isolation
  Last verification: PASS
```

---

# 128. Development diagnostics

Podrá mostrarse:

```text
Request finalized successfully.

EntityManager: CLOSED
IdentityMap: EMPTY
UnitOfWork: EMPTY
Transactions: 0
Leases: 0
Cursors: 0
TenantContext: CLEARED

Worker reuse: SAFE
```

---

# 129. Isolation failure diagnostic

```text
Database request isolation failure.

Execution:
req-3921

Remaining:
- 1 active ResultCursor
- 1 leased connection

Containment:
- cursor force-closed
- connection discarded

Worker:
HEALTHY
```

---

# 130. Severe failure diagnostic

```text
Database worker isolation failure.

Execution:
req-3921

Shared state contamination detected:
Worker-scoped listener retained request EntityManager.

Worker marked:
TAINTED

New admissions:
DISABLED

Requested action:
DRAIN_AND_RECYCLE
```

---

# 131. Production mode

Production defaults deberán priorizar:

```text
isolation
bounded resources
safe connection reuse
low telemetry overhead
automatic containment
controlled worker recycling
```

---

# 132. Development mode

Podrá habilitar:

```text
strict lifetime validation
resource leak tracing
scope-generation checks
verbose reset diagnostics
post-request isolation assertions
memory drift diagnostics
N+1 telemetry
query profiler
```

---

# 133. Strict mode

Conceptualmente:

```php
'database' => [
    'runtime' => [
        'strict_isolation' => true,
    ],
];
```

---

# 134. Production verification

No significa desactivar verificación.

Debe utilizar verificaciones económicas y deterministas.

---

# 135. Expensive verification

Puede samplearse cuando no comprometa correctness.

---

# 136. Mandatory checks

Nunca deberían omitirse:

```text
active transaction
leased connection ownership
scope generation
known active cursors
known unresolved transaction state
```

---

# 137. Configuration generations

El worker mantendrá:

```text
ConfigGeneration
MetadataGeneration
TopologyGeneration
CredentialGeneration
```

según corresponda.

---

# 138. Request snapshot

Al comenzar request:

```text
DatabaseExecutionEnvironment
```

capturará generaciones necesarias.

---

# 139. Mid-request config change

No deberá cambiar arbitrariamente la semántica del request.

---

# 140. Compatible change

Puede aplicarse a nuevas operaciones sólo si el contrato explícitamente lo permite.

---

# 141. Conservative default

Mantener generación estable durante la ejecución.

---

# 142. New requests

Adoptarán nueva generación.

---

# 143. Incompatible generation

Si infraestructura compartida no puede coexistir:

```text
drain
→ recycle
```

---

# 144. Credential rotation

Proceso:

```text
Credential Generation N
        ↓
Rotation
        ↓
Generation N+1
        ↓
New connections use N+1
        ↓
Old connections retire/drain
```

---

# 145. Forced credential expiration

Si N deja de ser válido inmediatamente:

```text
connections using N
→ retire/discard
```

---

# 146. Active transaction

No deberá migrarse transparentemente a otra conexión.

---

# 147. Topology update

Proceso:

```text
Topology N
   ↓
new topology discovered
   ↓
Topology N+1
   ↓
new routing decisions use N+1
```

---

# 148. Active transaction topology

Permanece pinned a su conexión mientras sea semánticamente válido.

---

# 149. Writer failover

No reanudar transaction automáticamente en nuevo writer como si nada hubiera ocurrido.

---

# 150. Unknown transaction outcome

Preservar UNKNOWN.

---

# 151. Worker shutdown

Pipeline:

```text
FrankenPHP Shutdown Requested
          ↓
Stop New Admissions
          ↓
DRAINING
          ↓
Wait Active Requests
          ↓
Cancel after Deadline
          ↓
Finalize Remaining DB Scopes
          ↓
Close Pools
          ↓
Flush Telemetry
          ↓
STOPPED
```

---

# 152. Shutdown deadline

Configurable.

---

# 153. Pool shutdown

Primero:

```text
reject new leases
```

después:

```text
wait active leases
```

finalmente:

```text
close physical connections
```

---

# 154. Forced shutdown

No deberá bloquear indefinidamente esperando:

```text
hung query
hung connection
broken network
```

---

# 155. Query deadline

Las operaciones deberán respetar el deadline restante del request cuando corresponda.

---

# 156. Finalization grace period

Se reservará tiempo para cleanup.

---

# 157. Deadline architecture

```text
Request Deadline
│
├── Application Budget
└── Finalization Reserve
```

---

# 158. Long transaction warning

Transactions abiertas durante toda una request larga podrán producir warnings.

---

# 159. Streaming responses

Requieren especial cuidado.

```text
HTTP response streaming
```

puede extender el execution scope.

---

# 160. Streaming response ≠ request logically finished

Si el código sigue consultando DB durante streaming:

```text
DatabaseExecutionScope
```

debe permanecer válido.

---

# 161. Early response

Enviar headers/body no implica destruir inmediatamente DatabaseContext si todavía existe código válido en ejecución.

---

# 162. Post-response work

Trabajo ejecutado después de enviar response deberá clasificarse explícitamente.

---

# 163. Post-response DB work

Dos opciones:

```text
same execution scope
```

si forma parte real del request lifecycle,

o:

```text
new background execution
```

si se desacopla.

---

# 164. No ambiguous background scope

No permitir:

```text
response sent
→ request scope reset
→ old callback uses EntityManager
```

---

# 165. Worker-mode optimization

Una vez establecida seguridad, FrankenPHP permite aprovechar:

```text
framework boot reuse
compiled metadata reuse
container compilation reuse
query compiler reuse
connection reuse
bounded caches
```

---

# 166. Optimization priority

Orden:

```text
Correctness
    ↓
Isolation
    ↓
Security
    ↓
Resource Safety
    ↓
Performance
```

---

# 167. Warm worker

VoltStack podrá precargar:

```text
metadata
mapping
dialect capabilities
compiled service graph
common query structures
```

---

# 168. Warm connection

No necesariamente abrir conexiones durante boot.

---

# 169. Preconnect policy

Opcional:

```php
enum FrankenPhpDatabasePreconnectPolicy
{
    case NONE;
    case WRITER;
    case REQUIRED_ENDPOINTS;
    case CUSTOM;
}
```

---

# 170. Preconnect tradeoff

Ventaja:

```text
first-request latency ↓
```

Costo:

```text
idle DB connections ↑
startup dependency ↑
```

---

# 171. Default

Preferible:

```text
lazy connection acquisition
```

salvo despliegues que requieran fail-fast.

---

# 172. Prepared statements

Podrán existir optimizaciones de reuse si driver/plataforma lo soportan.

Pero:

```text
PreparedStatementReuse
≠
ExecutionStateReuse
```

---

# 173. Prepared statement safety

Bindings nunca deberán persistir accidentalmente entre executions.

---

# 174. Temporary tables

Son especialmente peligrosas con connection reuse.

---

# 175. Temporary table policy

Si una conexión crea temporary tables:

```text
reset strategy must remove them
```

o:

```text
connection must be discarded
```

si no puede garantizarse.

---

# 176. Session variables

Mismo principio.

---

# 177. Advisory locks

Deben liberarse o provocar discard según plataforma.

---

# 178. Database roles

Restaurados.

---

# 179. Search path/schema

Restaurado.

---

# 180. Isolation level

Overrides por transaction/request deberán restaurarse.

---

# 181. Timezone/session locale

Restauradas cuando se modifiquen.

---

# 182. Connection cleanliness

Formalmente:

```text
Clean(C)
=
¬ActiveTransaction(C)
∧ ¬ActiveCursor(C)
∧ ¬Leased(C)
∧ SessionStateDefault(C)
∧ SecurityStateDefault(C)
∧ DriverStateReusable(C)
```

---

# 183. Reuse rule

```text
ReturnToPool(C)
⇒
Clean(C)
```

---

# 184. Unknown cleanliness

```text
Cleanliness(C) = UNKNOWN
⇒
Discard(C)
```

---

# 185. Request cleanliness

Formalmente:

```text
Clean(E)
=
NoActiveTransactions(E)
∧ NoConnectionLeases(E)
∧ NoActiveCursors(E)
∧ IdentityMapEmpty(E)
∧ UnitOfWorkEmpty(E)
∧ ContextCleared(E)
∧ NoScopeLeaks(E)
```

---

# 186. Worker reuse

```text
ReuseWorker(W)
⇒
Clean(Eprevious)
∨ FailureContained(Eprevious)
```

y:

```text
SharedWorkerState(W) = VALID
```

---

# 187. FrankenPHP integration manager

Se propone:

```php
interface FrankenPhpDatabaseIntegration
{
    public function bootWorker(): void;

    public function beginRequest(
        FrankenPhpRequestContext $request
    ): DatabaseExecutionScope;

    public function endRequest(
        DatabaseExecutionScope $scope
    ): DatabaseFinalizationReport;

    public function drainWorker(): void;

    public function shutdownWorker(): void;
}
```

---

# 188. Adapter implementation

```php
final class DefaultFrankenPhpDatabaseIntegration
    implements FrankenPhpDatabaseIntegration
{
    public function __construct(
        private DatabaseWorkerLifecycleManager $lifecycle,
        private DatabaseExecutionScopeFactory $scopes,
        private DatabaseExecutionScopeResolver $resolver,
    ) {}
}
```

---

# 189. Runtime bridge

La capa podría residir en:

```text
VoltStack/Quantum/Database/Runtime/FrankenPHP
```

---

# 190. Proposed directory structure

```text
src/Quantum/Database/Runtime/FrankenPHP/
│
├── Contract/
│   ├── FrankenPhpDatabaseIntegration.php
│   ├── FrankenPhpExecutionContext.php
│   └── FrankenPhpWorkerBridge.php
│
├── Adapter/
│   ├── FrankenPhpDatabaseRuntimeAdapter.php
│   └── FrankenPhpExecutionScopeResolver.php
│
├── Lifecycle/
│   ├── FrankenPhpWorkerBootstrap.php
│   ├── FrankenPhpRequestLifecycle.php
│   ├── FrankenPhpRequestFinalizer.php
│   ├── FrankenPhpWorkerDrain.php
│   └── FrankenPhpWorkerShutdown.php
│
├── Context/
│   ├── FrankenPhpDatabaseContextBinder.php
│   └── FrankenPhpExecutionContextStorage.php
│
├── Connection/
│   ├── FrankenPhpConnectionPoolIntegration.php
│   └── FrankenPhpConnectionReusePolicy.php
│
├── Health/
│   ├── FrankenPhpDatabaseHealthContributor.php
│   └── FrankenPhpDatabaseReadinessContributor.php
│
├── Telemetry/
│   └── FrankenPhpDatabaseTelemetryBridge.php
│
├── Diagnostics/
│   └── FrankenPhpDatabaseDiagnostics.php
│
└── Exception/
    ├── FrankenPhpDatabaseIntegrationException.php
    ├── FrankenPhpWorkerBootException.php
    ├── FrankenPhpRequestScopeException.php
    └── FrankenPhpWorkerLifecycleException.php
```

---

# 191. Framework-level integration

Podrá existir adicionalmente:

```text
src/Platform/Runtime/FrankenPHP/
```

para integración general del framework.

Database dependerá sólo de contratos apropiados.

---

# 192. RuntimeManagerServer

La instalación/configuración de FrankenPHP pertenecerá al sistema:

```text
Quantum/RuntimeManagerServer
```

no al Database core.

---

# 193. Separation

```text
RuntimeManagerServer
→ installs/configures FrankenPHP
```

```text
Database FrankenPHP Integration
→ adapts Database lifecycle to runtime
```

---

# 194. No server installation logic

Este módulo no deberá:

```text
download FrankenPHP
install binaries
configure system services
configure Caddy globally
manage OS packages
```

---

# 195. Error hierarchy

```text
FrankenPhpDatabaseIntegrationException
├── FrankenPhpRuntimeDetectionException
├── FrankenPhpWorkerBootException
├── FrankenPhpWorkerStateException
├── FrankenPhpRequestScopeException
├── FrankenPhpContextBindingException
├── FrankenPhpConnectionReuseException
├── FrankenPhpRequestFinalizationException
├── FrankenPhpIsolationException
├── FrankenPhpWorkerTaintedException
├── FrankenPhpWorkerDrainException
└── FrankenPhpWorkerShutdownException
```

---

# 196. Runtime-specific errors

Deberán traducirse a errores Database/runtime apropiados sin filtrar detalles innecesarios al dominio.

---

# 197. Testing

Se requerirán:

```text
unit tests
integration tests
worker-mode tests
connection reuse tests
scope isolation tests
transaction leak tests
tenant contamination tests
Fiber/context tests
memory tests
recycle tests
shutdown tests
fault injection
soak tests
```

---

# 198. Classic vs persistent parity

Misma suite funcional deberá ejecutarse contra:

```text
classic lifecycle
persistent FrankenPHP lifecycle
```

---

# 199. Sequential request test

Ejecutar:

```text
A → reset → B → reset → C
```

verificando ausencia de contaminación.

---

# 200. Concurrent request test

Ejecutar conceptualmente:

```text
A ──────────┐
B ──────┐   │
C ─────────────
```

y verificar scopes independientes.

---

# 201. ORM contamination test

A carga:

```text
User#1
```

B carga:

```text
User#1
```

Las instancias managed serán diferentes entre scopes.

---

# 202. Dirty state test

A modifica entidad sin flush.

B no observará estado PHP residual.

---

# 203. Transaction leak test

A abre transaction y falla.

Antes de B:

```text
active transaction = false
```

---

# 204. Connection session test

A cambia:

```text
role
timezone
schema
session variable
```

B recibe estado default esperado.

---

# 205. Tenant test

```text
A = tenant-alpha
B = tenant-beta
```

sin contaminación.

---

# 206. Security test

Privilegios temporales de A no aparecen en B.

---

# 207. Cursor test

A abandona cursor.

Finalization deberá cerrarlo o descartar conexión.

---

# 208. Streaming test

Early break deberá liberar recursos.

---

# 209. LazyCollection test

Iteración parcial no deberá filtrar recursos al siguiente request.

---

# 210. Exception test

Application exception seguida de request B.

B deberá comenzar limpio.

---

# 211. Cancellation test

Request cancelado seguido de B.

Mismo requisito.

---

# 212. Reset failure test

Conexión que falla reset:

```text
discard
```

y no será entregada a B.

---

# 213. Isolation unknown test

Si shared state queda UNKNOWN:

```text
worker tainted
```

---

# 214. Memory soak test

Ejemplo:

```text
100,000 requests
```

con mezcla de operaciones.

---

# 215. Soak assertions

Al terminar cada request:

```text
active request EntityManagers = active requests
active request IdentityMaps = active requests
active request UnitOfWorks = active requests
```

y al quedar idle:

```text
active request EntityManagers = 0
active request IdentityMaps = 0
active request UnitOfWorks = 0
active transactions = 0
active leases = 0
active cursors = 0
```

---

# 216. Worker recycle test

Forzar:

```text
max executions
```

y comprobar drain/recycle.

---

# 217. Memory threshold test

Forzar crecimiento y comprobar política.

---

# 218. Credential rotation test

A usa generation N.

Después:

```text
new connection → N+1
```

---

# 219. Topology failover test

Verificar que nuevas operaciones respetan nueva autoridad sin falsificar outcome de operaciones anteriores.

---

# 220. Configuration reload test

Request activo conserva snapshot válido.

Nuevo request obtiene nueva generación.

---

# 221. FrankenPHP invariants

## DB-FRANKENPHP-001

FrankenPHP será un adapter del Database runtime.

## DB-FRANKENPHP-002

Database core no dependerá directamente de FrankenPHP.

## DB-FRANKENPHP-003

FrankenPHP será el runtime HTTP predeterminado de VoltStack.

## DB-FRANKENPHP-004

Runtime predeterminado no significará runtime obligatorio.

## DB-FRANKENPHP-005

Classic y persistent modes conservarán semántica Database.

## DB-FRANKENPHP-006

Worker boot podrá reutilizar infraestructura.

## DB-FRANKENPHP-007

Request state nunca será worker-global por default.

## DB-FRANKENPHP-008

Cada request tendrá execution scope.

## DB-FRANKENPHP-009

Cada request tendrá DatabaseContext independiente.

## DB-FRANKENPHP-010

EntityManager será request/execution scoped.

## DB-FRANKENPHP-011

IdentityMap será request/execution scoped.

## DB-FRANKENPHP-012

UnitOfWork será request/execution scoped.

## DB-FRANKENPHP-013

TransactionContext será request/execution scoped.

## DB-FRANKENPHP-014

TenantContext será request/execution scoped.

## DB-FRANKENPHP-015

SecurityContext será request/execution scoped.

## DB-FRANKENPHP-016

QueryContext será request/execution scoped.

## DB-FRANKENPHP-017

Connection pool podrá ser worker-scoped.

## DB-FRANKENPHP-018

Physical connection podrá sobrevivir requests.

## DB-FRANKENPHP-019

Connection lease no sobrevivirá arbitrariamente requests.

## DB-FRANKENPHP-020

Physical connection reuse requerirá reset.

## DB-FRANKENPHP-021

Physical connection reuse requerirá verification.

## DB-FRANKENPHP-022

UNKNOWN connection cleanliness implicará discard.

## DB-FRANKENPHP-023

Active transaction impedirá pool reuse.

## DB-FRANKENPHP-024

Active cursor impedirá pool reuse.

## DB-FRANKENPHP-025

Session state deberá restaurarse.

## DB-FRANKENPHP-026

Database role deberá restaurarse.

## DB-FRANKENPHP-027

Tenant-specific connection state deberá restaurarse.

## DB-FRANKENPHP-028

Temporary state deberá eliminarse o causar discard.

## DB-FRANKENPHP-029

Request finalization será obligatoria.

## DB-FRANKENPHP-030

Application exception no omitirá finalization.

## DB-FRANKENPHP-031

Cancellation no omitirá finalization.

## DB-FRANKENPHP-032

Request end no hará auto-flush ORM por default.

## DB-FRANKENPHP-033

Unresolved transaction hará rollback cuando sea seguro.

## DB-FRANKENPHP-034

Rollback failure podrá descartar conexión.

## DB-FRANKENPHP-035

UNKNOWN commit permanecerá UNKNOWN.

## DB-FRANKENPHP-036

EntityManager se cerrará al finalizar.

## DB-FRANKENPHP-037

IdentityMap se vaciará.

## DB-FRANKENPHP-038

UnitOfWork se vaciará.

## DB-FRANKENPHP-039

DatabaseContext se eliminará.

## DB-FRANKENPHP-040

TenantContext se eliminará.

## DB-FRANKENPHP-041

SecurityContext se eliminará.

## DB-FRANKENPHP-042

Active ResultCursor deberá cerrarse.

## DB-FRANKENPHP-043

Active StreamingResult deberá cerrarse.

## DB-FRANKENPHP-044

Active LazyCollection source deberá cerrarse.

## DB-FRANKENPHP-045

Chunk traversal deberá finalizarse.

## DB-FRANKENPHP-046

ExecutionResourceRegistry rastreará recursos.

## DB-FRANKENPHP-047

Response completion no equivaldrá a Database finalization.

## DB-FRANKENPHP-048

Scope finalizing no será reutilizado.

## DB-FRANKENPHP-049

Fiber context deberá estar aislado.

## DB-FRANKENPHP-050

Concurrent executions deberán estar aisladas.

## DB-FRANKENPHP-051

No habrá global mutable current context.

## DB-FRANKENPHP-052

Worker-scoped service no capturará request service persistentemente.

## DB-FRANKENPHP-053

Container podrá validar lifetime inversions.

## DB-FRANKENPHP-054

Worker memory será observable.

## DB-FRANKENPHP-055

Persistent memory growth podrá disparar recycle.

## DB-FRANKENPHP-056

Recycle no sustituirá cleanup correcto.

## DB-FRANKENPHP-057

Worker taint detendrá nuevas admissions.

## DB-FRANKENPHP-058

Resource-local failure se contendrá localmente cuando sea posible.

## DB-FRANKENPHP-059

Shared-state corruption podrá reciclar worker.

## DB-FRANKENPHP-060

Worker health será independiente del request result.

## DB-FRANKENPHP-061

Database outage no requerirá necesariamente matar worker.

## DB-FRANKENPHP-062

Sticky routing será request-scoped.

## DB-FRANKENPHP-063

Replica health podrá ser worker-shared.

## DB-FRANKENPHP-064

Tenant routing nunca será static global.

## DB-FRANKENPHP-065

Privileged role nunca cruzará requests.

## DB-FRANKENPHP-066

Telemetry context nunca cruzará requests.

## DB-FRANKENPHP-067

Metrics tendrán cardinalidad bounded.

## DB-FRANKENPHP-068

Development mode podrá usar strict isolation.

## DB-FRANKENPHP-069

Production conservará mandatory correctness checks.

## DB-FRANKENPHP-070

Configuration será generation-aware.

## DB-FRANKENPHP-071

Metadata será generation-aware.

## DB-FRANKENPHP-072

Credentials serán generation-aware.

## DB-FRANKENPHP-073

Topology será generation-aware.

## DB-FRANKENPHP-074

Mid-request config mutation será evitada.

## DB-FRANKENPHP-075

Credential rotation no migrará transaction activa.

## DB-FRANKENPHP-076

Writer failover no migrará transaction transparentemente.

## DB-FRANKENPHP-077

Shutdown detendrá admissions antes de pools.

## DB-FRANKENPHP-078

Graceful shutdown drenará executions.

## DB-FRANKENPHP-079

Forced shutdown tendrá deadline.

## DB-FRANKENPHP-080

Shutdown no garantizará outcome conocido de commits interrumpidos.

## DB-FRANKENPHP-081

Streaming HTTP podrá extender execution scope.

## DB-FRANKENPHP-082

Response sent no significará automáticamente scope destroyed.

## DB-FRANKENPHP-083

Detached background work recibirá nuevo scope.

## DB-FRANKENPHP-084

Old EntityManager no será usado por background work.

## DB-FRANKENPHP-085

Framework boot reuse estará permitido.

## DB-FRANKENPHP-086

Compiled metadata reuse estará permitido.

## DB-FRANKENPHP-087

Compiled query reuse estará permitido.

## DB-FRANKENPHP-088

Connection reuse estará permitido bajo invariantes.

## DB-FRANKENPHP-089

Correctness tendrá prioridad sobre reuse.

## DB-FRANKENPHP-090

Isolation tendrá prioridad sobre throughput.

## DB-FRANKENPHP-091

Unknown cleanliness tendrá tratamiento conservador.

## DB-FRANKENPHP-092

Temporary tables serán consideradas durante reset.

## DB-FRANKENPHP-093

Session variables serán consideradas durante reset.

## DB-FRANKENPHP-094

Advisory locks serán consideradas durante reset.

## DB-FRANKENPHP-095

Schema/search path será considerado durante reset.

## DB-FRANKENPHP-096

Timezone será considerada durante reset.

## DB-FRANKENPHP-097

Isolation overrides serán considerados durante reset.

## DB-FRANKENPHP-098

Prepared statement reuse no implicará binding reuse.

## DB-FRANKENPHP-099

Pool capacity considerará número de workers.

## DB-FRANKENPHP-100

Pool capacity considerará DB global capacity.

## DB-FRANKENPHP-101

Connection acquisition será bounded.

## DB-FRANKENPHP-102

Idle connections serán gobernadas.

## DB-FRANKENPHP-103

Connection lifetime podrá ser bounded.

## DB-FRANKENPHP-104

Worker lifetime podrá ser bounded.

## DB-FRANKENPHP-105

Request count podrá ser bounded.

## DB-FRANKENPHP-106

Long DB operations tendrán resource governance.

## DB-FRANKENPHP-107

Import/export HTTP largo podrá delegarse a jobs.

## DB-FRANKENPHP-108

Finalization tendrá budget propio.

## DB-FRANKENPHP-109

Cancellation token podrá propagarse a Query Executor.

## DB-FRANKENPHP-110

Write cancellation no implicará no-write.

## DB-FRANKENPHP-111

Request cleanliness será verificable.

## DB-FRANKENPHP-112

Worker cleanliness será verificable.

## DB-FRANKENPHP-113

No active transaction quedará tras verified finalization.

## DB-FRANKENPHP-114

No active lease quedará tras verified finalization.

## DB-FRANKENPHP-115

No active cursor quedará tras verified finalization.

## DB-FRANKENPHP-116

No request IdentityMap quedará tras scope destruction.

## DB-FRANKENPHP-117

No request UnitOfWork quedará tras scope destruction.

## DB-FRANKENPHP-118

No request tenant state quedará tras scope destruction.

## DB-FRANKENPHP-119

No request security state quedará tras scope destruction.

## DB-FRANKENPHP-120

Runtime adapter tendrá conformance tests.

## DB-FRANKENPHP-121

Persistent mode tendrá sequential contamination tests.

## DB-FRANKENPHP-122

Persistent mode tendrá concurrent contamination tests.

## DB-FRANKENPHP-123

Persistent mode tendrá transaction leak tests.

## DB-FRANKENPHP-124

Persistent mode tendrá tenant leak tests.

## DB-FRANKENPHP-125

Persistent mode tendrá security leak tests.

## DB-FRANKENPHP-126

Persistent mode tendrá connection state tests.

## DB-FRANKENPHP-127

Persistent mode tendrá memory tests.

## DB-FRANKENPHP-128

Persistent mode tendrá soak tests.

## DB-FRANKENPHP-129

Persistent mode tendrá recycle tests.

## DB-FRANKENPHP-130

Persistent mode tendrá shutdown tests.

## DB-FRANKENPHP-131

Persistent mode tendrá fault-injection tests.

## DB-FRANKENPHP-132

Persistent mode tendrá config-generation tests.

## DB-FRANKENPHP-133

Persistent mode tendrá credential-rotation tests.

## DB-FRANKENPHP-134

Persistent mode tendrá topology-change tests.

## DB-FRANKENPHP-135

Persistent mode tendrá failover tests.

## DB-FRANKENPHP-136

FrankenPHP-specific code permanecerá fuera de ORM.

## DB-FRANKENPHP-137

FrankenPHP-specific code permanecerá fuera de Query Engine.

## DB-FRANKENPHP-138

FrankenPHP-specific code permanecerá fuera de Compiler.

## DB-FRANKENPHP-139

FrankenPHP-specific code permanecerá fuera de Driver contracts generales.

## DB-FRANKENPHP-140

RuntimeManagerServer gestionará instalación/configuración del servidor.

## DB-FRANKENPHP-141

Database integration sólo gestionará lifecycle Database.

## DB-FRANKENPHP-142

El mismo ORM deberá funcionar en otros runtimes.

## DB-FRANKENPHP-143

La misma Query API deberá funcionar en otros runtimes.

## DB-FRANKENPHP-144

Las garantías transaccionales no cambiarán por usar FrankenPHP.

## DB-FRANKENPHP-145

Las garantías de tenant isolation no cambiarán por usar FrankenPHP.

## DB-FRANKENPHP-146

Las garantías de seguridad no cambiarán por usar FrankenPHP.

## DB-FRANKENPHP-147

Las garantías de cache consistency no cambiarán por usar FrankenPHP.

## DB-FRANKENPHP-148

Las garantías de persistence consistency no cambiarán por usar FrankenPHP.

## DB-FRANKENPHP-149

El runtime podrá optimizar lifecycle, no redefinir semántica.

## DB-FRANKENPHP-150

Persistent execution será una optimización operacional, no una nueva semántica Database.

---

# 222. Arquitectura completa

```text
                         FRANKENPHP
                              │
                              ▼
                    VoltStack Runtime Layer
                              │
                              ▼
              FrankenPhpDatabaseRuntimeAdapter
                              │
                              ▼
                 DatabaseWorkerLifecycleManager
                              │
          ┌───────────────────┴───────────────────┐
          │                                       │
          ▼                                       ▼
 Shared Worker Infrastructure              Request Admission
          │                                       │
          │                                       ▼
          │                               Execution Scope
          │                                       │
          │                      ┌────────────────┼────────────────┐
          │                      │                │                │
          │                      ▼                ▼                ▼
          │               DatabaseContext   EntityManager   ResourceRegistry
          │                                     │
          │                              ┌──────┴──────┐
          │                              ▼             ▼
          │                         IdentityMap    UnitOfWork
          │
          ├── Driver Registry
          ├── Platform Registry
          ├── Type Registry
          ├── Metadata
          ├── Compiler Cache
          ├── Metadata Cache
          ├── Connection Pools
          └── Telemetry
                                                 │
                                                 ▼
                                           Application
                                                 │
                                                 ▼
                                           Finalization
                                                 │
                    ┌────────────────────────────┼────────────────────────────┐
                    │                            │                            │
                    ▼                            ▼                            ▼
             Resolve Transactions        Release Resources             Clear ORM
                    │                            │                            │
                    └────────────────────────────┼────────────────────────────┘
                                                 ▼
                                               Reset
                                                 │
                                                 ▼
                                              Verify
                                                 │
                           ┌─────────────────────┴─────────────────────┐
                           │                                           │
                           ▼                                           ▼
                         SAFE                                       UNSAFE
                           │                                           │
                           ▼                                           ▼
                    Worker Reusable                          Contain / Taint
                           │                                           │
                           ▼                                           ▼
                    Next Request                              Drain / Recycle
```

---

# 223. Modelo operacional recomendado

VoltStack deberá tratar FrankenPHP bajo cuatro niveles:

```text
LEVEL 1 — PROCESS
LEVEL 2 — WORKER
LEVEL 3 — EXECUTION/REQUEST
LEVEL 4 — DATABASE OPERATION
```

Cada nivel tendrá ownership claro.

---

# 224. Process level

Contendrá únicamente infraestructura apropiada para toda la vida del proceso.

---

# 225. Worker level

Podrá contener:

```text
connection pools
compiled metadata
bounded caches
runtime bridges
telemetry infrastructure
```

---

# 226. Request level

Contendrá:

```text
DatabaseContext
EntityManager
IdentityMap
UnitOfWork
TransactionContext
TenantContext
SecurityContext
ResourceRegistry
```

---

# 227. Operation level

Contendrá:

```text
QueryContext
bindings
execution plan
statement
result cursor
hydration session
```

según operación.

---

# 228. Optimización resultante

Esta separación permite obtener las ventajas de FrankenPHP:

```text
no repeated framework cold boot
no repeated metadata compilation
less connection establishment
warm query infrastructure
warm application container
```

sin sacrificar:

```text
request isolation
transaction safety
tenant isolation
security isolation
ORM correctness
```

---

# 229. Fórmula conceptual

El costo promedio por request puede representarse como:

```text
Trequest
=
Tscope
+
Tapplication
+
Tdatabase
+
Tfinalization
```

mientras costos como:

```text
Tframework_boot
Tmetadata_compile
Tservice_graph_compile
```

pueden amortizarse durante la vida del worker.

---

# 230. Connection benefit

Sin reuse:

```text
Tdatabase
=
Tconnect
+
Tquery
+
Tdisconnect
```

Con pool/reuse:

```text
Tdatabase
≈
Tacquire
+
Tquery
+
Treset
+
Treleases
```

siempre que:

```text
Treset + Tverify
```

mantengan correctness.

---

# 231. Regla de optimización

Nunca:

```text
skip reset
```

para reducir latencia.

Si reset es demasiado costoso, deberá optimizarse el mecanismo, no eliminarse la garantía.

---

# 232. Principio final

La integración FrankenPHP de VoltStack Database deberá seguir:

```text
BOOT INFRASTRUCTURE ONCE
```

```text
CREATE DATABASE SCOPE PER REQUEST
```

```text
ACQUIRE RESOURCES ON DEMAND
```

```text
REUSE ONLY VERIFIED RESOURCES
```

```text
FINALIZE EVERY REQUEST
```

```text
RESET EVERY REQUEST
```

```text
VERIFY BEFORE NEXT USE
```

```text
TAINT WHEN SAFETY IS UNKNOWN
```

```text
RECYCLE WHEN ISOLATION CANNOT BE PROVEN
```

En consecuencia:

> **FrankenPHP será una ventaja de rendimiento para VoltStack únicamente porque la infraestructura segura puede persistir; nunca porque el estado mutable de una ejecución sea permitido sobrevivir accidentalmente a la siguiente.**

---

# 233. Estado del Bloque 25

```text
BLOCK 25 — PERSISTENT RUNTIME

✓ 251_DATABASE_PERSISTENT_RUNTIME_ARCHITECTURE.md
✓ 252_DATABASE_REQUEST_SCOPE_SYSTEM.md
✓ 253_DATABASE_DATABASE_CONTEXT_SYSTEM.md
✓ 254_DATABASE_STATE_ISOLATION_SYSTEM.md
✓ 255_DATABASE_STATE_RESET_SYSTEM.md
✓ 256_DATABASE_CONNECTION_REUSE_SYSTEM.md
✓ 257_DATABASE_WORKER_LIFECYCLE_SYSTEM.md
✓ 258_DATABASE_FRANKENPHP_INTEGRATION_SYSTEM.md
│
├── 259_DATABASE_ROADRUNNER_INTEGRATION_SYSTEM.md
└── 260_DATABASE_OPENSWOOLE_INTEGRATION_SYSTEM.md
```

---

# 234. Siguiente documento

```text
259_DATABASE_ROADRUNNER_INTEGRATION_SYSTEM.md
```

El siguiente documento definirá la adaptación del mismo modelo:

```text
Persistent Infrastructure
        +
Execution Scope Isolation
        +
Connection Reuse
        +
Reset
        +
Verification
```

sobre RoadRunner, prestando especial atención a:

```text
RoadRunner Worker Lifecycle
PSR Worker Integration
Request/Job Boundaries
Worker State
Execution Scope
DatabaseContext
EntityManager
IdentityMap
UnitOfWork
Transactions
Connection Pools
Connection Reuse
Worker Reset
Memory Management
Worker Recycling
Failure Recovery
Jobs
Background Processing
Telemetry
Diagnostics
Graceful Shutdown
```

manteniendo la regla arquitectónica:

> **FrankenPHP, RoadRunner y OpenSwoole podrán tener modelos de ejecución diferentes, pero ninguno podrá cambiar las invariantes fundamentales del sistema Database de VoltStack.**