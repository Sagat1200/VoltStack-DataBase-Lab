# 257_DATABASE_WORKER_LIFECYCLE_SYSTEM.md

# VoltStack Quantum Database
## Database Worker Lifecycle System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 257 — Database Worker Lifecycle System  
**Bloque:** 25 — Persistent Runtime  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `256_DATABASE_CONNECTION_REUSE_SYSTEM.md`  
**Siguiente documento:** `258_DATABASE_FRANKENPHP_INTEGRATION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura del **Database Worker Lifecycle System** de VoltStack.

Su responsabilidad es gobernar cómo el subsistema Database vive dentro de procesos persistentes capaces de ejecutar múltiples:

- requests HTTP;
- jobs;
- comandos;
- tareas programadas;
- mensajes;
- eventos;
- Fibers;
- coroutines;
- operaciones internas.

La regla fundamental será:

> **Un worker VoltStack podrá procesar múltiples ejecuciones únicamente mientras su infraestructura compartida permanezca válida y cada ejecución anterior haya liberado, restaurado o aislado completamente todo estado que no deba atravesar el siguiente execution boundary.**

Formalmente:

```text
WorkerReusable(W)
=
InfrastructureHealthy(W)
∧ SharedStateValid(W)
∧ PreviousExecutionFinalized(W)
∧ NoLeakedExecutionState(W)
∧ ResourceBudgetsSatisfied(W)
∧ ¬WorkerTainted(W)
```

Por tanto:

```text
ProcessAlive
≠
WorkerHealthy
≠
WorkerReusable
```

---

# 2. Contexto

En PHP tradicional:

```text
BOOT
 ↓
REQUEST
 ↓
RESPONSE
 ↓
PROCESS TERMINATES
```

La terminación del proceso actúa como mecanismo natural de cleanup.

En un runtime persistente:

```text
BOOT
 ↓
REQUEST A
 ↓
RESET
 ↓
REQUEST B
 ↓
RESET
 ↓
JOB C
 ↓
RESET
 ↓
REQUEST D
 ↓
...
```

el proceso continúa vivo.

Esto cambia radicalmente la arquitectura del framework.

---

# 3. Problema central

Sin aislamiento explícito:

```text
Execution A
   │
   ├── EntityManager
   ├── IdentityMap
   ├── UnitOfWork
   ├── Transaction
   ├── Tenant
   ├── ConnectionLease
   └── Query Context
          │
          ▼
      Execution B
```

podría provocar contaminación entre ejecuciones.

Esto es inaceptable.

La arquitectura correcta será:

```text
Worker
│
├── Shared immutable/bounded infrastructure
│
├── Execution A Scope
│   └── destroyed/reset
│
├── Execution B Scope
│   └── destroyed/reset
│
└── Execution C Scope
```

---

# 4. Relación con documentos anteriores

Este documento especializa principalmente:

```text
251_DATABASE_PERSISTENT_RUNTIME_ARCHITECTURE.md
252_DATABASE_REQUEST_SCOPE_SYSTEM.md
253_DATABASE_DATABASE_CONTEXT_SYSTEM.md
254_DATABASE_STATE_ISOLATION_SYSTEM.md
255_DATABASE_STATE_RESET_SYSTEM.md
256_DATABASE_CONNECTION_REUSE_SYSTEM.md
```

y se integra con:

```text
12_DATABASE_CONNECTION_MANAGER.md
14_DATABASE_CONNECTION_POOLING_SYSTEM.md
15_DATABASE_CONNECTION_LIFECYCLE_SYSTEM.md
16_DATABASE_CONNECTION_STATE_AND_RESET_SYSTEM.md
123_DATABASE_IDENTITY_MAP_SYSTEM.md
124_DATABASE_UNIT_OF_WORK_ARCHITECTURE.md
164_DATABASE_TRANSACTION_ARCHITECTURE.md
216_DATABASE_TELEMETRY_ARCHITECTURE.md
235_DATABASE_RESILIENCE_ARCHITECTURE.md
241_DATABASE_RESOURCE_EXHAUSTION_PROTECTION_SYSTEM.md
248_DATABASE_MEMORY_MANAGEMENT_SYSTEM.md
249_DATABASE_RESOURCE_GOVERNANCE_SYSTEM.md
```

---

# 5. Objetivos

El sistema deberá definir:

1. worker identity;
2. worker boot;
3. initialization;
4. readiness;
5. execution admission;
6. execution scopes;
7. execution generations;
8. shared infrastructure;
9. mutable state isolation;
10. concurrent executions;
11. execution finalization;
12. cleanup;
13. reset;
14. verification;
15. worker tainting;
16. worker health;
17. worker degradation;
18. draining;
19. recycling;
20. graceful shutdown;
21. forced shutdown;
22. configuration reload;
23. credential rotation;
24. topology changes;
25. memory governance;
26. connection pool lifecycle;
27. telemetry;
28. diagnostics;
29. testing;
30. runtime-neutral contracts.

---

# 6. No objetivos

Este sistema no define directamente:

```text
HTTP routing
HTTP response generation
job queue protocol
FrankenPHP internals
RoadRunner protocol
OpenSwoole coroutine implementation
SQL compilation
ORM mapping
```

Estos sistemas se integrarán mediante adapters.

---

# 7. Worker

Un `DatabaseWorker` representa el contexto persistente dentro del cual Database puede atender múltiples ejecuciones.

Conceptualmente:

```php
interface DatabaseWorker
{
    public function id(): WorkerId;

    public function generation(): WorkerGeneration;

    public function state(): WorkerState;

    public function health(): WorkerHealth;
}
```

---

# 8. Worker ≠ Process

Aunque normalmente exista relación:

```text
1 OS process
→ 1 worker
```

VoltStack no deberá codificar esta suposición en el core.

Podrían existir runtimes donde:

```text
Process
├── Worker A
└── Worker B
```

o abstracciones equivalentes.

---

# 9. Worker ≠ Execution

Regla:

```text
Worker
≠
Request
≠
Job
≠
Command
≠
Execution Scope
```

---

# 10. Execution

Una `DatabaseExecution` representa una unidad lógica de trabajo atendida dentro del worker.

Ejemplos:

```text
HTTP request
queue job
scheduled task
message
CLI operation
RPC call
```

---

# 11. ExecutionKind

```php
enum DatabaseExecutionKind
{
    case HTTP_REQUEST;
    case JOB;
    case COMMAND;
    case SCHEDULED_TASK;
    case MESSAGE;
    case INTERNAL;
}
```

---

# 12. Worker state machine

Se propone:

```text
CREATED
   ↓
BOOTING
   ↓
INITIALIZING
   ↓
READY
   ↓
RUNNING
   ↓
READY
```

con ciclos repetidos:

```text
READY
 ↓
RUNNING
 ↓
FINALIZING
 ↓
RESETTING
 ↓
VERIFYING
 ↓
READY
```

y estados terminales/intermedios:

```text
DEGRADED
TAINTED
DRAINING
STOPPING
STOPPED
FAILED
```

---

# 13. WorkerState

```php
enum WorkerState
{
    case CREATED;
    case BOOTING;
    case INITIALIZING;
    case READY;
    case RUNNING;
    case DEGRADED;
    case TAINTED;
    case DRAINING;
    case STOPPING;
    case STOPPED;
    case FAILED;
}
```

---

# 14. Concurrency caveat

En runtimes concurrentes:

```text
READY → RUNNING → READY
```

no describe suficientemente el número de ejecuciones.

Por ello:

```text
WorkerState
```

representa el estado administrativo del worker, mientras:

```text
ActiveExecutionCount
```

representa concurrencia.

---

# 15. Worker health

Separar:

```text
WorkerState
```

de:

```text
WorkerHealth
```

---

# 16. WorkerHealth

```php
enum WorkerHealth
{
    case HEALTHY;
    case DEGRADED;
    case UNHEALTHY;
    case UNKNOWN;
}
```

---

# 17. State ≠ Health

Un worker puede estar:

```text
state = READY
health = DEGRADED
```

por ejemplo si una replica dejó de estar disponible pero el writer sigue operativo.

---

# 18. Worker identity

Cada worker tendrá:

```text
WorkerId
WorkerGeneration
BootId
```

---

# 19. WorkerGeneration

Una nueva instancia lógica del worker deberá recibir una nueva generación.

Esto evita confundir:

```text
Worker #4 before recycle
```

con:

```text
Worker #4 after recycle
```

---

# 20. BootId

Cada boot podrá tener un identificador único para:

- telemetry;
- diagnostics;
- leak detection;
- resource ownership;
- debugging.

---

# 21. Worker boot

Pipeline conceptual:

```text
Process/Runtime Starts
        ↓
Create Worker Identity
        ↓
Load Database Configuration
        ↓
Initialize Platform Registry
        ↓
Initialize Driver Registry
        ↓
Initialize Metadata Infrastructure
        ↓
Initialize Query Infrastructure
        ↓
Initialize Connection Infrastructure
        ↓
Initialize Pools
        ↓
Register Reset Participants
        ↓
Register Lifecycle Hooks
        ↓
Validate Required Capabilities
        ↓
READY
```

---

# 22. Boot failure

Si un componente obligatorio falla:

```text
BOOTING
   ↓
FAILED
```

El worker no deberá aceptar ejecuciones.

---

# 23. Partial boot

VoltStack no deberá marcar:

```text
READY
```

si la inicialización requerida está incompleta.

---

# 24. Lazy initialization

Algunos recursos podrán inicializarse lazily.

Ejemplo:

```text
connection pool
→ no physical connections until first demand
```

Esto no impide declarar el worker READY si la infraestructura necesaria para crear esos recursos ya fue validada.

---

# 25. Boot validation

Podrá distinguir:

```text
structural validation
runtime validation
connectivity validation
```

---

# 26. Connectivity at boot

No siempre será obligatorio abrir una conexión al DB durante boot.

Configurable:

```php
enum DatabaseBootConnectivityPolicy
{
    case NONE;
    case VALIDATE_CONFIGURATION;
    case CONNECT_PRIMARY;
    case CONNECT_REQUIRED_ENDPOINTS;
    case CUSTOM;
}
```

---

# 27. Boot vs readiness

```text
BootComplete
≠
DatabaseReachable
```

según política.

---

# 28. Worker shared infrastructure

Puede sobrevivir entre ejecuciones:

```text
immutable configuration snapshots
driver registry
dialect registry
platform capabilities
compiled metadata
metadata cache
query compilation cache
immutable compiler structures
connection pools
telemetry instruments
immutable mappings
```

---

# 29. Execution-scoped state

No deberá sobrevivir normalmente:

```text
DatabaseContext
EntityManager
IdentityMap
UnitOfWork
TransactionContext
ConnectionLease
tenant context
authorization query scope
query execution context
hydration session
relationship load session
pagination traversal state
chunk continuation
lazy iterator
active result cursor
```

---

# 30. Regla fundamental

```text
Reusable Infrastructure
≠
Reusable Execution State
```

---

# 31. Classification

Todo componente Database deberá clasificarse como:

```php
enum DatabaseLifetime
{
    case PROCESS;
    case WORKER;
    case EXECUTION;
    case OPERATION;
    case TRANSACTION;
    case CONNECTION;
}
```

---

# 32. Lifetime declaration

Los servicios críticos deberán declarar su lifetime mediante Container metadata o contrato equivalente.

---

# 33. Default rule

Si un componente contiene estado mutable relacionado con:

```text
tenant
transaction
entity
query
request
job
connection lease
```

no deberá ser `WORKER` por default.

---

# 34. Execution admission

Antes de crear un execution scope:

```text
CanAcceptExecution(worker, request)?
```

deberá evaluarse.

---

# 35. Admission conditions

Normalmente:

```text
WorkerState ∈ {READY, RUNNING}
∧ HealthAcceptable
∧ ¬Draining
∧ ConcurrencyBudgetAvailable
∧ MemoryBudgetAvailable
∧ ResourceBudgetAvailable
```

---

# 36. Admission result

```php
enum ExecutionAdmissionDecision
{
    case ACCEPT;
    case WAIT;
    case REJECT;
    case RECYCLE_REQUIRED;
}
```

---

# 37. Worker draining

Un worker en:

```text
DRAINING
```

no aceptará nuevas ejecuciones.

---

# 38. Existing executions

Podrán finalizar dentro de un deadline.

---

# 39. Execution generation

Cada ejecución tendrá:

```text
ExecutionId
ExecutionGeneration
```

---

# 40. Why generation

Protege contra referencias asíncronas tardías.

Ejemplo:

```text
Execution A ends

late callback from A fires

Execution B already running
```

El callback de A no deberá operar sobre B.

---

# 41. Scope token

Se propone:

```text
ExecutionScopeToken
```

ligado a:

```text
WorkerGeneration
ExecutionGeneration
```

---

# 42. Token validation

Antes de usar estado scope-bound:

```text
token.workerGeneration == currentWorkerGeneration
AND
token.executionGeneration == owningExecutionGeneration
```

---

# 43. Stale scope

Una referencia a un scope finalizado deberá fallar.

---

# 44. Scope creation

Al aceptar ejecución:

```text
Admission
   ↓
Create ExecutionScope
   ↓
Create DatabaseContext
   ↓
Resolve Tenant/Domain
   ↓
Create EntityManager
   ↓
Create IdentityMap
   ↓
Create UnitOfWork
   ↓
Create TransactionContext
   ↓
Create ResourceRegistry
   ↓
RUN
```

---

# 45. Scope ownership

Todo recurso mutable adquirido durante la ejecución deberá tener ownership rastreable.

---

# 46. ResourceRegistry

Cada scope tendrá un registro conceptual:

```text
ExecutionResourceRegistry
```

---

# 47. Registered resources

Ejemplos:

```text
ConnectionLease
ResultCursor
StreamingResult
TransactionContext
LazyIterator
Temporary Resource
HydrationSession
ChunkTraversal
```

---

# 48. Resource registry ≠ GC

El Garbage Collector no sustituye lifecycle management.

---

# 49. Explicit ownership

```text
resource acquired
→ register
```

```text
resource released
→ unregister
```

---

# 50. Finalization safety

Al terminar ejecución:

```text
registry
```

permite detectar recursos aún activos.

---

# 51. Execution lifecycle

```text
CREATED
 ↓
INITIALIZING
 ↓
ACTIVE
 ↓
FINALIZING
 ↓
RESETTING
 ↓
VERIFIED
 ↓
CLOSED
```

Alternativas:

```text
FAILED
CANCELLED
TAINTED
UNKNOWN
```

---

# 52. ExecutionState

```php
enum DatabaseExecutionState
{
    case CREATED;
    case INITIALIZING;
    case ACTIVE;
    case FINALIZING;
    case RESETTING;
    case VERIFIED;
    case CLOSED;
    case FAILED;
    case CANCELLED;
    case TAINTED;
    case UNKNOWN;
}
```

---

# 53. Execution result ≠ cleanup result

Una request puede terminar exitosamente pero cleanup fallar.

Ejemplo:

```text
HTTP 200 generated
+
connection reset failed
```

Por tanto:

```text
BusinessOutcome
≠
FinalizationOutcome
```

---

# 54. Finalization pipeline

Al terminar una ejecución:

```text
Stop New DB Operations
        ↓
Cancel/Complete Pending Operations
        ↓
Close Results/Cursors
        ↓
Resolve Transactions
        ↓
Flush Policy Check
        ↓
Release Connection Leases
        ↓
Clear ORM State
        ↓
Reset DatabaseContext
        ↓
Reset Registered Components
        ↓
Verify Isolation
        ↓
Close Scope
```

---

# 55. Flush on finalization

VoltStack no deberá ejecutar automáticamente:

```php
$entityManager->flush();
```

simplemente porque terminó un request.

---

# 56. Why

Una mutación no persistida puede representar:

- error de aplicación;
- trabajo incompleto;
- transaction fallida;
- operación cancelada.

Auto-flush sería peligroso.

---

# 57. Transaction finalization

Si existe transaction activa al terminar execution:

default:

```text
ROLLBACK
```

si el outcome es conocido y la conexión permite hacerlo.

---

# 58. Active transaction leak

Nunca:

```text
Execution A ends with transaction
       ↓
Execution B inherits transaction
```

---

# 59. Unknown transaction state

Si no puede verificarse rollback:

```text
connection → discard/quarantine
execution → TAINTED/UNKNOWN
```

según evidencia.

---

# 60. EntityManager finalization

Default:

```text
clear
close
discard scope
```

---

# 61. IdentityMap finalization

```text
clear all managed references
```

---

# 62. UnitOfWork finalization

Eliminar:

```text
new entities
dirty entities
removed entities
snapshots
change sets
pending persistence plans
```

---

# 63. Query state finalization

Eliminar:

```text
current query
bindings
query execution context
query timeout state
cancellation state
temporary optimizer/planner state
```

---

# 64. Tenant finalization

Tenant context deberá desaparecer con el scope.

---

# 65. Authorization scope

También deberá desaparecer.

---

# 66. Request-specific cache

Caches L0/request-local deberán vaciarse.

---

# 67. Shared caches

Caches worker/globales no deberán borrarse indiscriminadamente.

---

# 68. Reset precision

Regla:

```text
reset execution state
≠
flush all shared caches
```

---

# 69. Reset participants

Los componentes podrán implementar:

```php
interface DatabaseExecutionResetParticipant
{
    public function reset(DatabaseResetContext $context): void;
}
```

---

# 70. Verification participants

Separado:

```php
interface DatabaseExecutionIsolationVerifier
{
    public function verify(
        DatabaseExecutionVerificationContext $context
    ): DatabaseIsolationVerificationResult;
}
```

---

# 71. Reset ≠ Verify

Crítico:

```text
ResetCalled
≠
ResetSucceeded
≠
IsolationVerified
```

---

# 72. Verification result

```php
enum IsolationVerificationStatus
{
    case VERIFIED;
    case VERIFIED_WITH_WARNINGS;
    case FAILED;
    case UNKNOWN;
}
```

---

# 73. UNKNOWN

En persistent runtime:

```text
UNKNOWN
```

no deberá interpretarse como limpio.

---

# 74. Worker tainting

Si una ejecución deja estado cuyo aislamiento no puede garantizarse:

```text
worker → TAINTED
```

cuando el problema pueda afectar infraestructura compartida.

---

# 75. Execution taint ≠ Worker taint

Si sólo una conexión está dañada:

```text
discard connection
```

puede ser suficiente.

No es necesario reciclar worker.

---

# 76. Taint classification

Se propone:

```php
enum WorkerTaintSeverity
{
    case RESOURCE_LOCAL;
    case EXECUTION_LOCAL;
    case SHARED_COMPONENT;
    case WORKER_GLOBAL;
    case UNKNOWN;
}
```

---

# 77. Resource-local taint

Ejemplo:

```text
connection reset failed
```

Solución:

```text
discard connection
```

Worker puede continuar.

---

# 78. Execution-local taint

Ejemplo:

```text
EntityManager corrupted
```

si está completamente scope-bound:

```text
destroy scope
```

Worker puede continuar.

---

# 79. Shared-component taint

Ejemplo:

```text
mutable shared metadata registry corrupted
```

puede requerir:

```text
worker recycle
```

---

# 80. Unknown taint

Default conservador:

```text
drain worker
→ recycle
```

---

# 81. Worker taint reasons

```text
shared mutable state corruption
scope isolation verification failure
unrecoverable resource leak
configuration generation inconsistency
runtime integration failure
memory corruption symptoms
unbounded retained execution state
unknown cross-request contamination
```

---

# 82. Taint response

```text
TAINTED
   ↓
stop new admissions
   ↓
DRAINING
   ↓
finish active executions
   ↓
STOPPING
   ↓
recycle
```

---

# 83. Worker degradation

No todos los problemas requieren recycle.

Ejemplo:

```text
replica unavailable
```

puede producir:

```text
health = DEGRADED
```

mientras writer funciona.

---

# 84. Degraded policy

Puede:

```text
continue
reduce capabilities
route differently
reject specific operation classes
```

---

# 85. Worker health evaluator

```php
interface DatabaseWorkerHealthEvaluator
{
    public function evaluate(
        DatabaseWorkerSnapshot $worker
    ): DatabaseWorkerHealthResult;
}
```

---

# 86. Health dimensions

Podrán incluir:

```text
memory
connection pools
endpoint availability
reset failures
leak count
taint status
error rate
configuration freshness
credential freshness
topology freshness
```

---

# 87. Worker recycle

Recycle significa:

```text
stop accepting work
drain
shutdown resources
terminate/recreate worker
```

---

# 88. Recycle reasons

```text
max requests
max jobs
max lifetime
memory threshold
memory growth
taint
configuration change
credential policy
deployment
operator request
health degradation
runtime request
```

---

# 89. Max executions

Podrá configurarse:

```text
worker.max_executions
```

---

# 90. Max lifetime

También:

```text
worker.max_lifetime
```

---

# 91. Memory threshold

```text
worker.max_memory
```

---

# 92. Why recycle

Incluso con cleanup correcto, recycling periódico puede limitar:

- fragmentation;
- extension leaks;
- driver leaks;
- third-party library leaks;
- unexpected retained references.

---

# 93. Recycling ≠ correctness substitute

No deberá usarse como excusa para permitir contaminación.

```text
"We recycle every 500 requests"
```

no hace aceptable un IdentityMap global.

---

# 94. Memory baseline

Después de ejecuciones podrá observarse:

```text
post-reset memory
```

---

# 95. Memory drift

Sea:

```text
Mₙ = memory after reset of execution n
```

crecimiento persistente:

```text
Mₙ₊₁ > Mₙ
```

puede indicar retención.

---

# 96. Memory trend

No todo crecimiento es leak debido a:

- caches bounded;
- allocator behavior;
- JIT/runtime structures;
- compiled metadata.

Por ello se analizará tendencia y budgets.

---

# 97. Memory categories

Idealmente:

```text
shared bounded cache
connection resources
execution state
unknown retained memory
```

---

# 98. Execution budget

Cada ejecución podrá tener:

```php
final readonly class DatabaseExecutionBudget
{
    public function __construct(
        public ?Duration $deadline,
        public ?int $maxQueries,
        public ?int $maxConnections,
        public ?int $maxOpenCursors,
        public ?int $maxHydratedEntities,
        public ?int $maxMemoryBytes,
    ) {}
}
```

---

# 99. Worker budget

```php
final readonly class DatabaseWorkerBudget
{
    public function __construct(
        public int $maxConcurrentExecutions,
        public int $maxOpenConnections,
        public int $maxPendingAcquisitions,
        public int $maxMemoryBytes,
    ) {}
}
```

---

# 100. Execution count

El worker deberá conocer:

```text
accepted
active
completed
failed
cancelled
```

de forma bounded.

No guardar cada execution object históricamente.

---

# 101. Concurrent executions

FrankenPHP/RoadRunner/OpenSwoole pueden presentar distintos modelos de concurrencia.

El core deberá asumir:

```text
multiple active execution scopes may exist
```

aunque un adapter concreto sea secuencial.

---

# 102. No global current request

Prohibido:

```php
static $currentDatabaseContext;
```

---

# 103. Scope resolver

Se utilizará un mecanismo runtime-aware:

```text
ExecutionScopeResolver
```

---

# 104. Scope resolver contract

```php
interface DatabaseExecutionScopeResolver
{
    public function current(): DatabaseExecutionScope;
}
```

---

# 105. Runtime implementation

Puede basarse en:

```text
request-local context
Fiber local storage
coroutine context
runtime context ID
container scope
```

según adapter.

---

# 106. Context locality

```text
Execution A
→ Context A

Execution B
→ Context B
```

aunque corran simultáneamente.

---

# 107. Fiber safety

El contexto no deberá depender exclusivamente de:

```text
thread local
```

porque PHP puede ejecutar múltiples Fibers en un mismo thread.

---

# 108. Coroutine safety

Mismo principio para OpenSwoole.

---

# 109. Async continuation

Una continuación deberá conservar su scope token correcto.

---

# 110. Detached async work

Trabajo que deba sobrevivir al request deberá convertirse en:

```text
job/message/task
```

con un nuevo execution scope.

---

# 111. No scope escape

No deberá capturarse:

```text
EntityManager
ConnectionLease
DatabaseContext
```

para utilizarlo después del cierre de la ejecución.

---

# 112. Scope escape detection

Development tooling podrá detectar referencias utilizadas después del cierre mediante generation tokens.

---

# 113. Connection pool lifetime

Los pools normalmente serán:

```text
WORKER
```

---

# 114. Physical connection lifetime

Puede cruzar execution boundaries sólo mediante:

```text
release
reset
verify
reuse
```

---

# 115. Lease lifetime

Será:

```text
EXECUTION / TRANSACTION / OPERATION
```

según necesidad.

Nunca worker-global por default.

---

# 116. Connection pool boot

Durante boot:

```text
create pool structures
```

pero no necesariamente physical connections.

---

# 117. Pool during execution

```text
acquire
lease
release
```

---

# 118. Pool during drain

```text
reject new acquisitions from new executions
allow active leases to finish
```

---

# 119. Pool shutdown

```text
close idle
wait for active
force close after deadline
```

---

# 120. Connection drain

Cada pool podrá entrar en:

```text
DRAINING
```

independientemente.

---

# 121. Endpoint drain

Útil durante:

```text
failover
maintenance
credential rotation
deployment
```

---

# 122. Worker drain ≠ Pool drain

Un pool específico puede drenarse sin detener todo el worker.

---

# 123. Configuration snapshot

Cada execution deberá observar una configuración coherente.

---

# 124. Config generation

```text
ConfigurationGeneration
```

identificará la versión activa.

---

# 125. Config reload

No deberá mutarse arbitrariamente configuración compartida mientras una ejecución la utiliza.

---

# 126. Reload model

Preferir:

```text
Generation N
      ↓
new immutable configuration
      ↓
Generation N+1
```

---

# 127. Existing executions

Pueden continuar con:

```text
Generation N
```

si es seguro.

---

# 128. New executions

Usarán:

```text
Generation N+1
```

---

# 129. Incompatible reload

Si no puede coexistir:

```text
worker drain
→ recycle
```

---

# 130. Metadata generation

Mapping/metadata compilada también podrá tener:

```text
MetadataGeneration
```

---

# 131. Query cache generation

Compiled query caches deberán respetar generaciones relevantes.

---

# 132. No mixed incompatible generations

Una query no deberá combinar:

```text
metadata generation 10
+
compiler assumptions generation 12
```

si son incompatibles.

---

# 133. Credential rotation

Credential provider podrá publicar nueva generación.

---

# 134. Existing connections

Según policy:

```text
continue
drain
retire
reauthenticate
```

---

# 135. New connections

Usarán credenciales actuales.

---

# 136. Credentials and execution scopes

Passwords/tokens no deberán copiarse innecesariamente al execution scope.

---

# 137. Secret lifecycle

Secrets permanecerán en proveedores/estructuras protegidas de infraestructura.

---

# 138. Topology changes

Cambios en:

```text
writer
replicas
shards
endpoints
```

crearán nueva:

```text
TopologyGeneration
```

---

# 139. Existing execution routing

Una transaction activa conservará su connection affinity.

---

# 140. New operations

Fuera de transaction podrán usar nueva topología según consistencia y routing policy.

---

# 141. Failover

Worker no deberá reciclarse necesariamente por un failover normal.

Idealmente:

```text
topology update
→ pools update
→ stale connections retire
→ routing adapts
```

---

# 142. Unknown topology

Si la autoridad no puede determinarse:

```text
writes may be rejected
```

en lugar de asumir writer.

---

# 143. Worker and cache

Caches compartidas deberán ser:

```text
bounded
version-aware
safe for concurrent access
free of execution-specific mutable state
```

---

# 144. Metadata cache

Buen candidato para worker lifetime.

---

# 145. Query compilation cache

También.

---

# 146. Result cache

Su lifetime depende del provider, pero no deberá guardar objetos managed del EntityManager.

---

# 147. Entity cache

No deberá almacenar managed entity instances entre requests.

---

# 148. IdentityMap

Siempre execution-scoped.

---

# 149. Static properties

Las clases Database deberán evitar static mutable state.

---

# 150. Allowed static state

Sólo datos verdaderamente:

```text
immutable
constant
pure
```

---

# 151. Static caches

No recomendados salvo infraestructura explícitamente diseñada, bounded y resettable/versioned.

---

# 152. Event listeners

Listeners worker-shared no deberán capturar execution objects.

---

# 153. Closure capture risk

Ejemplo peligroso:

```php
$dispatcher->listen(function () use ($entityManager) {
    // ...
});
```

si el dispatcher es worker-shared.

---

# 154. Listener registration lifetime

Listeners estáticos:

```text
registered at boot
```

Listeners scope-specific:

```text
registered inside execution scope
removed at finalization
```

---

# 155. Telemetry context

Tracing deberá separar:

```text
worker identity
execution identity
transaction identity
connection identity
```

---

# 156. Telemetry context cleanup

Execution trace/span context no cruzará requests.

---

# 157. Logging context

Igual:

```text
request_id
tenant
user
job_id
```

deberá resetearse.

---

# 158. Database event context

Events deberán recibir contexto explícito o scope-resolved correctamente.

---

# 159. Background telemetry exporter

Puede ser worker-shared si no conserva scope mutable incorrectamente.

---

# 160. Finalization failure hierarchy

```text
DatabaseExecutionFinalizationException
├── PendingOperationFinalizationException
├── ActiveCursorFinalizationException
├── TransactionFinalizationException
├── ConnectionLeaseFinalizationException
├── OrmFinalizationException
├── StateResetException
├── IsolationVerificationException
└── UnknownFinalizationStateException
```

---

# 161. Finalization aggregation

Cleanup deberá intentar liberar recursos independientes aun cuando uno falle.

Ejemplo:

```text
cursor close fails
```

no debería impedir intentar:

```text
rollback transaction
clear EntityManager
release other connections
```

---

# 162. Error aggregation

Podrá producirse:

```text
DatabaseFinalizationReport
```

con múltiples fallos.

---

# 163. Finalization ordering

El orden importa.

No:

```text
release connection
↓
then rollback transaction
```

---

# 164. Correct order

Conceptualmente:

```text
stop work
↓
resolve active operations
↓
resolve transaction
↓
close dependent resources
↓
release/reset connections
↓
clear higher-level state
↓
verify
```

El orden concreto podrá ajustarse según dependency graph.

---

# 165. Cleanup dependency graph

Cada reset participant podrá declarar:

```text
dependencies
priority
phase
```

---

# 166. Reset phases

Propuesta:

```php
enum DatabaseResetPhase
{
    case STOP_OPERATIONS;
    case FINALIZE_RESULTS;
    case FINALIZE_TRANSACTIONS;
    case RELEASE_CONNECTIONS;
    case CLEAR_ORM;
    case CLEAR_CONTEXT;
    case VERIFY;
}
```

---

# 167. Reset ordering

Se resolverá como DAG cuando existan dependencias.

---

# 168. Circular reset dependency

Deberá detectarse durante boot.

---

# 169. Finalization deadline

Cleanup no deberá bloquear indefinidamente.

---

# 170. Deadline

```text
Execution Deadline
```

y:

```text
Finalization Deadline
```

pueden ser distintos.

---

# 171. Cleanup budget

Ejemplo:

```text
request execution timeout = 30s
cleanup grace = 2s
```

---

# 172. Cleanup deadline exceeded

Escalar:

```text
cancel
discard resources
taint worker if necessary
```

---

# 173. Cancellation

Cancellation de ejecución deberá propagarse a:

```text
query executor
streaming results
chunk processing
lazy collections
connection acquisition
```

---

# 174. Cancellation ≠ cleanup

Después de cancelar aún debe ejecutarse finalization.

---

# 175. Fatal errors

El runtime adapter deberá intentar lifecycle hooks cuando sea posible.

Pero el sistema no deberá depender de que todo fatal error permita cleanup PHP normal.

---

# 176. Worker process termination

La terminación del proceso sigue siendo última barrera de aislamiento.

Pero no la barrera normal entre requests.

---

# 177. Shutdown hooks

Se podrán registrar hooks para:

```text
drain pools
flush telemetry
close connections
```

---

# 178. Shutdown ordering

```text
stop admissions
↓
drain executions
↓
close DB pools
↓
flush bounded telemetry
↓
stop worker
```

---

# 179. Graceful shutdown

Estados:

```text
READY/RUNNING
      ↓
DRAINING
      ↓
STOPPING
      ↓
STOPPED
```

---

# 180. Drain deadline

Debe ser configurable.

---

# 181. Deadline expiration

Operaciones restantes podrán:

```text
cancel
rollback
close
discard
```

según semántica.

---

# 182. Forced shutdown

No prometerá completion de operaciones interrumpidas.

---

# 183. UNKNOWN outcomes

Si proceso muere durante commit:

```text
transaction outcome may be UNKNOWN
```

y deberá ser tratado por recovery/idempotency a nivel superior.

---

# 184. Shutdown ≠ rollback guarantee

Un kill abrupto no garantiza que la DB haya hecho rollback antes de ejecutar commit.

---

# 185. Worker lifecycle manager

Contrato:

```php
interface DatabaseWorkerLifecycleManager
{
    public function boot(): void;

    public function admit(
        DatabaseExecutionRequest $request
    ): DatabaseExecutionScope;

    public function finalize(
        DatabaseExecutionScope $scope
    ): DatabaseFinalizationReport;

    public function drain(): void;

    public function shutdown(): void;
}
```

---

# 186. Runtime adapter

```php
interface DatabaseRuntimeAdapter
{
    public function runtime(): DatabaseRuntime;

    public function currentExecutionId(): ?string;

    public function registerWorkerHooks(
        DatabaseWorkerLifecycleManager $manager
    ): void;
}
```

---

# 187. Runtime enum

```php
enum DatabaseRuntime
{
    case CLASSIC_PHP;
    case FRANKENPHP;
    case ROADRUNNER;
    case OPENSWOOLE;
    case CUSTOM;
}
```

---

# 188. Core independence

El Database core no deberá importar clases de:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

directamente.

---

# 189. Integration packages/adapters

Se utilizarán:

```text
Quantum/Database/Runtime/FrankenPHP
Quantum/Database/Runtime/RoadRunner
Quantum/Database/Runtime/OpenSwoole
```

o packages equivalentes.

---

# 190. FrankenPHP preparation

El adapter deberá mapear:

```text
worker boot
request start
request end
worker shutdown
```

al lifecycle Database.

---

# 191. RoadRunner preparation

Mapear:

```text
worker starts
job/request received
execution finishes
worker reset
worker stop
```

---

# 192. OpenSwoole preparation

Además deberá manejar:

```text
coroutine-local context
concurrent scopes
connection pool concurrency
```

---

# 193. Classic PHP

Podrá usar el mismo lifecycle:

```text
boot
execution
finalize
shutdown
```

aunque todo ocurra una sola vez.

---

# 194. Uniform semantics

Esto permite probar:

```text
persistent-runtime safety
```

incluso bajo classic PHP.

---

# 195. Worker registry

Podrá existir:

```text
DatabaseWorkerRegistry
```

pero normalmente contendrá sólo el worker actual.

---

# 196. Worker snapshot

Para diagnostics:

```php
final readonly class DatabaseWorkerSnapshot
{
    public function __construct(
        public WorkerId $id,
        public WorkerGeneration $generation,
        public WorkerState $state,
        public WorkerHealth $health,
        public int $activeExecutions,
        public int $completedExecutions,
        public int $openConnections,
        public int $activeTransactions,
        public int $openCursors,
        public int $memoryBytes,
    ) {}
}
```

---

# 197. Snapshot bounded

No incluir referencias a todos los EntityManagers históricos.

---

# 198. Worker diagnostics

Ejemplo:

```text
VoltStack Database Worker
─────────────────────────────────
Worker: db-worker-04
Generation: 12
Runtime: FrankenPHP

State: READY
Health: HEALTHY
Tainted: no

Executions
  Active:       0
  Completed:    82,441
  Failed:       173
  Cancelled:    28

Resources
  Connections:  8
  Idle:         8
  Leased:       0
  Transactions: 0
  Cursors:      0

Memory
  Current:      148 MB
  Baseline:     132 MB

Configuration generation: 31
Metadata generation:      18
Topology generation:      92
```

---

# 199. Debug isolation report

Después de request en development:

```text
Database Execution Finalization
────────────────────────────────
Execution: req-01842

Transactions:      CLEAN
Connection leases: CLEAN
Result cursors:    CLEAN
EntityManager:     CLOSED
IdentityMap:       EMPTY
UnitOfWork:        EMPTY
Tenant context:    CLEARED
Query context:     CLEARED
Lazy iterators:    CLOSED

Isolation: VERIFIED
```

---

# 200. Leak diagnostics

Ejemplo:

```text
Isolation verification failed.

Leaked resources:
- ConnectionLease conn-17
- ResultCursor cursor-83

Owning execution:
req-01842

Action:
connection discarded
worker remains healthy
```

---

# 201. Worker-level leak

Ejemplo:

```text
Shared listener retained execution-scoped EntityManager.

Severity:
WORKER_GLOBAL

Action:
worker marked TAINTED
new admissions stopped
worker recycle requested
```

---

# 202. Telemetry events

```text
database.worker.boot.started
database.worker.boot.completed
database.worker.boot.failed

database.worker.ready

database.execution.admitted
database.execution.started
database.execution.finalizing
database.execution.reset
database.execution.verified
database.execution.completed
database.execution.failed
database.execution.cancelled

database.worker.degraded
database.worker.tainted
database.worker.drain.started
database.worker.drain.completed
database.worker.recycle.requested
database.worker.shutdown.started
database.worker.shutdown.completed
```

---

# 203. Metrics

```text
database_worker_uptime_seconds
database_worker_active_executions
database_worker_completed_executions_total
database_worker_failed_executions_total
database_worker_tainted_total
database_worker_recycles_total
database_worker_memory_bytes

database_execution_duration_seconds
database_execution_finalization_duration_seconds
database_execution_reset_duration_seconds
database_execution_isolation_failure_total
database_execution_resource_leak_total
```

---

# 204. Metric cardinality

No usar:

```text
execution_id
tenant_id
connection_id
entity_id
```

como labels no acotados.

---

# 205. Tracing

Jerarquía posible:

```text
execution span
│
├── database query
├── database transaction
├── hydration
├── persistence
└── finalization
```

---

# 206. Finalization telemetry

Debe ser suficientemente ligera para ejecutarse en cada execution.

---

# 207. Sampling

Eventos de alto volumen podrán samplearse.

Errores de aislamiento:

```text
never silently sampled away
```

para diagnostics críticos.

---

# 208. Security

Un worker persistente aumenta el impacto de errores de aislamiento.

---

# 209. Security rule

Nunca deberán cruzar execution boundaries:

```text
tenant identity
authorization state
database role
sensitive query parameters
user identity
session-bound secrets
temporary privilege escalation
```

---

# 210. Privilege reset

Si una conexión ejecutó:

```text
SET ROLE elevated_role
```

deberá restaurarse antes del reuse.

---

# 211. Security context

`DatabaseSecurityContext` será execution-scoped.

---

# 212. Tenant security

Tenant context será execution-scoped incluso si physical connection es reutilizable.

---

# 213. Credential provider

Podrá ser worker-shared, pero secretos resueltos deberán manejarse bajo políticas específicas.

---

# 214. Diagnostics redaction

Worker dumps no expondrán:

```text
passwords
tokens
raw credentials
sensitive query parameters
PII
```

---

# 215. Worker health endpoint

La integración HTTP podrá exponer un health endpoint.

Pero el Database core sólo proporcionará:

```text
DatabaseWorkerHealthSnapshot
```

No dependerá de HTTP.

---

# 216. Readiness

Distinguir:

```text
liveness
readiness
health
```

---

# 217. Liveness

Pregunta:

> ¿El worker/proceso está funcionando?

---

# 218. Readiness

Pregunta:

> ¿Debe recibir nuevas ejecuciones?

---

# 219. Health

Pregunta:

> ¿Qué tan correctamente funciona la infraestructura Database?

---

# 220. Tainted worker

Puede estar:

```text
alive = yes
ready = no
health = unhealthy
```

---

# 221. Draining worker

Puede estar:

```text
alive = yes
ready = no
health = healthy
```

---

# 222. Readiness calculation

Conceptualmente:

```text
Ready(W)
=
StateAllowsAdmission(W)
∧ ¬Draining(W)
∧ ¬Tainted(W)
∧ ResourceBudgetAvailable(W)
∧ RequiredDependenciesAvailable(W)
```

---

# 223. Database outage

Policy configurable:

```text
worker remains alive
readiness false
```

o capacidades degradadas.

---

# 224. Circuit breaker interaction

Endpoint circuit breaker state podrá afectar readiness de operaciones específicas.

---

# 225. Retry interaction

Retries deberán permanecer dentro del execution scope actual.

---

# 226. Retry after scope close

Prohibido.

---

# 227. Scheduled retry

Debe convertirse en nueva ejecución/job.

---

# 228. Events after execution

Eventos síncronos deberán terminar antes de finalization.

---

# 229. Deferred events

Deberán materializar payload seguro y ejecutarse en nueva ejecución.

No conservar EntityManager del request anterior.

---

# 230. Outbox

Buen mecanismo para side effects post-commit.

---

# 231. ORM postCommit callbacks

No deberán escapar accidentalmente del scope.

---

# 232. Query cancellation

Al terminar request por timeout/client disconnect:

```text
CancellationToken
```

deberá propagarse cuando corresponda.

---

# 233. Client disconnect

No necesariamente implica cancelar DB automáticamente.

La policy decide según operación.

---

# 234. Write safety

Cancelar una write query no significa que no haya ocurrido.

Outcome puede ser UNKNOWN.

---

# 235. Worker taint and UNKNOWN query

Una UNKNOWN query outcome no necesariamente tainta worker.

Puede:

```text
discard connection
mark transaction unknown
fail execution
```

---

# 236. Scope-local containment

Principio:

> **Contener el fallo en el lifetime más pequeño posible.**

Orden:

```text
operation
↓
transaction
↓
connection
↓
execution
↓
worker
↓
process
```

---

# 237. Escalation

Sólo escalar cuando no pueda contenerse en el nivel actual.

---

# 238. Example

```text
bad query
→ operation failure
```

No:

```text
bad query
→ recycle worker
```

---

# 239. Example 2

```text
connection protocol corruption
→ discard connection
```

---

# 240. Example 3

```text
execution scope contamination
→ fail execution
→ verify worker
```

---

# 241. Example 4

```text
shared registry corruption
→ drain/recycle worker
```

---

# 242. Error hierarchy

```text
DatabaseWorkerLifecycleException
├── DatabaseWorkerBootException
├── DatabaseWorkerInitializationException
├── DatabaseWorkerAdmissionException
├── DatabaseWorkerNotReadyException
├── DatabaseWorkerDrainingException
├── DatabaseWorkerTaintedException
├── DatabaseWorkerHealthException
├── DatabaseExecutionLifecycleException
│   ├── DatabaseExecutionInitializationException
│   ├── DatabaseExecutionFinalizationException
│   ├── DatabaseExecutionResetException
│   └── DatabaseExecutionIsolationException
├── DatabaseWorkerDrainException
├── DatabaseWorkerRecycleException
└── DatabaseWorkerShutdownException
```

---

# 243. Directory structure

Propuesta:

```text
src/Quantum/Database/Runtime/Worker/
│
├── Contract/
│   ├── DatabaseWorker.php
│   ├── DatabaseWorkerLifecycleManager.php
│   ├── DatabaseWorkerHealthEvaluator.php
│   ├── DatabaseExecutionScopeResolver.php
│   ├── DatabaseExecutionResetParticipant.php
│   ├── DatabaseExecutionIsolationVerifier.php
│   └── DatabaseRuntimeAdapter.php
│
├── Model/
│   ├── WorkerId.php
│   ├── WorkerGeneration.php
│   ├── WorkerState.php
│   ├── WorkerHealth.php
│   ├── WorkerTaintSeverity.php
│   ├── DatabaseWorkerBudget.php
│   ├── DatabaseWorkerSnapshot.php
│   ├── DatabaseExecutionKind.php
│   ├── DatabaseExecutionState.php
│   ├── DatabaseExecutionBudget.php
│   ├── ExecutionId.php
│   ├── ExecutionGeneration.php
│   └── ExecutionScopeToken.php
│
├── Lifecycle/
│   ├── DefaultDatabaseWorkerLifecycleManager.php
│   ├── DatabaseWorkerBootstrap.php
│   ├── DatabaseWorkerAdmissionManager.php
│   ├── DatabaseWorkerDrainManager.php
│   ├── DatabaseWorkerRecycleManager.php
│   └── DatabaseWorkerShutdownManager.php
│
├── Execution/
│   ├── DatabaseExecutionScope.php
│   ├── DatabaseExecutionScopeFactory.php
│   ├── DatabaseExecutionResourceRegistry.php
│   ├── DatabaseExecutionFinalizer.php
│   └── DatabaseFinalizationReport.php
│
├── Reset/
│   ├── DatabaseResetCoordinator.php
│   ├── DatabaseResetPhase.php
│   ├── DatabaseResetContext.php
│   └── DatabaseResetReport.php
│
├── Isolation/
│   ├── DatabaseIsolationVerifier.php
│   ├── DatabaseIsolationVerificationResult.php
│   └── IsolationVerificationStatus.php
│
├── Health/
│   ├── DefaultDatabaseWorkerHealthEvaluator.php
│   ├── DatabaseWorkerHealthSnapshot.php
│   └── DatabaseWorkerReadinessEvaluator.php
│
├── Diagnostics/
│   ├── DatabaseWorkerDiagnostics.php
│   ├── DatabaseExecutionDiagnostics.php
│   └── DatabaseLeakDiagnostics.php
│
└── Exception/
    ├── DatabaseWorkerLifecycleException.php
    ├── DatabaseWorkerBootException.php
    ├── DatabaseWorkerAdmissionException.php
    ├── DatabaseWorkerNotReadyException.php
    ├── DatabaseWorkerDrainingException.php
    ├── DatabaseWorkerTaintedException.php
    ├── DatabaseExecutionLifecycleException.php
    ├── DatabaseExecutionFinalizationException.php
    ├── DatabaseExecutionResetException.php
    └── DatabaseExecutionIsolationException.php
```

---

# 244. Runtime adapters directory

```text
src/Quantum/Database/Runtime/
│
├── Worker/
│
├── Classic/
│   └── ClassicPhpDatabaseRuntimeAdapter.php
│
├── FrankenPHP/
│   └── FrankenPhpDatabaseRuntimeAdapter.php
│
├── RoadRunner/
│   └── RoadRunnerDatabaseRuntimeAdapter.php
│
└── OpenSwoole/
    └── OpenSwooleDatabaseRuntimeAdapter.php
```

Los siguientes documentos definirán cada integración.

---

# 245. Testing strategy

El sistema requerirá:

```text
unit tests
lifecycle integration tests
persistent worker tests
concurrency tests
leak tests
memory tests
fault injection
runtime adapter conformance
long-running soak tests
```

---

# 246. Lifecycle conformance test

Todo adapter deberá pasar:

```text
boot
→ execution
→ finalization
→ reset
→ second execution
→ shutdown
```

verificando aislamiento.

---

# 247. Tenant contamination test

```text
Execution A
tenant = ACME

Execution B
tenant = BETA
```

B nunca observará ACME.

---

# 248. IdentityMap contamination test

```text
Execution A
User#10 → Object A

Execution B
User#10 → Object B
```

No deberá reutilizarse Object A.

---

# 249. UnitOfWork contamination test

Cambios pendientes de A no aparecen en B.

---

# 250. Transaction contamination test

Transaction de A nunca cruza a B.

---

# 251. Connection reuse test

Physical connection sí podrá cruzar:

```text
A
↓
reset
↓
verify
↓
B
```

sin cruzar semantic state.

---

# 252. Query context test

Bindings de A no aparecen en B.

---

# 253. Tenant role test

DB role de A restaurado antes de B.

---

# 254. Cursor leak test

Cursor abandonado deberá:

```text
close
```

o provocar discard de conexión.

---

# 255. Lazy iterator leak test

Iterador activo al finalizar scope deberá cerrarse.

---

# 256. Event listener leak test

Listener scope-specific no deberá sobrevivir.

---

# 257. Telemetry context test

Request ID de A no aparece en spans de B.

---

# 258. Exception path test

Aunque aplicación lance excepción:

```text
finalization still executes
```

---

# 259. Cancellation path test

Aunque ejecución sea cancelada:

```text
cleanup still executes
```

---

# 260. Reset failure test

Debe determinar correctamente:

```text
resource-local discard
vs
worker taint
```

---

# 261. Unknown isolation test

Default:

```text
worker no longer ready
```

si el unknown puede afectar shared state.

---

# 262. Concurrent scope test

Ejecutar:

```text
A
B
C
```

simultáneamente.

Cada uno deberá mantener:

```text
DatabaseContext
EntityManager
IdentityMap
TransactionContext
tenant
leases
```

independientes.

---

# 263. Fiber switching test

Cambiar repetidamente entre Fibers y comprobar scope resolution.

---

# 264. Coroutine test

OpenSwoole deberá pasar equivalente.

---

# 265. Max concurrency test

Nunca superar:

```text
worker.maxConcurrentExecutions
```

---

# 266. Drain test

Durante drain:

```text
existing execution continues
new execution rejected
```

---

# 267. Shutdown test

Verificar:

```text
no idle connection survives
no active lease remains
telemetry bounded flush attempted
```

---

# 268. Forced shutdown test

Verificar que timeout de drain no cause deadlock.

---

# 269. Configuration reload test

A usa generation N.

B inicia después de reload y usa N+1.

---

# 270. Credential rotation test

Nuevas conexiones usan nueva generación.

---

# 271. Topology update test

Nuevas operaciones usan topology generation actual.

---

# 272. Failover test

Transaction existente mantiene semántica; nuevas operaciones no usan old writer como autoridad sin validación.

---

# 273. Worker recycle test

Después del threshold:

```text
worker drains
```

en lugar de aceptar trabajo ilimitadamente.

---

# 274. Memory leak test

Ejecutar decenas de miles de requests y observar post-reset memory.

---

# 275. Soak test

Ejemplo:

```text
100,000 executions
```

con mezcla de:

```text
reads
writes
transactions
exceptions
timeouts
cancellations
ORM hydration
lazy collections
chunk processing
```

---

# 276. Soak invariants

Al terminar:

```text
active executions = 0
active transactions = 0
leased connections = 0
open cursors = 0
IdentityMaps = 0 execution maps
UnitOfWorks = 0 execution UoWs
```

salvo infraestructura explícitamente compartida.

---

# 277. Worker invariant set

## DB-WORKER-001

Worker no equivaldrá a execution.

## DB-WORKER-002

Process no equivaldrá necesariamente a worker.

## DB-WORKER-003

Worker podrá atender múltiples ejecuciones.

## DB-WORKER-004

Cada ejecución tendrá scope independiente.

## DB-WORKER-005

DatabaseContext será execution-scoped.

## DB-WORKER-006

EntityManager será execution-scoped.

## DB-WORKER-007

IdentityMap será execution-scoped.

## DB-WORKER-008

UnitOfWork será execution-scoped.

## DB-WORKER-009

TransactionContext será execution-scoped.

## DB-WORKER-010

Tenant context será execution-scoped.

## DB-WORKER-011

Authorization DB context será execution-scoped.

## DB-WORKER-012

Query context será execution-scoped.

## DB-WORKER-013

Hydration session será operation/execution scoped.

## DB-WORKER-014

Result cursor no sobrevivirá arbitrariamente al execution boundary.

## DB-WORKER-015

Lazy iterator no sobrevivirá arbitrariamente al execution boundary.

## DB-WORKER-016

Connection pools podrán ser worker-scoped.

## DB-WORKER-017

Physical connections podrán ser worker-scoped.

## DB-WORKER-018

Connection leases no serán worker-globales.

## DB-WORKER-019

Physical connection reuse requerirá reset/verification.

## DB-WORKER-020

Worker boot deberá terminar antes de admission.

## DB-WORKER-021

Failed boot impedirá readiness.

## DB-WORKER-022

Lazy physical connection creation estará permitida.

## DB-WORKER-023

Boot complete no equivaldrá necesariamente a DB reachable.

## DB-WORKER-024

Connectivity boot policy será explícita.

## DB-WORKER-025

Reusable infrastructure no equivaldrá a reusable execution state.

## DB-WORKER-026

Component lifetimes serán explícitos.

## DB-WORKER-027

Mutable tenant/transaction state no será worker-scoped por default.

## DB-WORKER-028

Admission será explícita.

## DB-WORKER-029

Draining worker no aceptará nuevas ejecuciones.

## DB-WORKER-030

Execution generation protegerá contra stale callbacks.

## DB-WORKER-031

Scope token estará ligado a worker/execution generation.

## DB-WORKER-032

Stale scope usage deberá fallar.

## DB-WORKER-033

Cada scope tendrá resource ownership rastreable.

## DB-WORKER-034

GC no sustituirá resource lifecycle.

## DB-WORKER-035

Execution result no equivaldrá a cleanup result.

## DB-WORKER-036

Finalization siempre se intentará.

## DB-WORKER-037

Request end no provocará auto-flush ORM por default.

## DB-WORKER-038

Active transaction al final será rollback por default cuando sea posible.

## DB-WORKER-039

Transaction no cruzará execution boundary.

## DB-WORKER-040

UNKNOWN transaction state será preservado.

## DB-WORKER-041

IdentityMap se limpiará al finalizar.

## DB-WORKER-042

UnitOfWork se limpiará al finalizar.

## DB-WORKER-043

Query state se limpiará al finalizar.

## DB-WORKER-044

Tenant state se limpiará al finalizar.

## DB-WORKER-045

Authorization state se limpiará al finalizar.

## DB-WORKER-046

L0 cache se limpiará al finalizar.

## DB-WORKER-047

Shared cache no se borrará indiscriminadamente.

## DB-WORKER-048

Reset participants tendrán lifecycle explícito.

## DB-WORKER-049

Reset no equivaldrá a verification.

## DB-WORKER-050

UNKNOWN isolation no equivaldrá a VERIFIED.

## DB-WORKER-051

Taint será explícito.

## DB-WORKER-052

Execution taint no equivaldrá automáticamente a worker taint.

## DB-WORKER-053

Resource-local failure se contendrá localmente cuando sea posible.

## DB-WORKER-054

Shared-state corruption podrá requerir recycle.

## DB-WORKER-055

Unknown cross-request contamination requerirá respuesta conservadora.

## DB-WORKER-056

Worker health y state serán distintos.

## DB-WORKER-057

Degraded worker podrá continuar bajo policy.

## DB-WORKER-058

Worker recycling será explícito.

## DB-WORKER-059

Max execution count podrá disparar recycle.

## DB-WORKER-060

Max worker lifetime podrá disparar recycle.

## DB-WORKER-061

Memory threshold podrá disparar recycle.

## DB-WORKER-062

Recycle no sustituirá correctness.

## DB-WORKER-063

Post-reset memory será observable.

## DB-WORKER-064

Worker budgets serán bounded.

## DB-WORKER-065

Execution budgets serán bounded.

## DB-WORKER-066

Core deberá soportar múltiples scopes concurrentes.

## DB-WORKER-067

No existirá global mutable current DatabaseContext.

## DB-WORKER-068

Scope resolution será runtime-aware.

## DB-WORKER-069

Fiber context será aislado.

## DB-WORKER-070

Coroutine context será aislado.

## DB-WORKER-071

Detached async work tendrá nuevo execution scope.

## DB-WORKER-072

EntityManager no escapará del scope.

## DB-WORKER-073

ConnectionLease no escapará del scope sin contrato explícito.

## DB-WORKER-074

Pool normalmente tendrá worker lifetime.

## DB-WORKER-075

Physical connection podrá cruzar requests sólo limpia.

## DB-WORKER-076

Pool drain y worker drain serán conceptos distintos.

## DB-WORKER-077

Configuration será generation-aware.

## DB-WORKER-078

Config reload preferirá immutable snapshots.

## DB-WORKER-079

Existing executions no cambiarán config arbitrariamente.

## DB-WORKER-080

Incompatible config reload podrá requerir recycle.

## DB-WORKER-081

Metadata tendrá generation.

## DB-WORKER-082

Compiled query cache respetará generation compatibility.

## DB-WORKER-083

Credential rotation será generation-aware.

## DB-WORKER-084

Secrets no se copiarán innecesariamente al scope.

## DB-WORKER-085

Topology será generation-aware.

## DB-WORKER-086

Failover normal no requerirá necesariamente worker recycle.

## DB-WORKER-087

Unknown writer authority no se resolverá por suposición.

## DB-WORKER-088

Shared caches serán bounded.

## DB-WORKER-089

Entity cache no almacenará managed instances entre scopes.

## DB-WORKER-090

IdentityMap nunca será worker-global.

## DB-WORKER-091

Static mutable state será evitado.

## DB-WORKER-092

Worker-shared listeners no capturarán execution objects.

## DB-WORKER-093

Scope listeners serán removidos al finalizar.

## DB-WORKER-094

Telemetry context se limpiará.

## DB-WORKER-095

Logging context se limpiará.

## DB-WORKER-096

Finalization intentará limpiar recursos independientes aun ante errores.

## DB-WORKER-097

Finalization podrá producir reporte agregado.

## DB-WORKER-098

Finalization ordering respetará dependencias.

## DB-WORKER-099

Reset dependency cycles serán detectados.

## DB-WORKER-100

Finalization tendrá deadline.

## DB-WORKER-101

Cancellation no sustituirá cleanup.

## DB-WORKER-102

Fatal error handling no será única barrera de aislamiento.

## DB-WORKER-103

Worker shutdown detendrá admissions primero.

## DB-WORKER-104

Graceful shutdown drenará ejecuciones.

## DB-WORKER-105

Forced shutdown no prometerá completion.

## DB-WORKER-106

Abrupt shutdown podrá producir UNKNOWN outcomes.

## DB-WORKER-107

Database core será runtime-neutral.

## DB-WORKER-108

FrankenPHP estará detrás de adapter.

## DB-WORKER-109

RoadRunner estará detrás de adapter.

## DB-WORKER-110

OpenSwoole estará detrás de adapter.

## DB-WORKER-111

Classic PHP usará los mismos principios.

## DB-WORKER-112

Worker diagnostics serán bounded.

## DB-WORKER-113

Diagnostics no expondrán secretos.

## DB-WORKER-114

Telemetry labels evitarán cardinalidad no acotada.

## DB-WORKER-115

Tenant identity nunca cruzará scopes.

## DB-WORKER-116

Database role privilegiado nunca cruzará scopes.

## DB-WORKER-117

Sensitive query context nunca cruzará scopes.

## DB-WORKER-118

Liveness no equivaldrá a readiness.

## DB-WORKER-119

Readiness no equivaldrá a health.

## DB-WORKER-120

Tainted worker no estará ready.

## DB-WORKER-121

Draining worker no estará ready.

## DB-WORKER-122

Retry permanecerá dentro del scope activo.

## DB-WORKER-123

Scheduled retry será nueva ejecución.

## DB-WORKER-124

Deferred work no conservará EntityManager antiguo.

## DB-WORKER-125

Cancellation de write no implicará que write no ocurrió.

## DB-WORKER-126

UNKNOWN query outcome no implicará automáticamente worker taint.

## DB-WORKER-127

Failures se contendrán en el lifetime más pequeño posible.

## DB-WORKER-128

Operation failure no reciclará worker sin razón.

## DB-WORKER-129

Connection corruption podrá contenerse descartando conexión.

## DB-WORKER-130

Shared infrastructure corruption podrá escalar a worker.

## DB-WORKER-131

Worker lifecycle será testeable independientemente del runtime.

## DB-WORKER-132

Todo runtime adapter tendrá conformance tests.

## DB-WORKER-133

Persistent runtimes tendrán contamination tests.

## DB-WORKER-134

Persistent runtimes tendrán soak tests.

## DB-WORKER-135

Persistent runtimes tendrán memory tests.

## DB-WORKER-136

Concurrent runtimes tendrán context-isolation tests.

## DB-WORKER-137

Drain tendrá tests.

## DB-WORKER-138

Shutdown tendrá tests.

## DB-WORKER-139

Configuration reload tendrá tests.

## DB-WORKER-140

Credential rotation tendrá tests.

## DB-WORKER-141

Topology updates tendrán tests.

## DB-WORKER-142

Failover tendrá lifecycle tests.

## DB-WORKER-143

No active transaction deberá quedar después de verified finalization.

## DB-WORKER-144

No active connection lease deberá quedar después de verified finalization.

## DB-WORKER-145

No active cursor deberá quedar después de verified finalization.

## DB-WORKER-146

No execution IdentityMap deberá quedar después de scope close.

## DB-WORKER-147

No execution UnitOfWork deberá quedar después de scope close.

## DB-WORKER-148

No execution tenant context deberá quedar después de scope close.

## DB-WORKER-149

No execution security context deberá quedar después de scope close.

## DB-WORKER-150

No execution query state deberá quedar después de scope close.

## DB-WORKER-151

Worker-shared infrastructure deberá ser explícitamente segura para reuse.

## DB-WORKER-152

Shared mutable infrastructure requerirá synchronization/concurrency contract.

## DB-WORKER-153

Aislamiento tendrá prioridad sobre throughput.

## DB-WORKER-154

Correctness tendrá prioridad sobre worker reuse.

## DB-WORKER-155

Un worker incierto podrá reciclarse.

## DB-WORKER-156

Crear nuevo worker será preferible a reutilizar uno con contaminación desconocida.

## DB-WORKER-157

Worker reuse será una optimización operacional.

## DB-WORKER-158

La semántica Database no dependerá de que el worker sea reutilizado.

## DB-WORKER-159

Classic y persistent runtimes deberán observar las mismas invariantes Database.

## DB-WORKER-160

Ningún runtime adapter podrá debilitar las garantías fundamentales del Database core.

---

# 278. Modelo formal del lifecycle

Sea:

```text
W
```

un worker.

Su ciclo general será:

```text
Boot(W)
→ Initialize(W)
→ Ready(W)
→ { Execute(Eᵢ) → Finalize(Eᵢ) → Verify(Eᵢ) }*
→ Drain(W)
→ Shutdown(W)
```

---

# 279. Condición de siguiente ejecución

Una nueva ejecución `Eₙ₊₁` sólo deberá aceptarse si:

```text
ReadyForNext(W)
=
AdmissionAllowed(W)
∧ PreviousRequiredCleanupComplete(W)
∧ SharedInfrastructureValid(W)
∧ ResourceBudgetsAvailable(W)
```

---

# 280. Isolation condition

Para dos ejecuciones distintas:

```text
Eᵢ ≠ Eⱼ
```

deberá cumplirse:

```text
MutableExecutionState(Eᵢ)
∩
MutableExecutionState(Eⱼ)
=
∅
```

salvo recursos explícitamente compartidos mediante contratos concurrency-safe.

---

# 281. Physical connection exception

Una physical connection puede ser reutilizada temporalmente:

```text
C ∈ Resources(Eᵢ)
```

y posteriormente:

```text
C ∈ Resources(Eⱼ)
```

sólo si:

```text
Release(Eᵢ,C)
∧ Reset(C)
∧ Verify(C)
∧ NewLease(Eⱼ,C)
```

---

# 282. Scope validity

```text
ValidScope(S)
=
WorkerGeneration(S) = CurrentWorkerGeneration
∧ ExecutionGeneration(S) = ActiveExecutionGeneration(S)
∧ State(S) = ACTIVE
```

---

# 283. Worker readiness

```text
Ready(W)
=
StateAllowsAdmission(W)
∧ HealthAllowsAdmission(W)
∧ ¬Tainted(W)
∧ ¬Draining(W)
∧ BudgetAvailable(W)
```

---

# 284. Worker taint rule

```text
UnknownSharedIsolationState(W)
⇒
Tainted(W)
```

---

# 285. Escalation model

```text
Operation
   ↓
Transaction
   ↓
Connection
   ↓
Execution
   ↓
Worker
   ↓
Process
```

El error deberá escalar sólo hasta el nivel necesario para recuperar garantías.

---

# 286. Lifecycle architecture

```text
                 RUNTIME
                    │
                    ▼
             Worker Bootstrap
                    │
                    ▼
              ┌──────────┐
              │  READY   │◄──────────────────────────┐
              └────┬─────┘                           │
                   │                                 │
            execution arrives                       │
                   │                                 │
                   ▼                                 │
          Admission Controller                      │
                   │                                 │
          ┌────────┴────────┐                        │
          │                 │                        │
        ACCEPT            REJECT                     │
          │                                          │
          ▼                                          │
      Create Scope                                   │
          │                                          │
          ├── DatabaseContext                        │
          ├── EntityManager                          │
          ├── IdentityMap                            │
          ├── UnitOfWork                             │
          ├── TransactionContext                     │
          └── ResourceRegistry                       │
          │                                          │
          ▼                                          │
        ACTIVE                                       │
          │                                          │
          ▼                                          │
      Application                                    │
          │                                          │
          ▼                                          │
      FINALIZING                                     │
          │                                          │
          ├── stop operations                        │
          ├── resolve results                        │
          ├── resolve transactions                   │
          ├── release leases                         │
          ├── clear ORM                              │
          └── clear contexts                         │
          │                                          │
          ▼                                          │
       RESETTING                                     │
          │                                          │
          ▼                                          │
       VERIFYING                                     │
          │                                          │
      ┌───┴─────────────┐                            │
      │                 │                            │
   VERIFIED           FAILED                         │
      │                 │                            │
      │          containment possible?               │
      │             │         │                      │
      │            YES        NO                     │
      │             │         │                      │
      │             ▼         ▼                      │
      │          contain    TAINT                    │
      │             │         │                      │
      └─────────────┘         ▼                      │
          │               DRAINING                   │
          │                   │                      │
          └───────────────────┼──────────────────────┘
                              │
                              ▼
                           STOPPING
                              │
                              ▼
                            STOPPED
```

---

# 287. Recommended VoltStack model

La arquitectura recomendada será:

```text
PROCESS
└── WORKER
    │
    ├── Shared Database Infrastructure
    │   ├── Configuration snapshots
    │   ├── Driver registry
    │   ├── Dialect registry
    │   ├── Platform capabilities
    │   ├── Metadata cache
    │   ├── Compiled query cache
    │   ├── Connection pools
    │   └── Telemetry instruments
    │
    ├── EXECUTION A
    │   ├── DatabaseContext
    │   ├── EntityManager
    │   ├── IdentityMap
    │   ├── UnitOfWork
    │   ├── TransactionContext
    │   └── ResourceRegistry
    │
    └── EXECUTION B
        ├── DatabaseContext
        ├── EntityManager
        ├── IdentityMap
        ├── UnitOfWork
        ├── TransactionContext
        └── ResourceRegistry
```

A y B jamás compartirán directamente sus estados mutables.

---

# 288. Principio final

La arquitectura completa puede resumirse mediante:

```text
BOOT ONCE
```

pero:

```text
SCOPE EVERY EXECUTION
```

y:

```text
RESET EVERY EXECUTION
```

seguido por:

```text
VERIFY BEFORE REUSE
```

Por tanto:

```text
Persistent Worker
≠
Persistent Request State
```

```text
Persistent Worker
≠
Persistent EntityManager
```

```text
Persistent Worker
≠
Persistent IdentityMap
```

```text
Persistent Worker
≠
Persistent UnitOfWork
```

```text
Persistent Worker
≠
Persistent Transaction
```

mientras que:

```text
Persistent Worker
→ may retain immutable/bounded infrastructure
```

y:

```text
Persistent Worker
→ may retain clean reusable physical connections
```

La regla final de VoltStack será:

> **Persistir infraestructura; aislar ejecuciones; limpiar estado; verificar fronteras; contener fallos; reciclar el worker cuando ya no pueda demostrarse que la siguiente ejecución comenzará desde un estado seguro.**

---

# 289. Estado del Bloque 25

```text
BLOCK 25 — PERSISTENT RUNTIME

✓ 251_DATABASE_PERSISTENT_RUNTIME_ARCHITECTURE.md
✓ 252_DATABASE_REQUEST_SCOPE_SYSTEM.md
✓ 253_DATABASE_DATABASE_CONTEXT_SYSTEM.md
✓ 254_DATABASE_STATE_ISOLATION_SYSTEM.md
✓ 255_DATABASE_STATE_RESET_SYSTEM.md
✓ 256_DATABASE_CONNECTION_REUSE_SYSTEM.md
✓ 257_DATABASE_WORKER_LIFECYCLE_SYSTEM.md
│
├── 258_DATABASE_FRANKENPHP_INTEGRATION_SYSTEM.md
├── 259_DATABASE_ROADRUNNER_INTEGRATION_SYSTEM.md
└── 260_DATABASE_OPENSWOOLE_INTEGRATION_SYSTEM.md
```

---

# 290. Siguiente documento

```text
258_DATABASE_FRANKENPHP_INTEGRATION_SYSTEM.md
```

El siguiente documento aterrizará esta arquitectura sobre el runtime predeterminado de VoltStack:

```text
VoltStack
   │
   ▼
FrankenPHP
   │
   ▼
Persistent Worker
   │
   ├── Request A
   │     ↓
   │   Database Scope A
   │     ↓
   │   Finalize
   │     ↓
   │   Reset
   │     ↓
   │   Verify
   │
   ├── Request B
   │     ↓
   │   Database Scope B
   │
   └── ...
```

y deberá definir específicamente:

```text
FrankenPHP Runtime Adapter
Worker Boot
Request Hooks
Worker Mode
Classic Mode
Execution Scope Binding
Connection Pool Lifetime
Physical Connection Reuse
Request-local DatabaseContext
EntityManager Lifecycle
IdentityMap Lifecycle
UnitOfWork Lifecycle
Transaction Cleanup
Cursor Cleanup
Fiber Safety
Concurrent Requests
State Reset
Isolation Verification
Worker Tainting
Worker Recycling
Graceful Shutdown
Configuration Reload
Credential Rotation
Topology Changes
Telemetry
Diagnostics
Development Mode
Production Defaults
Testing
```

manteniendo como regla:

> **FrankenPHP será el runtime predeterminado de VoltStack, pero nunca se permitirá que la persistencia de sus workers convierta estado perteneciente a un request en estado perteneciente al siguiente.**