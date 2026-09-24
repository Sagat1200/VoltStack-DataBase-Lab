# 252_DATABASE_REQUEST_SCOPE_SYSTEM.md

# VoltStack Quantum Database
## Request Scope System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 252 — Database Request Scope System  
**Bloque:** 25 — Persistent Runtime  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `251_DATABASE_PERSISTENT_RUNTIME_ARCHITECTURE.md`  
**Siguiente documento:** `253_DATABASE_DATABASE_CONTEXT_SYSTEM.md`

---

# 1. Propósito

Este documento define el **Request Scope System** de `VoltStack/Quantum/Database`.

Su responsabilidad es crear una frontera de aislamiento para todo estado mutable de Database perteneciente a una ejecución concreta.

Aunque el nombre público sea:

```text
Request Scope
```

el concepto arquitectónico es más amplio:

```text
Database Execution Scope
```

porque VoltStack deberá soportar:

```text
HTTP Request
Queue Job
CLI Command
Scheduled Task
Test
Background Operation
RPC Request
WebSocket Operation
```

La regla central es:

> **Cada ejecución lógica de VoltStack deberá operar dentro de un Database Execution Scope propio. Ningún estado mutable perteneciente a una ejecución podrá ser reutilizado accidentalmente por otra.**

Formalmente:

```text
Execution A ≠ Execution B
        ↓
MutableDatabaseState(A)
∩
MutableDatabaseState(B)
=
∅
```

salvo infraestructura explícitamente compartida y diseñada para ello.

---

# 2. Contexto

En PHP tradicional, gran parte del aislamiento se obtiene indirectamente:

```text
Request
   ↓
PHP Process
   ↓
Response
   ↓
Process destroyed
```

En FrankenPHP, RoadRunner u OpenSwoole:

```text
Worker
├── Request A
├── Request B
├── Request C
└── Request D
```

El proceso permanece vivo.

Por ello VoltStack no puede depender de:

```text
process termination
```

para limpiar:

```text
EntityManager
IdentityMap
UnitOfWork
TransactionContext
TenantContext
ShardContext
ConnectionLease
QueryContext
Lazy iterators
Result cursors
Telemetry state
```

---

# 3. Request Scope ≠ PHP Request

Aunque HTTP será un consumidor importante:

```text
Database Request Scope
≠
strictly HTTP Request
```

El nombre representa una unidad lógica de ejecución aislada.

---

# 4. Terminología

En este documento:

```text
Execution
```

representa cualquier unidad de trabajo aislada.

Ejemplos:

```text
HTTP request
queue job
CLI command
scheduled execution
test case
background operation
```

---

# 5. Execution Scope

La abstracción principal será:

```text
DatabaseExecutionScope
```

---

# 6. Objetivos

El sistema deberá proporcionar:

1. creación explícita de scopes;
2. identificación única;
3. ownership de recursos;
4. aislamiento entre ejecuciones;
5. resolución contextual;
6. propagación segura;
7. finalización determinista;
8. cleanup;
9. detección de recursos abiertos;
10. integración con EntityManager;
11. integración con IdentityMap;
12. integración con UnitOfWork;
13. integración con conexiones;
14. integración con transacciones;
15. integración con tenant/shard;
16. integración con telemetry;
17. integración con cancellation;
18. soporte concurrente;
19. soporte persistent runtime;
20. diagnósticos de scope.

---

# 7. No objetivos

Request Scope no será responsable de implementar:

```text
SQL compilation
Query optimization
ORM persistence
Connection pooling
Transaction protocol
Tenant resolution business logic
Authorization rules
Telemetry backend
```

Su función es:

```text
lifecycle + ownership + isolation
```

---

# 8. Arquitectura general

```text
Runtime
   │
   ▼
Execution Begins
   │
   ▼
DatabaseScopeFactory
   │
   ▼
DatabaseExecutionScope
   │
   ├── DatabaseContext
   ├── EntityManager
   ├── IdentityMap
   ├── UnitOfWork
   ├── TransactionContext
   ├── RoutingContext
   ├── ResourceRegistry
   ├── CancellationContext
   └── TelemetryContext
   │
   ▼
Application Work
   │
   ▼
Scope Finalization
   │
   ├── close cursors
   ├── resolve transactions
   ├── release connections
   ├── clear ORM state
   ├── release resources
   └── verify isolation
   │
   ▼
CLOSED
```

---

# 9. Scope states

Se propone:

```php
enum DatabaseScopeState
{
    case CREATED;
    case ACTIVE;
    case FINALIZING;
    case CLOSED;
    case TAINTED;
}
```

---

# 10. CREATED

El scope existe pero todavía no ha sido activado.

---

# 11. ACTIVE

Puede ejecutar operaciones Database.

---

# 12. FINALIZING

No acepta nuevas operaciones.

Se están cerrando recursos.

---

# 13. CLOSED

El scope terminó.

Cualquier operación nueva deberá fallar.

---

# 14. TAINTED

El scope ha sufrido una condición donde su estado ya no puede considerarse confiable.

Ejemplos:

```text
unknown transaction outcome
cleanup failure
cross-scope resource corruption
connection state uncertainty
```

---

# 15. Transiciones

```text
CREATED
   │
   ▼
ACTIVE
   │
   ▼
FINALIZING
   │
   ├─────────────┐
   ▼             ▼
CLOSED        TAINTED
```

También:

```text
ACTIVE
   ↓
TAINTED
```

ante fallo crítico.

---

# 16. Transiciones prohibidas

Nunca:

```text
CLOSED → ACTIVE
```

Nunca:

```text
TAINTED → ACTIVE
```

por simple reset del objeto.

---

# 17. Scope ID

Cada scope tendrá:

```text
DatabaseScopeId
```

único dentro de la ejecución observable.

Ejemplo conceptual:

```text
dbscope_01K8...
```

---

# 18. Execution ID ≠ Scope ID

Una ejecución puede tener:

```text
ExecutionId
```

y:

```text
DatabaseScopeId
```

distintos.

Esto permite en el futuro scopes especializados.

---

# 19. Scope generation

También:

```text
ScopeGeneration
```

puede ayudar a detectar referencias obsoletas.

---

# 20. Scope Token

Los recursos recibirán:

```text
ScopeToken
```

para comprobar ownership.

```php
final readonly class ScopeToken
{
    public function __construct(
        public DatabaseScopeId $scopeId,
        public ScopeGeneration $generation,
    ) {}
}
```

---

# 21. DatabaseExecutionScope

Contrato conceptual:

```php
interface DatabaseExecutionScope
{
    public function id(): DatabaseScopeId;

    public function token(): ScopeToken;

    public function state(): DatabaseScopeState;

    public function context(): DatabaseContext;

    public function resources(): DatabaseScopeResourceRegistry;

    public function cancellation(): CancellationToken;

    public function assertActive(): void;
}
```

---

# 22. Scope Factory

Creación mediante:

```text
DatabaseScopeFactory
```

---

# 23. Factory contract

```php
interface DatabaseScopeFactory
{
    public function create(
        ExecutionDescriptor $execution
    ): DatabaseExecutionScope;
}
```

---

# 24. Factory responsibilities

La factory deberá:

```text
allocate ScopeId
create ScopeToken
create DatabaseContext
initialize resource registry
initialize cancellation context
bind runtime context
register scope services
```

pero evitar inicialización innecesaria.

---

# 25. Lazy component creation

No todos los requests utilizan Database.

Por tanto:

```text
HTTP Request
    ↓
no DB access
```

no deberá necesariamente crear:

```text
EntityManager
IdentityMap
UnitOfWork
Connection
```

---

# 26. Lazy scoped services

El scope podrá utilizar:

```text
ScopedLazy<T>
```

o mecanismo equivalente.

Ejemplo:

```text
DatabaseExecutionScope
├── DatabaseContext
├── EntityManager [lazy]
├── TransactionContext [lazy]
└── ConnectionLease [lazy]
```

---

# 27. Request Scope ≠ Connection

Crear scope no significa:

```text
open database connection
```

---

# 28. Request Scope ≠ Transaction

Crear scope tampoco significa:

```text
BEGIN
```

---

# 29. Request Scope ≠ EntityManager activation

Incluso EntityManager puede ser inicializado únicamente cuando se utilice ORM.

---

# 30. Scope activation

El runtime ejecutará conceptualmente:

```php
$scope = $scopeManager->beginExecution($descriptor);

try {
    return $application->handle($request);
} finally {
    $scopeManager->finishExecution($scope);
}
```

---

# 31. finally obligatorio

La arquitectura deberá garantizar:

```text
execution
    ↓
finally
    ↓
scope finalization
```

incluso cuando exista una excepción normal de aplicación.

---

# 32. Scope binding

Una vez creado, el scope deberá estar disponible para los servicios que lo requieran.

---

# 33. Scope Resolver

Se propone:

```text
DatabaseScopeResolver
```

---

# 34. Resolver contract

```php
interface DatabaseScopeResolver
{
    public function current(): DatabaseExecutionScope;

    public function hasActiveScope(): bool;
}
```

---

# 35. Resolver semantics

`current()` fuera de un scope deberá lanzar:

```text
DatabaseScopeNotActiveException
```

Nunca crear un scope implícito global.

---

# 36. Scoped resolution

Conceptualmente:

```text
RuntimeLocalStorage
      ↓
DatabaseScopeResolver
      ↓
current ScopeToken
      ↓
DatabaseExecutionScope
```

---

# 37. No static mutable current scope

Prohibido:

```php
final class Database
{
    public static ?DatabaseExecutionScope $scope;
}
```

---

# 38. Concurrencia

Esto es especialmente importante con:

```text
OpenSwoole
Fibers
coroutines
concurrent runtime execution
```

---

# 39. Concurrent executions

Ejemplo:

```text
Worker
├── Execution A
│   └── Scope A
│
└── Execution B
    └── Scope B
```

Debe cumplirse:

```text
current(A) = Scope A
current(B) = Scope B
```

aunque ambas ejecuciones vivan en el mismo proceso.

---

# 40. Scope ownership

Todo recurso contextual deberá registrar owner.

Ejemplo:

```text
ConnectionLease
owner = Scope A
```

---

# 41. Ownership validation

Antes de usar un recurso:

```text
CurrentScopeToken
        =
ResourceOwnerToken
```

cuando el tipo de recurso requiera validación.

---

# 42. Cross-scope resource access

Si:

```text
resource.owner = Scope A
current = Scope B
```

resultado:

```text
CrossScopeResourceAccessException
```

---

# 43. EntityManager scope

Por defecto:

```text
EntityManager Lifetime
=
DatabaseExecutionScope
```

---

# 44. EntityManager resolver

El Model API podrá utilizar:

```text
ModelContextResolver
    ↓
DatabaseScopeResolver
    ↓
EntityManager
```

sin mantener EntityManager estático.

---

# 45. Active Record convenience

Esto permite:

```php
$user = User::find(10);
```

sin convertir:

```text
User::$entityManager
```

en estado global.

---

# 46. EntityManager isolation

```text
Scope A → EntityManager A
Scope B → EntityManager B
```

---

# 47. EntityManager reuse

Por default:

```text
EntityManager A
```

no será reciclado para convertirse en:

```text
EntityManager B
```

mediante simple `clear()`.

Es preferible una instancia scoped nueva o un contexto mutable completamente aislado.

---

# 48. IdentityMap

Cada scope tendrá su propia IdentityMap.

```text
Scope A
└── IdentityMap A

Scope B
└── IdentityMap B
```

---

# 49. IdentityMap guarantee

Dentro de un scope:

```text
same EntityType
+
same identifier
+
same persistence domain
+
same tenant/shard context
```

produce la misma instancia managed.

---

# 50. Across scopes

Entre scopes:

```text
IdentityMap A
≠
IdentityMap B
```

Por tanto:

```php
$a = scopeA(fn () => User::find(1));
$b = scopeB(fn () => User::find(1));

$a !== $b;
```

es comportamiento normal.

---

# 51. UnitOfWork

Cada EntityManager scoped tendrá su:

```text
UnitOfWork
```

asociada.

---

# 52. Pending changes

Al terminar scope:

```text
pending changes
```

no deberán migrar a otro scope.

---

# 53. Unflushed changes

Por defecto:

```text
Unflushed changes
    ↓
discarded from ORM context
```

al cerrar scope.

No deberán:

```text
auto flush
```

silenciosamente.

---

# 54. Scope end ≠ flush

Regla:

```text
Request End
≠
Automatic ORM Flush
```

salvo una política superior explícita.

---

# 55. Scope end ≠ commit

También:

```text
Request End
≠
Automatic Commit
```

---

# 56. TransactionContext

Las transacciones deberán asociarse al scope.

---

# 57. Transaction ownership

```text
TransactionContext
owner = DatabaseScopeId
```

---

# 58. Transaction affinity

Una transaction no puede migrar automáticamente:

```text
Scope A → Scope B
```

---

# 59. Active transaction at finalization

Si el scope termina con:

```text
ACTIVE transaction
```

la política normal será:

```text
rollback
```

si su outcome sigue siendo conocido.

---

# 60. Unknown transaction

Si:

```text
transaction outcome = UNKNOWN
```

el scope deberá quedar:

```text
TAINTED
```

y los recursos asociados no podrán declararse limpios sin evidencia.

---

# 61. Nested transactions

Nested transaction scopes siguen perteneciendo al mismo DatabaseExecutionScope.

```text
Execution Scope
└── Transaction
    └── Savepoint
```

---

# 62. Connection leases

Una conexión física puede provenir de infraestructura worker-wide.

Pero:

```text
ConnectionLease
```

pertenece al scope.

---

# 63. Example

```text
ConnectionPool
     │
     ▼
PhysicalConnection #4
     │
     ▼
Lease #91
owner = Scope A
```

Al terminar Scope A:

```text
Lease #91
    ↓
release/reset/discard
```

---

# 64. Lease cannot escape

No se permitirá almacenar un lease activo para utilizarlo después del cierre del scope.

---

# 65. Routing Context

El scope mantendrá estado de routing relevante.

Ejemplos:

```text
writer affinity
sticky read state
replica selection context
minimum replication position
```

---

# 66. Routing isolation

Request A puede haber escrito y activar:

```text
sticky writer
```

Esto no deberá convertir Request B automáticamente en sticky.

---

# 67. Tenant Context

Tenant será parte del contexto scoped.

---

# 68. Tenant resolution

Conceptualmente:

```text
HTTP/Auth/Tenant Resolver
        ↓
Execution Descriptor
        ↓
Database Scope
        ↓
TenantContext
```

---

# 69. Tenant immutability

Una vez establecida la identidad efectiva del tenant, se preferirá:

```text
immutable TenantContext
```

durante el scope.

---

# 70. Tenant switching

No:

```php
Database::setTenant('acme');
...
Database::setTenant('contoso');
```

como mutación global.

---

# 71. Explicit tenant operation

Cuando sea necesario:

```php
$tenantOperations->run(
    $tenantId,
    fn () => ...
);
```

con una frontera explícita.

---

# 72. Cross-tenant work

Puede requerir:

```text
child operation scope
```

o contexto explícito separado.

No deberá contaminar el tenant principal del request.

---

# 73. Shard Context

Igual principio:

```text
ShardContext
```

será contextual.

---

# 74. Shard affinity

Una transaction distribuida ordinaria no deberá cambiar silenciosamente de shard.

---

# 75. Persistence Domain

El scope deberá conocer el dominio lógico de persistencia relevante:

```text
database
tenant
shard
connection domain
```

según configuración.

---

# 76. DatabaseContext

El siguiente documento definirá en profundidad:

```text
DatabaseContext
```

El scope será su propietario.

---

# 77. Relationship

```text
DatabaseExecutionScope
        │
        owns
        ▼
DatabaseContext
```

---

# 78. DatabaseContext ≠ Scope

Son conceptos relacionados pero distintos.

```text
Scope
=
lifecycle + ownership boundary

DatabaseContext
=
effective database execution context
```

---

# 79. Operation Scope

Dentro del request podrán existir operaciones.

```text
DatabaseExecutionScope
└── DatabaseOperationScope
```

---

# 80. Operation examples

```text
query
transaction callback
chunk traversal
bulk operation
import
export
lazy traversal
```

---

# 81. OperationContext

Puede contener:

```text
deadline
timeout
retry policy
cancellation
telemetry span
resource budget
consistency requirement
```

---

# 82. Operation scope ≠ new EntityManager

Crear una operación no implica crear otro ORM completo.

---

# 83. Nested operation

```text
Execution
└── Operation A
    └── Operation B
```

deberá tener reglas explícitas de herencia.

---

# 84. Context inheritance

Por defecto una operación hija podrá heredar:

```text
tenant
shard constraints
deadline upper bound
cancellation parent
security context
transaction affinity
```

---

# 85. Deadline narrowing

Si:

```text
Parent deadline = 5s
Child requested = 20s
```

el hijo no deberá extender arbitrariamente el deadline.

Conceptualmente:

```text
EffectiveDeadline
=
min(parent, child)
```

---

# 86. Cancellation propagation

```text
Parent cancelled
    ↓
Children cancelled
```

por default.

---

# 87. Child cancellation

Cancelar una operación hija no necesariamente cancela todo el request, salvo política.

---

# 88. CancellationToken

Cada execution tendrá token raíz.

```text
ExecutionCancellationToken
```

---

# 89. Cancellation integration

Deberá integrarse con:

```text
Query Executor
Streaming Result
Lazy Collection
Chunk Processing
Import
Export
Bulk Operations
Transaction Retry
```

---

# 90. Cancellation ≠ rollback guarantee

Cancelar una operación no implica automáticamente que el DBMS haya revertido todo.

---

# 91. Cancellation outcome

Puede ser:

```text
CANCELLED_CLEANLY
CANCELLED_WITH_ROLLBACK
CANCELLED_UNKNOWN
```

según evidencia.

---

# 92. Resource Registry

Cada scope mantendrá:

```text
DatabaseScopeResourceRegistry
```

---

# 93. Registered resources

Ejemplos:

```text
ConnectionLease
ResultCursor
StreamingResult
LazySource
TransactionResource
TemporaryResource
ResourcePermit
```

---

# 94. Registry responsibilities

```text
register
track ownership
track lifecycle
finalize
diagnose leaks
```

---

# 95. Registry ≠ Garbage Collector

El registry no reemplaza PHP GC.

Su propósito es:

```text
semantic resource lifecycle
```

---

# 96. Resource record

Conceptualmente:

```php
final class ScopedResourceEntry
{
    public ResourceId $id;

    public ScopeToken $owner;

    public ResourceType $type;

    public ResourceState $state;

    public ScopedDatabaseResource $resource;
}
```

---

# 97. Resource registration

Cuando se abre un cursor:

```text
open
 ↓
register
```

Cuando se cierra:

```text
close
 ↓
mark released
```

---

# 98. Early release

Los recursos deberán liberarse tan pronto como ya no sean necesarios.

No esperar siempre al final del request.

---

# 99. Scope cleanup as safety net

Finalization actúa como:

```text
last deterministic lifecycle boundary
```

no como excusa para mantener recursos abiertos.

---

# 100. Finalization

Al terminar execution:

```text
ACTIVE
   ↓
FINALIZING
```

---

# 101. New work blocked

En `FINALIZING`:

```text
new query
new transaction
new cursor
```

deberán rechazarse.

---

# 102. Finalization plan

Orden conceptual:

```text
1. mark FINALIZING
2. cancel pending work
3. stop accepting new DB operations
4. close active streams/cursors
5. finalize lazy sources
6. inspect ORM state
7. resolve active transactions
8. release connection leases
9. clear UoW
10. clear IdentityMap
11. close EntityManager
12. release resource permits
13. clear contextual references
14. verify resource registry
15. determine scope disposition
16. mark CLOSED or TAINTED
```

---

# 103. ORM finalization ordering

Debe evitarse:

```text
release connection
    ↓
then attempt rollback
```

Por ello el dependency graph es importante.

---

# 104. Resource dependency graph

Ejemplo:

```text
Transaction
    ↓ depends on
ConnectionLease

ResultCursor
    ↓ depends on
ConnectionLease
```

Por tanto:

```text
close cursor
rollback transaction
release lease
```

---

# 105. Finalization DAG

El sistema podrá construir o predefinir un DAG:

```text
Cursor
  ─────┐
       ▼
Transaction
       │
       ▼
ConnectionLease
```

dependiendo del tipo de recurso.

---

# 106. Finalization idempotency

Llamar dos veces a finalization deberá:

```text
return existing result
```

o comportarse de manera segura.

No ejecutar dos rollbacks inconsistentes.

---

# 107. Scope finalization result

```php
final readonly class ScopeFinalizationResult
{
    public function __construct(
        public DatabaseScopeId $scopeId,
        public ScopeCleanupStatus $status,
        public WorkerDisposition $workerDisposition,
        public array $resourceResults,
        public array $warnings,
    ) {}
}
```

---

# 108. Clean scope

Resultado ideal:

```text
CLEAN
```

---

# 109. Clean with warnings

Ejemplo:

```text
cursor leaked by application
but successfully closed by finalizer
```

Puede resultar:

```text
CLEAN_WITH_WARNINGS
```

---

# 110. Tainted scope

Ejemplo:

```text
rollback failed
```

Resultado:

```text
TAINTED
```

---

# 111. Worker disposition

El scope puede sugerir:

```text
REUSE
REUSE_WITH_WARNING
RECYCLE
TERMINATE
```

---

# 112. Scope finalization ≠ worker shutdown

Cerrar Request Scope no significa cerrar worker.

---

# 113. Connection reuse

Incluso cuando el scope termina:

```text
physical connection
```

puede volver al pool si es segura.

Pero:

```text
lease
```

termina.

---

# 114. ORM objects

EntityManager scoped no deberá regresar a un pool por defecto.

---

# 115. Reusable services

Servicios inmutables sí pueden permanecer:

```text
Metadata
TypeRegistry
Dialect
Compiler
Optimization Rules
```

---

# 116. Request-scoped services

Ejemplos:

```text
EntityManager
IdentityMap
UnitOfWork
DatabaseContext
TransactionContext
RoutingContext
NPlusOneExecutionTracker
QueryProfilerContext
```

---

# 117. Operation-scoped services

Ejemplos:

```text
QueryExecutionContext
RetryContext
Deadline
ResourceBudget
Cancellation child token
```

---

# 118. Service scope declaration

Los servicios Database deberán poder declarar:

```php
enum DatabaseServiceLifetime
{
    case APPLICATION;
    case WORKER;
    case EXECUTION;
    case OPERATION;
    case TRANSACTION;
}
```

---

# 119. Container integration

VoltStack Container deberá conocer estos lifetimes.

---

# 120. Captive dependency validation

Ejemplo inválido:

```text
WORKER service
    ↓ direct dependency
EXECUTION EntityManager
```

---

# 121. Safe resolver

Alternativa:

```text
WORKER service
    ↓
EntityManagerResolver
    ↓
current execution
```

---

# 122. Resolver lifetime

El resolver puede ser worker/application-safe si no retiene la instancia resuelta.

---

# 123. Resolver anti-pattern

Incorrecto:

```php
final class EntityManagerResolver
{
    private ?EntityManager $cached = null;
}
```

si esa caché sobrevive scopes.

---

# 124. Scope-aware facade

`DB` o `Model` podrán parecer globales al desarrollador.

Ejemplo:

```php
DB::table('users')->get();
```

Pero internamente:

```text
Facade
  ↓
Scoped Resolver
  ↓
Current DatabaseExecutionScope
  ↓
DatabaseContext
```

---

# 125. Facade ≠ global mutable service

Ésta será una regla general de VoltStack.

---

# 126. Async boundaries

Un scope no deberá propagarse automáticamente más allá de su lifetime.

---

# 127. Invalid capture

```php
$scope = $resolver->current();

$background->dispatch(function () use ($scope) {
    // invalid after request
});
```

---

# 128. Async handoff

Debe transferirse:

```text
safe immutable execution descriptor
```

y crear otro scope.

---

# 129. Fiber propagation

Si una Fiber pertenece al mismo request, podrá heredar scope mediante RuntimeLocalStorage compatible.

---

# 130. Fiber outliving request

No podrá continuar usando el scope después del cierre.

---

# 131. Concurrent fiber access

Incluso dentro del mismo scope, recursos concretos pueden no ser concurrent-safe.

---

# 132. Scope sharing ≠ resource concurrency

Ejemplo:

```text
same execution scope
```

no implica que:

```text
same PDO connection
```

pueda usarse concurrentemente desde dos operaciones.

---

# 133. Resource concurrency capability

Los recursos deberán declarar o conocer su capacidad:

```text
EXCLUSIVE
SERIALIZED
CONCURRENT_SAFE
UNKNOWN
```

---

# 134. UNKNOWN concurrency

```text
UNKNOWN
≠
CONCURRENT_SAFE
```

---

# 135. Scope fork

Podrá existir en el futuro:

```text
scope->forkOperation()
```

pero no:

```text
scope->clone()
```

para copiar recursos vivos.

---

# 136. Child operation ownership

Los recursos creados por operación hija podrán seguir siendo propiedad del execution scope con sub-owner operation ID.

---

# 137. Operation ID

```text
DatabaseOperationId
```

permitirá diagnóstico fino.

---

# 138. Ownership tuple

Un recurso puede identificar:

```text
ExecutionScopeId
OperationId
TransactionId
```

según necesidad.

---

# 139. Request Scope and transactions

El Request Scope no impondrá:

```text
one request = one transaction
```

---

# 140. Multiple transactions

Permitido:

```text
Request
├── Transaction A
├── no transaction
└── Transaction B
```

---

# 141. Whole-request transaction

Puede ofrecerse como política explícita, pero no default universal.

---

# 142. Riesgo de transaction-per-request

Puede causar:

```text
long transactions
locks
contention
streaming problems
external side-effect coupling
```

Por ello no será implícita.

---

# 143. Request Scope and retry

Reintentar una transaction:

```text
does not recreate entire HTTP request scope
```

necesariamente.

Puede crear:

```text
new transaction attempt context
```

dentro del mismo execution.

---

# 144. Retry state isolation

Cada attempt deberá limpiar:

```text
transaction-specific state
```

antes del siguiente intento.

---

# 145. ORM retry caveat

Un retry transaccional puede requerir reconstruir parte del estado ORM.

No se asumirá que:

```text
rollback
=
object graph rewind
```

---

# 146. Request Scope and Cache

Cache shared puede sobrevivir scopes.

Pero cualquier:

```text
request-local cache
```

deberá pertenecer al scope.

---

# 147. L0 Cache

Ejemplo:

```text
L0 request-local metadata/result helper cache
```

se elimina al finalizar scope.

---

# 148. L1/L2 Cache

Puede sobrevivir si está correctamente aislado por keys y políticas.

---

# 149. Request Scope and Events

Listeners shared podrán recibir:

```text
DatabaseEventContext
```

sin almacenar current request globalmente.

---

# 150. Event listener capture

Un listener no deberá guardar:

```text
$currentEntity
$currentTransaction
$currentTenant
```

en singleton mutable.

---

# 151. Request Scope and Telemetry

Cada scope tendrá correlación con:

```text
trace
request
worker
runtime
```

---

# 152. Scope telemetry

Eventos sugeridos:

```text
database.scope.created
database.scope.activated
database.scope.finalizing
database.scope.closed
database.scope.tainted
database.scope.resource_leak
database.scope.cross_scope_access
```

---

# 153. Scope metrics

```text
database_scope_duration
database_scope_active_resources
database_scope_cleanup_duration
database_scope_cleanup_failures
```

---

# 154. Cardinality

`scope.id` deberá utilizarse en traces/logs cuando sea apropiado, no necesariamente como label de métricas.

---

# 155. Debugging

Developer Toolbar podrá mostrar:

```text
Database Scope
──────────────
ID: dbscope_xxx
State: ACTIVE
Runtime: FrankenPHP
Tenant: acme
Shard: shard-02

ORM
Managed entities: 18
Pending changes: 2

Transactions
Active: 1

Resources
Connections: 1
Cursors: 0
Streams: 0
```

---

# 156. End-of-request debug

Después:

```text
Scope: CLOSED
Cleanup: CLEAN
Leaked resources: 0
```

---

# 157. Security

Scope isolation constituye una frontera de seguridad.

---

# 158. Tenant leakage

Nunca:

```text
Scope A tenant
    ↓
Scope B
```

---

# 159. Authorization leakage

Nunca reutilizar:

```text
previous actor
previous permissions
previous security filters
```

sin nueva resolución.

---

# 160. Query parameter leakage

Los parámetros de una query no deberán permanecer en servicios worker-shared.

---

# 161. Sensitive data cleanup

El scope deberá liberar referencias a:

```text
credentials
PII
query values
security context
tenant-specific objects
```

tan pronto como corresponda.

---

# 162. Scope serialization

`DatabaseExecutionScope` no será serializable.

---

# 163. Scope persistence

No se almacenará:

```text
scope object
```

en queue payload, session, cache o persistent storage.

---

# 164. Transfer descriptor

Si se requiere transferir contexto:

```text
DatabaseExecutionDescriptor
```

contendrá sólo datos permitidos.

---

# 165. Descriptor example

```php
final readonly class DatabaseExecutionDescriptor
{
    public function __construct(
        public ExecutionType $type,
        public ?TenantId $tenant,
        public ?PersistenceDomainId $domain,
        public ?TraceCorrelation $trace,
    ) {}
}
```

sin:

```text
Connection
EntityManager
Transaction
Cursor
```

---

# 166. Testing

El Request Scope System requiere pruebas específicas.

---

# 167. Scope creation test

Verificar:

```text
CREATED → ACTIVE
```

---

# 168. Scope close test

```text
ACTIVE → FINALIZING → CLOSED
```

---

# 169. Closed access test

Después de close:

```php
$scope->context();
```

si requiere estado activo deberá fallar apropiadamente.

---

# 170. Scope isolation test

```text
Scope A state
≠
Scope B state
```

---

# 171. EntityManager isolation test

```text
EM(A) !== EM(B)
```

---

# 172. IdentityMap isolation test

```text
IdentityMap(A) !== IdentityMap(B)
```

---

# 173. UoW isolation test

```text
UoW(A) !== UoW(B)
```

---

# 174. Tenant isolation test

```text
Tenant(A) cannot appear in B
```

---

# 175. Routing isolation test

Sticky state de A no aparece en B.

---

# 176. Transaction isolation test

Transaction de A no aparece en B.

---

# 177. Connection lease test

Lease de A no puede usarse desde B.

---

# 178. Cursor cleanup test

Cursor abierto durante A es cerrado en finalization.

---

# 179. Exception cleanup test

```text
application throws
    ↓
scope still finalizes
```

---

# 180. Unknown transaction test

```text
commit outcome unknown
    ↓
scope TAINTED
    ↓
unsafe connection discarded
```

---

# 181. Concurrent context test

Ejecutar A/B intercaladas y comprobar:

```text
currentScope(A) = A
currentScope(B) = B
```

---

# 182. Long worker test

Ejecutar miles de scopes:

```text
A1
A2
...
A100000
```

verificando:

```text
no stale contextual references
```

---

# 183. Resource leak test

Dejar intencionalmente:

```text
cursor
lease
transaction
```

y verificar detección.

---

# 184. Captive dependency test

Container deberá rechazar configuraciones inválidas cuando la validación esté habilitada.

---

# 185. Scope reuse test

Un objeto `DatabaseExecutionScope` cerrado no podrá reactivarse para otro request.

---

# 186. Performance requirements

Crear un scope deberá ser barato.

---

# 187. Lazy initialization

La mayor parte del costo deberá ocurrir únicamente si Database se utiliza.

---

# 188. Zero-query request

Idealmente:

```text
begin scope
application does no DB work
finish scope
```

deberá tener overhead mínimo.

---

# 189. Activated resource tracking

El scope registrará qué subsistemas realmente fueron utilizados.

---

# 190. Finalization optimization

No ejecutar:

```text
IdentityMap cleanup
```

si nunca existió IdentityMap.

No ejecutar:

```text
transaction cleanup
```

si nunca existió TransactionContext.

---

# 191. Precompiled cleanup plan

Servicios conocidos pueden aportar:

```text
ScopeCleanupHook
```

compilado durante bootstrap.

---

# 192. Cleanup Hook

```php
interface ScopeCleanupHook
{
    public function priority(): int;

    public function cleanup(
        DatabaseExecutionScope $scope
    ): ScopeCleanupResult;
}
```

---

# 193. Priority caveat

Una simple prioridad numérica puede no ser suficiente para dependencias complejas.

Por ello podrán declararse:

```text
before
after
dependsOn
```

---

# 194. Scope Cleanup Graph

```text
StreamingResult
      ↓
ResultCursor
      ↓
Transaction
      ↓
ConnectionLease
      ↓
EntityManager cleanup
      ↓
Context cleanup
```

según recursos activos.

---

# 195. Cleanup errors

Cada error deberá registrarse sin impedir necesariamente intentar limpiar otros recursos independientes.

---

# 196. Error aggregation

Podrá utilizarse:

```text
ScopeCleanupReport
```

con múltiples fallos.

---

# 197. Cleanup exception

La aplicación podrá haber fallado con excepción X y cleanup con Y.

VoltStack no deberá perder ninguna evidencia.

---

# 198. Primary failure

Se preservará la excepción principal y se adjuntará información de cleanup como secondary failure cuando corresponda.

---

# 199. Scope outcome

Separar:

```text
ExecutionOutcome
```

de:

```text
ScopeCleanupStatus
```

Ejemplo:

```text
ExecutionOutcome = FAILURE
ScopeCleanupStatus = CLEAN
```

es perfectamente válido.

---

# 200. Otro ejemplo

```text
ExecutionOutcome = SUCCESS
ScopeCleanupStatus = TAINTED
```

también puede ocurrir.

---

# 201. Success ≠ clean

Por tanto:

```text
ApplicationSuccess
≠
DatabaseScopeClean
```

---

# 202. Failure ≠ dirty

También:

```text
ApplicationFailure
≠
DatabaseScopeDirty
```

---

# 203. Request Scope diagnostics model

```php
final readonly class DatabaseScopeDiagnostics
{
    public function __construct(
        public DatabaseScopeId $scope,
        public DatabaseScopeState $state,
        public int $managedEntities,
        public int $pendingChanges,
        public int $activeTransactions,
        public int $connectionLeases,
        public int $activeCursors,
        public int $activeStreams,
        public int $resourcePermits,
    ) {}
}
```

---

# 204. Production cost

Diagnostics costosos deberán poder:

```text
disable
sample
bound
```

---

# 205. Scope policies

Configuración conceptual:

```php
DatabaseRuntimeConfig::make()
    ->scope(
        cleanup: ScopeCleanupMode::STRICT,
        leakDetection: true,
        ownershipValidation: true,
    );
```

---

# 206. Production defaults

VoltStack deberá mantener seguridad aunque ciertas verificaciones diagnósticas estén desactivadas.

---

# 207. Debug validation ≠ correctness

Correctness no dependerá exclusivamente de asserts de development.

---

# 208. FrankenPHP flow

```text
FrankenPHP Worker
      │
      ▼
Request received
      │
      ▼
FrankenPhpRuntimeAdapter
      │
      ▼
DatabaseScopeManager.beginExecution()
      │
      ▼
Scope ACTIVE
      │
      ▼
VoltStack Application
      │
      ▼
Response / Exception
      │
      ▼
DatabaseScopeManager.finishExecution()
      │
      ▼
Scope CLOSED
      │
      ▼
Worker ready for next request
```

---

# 209. RoadRunner flow

El mismo lifecycle conceptual:

```text
RoadRunner
    ↓
Runtime Adapter
    ↓
DatabaseExecutionScope
```

sin cambiar ORM.

---

# 210. OpenSwoole flow

```text
Worker
├── Coroutine A
│   └── Scope A
└── Coroutine B
    └── Scope B
```

requiere RuntimeLocalStorage concurrent-aware.

---

# 211. Traditional PHP

También podrá existir:

```text
Traditional PHP Request
        ↓
DatabaseExecutionScope
        ↓
process eventually terminates
```

Aunque el proceso vaya a morir, VoltStack seguirá finalizando explícitamente el scope.

---

# 212. Por qué hacerlo también en PHP tradicional

Porque mantiene:

```text
same semantics
same tests
same lifecycle
same cleanup rules
```

en todos los runtimes.

---

# 213. Scope manager

Coordinador:

```text
DatabaseScopeManager
```

---

# 214. Contract

```php
interface DatabaseScopeManager
{
    public function begin(
        ExecutionDescriptor $descriptor
    ): DatabaseExecutionScope;

    public function finish(
        DatabaseExecutionScope $scope,
        ExecutionOutcome $outcome
    ): ScopeFinalizationResult;
}
```

---

# 215. Manager responsibilities

```text
create
activate
bind
finalize
unbind
report
```

---

# 216. Manager non-responsibilities

No implementará:

```text
query execution
ORM flush
transaction protocol
SQL
pooling
```

---

# 217. Scoped service registry

Podrá existir:

```text
DatabaseScopedServiceRegistry
```

para servicios Database internos.

---

# 218. Scoped service identity

Dentro del mismo scope:

```text
resolve(EntityManager)
===
resolve(EntityManager)
```

---

# 219. Across scopes

```text
resolveA(EntityManager)
!==
resolveB(EntityManager)
```

---

# 220. Immutable service exception

Para un servicio worker/application:

```text
resolveA(TypeRegistry)
===
resolveB(TypeRegistry)
```

puede ser correcto.

---

# 221. Scope closure

Cuando el scope cierra, sus servicios deberán dejar de ser resolubles.

---

# 222. Stale service reference

Si código conserva referencia a:

```text
EntityManager A
```

y la usa después:

```text
EntityManagerClosedException
```

o error equivalente.

---

# 223. Scope generation validation

Servicios críticos pueden verificar:

```text
current generation
=
creation generation
```

---

# 224. Performance tradeoff

No todos los métodos deberán hacer una búsqueda global del scope.

Objetos scoped ya creados pueden contener:

```text
ScopeToken
```

barato.

---

# 225. ScopeToken ≠ scope object

Esto evita referencias innecesarias al grafo completo.

---

# 226. Weak references

Para tooling/debugging podrán utilizarse weak references cuando ayuden a detectar objetos retenidos sin impedir GC.

---

# 227. Leak detector

Podrá registrar weak references a servicios scoped.

Después de finalization:

```text
GC
 ↓
check remaining references
```

en testing/debug.

---

# 228. GC nondeterminism

No se dependerá de GC para correctness.

---

# 229. Resource release

Recursos externos deberán cerrarse determinísticamente.

---

# 230. Memory release

Las referencias internas deberán eliminarse para permitir GC.

---

# 231. Request Scope Directory

Propuesta:

```text
src/Quantum/Database/Runtime/Scope/
│
├── Contract/
│   ├── DatabaseExecutionScope.php
│   ├── DatabaseScopeFactory.php
│   ├── DatabaseScopeResolver.php
│   ├── DatabaseScopeManager.php
│   ├── ScopeCleanupHook.php
│   └── ScopedDatabaseResource.php
│
├── Model/
│   ├── DatabaseScopeId.php
│   ├── DatabaseScopeState.php
│   ├── ScopeGeneration.php
│   ├── ScopeToken.php
│   ├── DatabaseOperationId.php
│   ├── ExecutionDescriptor.php
│   └── ExecutionOutcome.php
│
├── Lifecycle/
│   ├── DefaultDatabaseExecutionScope.php
│   ├── DefaultDatabaseScopeFactory.php
│   ├── DefaultDatabaseScopeManager.php
│   ├── ScopeActivator.php
│   ├── ScopeFinalizer.php
│   └── ScopeCleanupGraph.php
│
├── Resource/
│   ├── DatabaseScopeResourceRegistry.php
│   ├── ScopedResourceEntry.php
│   ├── ResourceOwnership.php
│   └── ResourceConcurrency.php
│
├── Service/
│   ├── DatabaseScopedServiceRegistry.php
│   └── ScopedLazy.php
│
├── Diagnostics/
│   ├── DatabaseScopeDiagnostics.php
│   ├── DatabaseScopeLeakDetector.php
│   ├── ScopeCleanupReport.php
│   └── ScopeDiagnosticCollector.php
│
├── Telemetry/
│   └── DatabaseScopeTelemetry.php
│
└── Exception/
    ├── DatabaseScopeException.php
    ├── DatabaseScopeNotActiveException.php
    ├── DatabaseScopeClosedException.php
    ├── DatabaseScopeTaintedException.php
    ├── CrossScopeResourceAccessException.php
    ├── StaleDatabaseScopeException.php
    ├── DatabaseScopeLeakException.php
    └── DatabaseScopeCleanupException.php
```

---

# 232. Integration map

```text
                    DatabaseExecutionScope
                            │
       ┌────────────────────┼────────────────────┐
       │                    │                    │
       ▼                    ▼                    ▼
 DatabaseContext       EntityManager      ResourceRegistry
       │                    │                    │
 ┌─────┼─────┐        ┌─────┴─────┐       ┌─────┼─────┐
 ▼     ▼     ▼        ▼           ▼       ▼     ▼     ▼
Tenant Shard Routing UoW      IdentityMap Conn Cursor Permit
       │                    │                    │
       └────────────────────┼────────────────────┘
                            ▼
                     TransactionContext
                            │
                            ▼
                      ConnectionLease
```

---

# 233. Architectural Invariants

## DB-SCOPE-001

Toda ejecución Database tendrá un scope explícito.

## DB-SCOPE-002

HTTP Request será una clase de Execution.

## DB-SCOPE-003

Queue Job será una clase de Execution.

## DB-SCOPE-004

CLI Command podrá tener Execution Scope.

## DB-SCOPE-005

Test podrá tener Execution Scope.

## DB-SCOPE-006

Scope tendrá identidad única.

## DB-SCOPE-007

Scope tendrá lifecycle explícito.

## DB-SCOPE-008

`CLOSED → ACTIVE` estará prohibido.

## DB-SCOPE-009

`TAINTED → ACTIVE` estará prohibido.

## DB-SCOPE-010

Scope cerrado no será reciclado como nuevo scope.

## DB-SCOPE-011

Scope creation no abrirá conexión obligatoriamente.

## DB-SCOPE-012

Scope creation no iniciará transaction.

## DB-SCOPE-013

Scope creation no inicializará ORM completo si no es necesario.

## DB-SCOPE-014

Scoped services podrán ser lazy.

## DB-SCOPE-015

Toda ejecución terminará con finalization cuando sea posible.

## DB-SCOPE-016

Application exception no omitirá finalization.

## DB-SCOPE-017

No existirá current scope global mutable.

## DB-SCOPE-018

Current scope será execution-local.

## DB-SCOPE-019

Concurrent executions tendrán scopes distintos.

## DB-SCOPE-020

ScopeResolver fallará fuera de scope.

## DB-SCOPE-021

ScopeResolver no creará global fallback scope.

## DB-SCOPE-022

Todo recurso contextual tendrá owner.

## DB-SCOPE-023

Ownership podrá validarse mediante ScopeToken.

## DB-SCOPE-024

Cross-scope resource access será error.

## DB-SCOPE-025

EntityManager será execution-scoped por default.

## DB-SCOPE-026

EntityManager no será estático.

## DB-SCOPE-027

Model API resolverá EntityManager scoped.

## DB-SCOPE-028

IdentityMap será execution-scoped.

## DB-SCOPE-029

IdentityMap no cruzará requests.

## DB-SCOPE-030

Misma entidad en scopes distintos no implica misma instancia.

## DB-SCOPE-031

UnitOfWork será execution-scoped.

## DB-SCOPE-032

Pending ChangeSet no cruzará scopes.

## DB-SCOPE-033

Scope end no implicará auto-flush.

## DB-SCOPE-034

Scope end no implicará auto-commit.

## DB-SCOPE-035

TransactionContext pertenecerá al scope.

## DB-SCOPE-036

Transaction no migrará entre scopes.

## DB-SCOPE-037

Active transaction será resuelta antes de safe close.

## DB-SCOPE-038

UNKNOWN transaction outcome taintará contexto apropiado.

## DB-SCOPE-039

Nested transaction seguirá perteneciendo al execution scope.

## DB-SCOPE-040

ConnectionLease tendrá scope owner.

## DB-SCOPE-041

PhysicalConnection será distinta de ConnectionLease.

## DB-SCOPE-042

Lease terminará al finalizar scope.

## DB-SCOPE-043

Physical connection podrá sobrevivir mediante pool si es segura.

## DB-SCOPE-044

Sticky routing será scoped.

## DB-SCOPE-045

Replica routing context será scoped.

## DB-SCOPE-046

TenantContext será scoped.

## DB-SCOPE-047

Tenant no se almacenará en worker-global mutable state.

## DB-SCOPE-048

Tenant switching será explícito.

## DB-SCOPE-049

Cross-tenant operation tendrá frontera explícita.

## DB-SCOPE-050

ShardContext será scoped.

## DB-SCOPE-051

Shard affinity transaccional será preservada.

## DB-SCOPE-052

DatabaseContext será propiedad del scope.

## DB-SCOPE-053

DatabaseContext será distinto del scope.

## DB-SCOPE-054

Operation Scope podrá existir dentro de Execution Scope.

## DB-SCOPE-055

Operation Scope no implicará nuevo EntityManager.

## DB-SCOPE-056

Child deadline no ampliará parent deadline por default.

## DB-SCOPE-057

Cancellation del parent se propagará a children.

## DB-SCOPE-058

Child cancellation no cancelará necesariamente parent.

## DB-SCOPE-059

Cancellation no implicará rollback demostrado.

## DB-SCOPE-060

Resource Registry será scoped.

## DB-SCOPE-061

Resource Registry no sustituirá GC.

## DB-SCOPE-062

Recursos se liberarán tan pronto como sea posible.

## DB-SCOPE-063

Finalization será safety boundary.

## DB-SCOPE-064

FINALIZING no aceptará nuevas operaciones.

## DB-SCOPE-065

Finalization respetará resource dependencies.

## DB-SCOPE-066

Cursor será cerrado antes de liberar conexión requerida.

## DB-SCOPE-067

Transaction será resuelta antes de liberar su conexión.

## DB-SCOPE-068

Finalization será idempotente cuando sea posible.

## DB-SCOPE-069

Finalization producirá resultado explícito.

## DB-SCOPE-070

Cleanup status será distinto de execution outcome.

## DB-SCOPE-071

Application success no implicará clean scope.

## DB-SCOPE-072

Application failure no implicará dirty scope.

## DB-SCOPE-073

Scope podrá solicitar worker recycle.

## DB-SCOPE-074

Scope close no será worker shutdown.

## DB-SCOPE-075

EntityManager scoped no se poolará por default.

## DB-SCOPE-076

Immutable metadata podrá sobrevivir scopes.

## DB-SCOPE-077

Service lifetime será explícito.

## DB-SCOPE-078

Worker service no retendrá execution service.

## DB-SCOPE-079

Captive scoped dependency será inválida.

## DB-SCOPE-080

Resolver shared no cacheará scoped instance.

## DB-SCOPE-081

Facade podrá ser global en sintaxis, no en estado mutable.

## DB-SCOPE-082

Async operation no capturará scope expirado.

## DB-SCOPE-083

Async handoff creará nuevo execution scope.

## DB-SCOPE-084

Fiber podrá heredar contexto sólo dentro del lifetime válido.

## DB-SCOPE-085

Fiber no utilizará scope cerrado.

## DB-SCOPE-086

Same scope no implica resource concurrency safety.

## DB-SCOPE-087

UNKNOWN concurrency no equivaldrá a concurrent-safe.

## DB-SCOPE-088

Scope fork no clonará recursos vivos.

## DB-SCOPE-089

Resource ownership podrá incluir operation ID.

## DB-SCOPE-090

One request no equivaldrá a one transaction.

## DB-SCOPE-091

Whole-request transaction será explícita.

## DB-SCOPE-092

Transaction retry podrá tener attempt context propio.

## DB-SCOPE-093

Rollback no equivaldrá a object graph rewind.

## DB-SCOPE-094

Request-local cache terminará con scope.

## DB-SCOPE-095

Shared cache deberá usar keys correctamente aisladas.

## DB-SCOPE-096

Event context será scoped.

## DB-SCOPE-097

Singleton listener no almacenará request state.

## DB-SCOPE-098

Telemetry correlation será scoped.

## DB-SCOPE-099

Scope ID no se usará indiscriminadamente como metric label.

## DB-SCOPE-100

Scope isolation será frontera de seguridad.

## DB-SCOPE-101

Tenant state no cruzará scopes.

## DB-SCOPE-102

Authorization state no cruzará scopes.

## DB-SCOPE-103

Query parameters no quedarán en worker-shared services.

## DB-SCOPE-104

Sensitive references serán liberadas.

## DB-SCOPE-105

DatabaseExecutionScope no será serializable.

## DB-SCOPE-106

Scope no será almacenado en queues.

## DB-SCOPE-107

Scope no será almacenado en cache.

## DB-SCOPE-108

Scope no será almacenado en session.

## DB-SCOPE-109

Execution descriptor no contendrá live connection.

## DB-SCOPE-110

Execution descriptor no contendrá EntityManager.

## DB-SCOPE-111

Execution descriptor no contendrá TransactionContext.

## DB-SCOPE-112

Execution descriptor no contendrá ResultCursor.

## DB-SCOPE-113

Scope isolation tendrá tests dedicados.

## DB-SCOPE-114

Exception cleanup tendrá tests dedicados.

## DB-SCOPE-115

Transaction leakage tendrá tests dedicados.

## DB-SCOPE-116

IdentityMap leakage tendrá tests dedicados.

## DB-SCOPE-117

UnitOfWork leakage tendrá tests dedicados.

## DB-SCOPE-118

Tenant leakage tendrá tests dedicados.

## DB-SCOPE-119

Routing leakage tendrá tests dedicados.

## DB-SCOPE-120

Connection leakage tendrá tests dedicados.

## DB-SCOPE-121

Cursor leakage tendrá tests dedicados.

## DB-SCOPE-122

Concurrent isolation tendrá tests dedicados.

## DB-SCOPE-123

Long-worker stability tendrá tests dedicados.

## DB-SCOPE-124

Scope creation tendrá overhead acotado.

## DB-SCOPE-125

Zero-query request evitará inicialización innecesaria.

## DB-SCOPE-126

Cleanup se limitará preferentemente a componentes activados.

## DB-SCOPE-127

Cleanup dependencies no dependerán únicamente de prioridad arbitraria.

## DB-SCOPE-128

Cleanup errors podrán agregarse.

## DB-SCOPE-129

Cleanup error no ocultará automáticamente primary failure.

## DB-SCOPE-130

Debug diagnostics serán bounded.

## DB-SCOPE-131

Correctness no dependerá de debug assertions.

## DB-SCOPE-132

FrankenPHP utilizará el mismo scope abstraction.

## DB-SCOPE-133

RoadRunner utilizará el mismo scope abstraction.

## DB-SCOPE-134

OpenSwoole utilizará el mismo scope abstraction.

## DB-SCOPE-135

Traditional PHP utilizará el mismo lifecycle conceptual.

## DB-SCOPE-136

DatabaseScopeManager coordinará lifecycle.

## DB-SCOPE-137

DatabaseScopeManager no implementará ORM.

## DB-SCOPE-138

DatabaseScopeManager no implementará SQL execution.

## DB-SCOPE-139

DatabaseScopeManager no implementará connection pooling.

## DB-SCOPE-140

Dentro de un scope, scoped service resolution será estable.

## DB-SCOPE-141

Entre scopes, scoped service identity será distinta.

## DB-SCOPE-142

Application/worker service podrá compartirse cuando sea seguro.

## DB-SCOPE-143

Servicios scoped dejarán de ser utilizables después del cierre.

## DB-SCOPE-144

Stale service reference deberá fallar.

## DB-SCOPE-145

ScopeToken no requerirá retener grafo completo.

## DB-SCOPE-146

Weak references podrán utilizarse sólo como tooling.

## DB-SCOPE-147

GC no será mecanismo de resource correctness.

## DB-SCOPE-148

External resources se cerrarán determinísticamente.

## DB-SCOPE-149

Scope state será observable.

## DB-SCOPE-150

Scope cleanup state será observable.

## DB-SCOPE-151

Scope ownership violations serán observables.

## DB-SCOPE-152

Scope finalization ocurrirá antes de reutilización segura del worker.

## DB-SCOPE-153

Request Scope será una frontera lógica, no simplemente una variable de container.

## DB-SCOPE-154

Scope isolation aplicará también a Jobs.

## DB-SCOPE-155

Scope isolation aplicará también a CLI.

## DB-SCOPE-156

Scope isolation aplicará también a tests.

## DB-SCOPE-157

Scope isolation aplicará también a background operations.

## DB-SCOPE-158

No habrá `$currentEntityManager` estático.

## DB-SCOPE-159

No habrá `$currentTransaction` estático.

## DB-SCOPE-160

No habrá `$currentTenant` estático.

## DB-SCOPE-161

No habrá `$currentShard` estático.

## DB-SCOPE-162

No habrá `$currentConnection` estático contextual.

## DB-SCOPE-163

No habrá `$currentScope` global mutable.

## DB-SCOPE-164

Persistent runtime no modificará las reglas de aislamiento.

## DB-SCOPE-165

Scope finalization será parte del modelo de correctness de Database.

---

# 234. Modelo formal del Scope

Sea:

```text
S
```

un `DatabaseExecutionScope`.

Se define:

```text
S =
(
 id,
 generation,
 state,
 context,
 resources,
 services,
 cancellation
)
```

---

# 235. Propiedad de aislamiento

Para:

```text
S₁ ≠ S₂
```

deberá cumplirse:

```text
MutableContext(S₁)
∩
MutableContext(S₂)
=
∅
```

---

# 236. Propiedad de ownership

Para cada recurso contextual:

```text
r
```

existe exactamente un owner efectivo:

```text
Owner(r) = S
```

salvo recursos explícitamente compartidos.

---

# 237. Propiedad de uso

Una operación sobre `r` es válida si:

```text
CurrentScope = Owner(r)
```

o si el contrato de `r` declara explícitamente shared access.

---

# 238. Propiedad de cierre

Si:

```text
State(S) = CLOSED
```

entonces:

```text
NewDatabaseOperation(S)
=
INVALID
```

---

# 239. Propiedad de finalización

Para un scope limpio:

```text
Finalize(S)
→
NoActiveScopedResources(S)
```

---

# 240. Propiedad de incertidumbre

Si la limpieza de un recurso crítico no puede demostrarse:

```text
Clean(r) = UNKNOWN
```

entonces:

```text
Clean(S)
≠
TRUE
```

---

# 241. Scope lifecycle completo

```text
                    EXECUTION RECEIVED
                            │
                            ▼
                     CREATE SCOPE
                            │
                            ▼
                         CREATED
                            │
                            ▼
                         ACTIVE
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
         ORM              QUERY          TRANSACTION
          │                 │                 │
          ▼                 ▼                 ▼
     IdentityMap       ConnectionLease     TxContext
          │                 │                 │
          └─────────────────┼─────────────────┘
                            │
                            ▼
                     EXECUTION END
                            │
                            ▼
                       FINALIZING
                            │
                 ┌──────────┼──────────┐
                 ▼          ▼          ▼
             CLOSE I/O   RESOLVE TX   CLEAR ORM
                 │          │          │
                 └──────────┼──────────┘
                            ▼
                      VERIFY CLEAN
                            │
                   ┌────────┴────────┐
                   ▼                 ▼
                 CLEAN            UNCERTAIN
                   │                 │
                   ▼                 ▼
                 CLOSED           TAINTED
                   │                 │
                   ▼                 ▼
             REUSE WORKER       RECYCLE/ISOLATE
```

---

# 242. Relación con Persistent Runtime Architecture

El documento 251 definió:

```text
Persistent Worker
    ↓
multiple executions
```

Este documento añade la frontera:

```text
Persistent Worker
│
├── DatabaseExecutionScope A
│
├── DatabaseExecutionScope B
│
├── DatabaseExecutionScope C
│
└── DatabaseExecutionScope D
```

Por tanto:

```text
Worker Reuse
```

es seguro únicamente porque:

```text
Execution State
```

no se reutiliza accidentalmente.

---

# 243. Regla final

> **El Request Scope System será la frontera fundamental de aislamiento mutable de VoltStack Database. Cada request, job, comando o ejecución recibirá un contexto Database propio, con ownership explícito de EntityManager, IdentityMap, UnitOfWork, transacciones, leases, tenant, shard y demás recursos contextuales. El final de la ejecución deberá cerrar esa frontera antes de permitir la reutilización segura del worker.**

En forma resumida:

```text
Persistent Worker
       │
       ├── Shared Safe Infrastructure
       │
       ├── Scope A
       │     ├── ORM A
       │     ├── Context A
       │     └── Resources A
       │
       ├── Scope B
       │     ├── ORM B
       │     ├── Context B
       │     └── Resources B
       │
       └── Scope C
             ├── ORM C
             ├── Context C
             └── Resources C
```

con la garantía:

```text
A ≠ B ≠ C
```

y:

```text
MutableState(A)
∩
MutableState(B)
∩
MutableState(C)
=
∅
```

salvo infraestructura explícitamente compartida.

---

# 244. Estado del Bloque 25

```text
BLOCK 25 — PERSISTENT RUNTIME

✓ 251_DATABASE_PERSISTENT_RUNTIME_ARCHITECTURE.md
✓ 252_DATABASE_REQUEST_SCOPE_SYSTEM.md
│
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

# 245. Siguiente documento

```text
253_DATABASE_DATABASE_CONTEXT_SYSTEM.md
```

El siguiente documento definirá el objeto contextual que vive dentro del `DatabaseExecutionScope`:

```text
DatabaseExecutionScope
        │
        ▼
   DatabaseContext
        │
        ├── Persistence Domain
        ├── Connection Intent
        ├── Tenant Context
        ├── Shard Context
        ├── Read/Write Context
        ├── Consistency Context
        ├── Transaction Context
        ├── Security Context
        ├── Telemetry Context
        ├── Resource Context
        ├── Cancellation Context
        └── Runtime Context
```

y establecerá cómo toda operación de `VoltStack/Quantum/Database` obtiene una visión **explícita, tipada, inmutable donde sea posible y scope-safe** del contexto bajo el cual debe ejecutarse.