# 254_DATABASE_STATE_ISOLATION_SYSTEM.md

# VoltStack Quantum Database
## Database State Isolation System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 254 — Database State Isolation System  
**Bloque:** 25 — Persistent Runtime  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `253_DATABASE_DATABASE_CONTEXT_SYSTEM.md`  
**Siguiente documento:** `255_DATABASE_STATE_RESET_SYSTEM.md`

---

# 1. Propósito

Este documento define el **Database State Isolation System** de `VoltStack/Quantum/Database`.

Su objetivo es establecer las reglas, fronteras y mecanismos mediante los cuales VoltStack garantizará que el estado mutable producido durante una ejecución no pueda contaminar otra ejecución que utilice el mismo proceso, worker, runtime o infraestructura compartida.

La regla central es:

> **Compartir un proceso, worker, pool, metadata o infraestructura no implica compartir estado mutable de ejecución. Todo estado Database deberá tener un propietario, lifetime y frontera de aislamiento explícitos.**

Formalmente, para dos ejecuciones distintas:

```text
E₁ ≠ E₂
```

deberá cumplirse:

```text
MutableExecutionState(E₁)
∩
MutableExecutionState(E₂)
=
∅
```

salvo recursos cuya compartición haya sido explícitamente diseñada, sincronizada y declarada segura.

---

# 2. Motivación

En PHP tradicional suele existir:

```text
Request A
   ↓
PHP Process
   ↓
Process terminates
```

El sistema operativo elimina indirectamente gran parte del estado.

En un runtime persistente:

```text
Worker
│
├── Request A
├── Request B
├── Request C
├── Job D
└── Request E
```

el proceso puede vivir durante:

```text
minutes
hours
days
```

Por tanto, cualquier estado almacenado accidentalmente en:

```text
static properties
singletons
worker services
global variables
long-lived closures
connection objects
runtime locals incorrectly implemented
```

puede sobrevivir a la ejecución que lo creó.

---

# 3. Riesgo fundamental

Ejemplo incorrecto:

```php
final class TenantState
{
    public static ?string $tenant = null;
}
```

Request A:

```text
TenantState::$tenant = acme
```

Después llega Request B.

Si el valor no fue limpiado:

```text
Request B
    ↓
tenant = acme
```

aunque B pertenezca a otro tenant.

Esto puede convertirse en:

```text
data leak
authorization bypass
incorrect routing
cache poisoning
wrong writes
cross-tenant corruption
```

---

# 4. Reset ≠ Isolation

Distinción fundamental:

```text
State Isolation
≠
State Reset
```

Isolation responde:

> ¿Cómo impedimos que dos ejecuciones compartan estado mutable incorrectamente?

Reset responde:

> ¿Cómo dejamos limpio un recurso antes de reutilizarlo?

El documento `255_DATABASE_STATE_RESET_SYSTEM.md` definirá el segundo problema.

Este documento define el primero.

---

# 5. Isolation first

VoltStack seguirá el principio:

```text
Isolation
    ↓
primary correctness mechanism

Reset
    ↓
secondary lifecycle mechanism
```

No:

```text
everything global
    ↓
hopefully reset everything later
```

---

# 6. Scope ≠ Isolation Implementation

`DatabaseExecutionScope` define la frontera lógica.

Pero:

```text
Scope exists
```

no significa automáticamente:

```text
state is isolated
```

Cada componente deberá respetar esa frontera.

---

# 7. Context ≠ Isolation

Igualmente:

```text
DatabaseContext
```

describe el contexto efectivo.

Pero la arquitectura deberá impedir que servicios de otros scopes retengan ese contexto.

---

# 8. Objetivos

El sistema deberá garantizar aislamiento para:

1. DatabaseContext;
2. EntityManager;
3. IdentityMap;
4. UnitOfWork;
5. TransactionContext;
6. ConnectionLease;
7. connection session state;
8. routing state;
9. tenant state;
10. shard state;
11. consistency state;
12. query execution state;
13. prepared statement state cuando aplique;
14. ResultCursor;
15. StreamingResult;
16. LazyCollection;
17. Chunk processing;
18. import/export state;
19. retry state;
20. telemetry/profiling state;
21. N+1 detection state;
22. audit context;
23. cancellation state;
24. resource budgets;
25. temporary caches;
26. event execution state.

---

# 9. No objetivos

Este sistema no implementará:

```text
SQL generation
query planning
ORM persistence
connection pooling
transaction protocol
worker restart
```

Define las reglas que esos sistemas deberán respetar.

---

# 10. Clasificación de estado

Todo estado de Database deberá clasificarse.

Modelo inicial:

```php
enum DatabaseStateLifetime
{
    case APPLICATION;
    case WORKER;
    case EXECUTION;
    case OPERATION;
    case TRANSACTION;
    case RESOURCE;
}
```

---

# 11. Application State

Estado que puede vivir durante toda la aplicación.

Ejemplos:

```text
immutable configuration definitions
compiled metadata
type definitions
dialect definitions
optimization rules
compiler registries
```

---

# 12. Application state requirement

Deberá ser preferentemente:

```text
immutable
```

o internamente concurrency-safe.

---

# 13. Worker State

Puede vivir mientras exista el worker.

Ejemplos:

```text
connection pool
bounded compiled-query cache
metadata cache
runtime adapter
worker telemetry aggregator
```

---

# 14. Worker State ≠ Request State

Nunca deberá almacenar directamente:

```text
current tenant
current user
current EntityManager
current transaction
current query
current connection lease
```

---

# 15. Execution State

Pertenece a:

```text
HTTP request
queue job
CLI execution
test execution
background task
```

Ejemplos:

```text
DatabaseContext
EntityManager
IdentityMap
UnitOfWork
routing state
tenant binding
security context
```

---

# 16. Operation State

Pertenece a una operación específica.

Ejemplos:

```text
query execution
bulk operation
chunk traversal
import
export
lazy iteration
```

---

# 17. Transaction State

Pertenece a una transacción concreta.

Ejemplos:

```text
TransactionId
savepoint stack
isolation level
transaction connection binding
retry attempt
outcome state
```

---

# 18. Resource State

Pertenece a un recurso específico.

Ejemplos:

```text
ResultCursor
StreamingResult
ConnectionLease
PreparedStatementHandle
ResourcePermit
```

---

# 19. Clasificación ortogonal

Además del lifetime, se clasificará por mutabilidad:

```php
enum DatabaseStateMutability
{
    case IMMUTABLE;
    case MUTABLE_EXCLUSIVE;
    case MUTABLE_SYNCHRONIZED;
    case MUTABLE_CONTEXT_LOCAL;
}
```

---

# 20. State descriptor

Conceptualmente:

```php
final readonly class DatabaseStateDescriptor
{
    public function __construct(
        public DatabaseStateLifetime $lifetime,
        public DatabaseStateMutability $mutability,
        public DatabaseStateSharing $sharing,
    ) {}
}
```

---

# 21. Sharing classification

```php
enum DatabaseStateSharing
{
    case NOT_SHARED;
    case SHARED_READ_ONLY;
    case SHARED_SYNCHRONIZED;
    case SHARED_BY_PARTITION;
}
```

---

# 22. Unknown sharing

Regla:

```text
UNKNOWN
≠
SAFE_TO_SHARE
```

---

# 23. State ownership

Todo estado mutable deberá tener propietario.

Formalmente:

```text
MutableState
→
Owner
```

---

# 24. Owner types

El owner podrá ser:

```text
Application
Worker
ExecutionScope
Operation
Transaction
Resource
```

---

# 25. Ownership hierarchy

```text
Application
   │
   └── Worker
       │
       └── Execution
           │
           ├── Operation
           │
           └── Transaction
           │
           └── Resource
```

---

# 26. Ownership ≠ accessibility

Que un objeto sea accesible desde un componente no significa que ese componente sea su owner.

---

# 27. Execution isolation boundary

La frontera principal será:

```text
Execution A
───────────────
Execution B
```

Ningún estado mutable execution-local podrá cruzarla.

---

# 28. Isolation Domains

Se propone:

```text
DatabaseIsolationDomain
```

para representar una frontera concreta.

---

# 29. IsolationDomainId

```php
final readonly class IsolationDomainId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 30. Execution scope as isolation domain

Normalmente:

```text
DatabaseExecutionScope
=
root mutable isolation domain
```

---

# 31. Nested isolation domains

Podrán existir:

```text
Execution Isolation Domain
│
├── Transaction Isolation Domain
├── Operation Isolation Domain
└── Resource Isolation Domain
```

---

# 32. Isolation token

Los recursos críticos podrán asociarse a:

```text
IsolationToken
```

---

# 33. IsolationToken

Conceptualmente:

```php
final readonly class IsolationToken
{
    public function __construct(
        public IsolationDomainId $domain,
        public DatabaseScopeId $scope,
        public ScopeGeneration $generation,
    ) {}
}
```

---

# 34. Ownership validation

Antes de utilizar un recurso contextual:

```text
CurrentIsolationToken
        ↓
compatibility check
        ↓
ResourceIsolationToken
```

---

# 35. Identity vs compatibility

No todos los recursos requieren igualdad exacta.

Por ello:

```text
SameOwner
```

y:

```text
CompatibleOwner
```

serán conceptos distintos.

---

# 36. EntityManager

`EntityManager` será:

```text
EXECUTION
+
MUTABLE_EXCLUSIVE
+
NOT_SHARED
```

por defecto.

---

# 37. EntityManager isolation

```text
Request A
└── EntityManager A

Request B
└── EntityManager B
```

Nunca:

```text
Worker
└── EntityManager
    ├── Request A
    └── Request B
```

---

# 38. Why EntityManager cannot be worker-global

Porque contiene o coordina:

```text
IdentityMap
UnitOfWork
managed entities
pending changes
lifecycle state
```

---

# 39. IdentityMap

Clasificación:

```text
EXECUTION
+
MUTABLE_EXCLUSIVE
+
NOT_SHARED
```

---

# 40. IdentityMap contamination example

Request A:

```text
User#10
email = old@example.com
```

Request B consulta `User#10`.

Si IdentityMap fuera compartida:

```text
B could receive A's object
```

sin nueva lectura ni contexto correcto.

Esto está prohibido.

---

# 41. Tenant-aware identity is not enough

Aunque IdentityMap key contenga TenantId:

```text
tenant + entity + id
```

seguiría siendo incorrecto compartir una IdentityMap mutable entre executions.

---

# 42. UnitOfWork

Clasificación:

```text
EXECUTION
+
MUTABLE_EXCLUSIVE
+
NOT_SHARED
```

---

# 43. UoW contamination

Nunca:

```text
Request A schedules User#10 UPDATE
Request B flushes
```

---

# 44. Entity State

Estados:

```text
NEW
MANAGED
DIRTY
REMOVED
DETACHED
```

son relativos a un EntityManager/UoW específico.

---

# 45. MANAGED is contextual

Regla:

```text
Entity managed by EM-A
≠
Entity managed by EM-B
```

---

# 46. Cross-EntityManager entity use

Una entidad managed por A no deberá introducirse silenciosamente en B.

Podrá requerirse:

```text
detach
merge/reload strategy
explicit reattachment
identifier reference
```

según diseño final.

---

# 47. TransactionContext

Clasificación:

```text
TRANSACTION
+
MUTABLE_EXCLUSIVE
+
NOT_SHARED
```

---

# 48. Transaction state

Incluye:

```text
ACTIVE
COMMITTING
COMMITTED
ROLLING_BACK
ROLLED_BACK
UNKNOWN
```

---

# 49. Transaction isolation from executions

Transaction A pertenece a un execution concreto.

No puede aparecer en otro request porque comparten worker.

---

# 50. Transaction connection binding

Si:

```text
Transaction T
→
ConnectionLease L
```

entonces ambos deberán ser compatibles con el mismo isolation domain.

---

# 51. Savepoint stack

Será transaction-local.

Nunca worker-global.

---

# 52. Retry attempt state

Cada intento tendrá estado independiente:

```text
Transaction
├── Attempt 1
├── Attempt 2
└── Attempt 3
```

---

# 53. Attempt state leakage

Datos como:

```text
last exception
retry delay
deadlock evidence
```

no deberán contaminar otra operación.

---

# 54. DatabaseContext

Clasificación principal:

```text
EXECUTION
+
mostly IMMUTABLE
+
NOT_SHARED across executions
```

---

# 55. Context immutable sharing

Subcomponentes realmente inmutables pueden ser structural-shared.

Ejemplo:

```text
RuntimeCapabilities
```

pero no convierte al DatabaseContext vivo en worker-global.

---

# 56. Tenant state

Tenant binding será:

```text
EXECUTION
```

o una dimensión más estrecha en operaciones explícitas.

---

# 57. Tenant isolation invariant

```text
Tenant(A)
```

no deberá influir:

```text
Tenant(B)
```

---

# 58. Shard state

Igual:

```text
Shard(A)
```

no se hereda a B.

---

# 59. Routing state

Routing mutable incluye:

```text
sticky writer
selected endpoint
minimum replication position
authority epoch
read affinity
```

---

# 60. Routing lifetime

Normalmente:

```text
EXECUTION
```

con ciertos valores:

```text
TRANSACTION
```

u:

```text
OPERATION
```

---

# 61. Sticky connection leakage

Caso prohibido:

```text
Request A performs write
    ↓
sticky writer = true

Request B
    ↓
inherits sticky writer
```

---

# 62. Replica affinity

Si A fue enviado a replica-2, B no deberá quedar accidentalmente fijado a replica-2.

---

# 63. Connection architecture

Debe distinguirse:

```text
Physical Connection
Connection Wrapper
Connection Lease
Connection Session State
```

---

# 64. Physical connection sharing

Una conexión física puede reutilizarse secuencialmente entre executions mediante pool.

Esto no significa que el estado lógico sea compartido.

---

# 65. Connection Lease

Clasificación:

```text
EXECUTION/OPERATION/TRANSACTION
+
MUTABLE_EXCLUSIVE
+
NOT_SHARED
```

dependiendo del uso.

---

# 66. Physical connection

Puede ser:

```text
WORKER
+
MUTABLE_EXCLUSIVE
+
SHARED_BY_REUSE
```

es decir:

```text
shared sequentially
```

no concurrentemente salvo capability explícita.

---

# 67. Connection reuse boundary

```text
Scope A
   ↓
Lease A
   ↓
Physical Connection
   ↓
RESET
   ↓
Lease B
   ↓
Scope B
```

---

# 68. Reset required for reuse

Si el physical connection mantiene session state, deberá resetearse antes de entregarse a otro isolation domain.

---

# 69. Session state examples

```text
active transaction
session variables
temporary tables
search path
SQL mode
timezone
role
isolation setting
prepared statements
advisory locks
```

---

# 70. Connection state uncertainty

Si VoltStack no puede demostrar que el estado sea limpio:

```text
UNKNOWN
```

la conexión no deberá regresar como reusable.

---

# 71. UNKNOWN ≠ CLEAN

Regla:

```text
UNKNOWN connection state
≠
safe connection
```

---

# 72. Connection quarantine

Una conexión dudosa podrá marcarse:

```text
QUARANTINED
```

---

# 73. Connection discard

Si no puede limpiarse:

```text
discard physical connection
```

---

# 74. Prepared statements

Prepared statement state deberá clasificarse cuidadosamente.

---

# 75. Compiled statement ≠ live prepared handle

```text
CompiledQuery
```

puede ser immutable/shared.

Pero:

```text
DriverPreparedStatement
```

puede estar ligado a una conexión concreta.

---

# 76. Statement ownership

Un prepared statement handle deberá pertenecer al connection/resource domain correspondiente.

---

# 77. ResultCursor

Clasificación:

```text
RESOURCE
+
MUTABLE_EXCLUSIVE
+
NOT_SHARED
```

---

# 78. Cursor cannot cross scope

Un cursor abierto en A no podrá consumirse desde B.

---

# 79. StreamingResult

Misma regla.

```text
stream.owner = Scope A
```

---

# 80. LazyCollection

Una Lazy Collection tiene dos capas:

```text
Lazy Definition
Lazy Active Iteration
```

---

# 81. Lazy Definition

Si es completamente inmutable puede ser reusable bajo condiciones explícitas.

---

# 82. Active Lazy Iteration

Será:

```text
OPERATION/RESOURCE
+
MUTABLE_EXCLUSIVE
+
NOT_SHARED
```

---

# 83. Lazy context binding

La iteración deberá validar que el contexto de ejecución sea compatible con el contexto capturado.

---

# 84. Generator isolation

Un `Generator` activo no será compartido entre executions.

---

# 85. Chunk processing

El traversal state será:

```text
OPERATION
```

---

# 86. Chunk state

Ejemplos:

```text
current boundary
chunk number
rows processed
checkpoint candidate
horizon
retry state
```

---

# 87. No static last ID

Prohibido:

```php
private static int $lastProcessedId;
```

---

# 88. Import state

Será operation-local.

Incluye:

```text
row number
batch number
mapping state
error collection
checkpoint
```

---

# 89. Export state

Igualmente:

```text
current chunk
stream position
rows emitted
output resource
```

---

# 90. Pagination state

Page/cursor request será operation-local.

No existirá:

```text
static current page
```

---

# 91. Query execution state

Cada query tendrá:

```text
QueryExecutionId
```

y estado independiente.

---

# 92. Query parameters

Parameter bindings no deberán almacenarse en compiler shared state.

---

# 93. Compiler state

Compiler deberá ser:

```text
immutable/stateless
```

o utilizar:

```text
operation-local compilation context
```

---

# 94. Query Planner

Misma regla.

Un planner shared no retendrá:

```text
last query
last tenant
last parameters
last plan
```

como mutable request state.

---

# 95. Optimizer

Optimization rules podrán ser shared si son inmutables.

Optimization execution state será local.

---

# 96. Hydration

Hydration session state será operation-local.

---

# 97. Hydration state examples

```text
current row
assembly registry
join deduplication
partial coverage
reservation state
```

---

# 98. IdentityMap remains execution-scoped

Hydration session puede utilizar IdentityMap del execution.

Pero no sustituirla.

---

# 99. Relationship loading

Batch loader state será execution/operation scoped.

---

# 100. N+1 detection

El detector tendrá:

```text
shared detector logic
+
execution-local observations
```

---

# 101. N+1 state leakage

Las queries observadas en Request A no deberán hacer que Request B aparezca como N+1.

---

# 102. Query profiler

Mismo patrón:

```text
Profiler Engine
=
shared

Profile Session
=
execution-local
```

---

# 103. Slow query detector

Threshold configuration puede ser shared.

Observaciones de una ejecución deberán asociarse a su scope.

---

# 104. Telemetry state

Tracing context será execution/operation-local.

Exporter puede ser shared.

---

# 105. Event system

Event Dispatcher puede ser shared.

Pero:

```text
current event stack
listener execution state
deferred transaction events
```

deberán tener lifetime correcto.

---

# 106. Transaction events

Eventos:

```text
afterCommit
afterRollback
```

deberán permanecer ligados a la transaction correspondiente.

---

# 107. Deferred events

Nunca:

```text
Transaction A deferred event
    ↓
Transaction B commits
    ↓
event A dispatched
```

---

# 108. Persistence events

Event execution context deberá contener ownership correcto.

---

# 109. Audit state

Audit correlation será execution/operation scoped.

Audit sink puede ser shared.

---

# 110. Security state

Datos como:

```text
actor
authorization scope
data visibility policy
privilege elevation
```

serán execution-local salvo descriptor inmutable explícito.

---

# 111. Security isolation

Este aislamiento será obligatorio incluso cuando dos requests pertenezcan al mismo usuario.

---

# 112. Same user ≠ same execution

```text
User X / Request A
≠
User X / Request B
```

en términos de mutable Database state.

---

# 113. Resource budgets

Resource accounting deberá ser scoped.

---

# 114. Budget leakage

Si A consume:

```text
80% query budget
```

B no deberá heredar ese consumo salvo que exista deliberadamente un budget superior compartido.

---

# 115. Hierarchical budgets

Podrá existir:

```text
Worker Budget
   ↓
Execution Budget
      ↓
Operation Budget
```

---

# 116. Budget sharing explicit

Un worker-wide capacity limit es válido.

Pero deberá distinguirse de:

```text
request consumption
```

---

# 117. Cancellation

Cancellation state será execution/operation scoped.

---

# 118. Cancellation leakage

Request A cancelado no deberá cancelar Request B.

---

# 119. Parent-child cancellation

Sólo dentro del árbol correspondiente:

```text
Execution A
├── Operation A1
└── Operation A2
```

---

# 120. Deadlines

Deadlines serán scoped.

No utilizar:

```text
static current deadline
```

---

# 121. Runtime-local storage

Para resolver current scope/context se requerirá almacenamiento compatible con el modelo de ejecución.

---

# 122. RuntimeLocalStorage

Contrato conceptual:

```php
interface RuntimeLocalStorage
{
    public function get(RuntimeLocalKey $key): mixed;

    public function set(RuntimeLocalKey $key, mixed $value): void;

    public function remove(RuntimeLocalKey $key): void;
}
```

---

# 123. Runtime local ≠ process global

Implementación incorrecta:

```php
private static array $values = [];
```

si existen executions concurrentes.

---

# 124. Traditional PHP

Puede utilizar una implementación simple porque típicamente existe una ejecución por proceso/request.

Aun así deberá respetar el contrato.

---

# 125. FrankenPHP

Debe contemplar worker persistente y posible concurrencia según configuración/runtime.

---

# 126. RoadRunner

Debe aislar cada job/request procesado por worker.

---

# 127. OpenSwoole

Requiere aislamiento coroutine-aware.

---

# 128. Fiber-aware storage

Si VoltStack utiliza Fibers:

```text
Fiber A
→ context A

Fiber B
→ context B
```

cuando representan ejecuciones distintas.

---

# 129. Child Fiber

Una Fiber hija podrá heredar contexto sólo mediante política explícita.

---

# 130. Context propagation ≠ state sharing

Heredar valores inmutables:

```text
tenant id
trace id
```

no significa compartir objetos mutables como:

```text
EntityManager
ResultCursor
```

---

# 131. Coroutine-local storage

OpenSwoole adapter deberá utilizar una abstracción compatible con coroutine-local context.

El core no deberá depender directamente de APIs de Swoole.

---

# 132. Runtime adapter responsibility

```text
Core
    ↓
RuntimeLocalStorage Contract
    ↓
Runtime Adapter
    ├── FrankenPHP
    ├── RoadRunner
    └── OpenSwoole
```

---

# 133. Static state policy

VoltStack Database deberá establecer una política estricta para `static`.

---

# 134. Allowed static state

Podrá permitirse para:

```text
immutable constants
pure stateless singleton references
compiled immutable metadata
interned immutable values
```

si son realmente seguros.

---

# 135. Forbidden static state

Prohibido para:

```text
current tenant
current user
current shard
current transaction
current EntityManager
current UnitOfWork
current IdentityMap
current query
current parameters
current connection
current cursor
current page
last inserted entity
```

---

# 136. Static cache caveat

Un cache estático sólo será permitido si:

```text
bounded
correctly keyed
generation-aware
tenant-safe
concurrency-safe
resettable when required
```

y pertenece realmente al lifetime application/worker.

---

# 137. Static analysis

VoltStack podrá incorporar reglas de análisis estático para detectar:

```text
mutable static properties
```

en namespaces sensibles.

---

# 138. Example rule

Dentro de:

```text
VoltStack\Quantum\Database\ORM
```

una propiedad:

```php
private static ?EntityManager $manager;
```

deberá generar error arquitectónico.

---

# 139. Isolation annotations

Podrán definirse atributos:

```php
#[ExecutionScoped]
final class EntityManager
{
}
```

```php
#[WorkerSafe]
final readonly class TypeRegistry
{
}
```

---

# 140. Attributes ≠ correctness

Los atributos documentan y permiten validación.

No hacen seguro el código automáticamente.

---

# 141. Container validation

El Container podrá validar:

```text
lifetime dependency graph
```

---

# 142. Captive dependency

Caso:

```text
WorkerSingleton
    ↓
ExecutionScoped EntityManager
```

será inválido.

---

# 143. Dependency direction

Permitido:

```text
ExecutionScoped Service
    ↓
WorkerSafe Service
```

No necesariamente:

```text
WorkerSafe Service
    ↓
ExecutionScoped instance
```

---

# 144. Resolver exception

Un worker service podrá depender de:

```text
EntityManagerResolver
```

si el resolver no captura una instancia.

---

# 145. Closure capture

También deberá vigilarse:

```php
$entityManager = $resolver->current();

$workerCallback = function () use ($entityManager) {
    // dangerous
};
```

---

# 146. Long-lived closure policy

Closures almacenadas en worker services no deberán capturar scoped mutable state.

---

# 147. Event listener risk

Un listener singleton:

```php
final class QueryListener
{
    private ?QueryEvent $lastEvent = null;
}
```

puede convertirse en fuga.

---

# 148. Listener design

Preferir:

```text
stateless listener
```

o estado correctamente scoped.

---

# 149. Isolation Guard

Se propone:

```text
DatabaseIsolationGuard
```

---

# 150. Guard responsibilities

Podrá verificar:

```text
scope ownership
generation
resource owner
transaction affinity
tenant compatibility
runtime-local context
```

---

# 151. Production checks

Algunas verificaciones críticas deberán permanecer activas en producción.

Ejemplo:

```text
cross-scope connection lease use
```

---

# 152. Debug checks

Otras verificaciones más costosas podrán habilitarse en development/testing.

---

# 153. Isolation violation

Jerarquía:

```text
DatabaseStateIsolationException
├── CrossScopeStateAccessException
├── CrossOperationStateAccessException
├── CrossTransactionStateAccessException
├── StaleScopedStateException
├── InvalidStateOwnerException
├── UnsafeSharedStateException
├── ConcurrentResourceAccessException
├── ScopedStateLeakException
└── RuntimeLocalIsolationException
```

---

# 154. Violation handling

Una violación crítica podrá:

```text
fail operation
taint scope
quarantine connection
request worker recycle
```

dependiendo del recurso.

---

# 155. Cross-tenant violation

Deberá considerarse de alta severidad.

---

# 156. Cross-transaction connection use

También será crítico.

---

# 157. Isolation status

Podrá utilizarse:

```php
enum IsolationStatus
{
    case CLEAN;
    case SUSPECT;
    case VIOLATED;
    case UNKNOWN;
}
```

---

# 158. UNKNOWN ≠ CLEAN

Nuevamente:

```text
UNKNOWN
≠
CLEAN
```

---

# 159. Isolation evidence

La arquitectura deberá basar decisiones en evidencia.

Ejemplo:

```text
connection session reset succeeded
```

es evidencia.

Simplemente asumir:

```text
probably clean
```

no lo es.

---

# 160. Worker taint

El worker podrá adquirir:

```text
WorkerIsolationState::TAINTED
```

si ocurre una corrupción que no puede limitarse a un scope.

---

# 161. Worker recycle

En ese caso:

```text
finish current safe cleanup
    ↓
stop accepting new work
    ↓
recycle worker
```

---

# 162. Worker recycle ≠ request failure

Puede existir:

```text
request completed
+
worker must recycle
```

---

# 163. Scope taint ≠ worker taint

Un scope puede quedar tainted mientras el worker siga siendo reusable si el daño fue completamente contenido.

---

# 164. Containment

El sistema deberá determinar:

```text
CanFailureBeContained?
```

---

# 165. Containment examples

Cursor leak cerrado exitosamente:

```text
contained
```

Unknown physical connection state:

```text
contained if connection discarded
```

Unknown process-global mutation:

```text
possibly not contained
```

---

# 166. Isolation barriers

VoltStack utilizará varias barreras:

```text
Scope Barrier
Context Barrier
Container Lifetime Barrier
Resource Ownership Barrier
Runtime-Local Barrier
Transaction Barrier
Connection Lease Barrier
Generation Barrier
Security Barrier
```

---

# 167. Defense in depth

No depender de una sola barrera.

---

# 168. Example

Para impedir que una connection lease cruce scopes:

```text
Scoped Container
+
Lease Owner Token
+
Scope Generation
+
Finalization
+
Pool Reset
```

---

# 169. Generation barriers

Cada execution podrá tener:

```text
ScopeGeneration
```

---

# 170. Why generation matters

Si IDs fueran reutilizables accidentalmente:

```text
scope-id = 42
```

la generation evita aceptar una referencia antigua.

---

# 171. ABA-style problem

Ejemplo conceptual:

```text
Scope A token = (42, generation 7)
Scope B token = (42, generation 8)
```

Una referencia de A no será válida en B.

---

# 172. Object poisoning

Al cerrar un recurso scoped podrá marcarse:

```text
CLOSED
```

para que referencias antiguas fallen.

---

# 173. Closed EntityManager

No deberá reactivarse.

---

# 174. Closed Cursor

No deberá volver a leer.

---

# 175. Released Lease

No deberá ejecutar.

---

# 176. Stale context

No deberá derivar nuevas operaciones.

---

# 177. Isolation and caching

Caches compartidos son un área especial.

---

# 178. Cache state classes

```text
L0 execution-local
L1 worker-local
L2 distributed/shared
```

---

# 179. L0 isolation

Se elimina con execution.

---

# 180. L1 isolation

Debe utilizar keys completas y no almacenar live scoped objects.

---

# 181. Entity cache

No almacenará:

```text
managed entity object
```

para reutilizarlo entre scopes.

---

# 182. Result cache

No almacenará:

```text
ResultCursor
ConnectionLease
StreamingResult
```

---

# 183. Metadata cache

Puede compartirse porque almacena información estructural, no estado ORM vivo.

---

# 184. Hydration cache

Sólo podrá compartir:

```text
plans
accessors
compiled hydration metadata
```

no entidades hidratadas managed.

---

# 185. Query cache

Query model/plan cache deberá excluir parámetros o contexto mutable cuando no formen parte de la key correcta.

---

# 186. Tenant-aware shared caches

Si un cache contiene datos tenant-specific:

```text
TenantId
```

deberá formar parte de la identidad.

---

# 187. Security-aware caches

Igualmente para:

```text
authorization/data visibility scope
```

cuando cambie el resultado observable.

---

# 188. Isolation and Events

Un evento deberá incluir:

```text
ScopeId
OperationId
TransactionId
```

cuando corresponda.

---

# 189. Event payload ownership

No todos los eventos deberán transportar live mutable objects.

---

# 190. Async events

Si un evento cruza execution boundary:

```text
serialize descriptor/value data
```

no:

```text
serialize EntityManager
```

---

# 191. Isolation and telemetry

Telemetry deberá ayudar a detectar contaminación.

---

# 192. Correlation fields

Logs/traces podrán incluir:

```text
database.scope_id
database.operation_id
database.transaction_id
database.connection_lease_id
database.worker_id
```

con política de cardinalidad adecuada.

---

# 193. Isolation telemetry events

```text
database.isolation.violation
database.isolation.stale_state
database.isolation.cross_scope_access
database.isolation.concurrent_resource_access
database.isolation.worker_tainted
database.isolation.resource_quarantined
```

---

# 194. Metrics

Ejemplos:

```text
database_isolation_violations_total
database_stale_state_access_total
database_resource_quarantines_total
database_worker_taints_total
```

---

# 195. No PII

Telemetry de aislamiento no deberá exponer datos sensibles innecesarios.

---

# 196. Isolation diagnostics

Developer Toolbar podrá mostrar:

```text
Database Isolation
──────────────────
Scope: dbscope_01...
Generation: 291
Worker: worker-04

EntityManager:
  owner: current scope
  state: OPEN

Transaction:
  tx-91
  owner: current scope

Connection Lease:
  lease-18
  owner: current scope

Isolation violations:
  0
```

---

# 197. Debug ownership graph

En development:

```text
Scope A
├── EntityManager A
│   ├── IdentityMap A
│   └── UnitOfWork A
├── Transaction T1
│   └── Lease L1
└── Cursor C1
    └── Lease L1
```

---

# 198. Orphan detection

Un recurso activo sin owner válido:

```text
ORPHANED
```

será una anomalía.

---

# 199. Orphan resource

Ejemplo:

```text
Cursor active
Scope closed
```

---

# 200. Leak detector

`DatabaseStateLeakDetector` podrá comprobar:

```text
active resources after finalization
stale scoped services
unexpected strong references
open transactions
unreleased permits
```

---

# 201. Leak detection ≠ isolation

Una fuga puede existir sin contaminación todavía.

Pero aumenta el riesgo.

---

# 202. Leak severity

Clasificación:

```text
INFO
WARNING
ERROR
CRITICAL
```

---

# 203. Critical leak

Ejemplo:

```text
active transaction survived scope finalization
```

---

# 204. Isolation verification

Se propone:

```text
DatabaseIsolationVerifier
```

---

# 205. Verification stages

```text
AT_SCOPE_START
DURING_OPERATION
AT_SCOPE_FINALIZATION
BEFORE_RESOURCE_REUSE
```

---

# 206. Start verification

Puede comprobar:

```text
no stale runtime-local context
no inherited EntityManager
no inherited transaction
```

---

# 207. During-operation verification

Puede comprobar ownership de recursos críticos.

---

# 208. Finalization verification

Puede comprobar:

```text
no active cursors
no active transaction
no scoped connection leases
no pending async DB operation
```

---

# 209. Before reuse verification

Especialmente para:

```text
physical connection
worker
```

---

# 210. Isolation proof levels

Podrá utilizarse:

```php
enum IsolationEvidenceLevel
{
    case VERIFIED;
    case VERIFIED_WITH_WARNINGS;
    case PARTIAL;
    case UNKNOWN;
    case VIOLATED;
}
```

---

# 211. Strict mode

En testing:

```text
PARTIAL
UNKNOWN
```

podrán tratarse como failure para ciertos recursos.

---

# 212. Production mode

Podrá usar políticas conservadoras:

```text
unknown connection state
→ discard
```

---

# 213. Isolation Policy

Se propone:

```text
DatabaseIsolationPolicy
```

---

# 214. Policy example

```php
DatabaseIsolationPolicy::strict()
    ->crossScopeAccess(IsolationAction::THROW)
    ->staleState(IsolationAction::THROW)
    ->unknownConnectionState(IsolationAction::DISCARD_RESOURCE)
    ->workerGlobalLeak(IsolationAction::RECYCLE_WORKER);
```

---

# 215. Security cannot be disabled

Las garantías críticas de tenant/security isolation no deberán convertirse en simples debug options.

---

# 216. Testing strategy

El sistema requerirá pruebas específicas.

---

# 217. Sequential request test

```text
Request A
→ tenant A
→ close

Request B
→ tenant B
```

Verificar:

```text
B contains no A state
```

---

# 218. IdentityMap isolation test

```php
$userA = $scopeA->run(fn () => User::find(10));
$userB = $scopeB->run(fn () => User::find(10));

assert($userA !== $userB);
```

---

# 219. UoW isolation test

Modificar entidad en A sin flush.

Crear B.

Verificar:

```text
B UoW pending changes = 0
```

---

# 220. Transaction isolation test

Abrir transaction en A.

B no deberá observarla como current transaction.

---

# 221. Tenant isolation test

A:

```text
tenant = acme
```

B:

```text
tenant = contoso
```

Verificar ausencia total de contaminación.

---

# 222. Sticky routing test

A escribe.

B comienza.

Verificar:

```text
sticky(A) does not imply sticky(B)
```

---

# 223. Cancellation test

Cancelar A.

B continúa normalmente.

---

# 224. Deadline test

Deadline expirado de A no aparece en B.

---

# 225. Profiler test

Queries de A no aparecen en profile B.

---

# 226. N+1 detector test

Patrones observados en A no cuentan como accesses de B.

---

# 227. Cursor test

Cursor A utilizado desde B:

```text
CrossScopeStateAccessException
```

---

# 228. Lease test

Lease A usado en B:

```text
CrossScopeStateAccessException
```

---

# 229. Released lease test

Lease A después de finalization:

```text
StaleScopedStateException
```

---

# 230. Coroutine isolation test

Ejecutar:

```text
A1
B1
A2
B2
```

intercaladamente.

Verificar:

```text
current(A) always A
current(B) always B
```

---

# 231. Fiber isolation test

Mismo patrón para Fibers.

---

# 232. Long worker soak test

Ejecutar:

```text
100,000+
```

scopes secuenciales.

Medir:

```text
memory growth
stale references
resource count
isolation violations
```

---

# 233. Randomized isolation test

Intercalar aleatoriamente:

```text
query
transaction
cursor
tenant
cancel
retry
failover
```

entre múltiples scopes.

---

# 234. Fault injection

Simular:

```text
rollback failure
connection loss
cursor close failure
timeout
unknown commit
runtime interruption
```

y verificar containment.

---

# 235. Cross-tenant adversarial test

Intentar reutilizar deliberadamente:

```text
context
lease
entity
cursor
checkpoint
```

entre tenants.

---

# 236. Static-state test

Analizar clases Database buscando mutable static state prohibido.

---

# 237. Container lifetime test

Intentar inyectar:

```text
EntityManager
```

en singleton worker.

El container deberá rechazarlo cuando pueda detectarlo.

---

# 238. Closure capture test

Tooling podrá detectar ciertos casos de closures persistentes capturando scoped services.

---

# 239. Connection reuse isolation test

```text
Scope A
→ session state changed
→ release

reset

Scope B
→ acquire same physical connection
```

Verificar que B no observe estado de A.

---

# 240. Transaction residue test

Después de A:

```text
no open transaction
```

antes de entregar la conexión a B.

---

# 241. Temporary table test

Si el DBMS/session puede mantener tablas temporales:

```text
A creates temp state
```

la política de reset/reuse deberá impedir que B reciba contaminación.

---

# 242. Session role test

Si A cambia:

```text
SET ROLE
```

B no deberá heredar el role.

---

# 243. Session timezone test

Igualmente para timezone cuando afecte semántica.

---

# 244. Session variable test

Variables custom no deberán filtrarse entre leases.

---

# 245. Advisory lock test

Locks session-scoped deberán liberarse o causar descarte de conexión según plataforma/política.

---

# 246. Platform-specific isolation

El modelo será común.

La implementación de limpieza podrá variar por:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 247. Platform capabilities

Se usarán capabilities como:

```text
supportsSessionReset()
supportsDiscardAll()
supportsSessionRoleReset()
supportsAdvisoryLockInspection()
```

conceptualmente.

---

# 248. Platform-specific SQL location

El State Isolation System no generará SQL vendor-specific directamente.

---

# 249. Connection reset delegation

```text
State Isolation
      ↓
Connection State Reset Plan
      ↓
Platform/Dialect Capability
      ↓
Executor/Driver
```

---

# 250. SQLite

SQLite tendrá particularidades distintas:

```text
in-memory database lifetime
connection-bound database
temporary objects
transaction state
```

---

# 251. In-memory SQLite

Una conexión `:memory:` puede representar la propia base.

Por tanto:

```text
discard connection
```

puede significar:

```text
discard database
```

La política deberá conocer esta diferencia.

---

# 252. Testing SQLite caveat

No se asumirá que el comportamiento de aislamiento de SQLite representa automáticamente MySQL/PostgreSQL.

---

# 253. Isolation and parallel queries

Dos operaciones del mismo execution podrían ejecutarse concurrentemente.

Esto introduce otra frontera.

---

# 254. Same scope ≠ same mutable resource

Regla:

> Dos operaciones pertenecientes al mismo scope no adquieren automáticamente derecho a usar concurrentemente el mismo recurso mutable.

---

# 255. Connection concurrency

Una conexión podrá declarar:

```text
EXCLUSIVE
SERIALIZED
CONCURRENT_SAFE
```

---

# 256. Default

Cuando no exista evidencia:

```text
UNKNOWN
→
EXCLUSIVE
```

como política conservadora.

---

# 257. EntityManager concurrency

Por defecto:

```text
EntityManager
=
NOT CONCURRENT SAFE
```

---

# 258. Concurrent ORM operations

Podrán requerir:

```text
separate operation context
separate EntityManager
```

si se soportan explícitamente.

---

# 259. UnitOfWork concurrency

No se permitirá mutación concurrente no coordinada.

---

# 260. IdentityMap concurrency

Igualmente.

---

# 261. Isolation within request

Por tanto existen dos problemas:

```text
cross-execution isolation
```

y:

```text
intra-execution concurrency isolation
```

---

# 262. Operation isolation

Para operaciones concurrentes:

```text
Execution
├── Operation A
└── Operation B
```

podrán requerirse recursos independientes.

---

# 263. Transaction concurrency

Una misma transaction física no deberá usarse concurrentemente salvo soporte explícito demostrado.

---

# 264. Resource borrowing

No se transferirá ownership mediante simple referencia.

Si se permite transferencia:

```text
handoff protocol
```

deberá ser explícito.

---

# 265. Handoff state

Un recurso transferible podría usar:

```text
OWNED
TRANSFERRING
OWNED_BY_NEW_OWNER
```

pero inicialmente VoltStack deberá preferir recursos no transferibles.

---

# 266. Simplicity principle

Para V1:

```text
scoped mutable resources
=
non-transferable
```

será el default más seguro.

---

# 267. Isolation and object references

PHP permite conservar referencias fácilmente.

Por ello cerrar scope no elimina mágicamente objetos referenciados por código de aplicación.

---

# 268. Poison-after-close

Los objetos scoped importantes deberán conocer su estado:

```text
OPEN
CLOSED
TAINTED
```

---

# 269. Entity object caveat

Una entidad POPO puede sobrevivir al scope.

Pero pasará conceptualmente a:

```text
DETACHED
```

cuando su EntityManager cierre.

---

# 270. Detached entity

Podrá leerse como objeto de dominio según diseño.

Pero no deberá:

```text
lazy load
flush itself through old EM
use old transaction
```

---

# 271. Lazy proxy caveat

Un proxy/lazy relation sobreviviente al scope no podrá recuperar mágicamente un nuevo EntityManager global.

---

# 272. Detached lazy access

Resultado:

```text
DetachedLazyLoadingException
```

o equivalente.

---

# 273. Serialization

Serializar una entidad no deberá disparar lazy loading para recuperar contexto perdido.

---

# 274. Scope-local service escape

Tooling podrá detectar cuando un servicio scoped se almacena en una propiedad worker-long-lived.

---

# 275. Escape analysis

No será perfecta en runtime dinámico, pero podrá combinar:

```text
container validation
attributes
debug ownership tracking
static analysis
tests
```

---

# 276. Isolation Architecture

```text
                    APPLICATION
                         │
              Immutable Shared State
                         │
                         ▼
                       WORKER
                         │
            ┌────────────┴────────────┐
            │                         │
      Worker-safe Cache          Connection Pool
            │                         │
            └────────────┬────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
   EXECUTION A       EXECUTION B      EXECUTION C
        │                │                │
        ▼                ▼                ▼
   Context A         Context B         Context C
   ORM A             ORM B             ORM C
   UoW A             UoW B             UoW C
   IdentityMap A     IdentityMap B     IdentityMap C
        │                │                │
        ▼                ▼                ▼
  Operation A       Operation B       Operation C
        │                │                │
        ▼                ▼                ▼
     Lease A           Lease B           Lease C
        │                │                │
        └─────────────── Pool ────────────┘
```

La infraestructura puede compartirse.

El estado contextual no.

---

# 277. Directory Structure

Propuesta:

```text
src/Quantum/Database/Runtime/Isolation/
│
├── Contract/
│   ├── DatabaseIsolationGuard.php
│   ├── DatabaseIsolationVerifier.php
│   ├── RuntimeLocalStorage.php
│   ├── IsolationPolicy.php
│   └── IsolationAwareResource.php
│
├── Model/
│   ├── IsolationDomainId.php
│   ├── IsolationToken.php
│   ├── IsolationStatus.php
│   ├── IsolationEvidenceLevel.php
│   ├── DatabaseStateLifetime.php
│   ├── DatabaseStateMutability.php
│   ├── DatabaseStateSharing.php
│   └── DatabaseStateDescriptor.php
│
├── Ownership/
│   ├── StateOwner.php
│   ├── ResourceOwnership.php
│   ├── OwnershipValidator.php
│   └── OwnershipGraph.php
│
├── RuntimeLocal/
│   ├── RuntimeLocalKey.php
│   ├── RuntimeLocalRegistry.php
│   └── RuntimeLocalStorageResolver.php
│
├── Verification/
│   ├── DefaultDatabaseIsolationVerifier.php
│   ├── ScopeIsolationVerifier.php
│   ├── TransactionIsolationVerifier.php
│   ├── ResourceIsolationVerifier.php
│   └── ConnectionIsolationVerifier.php
│
├── Leak/
│   ├── DatabaseStateLeakDetector.php
│   ├── ScopedStateLeak.php
│   └── IsolationLeakReport.php
│
├── Diagnostics/
│   ├── DatabaseIsolationDiagnostics.php
│   ├── IsolationGraphRenderer.php
│   └── IsolationViolationReporter.php
│
├── Policy/
│   └── DefaultDatabaseIsolationPolicy.php
│
└── Exception/
    ├── DatabaseStateIsolationException.php
    ├── CrossScopeStateAccessException.php
    ├── CrossOperationStateAccessException.php
    ├── CrossTransactionStateAccessException.php
    ├── StaleScopedStateException.php
    ├── InvalidStateOwnerException.php
    ├── UnsafeSharedStateException.php
    ├── ConcurrentResourceAccessException.php
    ├── ScopedStateLeakException.php
    └── RuntimeLocalIsolationException.php
```

---

# 278. State Classification Matrix

| Componente | Lifetime | Mutable | Compartible |
|---|---|---:|---|
| TypeRegistry | Application | No | Sí |
| ORM Metadata | Application/Worker | No | Sí |
| Dialect | Application | No | Sí |
| Compiler Rules | Application | No | Sí |
| Compiled Query Cache | Worker | Sí/controlado | Sí |
| Connection Pool | Worker | Sí | Sí/controlado |
| DatabaseContext | Execution | Principalmente no | No entre executions |
| EntityManager | Execution | Sí | No |
| IdentityMap | Execution | Sí | No |
| UnitOfWork | Execution | Sí | No |
| RoutingState | Execution | Sí | No |
| Tenant Binding | Execution | No | No |
| TransactionContext | Transaction | Sí | No |
| ConnectionLease | Resource | Sí | No |
| ResultCursor | Resource | Sí | No |
| StreamingResult | Resource | Sí | No |
| Lazy Iteration | Operation | Sí | No |
| Chunk Traversal | Operation | Sí | No |
| Query Parameters | Operation | Sí | No |
| Hydration Session | Operation | Sí | No |
| Query Profiler Session | Execution | Sí | No |
| N+1 Observation State | Execution | Sí | No |
| Telemetry Exporter | Application/Worker | Sí/controlado | Sí |

---

# 279. Architectural Invariants

## DB-ISO-001

Todo mutable state tendrá lifetime explícito.

## DB-ISO-002

Todo mutable state tendrá owner explícito.

## DB-ISO-003

Unknown ownership no equivaldrá a safe ownership.

## DB-ISO-004

Execution state no será worker-global.

## DB-ISO-005

Operation state no será execution-global sin necesidad.

## DB-ISO-006

Transaction state será transaction-local.

## DB-ISO-007

Resource state tendrá resource ownership.

## DB-ISO-008

Shared mutable state requerirá sincronización explícita.

## DB-ISO-009

Shared immutable state será preferido.

## DB-ISO-010

Isolation será mecanismo primario.

## DB-ISO-011

Reset no sustituirá isolation.

## DB-ISO-012

Scope no garantizará aislamiento si los componentes ignoran ownership.

## DB-ISO-013

DatabaseContext será execution-bound.

## DB-ISO-014

EntityManager será execution-scoped por default.

## DB-ISO-015

EntityManager no será worker-global.

## DB-ISO-016

IdentityMap será execution-scoped.

## DB-ISO-017

IdentityMap no se compartirá entre requests.

## DB-ISO-018

Tenant-aware key no justificará IdentityMap global.

## DB-ISO-019

UnitOfWork será execution-scoped.

## DB-ISO-020

Pending changes no cruzarán executions.

## DB-ISO-021

Entity managed state será EntityManager-relative.

## DB-ISO-022

Cross-EntityManager attachment no será implícito.

## DB-ISO-023

TransactionContext será transaction-bound.

## DB-ISO-024

Transaction state no cruzará executions.

## DB-ISO-025

Savepoint stack será transaction-local.

## DB-ISO-026

Retry attempt state será attempt-local.

## DB-ISO-027

Tenant binding será scoped.

## DB-ISO-028

Shard binding será scoped.

## DB-ISO-029

Routing state será scoped.

## DB-ISO-030

Sticky writer state no cruzará requests.

## DB-ISO-031

Replica affinity no cruzará requests.

## DB-ISO-032

Physical connection será distinta de ConnectionLease.

## DB-ISO-033

Lease será scoped.

## DB-ISO-034

Physical connection podrá reutilizarse sólo bajo política segura.

## DB-ISO-035

Connection session state deberá aislarse.

## DB-ISO-036

UNKNOWN connection state no equivaldrá a CLEAN.

## DB-ISO-037

Connection dudosa será quarantine/discard según política.

## DB-ISO-038

CompiledQuery podrá compartirse si es inmutable.

## DB-ISO-039

Live prepared statement handle respetará connection ownership.

## DB-ISO-040

ResultCursor será resource-local.

## DB-ISO-041

ResultCursor no cruzará scope.

## DB-ISO-042

StreamingResult no cruzará scope.

## DB-ISO-043

Active Lazy Iteration será operation-local.

## DB-ISO-044

Generator activo no se compartirá.

## DB-ISO-045

Chunk traversal state será operation-local.

## DB-ISO-046

No existirá static last processed ID.

## DB-ISO-047

Import state será operation-local.

## DB-ISO-048

Export state será operation-local.

## DB-ISO-049

Pagination state será operation-local.

## DB-ISO-050

No existirá static current page.

## DB-ISO-051

Query execution state será operation-local.

## DB-ISO-052

Query parameters no vivirán en shared compiler state.

## DB-ISO-053

Compiler shared será stateless/immutable.

## DB-ISO-054

Planner shared será stateless/immutable.

## DB-ISO-055

Optimizer shared no retendrá request state.

## DB-ISO-056

Hydration session será operation-local.

## DB-ISO-057

Relationship loading state será scoped.

## DB-ISO-058

N+1 observation state será execution-local.

## DB-ISO-059

Query profile session será execution-local.

## DB-ISO-060

Telemetry correlation será scoped.

## DB-ISO-061

Event dispatcher shared no almacenará current event mutable.

## DB-ISO-062

Deferred transaction events pertenecerán a su transaction.

## DB-ISO-063

Security state será execution-local.

## DB-ISO-064

Same user no equivaldrá a same execution.

## DB-ISO-065

Resource consumption será correctamente scoped.

## DB-ISO-066

Cancellation state no cruzará executions.

## DB-ISO-067

Deadline state no cruzará executions.

## DB-ISO-068

Runtime-local storage no será process-global.

## DB-ISO-069

RuntimeLocalStorage tendrá adapters por runtime.

## DB-ISO-070

Core no dependerá directamente de Swoole APIs.

## DB-ISO-071

Core no dependerá directamente de RoadRunner APIs.

## DB-ISO-072

Core no dependerá directamente de FrankenPHP internals.

## DB-ISO-073

Mutable static contextual state estará prohibido.

## DB-ISO-074

Static current tenant estará prohibido.

## DB-ISO-075

Static current transaction estará prohibido.

## DB-ISO-076

Static current EntityManager estará prohibido.

## DB-ISO-077

Static current connection estará prohibido.

## DB-ISO-078

Static current query estará prohibido.

## DB-ISO-079

Static caches deberán demostrar seguridad.

## DB-ISO-080

Static analysis podrá validar arquitectura.

## DB-ISO-081

Scope attributes no sustituirán correctness.

## DB-ISO-082

Container validará captive dependencies cuando sea posible.

## DB-ISO-083

Worker singleton no capturará scoped EntityManager.

## DB-ISO-084

Long-lived closure no capturará scoped mutable resources.

## DB-ISO-085

Singleton listener no almacenará request state.

## DB-ISO-086

IsolationGuard validará recursos críticos.

## DB-ISO-087

Critical ownership checks podrán permanecer activos en producción.

## DB-ISO-088

Cross-tenant violation será crítica.

## DB-ISO-089

Cross-transaction resource use será crítico.

## DB-ISO-090

UNKNOWN isolation status no equivaldrá a CLEAN.

## DB-ISO-091

Isolation se basará en evidencia.

## DB-ISO-092

Worker podrá marcarse TAINTED.

## DB-ISO-093

Scope taint no equivaldrá automáticamente a worker taint.

## DB-ISO-094

Failure containment será explícito.

## DB-ISO-095

Isolation utilizará defense in depth.

## DB-ISO-096

Generation podrá proteger contra stale references.

## DB-ISO-097

Closed scoped resource no será reactivado.

## DB-ISO-098

Released lease no será reutilizado por referencia vieja.

## DB-ISO-099

L0 cache será execution-local.

## DB-ISO-100

L1 cache no almacenará live scoped objects.

## DB-ISO-101

Entity cache no almacenará managed entity object.

## DB-ISO-102

Result cache no almacenará live cursor.

## DB-ISO-103

Metadata cache podrá ser shared.

## DB-ISO-104

Hydration cache almacenará planes, no managed entities.

## DB-ISO-105

Tenant-specific cache será tenant-keyed.

## DB-ISO-106

Security-specific cache será security-scope-aware.

## DB-ISO-107

Async event no transportará EntityManager vivo.

## DB-ISO-108

Async event no transportará TransactionContext vivo.

## DB-ISO-109

Isolation telemetry tendrá cardinalidad controlada.

## DB-ISO-110

Resource sin owner será anomalía.

## DB-ISO-111

Leak detector no sustituirá isolation.

## DB-ISO-112

Active transaction después del scope será leak crítico.

## DB-ISO-113

Isolation verification ocurrirá en lifecycle boundaries.

## DB-ISO-114

Unknown physical connection state podrá causar discard.

## DB-ISO-115

Security isolation no será debug-only.

## DB-ISO-116

Tenant isolation no podrá desactivarse por performance.

## DB-ISO-117

Sequential execution isolation tendrá pruebas.

## DB-ISO-118

Concurrent execution isolation tendrá pruebas.

## DB-ISO-119

Long-worker isolation tendrá soak tests.

## DB-ISO-120

Fault injection verificará containment.

## DB-ISO-121

Connection reuse tendrá isolation tests.

## DB-ISO-122

Session role no cruzará leases.

## DB-ISO-123

Session timezone no cruzará leases cuando sea contextual.

## DB-ISO-124

Session variables no cruzarán leases.

## DB-ISO-125

Advisory locks deberán limpiarse o causar discard.

## DB-ISO-126

Platform-specific reset estará detrás de capabilities.

## DB-ISO-127

SQLite tendrá política específica cuando corresponda.

## DB-ISO-128

Same scope no equivaldrá a concurrent resource safety.

## DB-ISO-129

EntityManager no será concurrent-safe por default.

## DB-ISO-130

UnitOfWork no será concurrent-safe por default.

## DB-ISO-131

IdentityMap no será concurrent-safe por default.

## DB-ISO-132

Unknown concurrency capability será tratada conservadoramente.

## DB-ISO-133

Concurrent operations podrán requerir recursos separados.

## DB-ISO-134

Resource ownership transfer no será implícito.

## DB-ISO-135

V1 preferirá non-transferable scoped mutable resources.

## DB-ISO-136

Object reference survival no implicará context survival.

## DB-ISO-137

Entity sobreviviente podrá quedar DETACHED.

## DB-ISO-138

Detached entity no recuperará global EntityManager.

## DB-ISO-139

Detached lazy proxy no hará I/O implícito.

## DB-ISO-140

Serialization no disparará lazy loading por default.

## DB-ISO-141

Scoped state escape será detectable donde sea viable.

## DB-ISO-142

Isolation aplicará a HTTP.

## DB-ISO-143

Isolation aplicará a jobs.

## DB-ISO-144

Isolation aplicará a CLI.

## DB-ISO-145

Isolation aplicará a tests.

## DB-ISO-146

Isolation aplicará a background operations.

## DB-ISO-147

Isolation aplicará a Fibers.

## DB-ISO-148

Isolation aplicará a coroutines.

## DB-ISO-149

Persistent runtime no reducirá las garantías de aislamiento.

## DB-ISO-150

Traditional PHP seguirá las mismas reglas conceptuales.

## DB-ISO-151

FrankenPHP seguirá las mismas reglas conceptuales.

## DB-ISO-152

RoadRunner seguirá las mismas reglas conceptuales.

## DB-ISO-153

OpenSwoole seguirá las mismas reglas conceptuales.

## DB-ISO-154

Worker reuse sólo será permitido después de una frontera segura.

## DB-ISO-155

Connection reuse sólo será permitido después de una frontera segura.

## DB-ISO-156

Process reuse no implicará state reuse.

## DB-ISO-157

Worker reuse no implicará EntityManager reuse.

## DB-ISO-158

Worker reuse no implicará UnitOfWork reuse.

## DB-ISO-159

Worker reuse no implicará IdentityMap reuse.

## DB-ISO-160

Worker reuse no implicará TransactionContext reuse.

## DB-ISO-161

Worker reuse no implicará TenantContext reuse.

## DB-ISO-162

Worker reuse no implicará RoutingState reuse.

## DB-ISO-163

Shared infrastructure será explícitamente clasificada.

## DB-ISO-164

Mutable contextual state tendrá siempre una frontera de aislamiento.

## DB-ISO-165

La ausencia de evidencia de contaminación no será equivalente a una prueba de aislamiento.

---

# 280. Modelo formal

Sea:

```text
S(E)
```

el conjunto de estados mutables pertenecientes a una ejecución `E`.

Para:

```text
E₁ ≠ E₂
```

se requiere:

```text
S(E₁) ∩ S(E₂) = ∅
```

para estado `NOT_SHARED`.

---

# 281. Shared state

Sea:

```text
G
```

un recurso compartido.

Su compartición sólo será válida si:

```text
Shareable(G)
∧
ConcurrencySafe(G)
∧
ContextIndependent(G)
```

o si su particionamiento garantiza aislamiento.

---

# 282. Resource ownership

Para un recurso `R` no compartido:

```text
Owner(R) = E
```

y una operación desde `E'` será válida sólo si:

```text
E' = E
```

o existe una transferencia explícita soportada.

---

# 283. Stale resource

Si:

```text
State(R) = CLOSED
```

entonces:

```text
Use(R) = INVALID
```

independientemente de que todavía exista una referencia PHP al objeto.

---

# 284. Connection reuse

Para reutilizar una conexión física `C`:

```text
Reusable(C)
=
NoActiveTransaction(C)
∧
NoActiveCursor(C)
∧
SessionStateClean(C)
∧
NoUnknownOutcome(C)
∧
PlatformResetSatisfied(C)
```

---

# 285. Conservative reuse

Si:

```text
Reusable(C) = UNKNOWN
```

entonces:

```text
ReturnToPool(C) = false
```

por default.

---

# 286. Isolation and reset relationship

La relación con el siguiente documento será:

```text
State Isolation
      │
      ├── defines ownership
      ├── defines boundaries
      └── prevents cross-scope sharing
               │
               ▼
State Reset
      │
      ├── clears reusable state
      ├── restores baseline
      └── verifies reusable resources
```

---

# 287. Anti-patterns

## Anti-pattern 1 — Global EntityManager

```php
EntityManager::$current
```

Prohibido.

## Anti-pattern 2 — Global tenant

```php
Tenant::$current
```

Prohibido.

## Anti-pattern 3 — Worker singleton UoW

```text
Worker
└── UnitOfWork
```

Prohibido.

## Anti-pattern 4 — Global current transaction

```php
TransactionManager::$transaction
```

Prohibido.

## Anti-pattern 5 — Reuse without reset

```text
Lease A
→ release
→ Lease B
```

sin verificar session state.

Prohibido.

## Anti-pattern 6 — Shared managed entities

Guardar managed entity en singleton/cache worker.

Prohibido.

## Anti-pattern 7 — Cleanup-only architecture

```text
use globals everywhere
+
clear globals after request
```

No será el modelo de VoltStack.

## Anti-pattern 8 — Coroutine-unsafe globals

Un array estático usado como current-context storage.

Prohibido cuando existen ejecuciones concurrentes.

## Anti-pattern 9 — Silent detached lazy loading

Entidad de request anterior intenta resolver nueva conexión global.

Prohibido.

## Anti-pattern 10 — UNKNOWN treated as CLEAN

Prohibido.

---

# 288. Arquitectura final

```text
                       VOLTSTACK WORKER
                              │
               ┌──────────────┴──────────────┐
               │                             │
               ▼                             ▼
      IMMUTABLE/SAFE SHARED             RESOURCE POOLS
         INFRASTRUCTURE                       │
               │                              │
      ┌────────┼────────┐                     │
      ▼        ▼        ▼                     │
 Metadata   Compiler   Types                   │
                                                │
         ┌──────────────────────────────────────┘
         │
         ▼
 ┌──────────────────┐
 │ EXECUTION SCOPE A│
 ├──────────────────┤
 │ Context A        │
 │ EntityManager A  │
 │ IdentityMap A    │
 │ UnitOfWork A     │
 │ RoutingState A   │
 │ Security A       │
 │ Transaction A    │
 └────────┬─────────┘
          │
          ▼
       Lease A
          │
          ▼
  Physical Connection
          │
       RESET
          │
          ▼
 ┌──────────────────┐
 │ EXECUTION SCOPE B│
 ├──────────────────┤
 │ Context B        │
 │ EntityManager B  │
 │ IdentityMap B    │
 │ UnitOfWork B     │
 │ RoutingState B   │
 │ Security B       │
 │ Transaction B    │
 └────────┬─────────┘
          │
          ▼
       Lease B
```

La conexión física puede ser reutilizada.

El estado lógico no.

---

# 289. Regla final

> **VoltStack Database tratará todo estado mutable como propiedad de una frontera de ejecución concreta. Ningún objeto será considerado seguro para compartir únicamente porque PHP permita mantenerlo vivo entre requests. La compartición deberá ser una propiedad arquitectónica demostrable, no una consecuencia accidental del runtime.**

Por tanto:

```text
Shared Process
≠
Shared Request State
```

```text
Shared Worker
≠
Shared ORM State
```

```text
Shared Connection Pool
≠
Shared Connection Lease
```

```text
Same User
≠
Same Execution
```

```text
Same Tenant
≠
Same UnitOfWork
```

```text
Reset
≠
Isolation
```

y:

```text
UNKNOWN
≠
SAFE
```

La arquitectura objetivo queda:

```text
Immutable Shared Infrastructure
             │
             ▼
      Persistent Worker
             │
    ┌────────┼────────┐
    ▼        ▼        ▼
 Scope A   Scope B   Scope C
    │        │        │
 State A   State B   State C
    │        │        │
    └────────┴────────┘
             │
       Shared Pools
             │
       Safe Reuse Only
```

---

# 290. Estado del Bloque 25

```text
BLOCK 25 — PERSISTENT RUNTIME

✓ 251_DATABASE_PERSISTENT_RUNTIME_ARCHITECTURE.md
✓ 252_DATABASE_REQUEST_SCOPE_SYSTEM.md
✓ 253_DATABASE_DATABASE_CONTEXT_SYSTEM.md
✓ 254_DATABASE_STATE_ISOLATION_SYSTEM.md
│
├── 255_DATABASE_STATE_RESET_SYSTEM.md
├── 256_DATABASE_CONNECTION_REUSE_SYSTEM.md
├── 257_DATABASE_WORKER_LIFECYCLE_SYSTEM.md
├── 258_DATABASE_FRANKENPHP_INTEGRATION_SYSTEM.md
├── 259_DATABASE_ROADRUNNER_INTEGRATION_SYSTEM.md
└── 260_DATABASE_OPENSWOOLE_INTEGRATION_SYSTEM.md
```

---

# 291. Siguiente documento

```text
255_DATABASE_STATE_RESET_SYSTEM.md
```

El siguiente documento definirá la segunda mitad del modelo de seguridad para runtimes persistentes:

```text
Isolation
   ↓
prevents incorrect sharing

Reset
   ↓
restores reusable resources to a known baseline
```

Se documentarán:

```text
State Reset Architecture
Resettable State Classification
Reset Baselines
Reset Plans
Reset Ordering
EntityManager Reset
IdentityMap Reset
UnitOfWork Reset
Transaction Reset
Routing State Reset
Tenant/Shard State Reset
Query State Reset
Telemetry State Reset
Connection Session Reset
Prepared Statement Reset
Temporary Object Cleanup
Advisory Lock Cleanup
Reset Verification
Unknown State Handling
Resource Quarantine
Worker Reset
Reset Failures
Platform-specific Reset Capabilities
FrankenPHP Worker Safety
RoadRunner Worker Safety
OpenSwoole Worker Safety
Reset Testing
```

con la regla fundamental:

> **Un recurso sólo podrá reutilizarse cuando VoltStack pueda demostrar que ha regresado a un estado base válido; si el resultado del reset es desconocido, el recurso no será considerado limpio.**