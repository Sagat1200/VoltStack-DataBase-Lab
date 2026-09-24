# 209_DATABASE_EVENT_ARCHITECTURE.md

# VoltStack Quantum Database
## Database Event Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 209 — Database Event Architecture  
**Bloque:** 20 — Events  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `208_DATABASE_LARGE_DATASET_PROCESSING_SYSTEM.md`  
**Siguiente documento:** `210_DATABASE_QUERY_EVENT_SYSTEM.md`

---

# 1. Propósito

`Database Event Architecture` define el modelo transversal mediante el cual los componentes de `VoltStack/Quantum/Database` podrán comunicar hechos relevantes de su ciclo de vida sin introducir acoplamiento directo entre:

```text
Query Engine
Connection System
Transaction System
ORM
Persistence Engine
Schema
Migration
Cache
Telemetry
Security
Runtime
Extensions
```

Ejemplos de hechos observables:

```text
query planned
query started
query completed
query failed

connection acquired
connection released
connection failed

transaction started
transaction committed
transaction rolled back
transaction outcome unknown

entity persisted
entity removed

flush started
flush completed
flush failed
```

La regla central será:

> **Un evento de Database representa un hecho, transición o boundary observable del subsistema; no constituye por sí mismo una autorización para modificar arbitrariamente la operación que observa, ni reemplaza el estado canónico del componente que originó el evento.**

---

# 2. Objetivos

La arquitectura deberá proporcionar:

```text
typed events
stable contracts
explicit lifecycle boundaries
deterministic dispatch
event categories
event phases
transaction awareness
after-commit semantics
listener isolation
failure policies
event ordering
payload governance
security
bounded telemetry integration
extension points
runtime isolation
testing support
```

sin convertir el Event System en un mecanismo de control implícito del Database Engine.

---

# 3. Distinciones fundamentales

VoltStack deberá mantener:

```text
Database Event
≠
Domain Event
≠
Telemetry Event
≠
ORM Lifecycle Event
≠
Persistence Event
≠
Transaction Outcome
≠
Hook
≠
Interceptor
≠
Command
≠
Query
≠
State
```

Estas distinciones son esenciales.

---

# 4. Database Event ≠ Domain Event

Un evento:

```text
QueryExecuted
```

describe infraestructura.

Un evento:

```text
InvoicePaid
```

describe el dominio de negocio.

Por tanto:

```text
Database Event
    ↓
Infrastructure semantics

Domain Event
    ↓
Business semantics
```

El Database Event System no deberá convertirse en el Domain Event Bus de la aplicación.

---

# 5. Database Event ≠ Telemetry Event

Un Database Event puede alimentar Telemetry.

Pero:

```text
Database Event
≠
Telemetry Record
```

Ejemplo:

```text
QueryCompleted
      │
      ├── Listener
      ├── Cache integration
      └── Telemetry adapter
             ↓
          Span/Event
```

Telemetry podrá muestrear, agregar o descartar observaciones.

Eso no deberá cambiar el hecho original.

---

# 6. Database Event ≠ ORM Lifecycle Event

Un ORM Lifecycle Event es una categoría especializada.

Ejemplo:

```text
EntityPostLoad
EntityPrePersist
EntityPostPersist
```

No todos los eventos Database pertenecen al ORM.

---

# 7. Database Event ≠ Persistence Event

Ejemplo:

```text
FlushStarted
PersistencePlanCreated
EntityInsertScheduled
```

pertenece al pipeline de persistencia.

Mientras:

```text
ConnectionAcquired
```

no.

---

# 8. Database Event ≠ Transaction Outcome

Esto es crítico.

Si se emite:

```text
TransactionCommitted
```

el evento describe un outcome ya conocido.

El evento no produce el commit.

Por tanto:

```text
TransactionCommitted Event
≠
Transaction Commit
```

---

# 9. Event listener failure ≠ transaction failure

Si:

```text
COMMIT succeeds
↓
TransactionCommitted dispatched
↓
listener throws
```

la base sigue committed.

Nunca deberá reportarse:

```text
transaction rolled back
```

como consecuencia ficticia.

---

# 10. AfterCommit failure

Siempre:

```text
Commit Success
+
AfterCommit Listener Failure
=
Database Commit Still Successful
```

El listener failure deberá modelarse separadamente.

---

# 11. Database Event ≠ Hook

Un evento normalmente observa.

Un hook puede intervenir.

```text
Event
=
notification of fact/phase

Hook
=
controlled extension point
```

No deberán confundirse.

---

# 12. Database Event ≠ Interceptor

Un interceptor puede envolver una operación:

```text
before
↓
operation
↓
after
```

y potencialmente modificar comportamiento.

Los eventos no deberán asumir esas capacidades.

---

# 13. Database Event ≠ Command

Un evento dice:

```text
QueryCompleted
```

Un command dice:

```text
ExecuteQuery
```

Son direcciones semánticas distintas.

---

# 14. Principio de observabilidad segura

Por default:

```text
Events observe.
They do not redefine.
```

Las modificaciones de comportamiento deberán ocurrir mediante contracts especializados:

```text
Policy
Interceptor
Resolver
Planner extension
Compiler extension
Hook
```

no mediante listeners genéricos ocultos.

---

# 15. Arquitectura general

```text
Database Component
       │
       ▼
Event Factory
       │
       ▼
DatabaseEvent
       │
       ▼
DatabaseEventDispatcher
       │
       ├── Local Database Listeners
       ├── Framework Event Bridge
       ├── Telemetry Bridge
       ├── Extension Listeners
       └── Testing Observer
```

---

# 16. Arquitectura desacoplada

`Quantum/Database` no deberá depender directamente de una implementación concreta del Event System global.

En su lugar:

```text
Quantum/Database
      │
      ▼
DatabaseEventDispatcher contract
      │
      ▼
EventSystem Adapter
      │
      ▼
VoltStack Event System
```

---

# 17. Null dispatcher

Database deberá funcionar incluso sin Event System instalado.

```php
final class NullDatabaseEventDispatcher implements DatabaseEventDispatcher
{
    public function dispatch(DatabaseEvent $event): EventDispatchResult
    {
        return EventDispatchResult::ignored();
    }
}
```

Por tanto:

```text
EventSystem
=
optional integration
```

---

# 18. Core contract

```php
interface DatabaseEvent
{
    public function eventId(): DatabaseEventId;

    public function occurredAt(): Instant;

    public function context(): DatabaseEventContext;
}
```

---

# 19. Event immutability

Los eventos deberán ser conceptualmente:

```text
immutable
```

Ejemplo:

```php
final readonly class QueryCompleted implements DatabaseEvent
{
    public function __construct(
        private DatabaseEventId $eventId,
        private Instant $occurredAt,
        private DatabaseEventContext $context,
        public QueryExecutionSummary $execution,
    ) {}
}
```

---

# 20. Event ID

Cada instancia podrá recibir:

```text
DatabaseEventId
```

para:

```text
diagnostics
correlation
testing
logging
distributed bridges
```

---

# 21. Event ID ≠ Processing ID

Deben distinguirse:

```text
EventId
OperationId
QueryId
TransactionId
ConnectionId
RequestId
TraceId
```

---

# 22. Event time

El evento podrá contener:

```text
occurredAt
```

obtenido mediante el Clock abstraction de VoltStack.

No deberá utilizar directamente:

```php
new DateTimeImmutable();
```

si existe Clock contextual.

---

# 23. Event context

Modelo conceptual:

```php
final readonly class DatabaseEventContext
{
    public function __construct(
        public ?OperationId $operationId,
        public ?RequestId $requestId,
        public ?TraceId $traceId,
        public ?ConnectionIdentity $connection,
        public ?TransactionId $transactionId,
        public ?PersistenceDomain $domain,
        public ?TenantContextReference $tenant,
        public ?ShardId $shard,
        public DatabaseEventMetadata $metadata,
    ) {}
}
```

---

# 24. Context ≠ global mutable state

Nunca:

```php
DatabaseEvents::$currentContext;
```

---

# 25. Context propagation

El contexto deberá propagarse explícitamente:

```text
Request
↓
DatabaseContext
↓
Query Context
↓
Execution Context
↓
Database Event Context
```

---

# 26. Context snapshots

El evento deberá capturar la información necesaria en el momento de emisión.

No deberá depender de objetos mutables que posteriormente cambien.

---

# 27. Event categories

```php
enum DatabaseEventCategory
{
    case QUERY;
    case CONNECTION;
    case TRANSACTION;
    case ENTITY_LIFECYCLE;
    case PERSISTENCE;
    case SCHEMA;
    case MIGRATION;
    case CACHE;
    case LARGE_DATASET;
    case SECURITY;
    case EXTENSION;
}
```

---

# 28. Categoría ≠ clase concreta

La categoría sirve para:

```text
routing
filtering
diagnostics
policy
```

pero el contrato real deberá permanecer tipado por clase.

---

# 29. Event phases

Algunos procesos tendrán fases:

```php
enum DatabaseEventPhase
{
    case BEFORE;
    case STARTED;
    case COMPLETED;
    case FAILED;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 30. No universal phase requirement

No todos los eventos necesitan todas las fases.

Ejemplo:

```text
ConnectionAcquired
```

puede ser suficiente como hecho.

No deberá inventarse:

```text
BeforeConnectionAcquired
```

si no tiene semántica útil.

---

# 31. Facts vs lifecycle notifications

Distinguiremos:

```text
Fact Event
Lifecycle Event
```

Ejemplo de Fact:

```text
TransactionCommitted
```

Ejemplo de Lifecycle:

```text
QueryExecutionStarted
```

---

# 32. Before events

Los eventos `BEFORE` deberán utilizarse con cautela.

Por default serán:

```text
notification
```

no:

```text
mutation hook
```

---

# 33. Pre-operation mutation

Si una extensión necesita modificar una query antes de ejecución, deberá utilizar:

```text
Query Extension
Optimizer Rule
Planner Rule
Interceptor
```

no modificar `QueryExecutingEvent`.

---

# 34. Event payload immutability

No:

```php
$event->query->where(...);
```

si el evento representa una query ya planificada.

---

# 35. Event envelope

Podrá existir:

```php
final readonly class DatabaseEventEnvelope
{
    public function __construct(
        public DatabaseEvent $event,
        public DatabaseEventDescriptor $descriptor,
        public DatabaseEventContext $context,
    ) {}
}
```

---

# 36. Descriptor

```php
final readonly class DatabaseEventDescriptor
{
    public function __construct(
        public DatabaseEventName $name,
        public DatabaseEventCategory $category,
        public ?DatabaseEventPhase $phase,
        public int $schemaVersion,
    ) {}
}
```

---

# 37. Event name

Deberá ser estable.

Ejemplos:

```text
database.query.started
database.query.completed
database.query.failed

database.connection.acquired
database.connection.released

database.transaction.started
database.transaction.committed
database.transaction.rolled_back
database.transaction.unknown
```

---

# 38. PHP class ≠ external event name

Esto permite:

```text
internal refactoring
```

sin romper necesariamente bridges externos.

---

# 39. Event schema version

Eventos que crucen boundaries externos podrán tener:

```text
schemaVersion
```

independiente de la versión del framework.

---

# 40. Internal events

Eventos puramente internos no necesitarán serialización externa.

---

# 41. Externalizable events

Un evento podrá declarar:

```php
interface ExternalizableDatabaseEvent extends DatabaseEvent
{
    public function schemaVersion(): int;
}
```

---

# 42. Internal ≠ serializable

No deberá asumirse:

```text
DatabaseEvent
=
Serializable Message
```

---

# 43. No arbitrary serialization

No deberán serializarse directamente:

```text
PDO
Connection
EntityManager
TransactionContext
ResultCursor
Generator
Closure
Entity graph
```

---

# 44. Event payload design

Los payloads deberán preferir:

```text
IDs
immutable summaries
stable type IDs
bounded metadata
sanitized diagnostics
```

---

# 45. Event payload ≠ state dump

Un evento no deberá capturar todo:

```text
DatabaseContext
EntityManager
IdentityMap
UnitOfWork
```

solo porque esté disponible.

---

# 46. Payload size governance

Cada familia de eventos deberá definir límites razonables.

Especialmente:

```text
Query events
Bulk events
Large Dataset events
Migration events
```

---

# 47. SQL payload

Raw SQL será sensible.

Por default los eventos públicos deberán preferir:

```text
query fingerprint
operation type
duration
row count
parameter count
platform
```

---

# 48. Parameter values

No deberán incluirse automáticamente.

Incorrecto:

```text
password = "..."
token = "..."
email = "..."
```

---

# 49. Sanitized diagnostics

Podrá existir:

```php
interface DatabaseEventSanitizer
{
    public function sanitize(
        DatabaseEvent $event,
        DatabaseEventSecurityContext $context,
    ): DatabaseEvent;
}
```

---

# 50. Security principle

> **La observabilidad de Database nunca deberá convertirse en una vía lateral para exponer credenciales, parámetros sensibles, PII o información de tenants ajenos.**

---

# 51. Event dispatcher

Contrato:

```php
interface DatabaseEventDispatcher
{
    public function dispatch(
        DatabaseEvent $event,
    ): DatabaseEventDispatchResult;
}
```

---

# 52. Dispatch result

```php
final readonly class DatabaseEventDispatchResult
{
    public function __construct(
        public int $matchedListeners,
        public int $successfulListeners,
        public int $failedListeners,
        public DatabaseEventDispatchStatus $status,
    ) {}
}
```

---

# 53. Dispatch status

```php
enum DatabaseEventDispatchStatus
{
    case DISPATCHED;
    case PARTIALLY_DISPATCHED;
    case IGNORED;
    case FAILED;
}
```

---

# 54. Event delivery ≠ database outcome

Siempre:

```text
EventDispatchStatus
≠
DatabaseOperationStatus
```

---

# 55. Dispatcher responsibilities

El dispatcher podrá:

```text
resolve listeners
apply event filters
invoke listeners
apply listener failure policy
produce dispatch diagnostics
```

No deberá:

```text
execute query
commit transaction
persist entity
compile SQL
```

---

# 56. Listener contract

```php
interface DatabaseEventListener
{
    public function handle(DatabaseEvent $event): void;
}
```

---

# 57. Typed listeners

Preferencia:

```php
final class QueryPerformanceListener
{
    public function __invoke(QueryCompleted $event): void
    {
        // ...
    }
}
```

---

# 58. Listener registration

Ejemplo:

```php
DatabaseEvents::listen(
    QueryCompleted::class,
    QueryPerformanceListener::class,
);
```

La facade será convenience API.

El registry real no será static mutable global.

---

# 59. Listener registry

```php
interface DatabaseEventListenerRegistry
{
    public function listenersFor(
        DatabaseEventDescriptor $event,
    ): iterable;
}
```

---

# 60. Registry lifecycle

En producción podrá compilarse durante bootstrap.

```text
configuration
↓
listener discovery
↓
validation
↓
compiled registry
↓
freeze
```

---

# 61. Frozen registry

En persistent runtimes:

```text
immutable listener registry
```

podrá compartirse entre requests.

---

# 62. Runtime listener registration

Podrá permitirse únicamente en scopes controlados:

```text
testing
development
operation-local listeners
```

---

# 63. No global listener leakage

Registrar un listener durante Request A no deberá afectar Request B accidentalmente.

---

# 64. Listener priority

Podrá existir:

```php
#[DatabaseListener(priority: 100)]
```

---

# 65. Priority ordering

Orden conceptual:

```text
higher priority
↓
lower priority
```

pero deberá documentarse y permanecer determinista.

---

# 66. Equal priority

Se resolverá mediante orden estable de registro/compilación.

No dependerá de:

```text
hash iteration
filesystem nondeterminism
```

---

# 67. Event ordering

Debe distinguirse:

```text
listener ordering
```

de:

```text
event ordering
```

---

# 68. Local event ordering

Dentro de una ejecución síncrona:

```text
QueryStarted
↓
QueryCompleted
```

deberá respetarse.

---

# 69. Cross-worker ordering

No deberá asumirse:

```text
global total ordering
```

entre workers independientes.

---

# 70. Distributed ordering

Si eventos salen a infraestructura distribuida:

```text
Event A sent before B
```

no garantiza necesariamente:

```text
A consumed before B
```

sin protocolo específico.

---

# 71. Sequence numbers

Para streams que lo requieran podrá existir:

```text
OperationSequence
TransactionSequence
```

---

# 72. Sequence scope

Debe ser explícito:

```text
per operation
per transaction
per connection
```

No deberá fingirse secuencia global.

---

# 73. Synchronous dispatch

Será el modelo base para eventos internos de Database.

Ventajas:

```text
deterministic lifecycle
simple diagnostics
immediate observation
```

---

# 74. Async dispatch

No deberá aplicarse automáticamente a todos los eventos.

---

# 75. Async bridge

Arquitectura:

```text
Database Event
↓
Synchronous Local Dispatch
↓
Async Event Bridge
↓
Message/Queue
```

---

# 76. Event dispatcher ≠ queue producer

El core Database no deberá necesitar queue para emitir eventos.

---

# 77. Async event semantics

Un evento enviado asíncronamente deberá contener suficiente información estable para sobrevivir después del scope original.

---

# 78. No live object references

Nunca transportar:

```text
managed entity object
Connection
EntityManager
Result
TransactionContext
```

a un listener async.

---

# 79. Transaction awareness

Algunos eventos necesitan conocer el estado transaccional.

Ejemplo:

```text
EntityPersisted
```

puede ocurrir antes del commit físico.

---

# 80. Persisted ≠ committed

Siempre:

```text
Statement Executed
≠
Transaction Committed
```

---

# 81. Transaction-aware dispatch

Podrán existir políticas:

```php
enum DatabaseEventDeliveryMoment
{
    case IMMEDIATE;
    case AFTER_COMMIT;
    case AFTER_ROLLBACK;
    case AFTER_COMPLETION;
}
```

---

# 82. IMMEDIATE

Se emite durante el pipeline actual.

---

# 83. AFTER_COMMIT

Se entrega únicamente cuando el commit está confirmado.

---

# 84. AFTER_ROLLBACK

Se entrega cuando rollback está confirmado.

---

# 85. AFTER_COMPLETION

Puede observar:

```text
COMMITTED
ROLLED_BACK
FAILED
UNKNOWN
```

según contrato.

---

# 86. Deferred event buffer

Eventos `AFTER_COMMIT` podrán almacenarse temporalmente en:

```text
TransactionEventBuffer
```

---

# 87. TransactionEventBuffer scope

Deberá pertenecer al:

```text
TransactionContext
```

o scope equivalente.

Nunca será global.

---

# 88. Commit flow

```text
Operation
↓
queue after-commit events
↓
COMMIT
↓
confirmed success
↓
release after-commit events
```

---

# 89. Rollback flow

```text
Operation
↓
queue after-commit events
↓
ROLLBACK
↓
discard after-commit events
```

---

# 90. Unknown commit flow

```text
COMMIT sent
↓
connection lost
↓
outcome UNKNOWN
```

No se deberá:

```text
dispatch AfterCommit
```

ni:

```text
assume rollback
```

---

# 91. Unknown handling

Eventos dependientes del outcome permanecerán:

```text
UNRESOLVED
```

o serán descartados según política conservadora.

Podrá emitirse:

```text
TransactionOutcomeUnknown
```

---

# 92. Nested transactions

El Event System deberá respetar la arquitectura de:

```text
logical nested transactions
savepoints
physical transaction
```

---

# 93. Inner logical success ≠ physical commit

Si:

```text
Outer TX
 ├── Inner Scope
 │     └── success
 └── later rollback
```

no deberán publicarse como committed eventos del inner scope.

---

# 94. Savepoint release ≠ commit

Siempre:

```text
RELEASE SAVEPOINT
≠
COMMIT
```

---

# 95. AfterCommit attaches to physical outcome

Por default:

```text
AFTER_COMMIT
```

se vinculará al commit físico efectivo.

---

# 96. Transaction retries

Un intento fallido puede generar eventos internos.

Pero eventos externos de negocio no deberán duplicarse por cada retry accidentalmente.

---

# 97. Attempt awareness

Podrá incluirse:

```text
TransactionAttemptId
```

en contexto.

---

# 98. Attempt event ≠ final event

Ejemplo:

```text
TransactionAttemptDeadlocked
```

no equivale a:

```text
TransactionFailed
```

si luego un retry tiene éxito.

---

# 99. Event failure policy

No todos los listeners deberán tener la misma política.

```php
enum DatabaseListenerFailurePolicy
{
    case PROPAGATE;
    case RECORD_AND_CONTINUE;
    case DISABLE_LISTENER;
    case CUSTOM;
}
```

---

# 100. Default observer policy

Para listeners observacionales:

```text
RECORD_AND_CONTINUE
```

será generalmente preferible.

---

# 101. Critical extension listeners

Si un listener forma parte de una garantía contractual crítica, probablemente no debería ser un listener genérico.

Debería modelarse como:

```text
Hook
Policy
Interceptor
Coordinator
```

---

# 102. Listener exceptions

Nunca deberán perderse silenciosamente.

Podrán registrarse como:

```text
ListenerFailure
```

en diagnostics/telemetry.

---

# 103. Listener recursion

Un listener podría ejecutar otra query.

Ejemplo:

```text
QueryCompleted
↓
listener writes audit row
↓
QueryStarted
↓
QueryCompleted
```

Esto puede generar recursión.

---

# 104. Recursion guard

El Event System deberá poder controlar:

```text
maximum event depth
listener reentrancy
self-generated event suppression
```

---

# 105. Reentrancy policy

```php
enum ListenerReentrancyPolicy
{
    case ALLOW;
    case DENY_SELF;
    case DENY_EVENT_TYPE;
    case MAX_DEPTH;
}
```

---

# 106. Audit example

Un audit listener que escribe en Database deberá poder marcar:

```text
origin = database.audit
```

y evitar auditar infinitamente sus propias escrituras.

---

# 107. Event origin

```php
enum DatabaseEventOrigin
{
    case APPLICATION;
    case ORM;
    case QUERY_ENGINE;
    case TRANSACTION;
    case MIGRATION;
    case CACHE;
    case TELEMETRY;
    case AUDIT;
    case INTERNAL;
}
```

---

# 108. Origin metadata

Ayudará a:

```text
filter recursion
diagnostics
security
telemetry
```

---

# 109. Event suppression

Podrá existir suppression controlada.

Ejemplo:

```php
$context->events()->without(
    DatabaseEventCategory::TELEMETRY,
    $callback
);
```

pero deberá ser scope-local.

---

# 110. Suppression ≠ disabling correctness

No se podrán suprimir mediante convenience API componentes necesarios para correctness.

---

# 111. Suppression reasons

Deberán ser explícitos:

```text
testing
internal audit write
bootstrap
maintenance
telemetry noise reduction
```

---

# 112. Event filtering

```php
interface DatabaseEventFilter
{
    public function accepts(
        DatabaseEvent $event,
        DatabaseEventDispatchContext $context,
    ): bool;
}
```

---

# 113. Filtering ≠ authorization

Aunque un filtro pueda limitar listeners:

```text
Event Filter
≠
Security Authorization
```

---

# 114. Listener authorization

Listeners que accedan a datos sensibles deberán seguir las reglas de seguridad correspondientes.

---

# 115. Tenant-aware events

Cuando exista tenant:

```text
TenantContextReference
```

podrá incluirse.

---

# 116. Tenant isolation

Un listener tenant-scoped no deberá recibir eventos de otro tenant por error.

---

# 117. Global listeners

Listeners administrativos podrán observar múltiples tenants únicamente con autorización explícita.

---

# 118. Tenant ID cardinality

Telemetry no deberá convertir automáticamente cada tenant ID en metric label.

---

# 119. Shard-aware events

Eventos distribuidos podrán incluir:

```text
ShardId
ReplicationGroupId
EndpointRole
```

cuando sea necesario.

---

# 120. Endpoint data

No deberán exponerse credenciales.

---

# 121. Connection events

Especialización posterior:

```text
211_DATABASE_CONNECTION_EVENT_SYSTEM.md
```

deberá definir:

```text
acquire
open
reuse
release
failure
reset
failover
```

---

# 122. Query events

Especialización:

```text
210_DATABASE_QUERY_EVENT_SYSTEM.md
```

deberá definir:

```text
planning
compilation
execution
success
failure
cancellation
retry
```

sin duplicar Telemetry.

---

# 123. Transaction events

Especialización:

```text
212_DATABASE_TRANSACTION_EVENT_PIPELINE.md
```

deberá definir:

```text
begin
savepoint
commit
rollback
retry
unknown
afterCommit
```

---

# 124. Entity lifecycle events

Especialización:

```text
213_DATABASE_ENTITY_LIFECYCLE_EVENT_SYSTEM.md
```

deberá diferenciar:

```text
preLoad
postLoad
prePersist
postPersist
preRemove
postRemove
```

de Domain Events.

---

# 125. Persistence events

Especialización:

```text
214_DATABASE_PERSISTENCE_EVENT_SYSTEM.md
```

cubrirá:

```text
flush
change sets
persistence planning
insert/update/delete execution
consistency
```

---

# 126. Extension events

Especialización:

```text
215_DATABASE_EVENT_EXTENSION_SYSTEM.md
```

definirá extensibilidad sin romper invariantes.

---

# 127. Schema events

Aunque no tienen documento propio inmediato, podrán existir:

```text
SchemaIntrospectionStarted
SchemaIntrospectionCompleted
SchemaCompilationCompleted
```

si son útiles.

---

# 128. Migration events

Podrán incluir:

```text
MigrationStarted
MigrationBatchStarted
MigrationCompleted
MigrationFailed
MigrationRollbackCompleted
```

---

# 129. Cache events

Ejemplos:

```text
DatabaseCacheHit
DatabaseCacheMiss
DatabaseCacheInvalidated
```

pero deberán evitar alto volumen innecesario.

---

# 130. Large dataset events

Del documento 208:

```text
LargeDatasetStarted
PartitionStarted
PartitionCompleted
LargeDatasetCompleted
```

podrán integrarse a esta arquitectura.

---

# 131. Event volume classes

```php
enum DatabaseEventVolumeClass
{
    case LOW;
    case MEDIUM;
    case HIGH;
    case VERY_HIGH;
}
```

---

# 132. High-volume events

Ejemplos:

```text
QueryCompleted
CacheHit
EntityHydrated
```

podrían ocurrir millones de veces.

---

# 133. Volume policy

El sistema podrá aplicar:

```text
sampling
aggregation
listener restrictions
telemetry-only bridges
```

según categoría.

---

# 134. Sampling

Sampling no deberá alterar eventos necesarios para correctness.

Solo observabilidad.

---

# 135. Event sampling ≠ event loss semantics

Si un listener funcional necesita todos los eventos, no deberá depender de un stream muestreado.

---

# 136. Telemetry bridge

```php
final class DatabaseTelemetryEventBridge
{
    public function onDatabaseEvent(DatabaseEvent $event): void
    {
        // convert observable event to telemetry representation
    }
}
```

---

# 137. Event → Telemetry

La conversión podrá:

```text
sanitize
sample
aggregate
rename
reduce payload
```

---

# 138. Telemetry → Database Event

No deberá existir como dependencia inversa obligatoria.

---

# 139. Event correlation

Podrá correlacionarse:

```text
HTTP Request
    ↓
Database Operation
    ↓
Transaction
    ↓
Query
    ↓
Database Events
```

mediante IDs.

---

# 140. Trace integration

`TraceId` y `SpanId` podrán incorporarse mediante Telemetry adapter.

Database core no dependerá del SDK concreto.

---

# 141. Event propagation

Podrán existir scopes:

```php
enum DatabaseEventPropagationScope
{
    case OPERATION;
    case DATABASE;
    case FRAMEWORK;
    case EXTERNAL;
}
```

---

# 142. OPERATION

Solo listeners del scope actual.

---

# 143. DATABASE

Listeners registrados para Quantum Database.

---

# 144. FRAMEWORK

Bridge hacia Event System general.

---

# 145. EXTERNAL

Eventos explícitamente externalizables.

---

# 146. Default propagation

No todos los eventos deberán llegar a:

```text
FRAMEWORK
```

y mucho menos a:

```text
EXTERNAL
```

---

# 147. Internal noise

Eventos internos de alto volumen podrán permanecer dentro de:

```text
DATABASE
```

o:

```text
OPERATION
```

---

# 148. External event whitelist

Externalización deberá ser opt-in.

---

# 149. Event compatibility

Eventos públicos deberán tener política de compatibilidad.

---

# 150. Event contract changes

Cambios como:

```text
rename field
remove field
change meaning
change units
```

son breaking changes si el evento es público/externalizable.

---

# 151. Additive changes

Podrán permitirse dependiendo de la política de schema version.

---

# 152. Event registry

Podrá existir:

```php
interface DatabaseEventRegistry
{
    public function descriptorFor(
        string $eventClass,
    ): DatabaseEventDescriptor;
}
```

---

# 153. Stable IDs

No usar únicamente FQCN persistidos externamente.

Preferir:

```text
database.query.completed
```

---

# 154. Event discovery

Durante bootstrap:

```text
built-in events
+
package events
+
extension events
↓
validation
↓
registry
↓
freeze
```

---

# 155. Duplicate event name

Debe fallar durante bootstrap.

---

# 156. Unknown event

Un bridge externo deberá manejarlo de manera segura.

Nunca instanciar clases arbitrarias desde nombres no confiables.

---

# 157. Event deserialization security

Prohibido:

```php
unserialize($externalPayload);
```

para construir eventos arbitrarios.

---

# 158. Stable serializer

Eventos externalizables deberán utilizar:

```text
schema
stable primitive types
stable TypeIds
validated payload
```

---

# 159. Event metadata

```php
final readonly class DatabaseEventMetadata
{
    public function __construct(
        public array $values,
    ) {}
}
```

pero no deberá ser un `array` sin gobierno conceptual.

---

# 160. Metadata keys

Preferir namespaces:

```text
database.*
query.*
transaction.*
orm.*
extension.vendor.*
```

---

# 161. Metadata limits

Deberán existir límites para:

```text
key count
key length
value size
nesting depth
```

---

# 162. No arbitrary object metadata

Metadata deberá aceptar valores serializables/seguros.

---

# 163. Listener context

Un listener podrá recibir:

```php
final readonly class DatabaseListenerContext
{
    public function __construct(
        public DatabaseContextReference $database,
        public ListenerExecutionMetadata $execution,
    ) {}
}
```

sin recibir necesariamente el DatabaseContext mutable completo.

---

# 164. Least authority

Listeners recibirán únicamente las capacidades necesarias.

---

# 165. Listener with DB access

Si necesita ejecutar Database operations deberá obtener un servicio explícito mediante DI.

No a través del evento.

---

# 166. Event object ≠ Service Locator

Nunca:

```php
$event->database()->query(...);
```

como diseño general.

---

# 167. Listener dependency injection

```php
final class AuditQueryListener
{
    public function __construct(
        private AuditWriter $writer,
    ) {}
}
```

---

# 168. Testing architecture

El sistema deberá ofrecer:

```text
event fake
event recorder
event assertions
listener tests
ordering tests
transaction-boundary tests
```

---

# 169. DatabaseEventRecorder

```php
final class DatabaseEventRecorder
{
    public function recorded(): array;

    public function ofType(string $event): array;
}
```

---

# 170. Assertions

Ejemplo:

```php
DatabaseEvents::assertDispatched(
    QueryCompleted::class
);
```

---

# 171. Event fake

Un fake no deberá alterar automáticamente Database correctness.

---

# 172. Transaction tests

Deberán probar:

```text
afterCommit emitted on confirmed commit
afterCommit not emitted on rollback
afterCommit not emitted on unknown
afterRollback emitted on rollback
nested scopes behave correctly
```

---

# 173. Ordering tests

Ejemplo:

```text
TransactionStarted
QueryStarted
QueryCompleted
TransactionCommitted
AfterCommitEvent
```

---

# 174. Listener failure tests

Probar:

```text
listener throws before operation
listener throws after successful operation
listener throws after commit
listener recursively executes DB operation
```

---

# 175. Persistent runtime architecture

Crítico para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 176. Shared immutable state

Podrá compartirse:

```text
compiled event registry
listener descriptors
event metadata definitions
immutable routing tables
```

---

# 177. Scope-local mutable state

Deberá permanecer local:

```text
event buffer
suppression state
dispatch stack
recursion depth
transaction deferred events
operation listeners
tenant context
trace context
```

---

# 178. Request cleanup

```text
request starts
↓
event scope created
↓
database operations
↓
event buffers resolved
↓
listeners released
↓
suppression reset
↓
dispatch stack cleared
↓
request ends
```

---

# 179. No event leakage

Eventos pendientes de Request A jamás deberán publicarse durante Request B.

---

# 180. Transaction buffer cleanup

Al cerrar un TransactionContext:

```text
COMMITTED
ROLLED_BACK
FAILED
UNKNOWN
```

su event buffer deberá quedar resuelto/invalidado.

---

# 181. OpenSwoole

Dispatch state deberá ser coroutine-local.

---

# 182. FrankenPHP

No utilizar static mutable listener stacks por request.

---

# 183. RoadRunner

Worker reuse deberá resetear scopes.

---

# 184. Performance

El Event System deberá tener overhead bajo cuando no existen listeners.

---

# 185. Fast path

Conceptualmente:

```php
if (!$registry->hasListeners(QueryCompleted::class)) {
    return;
}
```

---

# 186. Lazy event construction

Para eventos costosos podrá evitarse construir payload completo si:

```text
no listeners
+
no telemetry bridge
```

---

# 187. EventFactory

```php
interface DatabaseEventFactory
{
    public function create(
        DatabaseEventDescriptor $descriptor,
        DatabaseEventContext $context,
        mixed $payload,
    ): DatabaseEvent;
}
```

---

# 188. No expensive diagnostics by default

No calcular:

```text
full query explain
full entity diff
full stack trace
```

solo para un evento que nadie consume.

---

# 189. High-frequency path

Query events deberán optimizar:

```text
listener lookup
context allocation
timestamp acquisition
payload creation
dispatch
```

---

# 190. Event batching

Telemetry podrá agregar eventos.

El Event System funcional no deberá batchificar eventos arbitrariamente si cambia ordering semantics.

---

# 191. Event listener timeout

Listeners externos o lentos podrán tener políticas de timeout en integraciones superiores.

---

# 192. Database core listeners

No deberían realizar network I/O pesado de forma síncrona.

---

# 193. Async side effects

Para efectos importantes posteriores al commit:

```text
Transaction
↓
Outbox
↓
Commit
↓
Publisher
```

será más robusto que depender exclusivamente de:

```text
afterCommit listener
↓
remote HTTP call
```

---

# 194. AfterCommit ≠ Outbox

Siempre:

```text
AfterCommit Callback
≠
Durable Outbox
```

---

# 195. Process crash after commit

Escenario:

```text
COMMIT succeeds
↓
process crashes
↓
afterCommit listener never runs
```

Por tanto, afterCommit no ofrece entrega durable por sí solo.

---

# 196. Durable events

Cuando se requiera durabilidad deberá usarse:

```text
transactional outbox
durable event store
```

como integración explícita.

---

# 197. Event delivery guarantees

Podrán modelarse:

```php
enum DatabaseEventDeliveryGuarantee
{
    case BEST_EFFORT;
    case IN_PROCESS;
    case DURABLE_AT_LEAST_ONCE;
    case CUSTOM;
}
```

---

# 198. No fake exactly-once

El Event System no deberá anunciar:

```text
EXACTLY_ONCE
```

sin protocolo capaz de demostrarlo end-to-end.

---

# 199. Event cancellation

Eventos observacionales no deberán cancelar operaciones.

---

# 200. Cancellable hooks

Si se requiere:

```text
veto
```

deberá existir un contrato separado.

Ejemplo:

```php
interface PersistenceGuard
{
    public function evaluate(
        PersistenceOperation $operation,
    ): GuardDecision;
}
```

---

# 201. Why not event veto?

Porque:

```text
listener order
+
listener discovery
+
hidden mutation
```

puede hacer que correctness dependa de efectos laterales difíciles de razonar.

---

# 202. Event mutation

Los listeners no modificarán el evento.

---

# 203. Event replacement

Tampoco:

```php
return $modifiedEvent;
```

como mecanismo estándar.

---

# 204. Event propagation stop

Para eventos internos podrá existir una política especializada de stop propagation, pero no deberá usarse para alterar el resultado de Database.

---

# 205. Listener isolation

Un listener deberá poder fallar sin corromper:

```text
Query Context
Transaction Context
EntityManager
UnitOfWork
Connection
```

---

# 206. Listener transaction access

Un listener ejecutado dentro de una transacción puede compartir el transaction scope solo si la integración lo declara.

---

# 207. Hidden transaction writes

Deberán evitarse.

Un listener que escriba en la misma transacción puede cambiar:

```text
lock set
deadlock probability
transaction duration
failure behavior
```

---

# 208. Listener side-effect classification

```php
enum DatabaseListenerSideEffect
{
    case NONE;
    case DATABASE_READ;
    case DATABASE_WRITE;
    case EXTERNAL_IO;
    case UNKNOWN;
}
```

---

# 209. Diagnostics warning

El sistema podrá advertir:

```text
Listener performs DATABASE_WRITE during Transaction event.
```

---

# 210. Event execution budget

Podrá existir:

```text
maximum listener count
maximum recursion depth
maximum dispatch duration
```

especialmente para persistent runtimes.

---

# 211. Extension event namespace

Paquetes externos deberán usar:

```text
extension.vendor.package.event
```

o identidad equivalente.

---

# 212. Extension collision

Dos extensiones no podrán registrar el mismo stable event name.

---

# 213. Extension event contracts

No podrán modificar invariantes de eventos built-in.

---

# 214. Event registry validation

Bootstrap deberá validar:

```text
duplicate names
invalid categories
invalid versions
invalid listener signatures
invalid priorities
unsupported externalization
unsafe metadata
```

---

# 215. Configuration

Ejemplo conceptual:

```php
return [

    'database' => [

        'events' => [

            'enabled' => true,

            'bridge' => 'voltstack',

            'listeners' => [
                QueryCompleted::class => [
                    QueryMetricsListener::class,
                ],
            ],

            'security' => [
                'include_sql' => false,
                'include_parameters' => false,
            ],

            'limits' => [
                'max_recursion_depth' => 8,
            ],

        ],

    ],

];
```

---

# 216. Enabled=false

Deshabilitar eventos observacionales no deberá romper Database core.

---

# 217. Mandatory internal signals

Si un mecanismo es necesario para correctness:

```text
cache invalidation
transaction synchronization
UoW synchronization
```

no deberá depender exclusivamente de un evento deshabilitable.

---

# 218. Critical principle

> **Correctness-critical coordination deberá utilizar contratos directos; Events serán principalmente un mecanismo de observación e integración desacoplada.**

---

# 219. Cache invalidation example

Incorrecto:

```text
UPDATE
↓
optional QueryCompleted listener
↓
invalidate cache
```

si deshabilitar eventos rompe coherencia.

Preferible:

```text
Persistence/Transaction coordination
↓
CacheInvalidation contract
```

y adicionalmente:

```text
CacheInvalidated event
```

para observación.

---

# 220. Transaction synchronization

Igualmente:

```text
Transaction Synchronization
≠
Transaction Event Listener
```

---

# 221. Directory structure

```text
src/Quantum/Database/Event/
│
├── Contract/
│   ├── DatabaseEvent.php
│   ├── DatabaseEventDispatcher.php
│   ├── DatabaseEventListener.php
│   ├── DatabaseEventListenerRegistry.php
│   ├── DatabaseEventFilter.php
│   └── ExternalizableDatabaseEvent.php
│
├── Model/
│   ├── DatabaseEventId.php
│   ├── DatabaseEventName.php
│   ├── DatabaseEventDescriptor.php
│   ├── DatabaseEventEnvelope.php
│   ├── DatabaseEventContext.php
│   ├── DatabaseEventMetadata.php
│   ├── DatabaseEventCategory.php
│   ├── DatabaseEventPhase.php
│   ├── DatabaseEventOrigin.php
│   ├── DatabaseEventVolumeClass.php
│   ├── DatabaseEventPropagationScope.php
│   ├── DatabaseEventDeliveryMoment.php
│   └── DatabaseEventDeliveryGuarantee.php
│
├── Dispatch/
│   ├── DefaultDatabaseEventDispatcher.php
│   ├── NullDatabaseEventDispatcher.php
│   ├── DatabaseEventDispatchContext.php
│   ├── DatabaseEventDispatchResult.php
│   ├── DatabaseEventDispatchStatus.php
│   ├── DatabaseListenerFailurePolicy.php
│   ├── ListenerReentrancyPolicy.php
│   └── DispatchStack.php
│
├── Registry/
│   ├── DatabaseEventRegistry.php
│   ├── CompiledDatabaseEventRegistry.php
│   ├── CompiledListenerRegistry.php
│   ├── DatabaseEventRegistryCompiler.php
│   └── DatabaseEventRegistryValidator.php
│
├── Context/
│   ├── DatabaseEventScope.php
│   ├── DatabaseEventScopeFactory.php
│   ├── DatabaseEventSuppression.php
│   └── DatabaseEventOriginContext.php
│
├── Transaction/
│   ├── TransactionEventBuffer.php
│   ├── DeferredDatabaseEvent.php
│   └── TransactionEventCoordinator.php
│
├── Security/
│   ├── DatabaseEventSanitizer.php
│   ├── DatabaseEventSecurityPolicy.php
│   └── DatabaseEventPayloadPolicy.php
│
├── Bridge/
│   ├── DatabaseEventBridge.php
│   ├── FrameworkEventBridge.php
│   ├── TelemetryEventBridge.php
│   └── AsyncEventBridge.php
│
├── Testing/
│   ├── FakeDatabaseEventDispatcher.php
│   ├── DatabaseEventRecorder.php
│   └── DatabaseEventAssertions.php
│
├── Diagnostics/
│   ├── DatabaseEventInspector.php
│   ├── ListenerFailure.php
│   └── DatabaseEventDispatchReport.php
│
└── Exception/
    ├── DatabaseEventException.php
    ├── DatabaseEventDispatchException.php
    ├── DatabaseEventListenerException.php
    ├── DatabaseEventRegistryException.php
    ├── DatabaseEventSerializationException.php
    ├── DatabaseEventSecurityException.php
    ├── DatabaseEventRecursionException.php
    └── DatabaseEventCompatibilityException.php
```

---

# 222. Especializaciones

Posteriormente:

```text
Event/
├── Query/
├── Connection/
├── Transaction/
├── Entity/
├── Persistence/
└── Extension/
```

podrán contener los contratos especializados de 210–215.

---

# 223. Dependency direction

Correcto:

```text
Query Engine ──────┐
Connection ────────┤
Transaction ───────┤
ORM ───────────────┤
Persistence ───────┤
                   ▼
          Database Event Contracts
                   │
                   ▼
            Event Dispatcher
                   │
             ┌─────┴─────┐
             ▼           ▼
         Listeners     Bridges
```

---

# 224. Incorrect dependency

No:

```text
Query Engine
↓
Global Event Framework implementation
↓
HTTP Event Bus
```

---

# 225. Event integration architecture

```text
Quantum Database
      │
      ▼
DatabaseEventDispatcher
      │
      ├───────────────┐
      ▼               ▼
Local Listeners   Framework Bridge
                       │
                       ▼
                VoltStack EventSystem
                       │
            ┌──────────┼──────────┐
            ▼          ▼          ▼
        Telemetry     App       Extensions
```

---

# 226. Architectural invariants

## DB-EVENT-001
Database Event será distinto de Domain Event.

## DB-EVENT-002
Database Event será distinto de Telemetry Event.

## DB-EVENT-003
Database Event será distinto de ORM Lifecycle Event.

## DB-EVENT-004
Database Event será distinto de Persistence Event.

## DB-EVENT-005
Database Event será distinto de Transaction Outcome.

## DB-EVENT-006
Database Event será distinto de Hook.

## DB-EVENT-007
Database Event será distinto de Interceptor.

## DB-EVENT-008
Database Event será distinto de Command.

## DB-EVENT-009
Database Event será distinto de canonical component state.

## DB-EVENT-010
Events observarán por default.

## DB-EVENT-011
Listeners genéricos no redefinirán query semantics.

## DB-EVENT-012
Listeners genéricos no redefinirán transaction semantics.

## DB-EVENT-013
Listeners genéricos no redefinirán persistence semantics.

## DB-EVENT-014
Correctness-critical behavior no dependerá exclusivamente de eventos opcionales.

## DB-EVENT-015
Quantum Database dependerá de contracts, no de EventSystem concreto.

## DB-EVENT-016
Null dispatcher será soportado.

## DB-EVENT-017
Database deberá funcionar sin EventSystem externo.

## DB-EVENT-018
Eventos serán conceptualmente inmutables.

## DB-EVENT-019
Event ID será distinto de Operation ID.

## DB-EVENT-020
Event ID será distinto de Transaction ID.

## DB-EVENT-021
Event context será explícito.

## DB-EVENT-022
Event context no será global mutable state.

## DB-EVENT-023
Event context deberá ser scope-safe.

## DB-EVENT-024
Event payload será bounded.

## DB-EVENT-025
Event payload no será state dump.

## DB-EVENT-026
Raw SQL no será público por default.

## DB-EVENT-027
Parameter values no serán públicos por default.

## DB-EVENT-028
Credentials jamás deberán exponerse.

## DB-EVENT-029
Sensitive data deberá sanitizarse.

## DB-EVENT-030
Categoría será distinta de event class.

## DB-EVENT-031
Phase no será obligatoria para todos los eventos.

## DB-EVENT-032
No se inventarán lifecycle phases sin semántica útil.

## DB-EVENT-033
Before event será observacional por default.

## DB-EVENT-034
Pre-operation mutation utilizará contratos especializados.

## DB-EVENT-035
PHP class será distinta de stable external event name.

## DB-EVENT-036
External events serán versionados.

## DB-EVENT-037
Internal event no implicará serializable event.

## DB-EVENT-038
PDO no será event payload externalizable.

## DB-EVENT-039
Connection no será event payload externalizable.

## DB-EVENT-040
EntityManager no será event payload externalizable.

## DB-EVENT-041
TransactionContext no será event payload externalizable.

## DB-EVENT-042
Generator no será event payload externalizable.

## DB-EVENT-043
Closure no será event payload externalizable.

## DB-EVENT-044
Entity graph no será externalizado automáticamente.

## DB-EVENT-045
Dispatcher será distinto de operation executor.

## DB-EVENT-046
Dispatch result será distinto de database outcome.

## DB-EVENT-047
Listener registry podrá compilarse.

## DB-EVENT-048
Production registry podrá congelarse.

## DB-EVENT-049
Runtime listener state será scope-local.

## DB-EVENT-050
Listener registration no deberá filtrarse entre requests.

## DB-EVENT-051
Listener priority será determinista.

## DB-EVENT-052
Equal priority tendrá orden estable.

## DB-EVENT-053
Listener ordering será distinto de event ordering.

## DB-EVENT-054
Local lifecycle ordering será preservado.

## DB-EVENT-055
Cross-worker global ordering no será asumido.

## DB-EVENT-056
Distributed total ordering no será fingido.

## DB-EVENT-057
Sequence scope será explícito.

## DB-EVENT-058
Synchronous dispatch será soportado.

## DB-EVENT-059
Async dispatch será integración.

## DB-EVENT-060
Database core no dependerá de Queue.

## DB-EVENT-061
Async events no transportarán live DB resources.

## DB-EVENT-062
Statement success será distinto de transaction commit.

## DB-EVENT-063
Persisted será distinto de committed.

## DB-EVENT-064
AfterCommit solo se entregará tras commit confirmado.

## DB-EVENT-065
AfterCommit no se entregará tras rollback.

## DB-EVENT-066
AfterCommit no se entregará cuando outcome sea UNKNOWN.

## DB-EVENT-067
AfterRollback requerirá rollback confirmado.

## DB-EVENT-068
Transaction event buffer será transaction-scoped.

## DB-EVENT-069
Transaction event buffer nunca será global.

## DB-EVENT-070
Savepoint release será distinto de commit.

## DB-EVENT-071
Inner logical transaction success será distinto de physical commit.

## DB-EVENT-072
AfterCommit se vinculará al physical commit por default.

## DB-EVENT-073
Transaction attempt será distinto de final transaction outcome.

## DB-EVENT-074
Retry events serán attempt-aware.

## DB-EVENT-075
Listener failure será distinto de transaction failure.

## DB-EVENT-076
AfterCommit listener failure no deshará commit.

## DB-EVENT-077
Listener exceptions serán observables.

## DB-EVENT-078
Observer listener failure podrá continuar según policy.

## DB-EVENT-079
Critical guarantees deberán usar contratos directos.

## DB-EVENT-080
Listener recursion será gobernada.

## DB-EVENT-081
Dispatch depth será bounded.

## DB-EVENT-082
Reentrancy policy será explícita.

## DB-EVENT-083
Audit listeners no deberán generar loops infinitos.

## DB-EVENT-084
Event origin será representable.

## DB-EVENT-085
Suppression será scope-local.

## DB-EVENT-086
Suppression no deshabilitará correctness-critical coordination.

## DB-EVENT-087
Filtering será distinto de authorization.

## DB-EVENT-088
Tenant event isolation será preservada.

## DB-EVENT-089
Global tenant listeners requerirán autorización.

## DB-EVENT-090
Tenant IDs no serán metric labels automáticos.

## DB-EVENT-091
Shard information será bounded.

## DB-EVENT-092
Connection credentials nunca serán event metadata.

## DB-EVENT-093
High-volume events serán identificables.

## DB-EVENT-094
Sampling solo afectará observabilidad.

## DB-EVENT-095
Sampling no afectará correctness.

## DB-EVENT-096
Telemetry bridge podrá reducir payload.

## DB-EVENT-097
Telemetry bridge podrá samplear.

## DB-EVENT-098
Telemetry no será dependencia inversa obligatoria.

## DB-EVENT-099
Trace integration será adapter-based.

## DB-EVENT-100
Propagation scope será explícito.

## DB-EVENT-101
No todos los eventos llegarán al framework global.

## DB-EVENT-102
External propagation será opt-in.

## DB-EVENT-103
Public event compatibility será gobernada.

## DB-EVENT-104
Stable event names no dependerán exclusivamente de FQCN.

## DB-EVENT-105
Duplicate stable event names serán rechazados.

## DB-EVENT-106
External payload no instanciará clases arbitrarias.

## DB-EVENT-107
Unsafe unserialize estará prohibido.

## DB-EVENT-108
Metadata tendrá límites.

## DB-EVENT-109
Metadata no aceptará arbitrary live objects.

## DB-EVENT-110
Listener recibirá least authority.

## DB-EVENT-111
Event no será Service Locator.

## DB-EVENT-112
Listener dependencies usarán DI.

## DB-EVENT-113
Event fake no romperá Database correctness.

## DB-EVENT-114
AfterCommit behavior será testeable.

## DB-EVENT-115
Rollback event behavior será testeable.

## DB-EVENT-116
Unknown outcome será testeable.

## DB-EVENT-117
Nested transaction event behavior será testeable.

## DB-EVENT-118
Persistent runtime registry podrá compartirse si es inmutable.

## DB-EVENT-119
Mutable dispatch state será scope-local.

## DB-EVENT-120
Deferred events no se filtrarán entre requests.

## DB-EVENT-121
Suppression state se reseteará.

## DB-EVENT-122
Dispatch stack se reseteará.

## DB-EVENT-123
FrankenPHP event state será request-scoped.

## DB-EVENT-124
RoadRunner event state será operation-scoped.

## DB-EVENT-125
OpenSwoole event state será coroutine-safe.

## DB-EVENT-126
No-listener path deberá ser barato.

## DB-EVENT-127
Expensive payload construction podrá ser lazy.

## DB-EVENT-128
High-frequency event dispatch deberá optimizarse.

## DB-EVENT-129
Event batching no cambiará functional ordering semantics.

## DB-EVENT-130
Core listeners deberán evitar slow external I/O.

## DB-EVENT-131
AfterCommit será distinto de Outbox.

## DB-EVENT-132
AfterCommit no garantizará durable delivery.

## DB-EVENT-133
Process crash after commit podrá perder best-effort callback.

## DB-EVENT-134
Durable delivery requerirá mecanismo durable.

## DB-EVENT-135
Exactly-once no será fingido.

## DB-EVENT-136
Observational events no cancelarán operaciones.

## DB-EVENT-137
Veto utilizará contrato especializado.

## DB-EVENT-138
Event mutation no será mecanismo estándar.

## DB-EVENT-139
Event replacement no será mecanismo estándar.

## DB-EVENT-140
Stop propagation no alterará Database outcome.

## DB-EVENT-141
Listener no deberá corromper DatabaseContext.

## DB-EVENT-142
Listener no deberá corromper TransactionContext.

## DB-EVENT-143
Listener no deberá corromper EntityManager.

## DB-EVENT-144
Listener no deberá corromper UnitOfWork.

## DB-EVENT-145
Listener no deberá corromper Connection.

## DB-EVENT-146
Listener DB writes serán explícitos.

## DB-EVENT-147
Hidden transactional writes serán diagnosticables.

## DB-EVENT-148
Listener side effects podrán clasificarse.

## DB-EVENT-149
Dispatch budget será configurable.

## DB-EVENT-150
Extension event namespace será estable.

## DB-EVENT-151
Extension collisions serán rechazadas.

## DB-EVENT-152
Extensions no romperán built-in event invariants.

## DB-EVENT-153
Registry validation ocurrirá en bootstrap cuando sea posible.

## DB-EVENT-154
Events disabled no romperá Database core.

## DB-EVENT-155
Cache correctness no dependerá de optional observer listeners.

## DB-EVENT-156
Transaction synchronization no dependerá de optional observer listeners.

## DB-EVENT-157
Event system será una integración transversal.

## DB-EVENT-158
Database event semantics tendrán precedencia sobre listener convenience.

## DB-EVENT-159
Known database outcome tendrá precedencia sobre listener outcome.

## DB-EVENT-160
UNKNOWN transaction outcome permanecerá UNKNOWN.

## DB-EVENT-161
Event delivery failure no reescribirá database reality.

## DB-EVENT-162
Security tendrá precedencia sobre observability richness.

## DB-EVENT-163
Bounded payload tendrá precedencia sobre convenience dumps.

## DB-EVENT-164
Deterministic dispatch tendrá precedencia sobre incidental registration order.

## DB-EVENT-165
Event contracts serán typed.

## DB-EVENT-166
Cross-system bridges preservarán semantic identity.

## DB-EVENT-167
Event schema version será distinta de framework version.

## DB-EVENT-168
Database events podrán existir sin externalization.

## DB-EVENT-169
Database events podrán existir sin telemetry.

## DB-EVENT-170
Database events podrán existir sin application listeners.

## DB-EVENT-171
Events no reemplazarán direct component contracts.

## DB-EVENT-172
Events no reemplazarán Query AST.

## DB-EVENT-173
Events no reemplazarán TransactionContext.

## DB-EVENT-174
Events no reemplazarán EntityState.

## DB-EVENT-175
Events no reemplazarán PersistencePlan.

## DB-EVENT-176
Events no reemplazarán Cache invalidation protocol.

## DB-EVENT-177
Events no reemplazarán Security policies.

## DB-EVENT-178
Events no reemplazarán Telemetry.

## DB-EVENT-179
Events no reemplazarán durable messaging.

## DB-EVENT-180
Database Event Architecture preservará las fronteras del subsistema.

---

# 227. Modelo formal

Sea una operación:

```text
O
```

con estado:

```text
S(O,t)
```

en tiempo `t`.

Un evento:

```text
E
```

representa una observación:

```text
E = Observe(O, phase, context, t)
```

No deberá asumirse:

```text
Apply(E) → mutate(O)
```

Por default:

```text
Dispatch(E)
```

produce únicamente efectos en listeners.

---

# 228. Operación y evento

Si:

```text
O
→
SUCCESS
```

y posteriormente:

```text
Dispatch(E_success)
→
ListenerFailure
```

entonces:

```text
DatabaseOutcome(O)
=
SUCCESS
```

mientras:

```text
EventDeliveryOutcome(E_success)
=
PARTIAL/FAILED
```

---

# 229. Transaction formalization

Sea:

```text
T
```

una transacción física.

Eventos `AFTER_COMMIT`:

```text
A(T)
```

solo podrán liberarse cuando:

```text
Outcome(T) = COMMITTED
```

Si:

```text
Outcome(T) = ROLLED_BACK
```

entonces:

```text
Dispatch(A(T)) = false
```

Si:

```text
Outcome(T) = UNKNOWN
```

también:

```text
Dispatch(A(T)) = false
```

por default conservador.

---

# 230. Nested scope

Sea:

```text
T
├── L1
└── L2
```

donde `L1` y `L2` son scopes lógicos.

Entonces:

```text
Complete(L1)
≠
Commit(T)
```

Por ello un evento:

```text
AfterCommit(L1)
```

deberá esperar el outcome físico de `T` salvo contrato explícito distinto.

---

# 231. Delivery durability

Para dispatch in-process:

```text
Commit(T)
↓
Process Crash
↓
Dispatch not guaranteed
```

Por tanto:

```text
AfterCommit
≠
Durable Delivery
```

Para entrega durable:

```text
T:
    business mutation
    +
    outbox record
↓
COMMIT
↓
outbox publisher
```

es el patrón recomendado.

---

# 232. Arquitectura final

```text
                         Quantum Database
                                │
          ┌─────────────────────┼─────────────────────┐
          ▼                     ▼                     ▼
      Query Engine          Transaction              ORM
          │                     │                     │
          └─────────────────────┼─────────────────────┘
                                ▼
                     Database Event Contracts
                                │
                                ▼
                       Event Factory/Envelope
                                │
                                ▼
                    DatabaseEventDispatcher
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
       Database Listeners   Event Bridges    Test Recorder
              │                 │
              │         ┌───────┼────────┐
              │         ▼       ▼        ▼
              │     EventSystem Telemetry Async
              │
              ▼
        Extensions
```

Para transacciones:

```text
Operation
    │
    ├── Immediate Event
    │       ↓
    │    Dispatcher
    │
    └── AfterCommit Event
            ↓
     TransactionEventBuffer
            ↓
        Physical Commit
          ┌─┴──────────────┐
          │                │
       SUCCESS          UNKNOWN/ROLLBACK
          │                │
          ▼                ▼
       Dispatch          Do not dispatch
```

---

# 233. Regla maestra final

La arquitectura de eventos de VoltStack Database deberá obedecer:

```text
Database Event
≠
Domain Event
```

```text
Database Event
≠
Telemetry Event
```

```text
Database Event
≠
Hook
```

```text
Database Event
≠
Command
```

```text
Database Event
≠
Canonical State
```

```text
Statement Success
≠
Transaction Commit
```

```text
Persisted
≠
Committed
```

```text
Savepoint Release
≠
Commit
```

```text
Inner Transaction Success
≠
Physical Commit
```

```text
Listener Failure
≠
Database Failure
```

```text
AfterCommit Listener Failure
≠
Commit Failure
```

```text
AfterCommit
≠
Durable Delivery
```

```text
AfterCommit
≠
Outbox
```

```text
Event Dispatch
≠
Queue Delivery
```

```text
Event Filtering
≠
Authorization
```

```text
Event Sampling
≠
Functional Delivery
```

```text
Event ID
≠
Operation ID
```

```text
Listener Ordering
≠
Global Event Ordering
```

```text
Events Disabled
≠
Database Correctness Disabled
```

y especialmente:

```text
Database Reality
≠
Event Delivery Outcome
```

---

# 234. Resultado arquitectónico

Con esta arquitectura VoltStack podrá proporcionar eventos suficientemente potentes para:

```text
extensions
debugging
profiling
auditing
framework integration
developer tooling
telemetry
testing
application observation
```

sin introducir el anti-patrón:

```text
Everything is an event
```

ni convertir el sistema en:

```text
Database Operation
↓
Unknown listener magic
↓
Unknown mutation
↓
Unknown side effect
↓
Unknown outcome
```

La arquitectura deseada será:

```text
Explicit operation
↓
Explicit canonical component
↓
Known outcome
↓
Typed event
↓
Controlled observation/integration
```

---

# 235. Bloque 20 — Estado

```text
BLOCK 20 — EVENTS

✓ 209_DATABASE_EVENT_ARCHITECTURE.md
○ 210_DATABASE_QUERY_EVENT_SYSTEM.md
○ 211_DATABASE_CONNECTION_EVENT_SYSTEM.md
○ 212_DATABASE_TRANSACTION_EVENT_PIPELINE.md
○ 213_DATABASE_ENTITY_LIFECYCLE_EVENT_SYSTEM.md
○ 214_DATABASE_PERSISTENCE_EVENT_SYSTEM.md
○ 215_DATABASE_EVENT_EXTENSION_SYSTEM.md
```

---

# 236. Siguiente documento

```text
210_DATABASE_QUERY_EVENT_SYSTEM.md
```

El siguiente documento especializará esta arquitectura para el ciclo completo de una query:

```text
Query Model
↓
Semantic Analysis
↓
Optimization
↓
Planning
↓
Compilation
↓
Execution
↓
Result
```

y deberá definir, entre otros:

```text
QueryPlanningStarted
QueryPlanned
QueryCompilationStarted
QueryCompiled
QueryExecutionStarted
QueryExecuted
QueryFailed
QueryCancelled
QueryRetryScheduled
QueryRetryStarted
QueryRetrySucceeded
QueryOutcomeUnknown
```

manteniendo la regla:

> **Los Query Events describen el ciclo de vida observable de una consulta; no constituyen un segundo Query Pipeline ni permiten modificar silenciosamente AST, plan, SQL compilado, parámetros o resultado mediante listeners genéricos.**