# 259_DATABASE_ROADRUNNER_INTEGRATION_SYSTEM.md

# VoltStack Quantum Database
## Database RoadRunner Integration System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 259 — Database RoadRunner Integration System  
**Bloque:** 25 — Persistent Runtime  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `258_DATABASE_FRANKENPHP_INTEGRATION_SYSTEM.md`  
**Siguiente documento:** `260_DATABASE_OPENSWOOLE_INTEGRATION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura de integración entre:

```text
VoltStack/Quantum/Database
```

y el runtime persistente:

```text
RoadRunner
```

RoadRunner será soportado por VoltStack como un **runtime opcional oficial**, mientras que FrankenPHP continuará siendo el runtime predeterminado.

La integración deberá conservar exactamente las mismas invariantes fundamentales definidas por el sistema Database independientemente del runtime utilizado.

Regla central:

> **RoadRunner podrá reutilizar procesos PHP, workers, infraestructura compilada y conexiones físicas, pero cada request, job o unidad de ejecución deberá recibir un Database Execution Scope completamente aislado.**

Formalmente:

```text
PersistentWorker
≠
PersistentRequestState
```

y:

```text
WorkerReuse
⇒
PreviousExecutionFinalized
∧ PreviousExecutionReset
∧ IsolationVerified
```

---

# 2. Relación con FrankenPHP

Los documentos:

```text
251_DATABASE_PERSISTENT_RUNTIME_ARCHITECTURE.md
252_DATABASE_REQUEST_SCOPE_SYSTEM.md
253_DATABASE_DATABASE_CONTEXT_SYSTEM.md
254_DATABASE_STATE_ISOLATION_SYSTEM.md
255_DATABASE_STATE_RESET_SYSTEM.md
256_DATABASE_CONNECTION_REUSE_SYSTEM.md
257_DATABASE_WORKER_LIFECYCLE_SYSTEM.md
258_DATABASE_FRANKENPHP_INTEGRATION_SYSTEM.md
```

definieron el modelo general.

RoadRunner no creará una arquitectura Database alternativa.

Existirá:

```text
Database Persistent Runtime Architecture
                  │
       ┌──────────┼───────────┐
       │          │           │
       ▼          ▼           ▼
 FrankenPHP   RoadRunner   OpenSwoole
   Adapter      Adapter      Adapter
```

---

# 3. Principio de portabilidad

Debe cumplirse:

```text
Application Database Code
          │
          ▼
VoltStack Database API
          │
          ▼
Database Runtime Contracts
          │
     ┌────┴────┐
     │         │
FrankenPHP  RoadRunner
```

La aplicación no deberá conocer el runtime.

Por ejemplo:

```php
$user = User::find(10);

$orders = Order::query()
    ->where('status', 'pending')
    ->get();
```

deberá conservar su semántica en:

```text
FrankenPHP
RoadRunner
OpenSwoole
Classic PHP
CLI
Queue Workers
```

---

# 4. RoadRunner como runtime opcional

La arquitectura objetivo será:

```text
VoltStack
│
├── Default Runtime
│   └── FrankenPHP
│
└── Optional Runtime Adapters
    ├── RoadRunner
    └── OpenSwoole
```

RoadRunner no será dependencia obligatoria de:

```text
ORM
Query Engine
Schema
Migrations
Driver
Compiler
Executor
Transaction System
```

---

# 5. Regla de dependencia

Permitido:

```text
RoadRunner Integration
        ↓
Database Runtime Contracts
        ↓
Database Core
```

Prohibido:

```text
Database Core
        ↓
RoadRunner
```

---

# 6. Objetivos

La integración deberá soportar:

1. detección del runtime;
2. worker bootstrap;
3. worker readiness;
4. request lifecycle;
5. job lifecycle;
6. execution scopes;
7. DatabaseContext;
8. EntityManager;
9. IdentityMap;
10. UnitOfWork;
11. TransactionContext;
12. connection pools;
13. connection reuse;
14. resource cleanup;
15. state reset;
16. isolation verification;
17. worker errors;
18. worker recycling;
19. memory governance;
20. request cancellation;
21. job cancellation;
22. graceful shutdown;
23. worker draining;
24. telemetry;
25. diagnostics;
26. configuration generations;
27. credential rotation;
28. topology changes;
29. fault containment;
30. runtime conformance testing.

---

# 7. No objetivos

Este sistema no deberá implementar:

```text
RoadRunner server internals
RoadRunner binary management
RPC protocol internals
HTTP routing
Queue broker implementation
RoadRunner plugin implementation
ORM semantics
SQL generation
Database drivers
```

---

# 8. RuntimeManagerServer

La instalación de RoadRunner corresponderá a:

```text
VoltStack/Quantum/RuntimeManagerServer
```

El Database adapter sólo gestionará:

```text
RoadRunner Runtime
       ↕
VoltStack Database Lifecycle
```

---

# 9. Separación de responsabilidades

```text
RuntimeManagerServer
│
├── install RoadRunner
├── configure RoadRunner
├── manage runtime package
└── runtime operational setup
```

mientras:

```text
Database RoadRunner Integration
│
├── worker lifecycle adaptation
├── execution scope creation
├── DatabaseContext binding
├── resource cleanup
├── state reset
├── isolation verification
└── Database telemetry integration
```

---

# 10. Modelo fundamental

RoadRunner mantiene procesos PHP persistentes.

Conceptualmente:

```text
RoadRunner
    │
    ├── Worker 1
    │     ├── Request A
    │     ├── Request B
    │     └── Request C
    │
    ├── Worker 2
    │     ├── Request D
    │     └── Request E
    │
    └── Worker N
```

Por tanto:

```text
PHP Process Lifetime
>
Request Lifetime
```

---

# 11. Consecuencia Database

El final de un request ya no equivale a:

```text
PHP process destruction
```

por lo que VoltStack deberá destruir explícitamente su estado lógico.

---

# 12. Worker lifecycle

Modelo:

```text
CREATED
   ↓
BOOTING
   ↓
READY
   ↓
EXECUTING
   ↓
RESETTING
   ↓
READY
```

con posibles transiciones:

```text
READY
 ↓
DRAINING
 ↓
STOPPING
 ↓
STOPPED
```

o:

```text
EXECUTING
 ↓
TAINTED
 ↓
DRAINING
 ↓
RECYCLE
```

---

# 13. Worker boot

Pipeline:

```text
RoadRunner starts PHP worker
            ↓
VoltStack bootstrap
            ↓
Database worker bootstrap
            ↓
Load configuration
            ↓
Initialize registries
            ↓
Initialize metadata
            ↓
Initialize connection infrastructure
            ↓
Register runtime adapter
            ↓
Register reset participants
            ↓
Register isolation verifiers
            ↓
READY
```

---

# 14. Infraestructura reutilizable

Podrá sobrevivir múltiples ejecuciones:

```text
DriverRegistry
PlatformRegistry
DialectRegistry
TypeRegistry
MetadataRegistry
CompiledMetadata
CompiledQueryCache
MetadataCache
ConnectionManager
ConnectionPools
TelemetryProvider
ImmutableServices
```

---

# 15. Estado que no puede sobrevivir

Deberá destruirse:

```text
DatabaseContext
EntityManager
IdentityMap
UnitOfWork
TransactionContext
ConnectionLease
QueryContext
HydrationSession
LazyIteration
ChunkTraversal
ImportExecutionState
ExportExecutionState
SecurityContext
TenantContext
RequestScopedEvents
```

---

# 16. Execution abstraction

RoadRunner puede ejecutar diferentes tipos de trabajo.

VoltStack deberá abstraerlos como:

```text
DatabaseExecution
```

---

# 17. Execution kinds

Ejemplo:

```php
enum DatabaseExecutionKind
{
    case HTTP_REQUEST;
    case JOB;
    case RPC;
    case BACKGROUND_TASK;
    case CUSTOM;
}
```

---

# 18. Request ≠ Job

Aunque ambos se ejecuten en un worker:

```text
HTTP Request
≠
Queue Job
```

pero ambos requieren:

```text
Execution Scope
DatabaseContext
ResourceRegistry
Finalization
Reset
Verification
```

---

# 19. Unified execution lifecycle

```text
Execution Accepted
       ↓
Create Scope
       ↓
Bind Context
       ↓
Execute Application
       ↓
Finalize
       ↓
Reset
       ↓
Verify
       ↓
Release Scope
```

---

# 20. HTTP execution

```text
RoadRunner HTTP Request
          ↓
RoadRunner Adapter
          ↓
DatabaseExecutionScope
          ↓
Application
```

---

# 21. Job execution

```text
RoadRunner Job
      ↓
RoadRunner Adapter
      ↓
DatabaseExecutionScope
      ↓
Job Handler
```

---

# 22. Isolation guarantee

Request A y Job B jamás compartirán accidentalmente:

```text
EntityManager
IdentityMap
UnitOfWork
TransactionContext
TenantContext
SecurityContext
```

---

# 23. DatabaseExecutionScope

Cada ejecución tendrá:

```php
final readonly class DatabaseExecutionScope
{
    public function __construct(
        public DatabaseExecutionId $id,
        public DatabaseExecutionKind $kind,
        public DatabaseRuntime $runtime,
        public DatabaseContext $context,
    ) {}
}
```

---

# 24. Runtime identity

Para RoadRunner:

```text
DatabaseRuntime::ROADRUNNER
```

---

# 25. Scope resolver

Se propone:

```php
interface DatabaseExecutionScopeResolver
{
    public function current(): DatabaseExecutionScope;
}
```

---

# 26. No static current request

Prohibido:

```php
Database::$currentRequest = $request;
```

---

# 27. No static EntityManager

Prohibido:

```php
EntityManager::$current = $manager;
```

---

# 28. No static tenant

Prohibido:

```php
Tenant::$current = $tenant;
```

---

# 29. Context locality

El contexto deberá resolverse mediante:

```text
scoped container
execution-local storage
Fiber-local abstraction
runtime context bridge
```

---

# 30. RoadRunner adapter

Contrato conceptual:

```php
final class RoadRunnerDatabaseRuntimeAdapter
    implements DatabaseRuntimeAdapter
{
    public function runtime(): DatabaseRuntime
    {
        return DatabaseRuntime::ROADRUNNER;
    }
}
```

---

# 31. RoadRunner execution bridge

```php
interface RoadRunnerDatabaseExecutionBridge
{
    public function begin(
        RoadRunnerExecutionContext $context
    ): DatabaseExecutionScope;

    public function complete(
        DatabaseExecutionScope $scope
    ): DatabaseFinalizationReport;
}
```

---

# 32. Execution pipeline

Conceptualmente:

```php
$scope = $databaseLifecycle->begin($execution);

try {
    return $application->handle($execution);
} finally {
    $databaseLifecycle->finalize($scope);
}
```

---

# 33. Finally es obligatorio

Cualquier:

```text
return
exception
HTTP error
job failure
cancellation
```

deberá pasar por finalization.

---

# 34. Request finalization

```text
Application finishes
        ↓
Freeze DB scope
        ↓
Cancel pending operations
        ↓
Close cursors
        ↓
Close streams
        ↓
Close lazy sources
        ↓
Resolve transaction state
        ↓
Release connection leases
        ↓
Clear UoW
        ↓
Clear IdentityMap
        ↓
Close EntityManager
        ↓
Clear DatabaseContext
        ↓
Reset scoped services
        ↓
Verify isolation
        ↓
Destroy scope
```

---

# 35. Job finalization

Mismo modelo.

Un job no podrá dejar estado para el siguiente job.

---

# 36. Worker loop

Modelo conceptual:

```text
while worker is alive:

    receive execution

    create Database scope

    try:
        process execution
    finally:
        finalize Database scope
        reset
        verify

    if worker unsafe:
        stop/recycle
```

---

# 37. EntityManager lifecycle

Por ejecución:

```text
CREATE
  ↓
OPEN
  ↓
USED
  ↓
CLOSE
  ↓
DESTROY
```

---

# 38. EntityManager no se reutiliza

Nunca:

```text
Request A EntityManager
       ↓
Request B
```

---

# 39. IdentityMap

Nueva por ejecución.

Formalmente:

```text
IdentityMap(E1)
∩
IdentityMap(E2)
=
∅
```

para scopes distintos.

---

# 40. Same entity across requests

Request A:

```text
User#42 → Object A
```

Request B:

```text
User#42 → Object B
```

---

# 41. UnitOfWork

No podrá conservar:

```text
NEW
DIRTY
REMOVED
MANAGED
```

entre ejecuciones.

---

# 42. Pending mutation

Ejemplo peligroso:

```text
Request A:
$user->name = 'Alice';

Request ends without flush.
```

Request B nunca deberá poder provocar:

```text
UPDATE users ...
```

por cambios de A.

---

# 43. Transaction lifecycle

Una transaction pertenece a:

```text
Execution Scope
```

salvo un modelo explícito de ejecución superior.

---

# 44. No cross-request transaction

Prohibido:

```text
Request A
BEGIN
   ↓
Request B
COMMIT
```

---

# 45. Request ends with active transaction

Default:

```text
ROLLBACK
```

cuando pueda ejecutarse con resultado conocido.

---

# 46. Rollback failure

Si no puede confirmarse:

```text
TransactionOutcome = UNKNOWN
Connection = DISCARD
```

---

# 47. Commit uncertainty

```text
COMMIT sent
    ↓
connection lost
```

produce:

```text
UNKNOWN
```

---

# 48. Retry prohibition

No deberá ejecutarse automáticamente:

```text
retry whole request
```

sólo porque el worker continúe vivo.

---

# 49. Retry architecture

Retry seguirá las reglas de:

```text
238_DATABASE_RETRY_POLICY_SYSTEM.md
```

---

# 50. Connection pools

RoadRunner puede beneficiarse especialmente de:

```text
persistent physical connections
```

pero bajo el modelo definido por:

```text
256_DATABASE_CONNECTION_REUSE_SYSTEM.md
```

---

# 51. Pool ownership

Conceptualmente:

```text
RoadRunner Worker
       │
       └── Database Connection Pool
```

---

# 52. Multiple workers

```text
RR Worker 1 → Pool 1
RR Worker 2 → Pool 2
RR Worker 3 → Pool 3
```

El core no asumirá pool global entre procesos.

---

# 53. Capacity formula

Con:

```text
W = number of workers
P = max physical connections per worker
```

el máximo teórico:

```text
Cmax ≈ W × P
```

---

# 54. Multiple logical databases

Si existen:

```text
D1
D2
D3
```

con pools independientes:

```text
Cmax ≈ Σ(W × Pi)
```

según configuración.

---

# 55. Capacity must be bounded

No deberá utilizarse:

```text
unbounded pool
```

---

# 56. Acquisition

Cada operación obtiene:

```text
ConnectionLease
```

no ownership directo permanente de la conexión.

---

# 57. Lease ownership

```text
Lease.executionId
```

deberá corresponder al execution scope.

---

# 58. Cross-scope lease use

Debe ser error.

---

# 59. Connection return

```text
Execution
   ↓
release lease
   ↓
reset connection
   ↓
verify connection
   ↓
pool
```

---

# 60. Unsafe connection

Si:

```text
Cleanliness = UNKNOWN
```

entonces:

```text
DISCARD
```

---

# 61. Connection reset

Considerará:

```text
transactions
savepoints
session variables
roles
schemas
search paths
timezone
isolation level
temporary state
active cursors
advisory locks
driver state
```

---

# 62. RoadRunner does not redefine reset

El adapter sólo invocará:

```text
Database Connection Reset System
```

---

# 63. Connection affinity

Transaction activa:

```text
TransactionContext
       ↓
ConnectionLease
       ↓
PhysicalConnection
```

permanece pinned.

---

# 64. Request without transaction

Podrá adquirir/liberar conexiones según política.

---

# 65. Job without transaction

Igual.

---

# 66. Queue jobs

RoadRunner puede utilizarse en arquitecturas donde el worker procesa trabajos repetidos.

VoltStack deberá tratar cada job como una nueva frontera de aislamiento.

---

# 67. Job A → Job B

```text
Job A
 ↓
Finalize
 ↓
Reset
 ↓
Verify
 ↓
Job B
```

---

# 68. Job retry

Un retry será una nueva ejecución lógica.

---

# 69. Retry scope

```text
Job Attempt 1 → ExecutionScope A
Job Attempt 2 → ExecutionScope B
```

---

# 70. ORM identity across retries

No compartir IdentityMap.

---

# 71. Job transaction ownership

Si el job inicia transaction:

```text
job execution owns transaction
```

si no existe una superior.

---

# 72. Job failure

No deberá dejar transaction abierta.

---

# 73. External side effects

Rollback DB no revierte:

```text
email
HTTP API
filesystem
message broker
```

---

# 74. Outbox

Para coordinación robusta:

```text
DB Transaction
    │
    ├── domain change
    └── outbox record
```

según sistemas superiores.

---

# 75. Job acknowledgement

Importante distinguir:

```text
Database commit
```

de:

```text
Job acknowledgement
```

---

# 76. Commit ≠ ACK

Una operación puede:

```text
DB commit successful
```

pero fallar antes del ACK.

Esto puede causar reentrega.

---

# 77. Job idempotency

Los handlers deberán diseñarse para:

```text
at-least-once delivery
```

cuando corresponda al broker/runtime.

Database no prometerá exactly-once.

---

# 78. Exactly-once invariant

```text
Persistent Worker
+
Transaction
+
Job Queue
≠
Exactly Once
```

---

# 79. Request cancellation

Cancellation deberá propagarse mediante:

```text
CancellationToken
```

---

# 80. Job cancellation

Mismo principio.

---

# 81. Cancellation hierarchy

```text
Worker Shutdown
       ↓
Execution Cancellation
       ↓
Database Operation Cancellation
```

---

# 82. Query cancellation

Si driver lo soporta:

```text
CancellationToken
      ↓
QueryExecutor
      ↓
Driver
```

---

# 83. Cancellation uncertainty

Cancelar un write no implica:

```text
write definitely did not happen
```

---

# 84. ExecutionResourceRegistry

Cada scope mantendrá registro de recursos activos.

Ejemplos:

```text
ConnectionLease
ResultCursor
StreamingResult
LazySource
ChunkTraversal
ImportReader
ExportWriter
```

---

# 85. Resource cleanup

Durante finalization:

```text
for each owned resource:
    close/release/cancel
```

siguiendo orden semántico.

---

# 86. Lazy Collection

Una lazy collection no consumida completamente deberá cerrar su fuente.

---

# 87. Streaming results

No deberán quedar abiertos después del execution boundary.

---

# 88. Chunk processing

El estado de chunk no sobrevivirá accidentalmente.

---

# 89. Import

Import execution state será scope-local.

---

# 90. Export

Export execution state será scope-local.

---

# 91. Long-running jobs

RoadRunner puede ejecutar trabajos largos.

Por tanto Database deberá gobernar:

```text
memory
connections
transaction duration
query duration
result size
IdentityMap growth
```

---

# 92. Long transaction warning

Un job de 2 horas no deberá implicar automáticamente:

```text
2-hour transaction
```

---

# 93. Chunked transaction policy

Preferible cuando sea semánticamente válido:

```text
Chunk 1 → transaction
Chunk 2 → transaction
Chunk 3 → transaction
```

en lugar de:

```text
whole dataset → one giant transaction
```

---

# 94. ORM memory

Procesar millones de entidades puede hacer crecer:

```text
IdentityMap
UnitOfWork
```

aunque el worker sea persistente.

---

# 95. Explicit ORM memory policy

Deberán utilizarse estrategias definidas por:

```text
248_DATABASE_MEMORY_MANAGEMENT_SYSTEM.md
249_DATABASE_RESOURCE_GOVERNANCE_SYSTEM.md
```

---

# 96. No hidden EntityManager clear

RoadRunner adapter nunca ejecutará arbitrariamente:

```php
$entityManager->clear();
```

durante una operación activa.

---

# 97. Clear belongs to lifecycle

El clear global del scope sólo ocurre durante finalization.

---

# 98. Batch operations

Durante una operación larga, clearing parcial deberá ser explícito y compatible con el ORM processing policy.

---

# 99. Worker memory accumulation

RoadRunner worker deberá observar:

```text
memory at boot
memory before execution
memory peak
memory after finalization
memory after reset
```

con sampling/configuración apropiada.

---

# 100. Memory drift

Definimos conceptualmente:

```text
Drift(n)
=
PostResetMemory(n)
-
BaselineMemory
```

---

# 101. Drift trend

Un único incremento no demuestra leak.

Pero crecimiento sostenido podrá producir:

```text
HEALTHY
→ DEGRADED
→ RECYCLE_REQUESTED
```

---

# 102. Worker recycle

Puede ocurrir por:

```text
execution count
worker age
memory threshold
reset failures
isolation failures
configuration changes
credential changes
operator policy
deployment
```

---

# 103. Recycle ≠ cleanup

Reiniciar workers periódicamente no sustituye:

```text
proper request cleanup
```

---

# 104. Max jobs

Puede existir:

```text
max executions per worker
```

---

# 105. Worker generation

Cada worker tendrá:

```text
WorkerGeneration
```

para diagnóstico.

---

# 106. Worker identity

No deberá utilizarse como tenant/security identity.

---

# 107. Tainted worker

Estados:

```php
enum DatabaseWorkerState
{
    case BOOTING;
    case READY;
    case BUSY;
    case DRAINING;
    case TAINTED;
    case STOPPING;
    case STOPPED;
}
```

---

# 108. Taint triggers

Ejemplos:

```text
shared scope contamination
unrecoverable reset infrastructure failure
corrupted shared registry
unknown global runtime state
repeated isolation verification failures
```

---

# 109. Tainted worker behavior

```text
TAINTED
   ↓
reject new executions
   ↓
drain active execution
   ↓
recycle
```

---

# 110. Local failure

Una mala conexión no deberá necesariamente contaminar worker.

```text
bad connection
→ discard connection
```

---

# 111. Request failure

Una excepción de negocio tampoco:

```text
request exception
→ finalize scope
→ worker READY
```

si cleanup es exitoso.

---

# 112. Failure containment levels

```text
OPERATION
EXECUTION
CONNECTION
POOL
WORKER
PROCESS
```

---

# 113. Smallest containment principle

El fallo deberá contenerse en el nivel más pequeño seguro.

---

# 114. Example

```text
Query timeout
→ query failure
```

no:

```text
kill worker
```

salvo daño adicional.

---

# 115. Isolation verification

Después de execution:

```text
active transactions == 0
active leases == 0
active cursors == 0
active streams == 0
request EntityManager closed
request IdentityMap cleared
request UnitOfWork cleared
DatabaseContext unbound
TenantContext unbound
SecurityContext unbound
```

---

# 116. Isolation report

```php
final readonly class DatabaseIsolationReport
{
    public function __construct(
        public DatabaseIsolationStatus $status,
        public array $violations,
        public array $containmentActions,
    ) {}
}
```

---

# 117. Isolation statuses

```php
enum DatabaseIsolationStatus
{
    case VERIFIED;
    case CONTAINED;
    case FAILED;
    case UNKNOWN;
}
```

---

# 118. UNKNOWN

Nunca será equivalente a:

```text
VERIFIED
```

---

# 119. UNKNOWN shared state

Podrá requerir worker recycle.

---

# 120. Tenant isolation

RoadRunner hace especialmente importante evitar:

```text
static current tenant
```

---

# 121. Tenant sequence test

```text
Request A → Tenant A
Request B → Tenant B
Request C → Tenant A
```

Cada uno debe reconstruir contexto.

---

# 122. Tenant connection state

Si Tenant A modifica:

```text
schema
role
database
session variable
```

la conexión deberá limpiarse antes de Tenant B.

---

# 123. Tenant cache key

Caches compartidas deberán incluir contexto tenant cuando semánticamente corresponda.

---

# 124. Tenant IdentityMap

Nunca worker-global.

---

# 125. Tenant UnitOfWork

Nunca worker-global.

---

# 126. Security isolation

Lo mismo aplica a:

```text
user identity
authorization scopes
database roles
security filters
sensitive diagnostics
```

---

# 127. Authorization query scope

Deberá reconstruirse por ejecución.

---

# 128. Elevated privileges

Si una operación usa privilegios elevados:

```text
elevate
↓
operation
↓
restore
```

---

# 129. Failure during restore

```text
connection → discard
```

---

# 130. Configuration generations

RoadRunner worker puede vivir durante cambios de configuración.

VoltStack modelará:

```text
ConfigurationGeneration
```

---

# 131. Generation snapshot

Cada execution podrá capturar:

```text
DatabaseEnvironmentSnapshot
```

---

# 132. Snapshot contents

Puede incluir:

```text
config generation
metadata generation
credential generation
topology generation
security policy generation
```

---

# 133. Mid-execution mutation

No cambiará silenciosamente:

```text
tenant routing
transaction semantics
dialect
credentials
metadata mapping
```

---

# 134. New generation

Nuevas ejecuciones podrán utilizarla.

---

# 135. Incompatible change

Si no puede coexistir:

```text
drain worker
→ recycle
```

---

# 136. Credential rotation

```text
Credential N
    ↓
Credential N+1
```

Nuevas conexiones:

```text
N+1
```

Conexiones antiguas:

```text
retire according to policy
```

---

# 137. Existing transaction

No se migrará de conexión.

---

# 138. Topology changes

Read/write routing deberá poder actualizar infraestructura compartida.

---

# 139. Existing execution

Podrá conservar topology generation consistente cuando sea necesario.

---

# 140. Writer failover

Nueva operación:

```text
new writer
```

Transacción activa anterior:

```text
no transparent migration
```

---

# 141. Read replicas

Health puede ser worker-scoped.

---

# 142. Sticky state

Read-your-writes sticky state será execution-scoped.

---

# 143. Replica lag

Información observada puede ser compartida como infraestructura.

---

# 144. Query routing decision

Siempre considera contexto actual.

---

# 145. Fiber/concurrency safety

Database no deberá asumir que:

```text
one PHP worker
=
one permanent execution context
```

---

# 146. Execution-local abstraction

Aunque una implementación concreta procese una ejecución a la vez, la arquitectura no dependerá de globals mutables.

---

# 147. Future compatibility

Esto permitirá integrar:

```text
Fibers
async operations
custom RoadRunner plugins
future concurrency models
```

sin rediseñar ORM.

---

# 148. Concurrent DB operations

Si se permiten dentro de una ejecución:

```text
Operation A
Operation B
```

deberán respetar:

```text
connection ownership
transaction affinity
EntityManager concurrency policy
```

---

# 149. EntityManager concurrency

Por default:

> **Un EntityManager mutable no se considerará concurrent-safe.**

---

# 150. Parallel queries

Podrán requerir:

```text
independent operation contexts
independent connection leases
read-only projections
```

---

# 151. Transaction concurrency

No se ejecutarán operaciones concurrentes sobre una misma conexión transaccional salvo soporte explícito y contrato seguro.

---

# 152. Runtime telemetry

Cada execution podrá incluir:

```text
runtime = roadrunner
worker.id
worker.generation
execution.id
execution.kind
```

---

# 153. HTTP telemetry

Correlación:

```text
RoadRunner Request
      ↓
HTTP Trace
      ↓
Database Execution
      ↓
Query
```

---

# 154. Job telemetry

```text
RoadRunner Job
      ↓
Job Trace
      ↓
Database Execution
      ↓
Transaction
      ↓
Query
```

---

# 155. Metric cardinality

No utilizar:

```text
raw SQL
tenant IDs
user IDs
job IDs
request IDs
```

como labels métricos no bounded.

---

# 156. Metrics

Ejemplos:

```text
database_roadrunner_executions_total
database_roadrunner_execution_failures_total
database_roadrunner_reset_failures_total
database_roadrunner_isolation_failures_total
database_roadrunner_worker_recycles_total
database_roadrunner_connection_reuses_total
database_roadrunner_connection_discards_total
database_roadrunner_active_executions
```

---

# 157. Prefer runtime-neutral metrics

Cuando sea posible:

```text
database_worker_executions_total{runtime="roadrunner"}
```

siempre con dimensiones bounded.

---

# 158. Query profiler

Seguirá funcionando normalmente.

---

# 159. Slow query detector

No dependerá de RoadRunner.

---

# 160. N+1 detector

Será execution-scoped.

Nunca acumulará correlaciones de:

```text
Request A + Request B
```

como si fueran una sola ejecución.

---

# 161. Developer Debug Toolbar

En HTTP podrá consumir información del scope actual.

---

# 162. Job diagnostics

Para jobs deberá existir una representación CLI/log/telemetry equivalente.

---

# 163. Debug information cleanup

El debug collector del request deberá destruirse tras exposición/exportación.

---

# 164. Sensitive telemetry

Aplicarán:

```text
232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM.md
233_DATABASE_QUERY_AUDIT_SYSTEM.md
```

---

# 165. Graceful shutdown

RoadRunner shutdown deberá traducirse a:

```text
STOP_NEW_EXECUTIONS
        ↓
DRAIN
        ↓
FINALIZE
        ↓
CLOSE_POOLS
        ↓
FLUSH_TELEMETRY
        ↓
STOP
```

---

# 166. Shutdown ordering

No cerrar connection pools antes de finalizar ejecuciones activas.

---

# 167. Drain state

```text
READY
 ↓
DRAINING
```

Durante DRAINING:

```text
new execution admission = disabled
existing executions = allowed to finish
```

---

# 168. Shutdown timeout

Deberá existir deadline.

---

# 169. Timeout exceeded

Podrá propagarse cancellation.

---

# 170. Hung DB operation

No deberá impedir shutdown indefinidamente.

---

# 171. Forced termination

Puede producir outcome desconocido.

No falsificar:

```text
rolled back
```

sin evidencia.

---

# 172. Worker recycling

Conceptualmente:

```text
Worker N
   ↓
DRAIN
   ↓
STOP
   ↓
Worker N+1
```

---

# 173. Deployment

Rolling deployment puede usar esta misma semántica.

---

# 174. Readiness

Un worker estará ready sólo si puede aceptar una nueva ejecución de forma segura.

---

# 175. Readiness factors

```text
worker state
Database shared-state health
pool health
critical configuration validity
isolation status
resource capacity
```

---

# 176. Liveness

Liveness no deberá confundirse con:

```text
database available
```

Un worker puede estar vivo pero temporalmente no ready.

---

# 177. Health model

```text
HEALTHY
DEGRADED
UNHEALTHY
TAINTED
```

---

# 178. Database outage

Puede:

```text
READY → DEGRADED
```

o temporalmente:

```text
ready = false
```

sin necesariamente reiniciar proceso.

---

# 179. Circuit breaker integration

```text
DB Endpoint
    ↓
Failures
    ↓
Circuit Breaker
    ↓
Routing Eligibility
```

---

# 180. Worker lifecycle independence

Circuit breaker abierto no significa automáticamente worker tainted.

---

# 181. Resource exhaustion

Integración con:

```text
241_DATABASE_RESOURCE_EXHAUSTION_PROTECTION_SYSTEM.md
249_DATABASE_RESOURCE_GOVERNANCE_SYSTEM.md
```

---

# 182. Admission control

Antes de aceptar trabajo DB intensivo podrá evaluarse:

```text
available connections
memory pressure
execution budget
worker health
deadline
```

---

# 183. Pool saturation

No crear conexiones infinitas.

---

# 184. Backpressure

Podrá:

```text
wait bounded time
reject
shed optional work
```

según política.

---

# 185. Queue workers and backpressure

Si DB está saturada, incrementar workers puede empeorar la situación.

---

# 186. Capacity relationship

```text
More RoadRunner Workers
        ↓
Potential DB Concurrency ↑
        ↓
Potential DB Connections ↑
```

---

# 187. Configuration validation

VoltStack podrá advertir:

```text
RoadRunner workers: 100
DB pool per worker: 20
Potential connections: 2000
DB configured max: 500
```

---

# 188. Recommended warning

```text
Potential database connection demand exceeds configured database capacity.
```

---

# 189. Runtime diagnostics command

Conceptualmente:

```text
php volt database:runtime
```

Salida:

```text
Database Runtime
────────────────────────────────────

Runtime:        RoadRunner
Mode:           Persistent Worker
Worker:         rr-worker-07
Generation:     21
State:          READY
Health:         HEALTHY

Executions
  Current:      HTTP_REQUEST
  Completed:    42,913
  Failed:       112

Database Scope
  Active:       1

ORM
  EntityManager: OPEN
  IdentityMap:   18 entities
  UnitOfWork:    clean

Transactions
  Active:        0

Connections
  Physical:      8
  Leased:        1
  Idle:          7

Isolation
  Previous:      VERIFIED

Memory
  Current:       91 MB
  Peak:          124 MB
```

---

# 190. Explain runtime

Podrá existir:

```text
php volt database:runtime --explain
```

---

# 191. Explain information

```text
runtime adapter
worker lifetime
scope lifetime
pool configuration
reset strategy
verification policy
recycle policy
resource budgets
```

---

# 192. Development strict mode

```php
'database' => [
    'runtime' => [
        'strict_isolation' => true,
        'detect_scope_leaks' => true,
        'verify_connection_state' => true,
    ],
];
```

---

# 193. Production mode

Mantendrá mandatory safety checks con menor overhead diagnóstico.

---

# 194. Scope leak detector

Podrá detectar referencias como:

```text
WorkerService
    ↓
Request EntityManager
```

---

# 195. Listener leak

También:

```text
Worker EventDispatcher
      ↓
Closure
      ↓
Request Context
```

---

# 196. Static leak

También:

```text
static property
      ↓
Entity
```

cuando pueda instrumentarse.

---

# 197. Scope generation

Cada execution tendrá:

```text
ScopeGeneration
```

---

# 198. Resource ownership

Un recurso creado en generation:

```text
G42
```

no podrá utilizarse desde:

```text
G43
```

---

# 199. Generation mismatch

Debe producir:

```text
StaleExecutionResourceException
```

o equivalente.

---

# 200. Connection exception

Una physical connection puede cruzar generaciones.

Su:

```text
ConnectionLease
```

no.

---

# 201. Immutable metadata

Puede cruzar executions si su generation sigue válida.

---

# 202. Compiled queries

También.

---

# 203. Cached entities

Managed entities nunca.

Entity Cache podrá almacenar representación canonical según sus propias reglas.

---

# 204. IdentityMap ≠ Entity Cache

Especialmente importante en workers persistentes.

---

# 205. Testing strategy

RoadRunner integration deberá contar con:

```text
Unit Tests
Adapter Tests
Lifecycle Tests
Persistent Worker Tests
Sequential Isolation Tests
Concurrent Context Tests
HTTP Tests
Job Tests
Transaction Leak Tests
Connection Reset Tests
Tenant Isolation Tests
Security Isolation Tests
Memory Tests
Fault Injection Tests
Soak Tests
Recycle Tests
Shutdown Tests
```

---

# 206. Sequential test

```text
Execution A
 ↓
Reset
 ↓
Execution B
 ↓
Reset
 ↓
Execution C
```

---

# 207. Assertions after each execution

```text
active transaction = 0
active lease = 0
active cursor = 0
active stream = 0
old DatabaseContext inaccessible
old EntityManager inaccessible
old TenantContext inaccessible
```

---

# 208. HTTP → HTTP test

A no contamina B.

---

# 209. Job → Job test

A no contamina B.

---

# 210. HTTP → Job test

HTTP execution no contamina job.

---

# 211. Job → HTTP test

Job no contamina HTTP request.

---

# 212. Tenant test

```text
A = Tenant X
B = Tenant Y
```

verificar:

```text
connection state
ORM state
cache keys
query scope
security filters
```

---

# 213. Transaction test

A abre transaction y lanza exception.

B deberá comenzar sin transaction.

---

# 214. Cursor leak test

A deja cursor abierto.

Finalizer deberá:

```text
close
```

o:

```text
discard underlying connection
```

si no puede limpiarse.

---

# 215. Connection state test

Modificar:

```text
timezone
schema
role
session variable
```

y verificar reset.

---

# 216. Memory soak

Procesar gran número de executions.

Ejemplo:

```text
100,000
```

---

# 217. Soak memory expectation

No se exige memoria perfectamente constante debido a:

```text
allocator behavior
legitimate caches
runtime internals
```

pero sí:

```text
bounded/stabilizing behavior
```

bajo workload estable.

---

# 218. Fault injection

Simular:

```text
DB disconnect
timeout
deadlock
commit uncertainty
rollback failure
reset failure
pool exhaustion
metadata generation change
credential rotation
worker shutdown
```

---

# 219. Runtime conformance suite

Se propone:

```text
DatabasePersistentRuntimeConformanceSuite
```

que deberán pasar:

```text
FrankenPHP Adapter
RoadRunner Adapter
OpenSwoole Adapter
```

---

# 220. Common conformance tests

```text
scope isolation
transaction isolation
connection reuse
reset
verification
tenant isolation
security isolation
resource cleanup
worker drain
worker recycle
shutdown
```

---

# 221. Runtime-specific tests

RoadRunner añadirá pruebas de:

```text
RoadRunner worker bridge
HTTP execution adaptation
job execution adaptation
worker errors
RoadRunner lifecycle integration
```

---

# 222. Proposed directory

```text
src/Quantum/Database/Runtime/RoadRunner/
│
├── Contract/
│   ├── RoadRunnerDatabaseIntegration.php
│   ├── RoadRunnerExecutionBridge.php
│   ├── RoadRunnerExecutionContext.php
│   └── RoadRunnerWorkerBridge.php
│
├── Adapter/
│   ├── RoadRunnerDatabaseRuntimeAdapter.php
│   ├── RoadRunnerHttpExecutionAdapter.php
│   └── RoadRunnerJobExecutionAdapter.php
│
├── Context/
│   ├── RoadRunnerDatabaseContextBinder.php
│   └── RoadRunnerExecutionScopeResolver.php
│
├── Lifecycle/
│   ├── RoadRunnerWorkerBootstrap.php
│   ├── RoadRunnerExecutionLifecycle.php
│   ├── RoadRunnerExecutionFinalizer.php
│   ├── RoadRunnerWorkerDrain.php
│   └── RoadRunnerWorkerShutdown.php
│
├── Connection/
│   ├── RoadRunnerConnectionPoolIntegration.php
│   └── RoadRunnerConnectionReusePolicy.php
│
├── Job/
│   ├── RoadRunnerDatabaseJobBridge.php
│   └── RoadRunnerJobDatabaseContextFactory.php
│
├── Health/
│   ├── RoadRunnerDatabaseHealthContributor.php
│   └── RoadRunnerDatabaseReadinessContributor.php
│
├── Telemetry/
│   └── RoadRunnerDatabaseTelemetryBridge.php
│
├── Diagnostics/
│   └── RoadRunnerDatabaseDiagnostics.php
│
└── Exception/
    ├── RoadRunnerDatabaseIntegrationException.php
    ├── RoadRunnerWorkerException.php
    ├── RoadRunnerExecutionException.php
    ├── RoadRunnerContextException.php
    └── RoadRunnerLifecycleException.php
```

---

# 223. Shared runtime architecture

No deberá duplicarse todo entre:

```text
FrankenPHP/
RoadRunner/
OpenSwoole/
```

La mayor parte residirá en:

```text
src/Quantum/Database/Runtime/
```

---

# 224. Common directory

```text
Runtime/
│
├── Contract/
├── Context/
├── Scope/
├── Lifecycle/
├── Reset/
├── Isolation/
├── Worker/
├── Health/
├── Resource/
├── Telemetry/
├── Diagnostics/
│
├── FrankenPHP/
├── RoadRunner/
└── OpenSwoole/
```

---

# 225. Adapter responsibility

Runtime adapters deberán ser delgados.

Idealmente:

```text
Runtime Event
    ↓
Adapter
    ↓
Common Database Lifecycle
```

---

# 226. Avoid architecture duplication

No crear:

```text
FrankenPHPEntityManager
RoadRunnerEntityManager
OpenSwooleEntityManager
```

---

# 227. Same EntityManager

Debe existir:

```text
EntityManager
```

independiente del runtime.

---

# 228. Same TransactionManager

Igual.

---

# 229. Same ConnectionManager

Igual.

---

# 230. Same QueryExecutor

Igual.

---

# 231. Runtime-specific integration only

RoadRunner-specific:

```text
worker hooks
execution context bridge
lifecycle signals
runtime diagnostics
runtime health bridge
```

---

# 232. RoadRunner integration invariants

## DB-ROADRUNNER-001

RoadRunner será runtime opcional oficial.

## DB-ROADRUNNER-002

FrankenPHP seguirá siendo runtime predeterminado.

## DB-ROADRUNNER-003

Database core no dependerá de RoadRunner.

## DB-ROADRUNNER-004

RoadRunner integration dependerá de Database runtime contracts.

## DB-ROADRUNNER-005

Cada RoadRunner execution tendrá scope propio.

## DB-ROADRUNNER-006

HTTP request tendrá scope propio.

## DB-ROADRUNNER-007

Job tendrá scope propio.

## DB-ROADRUNNER-008

RPC execution tendrá scope propio cuando se soporte.

## DB-ROADRUNNER-009

EntityManager no cruzará scopes.

## DB-ROADRUNNER-010

IdentityMap no cruzará scopes.

## DB-ROADRUNNER-011

UnitOfWork no cruzará scopes.

## DB-ROADRUNNER-012

TransactionContext no cruzará scopes.

## DB-ROADRUNNER-013

DatabaseContext no cruzará scopes.

## DB-ROADRUNNER-014

TenantContext no cruzará scopes.

## DB-ROADRUNNER-015

SecurityContext no cruzará scopes.

## DB-ROADRUNNER-016

ConnectionLease no cruzará scopes.

## DB-ROADRUNNER-017

PhysicalConnection podrá reutilizarse.

## DB-ROADRUNNER-018

Connection reuse requerirá reset.

## DB-ROADRUNNER-019

Connection reuse requerirá verification.

## DB-ROADRUNNER-020

UNKNOWN cleanliness implicará discard.

## DB-ROADRUNNER-021

Worker infrastructure podrá persistir.

## DB-ROADRUNNER-022

Request state no será worker-global.

## DB-ROADRUNNER-023

Job state no será worker-global.

## DB-ROADRUNNER-024

Request finalization será obligatoria.

## DB-ROADRUNNER-025

Job finalization será obligatoria.

## DB-ROADRUNNER-026

Exceptions no omitirán finalization.

## DB-ROADRUNNER-027

Cancellation no omitirá finalization.

## DB-ROADRUNNER-028

Active transaction no cruzará executions.

## DB-ROADRUNNER-029

Active cursor no cruzará executions.

## DB-ROADRUNNER-030

Active stream no cruzará executions.

## DB-ROADRUNNER-031

Lazy source no cruzará executions.

## DB-ROADRUNNER-032

Chunk state no cruzará executions.

## DB-ROADRUNNER-033

Import state no cruzará executions.

## DB-ROADRUNNER-034

Export state no cruzará executions.

## DB-ROADRUNNER-035

Request end no hará implicit flush.

## DB-ROADRUNNER-036

Job end no hará implicit flush.

## DB-ROADRUNNER-037

Unresolved transaction será resuelta conservadoramente.

## DB-ROADRUNNER-038

Commit uncertainty permanecerá UNKNOWN.

## DB-ROADRUNNER-039

Rollback uncertainty permanecerá UNKNOWN.

## DB-ROADRUNNER-040

Unsafe connection será descartada.

## DB-ROADRUNNER-041

Pool será bounded.

## DB-ROADRUNNER-042

Pool sizing considerará worker count.

## DB-ROADRUNNER-043

Worker count considerará DB capacity.

## DB-ROADRUNNER-044

Pool saturation aplicará backpressure.

## DB-ROADRUNNER-045

Pool saturation no creará conexiones ilimitadas.

## DB-ROADRUNNER-046

Worker recycle no sustituirá reset.

## DB-ROADRUNNER-047

Worker recycle no sustituirá isolation verification.

## DB-ROADRUNNER-048

Worker memory será observable.

## DB-ROADRUNNER-049

Memory growth podrá solicitar recycle.

## DB-ROADRUNNER-050

Worker taint detendrá nuevas executions.

## DB-ROADRUNNER-051

Local failures se contendrán localmente cuando sea seguro.

## DB-ROADRUNNER-052

Shared-state corruption podrá taint worker.

## DB-ROADRUNNER-053

Tenant isolation será obligatoria.

## DB-ROADRUNNER-054

Security isolation será obligatoria.

## DB-ROADRUNNER-055

Database role será restaurado antes del reuse.

## DB-ROADRUNNER-056

Schema/search path será restaurado.

## DB-ROADRUNNER-057

Session variables serán restauradas.

## DB-ROADRUNNER-058

Timezone overrides serán restaurados.

## DB-ROADRUNNER-059

Isolation overrides serán restaurados.

## DB-ROADRUNNER-060

Temporary state será eliminado o causará discard.

## DB-ROADRUNNER-061

Advisory locks serán liberados o causarán discard.

## DB-ROADRUNNER-062

Prepared statement reuse no implicará binding reuse.

## DB-ROADRUNNER-063

Worker-scoped service no retendrá request EntityManager.

## DB-ROADRUNNER-064

Worker-scoped listener no retendrá request context.

## DB-ROADRUNNER-065

No se usará global mutable current tenant.

## DB-ROADRUNNER-066

No se usará global mutable current transaction.

## DB-ROADRUNNER-067

No se usará global mutable current EntityManager.

## DB-ROADRUNNER-068

Scope generation podrá validar resource ownership.

## DB-ROADRUNNER-069

Stale scoped resource use deberá fallar.

## DB-ROADRUNNER-070

Compiled metadata podrá ser compartida.

## DB-ROADRUNNER-071

Compiled query cache podrá ser compartida.

## DB-ROADRUNNER-072

Managed entities no serán compartidas.

## DB-ROADRUNNER-073

IdentityMap no será tratada como Entity Cache.

## DB-ROADRUNNER-074

Job retry tendrá nuevo scope.

## DB-ROADRUNNER-075

Job retry no reutilizará EntityManager.

## DB-ROADRUNNER-076

Job retry no garantizará exactly-once.

## DB-ROADRUNNER-077

DB commit no equivaldrá a job ACK.

## DB-ROADRUNNER-078

External side effects no serán revertidos por DB rollback.

## DB-ROADRUNNER-079

Outbox podrá utilizarse para coordinación.

## DB-ROADRUNNER-080

Long-running job no implicará long-running transaction.

## DB-ROADRUNNER-081

ORM memory deberá gobernarse explícitamente.

## DB-ROADRUNNER-082

Adapter no ejecutará hidden EntityManager clear durante operaciones.

## DB-ROADRUNNER-083

Configuration será generation-aware.

## DB-ROADRUNNER-084

Metadata será generation-aware.

## DB-ROADRUNNER-085

Credentials serán generation-aware.

## DB-ROADRUNNER-086

Topology será generation-aware.

## DB-ROADRUNNER-087

Mid-execution semantic mutation será evitada.

## DB-ROADRUNNER-088

Credential rotation no migrará transactions.

## DB-ROADRUNNER-089

Writer failover no migrará transactions.

## DB-ROADRUNNER-090

Sticky routing será execution-scoped.

## DB-ROADRUNNER-091

Replica health podrá ser worker-shared.

## DB-ROADRUNNER-092

EntityManager no se considerará concurrent-safe por default.

## DB-ROADRUNNER-093

Parallel DB operations respetarán connection ownership.

## DB-ROADRUNNER-094

Transaction connection no será concurrentemente reutilizada sin soporte explícito.

## DB-ROADRUNNER-095

Telemetry será execution-aware.

## DB-ROADRUNNER-096

N+1 detection será execution-scoped.

## DB-ROADRUNNER-097

Profiler será execution-scoped.

## DB-ROADRUNNER-098

Debug collector será execution-scoped.

## DB-ROADRUNNER-099

Telemetry labels tendrán cardinalidad controlada.

## DB-ROADRUNNER-100

Graceful shutdown detendrá nuevas executions primero.

## DB-ROADRUNNER-101

Active executions serán drenadas antes del pool shutdown.

## DB-ROADRUNNER-102

Shutdown tendrá deadline.

## DB-ROADRUNNER-103

Hung DB operation no bloqueará shutdown indefinidamente.

## DB-ROADRUNNER-104

Forced shutdown podrá producir UNKNOWN outcomes.

## DB-ROADRUNNER-105

Readiness y liveness serán distintas.

## DB-ROADRUNNER-106

DB outage no implicará automáticamente worker death.

## DB-ROADRUNNER-107

Circuit breaker no implicará automáticamente worker taint.

## DB-ROADRUNNER-108

Resource exhaustion aplicará admission control.

## DB-ROADRUNNER-109

Más workers no implicará automáticamente mejor DB throughput.

## DB-ROADRUNNER-110

Configuration validation considerará potential connection demand.

## DB-ROADRUNNER-111

RoadRunner adapter tendrá conformance tests.

## DB-ROADRUNNER-112

HTTP sequential isolation será probado.

## DB-ROADRUNNER-113

Job sequential isolation será probado.

## DB-ROADRUNNER-114

HTTP-to-job isolation será probado.

## DB-ROADRUNNER-115

Job-to-HTTP isolation será probado.

## DB-ROADRUNNER-116

Tenant contamination será probado.

## DB-ROADRUNNER-117

Security contamination será probado.

## DB-ROADRUNNER-118

Transaction leakage será probado.

## DB-ROADRUNNER-119

Connection state leakage será probado.

## DB-ROADRUNNER-120

Cursor leakage será probado.

## DB-ROADRUNNER-121

Memory behavior será probado.

## DB-ROADRUNNER-122

Fault injection será probado.

## DB-ROADRUNNER-123

Worker recycling será probado.

## DB-ROADRUNNER-124

Graceful shutdown será probado.

## DB-ROADRUNNER-125

Credential rotation será probada.

## DB-ROADRUNNER-126

Topology changes serán probados.

## DB-ROADRUNNER-127

Failover será probado.

## DB-ROADRUNNER-128

RoadRunner-specific code no entrará al ORM.

## DB-ROADRUNNER-129

RoadRunner-specific code no entrará al Query Engine.

## DB-ROADRUNNER-130

RoadRunner-specific code no entrará al SQL Compiler.

## DB-ROADRUNNER-131

RoadRunner-specific code no entrará a generic Driver contracts.

## DB-ROADRUNNER-132

RuntimeManagerServer gestionará instalación del runtime.

## DB-ROADRUNNER-133

Database adapter gestionará lifecycle Database.

## DB-ROADRUNNER-134

RoadRunner no cambiará ORM semantics.

## DB-ROADRUNNER-135

RoadRunner no cambiará transaction semantics.

## DB-ROADRUNNER-136

RoadRunner no cambiará cache consistency semantics.

## DB-ROADRUNNER-137

RoadRunner no cambiará persistence consistency semantics.

## DB-ROADRUNNER-138

RoadRunner no cambiará security guarantees.

## DB-ROADRUNNER-139

RoadRunner no cambiará tenant isolation guarantees.

## DB-ROADRUNNER-140

Persistent execution será optimización operacional.

## DB-ROADRUNNER-141

Runtime adapter deberá permanecer delgado.

## DB-ROADRUNNER-142

Common lifecycle deberá residir fuera del adapter.

## DB-ROADRUNNER-143

FrankenPHP y RoadRunner compartirán conformance model.

## DB-ROADRUNNER-144

OpenSwoole deberá poder compartir el mismo conformance model.

## DB-ROADRUNNER-145

Runtime-specific differences no duplicarán Database architecture.

## DB-ROADRUNNER-146

Execution cleanup tendrá prioridad sobre worker reuse.

## DB-ROADRUNNER-147

Isolation tendrá prioridad sobre throughput.

## DB-ROADRUNNER-148

Correctness tendrá prioridad sobre persistent optimization.

## DB-ROADRUNNER-149

UNKNOWN nunca será convertido silenciosamente en SAFE.

## DB-ROADRUNNER-150

Un worker sólo será reutilizable cuando el estado anterior esté limpio o contenido de forma verificable.

---

# 233. Arquitectura completa

```text
                           ROADRUNNER
                               │
                               ▼
                    RoadRunner Worker
                               │
                               ▼
                VoltStack Runtime Integration
                               │
                               ▼
              RoadRunnerDatabaseRuntimeAdapter
                               │
                               ▼
                DatabaseWorkerLifecycleManager
                               │
          ┌────────────────────┴────────────────────┐
          │                                         │
          ▼                                         ▼
Persistent Worker Infrastructure             Execution Admission
          │                                         │
          │                                         ▼
          │                                  Execution Scope
          │                                         │
          │                     ┌───────────────────┼───────────────────┐
          │                     │                   │                   │
          │                     ▼                   ▼                   ▼
          │              DatabaseContext       EntityManager      ResourceRegistry
          │                                         │
          │                                 ┌───────┴───────┐
          │                                 ▼               ▼
          │                            IdentityMap       UnitOfWork
          │
          ├── Driver Registry
          ├── Platform Registry
          ├── Type Registry
          ├── Metadata
          ├── Query Compiler Infrastructure
          ├── Bounded Caches
          ├── Connection Pools
          └── Telemetry Infrastructure
                                                    │
                                                    ▼
                                              Application
                                              / Job Handler
                                                    │
                                                    ▼
                                               Finalization
                                                    │
                     ┌──────────────────────────────┼──────────────────────────────┐
                     │                              │                              │
                     ▼                              ▼                              ▼
              Resolve Transactions          Close Resources                 Clear ORM
                     │                              │                              │
                     └──────────────────────────────┼──────────────────────────────┘
                                                    ▼
                                                   Reset
                                                    │
                                                    ▼
                                                  Verify
                                                    │
                            ┌───────────────────────┴───────────────────────┐
                            │                                               │
                            ▼                                               ▼
                         VERIFIED                                        UNSAFE
                            │                                               │
                            ▼                                               ▼
                     Worker Reusable                              Contain / Taint
                            │                                               │
                            ▼                                               ▼
                     Next Execution                              Drain / Recycle
```

---

# 234. RoadRunner HTTP lifecycle

```text
RoadRunner
    │
    ▼
Receive HTTP Request
    │
    ▼
Create VoltStack Request
    │
    ▼
Create Database Execution Scope
    │
    ▼
Bind DatabaseContext
    │
    ▼
Application
    │
    ├── Query Engine
    ├── ORM
    ├── Transactions
    └── Database Resources
    │
    ▼
Generate HTTP Response
    │
    ▼
Database Finalization
    │
    ▼
State Reset
    │
    ▼
Isolation Verification
    │
    ▼
Return Worker to READY
```

---

# 235. RoadRunner job lifecycle

```text
RoadRunner Jobs
      │
      ▼
Receive Job
      │
      ▼
Create Job Execution Scope
      │
      ▼
Bind DatabaseContext
      │
      ▼
Job Handler
      │
      ├── Queries
      ├── ORM
      ├── Transactions
      └── Side Effects
      │
      ▼
Determine Job Result
      │
      ▼
Database Finalization
      │
      ▼
Reset
      │
      ▼
Verify
      │
      ├── safe ──────► ACK/NEXT
      │
      └── unsafe ────► containment/recycle
```

La coordinación exacta entre ACK y commit pertenecerá a las capas Job/Queue y Transaction, no al adapter RoadRunner.

---

# 236. Comparación conceptual FrankenPHP / RoadRunner

| Área | FrankenPHP | RoadRunner | Database Core |
|---|---|---|---|
| Runtime predeterminado | Sí | No | Independiente |
| Worker persistente | Soportado | Soportado | Abstraído |
| Request Scope | Sí | Sí | Común |
| Job Scope | Mediante integración superior | Importante | Común |
| EntityManager scoped | Sí | Sí | Obligatorio |
| IdentityMap scoped | Sí | Sí | Obligatorio |
| Connection reuse | Sí | Sí | Común |
| Reset | Sí | Sí | Común |
| Isolation verification | Sí | Sí | Común |
| Worker recycling | Adapter | Adapter | Política común |
| ORM especial | No | No | Uno solo |
| Query Engine especial | No | No | Uno solo |
| SQL Compiler especial | No | No | Uno solo |

---

# 237. Regla de diseño de adapters

Idealmente:

```text
90–95%
```

de la lógica persistent-runtime deberá permanecer compartida.

El adapter específico sólo deberá resolver diferencias reales del runtime.

Conceptualmente:

```text
RoadRunner Event
      ↓
Small Adapter
      ↓
Generic Database Lifecycle
```

---

# 238. Resultado arquitectónico

La incorporación de RoadRunner no deberá producir:

```text
VoltStack Database for FrankenPHP
VoltStack Database for RoadRunner
```

Deberá producir:

```text
VoltStack Database
        │
        └── Runtime Adapters
             ├── FrankenPHP
             ├── RoadRunner
             └── OpenSwoole
```

---

# 239. Principio final

El modelo RoadRunner deberá seguir:

```text
BOOT WORKER INFRASTRUCTURE ONCE
```

```text
CREATE EXECUTION SCOPE PER UNIT OF WORK
```

```text
BIND DATABASE CONTEXT EXPLICITLY
```

```text
ACQUIRE CONNECTIONS THROUGH LEASES
```

```text
NEVER SHARE ORM MUTABLE STATE
```

```text
FINALIZE EVERY EXECUTION
```

```text
RESET EVERY EXECUTION
```

```text
VERIFY BEFORE REUSE
```

```text
DISCARD UNSAFE CONNECTIONS
```

```text
TAINT UNSAFE WORKERS
```

```text
RECYCLE WHEN SHARED STATE CANNOT BE TRUSTED
```

La regla fundamental será:

> **RoadRunner permitirá que VoltStack conserve infraestructura costosa entre ejecuciones, pero ninguna optimización de worker persistente podrá convertir el estado mutable de una request o job en estado compartido del proceso.**

---

# 240. Estado del Bloque 25

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
✓ 259_DATABASE_ROADRUNNER_INTEGRATION_SYSTEM.md
│
└── 260_DATABASE_OPENSWOOLE_INTEGRATION_SYSTEM.md
```

---

# 241. Siguiente documento

```text
260_DATABASE_OPENSWOOLE_INTEGRATION_SYSTEM.md
```

El siguiente documento completará el Bloque 25 definiendo la integración con OpenSwoole, donde deberán analizarse especialmente:

```text
OpenSwoole Server Lifecycle
Persistent Workers
Coroutines
Coroutine Context
Concurrent Requests
Database Execution Scope
Coroutine-local DatabaseContext
EntityManager isolation
IdentityMap isolation
UnitOfWork isolation
Connection Pools
Coroutine-aware connection leasing
Transaction affinity
Connection reuse
State reset
Worker lifecycle
Memory management
Backpressure
Cancellation
Timeouts
Worker recycling
Telemetry
Graceful shutdown
```

La diferencia arquitectónica principal será que OpenSwoole hace todavía más importante la separación entre:

```text
Worker
≠
Request
≠
Coroutine
≠
Database Execution
≠
Transaction
≠
Connection
```

por lo que la siguiente especificación extenderá el modelo persistent-runtime hacia concurrencia y contexto coroutine-aware sin modificar las invariantes fundamentales de VoltStack Database.