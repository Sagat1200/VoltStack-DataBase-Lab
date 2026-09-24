# 251_DATABASE_PERSISTENT_RUNTIME_ARCHITECTURE.md

# VoltStack Quantum Database
## Persistent Runtime Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 251 — Database Persistent Runtime Architecture  
**Bloque:** 25 — Persistent Runtime  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `250_DATABASE_PERFORMANCE_BENCHMARK_SYSTEM.md`  
**Siguiente documento:** `252_DATABASE_REQUEST_SCOPE_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura general mediante la cual `VoltStack/Quantum/Database` operará de forma segura, eficiente y determinista dentro de **runtimes PHP persistentes**.

VoltStack tendrá como runtime principal:

```text
FrankenPHP
```

y podrá integrar mediante adaptadores oficiales:

```text
RoadRunner
OpenSwoole
```

En estos entornos, el proceso PHP puede permanecer vivo después de terminar una petición.

Por tanto:

```text
Process Lifetime
≠
Request Lifetime
```

Esta diferencia cambia profundamente las reglas tradicionales de administración de estado de una aplicación PHP.

Regla central:

> **En VoltStack, reutilizar un worker nunca significará reutilizar accidentalmente el estado mutable de una operación anterior. Todo estado de Database deberá tener un propietario y un scope explícitos, y todo estado request-scoped deberá quedar finalizado, liberado, reseteado o invalidado antes de reutilizar el worker.**

---

# 2. El problema fundamental

En el modelo PHP clásico:

```text
HTTP Request
     │
     ▼
PHP Process
     │
     ├── Application
     ├── Container
     ├── ORM
     ├── Connections
     └── State
     │
     ▼
Response
     │
     ▼
Process Ends
```

Gran parte del estado desaparece automáticamente.

Conceptualmente:

```text
Request Ends
    ↓
Process Ends
    ↓
Memory Released
    ↓
State Destroyed
```

Esto crea una forma implícita de aislamiento.

---

# 3. Runtime persistente

En un runtime persistente:

```text
Worker Process
│
├── Boot Application
│
├── Request A
│
├── Request B
│
├── Request C
│
├── Request D
│
└── ...
```

El proceso continúa vivo.

Por tanto:

```text
Request Ends
    ↓
Process Continues
```

y:

```text
Memory
Objects
Connections
Caches
Services
Statics
References
```

pueden continuar existiendo.

---

# 4. El riesgo

Supongamos:

```text
Request A
Tenant = ACME
```

y posteriormente:

```text
Request B
Tenant = CONTOSO
```

Si Database conserva accidentalmente:

```text
currentTenant = ACME
```

el resultado puede convertirse en:

```text
Request B
    ↓
queries executed
    ↓
ACME database
```

Esto constituye una violación crítica de aislamiento.

---

# 5. Persistent Runtime ≠ Long Transaction

Un worker persistente puede vivir durante horas.

Esto no significa:

```text
Worker Lifetime
=
Transaction Lifetime
```

Por tanto:

```text
Persistent Runtime
≠
Persistent Transaction
```

---

# 6. Persistent Runtime ≠ Persistent ORM State

Tampoco significa:

```text
Worker
    ↓
same IdentityMap forever
```

o:

```text
same UnitOfWork forever
```

Por defecto:

```text
IdentityMap
UnitOfWork
EntityManager mutable state
```

pertenecerán a scopes mucho más pequeños.

---

# 7. Persistent Runtime ≠ Global State

VoltStack no solucionará workers persistentes mediante:

```php
Database::$currentConnection
Database::$currentTenant
Database::$entityManager
Database::$transaction
```

Este patrón queda arquitectónicamente prohibido para estado contextual mutable.

---

# 8. Objetivos

La arquitectura deberá garantizar:

1. aislamiento entre requests;
2. ownership explícito del estado;
3. lifecycle explícito;
4. cleanup determinista;
5. reutilización segura de conexiones;
6. reutilización segura de servicios inmutables;
7. eliminación del estado ORM anterior;
8. eliminación de transacciones incompletas;
9. aislamiento de tenant;
10. aislamiento de shard;
11. aislamiento de seguridad;
12. aislamiento de recursos;
13. prevención de memory leaks;
14. prevención de connection leaks;
15. soporte de FrankenPHP;
16. soporte futuro de RoadRunner;
17. soporte futuro de OpenSwoole;
18. soporte de CLI y Jobs;
19. soporte de concurrencia;
20. observabilidad del lifecycle.

---

# 9. Principio de Scope

Todo estado deberá pertenecer a un scope.

Modelo general:

```text
Application Scope
       │
       ▼
Worker Scope
       │
       ▼
Request Scope
       │
       ▼
Operation Scope
       │
       ▼
Transaction Scope
```

No todos los runtimes necesitarán materializar cada nivel como un container separado, pero la semántica deberá existir.

---

# 10. Application Scope

Representa estado válido durante toda la aplicación.

Ejemplos:

```text
immutable configuration
compiled metadata
driver registry
dialect registry
platform definitions
type registry
compiler registry
immutable mapping metadata
```

---

# 11. Worker Scope

Representa estado asociado al proceso persistente.

Ejemplos potenciales:

```text
connection pools
local metadata caches
compiled query caches
runtime adapter
worker telemetry
resource pools
```

Siempre que puedan reutilizarse de forma segura.

---

# 12. Request Scope

Representa estado exclusivo de una petición.

Ejemplos:

```text
DatabaseContext
EntityManager
IdentityMap
UnitOfWork
TransactionContext
TenantContext
ShardContext
ReadWriteRoutingContext
StickyConnectionContext
QueryContext
ResourceBudgetContext
```

---

# 13. Operation Scope

Algunas operaciones pueden requerir un scope aún menor.

Ejemplo:

```text
HTTP Request
    │
    ├── operation A
    │
    ├── operation B
    │
    └── operation C
```

Cada una podría tener:

```text
query deadline
retry context
cancellation token
temporary resource leases
```

---

# 14. Transaction Scope

Estado como:

```text
transaction depth
savepoints
transaction outcome
transaction retries
transaction events
connection affinity
```

pertenece al contexto transaccional.

---

# 15. Scope hierarchy

Conceptualmente:

```text
Application
└── Worker
    └── Request
        └── Operation
            └── Transaction
```

Pero:

```text
Scope Hierarchy
≠
Object Ownership Hierarchy
```

necesariamente.

---

# 16. Scope lifetime

Formalmente:

```text
L(transaction)
≤
L(operation)
≤
L(request)
≤
L(worker)
≤
L(application)
```

en el caso HTTP habitual.

---

# 17. Jobs

Para Jobs:

```text
Worker
└── Job Scope
    └── Operation
        └── Transaction
```

Por tanto Request Scope deberá entenderse conceptualmente como:

```text
Execution Scope
```

aunque la API HTTP pueda denominarlo RequestScope.

---

# 18. CLI

CLI puede usar:

```text
Command Scope
```

---

# 19. Unified execution concept

Internamente podrá existir:

```text
DatabaseExecutionScope
```

con especializaciones:

```text
HttpRequestScope
JobScope
CommandScope
TestScope
```

---

# 20. Shared ≠ Mutable

Regla:

> Cuanto mayor sea el lifetime de un servicio, menor deberá ser la cantidad de estado contextual mutable que contenga.

Ideal:

```text
Application/Worker service
    ↓
immutable/stateless
```

---

# 21. Safe reusable components

Candidatos naturales:

```text
TypeRegistry
DialectRegistry
DriverRegistry
CompilerRegistry
compiled Metadata
Query optimization rules
Platform capabilities
immutable configuration
```

---

# 22. Unsafe reusable state

Por defecto no deberá sobrevivir entre requests:

```text
IdentityMap
UnitOfWork
EntityManager state
TransactionContext
TenantContext
ShardContext
current connection lease
current transaction
query parameters
query execution state
lazy iterators
active cursors
hydration session
temporary relationship loaders
```

---

# 23. Persistent safe object

Un objeto podrá sobrevivir requests si cumple propiedades como:

```text
immutable
stateless
thread/coroutine safe where applicable
no request references
no active DB resources
```

---

# 24. Scope Safety Classification

Se propone:

```php
enum RuntimeScopeSafety
{
    case APPLICATION_SAFE;
    case WORKER_SAFE;
    case EXECUTION_SCOPED;
    case OPERATION_SCOPED;
    case TRANSACTION_SCOPED;
    case NOT_SHAREABLE;
}
```

---

# 25. DatabaseContext

El núcleo contextual será:

```text
DatabaseContext
```

que representa el contexto efectivo de Database para una ejecución.

---

# 26. DatabaseContext conceptual

```php
final class DatabaseContext
{
    public function executionId(): ExecutionId;

    public function tenant(): ?TenantContext;

    public function shard(): ?ShardContext;

    public function transaction(): TransactionContext;

    public function routing(): RoutingContext;

    public function resources(): ResourceContext;

    public function cancellation(): CancellationToken;
}
```

---

# 27. DatabaseContext ≠ Global singleton

Nunca:

```php
DatabaseContext::current();
```

si ello implica estado global mutable.

---

# 28. Context resolution

El contexto deberá resolverse mediante infraestructura scope-aware.

Conceptualmente:

```text
Execution
   ↓
Scoped Container
   ↓
DatabaseContext
```

---

# 29. Context propagation

Las operaciones internas deberán recibir o resolver el contexto apropiado sin depender de global mutable state.

---

# 30. Context capture

Una operación asíncrona no deberá capturar accidentalmente referencias a un contexto que ya terminó.

---

# 31. Execution ID

Cada ejecución tendrá:

```text
ExecutionId
```

Ejemplo:

```text
req_01J...
job_01J...
cmd_01J...
```

---

# 32. Execution generation

Adicionalmente puede existir:

```text
ExecutionGeneration
```

para detectar objetos pertenecientes a scopes anteriores.

---

# 33. Stale Context Detection

Ejemplo:

```text
Object created in Request A
        ↓
used during Request B
        ↓
generation mismatch
        ↓
StaleDatabaseContextException
```

cuando sea técnicamente viable.

---

# 34. Runtime Lifecycle

Modelo general:

```text
PROCESS_START
      ↓
APPLICATION_BOOT
      ↓
WORKER_START
      ↓
┌───────────────────────────────┐
│ EXECUTION_START               │
│      ↓                        │
│ DATABASE_SCOPE_CREATE         │
│      ↓                        │
│ APPLICATION WORK             │
│      ↓                        │
│ DATABASE_SCOPE_FINALIZE       │
│      ↓                        │
│ DATABASE_SCOPE_RESET          │
│      ↓                        │
│ EXECUTION_END                 │
└───────────────────────────────┘
      ↓
NEXT EXECUTION
      ↓
...
      ↓
WORKER_STOP
      ↓
PROCESS_END
```

---

# 35. Boot ≠ Request Initialization

Application boot ocurre:

```text
once per worker/process
```

en muchos runtimes.

Request initialization ocurre:

```text
once per request
```

No deberán mezclarse.

---

# 36. Database Bootstrap

Bootstrap podrá registrar:

```text
drivers
dialects
types
metadata compilers
connection factories
services
```

pero no crear estado request-specific.

---

# 37. Lazy acquisition

Recursos caros deberán adquirirse cuando se necesiten.

Ejemplo:

```text
Request
   ↓
no database query
   ↓
no DB connection required
```

---

# 38. Connection reuse

Workers persistentes hacen atractiva la reutilización de conexiones.

Pero:

```text
Connection Reuse
≠
Connection State Reuse
```

---

# 39. Connection state contamination

Una conexión puede conservar:

```text
open transaction
session variables
temporary tables
isolation changes
locks
database selection
session timezone
role
search_path
```

según plataforma.

---

# 40. Connection reset

Antes de reutilizar una conexión deberá comprobarse/restaurarse un estado compatible.

Esto será desarrollado en:

```text
255_DATABASE_STATE_RESET_SYSTEM.md
256_DATABASE_CONNECTION_REUSE_SYSTEM.md
```

---

# 41. Transaction leak

Caso crítico:

```text
Request A
    ↓
BEGIN
    ↓
exception
    ↓
transaction not closed
    ↓
connection reused
    ↓
Request B
```

Debe impedirse.

---

# 42. Transaction finalization

Al finalizar scope:

```text
if transaction ACTIVE
    ↓
ROLLBACK
```

salvo reglas explícitas más estrictas.

Nunca:

```text
auto COMMIT unknown transaction
```

---

# 43. UNKNOWN transaction outcome

Si:

```text
COMMIT sent
connection lost
```

el outcome puede ser:

```text
UNKNOWN
```

La conexión/contexto no deberá simplemente resetearse como si nada hubiera ocurrido.

---

# 44. Tainted resources

Se introducirá el concepto:

```text
TAINTED
```

Un recurso tainted:

```text
must not return to normal reusable pool
```

hasta que exista recuperación demostrable.

---

# 45. Connection disposition

Al terminar una ejecución:

```text
SAFE_REUSE
RESET_REQUIRED
DISCARD
TAINTED
UNKNOWN
```

---

# 46. IdentityMap

Por defecto:

```text
IdentityMap Scope
=
Execution Scope
```

---

# 47. IdentityMap leakage

Incorrecto:

```text
Request A
User#10 → Alice
```

seguido de:

```text
Request B
find(User#10)
    ↓
IdentityMap
    ↓
Alice from Request A
```

---

# 48. UnitOfWork

También será execution-scoped.

---

# 49. UnitOfWork leakage

Un ChangeSet pendiente de Request A nunca deberá llegar a:

```text
Request B → flush()
```

---

# 50. EntityManager

EntityManager tendrá lifecycle explícito.

Conceptualmente:

```text
OPEN
TAINTED
CLOSED
```

---

# 51. EntityManager end-of-scope

Al finalizar execution:

```text
EntityManager
    ↓
finalize
    ↓
clear IdentityMap
    ↓
clear UnitOfWork
    ↓
release references
    ↓
CLOSED
```

---

# 52. Entity references outside scope

Una entidad puede escapar hacia código exterior.

Eso no significa que siga:

```text
MANAGED
```

después de terminar su EntityManager.

Conceptualmente pasa a ser:

```text
DETACHED
```

---

# 53. Detached entity

Una entidad detached no deberá encontrar mágicamente el nuevo EntityManager global.

---

# 54. Lazy loading after scope

Si una entidad intenta lazy load después de cerrar su contexto:

```text
LazyLoadingOutsideScopeException
```

o comportamiento equivalente explícito.

Nunca:

```text
resolve current global manager
```

porque podría pertenecer a otro request/tenant.

---

# 55. Lazy iterators

Un:

```text
LazyCollection
ResultCursor
StreamingResult
```

activo al finalizar request deberá cerrarse.

---

# 56. Resource ownership

Todo recurso activo deberá tener owner.

Ejemplo:

```text
ResultCursor
owner = QueryExecution
```

---

# 57. Ownership graph

```text
ExecutionScope
│
├── EntityManager
│   ├── IdentityMap
│   └── UnitOfWork
│
├── TransactionContext
│
├── ConnectionLeases
│
├── ActiveCursors
│
├── LazySources
│
└── ResourceLeases
```

---

# 58. Scope Resource Registry

Se propone:

```text
DatabaseScopeResourceRegistry
```

para registrar recursos que requieren finalización.

---

# 59. Resource contract

```php
interface ScopedDatabaseResource
{
    public function finalizeScope(
        ScopeFinalizationContext $context
    ): void;
}
```

---

# 60. Finalization ≠ Destructor

No se dependerá de:

```php
__destruct()
```

como mecanismo primario de correctness.

---

# 61. Destructor

Podrá existir únicamente como:

```text
last-resort safety mechanism
```

---

# 62. Deterministic cleanup

El runtime adapter deberá ejecutar cleanup explícito.

---

# 63. Cleanup order

El orden importa.

Propuesta conceptual:

```text
Stop new operations
       ↓
Cancel pending operations
       ↓
Close active result streams/cursors
       ↓
Finalize ORM operations
       ↓
Resolve/rollback active transactions
       ↓
Release connection leases
       ↓
Clear ORM state
       ↓
Release resource permits
       ↓
Clear contextual references
       ↓
Verify scope cleanliness
```

---

# 64. Cleanup DAG

En implementación real se deberá modelar dependencia entre recursos, no confiar únicamente en una lista arbitraria.

---

# 65. Cleanup idempotency

Cuando sea posible:

```text
cleanup(cleanup(x))
=
cleanup(x)
```

---

# 66. Cleanup failure

Un fallo de cleanup no deberá ocultarse.

---

# 67. Cleanup status

```php
enum ScopeCleanupStatus
{
    case CLEAN;
    case CLEAN_WITH_WARNINGS;
    case FAILED;
    case TAINTED;
    case UNKNOWN;
}
```

---

# 68. Worker disposition

Si el cleanup no puede demostrar seguridad:

```text
Worker
    ↓
should be recycled
```

cuando el runtime permita dicha operación.

---

# 69. Fail closed

Para aislamiento:

```text
Cannot prove clean
    ↓
do not blindly reuse
```

---

# 70. Worker Recycling

El sistema podrá solicitar:

```text
RECYCLE_WORKER
```

por razones como:

```text
state corruption
unknown transaction
connection contamination
memory threshold
resource leak
unrecoverable runtime state
```

---

# 71. Worker recycle ≠ normal reset

Reset es operación habitual.

Recycle es mecanismo de contención.

---

# 72. Runtime Adapter

La arquitectura central no dependerá directamente de FrankenPHP.

Se utilizará:

```text
PersistentRuntimeAdapter
```

---

# 73. Contract

```php
interface PersistentRuntimeAdapter
{
    public function runtime(): RuntimeId;

    public function capabilities(): RuntimeCapabilities;

    public function registerLifecycle(
        DatabaseRuntimeLifecycle $lifecycle
    ): void;
}
```

---

# 74. Runtime lifecycle interface

```php
interface DatabaseRuntimeLifecycle
{
    public function workerStarted(
        WorkerContext $context
    ): void;

    public function executionStarted(
        ExecutionContext $context
    ): void;

    public function executionFinished(
        ExecutionContext $context
    ): void;

    public function workerStopping(
        WorkerContext $context
    ): void;
}
```

---

# 75. Core independence

```text
Database Core
    ↓
Runtime Contracts
    ↓
Runtime Adapter
    ↓
FrankenPHP
```

Nunca:

```text
ORM
    ↓
FrankenPHP-specific API
```

---

# 76. FrankenPHP

FrankenPHP será la implementación de referencia.

Arquitectura:

```text
FrankenPHP Worker
      │
      ▼
FrankenPhpRuntimeAdapter
      │
      ▼
DatabaseRuntimeLifecycle
      │
      ▼
DatabaseExecutionScope
```

---

# 77. RoadRunner

```text
RoadRunner Worker
      │
      ▼
RoadRunnerRuntimeAdapter
      │
      ▼
same Database lifecycle
```

---

# 78. OpenSwoole

```text
OpenSwoole Worker
      │
      ▼
OpenSwooleRuntimeAdapter
      │
      ▼
Coroutine-aware scope resolution
```

---

# 79. OpenSwoole challenge

En concurrencia:

```text
Worker
├── Coroutine A
├── Coroutine B
└── Coroutine C
```

Por tanto:

```text
Worker-local state
≠
Request-local state
```

es todavía más importante.

---

# 80. Coroutine isolation

Nunca:

```php
static $currentDatabaseContext;
```

para contextos concurrentes.

---

# 81. Context Local Storage

Runtime adapters podrán proporcionar abstracciones equivalentes a:

```text
Execution Local Storage
Coroutine Local Storage
Fiber Local Storage
```

pero mediante contratos de VoltStack.

---

# 82. RuntimeLocalStorage

Conceptualmente:

```php
interface RuntimeLocalStorage
{
    public function get(ContextKey $key): mixed;

    public function set(ContextKey $key, mixed $value): void;

    public function remove(ContextKey $key): void;
}
```

---

# 83. Local Storage ≠ Global State

Aunque la API parezca contextual:

```text
current()
```

la implementación deberá resolver por execution/coroutine/fiber.

---

# 84. Concurrency model

Database deberá asumir que dos operaciones pueden ejecutarse simultáneamente dentro del mismo proceso.

---

# 85. Mutable worker singleton

Por tanto, un singleton mutable deberá considerarse sospechoso por default.

---

# 86. Immutable shared services

Ejemplo seguro:

```php
final readonly class PostgreSqlDialect
{
}
```

si no contiene estado contextual.

---

# 87. Mutable cache

Caches worker-local requerirán concurrencia segura cuando el runtime soporte ejecución concurrente real/intercalada.

---

# 88. Metadata cache

Puede ser worker-shared porque almacena:

```text
structural immutable knowledge
```

no entidades.

---

# 89. Entity Cache

Aunque sea compartido:

```text
Entity Cache
≠
IdentityMap
```

y nunca almacena instancias managed entre requests.

---

# 90. Query Cache

Puede ser compartido si:

```text
cache key
```

incluye correctamente los contextos relevantes.

---

# 91. Tenant leakage through cache

Una key incompleta:

```text
user:10
```

puede ser insegura en multitenancy.

Podría requerirse:

```text
tenant:ACME:user:10
```

según arquitectura.

---

# 92. Persistent Runtime amplifies cache bugs

Un cache incorrecto puede sobrevivir muchos requests.

Por ello scope y key identity son fundamentales.

---

# 93. Connection Manager

El ConnectionManager podrá ser worker-safe.

Pero:

```text
ConnectionManager
≠
CurrentConnection
```

---

# 94. Connection Lease

Cada uso contextual deberá modelarse como:

```text
ConnectionLease
```

---

# 95. Lease ownership

```text
Execution A
    ↓
Lease A
```

No deberá utilizarse desde:

```text
Execution B
```

salvo recurso explícitamente compartible.

---

# 96. Connection affinity

Durante una transacción:

```text
Transaction
    ↓
Connection affinity
```

deberá mantenerse.

---

# 97. Sticky connections

Sticky read/write state será:

```text
execution-scoped
```

salvo política explícita diferente.

---

# 98. Replica routing

Una decisión tomada en Request A no deberá contaminar Request B.

---

# 99. Shard routing

Igual:

```text
ShardContext
=
execution/operation scoped
```

---

# 100. Tenant Context

TenantContext deberá estar ligado al execution scope.

---

# 101. Tenant switching

Cambiar tenant dentro del mismo scope será restringido.

Preferencia:

```text
one execution
    ↓
one stable tenant context
```

para operaciones normales.

---

# 102. Cross-tenant operation

Deberá ser explícita:

```text
CrossTenantOperation
```

y no una mutación casual del `currentTenant`.

---

# 103. Security Context

Autorización y Database Access Security deberán mantenerse scope-aware.

---

# 104. Credentials

Credenciales resueltas para una conexión no deberán filtrarse hacia otra configuración/tenant.

---

# 105. Query Context

Información como:

```text
timeout
deadline
trace
audit context
security scope
tenant
shard
```

deberá derivarse del execution/operation actual.

---

# 106. Query Builder

Query Builder seguirá siendo una definición.

No deberá capturar:

```text
live connection
active transaction
request container
```

innecesariamente.

---

# 107. Query object lifetime

Un Query Model inmutable puede sobrevivir más tiempo que un request si no contiene bindings/contexto sensible.

---

# 108. Query Definition ≠ Query Execution

Esto facilita persistent runtimes:

```text
immutable query definition
        ↓
safe reusable
```

mientras:

```text
query execution context
        ↓
scope-local
```

---

# 109. Compiled Query

Puede ser worker-shared si está correctamente ligado a:

```text
platform
capabilities
metadata generation
schema assumptions
parameter layout
```

y no contiene valores request-specific inseguros.

---

# 110. Parameter values

Los valores reales:

```text
email = alice@example...
tenantId = ...
```

no deberán persistir accidentalmente en compiled query caches.

---

# 111. Hydration Plan

Puede ser reutilizable.

---

# 112. Hydration Session

No.

```text
HydrationPlan
    worker/application reusable

HydrationSession
    execution/query scoped
```

---

# 113. Relationship loaders

Los planes pueden ser compartidos.

Las colas de relaciones pendientes:

```text
request scoped
```

---

# 114. N+1 Detector

Configuración/reglas pueden compartirse.

Estado de detección:

```text
execution scoped
```

---

# 115. Query Profiler

El profiler actual deberá resetearse por execution.

---

# 116. Debug Information

Debug collectors deberán resetearse por execution.

---

# 117. Query Audit

Audit context:

```text
actor
tenant
request
operation
```

deberá ser scope-local.

---

# 118. Telemetry

Telemetry provider puede ser worker-safe.

Pero span/context actual:

```text
execution-local
```

---

# 119. Event System

Listeners pueden ser shared si son stateless.

Event context no.

---

# 120. Event listener state

Listeners con estado mutable request-specific deberán ser scoped.

---

# 121. Retry Context

Retry counters:

```text
operation scoped
```

No:

```text
worker global retry counter
```

---

# 122. Circuit Breaker

Aquí existe una distinción importante.

Circuit breaker puede ser:

```text
worker shared
```

porque representa salud de un recurso.

Pero:

```text
current operation attempt
```

es local.

---

# 123. Resource Governance

Pools y semáforos pueden ser worker-wide.

Leases y budgets:

```text
execution/operation scoped
```

---

# 124. Memory Management

Persistent runtime hace especialmente importante:

```text
retained memory
```

no solo peak memory.

---

# 125. Memory lifecycle

Ideal:

```text
Request
    ↓
allocate temporary state
    ↓
response
    ↓
release references
    ↓
GC/reuse
```

---

# 126. Memory leak definition

No toda memoria que permanece asignada al proceso es un leak.

Importa distinguir:

```text
allocator retained capacity
```

de:

```text
reachable stale request objects
```

---

# 127. Logical memory leak

Especialmente grave:

```text
static array
    ↓
stores every RequestContext
```

aunque el proceso aún tenga memoria disponible.

---

# 128. Scope Leak Detector

En development/testing podrá existir:

```text
DatabaseScopeLeakDetector
```

---

# 129. Leak detector targets

Podrá verificar:

```text
active transactions
active cursors
active connection leases
managed entities
pending changesets
lazy loaders
resource permits
scope references
```

---

# 130. End-of-request assertions

En modo testing:

```text
assertNoActiveTransaction()
assertNoOpenCursor()
assertNoConnectionLease()
assertEmptyIdentityMap()
assertEmptyUnitOfWork()
assertNoResourceLease()
```

---

# 131. Development warnings

Ejemplo:

```text
VoltStack Database Runtime Warning

Request completed with:
- 1 active transaction
- 2 open cursors
- 1 unreleased connection lease

Worker reuse has been blocked.
```

---

# 132. Production behavior

En producción se priorizará:

```text
safe cleanup
bounded telemetry
worker recycling
```

evitando diagnostics excesivamente caros.

---

# 133. Finalization Policy

```php
enum ScopeFinalizationPolicy
{
    case STRICT;
    case STANDARD;
    case DEVELOPMENT;
    case CUSTOM;
}
```

---

# 134. STRICT

Puede tratar cualquier recurso no cerrado como error severo.

---

# 135. STANDARD

Intenta cleanup seguro y reporta anomalías.

---

# 136. DEVELOPMENT

Además produce diagnósticos extensos.

---

# 137. Request failure

Una excepción de aplicación no elimina la obligación de cleanup.

```text
try
    execute request
finally
    finalize database scope
```

---

# 138. Fatal conditions

Cuando el runtime permita hooks, VoltStack intentará cleanup.

Pero no se prometerá recuperación imposible ante:

```text
process crash
SIGKILL
OOM kill
hardware failure
```

---

# 139. Cleanup ≠ Recovery

Cleanup restaura recursos locales.

Recovery puede requerir:

```text
transaction recovery
connection discard
worker recycle
external reconciliation
```

---

# 140. Database State Isolation

Será desarrollado específicamente en:

```text
254_DATABASE_STATE_ISOLATION_SYSTEM.md
```

---

# 141. State Reset

Será desarrollado en:

```text
255_DATABASE_STATE_RESET_SYSTEM.md
```

---

# 142. Connection Reuse

Será desarrollado en:

```text
256_DATABASE_CONNECTION_REUSE_SYSTEM.md
```

---

# 143. Worker Lifecycle

Será desarrollado en:

```text
257_DATABASE_WORKER_LIFECYCLE_SYSTEM.md
```

---

# 144. Runtime integrations

Posteriormente:

```text
258 FrankenPHP
259 RoadRunner
260 OpenSwoole
```

---

# 145. Persistent Runtime State Matrix

| Estado | Application | Worker | Execution | Operation | Transaction |
|---|---:|---:|---:|---:|---:|
| TypeRegistry | ✓ | ✓ | — | — | — |
| Dialect | ✓ | ✓ | — | — | — |
| Compiled Metadata | ✓ | ✓ | — | — | — |
| Query Compiler | ✓ | ✓ | — | — | — |
| Connection Pool | — | ✓ | — | — | — |
| EntityManager | — | — | ✓ | — | — |
| IdentityMap | — | — | ✓ | — | — |
| UnitOfWork | — | — | ✓ | — | — |
| TenantContext | — | — | ✓ | ✓ | — |
| ShardContext | — | — | ✓ | ✓ | — |
| QueryContext | — | — | — | ✓ | — |
| RetryContext | — | — | — | ✓ | — |
| TransactionContext | — | — | — | — | ✓ |
| ConnectionLease | — | — | ✓ | ✓ | ✓ |
| ActiveCursor | — | — | — | ✓ | ✓ |
| ResourceLease | — | — | ✓ | ✓ | — |

La tabla representa defaults conceptuales; componentes especializados podrán definir lifetimes más estrictos.

---

# 146. Scope compatibility

Un objeto de scope largo no deberá almacenar referencia directa a objeto de scope corto.

Ejemplo prohibido:

```text
WorkerSingleton
    ↓
$currentEntityManager
```

porque:

```text
Worker
>
Request
```

---

# 147. Captive Dependency

Esto se denominará:

```text
Captive Scoped Dependency
```

---

# 148. Dependency rule

Si:

```text
Lifetime(A) > Lifetime(B)
```

entonces:

```text
A must not retain B
```

salvo resolverlo dinámicamente dentro del scope válido mediante contrato seguro.

---

# 149. Scope Validator

El Container de VoltStack podrá validar estas dependencias.

---

# 150. Example

Incorrecto:

```php
final class DatabaseManager
{
    public function __construct(
        private EntityManager $entityManager,
    ) {}
}
```

si `DatabaseManager` es worker singleton y EntityManager request-scoped.

---

# 151. Correct pattern

```php
final class DatabaseManager
{
    public function __construct(
        private EntityManagerResolver $resolver,
    ) {}
}
```

si el resolver es stateless y obtiene el manager del execution scope actual.

---

# 152. Resolver safety

El resolver deberá fallar si no existe scope.

No deberá crear silenciosamente un contexto global.

---

# 153. Outside-scope access

Resultado:

```text
DatabaseScopeNotActiveException
```

---

# 154. Nested execution scopes

Por default:

```text
Request Scope
```

no deberá reemplazarse arbitrariamente por otro Request Scope.

---

# 155. Explicit nested operations

Sí podrán existir:

```text
Request
└── Operation Scope
    └── Transaction Scope
```

---

# 156. Scope token

Los recursos podrán registrar:

```text
ScopeToken
```

para verificar ownership.

---

# 157. Resource usage validation

Conceptualmente:

```php
$lease->assertOwnedBy($currentScope);
```

---

# 158. Cross-scope use

Deberá producir error:

```text
CrossScopeResourceAccessException
```

---

# 159. Connection pooling

Un objeto físico de conexión puede ser worker-level.

Pero su lease es scope-local.

```text
PhysicalConnection
    ↓
Pool
    ↓
Lease
    ↓
Request
```

---

# 160. Physical connection identity

La conexión no deberá asumir que sigue perteneciendo al tenant/request anterior después de reset.

---

# 161. Session state fingerprint

Podrá mantenerse:

```text
ConnectionStateFingerprint
```

para ayudar a decidir si una conexión puede reutilizarse.

---

# 162. State generation

Cambios relevantes pueden incrementar:

```text
ConnectionStateGeneration
```

---

# 163. Dirty connection

Una conexión modificada podrá quedar:

```text
DIRTY
```

hasta reset.

---

# 164. Connection lifecycle

```text
CREATED
   ↓
IDLE
   ↓
LEASED
   ↓
ACTIVE
   ↓
RETURNING
   ↓
RESETTING
   ↓
IDLE
```

o:

```text
TAINTED
   ↓
DISCARDED
```

---

# 165. Worker shutdown

Al detener worker:

```text
reject new DB operations
      ↓
cancel/drain active work
      ↓
close cursors
      ↓
finalize transactions
      ↓
release/discard connections
      ↓
flush telemetry
      ↓
close pools
```

---

# 166. Graceful shutdown

Se intentará respetar:

```text
shutdown deadline
```

---

# 167. Shutdown timeout

Si el deadline vence:

```text
remaining resources
    ↓
forced disposal
```

según capacidades del runtime.

---

# 168. Runtime Capabilities

```php
final readonly class RuntimeCapabilities
{
    public function supportsWorkerLifecycle(): bool;

    public function supportsConcurrentExecutions(): bool;

    public function supportsExecutionLocalStorage(): bool;

    public function supportsGracefulRecycle(): bool;

    public function supportsCancellation(): bool;
}
```

---

# 169. Capability-driven integration

Nunca:

```php
if ($runtime === 'openswoole') {
    ...
}
```

en el núcleo.

Preferencia:

```php
if ($runtimeCapabilities->supportsConcurrentExecutions()) {
    ...
}
```

---

# 170. Runtime-specific behavior

Sólo adapter:

```text
FrankenPHP Adapter
RoadRunner Adapter
OpenSwoole Adapter
```

conoce APIs concretas.

---

# 171. Runtime abstraction limit

VoltStack no deberá fingir que todos los runtimes son idénticos.

Si una capability no existe:

```text
UNSUPPORTED
```

o estrategia alternativa explícita.

---

# 172. Persistent Runtime Safety Levels

Se propone:

```text
SAFE
SAFE_WITH_RESET
REQUIRES_ISOLATION
UNSUPPORTED
UNKNOWN
```

---

# 173. UNKNOWN

Nunca:

```text
UNKNOWN = SAFE
```

---

# 174. Request completion barrier

Antes de entregar worker al siguiente request deberá cumplirse:

```text
DatabaseExecutionScope
    =
FINALIZED
```

---

# 175. Scope finalization barrier

Formalmente:

```text
Request(n+1)
```

no deberá iniciar con estado mutable de:

```text
Request(n)
```

visible desde Database.

---

# 176. Request isolation invariant

Para dos requests distintos:

```text
R1 ≠ R2
```

deberá cumplirse:

```text
MutableContext(R1)
∩
MutableContext(R2)
=
∅
```

excepto recursos explícitamente compartidos y diseñados para ello.

---

# 177. Shared immutable state

Sí puede existir:

```text
ImmutableState(R1)
∩
ImmutableState(R2)
≠
∅
```

Ejemplo:

```text
compiled metadata
```

---

# 178. Shared mutable infrastructure

También puede existir infraestructura mutable compartida:

```text
connection pool
cache
circuit breaker
```

pero deberá tener semántica explícita de concurrencia y nunca representar contexto de usuario/request.

---

# 179. Persistent Runtime Testing

La suite deberá simular:

```text
Request A
cleanup
Request B
cleanup
Request C
```

dentro del mismo proceso.

---

# 180. Isolation Test

Ejemplo:

```text
Request A:
tenant = ACME
load User#10

Request B:
tenant = CONTOSO
load User#10
```

Verificar:

```text
different persistence domain
no shared managed instance
no tenant leakage
```

---

# 181. Transaction leak test

```text
Request A
BEGIN
throw exception

Request B
SELECT ...
```

Debe verificarse que B no hereda transaction A.

---

# 182. UnitOfWork leak test

```text
Request A
modify entity
no flush

Request B
flush()
```

La modificación A nunca deberá persistirse.

---

# 183. IdentityMap leak test

La misma identidad lógica en scopes diferentes no implica misma instancia.

```text
instance(R1, User#1)
!== 
instance(R2, User#1)
```

---

# 184. Cursor leak test

Request A deja cursor abierto.

Finalization debe:

```text
close
or
taint/recycle
```

---

# 185. Connection leak test

Después de N requests:

```text
leased connections = 0
```

al finalizar cada scope, salvo recursos explícitamente activos fuera del modelo normal.

---

# 186. Memory stability test

Ejecutar:

```text
10
100
1K
10K
100K
```

requests simulados.

Medir:

```text
reachable request state
retained memory
pool size
cache size
```

---

# 187. Concurrent isolation test

Para runtimes concurrentes:

```text
Execution A → Tenant A
Execution B → Tenant B
```

intercaladas.

Nunca:

```text
A sees B context
B sees A context
```

---

# 188. Failure injection

Testing deberá inyectar:

```text
connection loss
commit uncertainty
rollback failure
cursor failure
cleanup failure
timeout
cancellation
OOM-like resource pressure
```

cuando sea viable.

---

# 189. Benchmark integration

Documento 250 medirá:

```text
request scope creation
scope finalization
state reset
connection reuse
worker memory growth
```

---

# 190. Telemetry

Métricas sugeridas:

```text
database.runtime.execution.started
database.runtime.execution.completed
database.runtime.execution.failed
database.runtime.cleanup.duration
database.runtime.cleanup.failure
database.runtime.resource.leak
database.runtime.worker.recycle
database.runtime.connection.discard
database.runtime.scope.violation
```

---

# 191. Cardinality

No usar como label sin control:

```text
requestId
userId
query SQL
```

en métricas agregadas.

---

# 192. Trace correlation

Trace/span puede incluir:

```text
execution.id
runtime
worker.id
scope.type
```

bajo políticas de cardinalidad.

---

# 193. Debug information

Development podrá mostrar:

```text
Runtime: FrankenPHP
Worker: worker-3
Execution: req-123

Database scope:
  EntityManager: CLOSED
  IdentityMap: 0
  UnitOfWork: 0
  Transactions: 0
  Connection leases: 0
  Cursors: 0
  Resource leases: 0

Cleanup:
  CLEAN
```

---

# 194. Scope leak diagnostic

Ejemplo:

```text
Database Scope Leak Detected

Execution:
req-123

Remaining resources:
Transaction #tx-7
Connection lease #conn-3
ResultCursor #cursor-9

Disposition:
WORKER_RECYCLE_REQUIRED
```

---

# 195. Performance

Scope safety deberá tener bajo overhead.

El hot path normal:

```text
create scope
use DB
finalize scope
```

deberá ser eficiente.

---

# 196. Correctness before micro-optimization

Nunca eliminar scope checks críticos únicamente para ahorrar unos nanosegundos sin evidencia.

---

# 197. Fast path

Production podrá usar:

```text
compiled scope metadata
precomputed reset plans
lightweight ownership tokens
```

---

# 198. Reset plan

Cada scope podrá tener:

```text
DatabaseResetPlan
```

precalculado según servicios activos.

---

# 199. Lazy reset registration

No es necesario resetear servicios que nunca fueron usados.

Ejemplo:

```text
Request without DB
    ↓
zero ORM cleanup work
```

---

# 200. Activated component tracking

El scope podrá registrar únicamente:

```text
activated components
```

---

# 201. Cleanup complexity

Idealmente proporcional a:

```text
resources actually used
```

y no a todos los servicios posibles.

---

# 202. Persistent Runtime Security

El problema es también de seguridad.

State leakage puede exponer:

```text
tenant data
user data
credentials
authorization context
query parameters
audit identity
```

---

# 203. Sensitive reference cleanup

Objetos con información sensible deberán liberar referencias al terminar scope.

---

# 204. Credential lifetime

Las credenciales deberán tener lifetime mínimo necesario.

---

# 205. Secrets in caches

Nunca almacenar secretos en:

```text
compiled query cache
metadata cache
debug state
benchmark report
```

---

# 206. Security context drift

Una query creada bajo un usuario y ejecutada bajo otro contexto deberá respetar las reglas del Query Security System.

---

# 207. Authorization ≠ captured forever

No deberá serializarse una autorización antigua dentro de un objeto reutilizable sin política explícita.

---

# 208. Serialization

Objetos runtime activos no deberán serializarse.

Ejemplos:

```text
EntityManager
ConnectionLease
TransactionContext
ResultCursor
DatabaseContext
```

---

# 209. Jobs

Para transferir trabajo a un Job se serializa:

```text
operation specification
entity identifier
tenant identifier
query definition when safe
```

no:

```text
live EntityManager
live connection
active cursor
```

---

# 210. Fork/process boundaries

Recursos DB abiertos antes de fork no deberán asumirse seguros después del fork salvo soporte explícito del driver/runtime.

---

# 211. Hot reload

En desarrollo, reload de código/configuración puede invalidar:

```text
metadata
compiled queries
platform capabilities
connection configuration
```

---

# 212. Generation model

Podrán existir generaciones:

```text
ApplicationGeneration
MetadataGeneration
ConfigurationGeneration
SchemaGeneration
```

---

# 213. Generation mismatch

Un cache generado para:

```text
MetadataGeneration = 12
```

no deberá reutilizarse ciegamente con:

```text
MetadataGeneration = 13
```

---

# 214. Worker staleness

Workers demasiado antiguos podrán reciclarse tras cambio relevante de generación.

---

# 215. Database Persistent Runtime Manager

Coordinador principal:

```text
DatabasePersistentRuntimeManager
```

---

# 216. Responsabilidades

```text
runtime registration
scope creation
scope finalization
resource tracking
reset coordination
worker disposition
runtime diagnostics
```

---

# 217. No God Object

No deberá implementar:

```text
ORM
transactions
connection pooling
query execution
```

Sólo coordinar lifecycle.

---

# 218. Proposed API

```php
interface DatabasePersistentRuntimeManager
{
    public function beginExecution(
        ExecutionDescriptor $execution
    ): DatabaseExecutionScope;

    public function finishExecution(
        DatabaseExecutionScope $scope,
        ExecutionOutcome $outcome
    ): ScopeFinalizationResult;
}
```

---

# 219. ExecutionDescriptor

```php
final readonly class ExecutionDescriptor
{
    public function __construct(
        public ExecutionId $id,
        public ExecutionType $type,
        public RuntimeId $runtime,
        public ?Deadline $deadline,
    ) {}
}
```

---

# 220. ExecutionOutcome

```php
enum ExecutionOutcome
{
    case SUCCESS;
    case FAILURE;
    case CANCELLED;
    case TIMEOUT;
    case UNKNOWN;
}
```

---

# 221. Finalization result

```php
final readonly class ScopeFinalizationResult
{
    public function __construct(
        public ScopeCleanupStatus $status,
        public WorkerDisposition $worker,
        public array $resourceResults,
    ) {}
}
```

---

# 222. WorkerDisposition

```php
enum WorkerDisposition
{
    case REUSE;
    case REUSE_WITH_WARNING;
    case RECYCLE;
    case TERMINATE;
}
```

---

# 223. Runtime failure model

```text
Application Failure
Database Failure
Cleanup Failure
Runtime Failure
Worker Failure
```

serán categorías distintas.

---

# 224. Application exception

No implica automáticamente:

```text
worker corrupted
```

si cleanup demuestra estado limpio.

---

# 225. Database cleanup failure

Puede requerir:

```text
RECYCLE
```

aunque la respuesta HTTP ya haya sido generada.

---

# 226. Response ≠ cleanup completion

Idealmente el runtime lifecycle deberá garantizar cleanup antes de considerar el worker reutilizable.

---

# 227. Streaming HTTP responses

Requieren cuidado porque:

```text
response lifetime
```

puede prolongar:

```text
database resource lifetime
```

---

# 228. Database cursor + streamed response

Puede mantener conexión durante streaming.

Debe ser explícito y gobernado.

---

# 229. Request end definition

Para Database:

```text
request ended
```

significa:

```text
no more application DB work allowed
+
scope finalization initiated
```

no necesariamente que bytes HTTP ya hayan abandonado completamente el sistema.

---

# 230. Background work

Trabajo iniciado dentro de request pero continuado después deberá migrar a un scope independiente.

---

# 231. Forbidden detached async work

Incorrecto:

```text
Request
  ↓
spawn async callback
  ↓
capture EntityManager
  ↓
Request ends
  ↓
callback continues
```

---

# 232. Correct async handoff

```text
Request
  ↓
create operation specification
  ↓
background execution
  ↓
new ExecutionScope
  ↓
new DatabaseContext
```

---

# 233. Scope transfer

No se transferirán recursos vivos.

Se transferirán:

```text
identifiers
immutable specifications
safe context descriptors
```

---

# 234. Persistent Runtime architecture

```text
                     VOLTSTACK APPLICATION
                              │
                              ▼
                      DATABASE BOOTSTRAP
                              │
                   ┌──────────┴──────────┐
                   ▼                     ▼
            Immutable Services     Runtime Adapter
                   │                     │
                   │                     ▼
                   │               Worker Lifecycle
                   │                     │
                   └──────────┬──────────┘
                              ▼
                        WORKER SCOPE
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
            ▼                 ▼                 ▼
      Connection Pool   Shared Caches    Runtime Services
            │
            └─────────────────┬─────────────────┘
                              │
                              ▼
                      EXECUTION START
                              │
                              ▼
                    DATABASE EXECUTION SCOPE
                              │
       ┌──────────────────────┼──────────────────────┐
       ▼                      ▼                      ▼
 EntityManager          DatabaseContext       Resource Registry
       │                      │                      │
 ┌─────┴─────┐          ┌─────┼─────┐          ┌─────┼─────┐
 ▼           ▼          ▼     ▼     ▼          ▼     ▼     ▼
UoW      IdentityMap Tenant Shard Routing  ConnLease Cursor Permit
       │                      │                      │
       └──────────────────────┼──────────────────────┘
                              ▼
                       DATABASE OPERATIONS
                              │
                              ▼
                       EXECUTION FINISH
                              │
                              ▼
                     FINALIZATION BARRIER
                              │
                              ▼
                         RESET / CLEANUP
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
                  CLEAN              TAINTED
                    │                   │
                    ▼                   ▼
              REUSE WORKER        RECYCLE WORKER
```

---

# 235. Directory proposal

```text
src/Quantum/Database/Runtime/
│
├── Contract/
│   ├── PersistentRuntimeAdapter.php
│   ├── DatabaseRuntimeLifecycle.php
│   ├── RuntimeLocalStorage.php
│   └── ScopedDatabaseResource.php
│
├── Context/
│   ├── DatabaseContext.php
│   ├── DatabaseExecutionScope.php
│   ├── ExecutionDescriptor.php
│   ├── ExecutionId.php
│   ├── ExecutionGeneration.php
│   └── ScopeToken.php
│
├── Lifecycle/
│   ├── DatabasePersistentRuntimeManager.php
│   ├── DatabaseWorkerLifecycle.php
│   ├── DatabaseExecutionLifecycle.php
│   └── ScopeFinalizer.php
│
├── Resource/
│   ├── DatabaseScopeResourceRegistry.php
│   ├── ScopedResourceEntry.php
│   ├── ResourceOwnership.php
│   └── ResourceDisposition.php
│
├── Reset/
│   ├── DatabaseResetPlan.php
│   ├── DatabaseResetCoordinator.php
│   └── ResetResult.php
│
├── Isolation/
│   ├── ScopeIsolationValidator.php
│   ├── ScopeOwnershipValidator.php
│   └── DatabaseScopeLeakDetector.php
│
├── Worker/
│   ├── WorkerContext.php
│   ├── WorkerDisposition.php
│   └── WorkerRecyclePolicy.php
│
├── Capability/
│   ├── RuntimeCapabilities.php
│   └── RuntimeScopeSafety.php
│
├── Telemetry/
│   └── DatabaseRuntimeTelemetry.php
│
├── FrankenPHP/
│   └── FrankenPhpRuntimeAdapter.php
│
├── RoadRunner/
│   └── RoadRunnerRuntimeAdapter.php
│
├── OpenSwoole/
│   └── OpenSwooleRuntimeAdapter.php
│
└── Exception/
    ├── DatabaseRuntimeException.php
    ├── DatabaseScopeNotActiveException.php
    ├── StaleDatabaseContextException.php
    ├── CrossScopeResourceAccessException.php
    ├── DatabaseScopeLeakException.php
    ├── DatabaseCleanupException.php
    └── RuntimeIsolationException.php
```

---

# 236. Architectural Invariants

## DB-RUNTIME-001

`Process Lifetime ≠ Request Lifetime`.

## DB-RUNTIME-002

Persistent worker no implicará persistent transaction.

## DB-RUNTIME-003

Persistent worker no implicará persistent EntityManager state.

## DB-RUNTIME-004

Persistent worker no implicará persistent IdentityMap.

## DB-RUNTIME-005

Persistent worker no implicará persistent UnitOfWork.

## DB-RUNTIME-006

Todo estado mutable tendrá owner.

## DB-RUNTIME-007

Todo estado mutable tendrá scope.

## DB-RUNTIME-008

Request state no sobrevivirá accidentalmente al request.

## DB-RUNTIME-009

Application scope contendrá preferentemente estado inmutable.

## DB-RUNTIME-010

Worker scope no almacenará current request state.

## DB-RUNTIME-011

IdentityMap será execution-scoped por default.

## DB-RUNTIME-012

UnitOfWork será execution-scoped por default.

## DB-RUNTIME-013

EntityManager mutable state será execution-scoped.

## DB-RUNTIME-014

TransactionContext será transaction/execution scoped.

## DB-RUNTIME-015

TenantContext no será worker-global.

## DB-RUNTIME-016

ShardContext no será worker-global.

## DB-RUNTIME-017

QueryContext no será worker-global.

## DB-RUNTIME-018

RetryContext no será worker-global.

## DB-RUNTIME-019

Current connection lease no será worker-global.

## DB-RUNTIME-020

Active cursor no será worker-global.

## DB-RUNTIME-021

Lazy iterator activo no sobrevivirá scope sin ownership explícito.

## DB-RUNTIME-022

DatabaseContext no será global singleton mutable.

## DB-RUNTIME-023

Scope resolver fallará fuera de scope.

## DB-RUNTIME-024

Resolver no creará silenciosamente global context.

## DB-RUNTIME-025

Stale context deberá detectarse cuando sea viable.

## DB-RUNTIME-026

Boot será distinto de request initialization.

## DB-RUNTIME-027

Bootstrap no creará request state.

## DB-RUNTIME-028

Conexiones serán adquiridas lazy cuando sea apropiado.

## DB-RUNTIME-029

Connection reuse será distinto de connection state reuse.

## DB-RUNTIME-030

Una conexión con transaction activa no volverá al pool normal.

## DB-RUNTIME-031

Transaction activa al terminar scope será resuelta explícitamente.

## DB-RUNTIME-032

Nunca se hará auto-commit de transaction desconocida.

## DB-RUNTIME-033

UNKNOWN transaction outcome permanecerá UNKNOWN.

## DB-RUNTIME-034

Tainted connection no será reutilizada ciegamente.

## DB-RUNTIME-035

Tainted EntityManager no será reutilizado.

## DB-RUNTIME-036

EntityManager será cerrado/finalizado por scope.

## DB-RUNTIME-037

IdentityMap será limpiado al terminar scope.

## DB-RUNTIME-038

UnitOfWork será limpiado al terminar scope.

## DB-RUNTIME-039

ChangeSet de un request no llegará al siguiente.

## DB-RUNTIME-040

Managed entity no seguirá managed después de cerrar su manager.

## DB-RUNTIME-041

Detached entity no resolverá automáticamente un nuevo manager global.

## DB-RUNTIME-042

Lazy loading fuera de scope fallará explícitamente.

## DB-RUNTIME-043

Active cursors serán cerrados durante finalization.

## DB-RUNTIME-044

Connection leases serán liberados durante finalization.

## DB-RUNTIME-045

Resource permits serán liberados durante finalization.

## DB-RUNTIME-046

Cleanup será determinista.

## DB-RUNTIME-047

Destructor no será mecanismo primario de correctness.

## DB-RUNTIME-048

Cleanup order respetará dependencias.

## DB-RUNTIME-049

Cleanup deberá ser idempotente cuando sea viable.

## DB-RUNTIME-050

Cleanup failure será observable.

## DB-RUNTIME-051

Estado no demostrablemente limpio podrá provocar worker recycle.

## DB-RUNTIME-052

Worker recycle será distinto de normal reset.

## DB-RUNTIME-053

Database Core no dependerá directamente de FrankenPHP.

## DB-RUNTIME-054

Database Core no dependerá directamente de RoadRunner.

## DB-RUNTIME-055

Database Core no dependerá directamente de OpenSwoole.

## DB-RUNTIME-056

Runtimes se integrarán mediante adapters.

## DB-RUNTIME-057

FrankenPHP será implementación de referencia.

## DB-RUNTIME-058

OpenSwoole deberá soportar isolation concurrente.

## DB-RUNTIME-059

Context local storage será execution-aware.

## DB-RUNTIME-060

Context local storage no será global mutable state.

## DB-RUNTIME-061

Dos ejecuciones concurrentes no compartirán contexto mutable.

## DB-RUNTIME-062

Worker singleton mutable será evitado.

## DB-RUNTIME-063

Metadata immutable podrá compartirse.

## DB-RUNTIME-064

Entity instances managed no serán worker-shared.

## DB-RUNTIME-065

Entity Cache seguirá siendo distinto de IdentityMap.

## DB-RUNTIME-066

Cache keys deberán preservar tenant/domain identity.

## DB-RUNTIME-067

ConnectionManager será distinto de CurrentConnection.

## DB-RUNTIME-068

ConnectionLease tendrá owner.

## DB-RUNTIME-069

ConnectionLease no cruzará scopes accidentalmente.

## DB-RUNTIME-070

Transaction mantendrá connection affinity.

## DB-RUNTIME-071

Sticky routing será scope-local.

## DB-RUNTIME-072

Replica routing state será scope-local.

## DB-RUNTIME-073

Shard routing state será scope-local.

## DB-RUNTIME-074

Tenant switching será explícito.

## DB-RUNTIME-075

Cross-tenant operation será explícita.

## DB-RUNTIME-076

Security context será scope-aware.

## DB-RUNTIME-077

Query execution context será scope-local.

## DB-RUNTIME-078

Query definition será distinta de execution state.

## DB-RUNTIME-079

Reusable Query Model no contendrá live connection.

## DB-RUNTIME-080

Reusable compiled query no contendrá sensitive parameter values.

## DB-RUNTIME-081

Hydration Plan podrá ser reusable.

## DB-RUNTIME-082

Hydration Session será scope-local.

## DB-RUNTIME-083

Relationship pending-load queues serán scope-local.

## DB-RUNTIME-084

N+1 detector execution state será scope-local.

## DB-RUNTIME-085

Profiler state será reseteado por execution.

## DB-RUNTIME-086

Debug state será reseteado por execution.

## DB-RUNTIME-087

Audit actor/context será scope-local.

## DB-RUNTIME-088

Telemetry current span/context será execution-local.

## DB-RUNTIME-089

Listener mutable request state será scoped.

## DB-RUNTIME-090

Retry counter será operation-local.

## DB-RUNTIME-091

Circuit breaker podrá ser worker-shared cuando represente recurso compartido.

## DB-RUNTIME-092

Resource pools podrán ser worker-shared.

## DB-RUNTIME-093

Resource leases serán scope-local.

## DB-RUNTIME-094

Persistent runtime deberá medir retained memory.

## DB-RUNTIME-095

Allocator retained memory no se confundirá automáticamente con leak.

## DB-RUNTIME-096

Reachable stale request objects serán considerados leak lógico.

## DB-RUNTIME-097

Development podrá detectar scope leaks.

## DB-RUNTIME-098

Testing podrá exigir zero active transaction after scope.

## DB-RUNTIME-099

Testing podrá exigir zero active cursor after scope.

## DB-RUNTIME-100

Testing podrá exigir zero connection lease after scope.

## DB-RUNTIME-101

Testing podrá exigir empty IdentityMap after finalization.

## DB-RUNTIME-102

Testing podrá exigir empty UnitOfWork after finalization.

## DB-RUNTIME-103

Application exception no omitirá cleanup.

## DB-RUNTIME-104

Cleanup no será confundido con recovery.

## DB-RUNTIME-105

Runtime capability checks serán preferidos a vendor conditionals.

## DB-RUNTIME-106

UNKNOWN runtime safety no equivaldrá a SAFE.

## DB-RUNTIME-107

Request siguiente no iniciará antes del finalization barrier correspondiente.

## DB-RUNTIME-108

Mutable state de requests distintos permanecerá aislado.

## DB-RUNTIME-109

Immutable state podrá compartirse.

## DB-RUNTIME-110

Shared mutable infrastructure deberá ser concurrency-safe.

## DB-RUNTIME-111

Long-lived service no retendrá short-lived dependency.

## DB-RUNTIME-112

Captive scoped dependency será detectable.

## DB-RUNTIME-113

Scope token podrá validar ownership.

## DB-RUNTIME-114

Cross-scope resource use producirá error.

## DB-RUNTIME-115

Physical connection y ConnectionLease serán conceptos distintos.

## DB-RUNTIME-116

Connection session state será reseteado antes de safe reuse.

## DB-RUNTIME-117

Dirty connection no será tratada como clean.

## DB-RUNTIME-118

Tainted connection podrá ser descartada.

## DB-RUNTIME-119

Worker shutdown rechazará nuevas operaciones.

## DB-RUNTIME-120

Worker shutdown cerrará recursos.

## DB-RUNTIME-121

Graceful shutdown respetará deadline cuando exista.

## DB-RUNTIME-122

Runtime adapter declarará capabilities.

## DB-RUNTIME-123

Core no fingirá capabilities inexistentes.

## DB-RUNTIME-124

Scope creation tendrá overhead acotado.

## DB-RUNTIME-125

Cleanup deberá ser proporcional a recursos activados cuando sea posible.

## DB-RUNTIME-126

Request sin Database no requerirá inicializar ORM completo.

## DB-RUNTIME-127

Sensitive references serán liberadas.

## DB-RUNTIME-128

Credentials no se almacenarán en caches de compilación.

## DB-RUNTIME-129

Runtime objects activos no serán serializados.

## DB-RUNTIME-130

Jobs recibirán specifications, no live DB resources.

## DB-RUNTIME-131

Background work tendrá scope independiente.

## DB-RUNTIME-132

Async callback no capturará EntityManager expirado.

## DB-RUNTIME-133

Fork boundaries no asumirán connection safety.

## DB-RUNTIME-134

Generation mismatch podrá invalidar caches.

## DB-RUNTIME-135

Worker stale podrá reciclarse.

## DB-RUNTIME-136

PersistentRuntimeManager coordinará, no implementará ORM.

## DB-RUNTIME-137

PersistentRuntimeManager no implementará transaction engine.

## DB-RUNTIME-138

PersistentRuntimeManager no implementará connection pool.

## DB-RUNTIME-139

ExecutionOutcome será explícito.

## DB-RUNTIME-140

ScopeFinalizationResult será explícito.

## DB-RUNTIME-141

WorkerDisposition será explícito.

## DB-RUNTIME-142

Application failure será distinto de cleanup failure.

## DB-RUNTIME-143

Response generation no eliminará obligación de cleanup.

## DB-RUNTIME-144

Streaming DB resources tendrán ownership explícito.

## DB-RUNTIME-145

Detached background operation creará nuevo scope.

## DB-RUNTIME-146

Scope transfer no transferirá live resources.

## DB-RUNTIME-147

Tenant isolation será probado en persistent worker.

## DB-RUNTIME-148

Transaction leakage será probado.

## DB-RUNTIME-149

UnitOfWork leakage será probado.

## DB-RUNTIME-150

IdentityMap leakage será probado.

## DB-RUNTIME-151

Cursor leakage será probado.

## DB-RUNTIME-152

Connection leakage será probado.

## DB-RUNTIME-153

Memory stability será probado.

## DB-RUNTIME-154

Concurrent scope isolation será probado.

## DB-RUNTIME-155

Failure injection será soportado en testing.

## DB-RUNTIME-156

Runtime lifecycle tendrá telemetry.

## DB-RUNTIME-157

Telemetry evitará cardinalidad descontrolada.

## DB-RUNTIME-158

Debug diagnostics podrán mostrar estado de cleanup.

## DB-RUNTIME-159

Performance benchmarks medirán scope overhead.

## DB-RUNTIME-160

Correctness tendrá prioridad sobre micro-optimización del lifecycle.

## DB-RUNTIME-161

No habrá static `$currentTenant`.

## DB-RUNTIME-162

No habrá static `$currentEntityManager`.

## DB-RUNTIME-163

No habrá static `$currentTransaction`.

## DB-RUNTIME-164

No habrá static `$currentConnection` contextual.

## DB-RUNTIME-165

No habrá static `$currentShard`.

## DB-RUNTIME-166

No habrá static `$currentRequestDatabaseContext`.

## DB-RUNTIME-167

State reset será parte explícita del lifecycle.

## DB-RUNTIME-168

Connection reuse será capability/policy driven.

## DB-RUNTIME-169

Worker lifecycle será runtime-aware.

## DB-RUNTIME-170

Persistent runtime será una propiedad arquitectónica de Database desde su diseño inicial.

---

# 237. Modelo formal de aislamiento

Sea:

```text
W
```

un worker y:

```text
E1, E2, ..., En
```

sus ejecuciones.

Para cualquier par:

```text
Ei ≠ Ej
```

el estado mutable contextual deberá satisfacer:

```text
MutableState(Ei)
∩
MutableState(Ej)
=
∅
```

salvo infraestructura compartida explícitamente diseñada para concurrencia.

---

# 238. Modelo de recursos

Para todo recurso:

```text
r
```

deberá existir:

```text
Owner(r)
Scope(r)
State(r)
Disposition(r)
```

---

# 239. Regla de reutilización

Un recurso podrá reutilizarse sólo si:

```text
Reusable(r)
=
OwnershipReleased(r)
∧
StateClean(r)
∧
NotTainted(r)
∧
CompatibleWithNextScope(r)
```

---

# 240. UNKNOWN

Si:

```text
StateClean(r) = UNKNOWN
```

no deberá inferirse:

```text
Reusable(r) = true
```

---

# 241. Modelo de finalización

Sea:

```text
F(E)
```

la finalización de una ejecución.

Para permitir reutilización del worker:

```text
ReusableWorker(W)
```

deberá existir evidencia suficiente de:

```text
NoActiveTransactions(E)
∧
NoActiveCursors(E)
∧
NoLeasedConnections(E)
∧
NoPendingOrmState(E)
∧
NoLeakedResourceLeases(E)
∧
NoCriticalScopeReferences(E)
```

o una estrategia segura equivalente.

---

# 242. Modelo de dependencia por lifetime

Sean componentes:

```text
A
B
```

si:

```text
Lifetime(A) > Lifetime(B)
```

entonces A no deberá retener B directamente.

Formalmente:

```text
LongLived(A)
∧
ShortLived(B)
→
¬Retains(A,B)
```

salvo proxy/resolver scope-aware que no capture la instancia.

---

# 243. Modelo de persistent runtime de VoltStack

```text
VoltStack
   │
   ▼
Application
   │
   ▼
RuntimeManagerServer
   │
   ├── FrankenPHP
   ├── RoadRunner
   └── OpenSwoole
          │
          ▼
Persistent Runtime Adapter
          │
          ▼
Worker
          │
          ├── Shared Immutable Infrastructure
          │
          ├── Connection Infrastructure
          │
          └── Execution Lifecycle
                    │
              ┌─────┴─────┐
              ▼           ▼
           Request       Job/CLI
              │
              ▼
       DatabaseExecutionScope
              │
     ┌────────┼─────────┐
     ▼        ▼         ▼
   ORM     Context   Resources
     │        │         │
     └────────┼─────────┘
              ▼
        DB Operations
              │
              ▼
        Finalization
              │
        ┌─────┴─────┐
        ▼           ▼
      CLEAN       TAINTED
        │           │
        ▼           ▼
      REUSE       RECYCLE
```

---

# 244. Decisión arquitectónica

VoltStack Database será diseñado desde el inicio bajo la premisa:

```text
PHP process may be persistent
```

aunque también pueda ejecutarse en:

```text
traditional request-per-process environments
CLI
tests
short-lived scripts
```

Esto evita construir Database suponiendo:

```text
"PHP will clean everything for us"
```

---

# 245. Consecuencia

El modelo tradicional:

```text
request ends
    ↓
everything disappears
```

no será una dependencia de correctness.

VoltStack implementará explícitamente:

```text
ownership
scope
lifecycle
finalization
reset
isolation
resource release
```

---

# 246. Beneficio

Esto permitirá que el mismo núcleo Database funcione correctamente sobre:

```text
Traditional PHP
FrankenPHP
RoadRunner
OpenSwoole
CLI workers
Queue workers
Long-running processes
Tests
```

sin crear diferentes ORM engines.

---

# 247. Relación con RuntimeManagerServer

La separación conceptual será:

```text
VoltStack RuntimeManagerServer
        │
        │ administra runtime
        ▼
FrankenPHP / RoadRunner / OpenSwoole
        │
        │ emite lifecycle
        ▼
Database Runtime Adapter
        │
        │ crea/finaliza scope
        ▼
Quantum Database
```

Por tanto:

```text
RuntimeManagerServer
≠
Database Runtime Manager
```

El primero administra el servidor/runtime.

El segundo administra el lifecycle de Database dentro de ese runtime.

---

# 248. Regla final

> **La persistencia del proceso debe ser una optimización de infraestructura, nunca una relajación del aislamiento. VoltStack reutilizará código compilado, metadata, caches estructurales, pools y servicios seguros; no reutilizará accidentalmente identidades ORM, transacciones, tenant context, cursors, leases ni estado mutable perteneciente a una ejecución anterior.**

En forma resumida:

```text
Persistent Worker
      │
      ├── Reuse safe infrastructure
      │
      ├── Isolate mutable context
      │
      ├── Own every resource
      │
      ├── Finalize every execution
      │
      ├── Reset reusable state
      │
      └── Recycle when safety cannot be proven
```

---

# 249. Estado del Bloque 25

```text
BLOCK 25 — PERSISTENT RUNTIME

✓ 251_DATABASE_PERSISTENT_RUNTIME_ARCHITECTURE.md
│
├── 252_DATABASE_REQUEST_SCOPE_SYSTEM.md
├── 253_DATABASE_DATABASE_CONTEXT_SYSTEM.md
├── 254_DATABASE_STATE_ISOLATION_SYSTEM.md
├── 255_DATABASE_STATE_RESET_SYSTEM.md
├── 256_DATABASE_CONNECTION_REUSE_SYSTEM.md
├── 257_DATABASE_WORKER_LIFECYCLE_SYSTEM.md
├── 258_DATABASE_FRANKENPHP_INTEGRATION_SYSTEM.md
├── 259_DATABASE_ROADRUNNER_INTEGRATION_SYSTEM.md
└── 260_DATABASE_OPENSWOOLE_INTEGRATION_SYSTEM.md
```

---

# 250. Siguiente documento

```text
252_DATABASE_REQUEST_SCOPE_SYSTEM.md
```

El siguiente documento profundizará específicamente en:

```text
Request / Execution Scope
        │
        ├── creation
        ├── ownership
        ├── contextual services
        ├── EntityManager scope
        ├── IdentityMap scope
        ├── UnitOfWork scope
        ├── connection leases
        ├── transactions
        ├── tenant/shard context
        ├── nested operations
        ├── cancellation
        ├── finalization
        └── disposal
```

y establecerá formalmente cómo VoltStack creará una **frontera de aislamiento Database por cada request, job, comando o ejecución**, independientemente de cuánto tiempo permanezca vivo el worker PHP.