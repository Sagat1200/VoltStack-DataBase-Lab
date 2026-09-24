# 315_DATABASE_EVENT_SYSTEM_INTEGRATION.md

# VoltStack Quantum Database
## Database Event System Integration

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 315 — Database Event System Integration  
**Bloque:** 32 — VoltStack Integration  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `314_DATABASE_CACHE_INTEGRATION_SYSTEM.md`  
**Siguiente documento:** `316_DATABASE_TELEMETRY_INTEGRATION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura mediante la cual:

```text
VoltStack/Quantum/Database
```

se integra con el sistema general de eventos de VoltStack:

```text
VoltStack Event System
```

preservando las fronteras internas de Database y evitando que el sistema de eventos se convierta en propietario de las semánticas de consultas, conexiones, transacciones, ORM, persistencia o base de datos.

La regla fundamental será:

> **Los eventos de Database describen hechos, transiciones y observaciones que han ocurrido o han sido determinadas por el subsistema Database; nunca sustituyen comandos, transacciones, estados internos ni resultados autoritativos de la base de datos.**

Formalmente:

```text
Database Operation
      ↓
Semantic Outcome
      ↓
Database Event
      ↓
Event Integration Bridge
      ↓
VoltStack Event System
      ↓
Listeners / Subscribers / Publishers
```

Nunca:

```text
Event Listener
      ↓
defines
      ↓
Database Truth
```

---

# 2. Principios fundamentales

VoltStack deberá preservar:

```text
Event
≠
Command
```

```text
Event
≠
Database Operation
```

```text
Event
≠
Transaction Outcome
```

```text
Event
≠
Domain Event
```

```text
Event
≠
Telemetry
```

```text
Event
≠
Hook
```

```text
Event
≠
Retry Policy
```

```text
Event
≠
Cache Invalidation Guarantee
```

Y especialmente:

```text
Listener Failure
≠
Database Failure
```

cuando el hecho de base de datos ya ocurrió de manera irreversible.

---

# 3. Relación con la arquitectura previa

Los documentos:

```text
209_DATABASE_EVENT_ARCHITECTURE.md
210_DATABASE_QUERY_EVENT_SYSTEM.md
211_DATABASE_CONNECTION_EVENT_SYSTEM.md
212_DATABASE_TRANSACTION_EVENT_PIPELINE.md
213_DATABASE_ENTITY_LIFECYCLE_EVENT_SYSTEM.md
214_DATABASE_PERSISTENCE_EVENT_SYSTEM.md
215_DATABASE_EVENT_EXTENSION_SYSTEM.md
```

definen la semántica interna de eventos de Database.

Este documento define específicamente:

```text
Database Internal Events
        ↓
Integration Boundary
        ↓
VoltStack Event System
```

Por tanto:

```text
209–215
=
Database Event Semantics

315
=
Framework Event Integration
```

---

# 4. Objetivos

El sistema deberá:

1. desacoplar Database del Event System concreto;
2. permitir Database sin Event System externo;
3. proporcionar un bridge explícito;
4. distinguir eventos internos de eventos públicos;
5. distinguir eventos síncronos y asíncronos;
6. soportar eventos de queries;
7. soportar eventos de conexiones;
8. soportar eventos transaccionales;
9. soportar eventos ORM;
10. soportar eventos de persistencia;
11. soportar eventos de schema;
12. soportar eventos de migraciones;
13. soportar eventos de caché relacionados con Database;
14. preservar orden;
15. definir failure semantics;
16. soportar `afterCommit`;
17. manejar rollback;
18. preservar outcomes `UNKNOWN`;
19. soportar correlation IDs;
20. incorporar DatabaseContext;
21. soportar tenant/shard context;
22. proteger información sensible;
23. integrarse con Telemetry;
24. funcionar en persistent workers;
25. soportar plugins;
26. versionar contratos públicos;
27. permitir testing determinista;
28. soportar outbox cuando se requiera publicación durable.

---

# 5. Arquitectura general

```text
                     Application
                          │
                          ▼
                  Database Public API
                          │
                          ▼
                 Database Subsystems
                          │
       ┌──────────────────┼───────────────────┐
       │                  │                   │
       ▼                  ▼                   ▼
     Query          Transaction             ORM
       │                  │                   │
       └──────────────────┼───────────────────┘
                          │
                          ▼
               Database Internal Events
                          │
                          ▼
               DatabaseEventDispatcher
                          │
             ┌────────────┴────────────┐
             │                         │
             ▼                         ▼
     Internal Listeners       Event Integration Bridge
                                       │
                                       ▼
                              VoltStack Event System
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
                Listeners         Subscribers         Publishers
```

---

# 6. Internal Events vs Framework Events

Deberán existir dos conceptos claramente separados:

```text
Database Internal Event
```

y:

```text
Framework/Public Event
```

No todo evento interno deberá publicarse externamente.

---

# 7. Internal Event

Representa información necesaria para coordinación dentro de Database.

Ejemplos:

```text
QueryPlanningCompleted
TransactionStateChanged
EntityHydrated
ConnectionResetCompleted
```

Puede ser demasiado detallado o inestable para convertirse en API pública.

---

# 8. Framework Event

Representa un contrato estable que puede ser consumido por:

```text
application listeners
plugins
extensions
framework modules
observability adapters
```

---

# 9. Internal Event ≠ Public API

Esta separación permitirá evolucionar internamente:

```text
Query Planner
Hydration Engine
Persistence Engine
Transaction Engine
```

sin convertir cada detalle en una obligación de backward compatibility.

---

# 10. Event Integration Boundary

Se propone:

```text
VoltStack\Quantum\Database\Integration\Event
```

como frontera entre:

```text
Database
```

y:

```text
VoltStack Event System
```

---

# 11. Dependencia opcional

Database Core dependerá conceptualmente de:

```text
DatabaseEventDispatcher
```

no directamente de:

```text
VoltStack\Quantum\Event\EventDispatcher
```

---

# 12. Adapter architecture

```text
Database
   ↓
DatabaseEventDispatcher Contract
   ↑
   │
NullDatabaseEventDispatcher
VoltStackEventDispatcherAdapter
```

---

# 13. Null Event Dispatcher

Cuando el Event System externo no esté instalado:

```text
Database
   ↓
NullDatabaseEventDispatcher
```

Database deberá continuar funcionando.

---

# 14. Events disabled ≠ Database disabled

Deshabilitar publicación externa no deberá afectar:

```text
query correctness
transaction correctness
ORM correctness
persistence correctness
schema correctness
```

---

# 15. Internal coordination caveat

Si Database necesita una notificación interna para preservar sus propias invariantes, ésta deberá formar parte de su arquitectura interna.

No deberá depender de que el Event System externo esté habilitado.

---

# 16. DatabaseEventDispatcher

Contrato conceptual:

```php
interface DatabaseEventDispatcher
{
    public function dispatch(
        DatabaseEvent $event,
    ): DatabaseEventDispatchResult;
}
```

---

# 17. Public Event Publisher

La publicación externa podrá separarse:

```php
interface DatabaseEventPublisher
{
    public function publish(
        PublicDatabaseEvent $event,
    ): DatabaseEventPublicationResult;
}
```

Esto permite distinguir:

```text
dispatch
```

de:

```text
publish
```

---

# 18. Dispatch ≠ Publish

Se utilizarán semánticas diferentes:

```text
dispatch
=
in-process event delivery

publish
=
exposure to framework/external event infrastructure
```

---

# 19. Synchronous ≠ Asynchronous

Otro eje independiente:

```text
Dispatch Mode
├── synchronous
└── asynchronous
```

No deberá inferirse:

```text
public event
=
async
```

ni:

```text
internal event
=
sync
```

aunque ésos puedan ser defaults frecuentes.

---

# 20. Event taxonomy

Database podrá producir categorías como:

```text
QUERY
CONNECTION
TRANSACTION
ENTITY
PERSISTENCE
SCHEMA
MIGRATION
CACHE
CAPABILITY
BACKUP
RESTORE
MAINTENANCE
ADMINISTRATION
RUNTIME
```

---

# 21. Query events

Ejemplos:

```text
DatabaseQueryPreparing
DatabaseQueryExecuting
DatabaseQueryExecuted
DatabaseQueryFailed
DatabaseQueryCancelled
DatabaseQueryTimedOut
```

---

# 22. QueryPreparing

Deberá representar una operación aún no ejecutada.

No deberá afirmar:

```text
DatabaseMutationOccurred
```

---

# 23. QueryExecuting

Representará el inicio de ejecución.

No deberá interpretarse como éxito.

---

# 24. QueryExecuted

Representará una ejecución cuyo outcome está suficientemente determinado.

---

# 25. QueryFailed

Deberá conservar:

```text
failure classification
execution phase
outcome evidence
retryability evidence
```

sin exponer secretos.

---

# 26. Query event payload

Conceptualmente:

```text
DatabaseQueryEvent
├── EventId
├── Timestamp
├── QueryOperationId
├── QueryFingerprint
├── QueryKind
├── LogicalDatabase
├── ConnectionIdentity?
├── TransactionIdentity?
├── TenantIdentity?
├── ShardIdentity?
├── Duration?
├── Outcome?
└── SafeMetadata
```

---

# 27. Raw SQL

Raw SQL no deberá incluirse automáticamente en eventos públicos.

---

# 28. Query parameters

Nunca deberán publicarse indiscriminadamente.

---

# 29. Parameter redaction

La política podrá producir:

```text
parameter_count
parameter_types
safe fingerprints
redacted values
```

según clasificación.

---

# 30. Query fingerprint

Será preferido para:

```text
correlation
grouping
diagnostics
telemetry
```

frente a SQL raw.

---

# 31. Connection events

Ejemplos:

```text
DatabaseConnectionOpening
DatabaseConnectionOpened
DatabaseConnectionFailed
DatabaseConnectionResetting
DatabaseConnectionReset
DatabaseConnectionClosed
DatabaseConnectionDiscarded
```

---

# 32. ConnectionOpened ≠ healthy forever

El evento sólo describe:

```text
connection opened successfully at time t
```

---

# 33. ConnectionReset

Deberá distinguir:

```text
RESET_SUCCESS
RESET_FAILED
RESET_UNKNOWN
```

cuando corresponda.

---

# 34. Discarded connection

Podrá indicar motivos como:

```text
BROKEN
DIRTY
UNKNOWN_STATE
RESET_FAILURE
RESOURCE_POLICY
WORKER_SHUTDOWN
```

---

# 35. Credentials

Nunca deberán incluirse en eventos.

---

# 36. DSN redaction

Una representación segura podrá mostrar:

```text
mysql://db.internal/app
```

pero no:

```text
mysql://user:password@db.internal/app
```

---

# 37. Transaction events

Ejemplos:

```text
DatabaseTransactionBeginning
DatabaseTransactionBegan
DatabaseTransactionCommitting
DatabaseTransactionCommitted
DatabaseTransactionRollingBack
DatabaseTransactionRolledBack
DatabaseTransactionOutcomeUnknown
```

---

# 38. Transaction events require precise semantics

Fundamental:

```text
Committing
≠
Committed
```

---

# 39. Before commit

```text
DatabaseTransactionCommitting
```

indica intención/inicio.

No éxito.

---

# 40. After commit

Sólo deberá producirse:

```text
DatabaseTransactionCommitted
```

cuando Database tenga evidencia suficiente de commit.

---

# 41. Commit UNKNOWN

Si ocurre:

```text
COMMIT sent
↓
connection lost
```

y no puede demostrarse resultado:

```text
DatabaseTransactionOutcomeUnknown
```

deberá representar la realidad.

---

# 42. UNKNOWN ≠ RolledBack

Nunca deberá transformarse automáticamente:

```text
UNKNOWN
→
DatabaseTransactionRolledBack
```

---

# 43. UNKNOWN ≠ Committed

Tampoco:

```text
UNKNOWN
→
DatabaseTransactionCommitted
```

---

# 44. Rollback event

Sólo deberá afirmar rollback cuando exista evidencia suficiente.

---

# 45. Savepoint events

Podrán existir:

```text
DatabaseSavepointCreated
DatabaseSavepointReleased
DatabaseSavepointRolledBack
```

si se consideran parte de la API pública necesaria.

---

# 46. Savepoint ≠ nested transaction

Los eventos deberán preservar esta distinción.

---

# 47. Transaction event ordering

Ejemplo exitoso:

```text
TransactionBeginning
      ↓
TransactionBegan
      ↓
TransactionCommitting
      ↓
TransactionCommitted
```

---

# 48. Failed commit

Puede producir:

```text
TransactionCommitting
      ↓
TransactionCommitFailed
```

y posteriormente:

```text
TransactionRolledBack
```

sólo si el rollback es realmente conocido.

---

# 49. Ambiguous commit

```text
TransactionCommitting
      ↓
TransactionOutcomeUnknown
```

No deberá fabricarse un evento terminal falso.

---

# 50. ORM events

Categorías:

```text
Entity lifecycle
Hydration
EntityManager lifecycle
UnitOfWork
Relationship loading
Persistence
```

---

# 51. Entity lifecycle events

Ejemplos:

```text
EntityLoaded
EntityPersistScheduled
EntityRemoveScheduled
EntityDetached
EntityRefreshed
```

---

# 52. persist event semantics

`persist()` no equivale a INSERT.

Por tanto:

```text
EntityPersistScheduled
```

no:

```text
EntityInserted
```

al llamar `persist()`.

---

# 53. remove event semantics

Igualmente:

```text
EntityRemoveScheduled
```

no significa:

```text
row deleted
```

---

# 54. Flush events

Podrán existir:

```text
OrmFlushStarting
OrmFlushPlanned
OrmFlushExecuting
OrmFlushCompleted
OrmFlushFailed
```

---

# 55. FlushCompleted ≠ TransactionCommitted

Regla crítica:

```text
flush()
≠
commit()
```

---

# 56. Persistence events

Ejemplos:

```text
EntityInsertPlanned
EntityInsertExecuted
EntityUpdatePlanned
EntityUpdateExecuted
EntityDeletePlanned
EntityDeleteExecuted
```

---

# 57. Executed mutation ≠ committed mutation

Dentro de una transacción:

```text
UPDATE executed
```

no significa:

```text
UPDATE committed
```

---

# 58. Two semantic levels

Por tanto se distinguirá:

```text
Persistence Execution Event
```

de:

```text
Committed Persistence Event
```

cuando sea necesario.

---

# 59. Committed entity events

Podrán existir eventos:

```text
EntityChangesCommitted
```

generados únicamente después de commit.

---

# 60. Domain events

Los eventos del ORM no deberán confundirse con eventos del dominio.

Ejemplo:

```text
User row updated
```

no implica automáticamente:

```text
CustomerEmailChanged
```

---

# 61. Domain event ownership

Eventos de negocio pertenecen a:

```text
application/domain
```

aunque puedan coordinarse con transacciones Database.

---

# 62. Domain event integration

VoltStack podrá proporcionar mecanismos para:

```text
collect domain events
→ transaction
→ afterCommit
→ publish
```

pero Database no deberá inventarlos automáticamente.

---

# 63. Entity payload policy

No deberá enviarse indiscriminadamente:

```php
$event->entity = $managedEntity;
```

a eventos públicos/asíncronos.

---

# 64. Why

Un managed entity puede contener:

```text
EntityManager references
lazy relationships
dirty state
sensitive data
request scope
tenant context
```

---

# 65. Public entity event payload

Preferir:

```text
EntityType
EntityIdentifier
ChangedFieldNames
SafeMetadata
TenantIdentity
TransactionIdentity
```

---

# 66. Entity snapshot

Cuando se requiera payload:

```text
immutable canonical snapshot
```

podrá utilizarse según política.

---

# 67. Snapshot ≠ managed entity

Debe permanecer explícito.

---

# 68. Hydration events

Podrán emitirse:

```text
EntityHydrationStarted
EntityHydrated
HydrationCompleted
HydrationFailed
```

principalmente para extensibilidad/diagnóstico interno.

---

# 69. Hydration event performance

No deberá emitirse un evento público pesado por cada fila por defecto.

---

# 70. High-volume events

Eventos de granularidad alta deberán poder:

```text
disable
sample
aggregate
keep internal only
```

---

# 71. Relationship events

Podrán incluir:

```text
RelationshipLoadRequested
RelationshipLoaded
RelationshipBatchLoaded
```

cuando resulten útiles.

---

# 72. Lazy load event

Puede ser útil para:

```text
N+1 detection
debugging
telemetry
```

pero Telemetry deberá consumir información estructurada, no depender de listeners arbitrarios.

---

# 73. Schema events

Ejemplos:

```text
SchemaIntrospectionStarted
SchemaIntrospectionCompleted
SchemaDiffCreated
SchemaCompilationCompleted
SchemaOperationExecuted
```

---

# 74. Schema event ≠ migration event

Schema representa estructura.

Migration representa evolución gobernada.

---

# 75. Migration events

Ejemplos:

```text
MigrationDiscovered
MigrationPlanningStarted
MigrationPlanCreated
MigrationExecuting
MigrationExecuted
MigrationFailed
MigrationRolledBack
MigrationBatchCompleted
```

---

# 76. Migration event safety

No deberá permitirse que un listener convierta una operación:

```text
UNSAFE
```

en:

```text
SAFE
```

saltándose el Migration Safety System.

---

# 77. Event listeners ≠ policy bypass

Los listeners podrán observar o solicitar acciones mediante APIs autorizadas, pero no mutar arbitrariamente decisiones internas.

---

# 78. Cache-related events

La integración con el documento 314 podrá producir:

```text
DatabaseCacheHit
DatabaseCacheMiss
DatabaseCacheInvalidated
DatabaseCacheProviderFailed
DatabaseCacheGenerationChanged
```

cuando sea apropiado.

---

# 79. Cache events ≠ invalidation mechanism

Fundamental:

> Una invalidación necesaria para correctness no dependerá exclusivamente de que un listener reciba `DatabaseCacheInvalidated`.

La invalidación deberá ejecutarse por el subsistema correspondiente.

El evento describe el hecho.

---

# 80. Capability events

Podrán existir:

```text
DatabaseCapabilitiesDiscovered
DatabaseCapabilityConflictDetected
DatabaseCapabilitySnapshotChanged
```

principalmente para operaciones y diagnostics.

---

# 81. Backup events

Ejemplos:

```text
DatabaseBackupStarted
DatabaseBackupCreated
DatabaseBackupVerified
DatabaseBackupFailed
```

---

# 82. BackupCreated ≠ BackupVerified

Los eventos deberán preservar la diferencia.

---

# 83. Restore events

```text
DatabaseRestoreStarted
DatabaseRestoreApplied
DatabaseRestoreValidated
DatabaseRestoreCompleted
DatabaseRestoreFailed
```

---

# 84. RestoreApplied ≠ RestoreCompleted

Debe preservarse el pipeline real.

---

# 85. Administration events

Operaciones privilegiadas podrán generar:

```text
DatabaseAdministrativeOperationStarted
DatabaseAdministrativeOperationCompleted
DatabaseAdministrativeOperationFailed
```

además de audit records.

---

# 86. Event ≠ Audit Record

Los eventos son comunicaciones del sistema.

Audit proporciona evidencia gobernada de acciones.

```text
Event
≠
Audit
```

---

# 87. Audit integration

Un Audit adapter podrá consumir eventos adecuados, pero operaciones críticas deberán generar audit evidence mediante el sistema de auditoría correspondiente.

---

# 88. Event envelope

Todos los eventos públicos de Database deberán utilizar un envelope coherente.

Conceptualmente:

```text
DatabaseEventEnvelope
├── EventId
├── EventName
├── EventVersion
├── OccurredAt
├── DatabaseOperationId?
├── CorrelationId?
├── CausationId?
├── RequestId?
├── TransactionId?
├── LogicalDatabase?
├── ConnectionId?
├── TenantId?
├── ShardId?
├── RuntimeContext?
├── Payload
└── Metadata
```

---

# 89. EventId

Cada instancia deberá poseer:

```text
EventId
```

estable y único dentro de los requisitos del sistema.

---

# 90. EventName

Preferir nombres estables:

```text
database.transaction.committed
database.query.executed
database.orm.entity.loaded
```

en contratos públicos.

---

# 91. PHP class ≠ event identity

El FQCN podrá mapearse internamente, pero no deberá ser necesariamente la única identidad durable.

---

# 92. Event version

Los eventos públicos deberán poder versionarse:

```text
database.transaction.committed:v1
```

conceptualmente.

---

# 93. Version ≠ framework release

Un evento puede mantener:

```text
v1
```

a través de múltiples versiones de VoltStack.

---

# 94. occurredAt

Representa cuándo ocurrió el hecho.

---

# 95. publishedAt

Para publicación asíncrona podrá existir:

```text
publishedAt
```

separado.

---

# 96. occurredAt ≠ publishedAt

Especialmente con outbox:

```text
event occurs
↓
commit
↓
outbox
↓
later publish
```

---

# 97. Correlation ID

Permitirá relacionar:

```text
HTTP request
Job
Database query
Transaction
Cache operation
Telemetry trace
```

---

# 98. Causation ID

Podrá expresar:

```text
Event B
caused by
Event/Operation A
```

sin crear dependencia circular.

---

# 99. DatabaseOperationId

Cada operación significativa podrá tener identidad interna para correlación.

---

# 100. TransactionId

Deberá ser una identidad lógica de VoltStack.

No necesariamente el ID interno del DBMS.

---

# 101. ConnectionId

No deberá exponer información sensible ni punteros/handles internos.

---

# 102. Tenant context

Eventos tenant-scoped deberán transportar una identidad segura de tenant cuando sea necesaria.

---

# 103. Tenant ID telemetry caution

La identidad real del tenant puede tener alta cardinalidad o sensibilidad.

El Event System y Telemetry aplicarán políticas distintas.

---

# 104. Shard context

Eventos shard-aware podrán incluir:

```text
ShardIdentity
ShardMapGeneration
```

cuando sea relevante.

---

# 105. Event immutability

Los eventos deberán ser:

```text
immutable
```

una vez creados.

---

# 106. Listener mutation prohibited

Un listener no deberá modificar:

```text
$event->transactionOutcome
```

o equivalentes.

---

# 107. Event Builder

Si se requiere construcción incremental interna:

```text
Mutable Builder
↓
finalize
↓
Immutable Event
```

---

# 108. Before-events

Existen casos donde un evento previo puede representar:

```text
operation about to happen
```

pero no deberá permitir mutación ilimitada de la operación.

---

# 109. Event vs Interceptor

Si se necesita modificar comportamiento:

```text
Interceptor
Policy
Middleware
Extension Point
```

es más apropiado que abusar de Events.

---

# 110. Example

Incorrecto:

```php
on(QueryExecuting::class, function ($event) {
    $event->sql = 'DROP TABLE users';
});
```

---

# 111. Query customization

Deberá hacerse mediante:

```text
Query AST extension
Compiler extension
Query middleware/interceptor
```

según arquitectura.

---

# 112. Listener ordering

Cuando listeners síncronos tengan orden significativo, éste deberá ser explícito.

---

# 113. Priority

Podrá existir:

```text
priority
```

pero:

```text
priority
≠
dependency
```

---

# 114. Listener dependencies

Si Listener B requiere resultado de A, probablemente no son listeners independientes.

Deberá modelarse una pipeline o servicio coordinador.

---

# 115. Event ordering guarantee

VoltStack deberá definir qué orden garantiza.

Ejemplo dentro de una misma transacción:

```text
TransactionBegan
before
TransactionCommitted
```

---

# 116. Cross-worker ordering

No deberá asumirse automáticamente orden global entre workers.

---

# 117. Distributed event ordering

Si se requiere, deberá implementarse mediante infraestructura específica.

---

# 118. Event sequence

Podrá incluirse:

```text
OperationSequence
```

para ordenar eventos dentro de un contexto concreto.

---

# 119. Sequence ≠ global timestamp order

Debe permanecer explícito.

---

# 120. Listener failure model

Un listener puede fallar:

```text
before operation
during operation
after operation
after commit
async publication
```

y cada caso tiene semánticas diferentes.

---

# 121. Pre-operation listener failure

Si el listener forma parte explícita de una pipeline síncrona crítica:

```text
listener failure
```

puede impedir que la operación comience.

Pero esto deberá ser declarado por contrato.

---

# 122. Observational listener failure

Un listener puramente observacional no debería romper una operación Database por defecto.

---

# 123. Critical vs observational listener

Se propone distinguir:

```text
CRITICAL
OBSERVATIONAL
```

o un modelo equivalente.

---

# 124. Critical listener caution

No deberá utilizarse `CRITICAL` como mecanismo arbitrario para inyectar lógica de negocio en Database.

---

# 125. After-operation failure

Si:

```text
INSERT executed
```

y luego un listener falla dentro de una transacción todavía abierta, la policy puede decidir abortar la transacción.

---

# 126. AfterCommit listener failure

Si:

```text
COMMIT = SUCCESS
```

y después falla:

```text
afterCommit listener
```

el estado sigue siendo:

```text
COMMITTED
```

---

# 127. Fundamental invariant

```text
AfterCommitListenerFailure
≠
DatabaseRollback
```

---

# 128. Error propagation

La aplicación podrá recibir:

```text
Database committed successfully,
post-commit side effect failed
```

como estado compuesto cuando corresponda.

---

# 129. PostCommitFailure

Podrá representarse mediante una excepción/resultado especializado que preserve:

```text
transactionOutcome = COMMITTED
```

---

# 130. Never hide commit

Incorrecto:

```text
throw GenericDatabaseException("operation failed")
```

si en realidad:

```text
DB committed
event listener failed
```

porque puede inducir al caller a reintentar la transacción.

---

# 131. Duplicate write danger

Si caller interpreta:

```text
failure
→ retry
```

podría duplicar una operación ya committed.

---

# 132. Async publication

Eventos públicos asíncronos no deberán publicarse antes de commit si representan cambios committed.

---

# 133. Incorrect async flow

```text
UPDATE
↓
publish event to queue
↓
COMMIT
```

Puede producir:

```text
consumer observes event
but DB rolls back
```

---

# 134. Correct async flow

```text
BEGIN
↓
UPDATE
↓
record pending event
↓
COMMIT
↓
publish
```

---

# 135. Crash window

Existe:

```text
COMMIT
↓
process crashes
↓
event never published
```

si sólo se utiliza `afterCommit` in-memory.

---

# 136. Durable publication

Para eventos que requieren garantía durable:

```text
Transactional Outbox
```

deberá ser el mecanismo recomendado.

---

# 137. Outbox architecture

```text
Application / ORM
       ↓
Database Transaction
       ├── business mutation
       └── outbox record
               ↓
             COMMIT
               ↓
         Outbox Dispatcher
               ↓
        VoltStack Event System
               ↓
        Broker / Consumers
```

---

# 138. Outbox record

Conceptualmente:

```text
DatabaseOutboxRecord
├── EventId
├── EventName
├── EventVersion
├── OccurredAt
├── Payload
├── Metadata
├── PublicationState
├── Attempts
└── NextAttemptAt?
```

---

# 139. Outbox ≠ event queue

Outbox pertenece al límite transaccional de persistencia.

La queue/broker es infraestructura de entrega.

---

# 140. Outbox atomicity

La mutación y el outbox record deberán escribirse en la misma transacción local cuando se pretenda atomicidad.

---

# 141. Cross-database outbox

No deberá afirmarse atomicidad si:

```text
business data
```

y:

```text
outbox
```

están en transacciones independientes sin protocolo distribuido.

---

# 142. Outbox dispatch

Deberá ser:

```text
idempotent
retry-aware
observable
```

---

# 143. At-least-once delivery

Una implementación típica puede ofrecer:

```text
at-least-once
```

por lo que consumidores deberán soportar duplicados.

---

# 144. Exactly-once claims

VoltStack no deberá afirmar:

```text
exactly once
```

sin demostrar todas las fronteras involucradas.

---

# 145. EventId for deduplication

El consumidor podrá utilizar:

```text
EventId
```

para idempotencia/deduplicación.

---

# 146. Delivery ≠ processing

```text
Event delivered
≠
Consumer successfully processed event
```

---

# 147. Publication state

Podrá distinguir:

```text
PENDING
CLAIMED
PUBLISHED
FAILED
DEAD_LETTER
UNKNOWN
```

según infraestructura.

---

# 148. Event retries

Retry de publicación:

```text
≠
retry database transaction
```

---

# 149. UNKNOWN transaction + events

Si commit es `UNKNOWN`, no deberá publicarse inmediatamente un evento:

```text
EntityChangesCommitted
```

---

# 150. UNKNOWN reconciliation

Podrá requerir:

```text
transaction reconciliation
outbox inspection
business state verification
```

antes de publicar un hecho committed.

---

# 151. Events during rollback

Eventos internos de ejecución pueden haber ocurrido antes del rollback.

Esto es válido si su semántica dice:

```text
statement executed
```

no:

```text
business state committed
```

---

# 152. Rolled-back persistence

Ejemplo:

```text
EntityUpdateExecuted
TransactionRolledBack
```

es una secuencia válida.

---

# 153. Consumer semantics

Por ello consumidores deberán seleccionar correctamente entre:

```text
execution events
```

y:

```text
committed events
```

---

# 154. Public defaults

Los eventos públicos orientados a aplicaciones deberían favorecer hechos con semántica clara y estable.

Eventos internos de bajo nivel podrán permanecer:

```text
internal/debug/telemetry
```

---

# 155. Event exposure policy

Se propone:

```text
INTERNAL
FRAMEWORK
APPLICATION
PUBLIC_DURABLE
```

---

# 156. INTERNAL

Sólo Database.

---

# 157. FRAMEWORK

Consumible por otros módulos oficiales de VoltStack.

---

# 158. APPLICATION

Consumible por listeners de la aplicación.

---

# 159. PUBLIC_DURABLE

Diseñado para publicación durable/external integration.

---

# 160. Exposure ≠ delivery mode

Son dimensiones distintas.

---

# 161. Event mapping

Un evento interno podrá transformarse:

```text
InternalDatabaseEvent
        ↓
DatabasePublicEventMapper
        ↓
PublicDatabaseEvent
```

---

# 162. Mapping advantages

Permite:

```text
redaction
versioning
payload stabilization
aggregation
renaming
compatibility
```

---

# 163. One internal → zero public

Muchos eventos internos no necesitarán publicación.

---

# 164. Many internal → one public

Ejemplo:

```text
InsertExecuted
UpdateExecuted
RelationshipMutationExecuted
TransactionCommitted
```

pueden producir:

```text
EntityChangesCommitted
```

agregado.

---

# 165. One internal → many public

Será posible, aunque deberá evitarse proliferación innecesaria.

---

# 166. Event registry

Se propone:

```text
DatabaseEventRegistry
```

para registrar definiciones públicas.

---

# 167. Event definition

Conceptualmente:

```php
final readonly class DatabaseEventDefinition
{
    public function __construct(
        public EventName $name,
        public EventVersion $version,
        public EventExposure $exposure,
        public EventDeliveryPolicy $delivery,
        public EventFailurePolicy $failurePolicy,
    ) {}
}
```

---

# 168. Registry freeze

Después del bootstrap:

```text
DatabaseEventRegistry
→ FROZEN
```

en producción.

---

# 169. Runtime registration

No deberán añadirse arbitrariamente definiciones globales por request.

---

# 170. Plugin events

Plugins podrán registrar:

```text
event listeners
event subscribers
custom event definitions
event mappers
```

durante bootstrap.

---

# 171. Plugin namespace

Eventos custom deberán utilizar namespace estable:

```text
vendor.package.database.*
```

o convención equivalente.

---

# 172. Plugin collision

Dos plugins no deberán registrar silenciosamente la misma identidad incompatible.

---

# 173. Event schema compatibility

Para eventos públicos/durables deberá existir una política de compatibilidad.

---

# 174. Compatible changes

Podrían incluir:

```text
adding optional fields
adding metadata
```

según contrato.

---

# 175. Breaking changes

Ejemplos:

```text
renaming required field
changing meaning
changing type
removing field
```

deberán requerir nueva versión.

---

# 176. Event schema ≠ PHP constructor signature

La compatibilidad durable debe definirse a nivel del contrato serializado.

---

# 177. Event serializer

Para eventos asíncronos:

```text
PublicDatabaseEvent
↓
DatabaseEventSerializer
↓
Safe Serialized Envelope
```

---

# 178. Serialization safety

Aplicarán principios similares a Cache:

```text
versioned
deterministic
safe
no arbitrary object graph
```

---

# 179. Entity serialization

No serializar managed entities directamente.

---

# 180. Exception serialization

No enviar objetos Throwable completos.

---

# 181. Failure representation

Preferir:

```text
error category
safe code
safe message
SQLSTATE?
vendor code?
```

según política.

---

# 182. Stack traces

No deberán incluirse en eventos públicos por defecto.

---

# 183. Event security classification

Cada definición podrá declarar:

```text
PUBLIC
INTERNAL
CONFIDENTIAL
RESTRICTED
```

o sistema equivalente.

---

# 184. Payload redaction

Antes de cruzar el integration boundary:

```text
Internal Event
↓
Redaction
↓
Public Event
```

---

# 185. Credentials invariant

Nunca deberán publicarse:

```text
password
token
private key
secret
raw credential DSN
```

---

# 186. Sensitive query data

No deberá publicarse automáticamente:

```text
raw parameter values
PII
payment data
authentication secrets
```

---

# 187. Tenant security

Un listener registrado dentro del contexto de Tenant A no deberá recibir accidentalmente payload privado de Tenant B debido a estado global residual.

---

# 188. Persistent runtime isolation

Especialmente importante para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 189. Listener registry lifetime

Definiciones/listeners inmutables pueden ser:

```text
worker-shared
```

---

# 190. Event dispatch context

Estado mutable como:

```text
current transaction
current tenant
current request
current causation chain
```

deberá ser:

```text
operation/request scoped
```

---

# 191. No static current event

Prohibido:

```php
static ?DatabaseEvent $currentEvent;
```

como contexto global.

---

# 192. FrankenPHP

Modelo:

```text
FrankenPHP Worker
├── Frozen Event Registry
├── Shared Dispatcher Adapter
│
├── Request A
│   └── Event Dispatch Context A
│
├── Request B
│   └── Event Dispatch Context B
│
└── Request C
    └── Event Dispatch Context C
```

---

# 193. RoadRunner

Aplicarán las mismas reglas de worker reuse.

---

# 194. OpenSwoole

El contexto deberá ser:

```text
coroutine-safe
```

y nunca compartirse accidentalmente entre coroutines.

---

# 195. Event recursion

Listeners pueden ejecutar nuevas operaciones Database.

Ejemplo:

```text
QueryExecuted
↓
listener
↓
another query
```

---

# 196. Recursion risk

Esto puede provocar:

```text
infinite loops
event storms
N+1 behavior
unexpected transactions
```

---

# 197. Reentrancy policy

El sistema deberá definir:

```text
ALLOW
GUARD
REJECT
```

por tipo de listener/evento cuando sea necesario.

---

# 198. Event depth

Podrá mantenerse:

```text
DispatchDepth
```

para diagnostics y protección.

---

# 199. Hard recursion limit

Podrá existir un límite defensivo configurable.

---

# 200. Event storm protection

Eventos de alta frecuencia podrán:

```text
aggregate
sample
remain internal
```

---

# 201. Event batching

Ejemplo:

En vez de 100,000 eventos:

```text
RowHydrated
```

podrá producirse:

```text
HydrationBatchCompleted
rows = 100000
```

---

# 202. Event batching ≠ loss of correctness

Sólo será válido para eventos observacionales donde no se requiera granularidad individual.

---

# 203. Listener side effects

Listeners que realizan:

```text
email
HTTP request
queue publish
filesystem
external API
```

introducen fallos externos.

---

# 204. Pre-commit external side effects

Deberán evitarse cuando el side effect representa un cambio que sólo debería ocurrir después del commit.

---

# 205. Example anti-pattern

```text
BEGIN
↓
UPDATE order
↓
send email
↓
ROLLBACK
```

Resultado:

```text
email says success
DB says rollback
```

---

# 206. Correct coordination

Preferir:

```text
BEGIN
↓
UPDATE
↓
register afterCommit/outbox event
↓
COMMIT
↓
side effect
```

---

# 207. External side effect failure

Si ocurre después del commit:

```text
DB remains committed
```

y el side effect deberá:

```text
retry
dead-letter
alert
reconcile
```

según política.

---

# 208. Event listener transactions

Un listener podrá iniciar otra transacción sólo mediante APIs normales.

No heredará mágicamente atomicidad.

---

# 209. REQUIRES_NEW

Si la arquitectura Transaction soporta:

```text
REQUIRES_NEW
```

su semántica deberá permanecer explícita.

---

# 210. Listener transaction ≠ parent transaction

Especialmente después de commit.

---

# 211. Event + Cache integration

Flujo:

```text
DB Mutation
↓
Transaction Commit
↓
Cache Invalidation
↓
DatabaseCacheInvalidated event
```

si se desea observar el resultado.

---

# 212. Event-before-cache-invalidated

Podrá existir:

```text
DatabaseCacheInvalidating
```

internamente, pero no deberá permitir cancelar una invalidación requerida para correctness.

---

# 213. Event + Telemetry integration

El documento siguiente:

```text
316_DATABASE_TELEMETRY_INTEGRATION_SYSTEM.md
```

definirá cómo los eventos se correlacionan con:

```text
traces
metrics
logs
profiles
diagnostics
```

---

# 214. Event ≠ Telemetry

No todo evento debe convertirse en:

```text
metric
log
span
```

---

# 215. Telemetry listener

Un adapter podrá consumir eventos relevantes, pero las métricas críticas de alto rendimiento podrán instrumentarse directamente en puntos especializados.

---

# 216. Avoid double instrumentation

Si:

```text
QueryExecutor
```

ya registra duración, un listener no deberá producir otra medición incompatible sin coordinación.

---

# 217. Event + Authorization

Listeners administrativos deberán respetar:

```text
AuthorizationContext
```

cuando intenten realizar acciones.

---

# 218. Receiving event ≠ authority

Un listener que recibe:

```text
MigrationCompleted
```

no adquiere automáticamente permisos administrativos.

---

# 219. Event + Authentication

No deberán enviarse credenciales/autenticadores.

Podrá existir una identidad segura del actor cuando Audit/Authorization lo requiera.

---

# 220. Event + Jobs/Queues

La integración futura:

```text
320_DATABASE_JOB_AND_QUEUE_INTEGRATION_SYSTEM.md
```

podrá transportar eventos durables mediante Queue.

---

# 221. Queue ≠ Event System

Aunque puedan integrarse:

```text
Event
≠
Job
```

---

# 222. Event + HTTP lifecycle

En requests HTTP:

```text
Request
↓
Database Operation
↓
Events
↓
Request end
```

cualquier evento diferido in-memory deberá resolverse antes del scope reset o transferirse a infraestructura durable.

---

# 223. Scope termination

No deberá quedar:

```text
pending afterCommit callback
pending event
transaction event buffer
```

en un worker después de finalizar el request.

---

# 224. Request reset

El reset deberá limpiar:

```text
dispatch context
transaction event buffers
correlation state
non-durable pending events
```

---

# 225. Worker leak invariant

Un evento de Request A nunca deberá aparecer como parte de Request B.

---

# 226. Event Buffer

Para transacciones se propone:

```text
TransactionEventBuffer
```

---

# 227. TransactionEventBuffer

Puede almacenar:

```text
pending afterCommit events
pending afterRollback events
domain publication requests
cache-related publication requests
```

según responsabilidades.

---

# 228. Buffer scope

```text
TransactionContext
```

será su owner natural.

---

# 229. Nested transaction events

La semántica deberá seguir el modelo real:

```text
JOIN
SAVEPOINT
REJECT
REQUIRES_NEW
```

---

# 230. JOIN

Eventos afterCommit pertenecen al commit físico externo.

---

# 231. SAVEPOINT

Liberar savepoint:

```text
≠
outer transaction committed
```

---

# 232. Savepoint rollback

Los eventos correspondientes al trabajo revertido al savepoint deberán eliminarse o marcarse según su semántica.

---

# 233. REQUIRES_NEW

Posee boundary de commit independiente.

---

# 234. Event ownership

Cada pending event deberá saber a qué:

```text
transaction scope
```

pertenece.

---

# 235. Outermost commit

Para transacciones joined:

```text
afterCommit
```

ocurrirá sólo después del commit físico real.

---

# 236. Retry integration

Si una transacción completa se reintenta:

```text
Attempt 1
→ rollback/deadlock

Attempt 2
→ commit
```

no deberán publicarse como committed los eventos del Attempt 1.

---

# 237. Attempt identity

Puede utilizarse:

```text
TransactionAttemptId
```

para diagnostics.

---

# 238. Logical operation identity

El mismo:

```text
LogicalTransactionId
```

puede abarcar múltiples attempts.

---

# 239. Retry-safe events

Eventos internos de attempt podrán existir.

Eventos committed sólo corresponden al attempt exitoso.

---

# 240. Deadlock event

Podrá emitirse:

```text
DatabaseTransactionDeadlockDetected
```

sin afirmar que toda operación lógica falló si será reintentada.

---

# 241. Retry event

Podrá existir:

```text
DatabaseTransactionRetryScheduled
```

con:

```text
attempt
reason
delay
```

sin datos sensibles.

---

# 242. Retry exhausted

Evento:

```text
DatabaseTransactionRetryExhausted
```

podrá representar el final lógico.

---

# 243. Event performance model

Sea:

```text
Tdb = Database operation time
Te = event dispatch overhead
```

el coste observado será:

```text
Ttotal = Tdb + Te
```

para listeners síncronos.

---

# 244. Event overhead budget

Los eventos de alta frecuencia deberán tener presupuestos explícitos.

---

# 245. Query event overhead

No deberá degradar significativamente:

```text
simple query
prepared execution
streaming
bulk operations
```

por defecto.

---

# 246. Lazy payload construction

Datos caros de diagnostics podrán construirse sólo si existen consumidores relevantes.

---

# 247. No eager stack trace

No generar stack trace completo para cada query sólo por si algún listener lo necesita.

---

# 248. Listener count

Un número elevado de listeners podrá medirse mediante Telemetry/Diagnostics.

---

# 249. Event benchmarks

Deberán incluir:

```text
no listeners
1 listener
multiple listeners
public mapping
serialization
outbox write
async publication
high-frequency query events
```

---

# 250. Backpressure

La publicación asíncrona deberá definir qué ocurre si:

```text
broker unavailable
queue full
publisher saturated
```

---

# 251. Durable event

No deberá descartarse silenciosamente.

---

# 252. Best-effort event

Podrá descartarse bajo policy explícita y observable.

---

# 253. Event delivery policy

Conceptualmente:

```text
INLINE
AFTER_COMMIT
BEST_EFFORT_ASYNC
DURABLE_OUTBOX
```

---

# 254. INLINE

Se despacha inmediatamente dentro del flujo actual.

---

# 255. AFTER_COMMIT

Sólo después de commit confirmado.

---

# 256. BEST_EFFORT_ASYNC

No ofrece durabilidad fuerte.

---

# 257. DURABLE_OUTBOX

La intención de publicación queda registrada transaccionalmente.

---

# 258. Delivery policy ≠ exposure

Por ejemplo:

```text
APPLICATION + INLINE
APPLICATION + AFTER_COMMIT
PUBLIC_DURABLE + DURABLE_OUTBOX
```

son combinaciones posibles.

---

# 259. Event failure policy

Conceptualmente:

```text
PROPAGATE
ISOLATE
RECORD_AND_CONTINUE
RETRY_PUBLICATION
DEAD_LETTER
```

dependiendo de fase y delivery mode.

---

# 260. Failure policy validation

No todas las combinaciones serán válidas.

Ejemplo:

```text
AFTER_COMMIT + ROLLBACK_TRANSACTION
```

es semánticamente imposible.

---

# 261. Configuration

Ejemplo conceptual:

```php
'database' => [
    'events' => [
        'enabled' => true,

        'query' => [
            'public' => false,
        ],

        'transaction' => [
            'public' => true,
        ],

        'orm' => [
            'public' => true,
        ],

        'durable' => [
            'driver' => 'outbox',
        ],
    ],
],
```

---

# 262. Configuration ≠ event registry mutation

Config define policy.

Registry define contratos.

---

# 263. Container integration

El documento 312 deberá permitir registrar:

```text
DatabaseEventDispatcher
DatabaseEventPublisher
DatabaseEventMapper
DatabaseEventSerializer
DatabaseEventRegistry
DatabaseEventPolicyResolver
```

---

# 264. Suggested lifetimes

```text
Event Registry
→ shared immutable

Event Definitions
→ shared immutable

Dispatcher Adapter
→ shared/stateless where possible

Transaction Event Buffer
→ transaction scoped

Dispatch Context
→ operation scoped
```

---

# 265. No singleton mutable event context

Regla obligatoria.

---

# 266. Testing architecture

Se propone:

```text
tests/Quantum/Database/Integration/Event/
├── DatabaseEventIntegrationTest.php
├── DatabaseEventDispatcherTest.php
├── NullDatabaseEventDispatcherTest.php
├── VoltStackEventAdapterTest.php
├── DatabaseEventEnvelopeTest.php
├── DatabaseEventOrderingTest.php
├── DatabaseEventRedactionTest.php
├── QueryEventIntegrationTest.php
├── ConnectionEventIntegrationTest.php
├── TransactionEventIntegrationTest.php
├── TransactionUnknownEventTest.php
├── TransactionRetryEventTest.php
├── SavepointEventTest.php
├── OrmEventIntegrationTest.php
├── PersistenceEventIntegrationTest.php
├── AfterCommitEventTest.php
├── RollbackEventTest.php
├── ListenerFailureTest.php
├── OutboxEventIntegrationTest.php
├── EventSerializationTest.php
├── EventVersioningTest.php
├── TenantEventIsolationTest.php
├── ShardEventIsolationTest.php
├── PersistentRuntimeEventTest.php
├── EventRecursionTest.php
└── EventPerformanceTest.php
```

---

# 267. Query event test

Demostrar:

```text
Preparing
→ Executing
→ Executed
```

para éxito.

---

# 268. Query failure test

Demostrar:

```text
Preparing
→ Executing
→ Failed
```

sin emitir:

```text
Executed(success)
```

---

# 269. Commit test

```text
TransactionBeginning
TransactionBegan
TransactionCommitting
TransactionCommitted
```

en orden.

---

# 270. UNKNOWN commit test

Deberá demostrar:

```text
TransactionCommitting
TransactionOutcomeUnknown
```

y ausencia de:

```text
TransactionCommitted
TransactionRolledBack
```

sin evidencia.

---

# 271. Rollback test

Debe demostrar que eventos `afterCommit` pendientes:

```text
are not published
```

---

# 272. Savepoint rollback test

Los eventos pertenecientes al trabajo revertido deberán seguir la policy definida.

---

# 273. Retry test

Intentos fallidos no deberán publicar eventos committed.

---

# 274. Listener failure before commit

Deberá probarse si:

```text
propagates
aborts
is isolated
```

según definición.

---

# 275. Listener failure after commit

Debe demostrar:

```text
transaction outcome remains COMMITTED
```

---

# 276. Outbox crash test

Escenario:

```text
DB commit
↓
process crash
↓
restart
↓
outbox dispatcher
↓
event published
```

---

# 277. Duplicate delivery test

El mismo EventId podrá entregarse más de una vez bajo at-least-once sin generar una identidad nueva.

---

# 278. Redaction test

Eventos nunca deberán contener:

```text
password
token
secret DSN
sensitive raw parameter
```

---

# 279. Tenant isolation test

Eventos de Tenant A no deberán incorporar estado residual de Tenant B.

---

# 280. Persistent runtime test

Miles de requests deberán demostrar:

```text
no listener leakage
no event buffer leakage
no correlation leakage
no transaction event leakage
```

---

# 281. Coroutine test

OpenSwoole deberá demostrar aislamiento entre coroutines.

---

# 282. Event recursion test

Un listener que ejecuta Database deberá:

```text
preserve causation
respect depth policy
avoid accidental infinite recursion
```

---

# 283. Performance test

Medir:

```text
dispatch overhead
mapping overhead
serialization overhead
outbox overhead
listener overhead
high-frequency event cost
```

---

# 284. Formal event model

Sea:

```text
O = database operation
R = semantic result/outcome
E = event
```

Entonces:

```text
E = Describe(O, R)
```

pero:

```text
E ≠ O
```

y:

```text
E ≠ R
```

---

# 285. Event truth requirement

Para un evento que afirma un hecho:

```text
Publish(E_fact)
⇒
Evidence(Fact(E_fact))
```

---

# 286. Commit event rule

```text
Emit(TransactionCommitted)
iff
CommitOutcome = COMMITTED
```

---

# 287. Unknown rule

```text
CommitOutcome = UNKNOWN
⇒
¬Emit(TransactionCommitted)
∧
¬Emit(TransactionRolledBack)
```

salvo que evidencia posterior resuelva el outcome.

---

# 288. AfterCommit rule

Sea:

```text
A = afterCommit event
```

Entonces:

```text
Publish(A)
only if
TransactionOutcome = COMMITTED
```

---

# 289. Rollback rule

```text
TransactionOutcome = ROLLED_BACK
⇒
Discard(PendingAfterCommitEvents)
```

---

# 290. Listener failure after commit

```text
Committed(T)
∧
ListenerFailure(E)
⇒
Committed(T)
```

---

# 291. Durable event rule

Para publicación durable:

```text
DurableIntent(E)
∧
BusinessMutation(M)
```

deberán quedar en el mismo atomic boundary cuando se prometa atomicidad:

```text
Commit(M ∧ Outbox(E))
```

---

# 292. Event ordering model

Para eventos dentro de una operación:

```text
sequence(E1) < sequence(E2)
```

podrá establecer:

```text
E1 happened-before E2
```

dentro del scope definido.

No implica orden global distribuido.

---

# 293. Security rule

```text
PublicPayload(E)
=
Redact(
    InternalPayload(E),
    EventSecurityPolicy
)
```

---

# 294. Context isolation

Para dos requests concurrentes:

```text
Context(R1) ∩ MutableEventState(R2) = ∅
```

salvo recursos inmutables explícitamente compartidos.

---

# 295. Core invariants

## DB-EVT-INT-001

Event ≠ Command.

## DB-EVT-INT-002

Event ≠ Transaction Outcome.

## DB-EVT-INT-003

Event ≠ Domain Event.

## DB-EVT-INT-004

Event ≠ Telemetry.

## DB-EVT-INT-005

Event ≠ Audit Record.

## DB-EVT-INT-006

Database funcionará sin Event System externo.

## DB-EVT-INT-007

Eventos internos no serán automáticamente API pública.

## DB-EVT-INT-008

Eventos públicos serán inmutables.

## DB-EVT-INT-009

Listeners no redefinirán Database Truth.

## DB-EVT-INT-010

Eventos no sustituirán APIs de extensión apropiadas.

---

# 296. Transaction invariants

## DB-EVT-INT-011

Committing ≠ Committed.

## DB-EVT-INT-012

Executed Mutation ≠ Committed Mutation.

## DB-EVT-INT-013

FlushCompleted ≠ TransactionCommitted.

## DB-EVT-INT-014

UNKNOWN ≠ COMMITTED.

## DB-EVT-INT-015

UNKNOWN ≠ ROLLED_BACK.

## DB-EVT-INT-016

AfterCommit sólo se ejecutará tras commit confirmado.

## DB-EVT-INT-017

Rollback eliminará pending afterCommit events.

## DB-EVT-INT-018

AfterCommit failure no revertirá un commit.

## DB-EVT-INT-019

Savepoint release ≠ outer commit.

## DB-EVT-INT-020

Retry attempt fallido no publicará committed events.

---

# 297. ORM invariants

## DB-EVT-INT-021

persist() ≠ INSERT.

## DB-EVT-INT-022

remove() ≠ DELETE inmediato.

## DB-EVT-INT-023

Entity event no implicará commit salvo que su nombre/contrato lo indique.

## DB-EVT-INT-024

Managed entity no será payload durable por defecto.

## DB-EVT-INT-025

Entity snapshot ≠ managed entity.

## DB-EVT-INT-026

Hydration events no modificarán dirty state.

## DB-EVT-INT-027

Lifecycle listeners no bypassarán UnitOfWork.

## DB-EVT-INT-028

ORM events no generarán domain events automáticamente.

## DB-EVT-INT-029

Relationship event no redefinirá relationship ownership.

## DB-EVT-INT-030

IdentityMap invariants permanecerán intactas ante listeners.

---

# 298. Publication invariants

## DB-EVT-INT-031

Dispatch ≠ Publish.

## DB-EVT-INT-032

Synchronous ≠ Asynchronous.

## DB-EVT-INT-033

Public ≠ Async.

## DB-EVT-INT-034

Durable event utilizará infraestructura durable.

## DB-EVT-INT-035

In-memory afterCommit no se declarará durable.

## DB-EVT-INT-036

Outbox ≠ Queue.

## DB-EVT-INT-037

Delivery ≠ Processing.

## DB-EVT-INT-038

At-least-once consumers tolerarán duplicados.

## DB-EVT-INT-039

Exactly-once no se afirmará sin evidencia completa.

## DB-EVT-INT-040

EventId será estable durante retries de publicación.

---

# 299. Security invariants

## DB-EVT-INT-041

Credentials nunca aparecerán en eventos.

## DB-EVT-INT-042

Raw parameters no serán públicos por defecto.

## DB-EVT-INT-043

Raw SQL no será público por defecto.

## DB-EVT-INT-044

Stack traces no serán payload público por defecto.

## DB-EVT-INT-045

Public events pasarán por redaction policy.

## DB-EVT-INT-046

Authorization no será sustituida por recibir un evento.

## DB-EVT-INT-047

Tenant context no cruzará scopes.

## DB-EVT-INT-048

Event serialization será segura.

## DB-EVT-INT-049

Arbitrary PHP object unserialization estará prohibido.

## DB-EVT-INT-050

Event metadata sensible será clasificada.

---

# 300. Runtime invariants

## DB-EVT-INT-051

Event registry podrá compartirse si es inmutable.

## DB-EVT-INT-052

Dispatch context será scoped.

## DB-EVT-INT-053

TransactionEventBuffer será transaction-scoped.

## DB-EVT-INT-054

No existirá static current event global.

## DB-EVT-INT-055

Request A no filtrará eventos a Request B.

## DB-EVT-INT-056

FrankenPHP worker reuse será seguro.

## DB-EVT-INT-057

RoadRunner worker reuse seguirá las mismas invariantes.

## DB-EVT-INT-058

OpenSwoole será coroutine-safe.

## DB-EVT-INT-059

Request reset limpiará pending non-durable events.

## DB-EVT-INT-060

Worker shutdown manejará buffers pendientes explícitamente.

---

# 301. Extensibility invariants

## DB-EVT-INT-061

Plugins registrarán listeners durante bootstrap.

## DB-EVT-INT-062

Registry estará frozen en runtime normal.

## DB-EVT-INT-063

Event name collisions serán rechazadas.

## DB-EVT-INT-064

Breaking public event changes requerirán nueva versión.

## DB-EVT-INT-065

PHP FQCN no será la única identidad durable.

## DB-EVT-INT-066

Listener priority ≠ dependency.

## DB-EVT-INT-067

Events no reemplazarán Interceptors.

## DB-EVT-INT-068

Events no reemplazarán Policies.

## DB-EVT-INT-069

Events no reemplazarán Middleware.

## DB-EVT-INT-070

Events no reemplazarán Query/Compiler extension points.

---

# 302. Operational invariants

## DB-EVT-INT-071

Event delivery failure será observable.

## DB-EVT-INT-072

Durable event failure no será descartado silenciosamente.

## DB-EVT-INT-073

Best-effort loss sólo ocurrirá bajo policy explícita.

## DB-EVT-INT-074

Event storm protection preservará semántica.

## DB-EVT-INT-075

Event recursion será acotable.

## DB-EVT-INT-076

Listener side effects externos respetarán transaction boundary.

## DB-EVT-INT-077

Pre-commit side effects no serán default para committed facts.

## DB-EVT-INT-078

Outbox dispatcher será retry-aware.

## DB-EVT-INT-079

Dead-letter state será observable cuando exista.

## DB-EVT-INT-080

UNKNOWN publication outcome no será reportado como éxito confirmado.

---

# 303. Performance invariants

## DB-EVT-INT-081

Query event overhead será medible.

## DB-EVT-INT-082

Payload caro podrá construirse lazy.

## DB-EVT-INT-083

Stack traces no se generarán innecesariamente.

## DB-EVT-INT-084

High-volume events podrán agregarse.

## DB-EVT-INT-085

No-listener path tendrá overhead mínimo.

## DB-EVT-INT-086

Async publication no bloqueará indefinidamente requests.

## DB-EVT-INT-087

Backpressure tendrá policy explícita.

## DB-EVT-INT-088

Outbox polling/dispatch tendrá resource budgets.

## DB-EVT-INT-089

Telemetry no duplicará innecesariamente event instrumentation.

## DB-EVT-INT-090

Event performance optimizations no romperán ordering guarantees declaradas.

---

# 304. Testing invariants

## DB-EVT-INT-091

Event ordering será probado.

## DB-EVT-INT-092

Commit/rollback events serán probados con DB real.

## DB-EVT-INT-093

UNKNOWN commit será probado.

## DB-EVT-INT-094

AfterCommit failure será probado.

## DB-EVT-INT-095

Outbox crash recovery será probado.

## DB-EVT-INT-096

Duplicate delivery será probado.

## DB-EVT-INT-097

Redaction será probada.

## DB-EVT-INT-098

Tenant isolation será probada.

## DB-EVT-INT-099

Persistent worker isolation será probada.

## DB-EVT-INT-100

Listener recursion será probada.

---

# 305. Anti-pattern: Event as Command

Incorrecto:

```text
QueryExecuting event
→ listener changes query arbitrarily
```

Utilizar Query Extension/Interceptor.

---

# 306. Anti-pattern: Event as transaction outcome

Incorrecto:

```text
TransactionCommitting
→ assume committed
```

---

# 307. Anti-pattern: Publish before commit

Incorrecto:

```text
UPDATE
↓
publish OrderPaid
↓
COMMIT
```

---

# 308. Anti-pattern: AfterCommit rollback illusion

Incorrecto:

```text
COMMIT
↓
listener fails
↓
report transaction rolled back
```

---

# 309. Anti-pattern: Managed entity in queue

Incorrecto:

```php
$queue->publish(new EntityUpdated($managedEntity));
```

---

# 310. Anti-pattern: Every row emits public event

Una hidratación de un millón de filas no debería producir por defecto un millón de eventos públicos.

---

# 311. Anti-pattern: Global mutable event context

```php
static $currentTransaction;
static $currentTenant;
```

Prohibido.

---

# 312. Anti-pattern: Listener performs hidden query

Un listener observacional que silenciosamente ejecuta queries por cada `EntityLoaded` puede introducir N+1.

Debe ser detectable y explícito.

---

# 313. Anti-pattern: Listener as security bypass

Recibir un evento administrativo no otorga autoridad administrativa.

---

# 314. Anti-pattern: Raw SQL everywhere

No utilizar SQL raw como payload/event identity por defecto.

---

# 315. Anti-pattern: Events as telemetry transport

No enviar millones de eventos sólo para calcular una métrica que puede instrumentarse directamente.

---

# 316. Anti-pattern: Events as audit log

Un Event Bus genérico no sustituye almacenamiento y garantías de auditoría.

---

# 317. Anti-pattern: Ignore UNKNOWN

Incorrecto:

```text
commit error
→ publish rollback event
```

sin evidencia.

---

# 318. Anti-pattern: Retry transaction because listener failed after commit

Puede duplicar escrituras.

---

# 319. Anti-pattern: Exactly-once marketing

No declarar exactly-once únicamente porque EventId existe.

---

# 320. Anti-pattern: Event schema tied to PHP internals

Evitar contratos durables como:

```text
serialized PHP object with FQCN
```

---

# 321. Proposed namespace

```text
VoltStack\Quantum\Database\Integration\Event
```

---

# 322. Proposed structure

```text
src/Quantum/Database/Integration/Event/
├── Contract/
│   ├── DatabaseEvent.php
│   ├── PublicDatabaseEvent.php
│   ├── DatabaseEventDispatcher.php
│   ├── DatabaseEventPublisher.php
│   └── DatabaseEventSerializer.php
│
├── Adapter/
│   ├── NullDatabaseEventDispatcher.php
│   └── VoltStackEventDispatcherAdapter.php
│
├── Envelope/
│   ├── DatabaseEventEnvelope.php
│   ├── EventId.php
│   ├── EventName.php
│   ├── EventVersion.php
│   ├── CorrelationId.php
│   └── CausationId.php
│
├── Registry/
│   ├── DatabaseEventRegistry.php
│   └── DatabaseEventDefinition.php
│
├── Policy/
│   ├── DatabaseEventPolicyResolver.php
│   ├── EventExposure.php
│   ├── EventDeliveryPolicy.php
│   ├── EventFailurePolicy.php
│   └── EventSecurityPolicy.php
│
├── Mapping/
│   ├── DatabasePublicEventMapper.php
│   └── DatabaseEventRedactor.php
│
├── Transaction/
│   ├── TransactionEventBuffer.php
│   ├── AfterCommitEventQueue.php
│   └── AfterRollbackEventQueue.php
│
├── Outbox/
│   ├── DatabaseOutboxRecord.php
│   ├── DatabaseOutboxRepository.php
│   ├── DatabaseOutboxDispatcher.php
│   ├── DatabaseOutboxSerializer.php
│   └── DatabaseOutboxRetryPolicy.php
│
├── Runtime/
│   ├── DatabaseEventDispatchContext.php
│   └── EventRecursionGuard.php
│
├── Security/
│   └── DatabaseEventPayloadPolicy.php
│
├── Diagnostics/
│   └── DatabaseEventDiagnostics.php
│
└── Telemetry/
    └── DatabaseEventTelemetryBridge.php
```

---

# 323. Event packages by domain

Los eventos públicos podrán organizarse:

```text
Event/
├── Query/
├── Connection/
├── Transaction/
├── ORM/
├── Persistence/
├── Schema/
├── Migration/
├── Cache/
├── Backup/
├── Restore/
└── Administration/
```

---

# 324. Dependency model

Permitido:

```text
Query Engine
Transaction
ORM
Schema
Migration
      ↓
Database Event Contracts
      ↓
Database Event Integration
      ↓
VoltStack Event System
```

No permitido:

```text
VoltStack Event System
      ↓
Query Planner internals
```

como dependencia inversa.

---

# 325. Complete synchronous flow

```text
Application
    ↓
Database API
    ↓
Query Executor
    ↓
Internal Event
    ↓
DatabaseEventDispatcher
    ↓
Internal listeners
    ↓
Public Event Mapper
    ↓
VoltStack Event Adapter
    ↓
Application Listener
```

---

# 326. Complete transactional flow

```text
Application
    ↓
BEGIN
    ↓
Database mutations
    ↓
Pending events
    ↓
TransactionEventBuffer
    ↓
COMMIT
    │
    ├── COMMITTED
    │      ↓
    │   AfterCommit Events
    │      ↓
    │   Event System
    │
    ├── ROLLED_BACK
    │      ↓
    │   Discard AfterCommit Events
    │
    └── UNKNOWN
           ↓
       Preserve uncertainty
           ↓
       Reconciliation Policy
```

---

# 327. Durable flow

```text
Application
      ↓
BEGIN
      ↓
Business Mutation
      +
Outbox Record
      ↓
COMMIT
      ↓
Outbox Dispatcher
      ↓
Event Serializer
      ↓
VoltStack Event System
      ↓
Queue/Broker
      ↓
Consumer
```

---

# 328. Integration with VoltStack architecture

El sistema completo queda:

```text
                         VoltStack
                            │
       ┌────────────────────┼────────────────────┐
       │                    │                    │
       ▼                    ▼                    ▼
    Database              Events             Telemetry
       │                    ▲                    ▲
       │                    │                    │
       └── Event Adapter ───┘                    │
       │                                         │
       └──────── Event/Telemetry Correlation ────┘
```

Database continúa siendo propietario de:

```text
database semantics
transaction outcomes
query execution
ORM state
persistence state
```

Event System será propietario de:

```text
listener registration
dispatch infrastructure
publication infrastructure
event transport integration
```

---

# 329. V1 implementation scope

La primera implementación deberá incluir:

```text
DatabaseEvent contract
DatabaseEventEnvelope
DatabaseEventDispatcher
Null dispatcher
VoltStack Event adapter
Event registry
Event definitions
Query events
Connection events
Transaction events
ORM lifecycle events
Persistence events
AfterCommit events
Rollback handling
UNKNOWN outcome event
Correlation
Redaction
Listener failure policy
Persistent runtime reset
Testing
```

---

# 330. V1 durable events

Outbox podrá implementarse inicialmente como módulo opcional, pero la arquitectura V1 deberá reservar sus contratos.

No deberá diseñarse `afterCommit` como si fuera durable.

---

# 331. V2

Agregar:

```text
Outbox implementation
async event publication
retry policies
dead-letter handling
event schema versioning tools
event diagnostics CLI
advanced event aggregation
```

---

# 332. V3

Agregar:

```text
distributed publication adapters
broker integrations
event schema registry
advanced replay tooling
outbox partitioning
high-volume event batching
```

---

# 333. V4

Posibles capacidades:

```text
change-data-capture integration
event stream projections
cross-service event contracts
advanced deduplication
event delivery observability
```

sin convertir Database en un Event Sourcing framework.

---

# 334. Event Sourcing boundary

Importante:

```text
Database Event Integration
≠
Event Sourcing
```

VoltStack podrá soportar Event Sourcing mediante otro paquete, pero Database no deberá asumir que:

```text
events are primary state
```

---

# 335. CDC boundary

Igualmente:

```text
Database Events
≠
Change Data Capture
```

CDC observa cambios a nivel DB/log.

Database Events representan hechos del runtime de VoltStack.

Ambos podrán integrarse en el futuro.

---

# 336. Checklist de implementación

Antes de considerar completa esta integración:

- [ ] Database funciona sin Event System externo.
- [ ] Null dispatcher disponible.
- [ ] VoltStack Event adapter implementado.
- [ ] eventos internos separados de públicos.
- [ ] dispatch separado de publish.
- [ ] sync separado de async.
- [ ] EventEnvelope estable.
- [ ] EventId.
- [ ] EventName.
- [ ] EventVersion.
- [ ] CorrelationId.
- [ ] CausationId.
- [ ] query events.
- [ ] connection events.
- [ ] transaction events.
- [ ] ORM events.
- [ ] persistence events.
- [ ] schema/migration events.
- [ ] Committing ≠ Committed.
- [ ] flush ≠ commit.
- [ ] persist ≠ insert.
- [ ] remove ≠ delete.
- [ ] UNKNOWN outcome preserved.
- [ ] TransactionEventBuffer.
- [ ] afterCommit.
- [ ] rollback discard.
- [ ] nested/savepoint semantics.
- [ ] retry semantics.
- [ ] listener failure policy.
- [ ] afterCommit failure preserves COMMITTED.
- [ ] payload redaction.
- [ ] credentials prohibited.
- [ ] managed entities prohibited in durable payloads.
- [ ] event serialization versioned.
- [ ] persistent runtime isolation.
- [ ] FrankenPHP tests.
- [ ] RoadRunner compatibility.
- [ ] OpenSwoole coroutine safety.
- [ ] event recursion guard.
- [ ] performance benchmarks.
- [ ] outbox contracts.
- [ ] durable publication tests.
- [ ] duplicate delivery tests.
- [ ] tenant isolation tests.
- [ ] shard context tests.

---

# 337. Principio definitivo

La integración del sistema de eventos de Database deberá seguir:

```text
Database decides what happened.
Event describes what happened.
Event System distributes that description.
Listeners react to that description.
```

Nunca:

```text
Listener decides retroactively
what the database did.
```

---

# 338. Regla transaccional definitiva

La arquitectura deberá preservar:

```text
Statement Executed
        ≠
Transaction Committed
```

```text
Flush Completed
        ≠
Transaction Committed
```

```text
Commit Attempted
        ≠
Transaction Committed
```

y:

```text
Transaction COMMITTED
        +
AfterCommit Listener FAILED
        =
Database remains COMMITTED
```

---

# 339. Resultado arquitectónico

Con esta arquitectura, VoltStack Database podrá integrarse con el Event System general para proporcionar:

```text
query observability
connection lifecycle events
transaction lifecycle events
ORM lifecycle integration
persistence notifications
schema/migration notifications
cache coordination observations
application listeners
plugins
afterCommit workflows
durable outbox publication
queue/broker integration
```

manteniendo las fronteras:

```text
Database
   ↓
Database Event Semantics
   ↓
Integration Boundary
   ↓
VoltStack Event System
   ↓
Listeners / Subscribers / Publishers
```

sin convertir:

```text
events
```

en sustitutos de:

```text
commands
transactions
database state
authorization
telemetry
audit
domain modeling
```

---

# 340. Siguiente documento

```text
316_DATABASE_TELEMETRY_INTEGRATION_SYSTEM.md
```

Definirá la integración entre:

```text
VoltStack/Quantum/Database
        ↕
VoltStack Telemetry
```

incluyendo:

```text
Database telemetry bridge
tracing
metrics
structured logging
query spans
connection spans
transaction spans
ORM spans
migration spans
cache telemetry
event correlation
query fingerprints
database operation IDs
trace propagation
runtime context
tenant/shard-safe observability
cardinality governance
sampling
redaction
sensitive data protection
performance overhead budgets
slow query integration
N+1 telemetry integration
profiler integration
debug toolbar integration
persistent worker isolation
FrankenPHP integration
RoadRunner/OpenSwoole integration
OpenTelemetry adapters
Prometheus metrics
testing
```

manteniendo como reglas centrales:

```text
Telemetry
≠
Database Semantics
```

```text
Observation
≠
Control
```

y:

```text
Telemetry Failure
≠
Database Failure
```