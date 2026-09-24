# 132_DATABASE_FLUSH_SYSTEM.md

# VoltStack Quantum Database
## Database Flush System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 132 — Database Flush System  
**Bloque:** 11 — Identity Map, Unit of Work & Persistence  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Flush System` define la frontera explícita mediante la cual VoltStack sincroniza el estado administrado por el ORM con la base de datos.

Su responsabilidad fundamental es transformar:

```text
PersistenceContext
+
UnitOfWork
+
Managed Entities
+
Entity Snapshots
+
Scheduled Operations
+
Lifecycle Mutations
```

en un:

```text
Stable PersistencePlan
```

y posteriormente coordinar su ejecución y reconciliación.

El flujo general es:

```text
EntityManager::flush()
        ↓
Flush Context
        ↓
UnitOfWork Collection
        ↓
Change Detection
        ↓
Lifecycle Stabilization
        ↓
Persistence Planning
        ↓
Plan Validation
        ↓
Plan Freeze
        ↓
Persistence Execution
        ↓
Outcome Aggregation
        ↓
ORM Reconciliation
        ↓
UnitOfWork Cleanup
```

Principio central:

> **`flush()` sincroniza el estado ORM con la base de datos; no significa `COMMIT`, no garantiza durabilidad transaccional y nunca deberá ocultar resultados de ejecución parciales o desconocidos.**

---

# 2. Problema arquitectónico

Antes de `flush()` pueden coexistir:

```text
NEW entities
MANAGED entities
DIRTY entities
REMOVED entities
relationship mutations
orphan removals
generated identities
optimistic versions
lifecycle callbacks
pending domain events
```

Además, estas operaciones pueden depender unas de otras.

Ejemplo:

```text
User NEW
 │
 └── Address NEW
       │
       └── Country MANAGED
```

Otro:

```text
Order MANAGED
 │
 ├── Line A UPDATED
 ├── Line B REMOVED
 └── Line C NEW
```

Ejecutar cada cambio inmediatamente impediría:

```text
dependency planning
batching
change consolidation
cycle resolution
lifecycle stabilization
operation ordering
transaction coordination
```

Por ello VoltStack necesita una frontera de sincronización explícita.

---

# 3. Definición

Formalmente:

```text
Flush
=
Stabilize(PersistenceContext)
+
Plan(UnitOfWork)
+
Execute(PersistencePlan)
+
Reconcile(ExecutionOutcomes)
```

No:

```text
Flush = Commit
```

ni:

```text
Flush = Save every object immediately
```

---

# 4. Posición arquitectónica

```text
Application
    │
    ▼
EntityManager
    │
    ├── persist()
    ├── remove()
    ├── managed mutations
    └── relationship mutations
    │
    ▼
UnitOfWork
    │
    ▼
┌──────────────────────────────────┐
│          FLUSH SYSTEM            │
│                                  │
│  Collect                         │
│     ↓                            │
│  Detect Changes                  │
│     ↓                            │
│  Lifecycle Stabilization         │
│     ↓                            │
│  Persistence Planner             │
│     ↓                            │
│  Freeze Plan                     │
│     ↓                            │
│  Execute                         │
│     ↓                            │
│  Reconcile                       │
└──────────────────────────────────┘
    │
    ▼
Persistence Engine
    │
    ▼
Query Engine
    │
    ▼
Execution Engine
    │
    ▼
Database
```

---

# 5. Separaciones fundamentales

```text
flush()
≠ commit()

flush()
≠ transaction()

flush()
≠ persist()

flush()
≠ save()

flush()
≠ change detection only

flush()
≠ SQL execution only

flush()
≠ UnitOfWork

flush()
≠ Persistence Planner

flush()
≠ Query Executor

flush()
≠ request termination

flush success
≠ transaction committed

flush failure
≠ database unchanged

flush retry
≠ universally safe
```

---

# 6. API principal

```php
interface FlushSystem
{
    public function flush(
        PersistenceContext $context,
        ?FlushOptions $options = null,
    ): FlushResult;
}
```

---

# 7. EntityManager

El API principal será normalmente:

```php
$entityManager->flush();
```

Internamente:

```text
EntityManager
    ↓
FlushSystem
```

El `EntityManager` coordina.

No implementa por sí mismo todo el algoritmo.

---

# 8. FlushOptions

```php
final readonly class FlushOptions
{
    public function __construct(
        public FlushMode $mode = FlushMode::AUTO,
        public FlushTransactionPolicy $transactionPolicy =
            FlushTransactionPolicy::JOIN_OR_CREATE,
        public FlushFailurePolicy $failurePolicy =
            FlushFailurePolicy::TAINT_ON_UNCERTAINTY,
    ) {}
}
```

---

# 9. FlushResult

```php
final readonly class FlushResult
{
    public function __construct(
        public FlushId $id,
        public FlushStatus $status,
        public FlushExecutionSummary $execution,
        public FlushReconciliationSummary $reconciliation,
        public FlushDiagnosticCollection $diagnostics,
    ) {}
}
```

---

# 10. FlushStatus

```php
enum FlushStatus
{
    case NO_CHANGES;
    case SUCCEEDED;
    case FAILED;
    case PARTIALLY_EXECUTED;
    case UNKNOWN;
}
```

---

# 11. NO_CHANGES

Caso:

```php
$entityManager->flush();
```

cuando:

```text
UnitOfWork = clean
```

Resultado:

```text
NO_CHANGES
```

sin necesidad de ejecutar queries.

---

# 12. FlushId

Cada intento obtiene identidad diagnóstica:

```php
final readonly class FlushId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Permite correlacionar:

```text
lifecycle
planning
queries
transactions
telemetry
errors
```

sin usar estado global.

---

# 13. FlushContext

```php
final readonly class FlushContext
{
    public function __construct(
        public FlushId $flushId,
        public PersistenceContext $persistenceContext,
        public UnitOfWork $unitOfWork,
        public IdentityMap $identityMap,
        public SnapshotRegistry $snapshots,
        public FlushOptions $options,
    ) {}
}
```

---

# 14. Estado interno del flush

El flush posee su propia máquina de estados.

```php
enum FlushPhase
{
    case CREATED;
    case COLLECTING;
    case DETECTING_CHANGES;
    case STABILIZING;
    case PLANNING;
    case VALIDATING;
    case FROZEN;
    case EXECUTING;
    case RECONCILING;
    case COMPLETED;
    case FAILED;
    case UNKNOWN;
}
```

---

# 15. FlushPhase ≠ EntityState

No confundir:

```text
FlushPhase::EXECUTING
```

con:

```text
EntityState::MANAGED
```

Son máquinas de estado distintas.

---

# 16. Pipeline maestro

```text
flush()
   │
   ▼
Acquire Flush Guard
   │
   ▼
Create FlushContext
   │
   ▼
preFlush
   │
   ▼
Collect UnitOfWork
   │
   ▼
Detect Changes
   │
   ▼
Resolve Relationships
   │
   ▼
Resolve Orphans
   │
   ▼
Run Pre-Persistence Lifecycle
   │
   ▼
Recompute Changes
   │
   ▼
Stabilize
   │
   ▼
Build Persistence Graph
   │
   ▼
Build Persistence Plan
   │
   ▼
Validate
   │
   ▼
Freeze
   │
   ▼
Transaction Coordination
   │
   ▼
Execute
   │
   ▼
Aggregate Outcomes
   │
   ▼
Reconcile
   │
   ▼
postPersist/postUpdate/postRemove
   │
   ▼
postFlush
   │
   ▼
Cleanup
   │
   ▼
Release Guard
```

---

# 17. Flush Guard

Antes de comenzar:

```text
PersistenceContext
```

debe adquirir un guard lógico.

---

# 18. Objetivo

Evitar:

```text
recursive flush
concurrent flush
plan mutation during execution
cross-context corruption
```

---

# 19. Recursive flush

Ejemplo prohibido:

```php
#[PrePersist]
public function beforePersist(): void
{
    $this->entityManager->flush();
}
```

Resultado:

```text
RecursiveFlushException
```

---

# 20. Regla

```text
FlushActive(Context)
⇒
SecondFlush(Context) rejected
```

---

# 21. Nested flush

VoltStack V1 no implementará:

```text
flush inside flush
```

como sub-flush independiente.

---

# 22. Razón

Generaría problemas con:

```text
snapshots
operation ordering
generated IDs
lifecycle callbacks
transactions
change sets
rollback
```

---

# 23. Concurrent flush

Tampoco se permitirá:

```text
Fiber A → flush(manager)
Fiber B → flush(same manager)
```

simultáneamente.

---

# 24. PersistenceContext ownership

Un `PersistenceContext` tiene un solo flush activo.

---

# 25. preFlush

`preFlush` es lifecycle de contexto.

No es callback de una entidad particular.

---

# 26. preFlush semantics

Permite trabajo controlado antes de recolectar el plan definitivo.

---

# 27. preFlush mutation

Puede permitir:

```text
persist(new entity)
remove(entity)
modify managed entity
modify relationship
```

bajo reglas controladas.

---

# 28. Consecuencia

Después de `preFlush`:

```text
UnitOfWork
```

debe volver a observarse.

---

# 29. No plan before preFlush

El plan definitivo no deberá congelarse antes de ejecutar las fases mutables permitidas.

---

# 30. UnitOfWork collection

La primera recolección identifica:

```text
new entities
managed entities
removed entities
explicitly dirty entities
relationship changes
collection changes
```

---

# 31. Change tracking

El sistema utiliza:

```text
125_DATABASE_CHANGE_TRACKING_SYSTEM.md
```

para detectar cambios.

---

# 32. Snapshots

Utiliza:

```text
126_DATABASE_ENTITY_SNAPSHOT_SYSTEM.md
```

para estrategias basadas en comparación.

---

# 33. ChangeSet

Resultado conceptual:

```text
Entity
  ↓
Current State
  +
Snapshot
  ↓
ChangeSet
```

---

# 34. ChangeSet ≠ UPDATE

Un ChangeSet puede terminar en:

```text
UPDATE
NO_OP
relationship operation
custom persistence operation
```

---

# 35. Dirty checking

Ejemplo:

```php
$user->rename('Alice');
```

No ejecuta UPDATE.

En flush:

```text
snapshot.name = Bob
current.name  = Alice
        ↓
ChangeSet
```

---

# 36. Explicit tracking

Si mapping usa explicit tracking:

```text
only entities marked dirty
```

son inspeccionadas según política.

---

# 37. Relationship collection

El flush debe incorporar:

```text
relationship additions
relationship removals
join-table changes
orphan candidates
cascade persistence
cascade removal
```

---

# 38. Cascade persist

Ejemplo:

```text
Order NEW
└── Line NEW
```

con:

```text
cascade persist
```

produce operaciones para ambos.

---

# 39. Cascade remove

Ejemplo:

```text
Parent REMOVED
└── Child MANAGED
```

puede programar child removal.

---

# 40. Cascade discovery

Debe ser:

```text
cycle-safe
idempotent
deterministic
```

---

# 41. Orphan resolution

No basta detectar que una entidad salió de una colección.

Debe determinarse:

```text
StillOrphanAtStabilization?
```

---

# 42. Lifecycle stabilization

Las fases `prePersist`, `preUpdate`, `preRemove` pueden alterar el UoW.

Ejemplo:

```text
prePersist(A)
    ↓
persist(B)
```

---

# 43. Problema

Si el plan ya estuviera congelado:

```text
B
```

quedaría fuera del flush o produciría una modificación inconsistente.

---

# 44. Solución

VoltStack utiliza un:

```text
Flush Stabilization Loop
```

---

# 45. Algoritmo conceptual

```text
pass = 1

repeat
    collect UnitOfWork
    detect changes
    resolve relationships
    resolve orphans
    execute eligible pre-lifecycle
    detect lifecycle mutations
    recompute affected ChangeSets
    validate graph

until stable
```

---

# 46. Estabilidad

```text
Stable(UoW)
=
NoNewScheduledEntities
∧
NoNewRemovals
∧
NoUnprocessedPersistentMutations
∧
NoUnresolvedRelationshipChanges
∧
NoNewOrphans
```

---

# 47. Bounded stabilization

Nunca:

```text
while(true)
```

sin límite.

---

# 48. Maximum passes

Configuración:

```php
'orm' => [
    'flush' => [
        'max_stabilization_passes' => 8,
    ],
],
```

---

# 49. Stabilization failure

Si no converge:

```text
FlushStabilizationException
```

---

# 50. Ejemplo de ciclo

```text
prePersist(A)
→ persist(B)

prePersist(B)
→ persist(C)

prePersist(C)
→ persist(A2)

...
```

debe detenerse.

---

# 51. Lifecycle execution once

Un mismo callback semántico no deberá ejecutarse repetidamente solo porque haya varias pasadas.

---

# 52. Lifecycle invocation registry

El flush mantiene:

```text
FlushLifecycleInvocationRegistry
```

---

# 53. Ejemplo

```text
prePersist(User#temp123)
```

ejecutado una vez.

Si después se recalcula el ChangeSet:

```text
no second prePersist
```

salvo que un contrato futuro declare explícitamente otra semántica.

---

# 54. preUpdate

Para una entidad dirty:

```text
ChangeSet
↓
preUpdate
↓
possible mutation
↓
recompute ChangeSet
```

---

# 55. Mutation after preUpdate

Ejemplo:

```php
#[PreUpdate]
public function updateTimestamp(): void
{
    $this->updatedAt = Clock::now();
}
```

El valor debe incorporarse al UPDATE.

---

# 56. postUpdate mutation

En cambio:

```text
postUpdate mutation
```

no modifica el plan ya ejecutado.

---

# 57. Future dirty state

Queda:

```text
MANAGED + DIRTY
```

para un flush posterior.

---

# 58. No recursive auto-flush

VoltStack no ejecutará automáticamente otro flush para persistir cambios de `post*`.

---

# 59. Persistence graph

Tras estabilización se construye:

```text
PersistenceOperationGraph
```

---

# 60. Nodes

Pueden incluir:

```text
InsertEntityOperation
UpdateEntityOperation
DeleteEntityOperation
RelationshipInsertOperation
RelationshipDeleteOperation
ForeignKeyUpdateOperation
CustomPersistenceOperation
```

---

# 61. Edges

Representan dependencias:

```text
A must execute before B
```

---

# 62. Ejemplo

```text
Insert Customer
      │
      ▼
Insert Order
      │
      ▼
Insert OrderLine
```

---

# 63. Delete ordering

Puede ser inverso:

```text
Delete OrderLine
      │
      ▼
Delete Order
```

---

# 64. Cycles

El planner debe resolver cuando sea posible mediante:

```text
nullable FK
deferred constraints
multi-phase updates
database capabilities
```

---

# 65. Unresolvable cycle

Produce:

```text
PersistenceDependencyCycleException
```

---

# 66. Persistence Planner

El flush delega a:

```text
128_DATABASE_PERSISTENCE_PLANNER_SYSTEM.md
```

---

# 67. Planner output

```text
PersistencePlan
```

---

# 68. PersistencePlan

Conceptualmente:

```php
final readonly class PersistencePlan
{
    public function __construct(
        public PersistencePlanId $id,
        public FlushId $flushId,
        public array $steps,
        public PersistenceDependencyGraph $graph,
        public PersistencePlanRequirements $requirements,
        public PersistencePlanFingerprint $fingerprint,
    ) {}
}
```

---

# 69. Plan validation

Antes de ejecución:

```text
validate(plan)
```

---

# 70. Validation checks

Entre otros:

```text
dependency consistency
entity state compatibility
identity requirements
connection requirements
transaction requirements
tenant consistency
shard consistency
optimistic lock requirements
capability compatibility
operation ordering
```

---

# 71. Plan freeze

Tras validación:

```text
PersistencePlan → FROZEN
```

---

# 72. Regla crítica

Después del freeze:

```text
plan structure cannot mutate
```

---

# 73. Frozen plan

No significa que las entidades sean inmutables.

Significa que cambios posteriores:

```text
do not silently alter current plan
```

---

# 74. Mutation after freeze

Puede:

```text
be rejected
be tracked for next flush
taint current flush if unsafe
```

según la fase.

---

# 75. During execution

Mutaciones estructurales del UoW desde callbacks internos deberán estar prohibidas.

---

# 76. Flush transaction policy

`flush()` no es `commit()`, pero puede requerir una transacción para ejecutar coherentemente múltiples operaciones.

---

# 77. Policies

```php
enum FlushTransactionPolicy
{
    case JOIN_OR_CREATE;
    case REQUIRE_EXISTING;
    case CREATE_IF_MULTIPLE_OPERATIONS;
    case NONE;
}
```

---

# 78. Default recomendado

```text
JOIN_OR_CREATE
```

para ORM write flushes.

---

# 79. Existing transaction

Si existe:

```text
Transaction = ACTIVE
```

el flush participa en ella.

---

# 80. No commit existing transaction

```text
flush()
```

no hace commit de una transacción propiedad de la aplicación.

---

# 81. Example

```php
$tx = $transactions->begin();

$em->persist($order);
$em->flush();

$payment->authorize();

$tx->commit();
```

Después del flush:

```text
Order synchronized
Transaction still active
```

---

# 82. Flush-created transaction

Si la política permite crear una transacción interna, debe existir ownership explícito.

---

# 83. Owned transaction

```text
FlushTransactionOwnership = OWNED
```

---

# 84. Joined transaction

```text
FlushTransactionOwnership = JOINED
```

---

# 85. NONE

Para ciertas operaciones/DBs:

```text
FlushTransactionOwnership = NONE
```

---

# 86. Important distinction

Incluso si el flush crea y finaliza una transacción interna:

```text
Flush abstraction
≠
Transaction abstraction
```

La transacción es una dependencia coordinada.

---

# 87. Transaction requirements

El `PersistencePlan` puede declarar:

```text
requiresTransaction = true
```

---

# 88. Capability mismatch

Si política:

```text
NONE
```

pero plan requiere transacción:

```text
FlushTransactionRequirementException
```

---

# 89. Multiple connections

Un flush que requiere varias conexiones introduce semántica distribuida.

---

# 90. V1 rule

Por defecto:

```text
one flush
→ one effective write database target
```

---

# 91. Cross-shard flush

No deberá convertirse accidentalmente en pseudo-transacción distribuida.

---

# 92. CrossDatabaseFlushPolicy

```php
enum CrossDatabaseFlushPolicy
{
    case REJECT;
    case EXPLICIT_NON_ATOMIC;
    case DISTRIBUTED_COORDINATOR_REQUIRED;
}
```

---

# 93. Default

```text
REJECT
```

para V1.

---

# 94. Tenant consistency

Un flush ordinario no mezcla:

```text
tenant A
tenant B
```

dentro del mismo PersistenceContext.

---

# 95. Execution phase

Una vez congelado:

```text
PersistencePlan
↓
PersistenceExecutor
```

---

# 96. PersistenceExecutor

```php
interface PersistencePlanExecutor
{
    public function execute(
        PersistencePlan $plan,
        PersistenceExecutionContext $context,
    ): PersistencePlanExecutionResult;
}
```

---

# 97. Step execution

```text
Step 1
↓
Outcome 1
↓
Step 2
↓
Outcome 2
↓
...
```

---

# 98. Insert steps

Utilizan:

```text
129_DATABASE_INSERT_PERSISTENCE_SYSTEM.md
```

---

# 99. Update steps

Utilizan:

```text
130_DATABASE_UPDATE_PERSISTENCE_SYSTEM.md
```

---

# 100. Delete steps

Utilizan:

```text
131_DATABASE_DELETE_PERSISTENCE_SYSTEM.md
```

---

# 101. Generated identity

Un INSERT puede producir:

```text
generated primary key
```

---

# 102. Dependency propagation

Ejemplo:

```text
INSERT User
→ generated id = 42
→ INSERT Profile(user_id = 42)
```

---

# 103. Generated value channel

El executor deberá poder propagar:

```text
GeneratedPersistenceValue
```

a steps dependientes.

---

# 104. Plan immutability

Esto no significa modificar la estructura del plan.

---

# 105. Runtime slots

El plan puede contener:

```text
GeneratedValueSlot<User.id>
```

que se resuelve durante ejecución.

---

# 106. Example

```text
Step 1:
  INSERT User
  produces slot USER_ID

Step 2:
  INSERT Profile
  consumes slot USER_ID
```

---

# 107. Generated identity reconciliation

Después de INSERT suficientemente cierto:

```text
assign generated ID
↓
construct EntityKey
↓
IdentityMap conflict check
↓
register/promote identity
↓
snapshot reconciliation
```

---

# 108. Generated identity conflict

Si:

```text
IdentityMap already contains User#42
```

para otra instancia:

```text
GeneratedIdentityConflictException
```

---

# 109. UNKNOWN insert

No establecer una identidad ficticia si el resultado no es suficientemente cierto.

---

# 110. Generated version

UPDATE puede producir:

```text
new version
updated timestamp
generated column
```

---

# 111. Reconciliation

Los valores generados se incorporan al estado administrado solo bajo reglas de certeza.

---

# 112. Optimistic locking

Un step puede fallar con:

```text
OptimisticLockConflict
```

---

# 113. Flush consequence

Si ocurre:

```text
FlushStatus = FAILED
```

o política equivalente.

---

# 114. Stop-on-failure

Default:

```text
stop executing subsequent dependent steps
```

---

# 115. Independent steps

Aunque sean aparentemente independientes, continuar tras fallo puede complicar atomicidad.

---

# 116. Default conservative policy

Dentro de un ORM flush:

```text
first semantic failure
→ stop plan execution
```

---

# 117. Failure phases

```php
enum FlushFailurePhase
{
    case PRE_FLUSH;
    case COLLECTION;
    case CHANGE_DETECTION;
    case STABILIZATION;
    case PLANNING;
    case VALIDATION;
    case TRANSACTION_PREPARATION;
    case EXECUTION;
    case RECONCILIATION;
    case POST_PERSISTENCE_LIFECYCLE;
    case POST_FLUSH;
    case CLEANUP;
}
```

---

# 118. Failure before execution

Si falla:

```text
planning
validation
pre lifecycle
```

y no se ha ejecutado DB I/O:

```text
DatabaseEffect = NONE
```

puede conocerse con certeza.

---

# 119. Failure during execution

Puede ser:

```text
CERTAIN_FAILURE
PARTIAL_EXECUTION
UNKNOWN
```

---

# 120. Partial execution

Ejemplo sin transacción:

```text
INSERT A → success
UPDATE B → success
DELETE C → failure
```

Resultado:

```text
PARTIALLY_EXECUTED
```

---

# 121. No false rollback assumption

Si no existe transacción:

```text
A and B may remain durable
```

---

# 122. Transactional partial execution

Dentro de transacción activa:

```text
some statements executed
```

pero todavía podrían ser revertidos.

---

# 123. Important dimensions

VoltStack deberá conservar por separado:

```text
StatementOutcome
TransactionOutcome
FlushOutcome
ORMReconciliationOutcome
```

---

# 124. Example

```text
StatementOutcome = SUCCEEDED
TransactionOutcome = ACTIVE
FlushOutcome = FAILED
```

es posible.

---

# 125. UNKNOWN

Caso:

```text
statement sent
↓
connection lost
```

---

# 126. Flush UNKNOWN

Si una operación crítica tiene outcome desconocido:

```text
FlushStatus = UNKNOWN
```

salvo que una capa inferior resuelva la incertidumbre.

---

# 127. EntityManager taint

Default:

```text
UNKNOWN
→ EntityManager::TAINTED
```

---

# 128. Why taint

El ORM ya no puede garantizar que:

```text
object state
=
database state
```

---

# 129. TAINTED restrictions

Un manager tainted no deberá permitir normalmente:

```text
persist()
remove()
flush()
refresh()
```

sin reconciliación/clear/close según política.

---

# 130. FlushFailurePolicy

```php
enum FlushFailurePolicy
{
    case TAINT_ON_UNCERTAINTY;
    case TAINT_ON_ANY_EXECUTION_FAILURE;
    case CLEAR_ON_FAILURE;
}
```

---

# 131. Default

```text
TAINT_ON_UNCERTAINTY
```

con reglas adicionales según reconciliación.

---

# 132. Reconciliation phase

Después de execution:

```text
PersistencePlanExecutionResult
↓
FlushReconciler
```

---

# 133. FlushReconciler

```php
interface FlushReconciler
{
    public function reconcile(
        PersistencePlan $plan,
        PersistencePlanExecutionResult $execution,
        FlushContext $context,
    ): FlushReconciliationResult;
}
```

---

# 134. Reconciliation responsibilities

Coordina:

```text
generated identifiers
generated values
EntityState
IdentityMap
Snapshots
ChangeSets
scheduled operations
relationship baselines
collection snapshots
```

---

# 135. Insert reconciliation

Conceptualmente:

```text
NEW
↓
INSERT success
↓
generated values
↓
IdentityMap registration
↓
snapshot baseline
↓
MANAGED
```

---

# 136. Update reconciliation

```text
MANAGED + DIRTY
↓
UPDATE success
↓
generated/version values
↓
new snapshot
↓
MANAGED + CLEAN
```

---

# 137. Delete reconciliation

```text
REMOVED
↓
DELETE success
↓
IdentityMap detach
↓
snapshot removal
↓
DETACHED
```

---

# 138. Relationship reconciliation

Después de éxito:

```text
collection current state
```

puede convertirse en:

```text
new collection baseline
```

---

# 139. ChangeSet cleanup

Los ChangeSets aplicados correctamente:

```text
discard
```

después de crear el nuevo baseline.

---

# 140. Do not clean failed entities

Una operación fallida no deberá marcarse limpia arbitrariamente.

---

# 141. Do not clean unknown entities

Tampoco una operación UNKNOWN.

---

# 142. Post-persistence lifecycle

Después de cada operación semánticamente exitosa:

```text
postPersist
postUpdate
postRemove
```

según corresponda.

---

# 143. Ordering

Debe ser determinista.

---

# 144. Post lifecycle ≠ transaction durability

Una vez más:

```text
postPersist
postUpdate
postRemove
postFlush
```

no implican commit.

---

# 145. postPersist mutation

Si callback modifica entidad:

```text
entity becomes dirty for next flush
```

---

# 146. postUpdate mutation

Igual.

---

# 147. postRemove mutation

No vuelve a administrar automáticamente la entidad.

---

# 148. postFlush

Se ejecuta después de completar la sincronización ORM del flush.

---

# 149. postFlush ≠ afterCommit

Regla absoluta:

```text
postFlush
≠
afterCommit
```

---

# 150. postFlush mutation

Puede crear:

```text
future UnitOfWork changes
```

pero no provoca un segundo flush automático.

---

# 151. Example

```php
#[PostFlush]
public function afterFlush(): void
{
    $this->entityManager->persist($audit);
}
```

Si se permite este patrón:

```text
$audit
```

queda pendiente para:

```text
next explicit flush
```

---

# 152. Recommended restriction

Preferentemente `postFlush` debería operar mediante un contexto restringido y no exponer arbitrariamente el EntityManager.

---

# 153. No automatic recursive flush

Nunca:

```text
postFlush
→ changes
→ automatic flush
→ postFlush
→ ...
```

---

# 154. Flush and domain events

Una entidad puede registrar:

```text
DomainEvent
```

durante el trabajo.

---

# 155. Recorded event

```text
DomainEvent recorded
≠
DomainEvent published
```

---

# 156. Event collection

El flush puede recolectar eventos de dominio relacionados con cambios exitosos.

---

# 157. Publication

Si requieren durabilidad:

```text
Transaction Commit
↓
Outbox / AfterCommit
↓
Publish
```

---

# 158. Transactional outbox

Puede formar parte del mismo `PersistencePlan` como operación persistente.

---

# 159. Advantage

```text
Entity mutation
+
Outbox record
```

pueden participar en la misma transacción.

---

# 160. External broker

La publicación real ocurre después.

---

# 161. Implicit flush

VoltStack deberá evitar flush implícito sorprendente.

---

# 162. Explicit default

```php
$entityManager->flush();
```

será la frontera principal.

---

# 163. Query-triggered autoflush

Podrá existir como modo opcional.

---

# 164. FlushMode

```php
enum FlushMode
{
    case MANUAL;
    case AUTO;
    case COMMIT_ONLY;
}
```

---

# 165. MANUAL

Solo flush explícito.

---

# 166. AUTO

Ciertas operaciones ORM podrán solicitar flush antes de una query cuando la coherencia semántica lo requiera.

---

# 167. COMMIT_ONLY

La sincronización puede vincularse explícitamente al flujo transaccional.

No será el comportamiento base sin configuración.

---

# 168. Autoflush danger

Ejemplo:

```php
$user->rename('Alice');

$repository->countByName('Alice');
```

Si la query debe observar el cambio dentro de la transacción:

```text
AUTO flush may be required
```

---

# 169. But

No toda query debe disparar flush.

---

# 170. Query space analysis

Una futura optimización puede determinar:

```text
Pending changes affect tables/query spaces used by query?
```

---

# 171. QuerySpace

Concepto:

```text
pending write set
∩
query read set
```

---

# 172. Autoflush condition

```text
AUTO_FLUSH_REQUIRED
=
PendingChanges
∧
ReadAfterWriteVisibilityRequired
∧
AffectedQuerySpacesOverlap
```

---

# 173. V1 simplification

VoltStack puede inicialmente usar:

```text
AUTO = flush all pending ORM changes before relevant entity queries
```

y optimizar después.

---

# 174. Model API

Ejemplo:

```php
$user->save();
```

La API Laravel-like deberá definir explícitamente si:

```text
save()
=
persist/register only
```

o:

```text
save()
=
persist + flush
```

---

# 175. Recommended VoltStack semantics

Para evitar dos motores:

```text
Model::save()
↓
canonical EntityManager/UoW
↓
flush according to Model API policy
```

Nunca SQL directo.

---

# 176. Save policy

Podría existir:

```php
$user->save(flush: true);
```

y:

```php
$user->save(flush: false);
```

---

# 177. Bulk persistence

El flush normal no debe materializar necesariamente un statement por entidad.

---

# 178. Batch planner

Operaciones compatibles podrán agruparse.

---

# 179. Example

```text
INSERT A
INSERT B
INSERT C
```

podrían convertirse downstream en:

```text
BatchInsertPersistenceStep
```

---

# 180. But

Batching no deberá cambiar:

```text
entity semantics
generated identity semantics
lifecycle semantics
failure semantics
```

---

# 181. Dedicated system

Se profundizará en:

```text
133_DATABASE_BATCH_PERSISTENCE_SYSTEM.md
```

---

# 182. Partial flush

API como:

```php
$entityManager->flush($entity);
```

parece conveniente, pero introduce riesgos.

---

# 183. Problem

La entidad puede depender de:

```text
other NEW entities
relationship changes
foreign keys
orphan removals
```

---

# 184. V1 recommendation

No soportar partial entity flush público.

---

# 185. Rule

```text
flush()
=
PersistenceContext synchronization boundary
```

no:

```text
flush(random subset)
```

---

# 186. Future selective flush

Si se implementa deberá operar sobre:

```text
dependency-closed persistence subgraph
```

---

# 187. Partial flush invariant

Nunca ejecutar un subconjunto que rompa dependencias.

---

# 188. Flush cancellation

Antes de ejecución puede existir cancelación.

---

# 189. Cancellation phases

```text
CREATED
COLLECTING
DETECTING_CHANGES
STABILIZING
PLANNING
VALIDATING
```

pueden ser cancelables.

---

# 190. After execution begins

La cancelación ya no implica:

```text
no DB effects
```

---

# 191. Cancellation outcome

```php
enum FlushCancellationOutcome
{
    case CANCELLED_BEFORE_EXECUTION;
    case CANCELLATION_REQUESTED_DURING_EXECUTION;
    case NOT_CANCELLABLE;
}
```

---

# 192. Cancellation token

```php
interface FlushCancellationToken
{
    public function isCancellationRequested(): bool;
}
```

---

# 193. Timeout

Flush puede tener presupuesto temporal.

---

# 194. FlushDeadline

```php
final readonly class FlushDeadline
{
    public function __construct(
        public DateTimeImmutable $at,
    ) {}
}
```

---

# 195. Propagation

El deadline deberá propagarse a:

```text
planner
query execution
statement timeout
connection acquisition
```

cuando sea soportado.

---

# 196. Timeout before execution

Resultado cierto:

```text
no database effect
```

si se sabe que ninguna operación comenzó.

---

# 197. Timeout during execution

Puede ser:

```text
FAILED
UNKNOWN
PARTIALLY_EXECUTED
```

según evidencia.

---

# 198. Cancellation ≠ rollback

Solicitar cancelación no significa que la DB haya revertido.

---

# 199. Flush retry

No se reintentará el flush completo ciegamente.

---

# 200. Why

Puede contener:

```text
generated IDs
non-idempotent triggers
partial execution
optimistic updates
delete side effects
```

---

# 201. Retry decision

Debe basarse en:

```text
operation replay safety
transaction outcome
execution certainty
driver failure classification
```

---

# 202. Whole flush replay

Solo permitido si existe evidencia de:

```text
NoDurableEffects
```

o mecanismo transaccional que garantice rollback.

---

# 203. Transaction rollback certainty

Si:

```text
transaction rolled back with certainty
```

puede abrirse la posibilidad de retry.

---

# 204. But object state

Aun así debe reconciliarse:

```text
generated IDs
versions
snapshots
lifecycle effects
```

antes de reintentar.

---

# 205. Lifecycle side effects

Callbacks pueden haber ejecutado código externo.

Por eso retry del flush puede no ser semánticamente seguro.

---

# 206. Recommendation

Lifecycle pre/post persistence deberá evitar side effects externos irreversibles.

---

# 207. Flush memory model

Durante un flush pueden coexistir:

```text
entities
snapshots
ChangeSets
PersistenceGraph
PersistencePlan
ExecutionResults
ReconciliationData
```

---

# 208. Large UnitOfWork

Puede consumir memoria significativa.

---

# 209. Memory governance

El sistema deberá medir:

```text
managed entity count
dirty entity count
scheduled operation count
snapshot bytes estimate
plan size
execution result size
```

---

# 210. Long-running jobs

Patrón recomendado:

```php
foreach ($chunks as $chunk) {
    // mutate entities

    $entityManager->flush();
    $entityManager->clear();
}
```

---

# 211. clear after flush

Permite liberar:

```text
IdentityMap
Snapshots
managed entities
```

---

# 212. Never auto-clear globally

`flush()` no deberá ejecutar `clear()` automáticamente por defecto.

---

# 213. Reason

El usuario puede esperar:

```text
same managed instances remain managed
```

después de flush.

---

# 214. Successful update state

Después de flush:

```text
MANAGED + CLEAN
```

continúa administrado.

---

# 215. Successful insert state

```text
NEW → MANAGED
```

---

# 216. Successful delete state

```text
REMOVED → DETACHED
```

---

# 217. UnitOfWork after successful flush

Debe contener únicamente:

```text
new changes produced after frozen/executed plan
```

si los hubiera.

---

# 218. Post lifecycle changes

Por ejemplo:

```text
postUpdate modifies entity
```

deben sobrevivir como pending changes.

---

# 219. Do not erase future dirty state

Cleanup deberá distinguir:

```text
changes applied by current flush
```

de:

```text
changes created after current operation
```

---

# 220. Snapshot baseline problem

Ejemplo:

```text
DB after UPDATE:
name = Alice

postUpdate:
entity.name = Carol
```

Nuevo baseline:

```text
snapshot.name = Alice
current.name  = Carol
```

Así:

```text
entity remains DIRTY
```

---

# 221. Correct reconciliation order

```text
apply DB-generated values
↓
capture persisted baseline
↓
execute post lifecycle
↓
detect future mutation
```

---

# 222. Wrong order

```text
postUpdate mutation
↓
snapshot current entity
```

haría desaparecer el cambio futuro.

---

# 223. Flush cleanup

```php
interface FlushCleanupSystem
{
    public function cleanup(
        FlushContext $context,
        FlushResult $result,
    ): void;
}
```

---

# 224. Cleanup responsibilities

```text
release flush guard
clear temporary lifecycle registry
clear temporary graph
release execution buffers
clear temporary generated-value slots
release planner state
```

---

# 225. Cleanup must not

```text
commit application transaction
clear managed context universally
flush again
publish durable domain events prematurely
```

---

# 226. Cleanup failure

Si falla después de successful execution:

```text
FlushOutcome
```

debe conservar que DB execution ocurrió.

---

# 227. Example

```text
Execution = SUCCEEDED
Reconciliation = SUCCEEDED
Cleanup = FAILED
```

No reportar simplemente:

```text
database write failed
```

---

# 228. Failure model

VoltStack conservará un:

```text
FlushFailureReport
```

---

# 229. FlushFailureReport

```php
final readonly class FlushFailureReport
{
    public function __construct(
        public FlushId $flushId,
        public FlushFailurePhase $phase,
        public FlushStatus $flushStatus,
        public PersistenceExecutionCertainty $certainty,
        public ?PersistenceOperationId $failedOperation,
        public TransactionOutcome $transactionOutcome,
        public FlushReconciliationStatus $reconciliationStatus,
        public Throwable $cause,
    ) {}
}
```

---

# 230. Diagnostic preservation

No perder:

```text
which phase failed
which operation failed
what executed before it
transaction state
reconciliation state
```

---

# 231. Error hierarchy

```text
DatabaseOrmException
└── FlushException
    ├── FlushAlreadyActiveException
    ├── RecursiveFlushException
    ├── ConcurrentFlushException
    ├── InvalidFlushStateException
    ├── FlushCollectionException
    ├── FlushChangeDetectionException
    ├── FlushLifecycleException
    ├── FlushStabilizationException
    ├── FlushStabilizationLimitException
    ├── FlushPlanningException
    ├── FlushPlanValidationException
    ├── FlushPlanMutationException
    ├── FlushDependencyException
    ├── FlushDependencyCycleException
    ├── FlushTransactionException
    ├── FlushTransactionRequirementException
    ├── CrossDatabaseFlushException
    ├── CrossShardFlushException
    ├── FlushExecutionException
    ├── FlushPartialExecutionException
    ├── FlushOutcomeUnknownException
    ├── FlushOptimisticLockException
    ├── FlushGeneratedValueException
    ├── FlushGeneratedIdentityException
    ├── FlushReconciliationException
    ├── FlushPartialReconciliationException
    ├── FlushPostLifecycleException
    ├── FlushCleanupException
    ├── FlushCancellationException
    ├── FlushTimeoutException
    ├── FlushRetrySafetyException
    ├── FlushRuntimeIsolationException
    └── FlushInvariantException
```

---

# 232. Persistent runtime

VoltStack está diseñado para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

donde el proceso puede sobrevivir a múltiples requests.

---

# 233. Shared state

Solo deberá compartirse si es inmutable:

```text
compiled metadata
compiled mappings
frozen lifecycle metadata
persistence strategy definitions
capability descriptors
```

---

# 234. Scoped flush state

Siempre scoped:

```text
FlushContext
FlushId
FlushPhase
UnitOfWork
IdentityMap
Snapshots
PersistenceGraph
PersistencePlan runtime
ExecutionResults
TransactionContext
TenantContext
LifecycleInvocationRegistry
```

---

# 235. No static current flush

Prohibido:

```php
static ?FlushContext $currentFlush;
```

---

# 236. Request A vs Request B

```text
Request A
  Flush A
  UoW A

Request B
  Flush B
  UoW B
```

Nunca:

```text
Flush A → Request B
```

---

# 237. Worker termination

Si request termina durante flush:

```text
runtime adapter
```

deberá limpiar el contexto lógico.

No deberá fingir resultado DB.

---

# 238. Abrupt worker death

Puede producir:

```text
UNKNOWN
```

desde la perspectiva de capas superiores.

---

# 239. Runtime reset

Reset deberá:

```text
release scoped references
discard pending UnitOfWork
release guards
clear runtime slots
```

---

# 240. Runtime reset ≠ rollback

Solo Transaction Manager puede afirmar rollback si existe evidencia.

---

# 241. Telemetry

Métricas propuestas:

```text
orm.flush.total
orm.flush.no_changes
orm.flush.success
orm.flush.failed
orm.flush.partial
orm.flush.unknown

orm.flush.duration
orm.flush.collection.duration
orm.flush.change_detection.duration
orm.flush.stabilization.duration
orm.flush.planning.duration
orm.flush.execution.duration
orm.flush.reconciliation.duration

orm.flush.stabilization.passes
orm.flush.entities.new
orm.flush.entities.dirty
orm.flush.entities.removed

orm.flush.operations.insert
orm.flush.operations.update
orm.flush.operations.delete
orm.flush.operations.relationship

orm.flush.generated_identifiers
orm.flush.optimistic_conflicts
orm.flush.reentrancy_blocked
orm.flush.concurrent_blocked
orm.flush.tainted_manager
```

---

# 242. High cardinality

No utilizar como metric labels:

```text
entity ID
tenant ID
query text
user ID
FlushId
```

---

# 243. FlushId in traces

Sí puede utilizarse como:

```text
trace/span attribute
```

según política de observabilidad.

---

# 244. Trace model

```text
orm.flush
 ├── collect
 ├── change_detection
 ├── lifecycle_stabilization
 ├── persistence_planning
 ├── transaction_prepare
 ├── execute
 │    ├── insert
 │    ├── update
 │    └── delete
 ├── reconcile
 └── cleanup
```

---

# 245. Debug toolbar

Podrá mostrar:

```text
Flush #F-123

Status:
  SUCCEEDED

Duration:
  18.4 ms

Managed:
  42

New:
  3

Dirty:
  7

Removed:
  2

Stabilization passes:
  2

Operations:
  INSERT 3
  UPDATE 7
  DELETE 2

Queries:
  9

Transaction:
  JOINED

Generated IDs:
  3

Optimistic conflicts:
  0

Post-flush dirty:
  1
```

---

# 246. Extension model

```php
interface FlushExtension
{
    public function contribute(
        FlushExtensionContext $context,
    ): FlushContribution;
}
```

---

# 247. Extension points

Podrán existir alrededor de:

```text
before collection
after collection
before planning
after planning
before execution
after execution
after reconciliation
```

pero no todos necesariamente públicos.

---

# 248. Internal vs public hooks

Debe distinguirse:

```text
ORM Lifecycle API
Framework Extension API
Internal Flush Pipeline Hook
```

---

# 249. Extension restrictions

Una extensión no podrá:

```text
recursively flush
mutate frozen plan silently
commit joined transaction
rollback joined transaction
bypass tenant constraints
ignore unknown outcomes
replace canonical EntityState
replace IdentityMap arbitrarily
```

---

# 250. Plan contribution

Una extensión que añada una operación deberá hacerlo:

```text
before plan freeze
```

---

# 251. After freeze contribution

Rechazada.

---

# 252. Deterministic ordering

Extensions tendrán:

```text
priority
stable registration identity
dependency declaration
```

---

# 253. Collision handling

No:

```text
last registered wins
```

para contribuciones incompatibles.

---

# 254. Security

Flush no reemplaza:

```text
authorization
validation
business invariants
```

---

# 255. Authorization timing

La autorización normalmente ocurre antes de llegar al persistence flush.

---

# 256. Persistence guards

Aun así, el sistema conserva invariantes como:

```text
tenant context
identity correctness
mandatory predicates
write routing
```

---

# 257. Query safety

Toda operación generada continúa usando:

```text
Query Model
typed parameters
SQL Compiler
prepared statements
```

---

# 258. No raw SQL

Flush nunca concatena SQL.

---

# 259. No credential handling

Flush no conoce passwords/DSNs.

---

# 260. No request-global tenant

Tenant se recibe desde el `PersistenceContext`.

---

# 261. Testing strategy

Debe cubrir:

```text
empty flush
insert flush
update flush
delete flush
mixed flush
relationship flush
cascade
orphans
lifecycle mutation
stabilization
generated identities
optimistic locking
failure
partial execution
unknown outcomes
transaction participation
rollback interaction
persistent workers
concurrency
extensions
telemetry
```

---

# 262. Test — empty flush

```text
clean UoW
→ flush
→ NO_CHANGES
→ zero DB statements
```

---

# 263. Test — persist does not flush

```php
$em->persist($user);
```

no ejecuta INSERT.

---

# 264. Test — remove does not flush

```php
$em->remove($user);
```

no ejecuta DELETE.

---

# 265. Test — update deferred

Mutar managed entity no ejecuta UPDATE antes de flush.

---

# 266. Test — insert

```text
NEW
→ flush
→ INSERT
→ MANAGED
```

---

# 267. Test — update

```text
MANAGED DIRTY
→ flush
→ UPDATE
→ MANAGED CLEAN
```

---

# 268. Test — delete

```text
REMOVED
→ flush
→ DELETE
→ DETACHED
```

---

# 269. Test — mixed operations

INSERT/UPDATE/DELETE respetan dependency graph.

---

# 270. Test — recursive flush

Callback intenta flush:

```text
RecursiveFlushException
```

---

# 271. Test — concurrent flush

Mismo manager desde dos fibers:

```text
ConcurrentFlushException
```

---

# 272. Test — lifecycle mutation

`prePersist` añade entidad.

Nueva entidad entra al mismo flush mediante stabilization.

---

# 273. Test — lifecycle once

`prePersist` no se ejecuta múltiples veces por varias stabilization passes.

---

# 274. Test — preUpdate mutation

Se recalcula ChangeSet.

---

# 275. Test — postUpdate mutation

Queda dirty para próximo flush.

---

# 276. Test — postFlush mutation

No produce recursive flush.

---

# 277. Test — stabilization limit

Mutación infinita falla determinísticamente.

---

# 278. Test — cascade cycle

No recursión infinita.

---

# 279. Test — dependency cycle

Se resuelve o falla explícitamente.

---

# 280. Test — plan freeze

Después del freeze no se modifica estructura.

---

# 281. Test — generated ID dependency

```text
INSERT parent
→ generated ID
→ INSERT child
```

funciona sin reconstruir el plan.

---

# 282. Test — generated identity conflict

Falla explícitamente.

---

# 283. Test — optimistic conflict

Detiene flush según policy.

---

# 284. Test — execution failure before any DB effect

Reporta certeza correcta.

---

# 285. Test — partial execution

No reporta simplemente FAILED ocultando operaciones exitosas.

---

# 286. Test — unknown outcome

Manager queda tainted según policy.

---

# 287. Test — transaction joined

Flush no hace commit.

---

# 288. Test — transaction owned

Ownership explícito y resultado registrado.

---

# 289. Test — cross-shard

Rechazado por defecto.

---

# 290. Test — cross-tenant

Rechazado.

---

# 291. Test — rollback

No asume que object state se reconcilia mágicamente.

---

# 292. Test — successful flush then rollback

Distingue:

```text
FlushOutcome = SUCCEEDED
TransactionOutcome = ROLLED_BACK
```

---

# 293. Test — snapshot baseline

Después de update exitoso refleja lo realmente persistido.

---

# 294. Test — postUpdate dirty

Nuevo cambio no desaparece durante cleanup.

---

# 295. Test — domain event

Recorded event no se publica como committed durante flush.

---

# 296. Test — outbox

Outbox operation puede integrarse en plan/transacción.

---

# 297. Test — cancellation before execution

No DB effects.

---

# 298. Test — cancellation during execution

No afirma ausencia de DB effects.

---

# 299. Test — timeout

Preserva certainty.

---

# 300. Test — retry

No replay ciego.

---

# 301. Test — large UoW

Métricas/memory governance funcionan.

---

# 302. Test — FrankenPHP

Request A no contamina Request B.

---

# 303. Test — RoadRunner

Worker reuse no conserva FlushContext.

---

# 304. Test — OpenSwoole

Coroutine scopes aislados.

---

# 305. Test — reset

No hace implicit flush.

---

# 306. Test — extension ordering

Determinista.

---

# 307. Test — extension after freeze

Rechazada.

---

# 308. Anti-patterns

## 308.1 `flush() = commit()`

Incorrecto.

## 308.2 `persist() = INSERT`

Incorrecto.

## 308.3 `remove() = DELETE`

Incorrecto.

## 308.4 Auto-flush al finalizar request

Prohibido por defecto.

## 308.5 Flush dentro de lifecycle callback

Prohibido.

## 308.6 Infinite stabilization

Prohibido.

## 308.7 Congelar plan antes de lifecycle mutable

Incorrecto.

## 308.8 Modificar plan durante execution

Prohibido.

## 308.9 Continuar ciegamente después de semantic failure

Peligroso.

## 308.10 Failure = DB unchanged

Incorrecto.

## 308.11 Timeout = rollback

Incorrecto.

## 308.12 UNKNOWN = FAILED

Incorrecto.

## 308.13 Retry completo automático

Peligroso.

## 308.14 postFlush = afterCommit

Incorrecto.

## 308.15 Publicar domain events irreversibles desde postFlush

Peligroso.

## 308.16 Snapshot después de postUpdate mutation

Puede ocultar future dirty state.

## 308.17 Auto-clear después de cada flush

No debe ser comportamiento universal.

## 308.18 Partial entity flush sin dependency closure

Peligroso.

## 308.19 Cross-shard flush implícito

Prohibido.

## 308.20 Estado estático del flush en persistent workers

Prohibido.

---

# 309. Architectural Invariants

## DB-ORM-FLUSH-001
`flush()` será la frontera explícita de sincronización ORM.

## DB-ORM-FLUSH-002
`flush()` será distinto de `commit()`.

## DB-ORM-FLUSH-003
`flush()` será distinto de `persist()`.

## DB-ORM-FLUSH-004
`flush()` será distinto de `remove()`.

## DB-ORM-FLUSH-005
`flush()` será distinto del UnitOfWork.

## DB-ORM-FLUSH-006
`flush()` será distinto del Persistence Planner.

## DB-ORM-FLUSH-007
`flush()` será distinto del Query Executor.

## DB-ORM-FLUSH-008
Flush success no implicará transaction commit.

## DB-ORM-FLUSH-009
Flush failure no implicará database unchanged.

## DB-ORM-FLUSH-010
Flush retry no será universalmente seguro.

## DB-ORM-FLUSH-011
Cada flush tendrá un FlushContext scoped.

## DB-ORM-FLUSH-012
Cada flush tendrá una identidad diagnóstica.

## DB-ORM-FLUSH-013
Flush phases serán explícitas.

## DB-ORM-FLUSH-014
FlushPhase será distinto de EntityState.

## DB-ORM-FLUSH-015
Un PersistenceContext tendrá como máximo un flush activo.

## DB-ORM-FLUSH-016
Recursive flush será rechazado.

## DB-ORM-FLUSH-017
Concurrent flush sobre el mismo manager será rechazado.

## DB-ORM-FLUSH-018
Nested flush no será soportado implícitamente.

## DB-ORM-FLUSH-019
Flush guard será liberado durante cleanup.

## DB-ORM-FLUSH-020
preFlush ocurrirá antes del plan definitivo.

## DB-ORM-FLUSH-021
preFlush podrá producir cambios solo dentro de mutation policy.

## DB-ORM-FLUSH-022
Cambios de preFlush serán recolectados.

## DB-ORM-FLUSH-023
UnitOfWork será recolectado antes de planning.

## DB-ORM-FLUSH-024
Change Tracking será utilizado como fuente canónica de cambios.

## DB-ORM-FLUSH-025
Snapshots serán utilizados según tracking strategy.

## DB-ORM-FLUSH-026
ChangeSet será distinto de SQL UPDATE.

## DB-ORM-FLUSH-027
Relationship changes formarán parte de la sincronización.

## DB-ORM-FLUSH-028
Cascade persist será cycle-safe.

## DB-ORM-FLUSH-029
Cascade remove será cycle-safe.

## DB-ORM-FLUSH-030
Cascade scheduling será idempotente.

## DB-ORM-FLUSH-031
Orphan removal será evaluado sobre estado estabilizado.

## DB-ORM-FLUSH-032
Lifecycle pre-events podrán provocar recomputación.

## DB-ORM-FLUSH-033
Flush utilizará stabilization loop.

## DB-ORM-FLUSH-034
Stabilization loop será bounded.

## DB-ORM-FLUSH-035
Stabilization no convergente fallará explícitamente.

## DB-ORM-FLUSH-036
prePersist no será reinvocado arbitrariamente por cada pass.

## DB-ORM-FLUSH-037
preUpdate mutation será incluida en ChangeSet final.

## DB-ORM-FLUSH-038
postUpdate mutation pertenecerá al siguiente flush.

## DB-ORM-FLUSH-039
post lifecycle no provocará auto recursive flush.

## DB-ORM-FLUSH-040
El UnitOfWork deberá estar estable antes de congelar plan.

## DB-ORM-FLUSH-041
PersistenceOperationGraph será explícito.

## DB-ORM-FLUSH-042
Graph nodes serán operaciones semánticas.

## DB-ORM-FLUSH-043
Graph edges representarán dependencias.

## DB-ORM-FLUSH-044
Persistence Planner determinará ordering.

## DB-ORM-FLUSH-045
Dependency cycles deberán resolverse o fallar explícitamente.

## DB-ORM-FLUSH-046
PersistencePlan será validado antes de ejecución.

## DB-ORM-FLUSH-047
PersistencePlan será congelado antes de ejecución.

## DB-ORM-FLUSH-048
Frozen plan no podrá mutarse silenciosamente.

## DB-ORM-FLUSH-049
Cambios posteriores al freeze no alterarán silenciosamente el plan actual.

## DB-ORM-FLUSH-050
Plan mutation durante execution será prohibida.

## DB-ORM-FLUSH-051
Flush podrá coordinar una transacción sin convertirse conceptualmente en Transaction Manager.

## DB-ORM-FLUSH-052
Joined transaction no será committed por flush.

## DB-ORM-FLUSH-053
Transaction ownership será explícito.

## DB-ORM-FLUSH-054
Plan requirements podrán exigir transacción.

## DB-ORM-FLUSH-055
Policy incompatible con transaction requirements fallará.

## DB-ORM-FLUSH-056
Cross-database flush no será implícitamente atómico.

## DB-ORM-FLUSH-057
Cross-shard flush será rechazado por defecto.

## DB-ORM-FLUSH-058
Cross-tenant flush será rechazado por defecto.

## DB-ORM-FLUSH-059
Execution consumirá un plan congelado.

## DB-ORM-FLUSH-060
Insert execution utilizará Insert Persistence System.

## DB-ORM-FLUSH-061
Update execution utilizará Update Persistence System.

## DB-ORM-FLUSH-062
Delete execution utilizará Delete Persistence System.

## DB-ORM-FLUSH-063
Generated values podrán alimentar operaciones dependientes.

## DB-ORM-FLUSH-064
Generated-value propagation no mutará estructura del plan.

## DB-ORM-FLUSH-065
Generated IDs se reconciliarán solo con suficiente certeza.

## DB-ORM-FLUSH-066
Generated ID conflict será explícito.

## DB-ORM-FLUSH-067
UNKNOWN insert no fabricará identidad.

## DB-ORM-FLUSH-068
Generated versions se reconciliarán explícitamente.

## DB-ORM-FLUSH-069
Optimistic conflict será semantic failure.

## DB-ORM-FLUSH-070
Default execution policy se detendrá ante primer semantic failure.

## DB-ORM-FLUSH-071
FlushFailurePhase será preservada.

## DB-ORM-FLUSH-072
Failure antes de DB I/O podrá declarar no database effects.

## DB-ORM-FLUSH-073
Failure durante execution conservará certainty.

## DB-ORM-FLUSH-074
Partial execution será first-class.

## DB-ORM-FLUSH-075
Partial execution no se ocultará bajo generic failure.

## DB-ORM-FLUSH-076
StatementOutcome será distinto de TransactionOutcome.

## DB-ORM-FLUSH-077
TransactionOutcome será distinto de FlushOutcome.

## DB-ORM-FLUSH-078
FlushOutcome será distinto de ReconciliationOutcome.

## DB-ORM-FLUSH-079
UNKNOWN será first-class.

## DB-ORM-FLUSH-080
UNKNOWN podrá taint EntityManager.

## DB-ORM-FLUSH-081
Manager tainted no continuará como si estuviera sincronizado.

## DB-ORM-FLUSH-082
Reconciliation será una fase explícita.

## DB-ORM-FLUSH-083
Insert success reconciliará EntityState.

## DB-ORM-FLUSH-084
Insert success reconciliará IdentityMap.

## DB-ORM-FLUSH-085
Insert success reconciliará snapshot.

## DB-ORM-FLUSH-086
Update success producirá nuevo persisted baseline.

## DB-ORM-FLUSH-087
Delete success reconciliará IdentityMap.

## DB-ORM-FLUSH-088
Delete success eliminará active snapshot.

## DB-ORM-FLUSH-089
Relationship success actualizará collection baseline.

## DB-ORM-FLUSH-090
Applied ChangeSets serán limpiados.

## DB-ORM-FLUSH-091
Failed ChangeSets no serán limpiados arbitrariamente.

## DB-ORM-FLUSH-092
Unknown ChangeSets no serán limpiados arbitrariamente.

## DB-ORM-FLUSH-093
postPersist ocurrirá solo tras semantic insert success.

## DB-ORM-FLUSH-094
postUpdate ocurrirá solo tras semantic update success.

## DB-ORM-FLUSH-095
postRemove ocurrirá solo tras semantic remove success.

## DB-ORM-FLUSH-096
postPersist no significará commit.

## DB-ORM-FLUSH-097
postUpdate no significará commit.

## DB-ORM-FLUSH-098
postRemove no significará commit.

## DB-ORM-FLUSH-099
postFlush no significará commit.

## DB-ORM-FLUSH-100
postFlush no será afterCommit.

## DB-ORM-FLUSH-101
post lifecycle mutation será future dirty state.

## DB-ORM-FLUSH-102
post lifecycle mutation no provocará implicit second flush.

## DB-ORM-FLUSH-103
Recorded domain event será distinto de published domain event.

## DB-ORM-FLUSH-104
Durability-dependent event publication ocurrirá después de transaction outcome.

## DB-ORM-FLUSH-105
Transactional outbox podrá participar en PersistencePlan.

## DB-ORM-FLUSH-106
Flush explícito será comportamiento base.

## DB-ORM-FLUSH-107
Autoflush será política explícita.

## DB-ORM-FLUSH-108
No toda query disparará flush necesariamente.

## DB-ORM-FLUSH-109
Query-space analysis podrá optimizar autoflush.

## DB-ORM-FLUSH-110
Model API utilizará el mismo Flush System.

## DB-ORM-FLUSH-111
Model save no generará SQL directamente.

## DB-ORM-FLUSH-112
Batching no alterará semántica ORM.

## DB-ORM-FLUSH-113
Batching no alterará lifecycle semantics.

## DB-ORM-FLUSH-114
Batching no ocultará failure semantics.

## DB-ORM-FLUSH-115
Partial entity flush no será público en V1.

## DB-ORM-FLUSH-116
Selective flush futuro requerirá dependency closure.

## DB-ORM-FLUSH-117
Cancellation antes de execution podrá garantizar no DB effect cuando sea demostrable.

## DB-ORM-FLUSH-118
Cancellation durante execution no implicará no DB effects.

## DB-ORM-FLUSH-119
Cancellation no significará rollback.

## DB-ORM-FLUSH-120
Flush deadline será propagable.

## DB-ORM-FLUSH-121
Timeout conservará execution certainty.

## DB-ORM-FLUSH-122
Timeout no será interpretado automáticamente como rollback.

## DB-ORM-FLUSH-123
Whole-flush retry no será ciego.

## DB-ORM-FLUSH-124
Retry requerirá replay safety.

## DB-ORM-FLUSH-125
Rollback certainty podrá influir en retry safety.

## DB-ORM-FLUSH-126
Lifecycle external side effects podrán hacer retry inseguro.

## DB-ORM-FLUSH-127
Flush tendrá memory governance.

## DB-ORM-FLUSH-128
Large UoW deberá ser observable.

## DB-ORM-FLUSH-129
Flush no ejecutará auto-clear por defecto.

## DB-ORM-FLUSH-130
Managed inserted entities continuarán managed tras flush.

## DB-ORM-FLUSH-131
Managed updated entities continuarán managed tras flush.

## DB-ORM-FLUSH-132
Deleted entities quedarán detached tras reconciliation.

## DB-ORM-FLUSH-133
Successful flush limpiará solo trabajo aplicado.

## DB-ORM-FLUSH-134
Post-flush future dirty state no será eliminado por cleanup.

## DB-ORM-FLUSH-135
Persisted baseline se capturará antes de considerar post-lifecycle future mutations.

## DB-ORM-FLUSH-136
Cleanup liberará temporary flush state.

## DB-ORM-FLUSH-137
Cleanup no hará commit.

## DB-ORM-FLUSH-138
Cleanup no hará rollback.

## DB-ORM-FLUSH-139
Cleanup no hará recursive flush.

## DB-ORM-FLUSH-140
Cleanup failure no borrará evidencia de successful DB execution.

## DB-ORM-FLUSH-141
Failure reports preservarán phase.

## DB-ORM-FLUSH-142
Failure reports preservarán execution certainty.

## DB-ORM-FLUSH-143
Failure reports preservarán transaction outcome.

## DB-ORM-FLUSH-144
Failure reports preservarán reconciliation status.

## DB-ORM-FLUSH-145
Compiled immutable metadata podrá compartirse entre workers.

## DB-ORM-FLUSH-146
FlushContext será scoped.

## DB-ORM-FLUSH-147
UnitOfWork será scoped.

## DB-ORM-FLUSH-148
IdentityMap será scoped.

## DB-ORM-FLUSH-149
SnapshotRegistry será scoped.

## DB-ORM-FLUSH-150
PersistencePlan runtime será scoped.

## DB-ORM-FLUSH-151
Lifecycle invocation state será scoped.

## DB-ORM-FLUSH-152
No existirá static current flush.

## DB-ORM-FLUSH-153
Request A no compartirá flush con Request B.

## DB-ORM-FLUSH-154
Coroutine A no compartirá flush state con B.

## DB-ORM-FLUSH-155
Runtime reset no ejecutará implicit flush.

## DB-ORM-FLUSH-156
Runtime reset no afirmará rollback.

## DB-ORM-FLUSH-157
Telemetry no utilizará entity IDs como metric labels.

## DB-ORM-FLUSH-158
Extensions contribuirán antes del plan freeze.

## DB-ORM-FLUSH-159
Extensions no mutarán frozen plan.

## DB-ORM-FLUSH-160
VoltStack solo considerará un flush correctamente sincronizado cuando el UnitOfWork haya sido estabilizado, el PersistencePlan validado y congelado, las operaciones ejecutadas con outcomes suficientemente conocidos y el estado ORM reconciliado sin confundir esa sincronización con la durabilidad final de una transacción.

---

# 310. Estructura propuesta

```text
src/Quantum/Database/ORM/Flush/
│
├── Contract/
│   ├── FlushSystem.php
│   ├── FlushGuard.php
│   ├── FlushStabilizer.php
│   ├── FlushReconciler.php
│   └── FlushCleanupSystem.php
│
├── Context/
│   ├── FlushContext.php
│   ├── FlushId.php
│   ├── FlushOptions.php
│   ├── FlushMode.php
│   └── FlushPhase.php
│
├── Guard/
│   ├── DefaultFlushGuard.php
│   ├── FlushGuardToken.php
│   └── FlushConcurrencyGuard.php
│
├── Collection/
│   ├── UnitOfWorkCollector.php
│   ├── FlushChangeCollector.php
│   ├── FlushRelationshipCollector.php
│   └── FlushOrphanCollector.php
│
├── Stabilization/
│   ├── DefaultFlushStabilizer.php
│   ├── FlushStabilizationPass.php
│   ├── FlushStabilizationResult.php
│   ├── FlushStabilityDetector.php
│   └── FlushLifecycleInvocationRegistry.php
│
├── Planning/
│   ├── FlushPersistenceGraphBuilder.php
│   ├── FlushPlanValidator.php
│   ├── FlushPlanFreezer.php
│   └── FlushPlanningResult.php
│
├── Transaction/
│   ├── FlushTransactionCoordinator.php
│   ├── FlushTransactionPolicy.php
│   ├── FlushTransactionOwnership.php
│   └── CrossDatabaseFlushPolicy.php
│
├── Execution/
│   ├── PersistencePlanExecutor.php
│   ├── PersistencePlanExecutionResult.php
│   ├── FlushExecutionSummary.php
│   ├── GeneratedValueSlot.php
│   └── GeneratedValueRegistry.php
│
├── Reconciliation/
│   ├── DefaultFlushReconciler.php
│   ├── FlushReconciliationResult.php
│   ├── FlushReconciliationStatus.php
│   ├── FlushReconciliationSummary.php
│   └── FlushPersistedBaselineManager.php
│
├── Cancellation/
│   ├── FlushCancellationToken.php
│   ├── FlushCancellationOutcome.php
│   └── FlushDeadline.php
│
├── Failure/
│   ├── FlushFailurePhase.php
│   ├── FlushFailurePolicy.php
│   └── FlushFailureReport.php
│
├── Result/
│   ├── FlushResult.php
│   └── FlushStatus.php
│
├── Extension/
│   ├── FlushExtension.php
│   ├── FlushExtensionRegistry.php
│   ├── FlushExtensionContext.php
│   └── FlushContribution.php
│
├── Telemetry/
│   ├── FlushTelemetry.php
│   ├── FlushStatistics.php
│   └── FlushDiagnostics.php
│
└── Exception/
    └── ...
```

---

# 311. Fórmulas fundamentales

## 311.1 Flush

```text
Flush(C)
=
Stabilize(UnitOfWork(C))
+
Plan(UnitOfWork(C))
+
Execute(PersistencePlan(C))
+
Reconcile(C)
```

---

# 312. Flush readiness

```text
FlushReady
=
ContextOpen
∧
NoActiveFlush
∧
UnitOfWorkValid
∧
RuntimeScopeValid
```

---

# 313. Stabilization

```text
Stable(U)
=
NoNewEntities(U)
∧
NoNewRemovals(U)
∧
NoUnprocessedPersistentMutations(U)
∧
NoUnresolvedRelationshipChanges(U)
∧
NoUnresolvedOrphans(U)
```

---

# 314. Stabilization loop

```text
Uₙ₊₁
=
Lifecycle(
    RelationshipResolution(
        ChangeDetection(Uₙ)
    )
)
```

hasta:

```text
Uₙ₊₁ = Uₙ
```

semánticamente.

---

# 315. Bounded stabilization

```text
Stable(U)
∨
PassCount ≥ MaxPasses
```

Si se alcanza el límite sin estabilidad:

```text
FlushStabilizationException
```

---

# 316. Plan eligibility

```text
PersistencePlanAllowed
=
UnitOfWorkCollected
∧
ChangeSetsComputed
∧
LifecyclePrePhasesCompleted
∧
LifecycleMutationsStabilized
∧
RelationshipGraphResolved
∧
PersistenceGraphValid
```

---

# 317. Frozen plan

```text
FrozenPlan
=
ValidatedPlan
∧
StructuralMutationDisabled
```

---

# 318. Generated value dependency

```text
Operation B depends on GeneratedValue(A)
⇒
A ≺ B
```

---

# 319. Successful flush

```text
SuccessfulFlush
=
PlanExecutedSuccessfully
∧
RequiredOutcomesCertain
∧
ORMReconciliationSuccessful
```

No incluye necesariamente:

```text
TransactionCommitted
```

---

# 320. Durability

```text
DurableFlushEffects
=
SuccessfulFlush
∧
TransactionOutcome = COMMITTED
```

cuando las operaciones participan en una transacción requerida.

---

# 321. Unknown flush

```text
∃ operation ∈ Plan:
    Outcome(operation) = UNKNOWN
⇒
FlushCertainty ≤ UNKNOWN
```

salvo reconciliación posterior concluyente.

---

# 322. Partial execution

```text
PartialExecution
=
∃ a,b ∈ Plan:
    ExecutedSuccessfully(a)
∧
NotSuccessfullyExecuted(b)
```

---

# 323. Reconciliation

```text
FlushReconciliation
=
EntityStateReconciliation
+
IdentityMapReconciliation
+
SnapshotReconciliation
+
ChangeSetReconciliation
+
RelationshipBaselineReconciliation
+
GeneratedValueReconciliation
```

---

# 324. Post-update future dirty state

```text
PersistedBaseline(E) = StatePersistedByCurrentFlush(E)

PostLifecycleState(E) ≠ PersistedBaseline(E)
⇒
DirtyForNextFlush(E)
```

---

# 325. Transaction separation

```text
FlushSucceeded
⇏
TransactionCommitted
```

---

# 326. Retry safety

```text
SafeFlushRetry
=
ExecutionEffectsKnown
∧
TransactionOutcomeKnown
∧
ReplaySafetySatisfied
∧
LifecycleSideEffectsSafe
∧
ORMStateReconciledForRetry
```

---

# 327. Autoflush

```text
AutoFlushRequired
=
PendingWrites
∧
VisibilityRequirement
∧
RelevantQuerySpaceOverlap
```

---

# 328. Runtime safety

```text
SafeFlushRuntime
=
ScopedFlushContext
∧
ScopedUnitOfWork
∧
ScopedIdentityMap
∧
ScopedSnapshots
∧
NoCrossRequestState
∧
NoUnsynchronizedConcurrentFlush
∧
DeterministicReset
```

---

# 329. Arquitectura maestra

```text
                         Application
                              │
                              ▼
                        EntityManager
                              │
              ┌───────────────┼────────────────┐
              ▼               ▼                ▼
           persist()        remove()       mutations
              │               │                │
              └───────────────┼────────────────┘
                              ▼
                          UnitOfWork
                              │
                              ▼
                           flush()
                              │
                              ▼
                        ┌───────────┐
                        │Flush Guard│
                        └─────┬─────┘
                              ▼
                           preFlush
                              │
                              ▼
                     UnitOfWork Collection
                              │
                              ▼
                       Change Detection
                              │
                              ▼
                    Relationship Resolution
                              │
                              ▼
                       Orphan Resolution
                              │
                              ▼
                 Pre-Persistence Lifecycle
                              │
                              ▼
                     Change Recalculation
                              │
                              ▼
                   Stabilization Detector
                         │           │
                   unstable         stable
                         │           │
                         └── repeat  ▼
                              Persistence Graph
                                    │
                                    ▼
                           Persistence Planner
                                    │
                                    ▼
                             Validate Plan
                                    │
                                    ▼
                              Freeze Plan
                                    │
                                    ▼
                       Transaction Coordination
                                    │
                                    ▼
                           Persistence Executor
                     ┌──────────────┼──────────────┐
                     ▼              ▼              ▼
                   INSERT         UPDATE         DELETE
                     │              │              │
                     └──────────────┼──────────────┘
                                    ▼
                          Execution Outcomes
                                    │
                   ┌────────────────┼────────────────┐
                   ▼                ▼                ▼
                SUCCESS           FAILED          UNKNOWN
                   │                                 │
                   ▼                                 ▼
              Reconciliation                     TAINT?
                   │
          ┌────────┼────────┬────────────┐
          ▼        ▼        ▼            ▼
       Identity  Snapshot  State      UnitOfWork
          Map
          └────────┼────────┴────────────┘
                   ▼
            Post Persistence
                Lifecycle
                   │
                   ▼
                postFlush
                   │
                   ▼
                 Cleanup
                   │
                   ▼
              FlushResult
                   │
                   ▼
         Transaction remains separate
```

---

# 330. Master Formula

```text
Database Flush System
=
Explicit Synchronization Boundary
+
Flush Context
+
Flush State Machine
+
Flush Guard
+
Reentrancy Protection
+
Concurrency Protection
+
preFlush Lifecycle
+
UnitOfWork Collection
+
Change Detection
+
Entity Snapshot Comparison
+
Relationship Change Collection
+
Cascade Resolution
+
Orphan Resolution
+
Pre-Persistence Lifecycle
+
Lifecycle Mutation Detection
+
ChangeSet Recalculation
+
Bounded Stabilization
+
Persistence Operation Graph
+
Dependency Resolution
+
Persistence Planning
+
Plan Validation
+
Plan Freeze
+
Transaction Coordination
+
Transaction Ownership
+
Cross-Database Protection
+
Persistence Plan Execution
+
Insert Persistence
+
Update Persistence
+
Delete Persistence
+
Generated Value Slots
+
Generated Identity Reconciliation
+
Optimistic Concurrency Handling
+
Execution Outcome Aggregation
+
Partial Execution Modeling
+
UNKNOWN Outcome Preservation
+
Entity State Reconciliation
+
IdentityMap Reconciliation
+
Snapshot Reconciliation
+
Relationship Baseline Reconciliation
+
Post-Persistence Lifecycle
+
postFlush Lifecycle
+
Future Dirty State Preservation
+
Domain Event Separation
+
After-Commit Separation
+
Cancellation
+
Timeout
+
Retry Safety
+
Memory Governance
+
Persistent Runtime Isolation
+
Telemetry
+
Diagnostics
+
Extension Governance
+
Failure Modeling
```

---

# 331. Master Rule

> **En VoltStack, `flush()` será la frontera explícita de sincronización entre el `PersistenceContext` y la base de datos. Antes de ejecutar cualquier operación, deberá recolectar y estabilizar el `UnitOfWork`, recalcular los cambios producidos por relaciones y lifecycle handlers, construir un grafo de persistencia, generar un `PersistencePlan` válido y congelarlo. Después deberá ejecutar INSERT, UPDATE, DELETE y operaciones relacionadas conservando la certeza real de cada resultado y reconciliar `EntityState`, `IdentityMap`, snapshots y ChangeSets. Un flush exitoso significa que la sincronización ORM correspondiente fue realizada correctamente; nunca deberá interpretarse por sí solo como prueba de que la transacción haya sido committed o que sus efectos sean definitivamente durables.**

---

# 332. Estado del Bloque 11

Hasta este punto:

```text
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
```

La arquitectura fundamental queda ahora conectada:

```text
                         ORM
                          │
                          ▼
                    EntityManager
                          │
                          ▼
                      UnitOfWork
                    /     |      \
                   /      |       \
             Identity  Change   Snapshots
                Map    Tracking
                   \      |       /
                    \     |      /
                     ▼    ▼     ▼
                      flush()
                         │
                         ▼
                Persistence Planner
                         │
                         ▼
                  PersistencePlan
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
        INSERT          UPDATE         DELETE
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                   Query Engine
                         │
                         ▼
                 Execution Engine
                         │
                         ▼
                     Database
                         │
                         ▼
                  Reconciliation
```

Quedan dos documentos para cerrar el bloque:

```text
133_DATABASE_BATCH_PERSISTENCE_SYSTEM.md
134_DATABASE_PERSISTENCE_CONSISTENCY_SYSTEM.md
```

---

# 333. Siguiente documento

```text
133_DATABASE_BATCH_PERSISTENCE_SYSTEM.md
```

Deberá formalizar:

```text
batch persistence architecture
operation grouping
batch eligibility
INSERT batching
UPDATE batching
DELETE batching
multi-row INSERT
statement reuse
prepared statement reuse
parameter matrix
generated identity handling
generated values
batch cardinality
entity-to-row correlation
partial batch failure
per-row outcome certainty
all-or-nothing vs partial semantics
optimistic locking in batches
relationship operations
dependency-aware batching
batch boundaries
transaction integration
database capability detection
MySQL/MariaDB/PostgreSQL/SQLite differences
maximum parameter limits
packet/query-size limits
adaptive batch sizing
memory governance
streaming batch execution
UnitOfWork integration
PersistencePlan integration
lifecycle preservation
reconciliation
retry safety
telemetry
persistent runtime isolation
extension model
testing
```

Regla central propuesta:

> **El batching en VoltStack será una optimización física del `PersistencePlan`, nunca una alteración de la semántica ORM: agrupar operaciones deberá preservar identidad, lifecycle, dependencias, generated values, optimistic locking, outcomes y reconciliación como si cada operación hubiese conservado su significado individual.**