# 214_DATABASE_PERSISTENCE_EVENT_SYSTEM.md

# VoltStack Quantum Database
## Database Persistence Event System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 214 — Database Persistence Event System  
**Bloque:** 20 — Events  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `213_DATABASE_ENTITY_LIFECYCLE_EVENT_SYSTEM.md`  
**Siguiente documento:** `215_DATABASE_EVENT_EXTENSION_SYSTEM.md`

---

# 1. Propósito

`Database Persistence Event System` define la arquitectura de eventos utilizada para observar el proceso mediante el cual el ORM de VoltStack intenta sincronizar su estado administrado con la base de datos.

El sistema cubre:

```text
EntityManager
↓
UnitOfWork
↓
Change Detection
↓
ChangeSets
↓
Persistence Graph
↓
Persistence Planning
↓
Insert / Update / Delete Operations
↓
Dependency Ordering
↓
Flush
↓
Query Engine
↓
Execution Engine
↓
Transaction Coordination
↓
Consistency Reconciliation
```

La regla central será:

> **Un Persistence Event describe una fase o resultado del intento del ORM de sincronizar cambios con la base de datos; no constituye por sí mismo evidencia de que dichos cambios hayan sido committed de forma durable.**

Por tanto:

```text
PersistenceExecuted
≠
TransactionCommitted
```

---

# 2. Posición arquitectónica

El sistema se sitúa entre:

```text
ORM State
```

y:

```text
Query / Execution / Transaction Infrastructure
```

Arquitectura:

```text
EntityManager
     │
     ▼
UnitOfWork
     │
     ▼
ChangeSets
     │
     ▼
Persistence Engine
     │
     ├──────────────► Persistence Event System
     │
     ▼
Persistence Planner
     │
     ▼
Query Model
     │
     ▼
Query Engine
     │
     ▼
Execution Engine
     │
     ▼
Transaction Manager
```

---

# 3. Documentos relacionados

Este sistema especializa y conecta:

```text
118_DATABASE_ENTITY_MANAGER_SYSTEM.md
121_DATABASE_ENTITY_STATE_SYSTEM.md
123_DATABASE_IDENTITY_MAP_SYSTEM.md
124_DATABASE_UNIT_OF_WORK_ARCHITECTURE.md
125_DATABASE_CHANGE_TRACKING_SYSTEM.md
126_DATABASE_ENTITY_SNAPSHOT_SYSTEM.md
127_DATABASE_PERSISTENCE_ENGINE.md
128_DATABASE_PERSISTENCE_PLANNER_SYSTEM.md
129_DATABASE_INSERT_PERSISTENCE_SYSTEM.md
130_DATABASE_UPDATE_PERSISTENCE_SYSTEM.md
131_DATABASE_DELETE_PERSISTENCE_SYSTEM.md
132_DATABASE_FLUSH_SYSTEM.md
133_DATABASE_BATCH_PERSISTENCE_SYSTEM.md
134_DATABASE_PERSISTENCE_CONSISTENCY_SYSTEM.md

164_DATABASE_TRANSACTION_ARCHITECTURE.md
165_DATABASE_TRANSACTION_MANAGER_SYSTEM.md
166_DATABASE_TRANSACTION_CONTEXT_SYSTEM.md
170_DATABASE_TRANSACTION_RETRY_SYSTEM.md

209_DATABASE_EVENT_ARCHITECTURE.md
210_DATABASE_QUERY_EVENT_SYSTEM.md
211_DATABASE_CONNECTION_EVENT_SYSTEM.md
212_DATABASE_TRANSACTION_EVENT_PIPELINE.md
213_DATABASE_ENTITY_LIFECYCLE_EVENT_SYSTEM.md
```

---

# 4. Distinciones fundamentales

VoltStack deberá preservar:

```text
Persistence Event
≠
Entity Lifecycle Event
≠
Query Event
≠
Connection Event
≠
Transaction Event
≠
Domain Event
≠
Cache Event
```

---

# 5. Persistence Event ≠ Entity Lifecycle Event

Ejemplo:

```text
EntityPreUpdate
```

es un evento/hook relacionado con una entidad.

Mientras:

```text
PersistenceUpdateExecuted
```

describe una operación del Persistence Engine.

---

# 6. Persistence Event ≠ Query Event

Una operación:

```text
PersistenceUpdateOperation
```

puede producir una query:

```text
UPDATE users ...
```

pero ambas pertenecen a capas diferentes.

```text
Persistence Operation
↓
Query Model
↓
Query Execution
```

---

# 7. Persistence Event ≠ Transaction Event

```text
PersistenceInsertExecuted
```

solo significa que el statement correspondiente produjo un resultado considerado exitoso.

No significa:

```text
TransactionCommitted
```

---

# 8. Persistence Event ≠ Domain Event

Ejemplo:

```text
PersistenceInsertExecuted
```

no equivale a:

```text
CustomerRegistered
```

El segundo expresa significado de negocio.

---

# 9. Persistence Event ≠ durability proof

Regla crítica:

```text
Statement success
≠
Flush success
≠
Transaction success
≠
Commit durability
```

---

# 10. Concepto de Persistence Operation

VoltStack representará cada intención física de persistencia mediante:

```php
interface PersistenceOperation
{
    public function id(): PersistenceOperationId;

    public function kind(): PersistenceOperationKind;

    public function entityType(): EntityType;
}
```

---

# 11. PersistenceOperationKind

```php
enum PersistenceOperationKind
{
    case INSERT;
    case UPDATE;
    case DELETE;
}
```

Podrán existir posteriormente:

```text
UPSERT
RELATIONSHIP_INSERT
RELATIONSHIP_DELETE
CUSTOM
```

si la arquitectura los requiere.

---

# 12. ChangeSet ≠ Persistence Operation

Un:

```text
ChangeSet
```

describe cambios detectados.

Una:

```text
PersistenceOperation
```

describe trabajo que el Persistence Engine decidió realizar.

---

# 13. Persistence Operation ≠ Query

La operación deberá convertirse posteriormente en:

```text
Query Model
```

a través del Persistence Planner/Engine.

Nunca generará SQL directamente.

---

# 14. Persistence Operation ID

Cada operación tendrá:

```text
PersistenceOperationId
```

para correlación.

Ejemplo:

```text
FlushId: FL-91
PersistenceOperationId: PO-402
QueryId: Q-814
TransactionId: TX-71
```

---

# 15. Flush ID

Cada ejecución de `flush()` deberá poder correlacionarse mediante:

```text
PersistenceFlushId
```

---

# 16. Flush ≠ Transaction

```text
PersistenceFlushId
≠
TransactionId
```

Un flush puede ejecutarse:

```text
inside existing transaction
```

o:

```text
using internally coordinated transaction
```

según política.

---

# 17. Persistence Event contract

```php
interface PersistenceEvent extends DatabaseEvent
{
    public function persistenceContext(): PersistenceEventContext;

    public function phase(): PersistenceEventPhase;
}
```

---

# 18. PersistenceEventContext

```php
final readonly class PersistenceEventContext
{
    public function __construct(
        public PersistenceFlushId $flushId,
        public EntityManagerScopeId $entityManagerScope,
        public ?PersistenceOperationId $operationId,
        public ?TransactionId $transactionId,
        public PersistenceDomain $domain,
        public ?TenantContextReference $tenant,
        public ?ShardId $shard,
        public PersistenceEventMetadata $metadata,
    ) {}
}
```

---

# 19. Context snapshots

El contexto deberá ser:

```text
immutable
bounded
sanitized
```

No una referencia mutable a todo el EntityManager.

---

# 20. Persistence event phases

```php
enum PersistenceEventPhase
{
    case FLUSH;
    case CHANGE_ANALYSIS;
    case PLANNING;
    case INSERT;
    case UPDATE;
    case DELETE;
    case EXECUTION;
    case RECONCILIATION;
}
```

---

# 21. Arquitectura de eventos

```text
flush()
  │
  ▼
PersistenceFlushStarted
  │
  ▼
UnitOfWork Analysis
  │
  ▼
PersistenceChangesCollected
  │
  ▼
PersistencePlanCreated
  │
  ▼
PersistenceExecutionStarted
  │
  ├── INSERT
  ├── UPDATE
  └── DELETE
  │
  ▼
PersistenceExecutionCompleted
  │
  ▼
PersistenceReconciliation
  │
  ▼
PersistenceFlushCompleted
```

En error:

```text
PersistenceFlushStarted
↓
...
↓
Failure
↓
PersistenceFlushFailed
```

o:

```text
PersistenceOutcomeUnknown
```

cuando la realidad no puede demostrarse.

---

# 22. PersistenceFlushStarted

Representará el inicio lógico de:

```php
$entityManager->flush();
```

---

# 23. FlushStarted ≠ transaction started

No necesariamente se habrá abierto todavía una transacción física.

---

# 24. FlushStarted payload

Podrá contener:

```text
FlushId
EntityManagerScopeId
transaction context if already present
tenant
shard/domain
flush mode
timestamp
```

---

# 25. Flush target

VoltStack podrá permitir:

```text
flush()
```

y eventualmente:

```text
flush($entity)
```

o scopes específicos.

El evento deberá describir el target.

---

# 26. FlushTarget

```php
enum PersistenceFlushTargetKind
{
    case UNIT_OF_WORK;
    case ENTITY;
    case ENTITY_SET;
}
```

---

# 27. Partial flush

Si se soporta, deberá documentarse como operación avanzada.

Puede romper fácilmente dependencias entre entidades.

---

# 28. Default

La recomendación será:

```text
UnitOfWork-level flush
```

como comportamiento normal.

---

# 29. Change analysis

Después de iniciar flush:

```text
UnitOfWork
↓
Change Tracking
↓
ChangeSets
```

---

# 30. PersistenceChangesCollected

Podrá emitir un resumen:

```php
final readonly class PersistenceChangeSummary
{
    public function __construct(
        public int $insertCount,
        public int $updateCount,
        public int $deleteCount,
        public int $relationshipChangeCount,
    ) {}
}
```

---

# 31. Change summary ≠ Persistence Plan

Los cambios todavía pueden requerir:

```text
dependency analysis
generated-ID barriers
relationship ordering
constraint ordering
batch grouping
```

---

# 32. Persistence planning

El planner transforma:

```text
ChangeSets
+
Entity Metadata
+
Relationship Metadata
+
Platform Capabilities
+
Persistence Context
```

en:

```text
Persistence Plan
```

---

# 33. PersistencePlanCreated

Evento:

```php
final readonly class PersistencePlanCreated implements PersistenceEvent
{
    public function __construct(
        public PersistenceEventContext $context,
        public PersistencePlanSummary $plan,
    ) {}
}
```

---

# 34. Persistence Plan ≠ Execution Plan

No confundir con:

```text
Database Query Execution Plan
```

del Query Planner.

---

# 35. PersistencePlanSummary

Podrá incluir:

```text
operation count
insert count
update count
delete count
dependency stages
generated-ID barriers
batch groups
cross-shard information
transaction requirement
```

---

# 36. No entity graphs in event payload

No deberá incluir el grafo completo de entidades.

---

# 37. Dependency stages

Ejemplo:

```text
Stage 1
  INSERT company

Stage 2
  INSERT user
  INSERT department

Stage 3
  UPDATE relationship

Stage 4
  DELETE obsolete association
```

---

# 38. PersistenceOperationScheduled

Cada operación podrá producir:

```text
PersistenceOperationScheduled
```

si se habilita observación detallada.

---

# 39. Event volume

Para grandes UnitOfWork:

```text
10,000 entities
```

podrían existir miles de operaciones.

Por tanto, eventos por operación deberán ser configurables.

---

# 40. Event detail levels

```php
enum PersistenceEventDetailLevel
{
    case SUMMARY;
    case OPERATION;
    case DEBUG;
}
```

---

# 41. Default detail

Producción:

```text
SUMMARY
```

o una combinación limitada.

Testing/debug:

```text
OPERATION
DEBUG
```

---

# 42. PersistenceExecutionStarted

Se emitirá antes de ejecutar el plan.

---

# 43. ExecutionStarted ≠ transaction started

La transaction puede:

```text
already exist
```

o abrirse según la política de flush.

---

# 44. Insert persistence events

Familia:

```text
PersistenceInsertStarted
PersistenceInsertExecuted
PersistenceInsertFailed
PersistenceInsertOutcomeUnknown
```

---

# 45. PersistenceInsertStarted

Significa:

> El Persistence Engine está comenzando el procesamiento de una operación INSERT concreta.

---

# 46. InsertStarted ≠ QueryExecuting

Todavía pueden existir:

```text
Query Model generation
parameter preparation
generated-ID strategy
```

antes del Query Event.

---

# 47. Insert execution pipeline

```text
PersistenceInsertStarted
↓
Insert Persistence System
↓
Query Model
↓
Query Compiler
↓
QueryExecutor
↓
QueryExecuting
↓
DB statement
↓
QueryExecuted
↓
PersistenceInsertExecuted
```

---

# 48. PersistenceInsertExecuted

Solo se emite si la operación de persistencia puede considerarse ejecutada con éxito.

---

# 49. InsertExecuted ≠ committed

Regla absoluta:

```text
PersistenceInsertExecuted
≠
EntityCreatedCommitted
```

---

# 50. Generated IDs

Después del INSERT:

```text
generated identifier
```

puede quedar disponible.

Podrá correlacionarse con:

```text
EntityIdentifierAssigned
```

del documento 213.

---

# 51. Generated ID ≠ durable ID

Si posteriormente hay rollback:

```text
ID assigned in PHP
```

no prueba que la fila exista committed.

---

# 52. Insert failure

```text
PersistenceInsertFailed
```

deberá contener:

```text
failure category
operation ID
query correlation
retryability classification
transaction impact
```

sin exponer SQL sensible.

---

# 53. Insert UNKNOWN

Si no puede determinarse si la operación ocurrió:

```text
PersistenceInsertOutcomeUnknown
```

---

# 54. UNKNOWN ≠ failed

```text
UNKNOWN
≠
FAILED
```

---

# 55. Update events

Familia:

```text
PersistenceUpdateStarted
PersistenceUpdateExecuted
PersistenceUpdateFailed
PersistenceUpdateOutcomeUnknown
```

---

# 56. Update context

Podrá incluir:

```text
EntityTypeId
entity identifier reference
changed-field count
optimistic lock presence
operation ID
```

---

# 57. Sensitive field values

Nunca deberán exponerse por default.

---

# 58. UpdateExecuted

Significa:

```text
UPDATE persistence operation completed successfully
```

no:

```text
transaction committed
```

---

# 59. Affected row count

Podrá registrarse:

```text
affectedRows
```

cuando sea semánticamente confiable.

---

# 60. affectedRows semantics

No deberán universalizarse entre plataformas sin normalización.

---

# 61. Optimistic locking

Para una operación:

```sql
UPDATE ...
WHERE id = ?
AND version = ?
```

si:

```text
affectedRows = 0
```

puede significar:

```text
OptimisticLockConflict
```

---

# 62. Conflict event

Podrá correlacionarse con:

```text
EntityOptimisticLockConflict
```

y:

```text
PersistenceOptimisticLockConflict
```

según capa.

---

# 63. Delete events

Familia:

```text
PersistenceDeleteStarted
PersistenceDeleteExecuted
PersistenceDeleteFailed
PersistenceDeleteOutcomeUnknown
```

---

# 64. DeleteExecuted ≠ committed deletion

La fila puede reaparecer después de rollback.

---

# 65. Relationship persistence

Podrán existir operaciones:

```text
association insert
association delete
foreign-key update
```

---

# 66. Relationship persistence events

Podrán modelarse mediante:

```text
PersistenceRelationshipOperationStarted
PersistenceRelationshipOperationExecuted
PersistenceRelationshipOperationFailed
```

si se requiere visibilidad específica.

---

# 67. Rich association entities

Si una tabla de asociación representa una entidad real:

```text
Membership
```

deberá seguir lifecycle/persistence normal de entidad.

---

# 68. Persistence operation source

```php
enum PersistenceOperationSource
{
    case ENTITY_INSERT;
    case ENTITY_UPDATE;
    case ENTITY_DELETE;
    case RELATIONSHIP;
    case CASCADE;
    case ORPHAN_REMOVAL;
    case INTERNAL;
}
```

---

# 69. Cascades

Una cascada:

```text
persist Company
↓
persist Departments
↓
persist Employees
```

podrá generar múltiples Persistence Operations.

---

# 70. Cascade ≠ transaction

La atomicidad depende de Transaction System.

---

# 71. Persistence batches

Operaciones compatibles podrán agruparse:

```text
Persistence Batch
```

---

# 72. Batch event family

```text
PersistenceBatchStarted
PersistenceBatchExecuted
PersistenceBatchFailed
PersistenceBatchOutcomeUnknown
```

---

# 73. Batch ≠ Bulk Operation

Importante:

```text
ORM Persistence Batch
≠
Set-Based Bulk Insert/Update/Delete
```

---

# 74. ORM batch

Puede seguir teniendo:

```text
entities
UnitOfWork
IdentityMap
change tracking
lifecycle
```

---

# 75. Bulk operation

Puede operar directamente sobre conjuntos sin cargar entidades.

---

# 76. Batch event correlation

```text
FlushId
↓
BatchId
↓
PersistenceOperationIds
↓
QueryIds
```

---

# 77. BatchId

```php
final readonly class PersistenceBatchId
{
    // ...
}
```

---

# 78. Multi-row INSERT

Un batch de inserts puede compilarse a:

```sql
INSERT INTO ...
VALUES (...), (...), (...)
```

si la plataforma lo permite.

---

# 79. Physical statement ≠ logical operation count

Ejemplo:

```text
100 logical inserts
↓
1 SQL statement
```

---

# 80. Event model must preserve both

```text
logicalOperationCount = 100
physicalStatementCount = 1
```

---

# 81. Query correlation

Un Persistence Event podrá referenciar:

```text
QueryId
```

pero no deberá duplicar todos los datos del Query Event.

---

# 82. Persistence → Query relation

```text
PersistenceOperationId
1
│
├── QueryId A
├── QueryId B
└── QueryId C
```

Una operación puede requerir más de una query.

---

# 83. Query → Persistence relation

También una query batch puede representar varias operaciones.

Por tanto:

```text
PersistenceOperationId
↔
QueryId
```

puede ser N:M.

---

# 84. Correlation model

```php
final readonly class PersistenceQueryCorrelation
{
    public function __construct(
        public PersistenceFlushId $flush,
        public array $operations,
        public array $queries,
    ) {}
}
```

---

# 85. Transaction integration

El Persistence Engine deberá conocer el contexto transaccional suficiente para correlación.

No deberá convertirse en Transaction Manager.

---

# 86. Transaction ownership

El flush puede operar:

### Existing transaction

```text
caller owns transaction
```

### Internal transaction

```text
persistence coordinator owns transaction
```

---

# 87. Ownership matters

El Persistence System nunca deberá hacer:

```text
commit caller transaction
```

solo porque terminó flush.

---

# 88. Flush completed inside existing transaction

Puede ocurrir:

```text
PersistenceFlushCompleted
↓
application does more work
↓
TransactionRolledBack
```

---

# 89. Therefore

```text
PersistenceFlushCompleted
≠
Database changes committed
```

---

# 90. PersistenceFlushCompleted

Significará:

> El Persistence Engine completó satisfactoriamente el trabajo de sincronización correspondiente al flush bajo el conocimiento disponible en ese boundary.

---

# 91. Flush completed and internal transaction

Incluso si el Persistence Engine posee la transacción, deberán distinguirse:

```text
execution complete
commit attempted
commit confirmed
```

---

# 92. Stronger event

Si se necesita:

```text
PersistenceFlushCommitted
```

solo podrá emitirse después de:

```text
TransactionOutcome = COMMITTED
```

---

# 93. Avoid duplicate transaction semantics

Preferentemente:

```text
TransactionCommitted
```

seguirá siendo la autoridad.

`PersistenceFlushCommitted` será una proyección/correlación opcional.

---

# 94. PersistenceFlushFailed

Representará failure conocido.

---

# 95. Failed ≠ rolled back

Una falla puede ocurrir:

```text
before transaction
inside transaction
during rollback
```

Por tanto deberá incluir contexto.

---

# 96. PersistenceOutcomeUnknown

Evento crítico:

```text
PersistenceOutcomeUnknown
```

se utilizará cuando no pueda demostrarse:

```text
what database state actually became visible/durable
```

---

# 97. UNKNOWN examples

```text
statement sent
↓
connection lost
```

o:

```text
COMMIT sent
↓
connection lost before confirmation
```

---

# 98. Operation UNKNOWN vs Transaction UNKNOWN

Deben distinguirse.

```text
PersistenceOperationOutcomeUnknown
```

puede ocurrir a nivel statement.

```text
TransactionOutcomeUnknown
```

ocurre a nivel transacción.

---

# 99. Propagation

Un transaction UNKNOWN normalmente implica que:

```text
final persistence durability
```

también es UNKNOWN.

---

# 100. No invented success

VoltStack nunca convertirá:

```text
UNKNOWN
```

en:

```text
SUCCESS
```

por conveniencia.

---

# 101. No invented failure

Tampoco:

```text
UNKNOWN
```

en:

```text
FAILED
```

sin evidencia.

---

# 102. Persistence outcome model

```php
enum PersistenceOutcome
{
    case SUCCEEDED;
    case FAILED;
    case CANCELLED;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 103. PARTIAL

Puede ser relevante cuando:

```text
operation 1 succeeded
operation 2 succeeded
operation 3 failed
```

sin atomicidad que pueda demostrarse.

---

# 104. Transactional rollback

Si todo fue revertido con rollback confirmado:

```text
database durable outcome
=
ROLLED_BACK
```

aunque algunas operaciones hayan tenido:

```text
PersistenceInsertExecuted
```

antes.

---

# 105. Operation outcome ≠ final database reality

Regla crítica.

---

# 106. Persistence Consistency integration

Después de ejecución:

```text
Persistence Engine
↓
Persistence Consistency System
```

determina qué puede afirmarse del ORM.

---

# 107. Consistency statuses

Recordando 134:

```text
CONSISTENT
STALE
UNCERTAIN
INCONSISTENT
TAINTED
UNKNOWN
```

---

# 108. Reconciliation events

Podrán existir:

```text
PersistenceReconciliationStarted
PersistenceReconciliationCompleted
PersistenceReconciliationFailed
PersistenceConsistencyUnknown
```

---

# 109. Reconciliation ≠ rollback

Reconciliation analiza:

```text
ORM knowledge
vs
known DB outcome
```

---

# 110. EntityManager taint

Un resultado incierto podrá causar:

```text
PersistenceOutcomeUnknown
↓
EntityManagerTainted
```

---

# 111. Event ordering

El ordering deberá ser explícito.

Ejemplo insert:

```text
EntityPersistScheduled
↓
PersistenceFlushStarted
↓
PersistenceChangesCollected
↓
EntityPrePersist
↓
PersistencePlanCreated
↓
PersistenceExecutionStarted
↓
PersistenceInsertStarted
↓
QueryExecuting
↓
QueryExecuted
↓
PersistenceInsertExecuted
↓
EntityPostPersist
↓
PersistenceExecutionCompleted
↓
PersistenceReconciliationCompleted
↓
PersistenceFlushCompleted
```

---

# 112. Transaction events around flow

Si el flush abre una transacción:

```text
PersistenceFlushStarted
↓
TransactionStarted
↓
...
↓
PersistenceFlushCompleted
↓
TransactionCommitStarted
↓
TransactionCommitted
```

El ordering concreto dependerá del ownership model.

---

# 113. Alternative internal transaction boundary

También puede definirse:

```text
PersistenceFlushStarted
↓
TransactionStarted
↓
PersistenceExecution
↓
TransactionCommitStarted
↓
TransactionCommitted
↓
PersistenceFlushCompleted
```

si `flush()` posee completamente la transacción.

---

# 114. Requirement

VoltStack deberá documentar el boundary exacto según:

```text
transaction ownership
```

---

# 115. Recommended semantics

Definir:

```text
PersistenceExecutionCompleted
```

como final del trabajo ORM/statement.

Y reservar:

```text
PersistenceFlushCompleted
```

para el final del contrato completo de `flush()`.

---

# 116. Existing transaction

Si la transacción es externa:

```text
flush completed
```

sin commit.

---

# 117. Owned transaction

Si el flush abre y posee la transacción:

```text
flush completed
```

puede ocurrir después de commit confirmado.

---

# 118. Therefore event metadata

Debe incluir:

```text
transactionOwnership
commitIncludedInFlush
```

---

# 119. TransactionOwnership

```php
enum TransactionOwnership
{
    case NONE;
    case CALLER;
    case PERSISTENCE_ENGINE;
}
```

---

# 120. FlushOutcome

```php
final readonly class PersistenceFlushOutcome
{
    public function __construct(
        public PersistenceOutcome $execution,
        public ?TransactionOutcome $transaction,
        public PersistenceConsistencyStatus $consistency,
        public TransactionOwnership $ownership,
    ) {}
}
```

---

# 121. Listener model

Persistence Events serán observacionales por default.

---

# 122. Listeners must not mutate persistence plan

Un listener genérico no deberá poder cambiar:

```text
operation order
entity values
query
transaction
```

---

# 123. Persistence extensions ≠ listeners

Si se desea modificar planificación, deberá utilizarse un extension point explícito.

---

# 124. Why

Evita:

```text
event listener order
=
hidden persistence semantics
```

---

# 125. Pre-persistence mutation

Las mutaciones de entidad deberán usar:

```text
Entity Lifecycle Hook
```

del documento 213.

---

# 126. Persistence policy extensions

Cambios técnicos de planificación deberán usar:

```text
Persistence Planner Extension
```

no eventos observacionales.

---

# 127. Event cancellation

Generic persistence observers:

```text
cannot cancel
```

por default.

---

# 128. Veto

Un veto funcional deberá ocurrir antes, mediante lifecycle/policy/validation.

---

# 129. Listener exceptions

Un listener puede fallar.

Debe distinguirse:

```text
Persistence failed
```

de:

```text
Persistence observer failed
```

---

# 130. Pre-execution listener failure

Si un observer síncrono configurado como critical falla antes de ejecutar, la política puede abortar.

Pero deberá reportarse como:

```text
ObserverFailure
```

no falsamente como SQL failure.

---

# 131. Post-execution listener failure

Si:

```text
INSERT succeeded
↓
listener failed
```

no deberá reescribirse como:

```text
INSERT failed
```

---

# 132. Transaction consequences

Si la transacción sigue abierta, la excepción del listener podría causar rollback por política superior.

---

# 133. Reality remains precise

Entonces:

```text
InsertExecuted = true
ListenerFailed = true
TransactionRolledBack = true
```

---

# 134. Async listeners

Eventos persistence internos normalmente serán:

```text
IN_PROCESS
```

---

# 135. Externalization

No se recomienda externalizar directamente:

```text
PersistenceInsertExecuted
```

como mensaje de negocio.

---

# 136. Why

Porque puede ocurrir rollback.

---

# 137. External side effects

Usar:

```text
Domain Event
↓
Outbox
↓
Transaction Commit
↓
Delivery
```

---

# 138. Outbox integration

Un outbox record puede ser insertado como parte del mismo persistence plan/transaction.

---

# 139. Outbox insert ≠ message delivered

Aun después del commit:

```text
outbox committed
≠
external broker delivery completed
```

---

# 140. Retry integration

Persistence puede ejecutarse dentro de:

```text
Transaction Retry System
```

---

# 141. Attempt identity

Cada intento deberá tener:

```text
TransactionAttemptId
```

o:

```text
PersistenceAttemptId
```

---

# 142. Retry correlation

```text
Logical Flush
  │
  ├── Attempt 1
  │    ├── operations
  │    └── deadlock
  │
  └── Attempt 2
       ├── operations
       └── success
```

---

# 143. Logical operation ≠ attempt

Un retry puede volver a ejecutar operaciones.

---

# 144. PersistenceAttemptId

```php
final readonly class PersistenceAttemptId
{
    // ...
}
```

---

# 145. Retry event family

Podrán existir:

```text
PersistenceAttemptStarted
PersistenceAttemptFailed
PersistenceAttemptRetryScheduled
PersistenceAttemptCompleted
```

---

# 146. Retry events ≠ Transaction Retry Events

Se correlacionan, pero cada capa conserva su semántica.

---

# 147. Side effects

Persistence listeners no deberán realizar side effects no idempotentes dentro de retryable boundaries.

---

# 148. Duplicate event observation

Un listener puede observar:

```text
PersistenceInsertExecuted
```

en un intento que posteriormente hace rollback y se reintenta.

---

# 149. Therefore

Los listeners deberán conocer:

```text
attempt ID
transaction ID
flush ID
```

cuando sea necesario.

---

# 150. Commit-aware deduplication

No debe deduplicarse un evento simplemente por:

```text
Entity ID
```

porque pueden existir múltiples operaciones legítimas.

---

# 151. Batch persistence

Para `flush()` con miles de entidades:

```text
PersistenceFlushStarted
↓
PersistencePlanCreated
↓
Batch 1
↓
Batch 2
↓
...
↓
PersistenceFlushCompleted
```

---

# 152. Batch observability

Eventos agregados serán preferibles en producción.

---

# 153. Batch summary

```php
final readonly class PersistenceBatchSummary
{
    public function __construct(
        public PersistenceBatchId $id,
        public int $logicalOperations,
        public int $physicalStatements,
        public PersistenceOperationKind $kind,
    ) {}
}
```

---

# 154. Batch failure

Debe poder indicar:

```text
first failing operation
operations known executed
operations not attempted
operations uncertain
```

sin transportar entidades completas.

---

# 155. Large Dataset Processing

Este sistema no deberá utilizarse como sustituto de:

```text
208_DATABASE_LARGE_DATASET_PROCESSING_SYSTEM
```

---

# 156. UnitOfWork memory

Los eventos no deberán mantener referencias que impidan liberar entidades después de:

```text
flush
clear
detach
```

---

# 157. Bulk Insert integration

`203_DATABASE_BULK_INSERT_SYSTEM` puede operar sin UnitOfWork.

Por tanto:

```text
BulkInsertEvent
≠
PersistenceInsertEvent
```

por default.

---

# 158. Bulk Update integration

Igual:

```text
BulkUpdateEvent
≠
PersistenceUpdateEvent
```

---

# 159. Bulk Delete integration

Igual:

```text
BulkDeleteEvent
≠
PersistenceDeleteEvent
```

---

# 160. Import integration

Import en modo ORM:

```text
Import
↓
EntityManager
↓
Persistence Events
```

Import directo:

```text
Import
↓
Bulk Engine
```

no deberá fingir eventos ORM.

---

# 161. Cache invalidation

Persistence puede producir invalidation intents.

Pero:

```text
Persistence Event
≠
Cache Invalidation Event
```

---

# 162. Correct timing

Invalidaciones que dependen de commit deberán registrarse:

```text
during persistence
↓
defer
↓
after commit
```

---

# 163. Rollback

Las invalidaciones deferred deberán cancelarse.

---

# 164. UNKNOWN

Ante transaction UNKNOWN:

```text
conservative invalidation
```

puede ser apropiada según Cache Consistency policy.

---

# 165. Event System does not decide cache truth

La decisión corresponde a:

```text
Cache Consistency System
Cache Invalidation System
```

---

# 166. Security

Persistence events pueden revelar información extremadamente sensible.

---

# 167. Prohibited default payload

No incluir:

```text
password
token
secret
credit-card data
raw entity
raw SQL
all bindings
complete ChangeSet
```

---

# 168. Sanitization

Antes de logging/externalization:

```text
Persistence Event
↓
PersistenceEventSanitizer
↓
Telemetry/Event Sink
```

---

# 169. Entity identifier exposure

También deberá estar sujeto a policy.

---

# 170. Tenant IDs

No deberán convertirse automáticamente en telemetry labels.

---

# 171. Query bindings

Los Persistence Events no deberán duplicar query bindings.

La Query Event layer aplicará su propia política.

---

# 172. Security principle

```text
Correlation
>
Payload duplication
```

---

# 173. Telemetry integration

Persistence Events podrán alimentar:

```text
ORM Telemetry
Persistence metrics
Tracing
Debugging
Profiling
```

---

# 174. Suggested metrics

```text
db.persistence.flush.count
db.persistence.flush.duration
db.persistence.operation.count
db.persistence.insert.count
db.persistence.update.count
db.persistence.delete.count
db.persistence.failure.count
db.persistence.unknown.count
db.persistence.batch.size
```

---

# 175. Cardinality policy

Labels permitidos:

```text
operation kind
outcome
driver/platform family
```

con cuidado.

---

# 176. Labels to avoid

```text
entity ID
user ID
tenant ID
query ID
transaction ID
```

---

# 177. EntityType metric label

Solo si:

```text
bounded known entity set
```

y telemetry policy lo permite.

---

# 178. Tracing

Span conceptual:

```text
database.persistence.flush
```

con child spans opcionales:

```text
database.persistence.plan
database.persistence.execute
database.persistence.reconcile
```

---

# 179. Query spans

No deberán duplicarse.

Persistence spans podrán enlazar:

```text
Query spans
```

mediante trace context.

---

# 180. Event timestamps

Usar:

```text
monotonic clock
```

para duración.

Wall clock para timestamps externos cuando sea necesario.

---

# 181. Persistent runtime

Crítico para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 182. Scope-local state

Será scope-local:

```text
PersistenceFlushId
PersistenceAttemptId
PersistenceBatchId
operation registry
event buffers
transaction correlation
EntityManager reference
tenant context
shard context
listener invocation state
```

---

# 183. Shared immutable state

Puede compartirse:

```text
compiled metadata
event descriptors
listener definitions
immutable policies
```

---

# 184. Forbidden static state

No:

```php
PersistenceEvents::$currentFlush;
PersistenceEvents::$currentEntity;
PersistenceEvents::$currentTransaction;
```

---

# 185. Coroutine safety

OpenSwoole deberá mantener context isolation por coroutine/operation scope.

---

# 186. Worker reuse

RoadRunner/FrankenPHP deberán limpiar:

```text
event buffers
operation correlations
EntityManager references
transaction references
```

al terminar request/job.

---

# 187. Failure during cleanup

No deberá provocar que el siguiente request herede estado anterior.

---

# 188. Reentrancy

Un persistence listener podría intentar:

```php
$entityManager->flush();
```

durante otro flush.

---

# 189. Default policy

Nested/reentrant flush deberá estar:

```text
FORBIDDEN
```

por default.

---

# 190. Why

Puede corromper:

```text
UnitOfWork
Persistence Plan
operation ordering
event ordering
```

---

# 191. ReentrantFlushException

```text
PersistenceReentrantFlushException
```

---

# 192. Listener-triggered persist()

Un listener observacional tampoco debería modificar el UoW.

---

# 193. Lifecycle hooks

Si se permite mutación, debe ocurrir en boundaries explícitos antes de congelar el plan.

---

# 194. Persistence plan immutability

Después de:

```text
PersistencePlanFrozen
```

los observers no podrán modificarlo.

---

# 195. Optional event

```text
PersistencePlanFrozen
```

podrá ser interno/debug.

---

# 196. Planning retries

Si lifecycle hooks cambian entidades antes del freeze:

```text
ChangeSets
↓
Hooks
↓
Recompute
↓
Plan
```

según documento 213.

---

# 197. Plan identity

```php
final readonly class PersistencePlanId
{
    // ...
}
```

---

# 198. Plan version

Si se recompone:

```text
PlanGeneration
```

podrá incrementarse.

---

# 199. Event sequence integrity

Cada Persistence Event deberá llevar:

```text
sequence number
```

opcional dentro del flush para diagnostics.

---

# 200. Sequence ≠ global ordering

No deberá existir un contador global compartido requerido para correctness.

---

# 201. PersistenceEventSequence

```php
final readonly class PersistenceEventSequence
{
    public function __construct(
        public int $value,
    ) {}
}
```

---

# 202. Cancellation

Si el request es cancelado:

```text
PersistenceCancelled
```

podrá emitirse.

---

# 203. Cancellation ≠ rollback

Si ya hubo statements:

```text
cancelled
```

no demuestra que hayan sido revertidos.

---

# 204. Deadline exceeded

Similarmente:

```text
PersistenceDeadlineExceeded
```

describe causa operacional.

No determina realidad final.

---

# 205. Resource exhaustion

Puede ocurrir:

```text
memory limit
connection unavailable
statement budget exhausted
```

antes o durante persistencia.

---

# 206. Failure classification

```php
enum PersistenceFailureCategory
{
    case VALIDATION;
    case PLANNING;
    case CONNECTION;
    case QUERY_EXECUTION;
    case CONSTRAINT;
    case OPTIMISTIC_LOCK;
    case DEADLOCK;
    case TIMEOUT;
    case CANCELLATION;
    case RESOURCE_EXHAUSTION;
    case LISTENER;
    case INTERNAL;
    case UNKNOWN;
}
```

---

# 207. Retryability

Failure event podrá incluir:

```text
RETRYABLE
NOT_RETRYABLE
UNKNOWN
```

pero la decisión final corresponde al Retry System.

---

# 208. Failure ≠ retry permission

Regla:

```text
Retryable-looking error
≠
Safe to replay transaction
```

---

# 209. Diagnostics

VoltStack deberá poder explicar un flush.

Ejemplo:

```text
PERSISTENCE EXPLAIN

Flush:
  FL-91

Entity Manager:
  EM-17

Changes:
  Inserts: 120
  Updates: 48
  Deletes: 3

Plan:
  Operations: 171
  Stages: 4
  Batches: 6

Transaction:
  Ownership: CALLER
  Existing Transaction: TX-71

Events:
  Detail: SUMMARY

Retry:
  Attempt: 1
  Replay Safe: UNKNOWN

Cache:
  Deferred invalidations: 14

Distribution:
  Database: primary
  Shard: shard-04

Outcome:
  Execution: SUCCEEDED
  Transaction: ACTIVE
  Durability: NOT_YET_KNOWN
```

---

# 210. Important diagnostic wording

Nunca mostrar:

```text
Committed: yes
```

solo porque:

```text
flush succeeded
```

---

# 211. Testing API

Ejemplo:

```php
DatabaseEvents::fake();

$entityManager->persist($user);
$entityManager->flush();

DatabaseEvents::assertDispatched(
    PersistenceFlushStarted::class
);

DatabaseEvents::assertDispatched(
    PersistenceInsertExecuted::class
);
```

---

# 212. Test must not assume commit

Si la transacción sigue abierta:

```php
DatabaseEvents::assertNotDispatched(
    EntityCreatedCommitted::class
);
```

---

# 213. Rollback test

```text
PersistenceInsertExecuted
✓

TransactionRolledBack
✓

EntityCreatedCommitted
✗
```

---

# 214. UNKNOWN test

```text
PersistenceInsertOutcomeUnknown
✓

PersistenceInsertExecuted
✗ unless independently proven

EntityCreatedCommitted
✗
```

---

# 215. Retry test

```text
FlushId = same logical flush

Attempt 1
  deadlock

Attempt 2
  success
```

Los eventos deberán poder distinguir ambos attempts.

---

# 216. Batch test

Validar:

```text
logical operations = 100
physical statements = 1
```

sin perder semántica.

---

# 217. Listener failure test

```text
PersistenceInsertExecuted
↓
observer throws
↓
transaction rollback
```

deberá conservar los tres hechos.

---

# 218. Persistent runtime test

Dos requests consecutivos no deberán compartir:

```text
FlushId
AttemptId
BatchId
operation registry
event buffer
EntityManager
```

---

# 219. Event recording

Testing podrá activar:

```text
DEBUG
```

sin cambiar comportamiento de persistencia.

---

# 220. Event recording ≠ event replay

Reproducir eventos persistence no deberá utilizarse para reconstruir DB state.

---

# 221. Persistence Events ≠ Event Sourcing

Regla importante:

```text
Persistence Event System
≠
Event Sourcing System
```

---

# 222. Why

Los eventos persistence describen infraestructura ORM.

No son necesariamente:

```text
complete
durable
business-semantic
replayable
```

---

# 223. Audit integration

Persistence Events pueden alimentar auditoría técnica.

Pero:

```text
Persistence Event
≠
Audit Record
```

---

# 224. Audit durability

Un sistema de auditoría puede requerir almacenamiento transaccional/durable independiente.

---

# 225. Error hierarchy

```text
PersistenceEventException
├── PersistenceEventDispatchException
├── PersistenceEventListenerException
├── PersistenceEventCorrelationException
├── PersistenceEventSecurityException
├── PersistenceEventStateException
├── PersistenceEventOrderingException
├── PersistenceEventVolumeException
├── PersistenceEventSerializationException
└── PersistenceEventCompatibilityException
```

El Persistence Engine mantendrá además sus propias excepciones operacionales.

---

# 226. Persistence failures ≠ Event failures

Ejemplo:

```text
PersistenceExecutionException
```

pertenece al persistence subsystem.

Mientras:

```text
PersistenceEventDispatchException
```

pertenece al Event System.

---

# 227. Event system failure isolation

El sistema deberá evitar que un fallo de observabilidad destruya innecesariamente el persistence pipeline.

---

# 228. Critical listeners

Si existe un listener realmente crítico, deberá declararse explícitamente.

---

# 229. ListenerCriticality

```php
enum ListenerCriticality
{
    case OBSERVATIONAL;
    case CRITICAL;
}
```

---

# 230. Default

```text
OBSERVATIONAL
```

---

# 231. Critical listener caution

No deberá usarse para implementar:

```text
business validation
authorization
domain invariants
```

si existen capas apropiadas para ello.

---

# 232. Event descriptors

Cada tipo de evento podrá declarar:

```text
phase
volume class
externalization eligibility
sensitivity
transaction awareness
default criticality
```

---

# 233. Example descriptor

```php
final readonly class PersistenceEventDescriptor
{
    public function __construct(
        public PersistenceEventType $type,
        public PersistenceEventPhase $phase,
        public EventVolumeClass $volume,
        public EventSensitivity $sensitivity,
        public ExternalizationPolicy $externalization,
    ) {}
}
```

---

# 234. Event registry

Los tipos core deberán registrarse de forma compilable/determinista.

---

# 235. Extension events

Paquetes podrán agregar eventos mediante el sistema definido en:

```text
215_DATABASE_EVENT_EXTENSION_SYSTEM.md
```

---

# 236. Core event namespace

```text
VoltStack\Quantum\Database\Event\Persistence
```

---

# 237. Directory structure

```text
src/Quantum/Database/Event/Persistence/
│
├── Contract/
│   ├── PersistenceEvent.php
│   └── PersistenceEventListener.php
│
├── Context/
│   ├── PersistenceEventContext.php
│   ├── PersistenceEventMetadata.php
│   └── PersistenceEventSequence.php
│
├── Id/
│   ├── PersistenceFlushId.php
│   ├── PersistencePlanId.php
│   ├── PersistenceOperationId.php
│   ├── PersistenceBatchId.php
│   └── PersistenceAttemptId.php
│
├── Event/
│   ├── PersistenceFlushStarted.php
│   ├── PersistenceChangesCollected.php
│   ├── PersistencePlanCreated.php
│   ├── PersistencePlanFrozen.php
│   ├── PersistenceExecutionStarted.php
│   ├── PersistenceOperationScheduled.php
│   │
│   ├── PersistenceInsertStarted.php
│   ├── PersistenceInsertExecuted.php
│   ├── PersistenceInsertFailed.php
│   ├── PersistenceInsertOutcomeUnknown.php
│   │
│   ├── PersistenceUpdateStarted.php
│   ├── PersistenceUpdateExecuted.php
│   ├── PersistenceUpdateFailed.php
│   ├── PersistenceUpdateOutcomeUnknown.php
│   │
│   ├── PersistenceDeleteStarted.php
│   ├── PersistenceDeleteExecuted.php
│   ├── PersistenceDeleteFailed.php
│   ├── PersistenceDeleteOutcomeUnknown.php
│   │
│   ├── PersistenceBatchStarted.php
│   ├── PersistenceBatchExecuted.php
│   ├── PersistenceBatchFailed.php
│   ├── PersistenceBatchOutcomeUnknown.php
│   │
│   ├── PersistenceExecutionCompleted.php
│   ├── PersistenceReconciliationStarted.php
│   ├── PersistenceReconciliationCompleted.php
│   ├── PersistenceReconciliationFailed.php
│   ├── PersistenceConsistencyUnknown.php
│   │
│   ├── PersistenceFlushCompleted.php
│   ├── PersistenceFlushFailed.php
│   ├── PersistenceFlushCancelled.php
│   └── PersistenceOutcomeUnknown.php
│
├── Retry/
│   ├── PersistenceAttemptStarted.php
│   ├── PersistenceAttemptFailed.php
│   ├── PersistenceAttemptRetryScheduled.php
│   └── PersistenceAttemptCompleted.php
│
├── Model/
│   ├── PersistenceEventPhase.php
│   ├── PersistenceOutcome.php
│   ├── PersistenceFailureCategory.php
│   ├── PersistenceOperationKind.php
│   ├── PersistenceOperationSource.php
│   ├── PersistenceEventDetailLevel.php
│   ├── PersistenceFlushTargetKind.php
│   ├── TransactionOwnership.php
│   └── ListenerCriticality.php
│
├── Summary/
│   ├── PersistenceChangeSummary.php
│   ├── PersistencePlanSummary.php
│   ├── PersistenceBatchSummary.php
│   └── PersistenceFlushOutcome.php
│
├── Correlation/
│   └── PersistenceQueryCorrelation.php
│
├── Security/
│   ├── PersistenceEventSanitizer.php
│   ├── PersistenceEventExposurePolicy.php
│   └── PersistenceSensitiveFieldPolicy.php
│
├── Diagnostics/
│   ├── PersistenceEventInspector.php
│   ├── PersistenceEventRecorder.php
│   └── PersistenceEventExplain.php
│
├── Testing/
│   ├── FakePersistenceEventDispatcher.php
│   ├── PersistenceEventAssertions.php
│   └── PersistenceEventSequenceAssertions.php
│
└── Exception/
    ├── PersistenceEventException.php
    ├── PersistenceEventDispatchException.php
    ├── PersistenceEventListenerException.php
    ├── PersistenceEventCorrelationException.php
    ├── PersistenceEventSecurityException.php
    ├── PersistenceEventStateException.php
    ├── PersistenceEventOrderingException.php
    ├── PersistenceEventVolumeException.php
    ├── PersistenceEventSerializationException.php
    └── PersistenceEventCompatibilityException.php
```

---

# 238. Event sequence — INSERT

```text
EntityManager::flush()
        │
        ▼
PersistenceFlushStarted
        │
        ▼
PersistenceChangesCollected
        │
        ▼
EntityPrePersist
        │
        ▼
PersistencePlanCreated
        │
        ▼
PersistencePlanFrozen
        │
        ▼
PersistenceExecutionStarted
        │
        ▼
PersistenceInsertStarted
        │
        ▼
QueryExecuting
        │
        ▼
INSERT
        │
        ▼
QueryExecuted
        │
        ▼
PersistenceInsertExecuted
        │
        ├──► EntityIdentifierAssigned
        │
        ▼
EntityPostPersist
        │
        ▼
PersistenceExecutionCompleted
        │
        ▼
PersistenceReconciliationCompleted
        │
        ▼
PersistenceFlushCompleted
```

Commit ocurre en el boundary correspondiente:

```text
TransactionCommitStarted
↓
TransactionCommitted
↓
AfterCommit Events
```

---

# 239. Event sequence — UPDATE

```text
Entity mutation
↓
EntityDirtyDetected
↓
flush()
↓
PersistenceFlushStarted
↓
ChangeSet
↓
EntityPreUpdate
↓
PersistencePlanCreated
↓
PersistenceUpdateStarted
↓
QueryExecuting
↓
UPDATE
↓
QueryExecuted
↓
PersistenceUpdateExecuted
↓
EntityPostUpdate
↓
PersistenceExecutionCompleted
↓
PersistenceFlushCompleted
```

---

# 240. Event sequence — DELETE

```text
remove(entity)
↓
EntityRemoveScheduled
↓
flush()
↓
PersistenceFlushStarted
↓
EntityPreRemove
↓
PersistencePlanCreated
↓
PersistenceDeleteStarted
↓
QueryExecuting
↓
DELETE
↓
QueryExecuted
↓
PersistenceDeleteExecuted
↓
EntityPostRemove
↓
PersistenceExecutionCompleted
↓
PersistenceFlushCompleted
```

---

# 241. Failure sequence

```text
PersistenceUpdateStarted
↓
QueryExecuting
↓
Database Error
↓
QueryFailed
↓
PersistenceUpdateFailed
↓
PersistenceExecutionFailed
↓
Transaction Rollback
↓
PersistenceReconciliation
↓
PersistenceFlushFailed
```

---

# 242. UNKNOWN sequence

```text
PersistenceUpdateStarted
↓
QueryExecuting
↓
Statement sent
↓
Connection lost
↓
QueryOutcomeUnknown
↓
PersistenceUpdateOutcomeUnknown
↓
TransactionOutcomeUnknown
↓
PersistenceConsistencyUnknown
↓
EntityManagerTainted
```

---

# 243. Architectural invariants

## DB-PEVENT-001
Persistence Event será distinto de Entity Lifecycle Event.

## DB-PEVENT-002
Persistence Event será distinto de Query Event.

## DB-PEVENT-003
Persistence Event será distinto de Transaction Event.

## DB-PEVENT-004
Persistence Event será distinto de Connection Event.

## DB-PEVENT-005
Persistence Event será distinto de Domain Event.

## DB-PEVENT-006
Persistence Event será distinto de Cache Event.

## DB-PEVENT-007
Persistence Event no será prueba automática de durability.

## DB-PEVENT-008
Statement success será distinto de transaction commit.

## DB-PEVENT-009
Flush success será distinto de transaction commit cuando la transacción sea externa.

## DB-PEVENT-010
ChangeSet será distinto de PersistenceOperation.

## DB-PEVENT-011
PersistenceOperation será distinto de Query Model.

## DB-PEVENT-012
PersistenceOperation será distinto de SQL.

## DB-PEVENT-013
Persistence Engine nunca generará SQL directamente.

## DB-PEVENT-014
PersistenceOperationId será distinto de QueryId.

## DB-PEVENT-015
PersistenceFlushId será distinto de TransactionId.

## DB-PEVENT-016
PersistenceAttemptId será distinto de FlushId.

## DB-PEVENT-017
PersistenceBatchId será distinto de OperationId.

## DB-PEVENT-018
Persistence event context será immutable.

## DB-PEVENT-019
Persistence event context será bounded.

## DB-PEVENT-020
Persistence event context será sanitized.

## DB-PEVENT-021
FlushStarted no implicará transaction started.

## DB-PEVENT-022
ChangesCollected no implicará persistence plan final.

## DB-PEVENT-023
PersistencePlan será distinto de Query Execution Plan.

## DB-PEVENT-024
PersistencePlanCreated no implicará execution.

## DB-PEVENT-025
OperationScheduled no implicará query execution.

## DB-PEVENT-026
InsertStarted no implicará QueryExecuting.

## DB-PEVENT-027
InsertExecuted no implicará commit.

## DB-PEVENT-028
UpdateExecuted no implicará commit.

## DB-PEVENT-029
DeleteExecuted no implicará commit.

## DB-PEVENT-030
Generated ID no implicará commit.

## DB-PEVENT-031
Generated version no implicará commit.

## DB-PEVENT-032
FAILED será distinto de UNKNOWN.

## DB-PEVENT-033
UNKNOWN nunca se convertirá silenciosamente en FAILED.

## DB-PEVENT-034
UNKNOWN nunca se convertirá silenciosamente en SUCCEEDED.

## DB-PEVENT-035
Operation outcome será distinto de final database reality.

## DB-PEVENT-036
Rollback confirmado podrá revertir statements previamente executed.

## DB-PEVENT-037
Statement rollback no implicará object graph rewind.

## DB-PEVENT-038
Reconciliation será distinta de rollback.

## DB-PEVENT-039
Persistence Consistency será authority de ORM/DB knowledge reconciliation.

## DB-PEVENT-040
Transaction Manager será authority de commit outcome.

## DB-PEVENT-041
Query Event System será authority de query execution observation.

## DB-PEVENT-042
Entity Lifecycle Event System será authority de entity lifecycle observation.

## DB-PEVENT-043
Persistence Event System observará synchronization pipeline.

## DB-PEVENT-044
Generic persistence listeners serán observacionales.

## DB-PEVENT-045
Generic listeners no modificarán persistence plan.

## DB-PEVENT-046
Generic listeners no modificarán UnitOfWork.

## DB-PEVENT-047
Entity mutation utilizará lifecycle hook explícito.

## DB-PEVENT-048
Persistence planning customization utilizará extension contract.

## DB-PEVENT-049
Listener order no definirá persistence semantics.

## DB-PEVENT-050
Observer failure será distinto de persistence failure.

## DB-PEVENT-051
Post-execution listener failure no reescribirá statement success.

## DB-PEVENT-052
Listener failure podrá provocar rollback solo mediante policy superior.

## DB-PEVENT-053
Persistence events serán in-process por default.

## DB-PEVENT-054
Persistence events no serán integration events por default.

## DB-PEVENT-055
Persistence events no serán domain events.

## DB-PEVENT-056
Persistence Event System no será Event Sourcing.

## DB-PEVENT-057
Persistence Event System no será Audit System.

## DB-PEVENT-058
Outbox será preferible para side effects externos.

## DB-PEVENT-059
Outbox insert será distinto de message delivery.

## DB-PEVENT-060
Retry attempt será distinto de logical flush.

## DB-PEVENT-061
Retries podrán repetir persistence events.

## DB-PEVENT-062
Attempt identity será preservada.

## DB-PEVENT-063
Side-effecting listeners dentro de retryable boundaries serán desaconsejados.

## DB-PEVENT-064
ORM Batch será distinto de Bulk Operation.

## DB-PEVENT-065
Logical operation count será distinto de physical statement count.

## DB-PEVENT-066
Una operation podrá producir múltiples queries.

## DB-PEVENT-067
Una query podrá representar múltiples logical operations.

## DB-PEVENT-068
Persistence↔Query correlation podrá ser N:M.

## DB-PEVENT-069
Persistence Engine no hará commit de caller-owned transaction.

## DB-PEVENT-070
Transaction ownership será explícito.

## DB-PEVENT-071
FlushCompleted dentro de caller transaction no significará committed.

## DB-PEVENT-072
PersistenceFlushCommitted requerirá COMMITTED probado.

## DB-PEVENT-073
PersistenceFlushCommitted no reemplazará TransactionCommitted.

## DB-PEVENT-074
FlushFailed será distinto de RolledBack.

## DB-PEVENT-075
Cancellation será distinta de rollback.

## DB-PEVENT-076
Timeout será distinto de rollback.

## DB-PEVENT-077
Retryability será distinta de replay safety.

## DB-PEVENT-078
Deadlock no autorizará statement-only replay automáticamente.

## DB-PEVENT-079
Persistence cache invalidation dependerá de transaction outcome cuando corresponda.

## DB-PEVENT-080
Uncommitted persistence no publicará committed cache state.

## DB-PEVENT-081
Rollback cancelará deferred cache publication.

## DB-PEVENT-082
UNKNOWN podrá provocar conservative invalidation.

## DB-PEVENT-083
Persistence events no expondrán secrets.

## DB-PEVENT-084
Persistence events no expondrán raw entities por default.

## DB-PEVENT-085
Persistence events no expondrán full ChangeSets por default.

## DB-PEVENT-086
Persistence events no duplicarán raw query bindings.

## DB-PEVENT-087
Correlation será preferible a payload duplication.

## DB-PEVENT-088
Entity IDs no serán telemetry labels.

## DB-PEVENT-089
Tenant IDs no serán telemetry labels.

## DB-PEVENT-090
Query IDs no serán telemetry labels.

## DB-PEVENT-091
Transaction IDs no serán telemetry labels.

## DB-PEVENT-092
Telemetry de persistence será bounded.

## DB-PEVENT-093
Per-operation events serán configurables.

## DB-PEVENT-094
Summary mode será apropiado para high-volume workloads.

## DB-PEVENT-095
Debug event detail no cambiará persistence semantics.

## DB-PEVENT-096
Persistent runtime mutable state será scope-local.

## DB-PEVENT-097
No existirá static current flush.

## DB-PEVENT-098
No existirá static current persistence operation.

## DB-PEVENT-099
No existirá static current entity.

## DB-PEVENT-100
No existirá static current transaction en Persistence Event System.

## DB-PEVENT-101
Event buffers serán limpiados al terminar scope.

## DB-PEVENT-102
EntityManager references serán liberadas al terminar scope.

## DB-PEVENT-103
FrankenPHP no heredará persistence event state entre requests.

## DB-PEVENT-104
RoadRunner no heredará persistence event state entre jobs.

## DB-PEVENT-105
OpenSwoole mantendrá coroutine isolation.

## DB-PEVENT-106
Reentrant flush estará prohibido por default.

## DB-PEVENT-107
Persistence listeners no iniciarán nested flush por default.

## DB-PEVENT-108
Persistence plan congelado será immutable.

## DB-PEVENT-109
Plan mutation posterior al freeze será error.

## DB-PEVENT-110
Lifecycle mutation deberá ocurrir antes del plan freeze.

## DB-PEVENT-111
Event sequence podrá ser diagnosticada.

## DB-PEVENT-112
Sequence number será local al flush, no global correctness mechanism.

## DB-PEVENT-113
Bulk Insert no fingirá ORM persistence events.

## DB-PEVENT-114
Bulk Update no fingirá ORM persistence events.

## DB-PEVENT-115
Bulk Delete no fingirá ORM persistence events.

## DB-PEVENT-116
Import directo no fingirá ORM persistence events.

## DB-PEVENT-117
Import ORM sí podrá producir persistence events.

## DB-PEVENT-118
Factory make() no producirá persistence events.

## DB-PEVENT-119
Seeder solo producirá persistence events si usa ORM persistence.

## DB-PEVENT-120
Fixture solo producirá persistence events si usa ORM persistence.

## DB-PEVENT-121
Lazy hydration por sí sola no producirá persistence events.

## DB-PEVENT-122
Lazy processing que modifique y flush sí podrá producirlos.

## DB-PEVENT-123
Chunk processing no será persistence event subsystem.

## DB-PEVENT-124
Large Dataset System no será persistence event subsystem.

## DB-PEVENT-125
Persistence event recording no será persistence replay.

## DB-PEVENT-126
Persistence events no reconstruirán database state.

## DB-PEVENT-127
Persistence events no reemplazarán transaction log.

## DB-PEVENT-128
Persistence events no reemplazarán WAL/binlog.

## DB-PEVENT-129
Persistence events no reemplazarán Change Tracking.

## DB-PEVENT-130
Persistence events no reemplazarán UnitOfWork.

## DB-PEVENT-131
Persistence events no reemplazarán Persistence Planner.

## DB-PEVENT-132
Persistence events no reemplazarán Query Engine.

## DB-PEVENT-133
Persistence events no reemplazarán Execution Engine.

## DB-PEVENT-134
Persistence events no reemplazarán Transaction Manager.

## DB-PEVENT-135
Persistence events no reemplazarán Cache Consistency System.

## DB-PEVENT-136
Persistence events no reemplazarán Telemetry System.

## DB-PEVENT-137
Persistence events no reemplazarán Security System.

## DB-PEVENT-138
Persistence outcome deberá preservar PARTIAL cuando aplique.

## DB-PEVENT-139
Persistence outcome deberá preservar UNKNOWN cuando aplique.

## DB-PEVENT-140
Commit UNKNOWN implicará durability UNKNOWN.

## DB-PEVENT-141
Database reality será distinta de ORM knowledge.

## DB-PEVENT-142
Event delivery reality será distinta de database reality.

## DB-PEVENT-143
Listener delivery success no implicará persistence success.

## DB-PEVENT-144
Listener delivery failure no implicará persistence failure.

## DB-PEVENT-145
OperationExecuted podrá ocurrir antes de rollback.

## DB-PEVENT-146
OperationExecuted podrá ocurrir en un retry fallido.

## DB-PEVENT-147
Committed notifications requerirán after-commit boundary.

## DB-PEVENT-148
After-commit listener failure no deshará commit.

## DB-PEVENT-149
Security tendrá prioridad sobre diagnostic payload richness.

## DB-PEVENT-150
Correctness tendrá prioridad sobre event convenience.

## DB-PEVENT-151
Explicit uncertainty tendrá prioridad sobre invented certainty.

## DB-PEVENT-152
Persistence event extension deberá preservar core invariants.

## DB-PEVENT-153
Driver-specific semantics no escaparán directamente a eventos core.

## DB-PEVENT-154
Platform capability differences serán normalizadas antes de eventos core cuando sea posible.

## DB-PEVENT-155
MySQL y MariaDB permanecerán plataformas distintas cuando sus capacidades difieran.

## DB-PEVENT-156
Persistence Event System será independiente de HTTP.

## DB-PEVENT-157
Persistence Event System será independiente de CLI.

## DB-PEVENT-158
Persistence Event System será independiente de concrete runtime.

## DB-PEVENT-159
Persistence Event System será compatible con testing determinista.

## DB-PEVENT-160
Todo evento deberá representar exactamente el boundary que su nombre afirma.

---

# 244. Modelo formal

Sea:

```text
U
```

el estado del `UnitOfWork`.

El Persistence Planner produce:

```text
P = Plan(U)
```

donde:

```text
P = {o1, o2, ..., on}
```

y cada:

```text
oi
```

es una operación lógica de persistencia.

---

# 245. Ejecución

Para cada operación:

```text
Execute(oi)
→
SUCCEEDED | FAILED | UNKNOWN
```

Pero:

```text
Execute(oi) = SUCCEEDED
```

no implica:

```text
Durable(oi) = true
```

---

# 246. Durabilidad

Solo puede afirmarse:

```text
Durable(P)
```

cuando el Transaction System demuestra:

```text
TransactionOutcome = COMMITTED
```

para el boundary transaccional correspondiente.

---

# 247. Rollback

Si:

```text
Execute(oi) = SUCCEEDED
```

pero posteriormente:

```text
TransactionOutcome = ROLLED_BACK
```

entonces:

```text
Durable(oi) = false
```

---

# 248. Unknown commit

Si:

```text
COMMIT sent
```

y posteriormente:

```text
connection lost
```

sin confirmación:

```text
TransactionOutcome = UNKNOWN
```

por tanto:

```text
Durable(P) = UNKNOWN
```

---

# 249. Event interpretation

Para cualquier evento:

```text
PersistenceOperationExecuted(oi)
```

solo puede inferirse:

```text
ExecutionKnowledge(oi) = SUCCEEDED
```

No:

```text
Durable(oi)
```

---

# 250. Flush interpretation

Para:

```text
PersistenceFlushCompleted(F)
```

debe interpretarse según:

```text
TransactionOwnership(F)
```

Si:

```text
CALLER
```

entonces:

```text
FlushCompleted(F)
↛
TransactionCommitted
```

---

# 251. Arquitectura final

```text
                       EntityManager
                            │
                            ▼
                         UnitOfWork
                            │
                            ▼
                      Change Tracking
                            │
                            ▼
                         ChangeSets
                            │
                            ▼
                 PersistenceFlushStarted
                            │
                            ▼
                   Persistence Planner
                            │
                            ▼
                  PersistencePlanCreated
                            │
                            ▼
                     Persistence Plan
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
           INSERT         UPDATE        DELETE
              │             │             │
              ▼             ▼             ▼
          Query Model    Query Model    Query Model
              │             │             │
              └─────────────┼─────────────┘
                            ▼
                       Query Engine
                            │
                            ▼
                     Execution Engine
                            │
                            ▼
                  Persistence Results
                            │
                            ▼
                Consistency Reconciliation
                            │
                            ▼
                 PersistenceFlushCompleted
                            │
                            ▼
                  Transaction Boundary
                            │
              ┌─────────────┼──────────────┐
              ▼             ▼              ▼
          COMMITTED      ROLLED BACK      UNKNOWN
              │             │              │
              ▼             ▼              ▼
         AfterCommit      discard      uncertainty
          actions          pending        handling
```

---

# 252. Regla maestra final

VoltStack deberá mantener siempre:

```text
ChangeSet
≠
Persistence Operation
```

```text
Persistence Operation
≠
Query
```

```text
Query
≠
SQL String
```

```text
Persistence Plan
≠
Query Execution Plan
```

```text
Flush
≠
Transaction
```

```text
Flush Success
≠
Commit
```

```text
Insert Executed
≠
Insert Committed
```

```text
Update Executed
≠
Update Committed
```

```text
Delete Executed
≠
Delete Committed
```

```text
Generated ID
≠
Durable Row
```

```text
Persistence Failure
≠
Rollback
```

```text
Cancellation
≠
Rollback
```

```text
UNKNOWN
≠
FAILED
```

```text
UNKNOWN
≠
SUCCESS
```

```text
ORM Batch
≠
Bulk Operation
```

```text
Logical Operations
≠
Physical Statements
```

```text
Persistence Event
≠
Entity Lifecycle Event
```

```text
Persistence Event
≠
Query Event
```

```text
Persistence Event
≠
Transaction Event
```

```text
Persistence Event
≠
Domain Event
```

```text
Persistence Event
≠
Event Sourcing Event
```

y especialmente:

```text
Persistence Execution Knowledge
≠
Database Durability
```

---

# 253. Resultado arquitectónico

Con este sistema, VoltStack podrá observar detalladamente:

```text
flush
change analysis
persistence planning
insert operations
update operations
delete operations
batches
retries
failures
unknown outcomes
reconciliation
```

sin contaminar las responsabilidades de:

```text
Entity Lifecycle
Query Engine
Execution Engine
Transaction Manager
Cache
Telemetry
Domain Events
```

La cadena de autoridad quedará:

```text
Entity State System
=
estado ORM

UnitOfWork
=
trabajo ORM pendiente

Persistence Planner
=
plan de sincronización

Persistence Engine
=
ejecución de sincronización ORM

Query Engine
=
semántica de consultas

Execution Engine
=
ejecución de statements

Transaction Manager
=
commit / rollback / unknown authority

Persistence Consistency System
=
reconciliación ORM ↔ DB knowledge

Persistence Event System
=
observabilidad tipada del pipeline
```

---

# 254. Bloque 20 — Estado

```text
BLOCK 20 — EVENTS

✓ 209_DATABASE_EVENT_ARCHITECTURE.md
✓ 210_DATABASE_QUERY_EVENT_SYSTEM.md
✓ 211_DATABASE_CONNECTION_EVENT_SYSTEM.md
✓ 212_DATABASE_TRANSACTION_EVENT_PIPELINE.md
✓ 213_DATABASE_ENTITY_LIFECYCLE_EVENT_SYSTEM.md
✓ 214_DATABASE_PERSISTENCE_EVENT_SYSTEM.md
○ 215_DATABASE_EVENT_EXTENSION_SYSTEM.md
```

---

# 255. Siguiente documento

```text
215_DATABASE_EVENT_EXTENSION_SYSTEM.md
```

El siguiente documento cerrará el bloque de eventos definiendo cómo paquetes oficiales, aplicaciones y extensiones podrán incorporar:

```text
custom database events
custom listeners
event subscribers
event bridges
event descriptors
event policies
event middleware
event filtering
event aggregation
event sinks
event serializers
event adapters
```

sin romper las fronteras entre:

```text
Query Events
Connection Events
Transaction Events
Entity Lifecycle Events
Persistence Events
Domain Events
Telemetry
```

La regla central será:

> **Extender el Database Event System deberá significar añadir comportamiento observable mediante contratos públicos y estables, no obtener acceso irrestricto a los estados internos del Database Engine ni alterar silenciosamente sus invariantes.**