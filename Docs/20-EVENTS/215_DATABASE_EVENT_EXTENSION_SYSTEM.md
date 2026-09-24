# 215_DATABASE_EVENT_EXTENSION_SYSTEM.md

# VoltStack Quantum Database
## Database Event Extension System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 215 — Database Event Extension System  
**Bloque:** 20 — Events  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `214_DATABASE_PERSISTENCE_EVENT_SYSTEM.md`  
**Siguiente documento:** `216_DATABASE_TELEMETRY_ARCHITECTURE.md`

---

# 1. Propósito

`Database Event Extension System` define cómo aplicaciones, paquetes oficiales de VoltStack y extensiones de terceros podrán ampliar el sistema de eventos de `VoltStack/Quantum/Database` sin romper sus invariantes internas.

El sistema deberá permitir extender:

```text
Database Events
Event Listeners
Event Subscribers
Event Filters
Event Bridges
Event Serializers
Event Sinks
Event Policies
Event Diagnostics
Event Aggregators
Event Adapters
```

manteniendo separadas las responsabilidades de:

```text
Query Engine
Connection System
Transaction System
ORM
Persistence Engine
Telemetry
Security
Runtime
```

La regla central será:

> **Extender el Database Event System significa registrar contratos adicionales de observación e integración sobre boundaries públicos y estables; no significa obtener acceso irrestricto a los estados internos del Database Engine ni alterar silenciosamente Query, Connection, Transaction, ORM o Persistence mediante listeners genéricos.**

---

# 2. Cierre arquitectónico del Bloque 20

Este documento cierra:

```text
209_DATABASE_EVENT_ARCHITECTURE.md
210_DATABASE_QUERY_EVENT_SYSTEM.md
211_DATABASE_CONNECTION_EVENT_SYSTEM.md
212_DATABASE_TRANSACTION_EVENT_PIPELINE.md
213_DATABASE_ENTITY_LIFECYCLE_EVENT_SYSTEM.md
214_DATABASE_PERSISTENCE_EVENT_SYSTEM.md
215_DATABASE_EVENT_EXTENSION_SYSTEM.md
```

La arquitectura completa queda:

```text
Database Components
      │
      ├── Query
      ├── Connection
      ├── Transaction
      ├── Entity Lifecycle
      └── Persistence
      │
      ▼
Database Event Contracts
      │
      ▼
Database Event Dispatcher
      │
      ├── Built-in Listeners
      ├── Application Listeners
      ├── Package Subscribers
      ├── Telemetry Bridge
      ├── Framework Event Bridge
      ├── Diagnostic Observers
      └── External Bridges
```

---

# 3. Objetivos

El sistema deberá permitir:

```text
custom event definitions
custom event descriptors
typed listeners
event subscribers
listener priorities
listener filters
operation-scoped listeners
package event registration
framework bridges
telemetry bridges
external event bridges
safe serializers
custom sinks
event aggregation
custom diagnostics
extension metadata
event compatibility contracts
event capability discovery
```

sin introducir:

```text
global mutable registries
hidden event magic
unbounded payloads
unsafe serialization
runtime state leakage
FQCN-based external protocols
listener-based core mutation
```

---

# 4. No objetivos

El Event Extension System no será responsable de:

```text
query transformation
query optimization
routing decisions
connection selection
transaction control
UnitOfWork mutation
persistence planning
authorization
cache correctness
domain event modeling
durable messaging
telemetry storage
```

---

# 5. Distinciones fundamentales

VoltStack deberá preservar:

```text
Event Extension
≠
Query Extension

Event Extension
≠
Persistence Extension

Event Extension
≠
Driver Extension

Event Extension
≠
Dialect Extension

Event Extension
≠
ORM Hook

Event Extension
≠
Security Policy

Event Extension
≠
Telemetry Provider
```

---

# 6. Event Extension ≠ Query Extension

Si una extensión necesita modificar:

```text
Query AST
Semantic Graph
Query Plan
SQL Compilation
```

deberá utilizar los extension points del Query Engine.

No:

```php
DatabaseEvents::listen(
    QueryPlanningStarted::class,
    function ($event) {
        $event->query()->where(...);
    }
);
```

---

# 7. Event Extension ≠ Persistence Extension

Si se necesita cambiar:

```text
operation ordering
change-set interpretation
persistence plan
batch strategy
```

deberá utilizarse:

```text
Persistence Planner Extension
```

y no listeners observacionales.

---

# 8. Event Extension ≠ ORM Hook

Un `prePersist` mutante pertenece al:

```text
Entity Lifecycle Hook System
```

No al registry genérico de Database Events.

---

# 9. Event Extension ≠ Telemetry Provider

Un provider de Telemetry puede consumir Database Events.

Pero:

```text
Event Extension
≠
Telemetry Backend
```

---

# 10. Tipos de extensiones

VoltStack reconocerá conceptualmente:

```php
enum DatabaseEventExtensionType
{
    case EVENT;
    case LISTENER;
    case SUBSCRIBER;
    case FILTER;
    case BRIDGE;
    case SERIALIZER;
    case SINK;
    case AGGREGATOR;
    case DIAGNOSTIC;
    case POLICY;
}
```

---

# 11. Custom events

Una extensión podrá definir un evento propio:

```php
final readonly class ReplicaPromotionObserved implements DatabaseEvent
{
    public function __construct(
        private DatabaseEventId $eventId,
        private Instant $occurredAt,
        private DatabaseEventContext $context,
        public ReplicaPromotionSummary $promotion,
    ) {}

    public function eventId(): DatabaseEventId
    {
        return $this->eventId;
    }

    public function occurredAt(): Instant
    {
        return $this->occurredAt;
    }

    public function context(): DatabaseEventContext
    {
        return $this->context;
    }
}
```

---

# 12. Custom event requirements

Todo evento extensible deberá declarar:

```text
stable event name
category
schema version if externalizable
volume class
sensitivity classification
propagation policy
default delivery semantics
```

---

# 13. Event descriptor

```php
final readonly class DatabaseEventDescriptor
{
    public function __construct(
        public DatabaseEventName $name,
        public DatabaseEventCategory $category,
        public ?DatabaseEventPhase $phase,
        public EventVolumeClass $volume,
        public EventSensitivity $sensitivity,
        public EventPropagationPolicy $propagation,
        public int $schemaVersion,
    ) {}
}
```

---

# 14. Stable event names

Las extensiones deberán usar nombres estables.

Ejemplo:

```text
extension.w4.storage.replica.promotion.observed
```

No:

```text
W4\Storage\Database\Event\ReplicaPromotionObserved
```

como protocolo externo.

---

# 15. Namespace de extensiones

Formato recomendado:

```text
extension.{vendor}.{package}.{event}
```

Ejemplos:

```text
extension.w4.neuronstorage.provider.selected
extension.acme.database.routing.degraded
extension.vendor.audit.query.flagged
```

---

# 16. Reserved namespaces

VoltStack reservará:

```text
database.*
voltstack.database.*
```

para eventos core oficiales.

---

# 17. Namespace collision

Dos extensiones no podrán registrar:

```text
same stable event name
```

---

# 18. Duplicate event policy

Durante bootstrap:

```text
duplicate event name
↓
configuration/bootstrap failure
```

No deberá resolverse silenciosamente por "último gana".

---

# 19. Event versioning

Un evento externalizable deberá declarar:

```text
schemaVersion
```

separado de:

```text
package version
framework version
```

---

# 20. Version evolution

Ejemplo:

```text
v1
{
  "endpoint": "...",
  "reason": "..."
}

v2
{
  "endpoint": "...",
  "reason": "...",
  "authority_epoch": 17
}
```

---

# 21. Breaking change

Cambios como:

```text
remove field
rename field
change field meaning
change units
change nullability semantics
```

deberán incrementar schema version cuando el contrato externo lo requiera.

---

# 22. Internal custom events

Un evento puramente interno podrá no requerir serializer externo.

---

# 23. Externalizable custom events

Deberán implementar:

```php
interface ExternalizableDatabaseEvent extends DatabaseEvent
{
    public function schemaVersion(): int;
}
```

---

# 24. Custom listeners

Una extensión podrá registrar listeners tipados:

```php
final class ReplicaPromotionListener
{
    public function __invoke(
        ReplicaPromotionObserved $event
    ): void {
        // observation only
    }
}
```

---

# 25. Listener registration

Registro conceptual:

```php
$events->listen(
    ReplicaPromotionObserved::class,
    ReplicaPromotionListener::class,
);
```

---

# 26. Listener descriptors

```php
final readonly class DatabaseListenerDescriptor
{
    public function __construct(
        public ListenerId $id,
        public string $eventType,
        public int $priority,
        public ListenerCriticality $criticality,
        public ListenerReentrancyPolicy $reentrancy,
        public ListenerScope $scope,
    ) {}
}
```

---

# 27. Listener ID

Cada listener deberá poder tener:

```text
stable ListenerId
```

para:

```text
diagnostics
configuration
disable/enable
testing
telemetry
```

---

# 28. Listener ID ≠ FQCN

Se podrá usar FQCN internamente, pero la identidad pública podrá ser:

```text
w4.neuronstorage.replica_listener
```

---

# 29. Listener scope

```php
enum ListenerScope
{
    case GLOBAL_IMMUTABLE;
    case APPLICATION;
    case OPERATION;
    case REQUEST;
    case TEST;
}
```

---

# 30. GLOBAL_IMMUTABLE

Adecuado para:

```text
compiled package listeners
stateless adapters
immutable bridges
```

---

# 31. OPERATION / REQUEST

Adecuado para:

```text
temporary diagnostics
testing
request-specific observation
```

---

# 32. Scope leakage prohibition

Un listener registrado para:

```text
Request A
```

no deberá recibir eventos de:

```text
Request B
```

---

# 33. Event subscribers

Para paquetes con múltiples listeners:

```php
interface DatabaseEventSubscriber
{
    public static function subscriptions(): iterable;
}
```

---

# 34. Subscription descriptor

```php
final readonly class DatabaseEventSubscription
{
    public function __construct(
        public string $event,
        public string $handler,
        public int $priority = 0,
    ) {}
}
```

---

# 35. Subscriber example

```php
final class DatabaseDiagnosticsSubscriber
    implements DatabaseEventSubscriber
{
    public static function subscriptions(): iterable
    {
        yield new DatabaseEventSubscription(
            QueryExecuted::class,
            'onQueryExecuted',
        );

        yield new DatabaseEventSubscription(
            TransactionOutcomeUnknown::class,
            'onUnknownTransaction',
            priority: 100,
        );
    }
}
```

---

# 36. Subscriber ≠ Event Bus

Un subscriber es una conveniencia de registro.

No crea un segundo dispatcher.

---

# 37. Registration pipeline

```text
Core Events
+
Official Packages
+
Application
+
Third-party Packages
↓
Extension Discovery
↓
Validation
↓
Conflict Detection
↓
Registry Compilation
↓
Registry Freeze
```

---

# 38. DatabaseEventExtensionProvider

Paquetes podrán registrar extensiones mediante:

```php
interface DatabaseEventExtensionProvider
{
    public function register(
        DatabaseEventExtensionRegistry $registry
    ): void;
}
```

---

# 39. Provider responsibilities

Podrá registrar:

```text
event descriptors
listeners
subscribers
filters
bridges
serializers
diagnostics
```

---

# 40. Provider must not

No deberá:

```text
execute queries during registration
open connections
begin transactions
mutate EntityManager state
perform network I/O
```

por default.

---

# 41. Bootstrap purity

La fase de registro deberá ser:

```text
deterministic
side-effect bounded
cacheable when possible
```

---

# 42. Registry

```php
interface DatabaseEventExtensionRegistry
{
    public function event(
        DatabaseEventDescriptor $descriptor,
        string $eventClass,
    ): void;

    public function listener(
        DatabaseListenerDescriptor $listener,
    ): void;

    public function subscriber(
        string $subscriberClass,
    ): void;
}
```

---

# 43. Registry compilation

Después del bootstrap:

```text
Mutable Registration Registry
↓
Validation
↓
CompiledDatabaseEventRegistry
↓
Freeze
```

---

# 44. Frozen registry

En producción/persistent runtimes deberá preferirse:

```text
immutable compiled registry
```

---

# 45. Runtime registration

Solo deberá permitirse en scopes explícitos.

---

# 46. Event filters

Una extensión podrá registrar:

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

# 47. Filter use cases

Válidos:

```text
ignore internal health-check queries
observe only writes
observe only transaction failures
observe selected event categories
development-only diagnostics
```

---

# 48. Filter ≠ Authorization

Nunca:

```text
filter rejected event
=
operation unauthorized
```

---

# 49. Filter cannot rewrite operation

Un filter decide:

```text
listener receives event?
```

No:

```text
database operation executes?
```

---

# 50. Filter ordering

Deberá ser:

```text
deterministic
```

y documentado.

---

# 51. Filter composition

Conceptualmente:

```text
Global Filter
AND
Listener Filter
AND
Scope Filter
AND
Security Filter
```

para delivery.

---

# 52. Security filtering

Podrá ocultar eventos/payloads según:

```text
listener privilege
environment
tenant scope
event sensitivity
```

---

# 53. Payload policy

```php
interface DatabaseEventPayloadPolicy
{
    public function transform(
        DatabaseEvent $event,
        DatabaseListenerDescriptor $listener,
    ): DatabaseEventView;
}
```

---

# 54. EventView ≠ Event mutation

El core event permanece inmutable.

El listener puede recibir una vista sanitizada.

---

# 55. DatabaseEventView

```php
final readonly class DatabaseEventView
{
    public function __construct(
        public DatabaseEventName $name,
        public DatabaseEventContext $context,
        public array $payload,
    ) {}
}
```

---

# 56. Privileged local listeners

Algunos listeners internos podrán recibir:

```text
full in-process typed event
```

según contract.

---

# 57. External listeners

Deberán recibir:

```text
sanitized serialized representation
```

---

# 58. Event bridge

Una extensión podrá conectar Database Events con otro sistema.

```php
interface DatabaseEventBridge
{
    public function publish(
        DatabaseEvent $event,
        DatabaseEventBridgeContext $context,
    ): void;
}
```

---

# 59. Bridge examples

```text
VoltStack global Event System
Telemetry
OpenTelemetry adapter
Audit pipeline
Queue/message broker
debug toolbar
external monitoring
```

---

# 60. Bridge ≠ listener with unrestricted access

Un bridge tendrá un contrato específico y policy de payload.

---

# 61. Bridge direction

Principalmente:

```text
Database Events
↓
External System
```

---

# 62. Reverse bridge

Introducir eventos externos dentro de Database deberá ser una feature distinta.

No deberá permitirse automáticamente.

---

# 63. Why

Evita:

```text
untrusted external message
↓
instantiate arbitrary internal database event
↓
listener magic
```

---

# 64. FrameworkEventBridge

Arquitectura:

```text
Database Event
↓
Framework Event Bridge
↓
VoltStack EventSystem
↓
Application Listeners
```

---

# 65. Core independence

`Quantum/Database` seguirá funcionando con:

```text
NullDatabaseEventDispatcher
```

sin EventSystem global.

---

# 66. TelemetryEventBridge

```text
Database Event
↓
Telemetry Bridge
↓
Metric / Span / Log
```

---

# 67. Telemetry bridge may sample

Puede aplicar:

```text
sampling
aggregation
cardinality reduction
redaction
```

---

# 68. Telemetry bridge must not

No deberá:

```text
change query
change transaction
change connection
change entity
```

---

# 69. Async bridge

```text
Database Event
↓
Async Bridge
↓
Serializable Envelope
↓
Queue/Broker
```

---

# 70. Async ≠ durable by default

Si el bridge hace simplemente:

```text
afterCommit callback
→ publish
```

puede perder el mensaje por crash.

---

# 71. Durable bridge

Para garantías fuertes:

```text
Database Event / Domain Event
↓
Outbox
↓
Commit
↓
Publisher
```

---

# 72. Event bridge delivery guarantees

```php
enum EventBridgeDeliveryGuarantee
{
    case BEST_EFFORT;
    case IN_PROCESS;
    case DURABLE_AT_LEAST_ONCE;
    case CUSTOM;
}
```

---

# 73. No generic EXACTLY_ONCE

No se prometerá:

```text
EXACTLY_ONCE
```

sin un protocolo end-to-end explícito.

---

# 74. Event sinks

Un sink consume eventos para almacenarlos/procesarlos.

```php
interface DatabaseEventSink
{
    public function write(
        DatabaseEventRecord $record
    ): void;
}
```

---

# 75. Sink examples

```text
diagnostic buffer
audit store
log sink
development timeline
file sink
testing recorder
```

---

# 76. Sink ≠ event dispatcher

Dispatcher decide delivery.

Sink consume una representación.

---

# 77. Sink failure

No deberá reescribir el outcome de Database salvo policy explícita y boundary seguro.

---

# 78. Default sink criticality

```text
OBSERVATIONAL
```

---

# 79. Serializer extension

Eventos externalizables requerirán:

```php
interface DatabaseEventSerializer
{
    public function serialize(
        ExternalizableDatabaseEvent $event,
    ): SerializedDatabaseEvent;
}
```

---

# 80. Deserializer caution

Si se soporta ingestión:

```php
interface DatabaseEventDeserializer
{
    public function deserialize(
        SerializedDatabaseEvent $event,
    ): ExternalDatabaseEventView;
}
```

No deberá instanciar clases arbitrarias.

---

# 81. Stable type registry

La serialización deberá utilizar:

```text
stable event names
stable schema versions
stable TypeIds
```

---

# 82. Unsafe PHP serialization

Prohibido:

```php
serialize($event);
unserialize($payload);
```

como formato externo general.

---

# 83. Safe formats

Podrán soportarse:

```text
JSON
MessagePack
CBOR
custom typed format
```

si respetan schemas y límites.

---

# 84. Serializer responsibilities

```text
schema validation
type normalization
redaction
size limits
depth limits
version handling
```

---

# 85. Serializer must not

No deberá activar:

```text
lazy loading
queries
entity hydration
service lookup
```

durante serialization.

---

# 86. Event serialization side effects

Regla:

> **Serializar un Database Event debe ser una operación de observación pura respecto al Database subsystem.**

---

# 87. Event aggregator

Para eventos de alto volumen:

```php
interface DatabaseEventAggregator
{
    public function consume(DatabaseEvent $event): void;

    public function flush(): iterable;
}
```

---

# 88. Aggregator use cases

Ejemplo:

```text
10,000 QueryExecuted
↓
aggregate
↓
{
  query_count: 10000,
  total_duration: ...,
  slow_queries: ...
}
```

---

# 89. Aggregation ≠ functional event delivery

Un listener funcional no deberá depender de un aggregate si requiere eventos individuales.

---

# 90. Aggregator scope

Podrá ser:

```text
request
operation
transaction
time window
worker
```

según tipo.

---

# 91. Worker aggregate caution

En persistent runtimes, agregadores worker-scoped deberán evitar mezclar:

```text
tenant/request sensitive data
```

sin separación.

---

# 92. Aggregator reset

Deberá ocurrir en boundary definido.

---

# 93. Event middleware

Podrá existir una abstracción controlada:

```php
interface DatabaseEventMiddleware
{
    public function process(
        DatabaseEventEnvelope $event,
        DatabaseEventMiddlewareNext $next,
    ): DatabaseEventDispatchResult;
}
```

---

# 94. Middleware responsibilities

Permitidas:

```text
redaction
diagnostics
delivery timing
listener timing
bridge routing
sampling
bounded metadata
```

---

# 95. Middleware restrictions

No deberá usar el evento para modificar el Database operation original.

---

# 96. Event middleware ≠ database middleware

Es middleware de delivery.

No del Query/Transaction pipeline.

---

# 97. Middleware ordering

Deberá ser explícito y determinista.

Ejemplo:

```text
Security
↓
Redaction
↓
Filtering
↓
Diagnostics
↓
Dispatch
↓
Bridge
```

---

# 98. Recommended core middleware order

```text
Context Validation
↓
Security Policy
↓
Payload Sanitization
↓
Event Filtering
↓
Dispatch Budget
↓
Listener Dispatch
↓
External Bridges
↓
Diagnostics Finalization
```

---

# 99. Middleware recursion

Un middleware podrá causar nuevos events indirectamente.

Deberá respetar:

```text
DispatchStack
ReentrancyPolicy
MaxDepth
```

---

# 100. Event policy extensions

Una extensión podrá proveer policies como:

```text
event enabled?
externalizable?
sampling rate?
sensitive field exposure?
listener timeout?
```

---

# 101. Policy ≠ core behavior policy

Ejemplo:

```text
QueryEventExposurePolicy
```

es válido.

Pero:

```text
WhichReplicaShouldQueryUse
```

no pertenece al Event Extension System.

---

# 102. DatabaseEventPolicy contract

```php
interface DatabaseEventPolicy
{
    public function evaluate(
        DatabaseEventDescriptor $event,
        DatabaseEventPolicyContext $context,
    ): DatabaseEventPolicyDecision;
}
```

---

# 103. Policy decisions

```php
enum DatabaseEventPolicyDecision
{
    case ALLOW;
    case DENY_DELIVERY;
    case SANITIZE;
    case SAMPLE;
}
```

---

# 104. DENY_DELIVERY ≠ deny database operation

Distinción fundamental.

---

# 105. Extension capabilities

Cada extensión deberá declarar lo que necesita.

```php
enum DatabaseEventExtensionCapability
{
    case REGISTER_EVENT;
    case REGISTER_LISTENER;
    case REGISTER_SUBSCRIBER;
    case REGISTER_FILTER;
    case REGISTER_BRIDGE;
    case REGISTER_SERIALIZER;
    case REGISTER_SINK;
    case REGISTER_AGGREGATOR;
    case REGISTER_DIAGNOSTIC;
}
```

---

# 106. Least capability principle

Una extensión solo deberá recibir las capacidades declaradas.

---

# 107. Extension context

```php
final readonly class DatabaseEventExtensionContext
{
    public function __construct(
        public DatabaseEventRegistryView $events,
        public DatabaseEventConfiguration $configuration,
        public DatabaseEventExtensionCapabilities $capabilities,
    ) {}
}
```

---

# 108. No EntityManager in extension bootstrap

No proporcionar:

```text
EntityManager
ConnectionManager
TransactionManager
```

por default al extension registration context.

---

# 109. Why

Reduce:

```text
bootstrap side effects
hidden coupling
runtime leakage
```

---

# 110. Package identity

Cada extensión deberá declarar:

```text
vendor
package
version
extension ID
```

---

# 111. DatabaseEventExtensionId

Ejemplo:

```text
w4.neuronstorage.database-events
```

---

# 112. Extension metadata

```php
final readonly class DatabaseEventExtensionMetadata
{
    public function __construct(
        public DatabaseEventExtensionId $id,
        public string $version,
        public array $requires,
        public array $capabilities,
    ) {}
}
```

---

# 113. Dependency declarations

Una extensión podrá requerir:

```text
specific event contract version
specific database capability
specific package
```

---

# 114. Event extension dependency graph

```text
Core
├── Extension A
├── Extension B
│   └── requires A
└── Extension C
```

deberá resolverse determinísticamente.

---

# 115. Circular dependencies

Deberán rechazarse en bootstrap.

---

# 116. Optional dependencies

Podrán declararse explícitamente.

---

# 117. Missing optional package

No deberá romper el Event System si la extensión puede operar sin él.

---

# 118. Event capability discovery

Una extensión podrá consultar:

```php
$registry->supportsEvent(
    QueryExecuted::class
);
```

o stable event name.

---

# 119. Event capability ≠ listener registered

Separar:

```text
event available
```

de:

```text
listener present
```

---

# 120. Extension compatibility

Cada package podrá declarar:

```text
minimum Database Event API version
maximum tested API version
required event schemas
```

---

# 121. DatabaseEventApiVersion

```php
final readonly class DatabaseEventApiVersion
{
    public function __construct(
        public int $major,
        public int $minor,
    ) {}
}
```

---

# 122. Compatibility rule

Cambios breaking en contratos públicos deberán respetar versionado.

---

# 123. Deprecation

Un evento/listener API podrá marcarse:

```text
deprecated
```

antes de eliminarse.

---

# 124. Deprecated event alias

Podrá existir temporalmente:

```text
old stable event name
→
new descriptor
```

solo si la semántica sigue siendo equivalente.

---

# 125. No misleading alias

Si la semántica cambia:

```text
do not alias
```

Deberá crearse nueva versión/evento.

---

# 126. Built-in event protection

Extensiones no podrán reemplazar silenciosamente:

```text
database.query.executed
database.transaction.committed
```

---

# 127. Built-in event override

Por default:

```text
FORBIDDEN
```

---

# 128. Listener addition allowed

Sí podrán añadir listeners a eventos built-in.

---

# 129. Event descriptor mutation forbidden

Una extensión no podrá cambiar:

```text
category
meaning
schema version
sensitivity
```

de un evento core existente.

---

# 130. Core invariant protection

La registry validation deberá comparar:

```text
registered descriptor
vs
built-in immutable descriptor
```

---

# 131. Reserved listener priority ranges

Podrá considerarse:

```text
1000..1999 core
500..999 official packages
0..499 application
negative low-priority observers
```

pero esto no deberá ser obligatorio si aumenta complejidad.

---

# 132. Better recommendation

Preferir:

```text
explicit numeric priority
+
deterministic tie-breaker
```

sin reservar demasiada semántica global.

---

# 133. Listener tie-breaker

Podrá ser:

```text
extension ID
listener ID
registration sequence at compile time
```

determinista.

---

# 134. Listener dependencies

Si un listener realmente requiere ejecutarse:

```text
after X
before Y
```

podrá declararlo explícitamente.

---

# 135. Ordering graph

```php
final readonly class ListenerOrderingConstraint
{
    public function __construct(
        public array $before = [],
        public array $after = [],
    ) {}
}
```

---

# 136. Cyclic ordering

Deberá fallar en bootstrap.

---

# 137. Avoid hidden semantic dependency

Los listeners observacionales no deberían depender fuertemente de orden entre sí.

---

# 138. Extension listener failure

Cada listener deberá declarar:

```text
criticality
failure policy
```

o heredar defaults.

---

# 139. Failure policies

```php
enum ExtensionListenerFailurePolicy
{
    case RECORD_AND_CONTINUE;
    case PROPAGATE;
    case DISABLE_FOR_SCOPE;
    case CUSTOM;
}
```

---

# 140. Default

```text
RECORD_AND_CONTINUE
```

para observers.

---

# 141. PROPAGATE caution

Solo deberá permitirse donde el lifecycle boundary todavía puede abortarse de forma segura.

---

# 142. Post-commit propagation

Un listener after-commit que falla:

```text
cannot uncommit
```

---

# 143. Extension health

Un listener que falla repetidamente podrá:

```text
be disabled for scope
emit health diagnostic
```

---

# 144. Circuit breaker for listeners

Podrá existir como integración:

```text
listener health policy
```

pero no deberá afectar Database operation correctness.

---

# 145. Extension isolation

Una extensión defectuosa no deberá:

```text
corrupt event registry
corrupt transaction state
corrupt connection state
```

---

# 146. Exception isolation

El dispatcher deberá capturar exceptions según failure policy.

---

# 147. Listener timeout

Listeners externos/lentos podrán tener:

```text
execution budget
```

---

# 148. In-process hard timeout limitation

PHP no siempre puede interrumpir arbitrariamente un listener de forma segura.

Por tanto:

```text
listener timeout
```

puede ser:

```text
diagnostic budget
async boundary
cooperative cancellation
```

según runtime.

---

# 149. No false timeout guarantees

No prometer:

```text
hard preemption
```

si runtime no lo soporta.

---

# 150. Dispatch budget

```php
final readonly class DatabaseEventDispatchBudget
{
    public function __construct(
        public ?int $maxListeners,
        public ?Duration $maxDuration,
        public ?int $maxDepth,
    ) {}
}
```

---

# 151. Max listener count

Previene accidental:

```text
10,000 listeners
```

en un hot-path event.

---

# 152. Max recursion depth

Protege contra loops.

---

# 153. Event recursion example

```text
QueryExecuted
↓
Audit Listener
↓
INSERT audit
↓
QueryExecuted
↓
Audit Listener
↓
...
```

---

# 154. Origin-based filtering

Una extensión podrá declarar:

```text
ignore events where origin = AUDIT
```

---

# 155. Reentrancy policy

```php
enum ExtensionReentrancyPolicy
{
    case ALLOW;
    case DENY_SELF;
    case DENY_SAME_EVENT;
    case DENY_DATABASE_REENTRY;
    case CUSTOM;
}
```

---

# 156. Database reentry

Un listener puede ejecutar otra DB operation.

Esto deberá ser explícito.

---

# 157. Listener capability declaration

```php
enum ListenerDatabaseAccess
{
    case NONE;
    case READ;
    case WRITE;
    case UNKNOWN;
}
```

---

# 158. Diagnostic value

VoltStack podrá advertir:

```text
TransactionCommitted listener performs DATABASE_WRITE.
```

o:

```text
QueryExecuted listener reenters Database.
```

---

# 159. Listener side-effect classification

```php
enum ListenerSideEffectClass
{
    case PURE;
    case DATABASE_READ;
    case DATABASE_WRITE;
    case EXTERNAL_IO;
    case UNKNOWN;
}
```

---

# 160. Retry-aware diagnostics

Side-effecting listeners dentro de retryable transactions deberán advertirse.

---

# 161. Event extension security model

Las extensiones tendrán acceso a información potencialmente sensible.

Por tanto se requerirá:

```text
payload minimization
redaction
scope validation
tenant isolation
safe serialization
least authority
```

---

# 162. Tenant-aware listeners

Un listener podrá declarar:

```text
TENANT_LOCAL
GLOBAL_ADMINISTRATIVE
TENANT_AGNOSTIC
```

---

# 163. Tenant scope

```php
enum EventListenerTenantScope
{
    case TENANT_LOCAL;
    case TENANT_AGNOSTIC;
    case GLOBAL_AUTHORIZED;
}
```

---

# 164. Default

Aplicaciones tenant-aware deberán preferir:

```text
TENANT_LOCAL
```

---

# 165. Tenant leakage prevention

El dispatcher deberá verificar:

```text
event tenant context
listener scope
authorization/policy
```

antes de delivery sensible.

---

# 166. GLOBAL_AUTHORIZED

Debe requerir configuración explícita.

---

# 167. Shard-aware extension

Listeners podrán recibir:

```text
ShardId
```

si policy permite.

---

# 168. Credentials prohibited

Nunca entregar:

```text
DB password
private key
raw DSN with secrets
access token
```

a listeners.

---

# 169. Query parameters

Por default:

```text
REDACTED
```

---

# 170. Entity values

Por default:

```text
NOT INCLUDED
```

---

# 171. Extension secrets

Una extensión podrá tener sus propios secrets via config/container.

No deberán almacenarse en event metadata.

---

# 172. Event metadata namespaces

Extensiones deberán usar:

```text
extension.{vendor}.{package}.*
```

para metadata custom.

---

# 173. Metadata collision

Core podrá rechazar claves reservadas.

---

# 174. Metadata type safety

Valores permitidos:

```text
string
int
float
bool
null
bounded arrays of primitives
stable IDs
```

---

# 175. Live objects prohibited in external metadata

No:

```text
PDO
EntityManager
Connection
Entity
Closure
Resource
Generator
```

---

# 176. Metadata size budget

```php
final readonly class EventMetadataBudget
{
    public function __construct(
        public int $maxKeys,
        public int $maxKeyLength,
        public int $maxValueBytes,
        public int $maxTotalBytes,
        public int $maxDepth,
    ) {}
}
```

---

# 177. Oversized metadata

Deberá:

```text
reject
truncate through explicit policy
or sanitize
```

Nunca crecer silenciosamente sin límite.

---

# 178. Event payload schema validation

Externalizable extensions deberán registrar schemas.

---

# 179. Schema registry

```php
interface DatabaseEventSchemaRegistry
{
    public function register(
        DatabaseEventName $name,
        int $version,
        DatabaseEventSchema $schema,
    ): void;
}
```

---

# 180. Schema validation

Debe ocurrir antes de externalización.

---

# 181. Invalid payload

No deberá publicarse externamente.

---

# 182. Event bridge routing

Una extensión podrá declarar:

```text
which events
which bridge
which conditions
```

---

# 183. Route descriptor

```php
final readonly class DatabaseEventBridgeRoute
{
    public function __construct(
        public DatabaseEventSelector $selector,
        public BridgeId $bridge,
        public EventBridgeDeliveryGuarantee $guarantee,
    ) {}
}
```

---

# 184. Event selector

Podrá seleccionar por:

```text
event name
category
origin
phase
volume class
sensitivity
```

---

# 185. Avoid arbitrary PHP callbacks in compiled registry

Para caches/compilation, preferir descriptors serializables.

---

# 186. Application closures

Podrán permitirse en development/runtime local, pero:

```text
not compile-safe
not externalizable
not persistent-registry friendly
```

---

# 187. Production recommendation

Usar:

```text
listener classes
subscriber classes
immutable descriptors
```

---

# 188. Dependency injection

Listeners/subscribers deberán resolverse por Container.

---

# 189. Listener lifecycle

Podrán ser:

```text
singleton stateless
operation scoped
transient
```

---

# 190. Singleton listener restriction

Solo si:

```text
stateless
thread/coroutine safe
no request-specific mutable fields
```

---

# 191. Persistent runtime danger

Incorrecto:

```php
final class QueryListener
{
    private array $queries = [];

    public function __invoke(QueryExecuted $event): void
    {
        $this->queries[] = $event;
    }
}
```

si el listener singleton persiste entre requests.

---

# 192. Safer approach

Estado mutable deberá vivir en:

```text
request/operation scoped collector
```

---

# 193. FrankenPHP

El registry compilado podrá persistir.

Los collectors/listener state mutables no.

---

# 194. RoadRunner

Cada job deberá recibir:

```text
fresh event scope
fresh operation collectors
fresh temporary listeners
```

---

# 195. OpenSwoole

Mutable listener state deberá ser:

```text
coroutine-safe
```

o scope-local.

---

# 196. Event bridge connections

Bridges que mantengan conexiones externas podrán ser persistent services si son seguros.

---

# 197. Bridge request context

Nunca deberán almacenar en fields compartidos:

```text
current tenant
current request
current transaction
```

---

# 198. Context must be passed

```php
$bridge->publish(
    event: $event,
    context: $bridgeContext,
);
```

---

# 199. Extension reset contract

Podrá existir:

```php
interface ResettableDatabaseEventExtension
{
    public function reset(
        DatabaseEventResetContext $context
    ): void;
}
```

---

# 200. Reset use cases

Para:

```text
operation-scoped caches
diagnostic buffers
temporary aggregation
```

---

# 201. Reset ≠ core DB reset

No confundir con:

```text
Connection reset
EntityManager clear
Transaction cleanup
```

---

# 202. Extension cleanup

Debe ejecutarse al cerrar scope.

---

# 203. Cleanup failure

Deberá ser observable.

Pero no deberá contaminar el siguiente request.

---

# 204. Testing extensions

El sistema deberá permitir:

```text
fake listener
fake bridge
fake serializer
fake sink
fake subscriber
```

---

# 205. Test listener

```php
final class RecordingListener
{
    public array $events = [];

    public function __invoke(DatabaseEvent $event): void
    {
        $this->events[] = $event;
    }
}
```

solo en test scope.

---

# 206. Test isolation

Cada test deberá recibir registry/scope aislado cuando modifique listeners runtime.

---

# 207. Assert listener registered

```php
DatabaseEvents::assertListenerRegistered(
    QueryExecuted::class,
    QueryMetricsListener::class,
);
```

---

# 208. Assert bridge route

```php
DatabaseEvents::assertBridgeConfigured(
    TransactionOutcomeUnknown::class,
    OperationsAlertBridge::class,
);
```

---

# 209. Registry snapshot testing

Podrá generarse un snapshot determinista de:

```text
events
listeners
priorities
bridges
schemas
```

---

# 210. Registry snapshot use

Útil para detectar cambios accidentales de configuración.

---

# 211. Extension validation

Bootstrap deberá validar:

```text
event name uniqueness
schema version validity
listener signatures
subscriber methods
priority graph
ordering cycles
bridge availability
serializer compatibility
scope compatibility
security policy
capability declarations
```

---

# 212. Validation failure

Debe fallar rápido.

---

# 213. Fail fast

Es preferible:

```text
bootstrap error
```

a:

```text
runtime production surprise
```

---

# 214. Lazy extension discovery

En entornos donde bootstrap rápido sea crítico, podrán cachearse manifests compilados.

---

# 215. Event manifest

```php
final readonly class DatabaseEventExtensionManifest
{
    public function __construct(
        public array $events,
        public array $listeners,
        public array $subscribers,
        public array $bridges,
        public array $serializers,
    ) {}
}
```

---

# 216. Manifest cache

Podrá almacenarse durante deployment/build.

---

# 217. Manifest invalidation

Debe depender de:

```text
framework version
package versions
event API version
configuration generation
```

---

# 218. Manifest ≠ runtime mutable state

Debe ser immutable.

---

# 219. Package unload

En runtimes normales, la registry se considera congelada.

No será requisito soportar hot-unload dinámico de paquetes.

---

# 220. Development hot reload

Podrá reconstruir el registry completo.

No modificar parcialmente uno congelado.

---

# 221. Extension diagnostics

API conceptual:

```php
DB::events()->extensions()->explain();
```

---

# 222. Example diagnostics

```text
DATABASE EVENT EXTENSIONS

Extension:
  w4.neuronstorage.database-events

Version:
  1.4.0

Capabilities:
  REGISTER_EVENT
  REGISTER_LISTENER
  REGISTER_BRIDGE

Events:
  extension.w4.neuronstorage.provider.selected@1
  extension.w4.neuronstorage.provider.migrated@1

Listeners:
  QueryExecuted
    → StorageQueryDiagnosticsListener
      priority: 10
      criticality: OBSERVATIONAL

Bridges:
  TransactionOutcomeUnknown
    → OperationsAlertBridge
      delivery: BEST_EFFORT

Security:
  SQL values: REDACTED
  Tenant scope: TENANT_LOCAL

Runtime:
  registry: COMPILED
  mutable state: OPERATION_SCOPED
```

---

# 223. Event graph diagnostics

Podrá mostrarse:

```text
QueryExecuted
├── SlowQueryListener
├── ORMProfilerSubscriber::onQuery
├── TelemetryBridge
└── DebugToolbarBridge
```

---

# 224. Dispatch explain

Conceptualmente:

```php
DB::events()->explainDispatch(
    QueryExecuted::class
);
```

---

# 225. Explain output

```text
EVENT DISPATCH PLAN

Event:
  database.query.executed

Category:
  QUERY

Volume:
  HIGH

Listeners:
  1. QueryProfilerListener
     priority: 100
     scope: APPLICATION

  2. DebugToolbarListener
     priority: 0
     scope: REQUEST

Filters:
  IgnoreHealthCheckQueries

Middleware:
  Security
  Sanitizer
  DispatchBudget

Bridges:
  Telemetry

Externalization:
  disabled

Max Depth:
  8
```

---

# 226. Extension performance

High-frequency events requieren extremo cuidado.

---

# 227. Hot-path event examples

```text
QueryExecuted
ConnectionAcquired
ConnectionReleased
EntityHydrated
```

---

# 228. Listener lookup

Registry compilado deberá permitir búsqueda aproximada:

```text
O(1)
```

por event type/name cuando sea práctico.

---

# 229. Empty fast path

Si:

```text
no listeners
no bridges
no diagnostics
```

el costo deberá ser mínimo.

---

# 230. Subscriber expansion

Durante compilación:

```text
Subscriber
↓
static subscriptions
↓
individual listener descriptors
```

evitando reflexión repetida en runtime.

---

# 231. Attribute discovery

VoltStack podrá soportar:

```php
#[ListenTo(QueryExecuted::class, priority: 50)]
```

pero deberá compilarse durante bootstrap cuando sea posible.

---

# 232. Reflection hot path

No deberá utilizarse en cada dispatch.

---

# 233. Listener instantiation

Podrá ser lazy.

No instanciar cientos de listeners si el evento nunca ocurre.

---

# 234. Bridge instantiation

Igualmente podrá resolverse bajo demanda.

---

# 235. Event serialization cost

No serializar un evento si ningún bridge externo lo consume.

---

# 236. Security transformation cost

Sanitización pesada podrá hacerse solo para listeners/bridges que requieran vistas externas.

---

# 237. Local trusted listener path

Podrá evitar serialization completa.

---

# 238. Extension telemetry

El Event Extension System podrá medir:

```text
listener executions
listener failures
listener duration
bridge publications
bridge failures
filtered events
sampled events
recursion prevention
```

---

# 239. Recommended metrics

```text
db.event.dispatch.count
db.event.listener.duration
db.event.listener.failure
db.event.bridge.publish
db.event.bridge.failure
db.event.filtered
db.event.recursion.prevented
```

---

# 240. Cardinality

No usar como labels:

```text
EventId
RequestId
TransactionId
QueryId
TenantId
```

---

# 241. Extension ID label

Puede ser aceptable si el número de extensiones es bounded.

---

# 242. Listener ID label

Igual, si registry es bounded.

---

# 243. Logging

Logs de listener failure deberán incluir:

```text
event name
listener ID
extension ID
failure category
```

sin payload sensible por default.

---

# 244. Event extension error hierarchy

```text
DatabaseEventExtensionException
├── EventExtensionRegistrationException
├── EventExtensionConflictException
├── EventExtensionValidationException
├── EventExtensionCompatibilityException
├── EventExtensionCapabilityException
├── EventExtensionSecurityException
├── EventExtensionSerializationException
├── EventExtensionBridgeException
├── EventExtensionListenerException
├── EventExtensionOrderingException
├── EventExtensionScopeException
└── EventExtensionRuntimeException
```

---

# 245. Registration exception

Ejemplo:

```text
listener signature invalid
```

---

# 246. Conflict exception

Ejemplo:

```text
duplicate stable event name
```

---

# 247. Compatibility exception

Ejemplo:

```text
extension requires Event API v3
framework provides v2
```

---

# 248. Capability exception

Ejemplo:

```text
extension attempts to register bridge
without REGISTER_BRIDGE capability
```

---

# 249. Ordering exception

Ejemplo:

```text
A after B
B after A
```

---

# 250. Scope exception

Ejemplo:

```text
request-scoped listener inserted into immutable global singleton registry incorrectly
```

---

# 251. Directory structure

```text
src/Quantum/Database/Event/Extension/
│
├── Contract/
│   ├── DatabaseEventExtensionProvider.php
│   ├── DatabaseEventExtensionRegistry.php
│   ├── DatabaseEventSubscriber.php
│   ├── DatabaseEventBridge.php
│   ├── DatabaseEventSerializer.php
│   ├── DatabaseEventSink.php
│   ├── DatabaseEventAggregator.php
│   ├── DatabaseEventMiddleware.php
│   ├── DatabaseEventPolicy.php
│   └── ResettableDatabaseEventExtension.php
│
├── Registry/
│   ├── MutableDatabaseEventExtensionRegistry.php
│   ├── CompiledDatabaseEventExtensionRegistry.php
│   ├── DatabaseEventExtensionRegistryCompiler.php
│   ├── DatabaseEventExtensionRegistryValidator.php
│   ├── DatabaseEventSchemaRegistry.php
│   └── DatabaseEventExtensionManifest.php
│
├── Model/
│   ├── DatabaseEventExtensionId.php
│   ├── DatabaseEventExtensionType.php
│   ├── DatabaseEventExtensionMetadata.php
│   ├── DatabaseEventExtensionCapability.php
│   ├── DatabaseEventApiVersion.php
│   ├── DatabaseListenerDescriptor.php
│   ├── ListenerId.php
│   ├── ListenerScope.php
│   ├── ListenerCriticality.php
│   ├── ListenerSideEffectClass.php
│   ├── ListenerDatabaseAccess.php
│   ├── EventListenerTenantScope.php
│   └── ListenerOrderingConstraint.php
│
├── Subscription/
│   ├── DatabaseEventSubscription.php
│   ├── SubscriberCompiler.php
│   └── SubscriberValidator.php
│
├── Filter/
│   ├── DatabaseEventFilter.php
│   ├── DatabaseEventSelector.php
│   └── EventFilterChain.php
│
├── Middleware/
│   ├── DatabaseEventMiddlewarePipeline.php
│   ├── DatabaseEventMiddlewareNext.php
│   ├── EventSecurityMiddleware.php
│   ├── EventSanitizationMiddleware.php
│   ├── EventFilteringMiddleware.php
│   ├── EventDispatchBudgetMiddleware.php
│   └── EventDiagnosticsMiddleware.php
│
├── Bridge/
│   ├── DatabaseEventBridgeContext.php
│   ├── DatabaseEventBridgeRoute.php
│   ├── EventBridgeDeliveryGuarantee.php
│   ├── FrameworkEventBridge.php
│   ├── TelemetryEventBridge.php
│   └── AsyncEventBridge.php
│
├── Serialization/
│   ├── SerializedDatabaseEvent.php
│   ├── DatabaseEventSchema.php
│   ├── DatabaseEventPayloadPolicy.php
│   ├── DatabaseEventView.php
│   └── SafeDatabaseEventSerializer.php
│
├── Aggregation/
│   ├── DatabaseEventAggregatorRegistry.php
│   ├── EventAggregationContext.php
│   └── EventAggregationResult.php
│
├── Runtime/
│   ├── DatabaseEventExtensionScope.php
│   ├── DatabaseEventResetContext.php
│   ├── DatabaseEventDispatchBudget.php
│   └── EventMetadataBudget.php
│
├── Security/
│   ├── DatabaseEventExtensionSecurityPolicy.php
│   ├── DatabaseEventListenerAccessPolicy.php
│   └── DatabaseEventPayloadSecurityPolicy.php
│
├── Diagnostics/
│   ├── DatabaseEventExtensionInspector.php
│   ├── DatabaseEventDispatchExplainer.php
│   ├── DatabaseEventGraph.php
│   └── DatabaseEventExtensionHealth.php
│
├── Testing/
│   ├── FakeDatabaseEventBridge.php
│   ├── FakeDatabaseEventSink.php
│   ├── FakeDatabaseEventSubscriber.php
│   ├── DatabaseEventRegistryAssertions.php
│   └── DatabaseEventExtensionSnapshot.php
│
└── Exception/
    ├── DatabaseEventExtensionException.php
    ├── EventExtensionRegistrationException.php
    ├── EventExtensionConflictException.php
    ├── EventExtensionValidationException.php
    ├── EventExtensionCompatibilityException.php
    ├── EventExtensionCapabilityException.php
    ├── EventExtensionSecurityException.php
    ├── EventExtensionSerializationException.php
    ├── EventExtensionBridgeException.php
    ├── EventExtensionListenerException.php
    ├── EventExtensionOrderingException.php
    ├── EventExtensionScopeException.php
    └── EventExtensionRuntimeException.php
```

---

# 252. Extension registration flow

```text
Package Service Provider
        │
        ▼
DatabaseEventExtensionProvider
        │
        ▼
Mutable Registry
        │
        ├── Events
        ├── Listeners
        ├── Subscribers
        ├── Filters
        ├── Bridges
        ├── Serializers
        └── Sinks
        │
        ▼
Validation
        │
        ▼
Conflict Detection
        │
        ▼
Dependency Ordering
        │
        ▼
Registry Compilation
        │
        ▼
Immutable Runtime Registry
```

---

# 253. Dispatch flow con extensiones

```text
Database Component
       │
       ▼
DatabaseEvent
       │
       ▼
DatabaseEventDispatcher
       │
       ▼
Extension Middleware Pipeline
       │
       ├── Security
       ├── Sanitization
       ├── Filtering
       └── Budget
       │
       ▼
Compiled Listener Registry
       │
       ├── Core Listeners
       ├── Package Listeners
       └── App Listeners
       │
       ▼
Bridges / Sinks / Aggregators
```

---

# 254. Package example

Supongamos un paquete:

```text
voltstack/database-audit
```

Registro:

```php
final class DatabaseAuditEventProvider
    implements DatabaseEventExtensionProvider
{
    public function register(
        DatabaseEventExtensionRegistry $registry
    ): void {
        $registry->subscriber(
            DatabaseAuditSubscriber::class
        );

        $registry->event(
            descriptor: new DatabaseEventDescriptor(
                name: new DatabaseEventName(
                    'extension.voltstack.audit.anomaly.detected'
                ),
                category: DatabaseEventCategory::EXTENSION,
                phase: null,
                volume: EventVolumeClass::LOW,
                sensitivity: EventSensitivity::SENSITIVE,
                propagation: EventPropagationPolicy::DATABASE,
                schemaVersion: 1,
            ),
            eventClass: DatabaseAuditAnomalyDetected::class,
        );
    }
}
```

---

# 255. Subscriber example

```php
final class DatabaseAuditSubscriber
    implements DatabaseEventSubscriber
{
    public static function subscriptions(): iterable
    {
        yield new DatabaseEventSubscription(
            QueryExecuted::class,
            'onQueryExecuted',
            priority: -100,
        );

        yield new DatabaseEventSubscription(
            TransactionOutcomeUnknown::class,
            'onTransactionUnknown',
            priority: 100,
        );
    }

    public function onQueryExecuted(
        QueryExecuted $event
    ): void {
        // technical audit observation
    }

    public function onTransactionUnknown(
        TransactionOutcomeUnknown $event
    ): void {
        // technical audit observation
    }
}
```

---

# 256. Extension anti-pattern

Incorrecto:

```php
final class TenantListener
{
    public function __invoke(
        QueryPlanningStarted $event
    ): void {
        $event->query()
            ->where('tenant_id', CurrentTenant::id());
    }
}
```

---

# 257. Why incorrect

Porque convierte:

```text
Event Listener
```

en:

```text
hidden Query Security Scope
```

---

# 258. Correct architecture

```text
Tenant Query Scope / Semantic Rule
↓
Query Model
↓
Query Events
```

---

# 259. Another anti-pattern

Incorrecto:

```php
final class TransactionListener
{
    public function __invoke(
        TransactionCommitted $event
    ): void {
        $event->transaction()->rollback();
    }
}
```

No tiene sentido semántico.

El commit ya ocurrió.

---

# 260. Another anti-pattern

Incorrecto:

```php
final class PersistenceListener
{
    public function __invoke(
        PersistencePlanCreated $event
    ): void {
        $event->plan()->removeOperation(...);
    }
}
```

Deberá utilizar:

```text
Persistence Planner Extension
```

---

# 261. Another anti-pattern

Incorrecto:

```php
final class ConnectionListener
{
    public function __invoke(
        ConnectionAcquired $event
    ): void {
        $event->connection()->exec(
            'SET tenant_id = 5'
        );
    }
}
```

La configuración tenant pertenece al Connection/Tenant integration.

---

# 262. Extension design rule

Cada necesidad deberá preguntarse:

```text
¿Solo observa?
    → Event Extension

¿Modifica Query semantics?
    → Query Extension

¿Modifica Persistence planning?
    → Persistence Extension

¿Modifica entity lifecycle?
    → Lifecycle Hook

¿Modifica routing?
    → Routing Policy

¿Modifica transaction behavior?
    → Transaction Policy/Synchronization
```

---

# 263. Architectural invariants

## DB-EEXT-001
Event Extension será distinto de Query Extension.

## DB-EEXT-002
Event Extension será distinto de Persistence Extension.

## DB-EEXT-003
Event Extension será distinto de Driver Extension.

## DB-EEXT-004
Event Extension será distinto de Dialect Extension.

## DB-EEXT-005
Event Extension será distinto de ORM Lifecycle Hook.

## DB-EEXT-006
Event Extension será distinto de Security Policy.

## DB-EEXT-007
Event Extension será distinto de Telemetry Provider.

## DB-EEXT-008
Event extensions observarán por default.

## DB-EEXT-009
Generic event listeners no modificarán Query AST.

## DB-EEXT-010
Generic event listeners no modificarán Query Plan.

## DB-EEXT-011
Generic event listeners no modificarán Compiled Query.

## DB-EEXT-012
Generic event listeners no modificarán Connection routing.

## DB-EEXT-013
Generic event listeners no controlarán commit/rollback.

## DB-EEXT-014
Generic event listeners no modificarán UnitOfWork.

## DB-EEXT-015
Generic event listeners no modificarán Persistence Plan.

## DB-EEXT-016
Custom events tendrán stable names.

## DB-EEXT-017
External event identity no dependerá de FQCN.

## DB-EEXT-018
Core event namespaces estarán reservados.

## DB-EEXT-019
Extension event namespaces deberán ser namespaced.

## DB-EEXT-020
Duplicate event names serán rechazados.

## DB-EEXT-021
Externalizable events tendrán schema version.

## DB-EEXT-022
Schema version será distinta de package version.

## DB-EEXT-023
Schema version será distinta de framework version.

## DB-EEXT-024
Internal event no implicará externalizable event.

## DB-EEXT-025
Listener IDs serán stable.

## DB-EEXT-026
Listener ID será distinto de FQCN.

## DB-EEXT-027
Listener scope será explícito.

## DB-EEXT-028
Request-scoped listener no se filtrará entre requests.

## DB-EEXT-029
Operation-scoped listener no se filtrará entre operations.

## DB-EEXT-030
Subscriber será conveniencia de registro, no segundo dispatcher.

## DB-EEXT-031
Extension registration será determinista.

## DB-EEXT-032
Extension registration será side-effect bounded.

## DB-EEXT-033
Registration no ejecutará queries por default.

## DB-EEXT-034
Registration no abrirá connections por default.

## DB-EEXT-035
Registration no abrirá transactions por default.

## DB-EEXT-036
Production registry podrá congelarse.

## DB-EEXT-037
Frozen registry será immutable.

## DB-EEXT-038
Runtime registration será scope-controlled.

## DB-EEXT-039
Filter será distinto de Authorization.

## DB-EEXT-040
Filter decidirá delivery, no DB execution.

## DB-EEXT-041
Filter ordering será deterministic.

## DB-EEXT-042
Event View será distinta del core Event.

## DB-EEXT-043
Payload sanitization no mutará evento core.

## DB-EEXT-044
External listeners recibirán sanitized payload.

## DB-EEXT-045
Bridge será distinto de unrestricted listener.

## DB-EEXT-046
Database Event Bridge será outbound por default.

## DB-EEXT-047
Reverse event injection no será automática.

## DB-EEXT-048
Quantum Database no dependerá de external Event System.

## DB-EEXT-049
Telemetry bridge podrá samplear.

## DB-EEXT-050
Telemetry sampling no afectará functional listeners.

## DB-EEXT-051
Async bridge no implicará durable delivery.

## DB-EEXT-052
AfterCommit callback no implicará durable delivery.

## DB-EEXT-053
Durable delivery requerirá durable protocol.

## DB-EEXT-054
Exactly-once no será afirmado genéricamente.

## DB-EEXT-055
Sink será distinto de Dispatcher.

## DB-EEXT-056
Sink failure no reescribirá DB outcome por default.

## DB-EEXT-057
Unsafe PHP unserialize estará prohibido.

## DB-EEXT-058
Serializer utilizará stable schemas.

## DB-EEXT-059
Serializer no activará lazy loading.

## DB-EEXT-060
Serializer no ejecutará DB queries.

## DB-EEXT-061
Serializer no resolverá arbitrary services.

## DB-EEXT-062
Aggregator será distinto de functional listener delivery.

## DB-EEXT-063
Aggregation no sustituirá eventos individuales cuando sean contractuales.

## DB-EEXT-064
Aggregator scope será explícito.

## DB-EEXT-065
Worker aggregation respetará tenant isolation.

## DB-EEXT-066
Event middleware será delivery middleware.

## DB-EEXT-067
Event middleware será distinto de Database Operation Middleware.

## DB-EEXT-068
Middleware no modificará Database operation original.

## DB-EEXT-069
Middleware ordering será deterministic.

## DB-EEXT-070
Event policy será distinta de Query Routing Policy.

## DB-EEXT-071
DENY_DELIVERY será distinto de deny DB operation.

## DB-EEXT-072
Extension capabilities serán explícitas.

## DB-EEXT-073
Least capability principle será aplicado.

## DB-EEXT-074
Extension bootstrap no recibirá EntityManager por default.

## DB-EEXT-075
Extension bootstrap no recibirá ConnectionManager por default.

## DB-EEXT-076
Extension bootstrap no recibirá TransactionManager por default.

## DB-EEXT-077
Package identity será stable.

## DB-EEXT-078
Extension dependency graph será deterministic.

## DB-EEXT-079
Circular dependencies serán rechazadas.

## DB-EEXT-080
Optional dependencies serán explícitas.

## DB-EEXT-081
Event availability será distinta de listener availability.

## DB-EEXT-082
Public Event API será versionada.

## DB-EEXT-083
Breaking API changes respetarán compatibility policy.

## DB-EEXT-084
Deprecation precederá removal cuando sea razonable.

## DB-EEXT-085
Misleading aliases estarán prohibidos.

## DB-EEXT-086
Built-in event descriptors no serán overrideable por default.

## DB-EEXT-087
Extensions podrán añadir listeners a built-in events.

## DB-EEXT-088
Extensions no podrán cambiar built-in event meaning.

## DB-EEXT-089
Listener priority será deterministic.

## DB-EEXT-090
Ordering constraints podrán declararse.

## DB-EEXT-091
Ordering cycles serán rechazados.

## DB-EEXT-092
Observers no deberían depender fuertemente del ordering.

## DB-EEXT-093
Listener failure policy será explícita.

## DB-EEXT-094
Observational listener default será RECORD_AND_CONTINUE.

## DB-EEXT-095
AfterCommit listener failure no deshará commit.

## DB-EEXT-096
Extension health podrá diagnosticarse.

## DB-EEXT-097
Extension exception no deberá corromper registry.

## DB-EEXT-098
Extension exception no deberá corromper transaction state.

## DB-EEXT-099
Extension exception no deberá corromper connection state.

## DB-EEXT-100
Listener timeout guarantees respetarán runtime capabilities.

## DB-EEXT-101
Dispatch budget será bounded.

## DB-EEXT-102
Listener count podrá limitarse.

## DB-EEXT-103
Recursion depth podrá limitarse.

## DB-EEXT-104
Event recursion será detectable.

## DB-EEXT-105
Origin-based filtering será soportable.

## DB-EEXT-106
Listener DB access podrá clasificarse.

## DB-EEXT-107
Listener side effects podrán clasificarse.

## DB-EEXT-108
Retry-sensitive side effects serán diagnosticables.

## DB-EEXT-109
Event payload minimization será default.

## DB-EEXT-110
Tenant isolation será preservada.

## DB-EEXT-111
Global tenant listeners requerirán autorización explícita.

## DB-EEXT-112
Credentials nunca serán event payload.

## DB-EEXT-113
Query parameter values estarán redacted por default.

## DB-EEXT-114
Entity values no serán incluidos por default.

## DB-EEXT-115
Extension secrets no se almacenarán en event metadata.

## DB-EEXT-116
Extension metadata tendrá namespace propio.

## DB-EEXT-117
Core metadata keys estarán protegidas.

## DB-EEXT-118
External metadata no contendrá live objects.

## DB-EEXT-119
Metadata size será bounded.

## DB-EEXT-120
Oversized metadata no crecerá silenciosamente.

## DB-EEXT-121
External payload schema será validado.

## DB-EEXT-122
Invalid external payload no será publicado.

## DB-EEXT-123
Bridge routes serán explícitas.

## DB-EEXT-124
Event selectors serán declarativos cuando sea posible.

## DB-EEXT-125
Production registry preferirá class descriptors sobre closures.

## DB-EEXT-126
Listener dependencies usarán Container.

## DB-EEXT-127
Singleton listeners deberán ser stateless/coroutine-safe.

## DB-EEXT-128
Request-specific mutable listener state no vivirá en singleton.

## DB-EEXT-129
FrankenPHP mutable event state será request-scoped.

## DB-EEXT-130
RoadRunner mutable event state será operation-scoped.

## DB-EEXT-131
OpenSwoole mutable event state será coroutine-safe.

## DB-EEXT-132
Bridge shared services no almacenarán current request state.

## DB-EEXT-133
Context será pasado explícitamente.

## DB-EEXT-134
Extension cleanup será scope-bound.

## DB-EEXT-135
Cleanup failure no contaminará siguiente scope.

## DB-EEXT-136
Testing registry modifications estarán aisladas.

## DB-EEXT-137
Registry snapshots serán deterministic.

## DB-EEXT-138
Extension validation ocurrirá en bootstrap cuando sea posible.

## DB-EEXT-139
Invalid listener signatures fallarán rápido.

## DB-EEXT-140
Invalid bridge compatibility fallará rápido.

## DB-EEXT-141
Manifest podrá cachearse.

## DB-EEXT-142
Manifest será immutable.

## DB-EEXT-143
Manifest invalidation será version-aware.

## DB-EEXT-144
Hot-unload no será requisito core.

## DB-EEXT-145
Development reload podrá reconstruir registry completo.

## DB-EEXT-146
Extension diagnostics serán disponibles.

## DB-EEXT-147
Dispatch graph será explainable.

## DB-EEXT-148
Hot-path listener lookup será optimizado.

## DB-EEXT-149
No-listener fast path será preservado.

## DB-EEXT-150
Subscriber expansion ocurrirá en compile/bootstrap.

## DB-EEXT-151
Reflection repetida en hot-path será evitada.

## DB-EEXT-152
Listener instantiation podrá ser lazy.

## DB-EEXT-153
Bridge instantiation podrá ser lazy.

## DB-EEXT-154
Serialization ocurrirá solo cuando sea necesaria.

## DB-EEXT-155
Heavy sanitization se aplicará solo cuando sea necesaria.

## DB-EEXT-156
Extension telemetry tendrá bounded cardinality.

## DB-EEXT-157
EventId no será metric label.

## DB-EEXT-158
TransactionId no será metric label.

## DB-EEXT-159
QueryId no será metric label.

## DB-EEXT-160
TenantId no será metric label automático.

## DB-EEXT-161
Event Extension System no será Event Sourcing.

## DB-EEXT-162
Event Extension System no será Domain Event Bus.

## DB-EEXT-163
Event Extension System no será Queue System.

## DB-EEXT-164
Event Extension System no será Audit Store.

## DB-EEXT-165
Event Extension System no será Telemetry Store.

## DB-EEXT-166
Event Extension System no será Query Middleware.

## DB-EEXT-167
Event Extension System no será Transaction Manager.

## DB-EEXT-168
Event Extension System no será Persistence Planner.

## DB-EEXT-169
Event Extension System no será Security Engine.

## DB-EEXT-170
Event Extension System preservará invariantes de todos los eventos core.

---

# 264. Modelo formal

Sea:

```text
E
```

el conjunto de eventos core y:

```text
X
```

el conjunto de eventos agregados por extensiones.

Entonces:

```text
E' = E ∪ X
```

pero ninguna extensión podrá redefinir semánticamente:

```text
e ∈ E
```

sin una versión explícita del contrato.

---

# 265. Listener model

Sea:

```text
L(e)
```

el conjunto ordenado de listeners para evento `e`.

Entonces:

```text
Dispatch(e)
=
Invoke(
    Filter(
        L(e)
    )
)
```

bajo:

```text
security policy
scope policy
failure policy
dispatch budget
```

---

# 266. Extension safety

Para cualquier listener:

```text
l ∈ L(e)
```

su resultado:

```text
Outcome(l)
```

no deberá redefinir automáticamente:

```text
DatabaseOutcome
```

---

# 267. Event mutation formalization

Para un evento inmutable:

```text
e_before = e_after
```

respecto a su semántica core después de pasar por listeners.

Las vistas sanitizadas podrán variar:

```text
View(e, listenerA)
≠
View(e, listenerB)
```

sin cambiar `e`.

---

# 268. Bridge model

Sea:

```text
B(e)
```

el conjunto de bridges aplicables.

Entonces:

```text
BridgeDelivery(e)
```

es posterior o paralelo a la observación local según policy, pero:

```text
BridgeFailure(e)
↛
DatabaseOperationFailure
```

por default.

---

# 269. Durable delivery model

Si se requiere:

```text
Database Commit
+
eventual external delivery
```

la arquitectura deberá utilizar:

```text
DurableRecord
within
same transactional authority
```

cuando sea posible.

Por tanto:

```text
InProcessBridge
≠
DurableDelivery
```

---

# 270. Arquitectura final del sistema de extensiones

```text
                        Application / Packages
                                  │
                                  ▼
                    Event Extension Providers
                                  │
                                  ▼
                     Mutable Extension Registry
                                  │
                        ┌─────────┼──────────┐
                        ▼         ▼          ▼
                     Events    Listeners   Bridges
                        │         │          │
                        └─────────┼──────────┘
                                  ▼
                       Registry Validation
                                  │
                                  ▼
                       Registry Compilation
                                  │
                                  ▼
                      Immutable Runtime Registry
                                  │
                                  ▼
                        Database Event Dispatcher
                                  │
              ┌───────────────────┼────────────────────┐
              ▼                   ▼                    ▼
          Filters             Listeners             Bridges
              │                   │                    │
              ▼                   ▼                    ▼
          Security            Subscribers          Telemetry /
          Policies                                Framework /
                                                  External
```

---

# 271. Arquitectura completa del Bloque 20

```text
                            Quantum Database
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        ▼                         ▼                         ▼
   Query Engine             Connection System          Transaction
        │                         │                         │
        │                         │                         │
        ▼                         ▼                         ▼
   Query Events            Connection Events       Transaction Events
        │                         │                         │
        └──────────────┬──────────┴───────────┬─────────────┘
                       │                      │
                       ▼                      ▼
                Entity Lifecycle        Persistence Engine
                     Events                   │
                       │                      ▼
                       └────────────── Persistence Events
                                      │
                                      ▼
                           Database Event Architecture
                                      │
                                      ▼
                            Database Event Dispatcher
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
               Core Listeners    Extensions         Bridges
                                      │                 │
                                      ▼                 ▼
                                  Packages         EventSystem /
                                                   Telemetry /
                                                    External
```

---

# 272. Reglas maestras del Bloque 20

VoltStack mantendrá:

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
Query Event
≠
Query
```

```text
Connection Event
≠
Connection
```

```text
Transaction Event
≠
Transaction Outcome
```

```text
Entity Lifecycle Event
≠
Persistence Event
```

```text
Persistence Event
≠
Transaction Commit
```

```text
Event Extension
≠
Database Operation Extension
```

```text
Listener
≠
Interceptor
```

```text
Listener
≠
Policy
```

```text
Filter
≠
Authorization
```

```text
AfterCommit Callback
≠
Transactional Outbox
```

```text
Async Bridge
≠
Durable Delivery
```

```text
Stable Event Name
≠
PHP FQCN
```

```text
Event Schema Version
≠
Framework Version
```

```text
External Event
≠
Internal Live Object
```

y especialmente:

```text
Observability
≠
Correctness Authority
```

---

# 273. Filosofía final

El Event System de VoltStack Database deberá seguir:

```text
Explicit boundaries
over
listener magic

Typed contracts
over
stringly-typed events

Stable event identity
over
FQCN coupling

Immutable observations
over
hidden mutation

Scoped state
over
global static state

Sanitized payloads
over
full state dumps

Deterministic registration
over
incidental discovery order

Durable protocols
over
best-effort callbacks
```

---

# 274. Estado final del Bloque 20

```text
BLOCK 20 — EVENTS

✓ 209_DATABASE_EVENT_ARCHITECTURE.md
✓ 210_DATABASE_QUERY_EVENT_SYSTEM.md
✓ 211_DATABASE_CONNECTION_EVENT_SYSTEM.md
✓ 212_DATABASE_TRANSACTION_EVENT_PIPELINE.md
✓ 213_DATABASE_ENTITY_LIFECYCLE_EVENT_SYSTEM.md
✓ 214_DATABASE_PERSISTENCE_EVENT_SYSTEM.md
✓ 215_DATABASE_EVENT_EXTENSION_SYSTEM.md
```

Con esto queda **formalmente cerrado el Bloque 20 — Events**.

---

# 275. Resultado del Bloque 20

La capa de eventos resultante permite que VoltStack observe:

```text
Query lifecycle
Connection lifecycle
Transaction lifecycle
ORM entity lifecycle
Persistence lifecycle
```

y los exponga a:

```text
applications
packages
debugging
profiling
telemetry
audit integrations
framework integrations
developer tools
```

sin entregar el control semántico de Database a un Event Bus.

La dependencia ideal queda:

```text
Core Database Components
        │
        ▼
Explicit Core Contracts
        │
        ├────────► Correctness-Critical Coordinators
        │
        └────────► Typed Database Events
                          │
                          ▼
                     Observation
                          │
                 ┌────────┼────────┐
                 ▼        ▼        ▼
              App      Packages  Telemetry
```

---

# 276. Siguiente bloque

El siguiente bloque será:

```text
BLOCK 21 — TELEMETRY AND DEBUGGING
```

compuesto por:

```text
216_DATABASE_TELEMETRY_ARCHITECTURE.md
217_DATABASE_QUERY_TELEMETRY_SYSTEM.md
218_DATABASE_CONNECTION_TELEMETRY_SYSTEM.md
219_DATABASE_TRANSACTION_TELEMETRY_SYSTEM.md
220_DATABASE_ORM_TELEMETRY_SYSTEM.md
221_DATABASE_QUERY_PROFILER_SYSTEM.md
222_DATABASE_SLOW_QUERY_DETECTION_SYSTEM.md
223_DATABASE_N_PLUS_ONE_TELEMETRY_SYSTEM.md
224_DATABASE_DEBUG_INFORMATION_SYSTEM.md
225_DATABASE_DEVELOPER_DEBUG_TOOLBAR_INTEGRATION.md
```

---

# 277. Siguiente documento

```text
216_DATABASE_TELEMETRY_ARCHITECTURE.md
```

El siguiente documento definirá la arquitectura transversal de observabilidad de Database:

```text
Database Operations
↓
Events / Direct Instrumentation
↓
Telemetry Context
↓
Metrics
Traces
Logs
Profiles
Diagnostics
↓
Telemetry Providers
```

incluyendo:

```text
telemetry contracts
instrumentation boundaries
metrics architecture
trace architecture
structured logs
correlation IDs
sampling
cardinality governance
sensitive-data redaction
query fingerprints
transaction spans
connection metrics
ORM metrics
runtime metrics
persistent-worker isolation
provider adapters
OpenTelemetry integration boundaries
```

bajo una distinción central:

> **Telemetry observará y medirá el comportamiento de Database; nunca deberá convertirse en una dependencia necesaria para la ejecución correcta de queries, connections, transactions, ORM o persistence.**