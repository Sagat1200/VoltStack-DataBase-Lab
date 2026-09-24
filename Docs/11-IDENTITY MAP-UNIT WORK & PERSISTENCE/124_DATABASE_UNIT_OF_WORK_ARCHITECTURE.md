# 124_DATABASE_UNIT_OF_WORK_ARCHITECTURE.md

# VoltStack Quantum Database
## Database Unit of Work Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 124 — Database Unit of Work Architecture  
**Bloque:** 11 — Identity Map, Unit of Work & Persistence  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Unit of Work Architecture` define la arquitectura mediante la cual el ORM de VoltStack observa, registra, coordina y transforma los cambios realizados sobre entidades administradas en un conjunto determinista de operaciones de persistencia.

La responsabilidad central puede expresarse como:

```text
Application
    ↓
Entity Mutations
    ↓
UnitOfWork
    ↓
Change Detection
    ↓
Entity Change Graph
    ↓
Persistence Planner
    ↓
Persistence Plan
    ↓
Query Model / AST
    ↓
Execution Engine
```

La regla fundamental será:

> **UnitOfWork describe y coordina los cambios ORM pendientes dentro de un PersistenceContext; no genera SQL, no ejecuta queries y no constituye por sí mismo una transacción de base de datos.**

---

# 2. Problema que resuelve

Una aplicación puede modificar un grafo completo de objetos:

```php
$order->changeAddress($address);
$order->addItem($item);
$order->customer()->rename('Alice');

$entityManager->remove($oldItem);
```

El ORM necesita determinar:

```text
¿Qué entidades son nuevas?
¿Cuáles fueron modificadas?
¿Cuáles fueron eliminadas?
¿Qué relaciones cambiaron?
¿Qué colecciones cambiaron?
¿Qué operaciones dependen de otras?
¿Qué IDs aún deben generarse?
¿Qué cascades deben descubrirse?
¿Qué orphan removals existen?
¿En qué orden debe persistirse todo?
```

UnitOfWork centraliza esa coordinación.

---

# 3. UnitOfWork ≠ Transaction

Esta separación es crítica.

```text
UnitOfWork
≠
Database Transaction
```

UnitOfWork representa:

```text
in-memory persistence work
```

Transaction representa:

```text
database atomicity + isolation boundary
```

---

# 4. Ejemplo

```php
$user->rename('Alice');
```

puede hacer que UnitOfWork detecte:

```text
User#10
name:
    old = Bob
    new = Alice
```

sin que exista todavía una transacción activa.

---

# 5. flush() ≠ commit()

```php
$entityManager->flush();
```

significa:

```text
synchronize pending ORM state
```

No significa necesariamente:

```text
COMMIT
```

---

# 6. Transaction externa

Será válido:

```php
$transactions->transactional(function () use ($entityManager) {
    $entityManager->persist($order);
    $entityManager->flush();

    // additional work

    $entityManager->flush();
});
```

con:

```text
one transaction
+
multiple flushes
```

---

# 7. UnitOfWork ≠ Identity Map

El documento `123_DATABASE_IDENTITY_MAP_SYSTEM.md` estableció:

```text
IdentityMap:
EntityKey → canonical entity instance
```

UnitOfWork mantiene información sobre:

```text
entity persistence work
```

---

# 8. Relación

```text
PersistenceContext
│
├── IdentityMap
│     └── Who is the canonical object?
│
└── UnitOfWork
      └── What persistence work is pending?
```

---

# 9. UnitOfWork ≠ Entity State

`Entity State System` define estados como:

```text
NEW
MANAGED
REMOVED
DETACHED
```

UnitOfWork consume y coordina esos estados.

No deberá redefinir su semántica.

---

# 10. UnitOfWork ≠ Change Tracking

Change Tracking responde:

> ¿Qué cambió en una entidad?

UnitOfWork responde:

> ¿Qué conjunto completo de cambios debe coordinarse para persistencia?

Por tanto:

```text
ChangeTracking
⊂
UnitOfWork workflow
```

pero:

```text
ChangeTracking
≠
UnitOfWork
```

---

# 11. UnitOfWork ≠ Snapshot System

Snapshots mantienen:

```text
original known entity state
```

UnitOfWork los consume para:

```text
dirty checking
ChangeSet calculation
```

---

# 12. UnitOfWork ≠ Persistence Engine

UnitOfWork produce información estructurada sobre cambios.

Persistence Engine transforma esa información en operaciones ejecutables.

```text
UnitOfWork
    ↓
Change Model
    ↓
Persistence Engine
```

---

# 13. UnitOfWork ≠ Persistence Planner

UnitOfWork puede descubrir dependencias lógicas.

Persistence Planner decide:

```text
physical persistence ordering
operation grouping
execution strategy
```

---

# 14. UnitOfWork ≠ Query Builder

Nunca deberá producir directamente:

```php
DB::table('users')->update(...);
```

---

# 15. UnitOfWork ≠ SQL Compiler

Nunca deberá generar:

```sql
UPDATE users SET name = ? WHERE id = ?
```

---

# 16. UnitOfWork ≠ Query Executor

Nunca deberá llamar directamente:

```text
PDOStatement::execute()
```

---

# 17. Arquitectura general

```text
                     PersistenceContext
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
     IdentityMap      EntityStateRegistry   UnitOfWork
          │                 │                 │
          └──────────┬──────┘                 │
                     │                        │
                     ▼                        ▼
                Managed Entities       Change Discovery
                                              │
                          ┌───────────────────┼───────────────────┐
                          │                   │                   │
                          ▼                   ▼                   ▼
                    Field Changes      Relation Changes    Collection Changes
                          │                   │                   │
                          └───────────────────┼───────────────────┘
                                              ▼
                                       EntityChangeGraph
                                              │
                                              ▼
                                      Persistence Planner
                                              │
                                              ▼
                                       Persistence Plan
                                              │
                                              ▼
                                       Persistence Engine
                                              │
                                              ▼
                                      Query Model / AST
```

---

# 18. UnitOfWork lifecycle

Conceptualmente:

```text
EMPTY
  ↓
TRACKING
  ↓
PREPARING
  ↓
FROZEN
  ↓
FLUSHING
  ↓
RECONCILING
  ↓
TRACKING
```

con estados alternativos:

```text
TAINTED
CLOSED
```

---

# 19. UnitOfWorkState

```php
enum UnitOfWorkState
{
    case EMPTY;
    case TRACKING;
    case PREPARING;
    case FROZEN;
    case FLUSHING;
    case RECONCILING;
    case TAINTED;
    case CLOSED;
}
```

---

# 20. EMPTY

Significa:

```text
no tracked persistence work
```

No necesariamente:

```text
IdentityMap empty
```

Puede haber entidades managed sin cambios.

---

# 21. TRACKING

Estado normal durante el trabajo de aplicación.

Permite:

```text
persist()
remove()
entity mutations
relationship mutations
```

---

# 22. PREPARING

Durante preparación de flush:

```text
discover cascades
detect changes
normalize scheduled work
calculate ChangeSets
build dependency graph
```

---

# 23. FROZEN

Una vez construido el input estable para Persistence Planner:

```text
pending work
```

deberá quedar congelado para ese ciclo de flush.

---

# 24. FLUSHING

Persistence Engine está ejecutando el plan derivado del UnitOfWork.

---

# 25. RECONCILING

Después de ejecución:

```text
generated IDs
snapshots
entity states
scheduled removals
dirty flags
relationship state
```

deben reconciliarse con el outcome real.

---

# 26. TAINTED

Significa:

```text
UnitOfWork can no longer safely assume
that its in-memory state matches known persistence effects
```

Ejemplo:

```text
connection lost after sending UPDATE
outcome UNKNOWN
```

---

# 27. CLOSED

No acepta nuevo trabajo.

---

# 28. Scope

UnitOfWork pertenece a:

```text
one PersistenceContext
```

---

# 29. Scope invariant

```text
PersistenceContext A
→ UnitOfWork A

PersistenceContext B
→ UnitOfWork B
```

Nunca:

```text
global UnitOfWork
```

---

# 30. Core contract

Contrato conceptual:

```php
interface UnitOfWork
{
    public function persist(object $entity): void;

    public function remove(object $entity): void;

    public function registerManaged(object $entity): void;

    public function detach(object $entity): void;

    public function state(object $entity): EntityState;

    public function prepareFlush(): PreparedUnitOfWork;

    public function reconcile(
        PersistenceOutcome $outcome
    ): void;

    public function clear(): void;

    public function size(): int;
}
```

---

# 31. UnitOfWork no será God Object

El contrato público puede coordinar múltiples subsistemas internos:

```text
UnitOfWork
│
├── EntityRegistration
├── ChangeTracker
├── SnapshotRegistry
├── ScheduledOperationRegistry
├── RelationshipChangeTracker
├── CollectionChangeTracker
├── CascadeDiscovery
├── OrphanRemovalRegistry
├── ChangeGraphBuilder
└── FlushPreparation
```

---

# 32. persist()

Regla:

```text
persist(entity)
≠
INSERT entity
```

---

# 33. persist() significa

Conceptualmente:

```text
bring entity into persistence coordination
```

dependiendo de su estado actual.

---

# 34. persist NEW

```text
NEW entity
    ↓ persist()
scheduled for persistence
```

---

# 35. persist MANAGED

Normalmente:

```text
MANAGED entity
    ↓ persist()
no duplicate scheduling
```

La operación deberá ser idempotente.

---

# 36. persist REMOVED

La política deberá ser explícita.

Puede representar:

```text
cancel scheduled removal
```

si todavía no se ejecutó y el lifecycle lo permite.

No deberá decidirse mediante efectos laterales accidentales.

---

# 37. persist DETACHED

V1 deberá rechazarlo por defecto:

```text
DetachedEntityPassedToPersistException
```

en lugar de implementar un `merge()` implícito.

---

# 38. persist UNKNOWN

Si el estado persistente es incierto:

```text
persist()
```

no deberá convertirlo mágicamente a `MANAGED`.

---

# 39. remove()

Regla:

```text
remove(entity)
≠
DELETE immediately
```

---

# 40. remove MANAGED

```text
MANAGED
   ↓
remove()
   ↓
scheduled removal
```

La transición detallada pertenece al Entity State System.

---

# 41. remove NEW

Si la entidad todavía no fue persistida:

```text
NEW
→ remove()
```

puede cancelar su futura inserción.

---

# 42. remove DETACHED

Debe rechazarse por defecto.

---

# 43. remove UNKNOWN

No deberá realizarse automáticamente sin reconciliación.

---

# 44. Scheduled work

UnitOfWork mantendrá registros conceptuales separados:

```text
scheduled insertions
scheduled updates
scheduled removals
scheduled relation changes
scheduled collection changes
scheduled orphan removals
```

---

# 45. Scheduling ≠ physical operation

```text
ScheduledInsertion
```

significa:

```text
entity requires persistence as new entity
```

No significa todavía:

```text
one SQL INSERT
```

---

# 46. Persistence Planner puede transformar

Por ejemplo:

```text
100 ScheduledInsertions
```

podrían convertirse en:

```text
100 individual INSERTs
```

o:

```text
bulk INSERT groups
```

si las capacidades y semántica lo permiten.

---

# 47. Entity registration

Una entidad puede ser registrada en UnitOfWork mediante:

```text
hydration
persist()
relationship cascade
reference resolution
generated entity graph discovery
```

---

# 48. registerManaged()

Hydration podrá registrar:

```text
Entity
+
EntityKey
+
original snapshot
```

como managed.

---

# 49. Atomic registration

Deberá coordinarse:

```text
IdentityMap
+
EntityStateRegistry
+
SnapshotRegistry
+
UnitOfWork
```

---

# 50. Registration inconsistency

No deberá quedar:

```text
IdentityMap contains entity
EntityState says DETACHED
UnitOfWork thinks MANAGED
```

---

# 51. PersistenceContext coordinator

Se recomienda:

```text
PersistenceContextCoordinator
```

para operaciones multi-registry.

---

# 52. UnitOfWork entry

Podrá existir:

```php
final class UnitOfWorkEntry
{
    public function __construct(
        public readonly object $entity,
        public readonly EntityType $type,
        public readonly EntityState $state,
    ) {}
}
```

Pero no deberá duplicar arbitrariamente todo el estado mantenido por otros registries.

---

# 53. Object identity indexing

UnitOfWork podrá indexar internamente mediante:

```text
spl_object_id(entity)
```

porque está rastreando una instancia PHP específica.

---

# 54. Object ID ≠ Entity ID

```text
spl_object_id
≠
EntityIdentifier
```

---

# 55. NEW entities without persistent IDs

UnitOfWork deberá poder rastrearlas antes de que exista `EntityKey`.

---

# 56. TemporaryEntityToken

Se propone:

```php
final readonly class TemporaryEntityToken
{
    public function __construct(
        public int|string $value,
    ) {}
}
```

---

# 57. Temporary token purpose

Permite representar:

```text
NEW Order
NEW OrderItem
```

en un graph antes de generar IDs.

---

# 58. TemporaryEntityToken ≠ persistent identity

Nunca deberá:

```text
serialize as entity ID
expose through API
become database identifier
enter second-level cache as identity
```

---

# 59. EntityHandle

Puede utilizarse una abstracción:

```text
EntityHandle
```

que represente:

```text
managed established EntityKey
```

o:

```text
temporary UnitOfWork token
```

---

# 60. Conceptual model

```php
interface EntityHandle {}

final readonly class PersistentEntityHandle implements EntityHandle
{
    public function __construct(
        public EntityKey $key
    ) {}
}

final readonly class TemporaryEntityHandle implements EntityHandle
{
    public function __construct(
        public TemporaryEntityToken $token
    ) {}
}
```

---

# 61. EntityHandle ≠ application API

Es una abstracción interna de persistencia.

---

# 62. Change detection

Durante `prepareFlush()`:

```text
managed entities
    ↓
Change Tracking System
    ↓
ChangeSets
```

---

# 63. ChangeSet

Conceptualmente:

```text
ChangeSet
=
FieldChanges
+
RelationshipChanges
+
CollectionChanges
+
PersistenceMetadata
```

La estructura exacta se formalizará en:

```text
125_DATABASE_CHANGE_TRACKING_SYSTEM.md
```

---

# 64. ChangeSet ≠ SQL

Ejemplo:

```text
User#10

name:
    Bob → Alice

status:
    pending → active
```

no:

```sql
UPDATE users ...
```

---

# 65. Dirty detection

Una entidad podrá estar:

```text
MANAGED
```

pero sin cambios.

Entonces:

```text
ChangeSet = empty
```

---

# 66. Dirty entity

Conceptualmente:

```text
MANAGED + non-empty ChangeSet
→ requires persistence consideration
```

---

# 67. DIRTY no necesariamente Entity State persistente

Si `DIRTY` se utiliza como estado conceptual, deberá mantenerse clara la distinción entre:

```text
lifecycle state
```

y:

```text
change-tracking condition
```

---

# 68. Change tracking strategies

VoltStack soportará conceptualmente:

```text
SNAPSHOT
EXPLICIT
NOTIFY
CUSTOM
```

---

# 69. SNAPSHOT

```text
OriginalSnapshot
vs
CurrentState
→ ChangeSet
```

---

# 70. EXPLICIT

La aplicación/ORM marca cambios explícitamente.

---

# 71. NOTIFY

La entidad o instrumentación notifica mutaciones.

---

# 72. CUSTOM

Una extensión controlada define la estrategia.

---

# 73. Strategy ≠ UnitOfWork

UnitOfWork coordina el resultado.

No deberá contener toda la lógica de cada estrategia.

---

# 74. Snapshot ownership

Los snapshots pertenecen al:

```text
Entity Snapshot System
```

del documento:

```text
126_DATABASE_ENTITY_SNAPSHOT_SYSTEM.md
```

---

# 75. UnitOfWork consumes snapshots

```text
SnapshotRegistry
       ↓
ChangeTracker
       ↓
ChangeSet
       ↓
UnitOfWork
```

---

# 76. Relationship changes

No todos los cambios persistentes son fields escalares.

Ejemplo:

```php
$order->setCustomer($customer);
```

puede implicar:

```text
relationship ownership change
foreign-key persistence change
```

---

# 77. Collection changes

Ejemplo:

```php
$user->roles()->add($adminRole);
$user->roles()->remove($viewerRole);
```

puede producir:

```text
collection additions
collection removals
join-table operations
orphan rules
```

---

# 78. Collection snapshot

Para relaciones to-many puede requerirse:

```text
original collection state
vs
current collection state
```

---

# 79. Collection identity

La comparación deberá basarse en:

```text
entity identity
```

y no necesariamente en:

```text
PHP array index
```

---

# 80. Collection order

Si el mapping declara orden persistente:

```text
collection reorder
```

puede ser un cambio persistente.

Si no:

```text
in-memory order change
```

puede ser irrelevante.

---

# 81. Relationship ownership

UnitOfWork deberá respetar metadata:

```text
owning side
inverse side
```

---

# 82. Inverse-only mutation

Modificar únicamente el lado inverso podría no generar persistencia si el owning side no cambió.

Esto deberá ser diagnosticable.

---

# 83. Consistency validation

Opcionalmente:

```text
relationship consistency validator
```

podrá detectar:

```text
owning side says A
inverse side says B
```

---

# 84. Cascade persist

Supongamos:

```php
$order = new Order();
$order->addItem(new OrderItem());

$entityManager->persist($order);
```

con:

```text
cascade persist = true
```

UnitOfWork deberá descubrir también `OrderItem`.

---

# 85. Cascade discovery

```text
persist(Order)
    ↓
inspect relationship metadata
    ↓
discover OrderItem
    ↓
register NEW OrderItem
```

---

# 86. Cascade ≠ recursion without limits

El graph puede contener ciclos:

```text
A → B → C → A
```

---

# 87. Visited set

Cascade discovery deberá usar:

```text
visited object identity / entity handle
```

para evitar loops infinitos.

---

# 88. Cascade persist ≠ database cascade

Regla:

```text
ORM cascade persist
≠
DB ON INSERT CASCADE
```

---

# 89. Cascade remove ≠ ON DELETE CASCADE

Igualmente:

```text
ORM cascade remove
≠
ON DELETE CASCADE
```

---

# 90. Orphan removal

Ejemplo:

```php
$order->removeItem($item);
```

con:

```text
orphanRemoval = true
```

puede programar:

```text
DELETE OrderItem
```

---

# 91. Orphan removal ≠ collection removal

Remover de una colección no implica automáticamente eliminar la entidad.

Solo cuando metadata lo declara.

---

# 92. Orphan candidate

UnitOfWork deberá distinguir:

```text
OrphanCandidate
```

de:

```text
ConfirmedOrphanRemoval
```

---

# 93. Orphan stabilization

Antes de programar eliminación:

```text
candidate orphan
```

debe verificarse contra el estado final del graph.

Una entidad removida y posteriormente reinsertada en la relación durante el mismo UnitOfWork puede dejar de ser orphan.

---

# 94. Stabilization phase

Esto justifica una fase:

```text
discover
→ normalize
→ stabilize
→ freeze
```

---

# 95. Cascade discovery fixed point

El graph deberá descubrirse hasta alcanzar:

```text
fixed point
```

---

# 96. Fórmula

```text
G₀ = explicitly registered entities

Gₙ₊₁ =
Gₙ
∪
CascadeDiscoveries(Gₙ)
∪
RelationshipDependencies(Gₙ)

Stop when:

Gₙ₊₁ = Gₙ
```

---

# 97. Bounded discovery

La implementación deberá protegerse contra:

```text
unbounded graph expansion
```

mediante resource governance.

---

# 98. Maximum graph size

Podrán existir límites:

```text
maxTrackedEntities
maxCascadeDepth
maxRelationshipEdges
maxChangeSetCount
```

---

# 99. Exceeding limits

Nunca:

```text
silently truncate graph
```

Debe producir:

```text
UnitOfWorkResourceLimitException
```

---

# 100. Entity Change Graph

El resultado lógico central será:

```text
EntityChangeGraph
```

---

# 101. Graph nodes

Cada node puede representar:

```text
EntityHandle
EntityType
PersistenceIntent
ChangeSet
State
Metadata reference
```

---

# 102. Graph edges

Ejemplos:

```text
REQUIRES_ID_FROM
MUST_EXIST_BEFORE
RELATIONSHIP_DEPENDS_ON
DELETE_AFTER
CASCADE_FROM
ORPHANED_BY
```

---

# 103. Graph ≠ execution order

El graph describe dependencias.

No decide necesariamente el orden físico final.

---

# 104. Example insert graph

```text
Order
 │
 ├── requires Customer
 │
 └── parent of
      │
      ▼
   OrderItem
```

Podría producir:

```text
Customer
→ Order
→ OrderItem
```

si IDs/FKs lo requieren.

---

# 105. Existing referenced entity

Si Customer ya existe:

```text
Customer#10 MANAGED
```

puede ser una dependency node sin requerir INSERT.

---

# 106. Generated identifier dependency

Ejemplo:

```text
Order
AUTO_INCREMENT ID
    ↓
OrderItem.order_id
```

Entonces:

```text
OrderItem insert
```

depende del generated identifier de `Order`.

---

# 107. Dependency type

```text
REQUIRES_GENERATED_IDENTIFIER
```

deberá ser explícita.

---

# 108. Cycles

Ejemplo:

```text
A requires B
B requires A
```

---

# 109. Cycle ≠ arbitrary ordering

No deberá resolverse mediante:

```text
sort alphabetically
```

---

# 110. Cycle strategies

Dependiendo de metadata/capabilities:

```text
nullable intermediate FK
deferred constraints
post-insert update
application-generated IDs
manual strategy
unsupported cycle
```

---

# 111. UnitOfWork doesn't select SQL strategy

Puede identificar:

```text
dependency cycle
```

pero Persistence Planner decide cómo resolverla.

---

# 112. PersistenceIntent

Se propone:

```php
enum PersistenceIntent
{
    case INSERT;
    case UPDATE;
    case DELETE;
    case RELATIONSHIP_CHANGE;
    case COLLECTION_CHANGE;
    case NO_OP;
}
```

---

# 113. Intent ≠ statement

```text
INSERT intent
```

no equivale necesariamente a:

```text
one SQL INSERT
```

---

# 114. PreparedUnitOfWork

`prepareFlush()` deberá producir un objeto immutable.

```php
final readonly class PreparedUnitOfWork
{
    public function __construct(
        public UnitOfWorkId $unitOfWorkId,
        public FlushCycleId $flushCycleId,
        public EntityChangeGraph $changeGraph,
        public ChangeSetCollection $changeSets,
        public PersistenceIntentCollection $intents,
        public UnitOfWorkFingerprint $fingerprint,
    ) {}
}
```

---

# 115. Mutable UoW ≠ PreparedUnitOfWork

```text
UnitOfWork
=
mutable tracking context
```

```text
PreparedUnitOfWork
=
immutable flush input
```

---

# 116. Freeze boundary

Una vez producido:

```text
PreparedUnitOfWork
```

las mutaciones posteriores no deberán alterar ese objeto.

---

# 117. Application mutation during flush

Modificar entidades mientras el flush está en curso es peligroso.

---

# 118. Reentrant mutation policy

V1 deberá prohibir o restringir:

```text
arbitrary entity graph mutation during FLUSHING
```

---

# 119. Lifecycle callbacks

Un callback como:

```text
prePersist
preUpdate
```

puede modificar estado.

---

# 120. Callback stabilization

Por ello el pipeline puede requerir:

```text
detect changes
→ invoke allowed pre-lifecycle hooks
→ detect affected changes again
→ stabilize
→ freeze
```

---

# 121. Infinite lifecycle mutation

Ejemplo:

```text
preUpdate changes field
→ new ChangeSet
→ callback changes another field
→ ...
```

debe estar acotado.

---

# 122. Stabilization iteration limit

VoltStack deberá establecer:

```text
maxStabilizationPasses
```

---

# 123. Non-convergent UoW

Si no alcanza fixed point:

```text
UnitOfWorkStabilizationException
```

---

# 124. Lifecycle callback ≠ domain event

La coordinación con eventos deberá mantener la distinción establecida en ORM Architecture.

---

# 125. postPersist timing

`postPersist` no significa:

```text
transaction committed
```

UnitOfWork no deberá reinterpretarlo como tal.

---

# 126. Flush pipeline

```text
TRACKING
   │
   ▼
Validate Context
   │
   ▼
Discover Cascades
   │
   ▼
Detect Changes
   │
   ▼
Discover Relationship Changes
   │
   ▼
Discover Orphans
   │
   ▼
Run Allowed Pre-Lifecycle Hooks
   │
   ▼
Recompute Affected Changes
   │
   ▼
Stabilize Graph
   │
   ▼
Validate Graph
   │
   ▼
Freeze
   │
   ▼
PreparedUnitOfWork
   │
   ▼
Persistence Planner
```

---

# 127. No hidden query during preparation

`prepareFlush()` no deberá ejecutar consultas arbitrarias para “adivinar” el estado DB.

---

# 128. Explicit validation query

Si una estrategia requiere validación DB:

```text
database validation requirement
```

deberá expresarse explícitamente y ejecutarse en la capa correspondiente.

---

# 129. UnitOfWork fingerprint

`PreparedUnitOfWork` podrá tener:

```text
UnitOfWorkFingerprint
```

---

# 130. Fingerprint purpose

Permite detectar:

```text
stale persistence plans
unexpected mutation
mismatched reconciliation
diagnostic correlation
```

---

# 131. Fingerprint inputs

Podrá incluir:

```text
PersistenceContext identity
entity handles
entity types
persistence intents
ChangeSet fingerprints
relationship changes
relevant metadata version
```

---

# 132. Fingerprint excludes live object addresses

No deberá depender de:

```text
spl_object_id
memory address
random iteration order
```

para representación semántica durable.

---

# 133. Determinism

Mismos inputs lógicos deberán producir:

```text
same PreparedUnitOfWork semantics
```

---

# 134. Object traversal order

No deberá depender accidentalmente de:

```text
hash-map insertion order
reflection property order not guaranteed by contract
```

sin normalización.

---

# 135. Persistence Planner handoff

```text
PreparedUnitOfWork
        ↓
PersistencePlanner
        ↓
PersistencePlan
```

---

# 136. UnitOfWork does not execute plan

Después del handoff:

```text
UnitOfWork
```

espera un outcome estructurado.

---

# 137. PersistencePlan

Será formalizado en:

```text
128_DATABASE_PERSISTENCE_PLANNER_SYSTEM.md
```

---

# 138. Persistence Engine

Será formalizado en:

```text
127_DATABASE_PERSISTENCE_ENGINE.md
```

---

# 139. Query generation boundary

Solo después de planificación:

```text
Persistence Operation
        ↓
Query Model / AST
```

---

# 140. No SQL

En ningún punto UnitOfWork deberá conocer:

```text
INSERT INTO
UPDATE
DELETE FROM
RETURNING
LAST_INSERT_ID()
```

---

# 141. Capability independence

UnitOfWork tampoco deberá preguntar directamente:

```php
if ($database === 'postgresql') { ... }
```

---

# 142. Capability requirements

Podrá expresar:

```text
requires generated identifier retrieval
requires deferred constraint capability
```

pero la resolución pertenece a Planner/Platform.

---

# 143. Execution outcome

Después de persistencia:

```text
PersistenceOutcome
```

deberá informar efectos conocidos.

---

# 144. Outcome dimensions

Ejemplo:

```text
operation status
outcome certainty
generated identifiers
affected entities
confirmed inserts
confirmed updates
confirmed deletes
failed operations
unknown operations
```

---

# 145. Success

En éxito conocido:

```text
reconcile snapshots
clear scheduled work
establish generated identities
update entity states
remove confirmed deleted entities where required
```

---

# 146. Failure before effects

Si:

```text
execution failed before any persistence effect
```

el UnitOfWork puede permanecer pendiente para retry si la política lo permite.

---

# 147. Partial execution

Caso:

```text
INSERT A succeeded
INSERT B succeeded
UPDATE C failed
```

sin transaction rollback garantizado.

No puede fingirse:

```text
nothing happened
```

---

# 148. Partial outcome

Debe preservarse:

```text
PARTIALLY_APPLIED
```

---

# 149. UNKNOWN outcome

Caso:

```text
UPDATE sent
connection lost
```

debe conservar:

```text
UNKNOWN
```

---

# 150. UNKNOWN ≠ FAILED

Un failure conocido:

```text
operation definitely did not apply
```

es distinto de:

```text
we do not know whether it applied
```

---

# 151. UnitOfWork taint

Si la reconciliación segura no es posible:

```text
UnitOfWork → TAINTED
EntityManager → TAINTED
```

---

# 152. No blind retry

Nunca:

```text
UNKNOWN
→ retry flush automatically
```

sin demostrar:

```text
replayability
idempotency
outcome reconciliation
```

---

# 153. Retry responsibility

Las políticas de retry pertenecen principalmente a:

```text
Persistence Engine
Transaction System
Execution Retry System
```

UnitOfWork preserva la información necesaria.

---

# 154. Transaction rollback

Si todo el persistence plan se ejecutó dentro de una transaction y existe rollback confirmado:

```text
database effects may be known reverted
```

pero:

```text
in-memory entity state
```

todavía requiere reconciliación.

---

# 155. Rollback ≠ rewind object graph

La DB puede revertir:

```text
UPDATE users
```

pero no revertirá automáticamente:

```php
$user->rename('Alice');
```

en memoria.

---

# 156. UnitOfWork snapshot after rollback

Podrá requerirse conservar/restaurar:

```text
original snapshot
pending ChangeSet
generated identifier semantics
entity state
```

según el caso.

---

# 157. Transaction rollback policy

No deberá implementarse como:

```text
rollback DB
→ clear all UoW state
```

automáticamente.

---

# 158. Generated identifiers after rollback

Como estableció Identity Map:

```text
generated ID assigned
+
transaction rollback
```

no implica:

```text
identifier becomes null
```

---

# 159. Transaction boundaries

UnitOfWork no deberá asumir:

```text
one flush = one transaction
```

---

# 160. Auto-transaction policy

EntityManager/Persistence Engine puede ofrecer:

```text
auto transaction around flush
```

como política de conveniencia.

Pero arquitectónicamente seguirá siendo:

```text
UnitOfWork
≠
Transaction
```

---

# 161. Multiple flush cycles

Un mismo UnitOfWork puede experimentar:

```text
Flush Cycle 1
Flush Cycle 2
Flush Cycle 3
```

durante su lifetime.

---

# 162. FlushCycleId

Cada preparación deberá tener:

```text
FlushCycleId
```

distinto.

---

# 163. UnitOfWorkId

El UnitOfWork completo podrá tener:

```text
UnitOfWorkId
```

para telemetry/diagnostics.

---

# 164. FlushCycleId ≠ UnitOfWorkId

```text
one UnitOfWork
→ many FlushCycles
```

---

# 165. New changes after successful flush

Después de reconciliar:

```text
entity modified again
```

deberá producir un nuevo ChangeSet en el siguiente flush.

---

# 166. Snapshot advancement

Después de éxito confirmado:

```text
new snapshot
=
successfully synchronized state
```

---

# 167. Snapshot must not advance on unknown

Nunca:

```text
UNKNOWN outcome
→ snapshot = current entity
```

porque ocultaría incertidumbre.

---

# 168. Snapshot must not advance on failure

Un UPDATE fallido no deberá marcar cambios como sincronizados.

---

# 169. Selective reconciliation

En partial execution:

```text
successful operation
failed operation
unknown operation
```

podrían requerir tratamientos diferentes.

---

# 170. Atomic transaction simplification

Si existe:

```text
confirmed full transaction rollback
```

puede simplificarse la reconciliación de efectos DB.

Pero la arquitectura no dependerá de que todos los motores/operaciones sean transaccionales.

---

# 171. Database capabilities

Ejemplos:

```text
transactional DDL
RETURNING
deferrable constraints
savepoints
```

pueden afectar Persistence Planner, no el núcleo del UnitOfWork.

---

# 172. Bulk operations

Una Entity Query bulk update:

```text
UPDATE all active users...
```

puede modificar DB sin pasar por entidades individuales.

---

# 173. Bulk bypass problem

Entonces UnitOfWork puede contener:

```text
User#10 managed with stale state
```

---

# 174. Managed-state policy

Como estableció ORM Architecture:

```text
CLEAR_AFFECTED
REFRESH_AFFECTED
REJECT_IF_MANAGED
ALLOW_STALE_EXPLICITLY
```

---

# 175. Bulk operations ≠ UnitOfWork entity changes

No deberán simularse millones de ChangeSets solo para representar un bulk query.

---

# 176. External DB mutations

Cambios ejecutados por:

```text
raw SQL
another process
another service
database trigger
```

pueden invalidar assumptions del UnitOfWork.

---

# 177. UnitOfWork cannot detect arbitrary external mutation

No deberá fingir que puede.

---

# 178. Refresh/Clear

La aplicación podrá requerir:

```text
refresh
clear
detach
explicit invalidation
```

---

# 179. Optimistic locking

Un ChangeSet puede incluir:

```text
expected version
```

para el Persistence Planner.

---

# 180. Version token

UnitOfWork podrá transportar:

```text
OriginalVersion
CurrentVersion
```

pero no ejecutará el lock.

---

# 181. Zero affected rows

Un UPDATE optimistically locked con:

```text
affected rows = 0
```

puede representar:

```text
OptimisticLockConflict
```

El Persistence Engine deberá traducirlo semánticamente.

---

# 182. Pessimistic locking

No pertenece al UnitOfWork.

Pertenece a:

```text
Transaction
Entity Query
Concurrency Control
```

---

# 183. Read-only entities

Un EntityManager puede registrar entidades como:

```text
READ_ONLY
```

---

# 184. Read-only tracking

Podrá:

```text
skip snapshots
skip dirty checking
reject persistence mutation
```

según política.

---

# 185. Read-only ≠ detached

Una entidad read-only puede seguir:

```text
IdentityMap managed
```

para identidad canónica.

---

# 186. Immutable entities

Entidades immutable/readonly requieren estrategia explícita.

---

# 187. Replacement problem

Si:

```php
$user2 = $user->renamed('Alice');
```

crea una nueva instancia con el mismo ID:

```text
IdentityMap conflict
```

si `$user` sigue managed.

---

# 188. Immutable entity strategy

No deberá pretenderse que immutable entities funcionan igual que mutable entities sin contrato adicional.

---

# 189. Possible future strategies

```text
MANAGED_REPLACEMENT
EXPLICIT_REATTACH
STATE_OBJECT_MUTATION
CUSTOM
```

pero no serán comportamiento mágico de V1.

---

# 190. Value Objects

Un cambio dentro de un Value Object embebido puede representar un field change de la entidad.

---

# 191. Value Object identity

Value Objects no entran al UnitOfWork como entities salvo que sean entities reales.

---

# 192. Embedded objects

```text
User
└── Address ValueObject
```

se rastrea como parte del estado persistente de User.

---

# 193. Entity relationship vs embedded object

UnitOfWork deberá usar metadata para distinguirlos.

---

# 194. Domain events

UnitOfWork no deberá convertirse en Domain Event Bus.

---

# 195. Persistence lifecycle events

Puede emitir eventos como:

```text
UnitOfWorkPreparing
UnitOfWorkPrepared
FlushStarted
FlushSucceeded
FlushFailed
FlushUnknown
```

mediante integración con Event System.

---

# 196. Event observer ≠ semantic authority

Listeners no deberán modificar arbitrariamente el UnitOfWork frozen.

---

# 197. Pre-flush extension point

Si se permite mutación:

```text
before stabilization/freeze
```

deberá ocurrir en una fase explícita.

---

# 198. Post-freeze mutation

Deberá rechazarse o programarse para un ciclo posterior.

---

# 199. Reentrant flush

```php
preUpdate(function () use ($em) {
    $em->flush();
});
```

deberá rechazarse.

---

# 200. Exception

```text
ReentrantFlushException
```

---

# 201. Nested UnitOfWork

Un mismo EntityManager no deberá crear UnitOfWorks anidados para simular nested transactions.

---

# 202. Nested transaction ≠ nested UnitOfWork

```text
Transaction savepoint
```

no implica:

```text
new UnitOfWork
```

---

# 203. Multiple EntityManagers

Dos EntityManagers pueden tener:

```text
UnitOfWork A
UnitOfWork B
```

incluso sobre la misma DB.

---

# 204. No automatic cross-manager coherence

Si ambos cargan `User#10`:

```text
EM A → object A
EM B → object B
```

esto es válido porque:

```text
PersistenceContext A
≠
PersistenceContext B
```

---

# 205. Cross-manager entity passing

Pasar una managed entity de A a B deberá ser explícitamente detectado/rechazado.

---

# 206. No global canonical entity

Identity Map garantiza unicidad:

```text
within one PersistenceContext
```

no en todo el proceso.

---

# 207. Persistent runtime

FrankenPHP:

```text
Worker
│
├── Shared immutable metadata
│
├── Request A
│   └── UnitOfWork A
│
├── reset
│
└── Request B
    └── UnitOfWork B
```

---

# 208. Mutable state never worker-global

Nunca compartir:

```text
scheduled insertions
ChangeSets
snapshots
orphan candidates
temporary entity tokens
flush state
```

entre requests.

---

# 209. Runtime reset

Al finalizar request/job:

```text
UnitOfWork
→ clear/dispose
```

---

# 210. No implicit flush on shutdown

Regla crítica:

> VoltStack nunca deberá ejecutar `flush()` automáticamente al terminar una request únicamente porque existan cambios pendientes.

---

# 211. Why

Un destructor/shutdown hook no es un boundary seguro para:

```text
database writes
transactions
error handling
authorization
telemetry
```

---

# 212. Pending work at scope end

Podrá generar diagnostic:

```text
UnitOfWorkDisposedWithPendingChanges
```

en modo desarrollo.

---

# 213. Production policy

Podrá:

```text
discard pending in-memory work
+
emit telemetry
```

pero no persistirlo silenciosamente.

---

# 214. Queue workers

En long-running jobs:

```text
flush
clear
```

deberá hacerse en boundaries explícitos.

---

# 215. Long UnitOfWork

Problemas:

```text
memory growth
expensive dirty checking
huge ChangeSets
large dependency graphs
long transactions
stale entity state
```

---

# 216. Resource metrics

Se deberán medir:

```text
tracked entity count
managed entity count
new entity count
removed entity count
dirty entity count
relationship change count
collection change count
orphan count
graph node count
graph edge count
snapshot bytes estimate
flush preparation time
stabilization passes
```

---

# 217. Telemetry

Ejemplos:

```text
orm.uow.entities.tracked
orm.uow.entities.new
orm.uow.entities.dirty
orm.uow.entities.removed
orm.uow.relationship_changes
orm.uow.collection_changes
orm.uow.orphans
orm.uow.prepare.duration
orm.uow.stabilization.passes
orm.uow.flush.duration
orm.uow.flush.outcome
orm.uow.graph.nodes
orm.uow.graph.edges
```

---

# 218. Telemetry ≠ semantics

Desactivar telemetry no deberá cambiar el resultado del UnitOfWork.

---

# 219. Debug Toolbar

Podrá mostrar:

```text
UnitOfWork

State: TRACKING

Tracked Entities:      154
New:                    12
Dirty:                   8
Removed:                 2

Relationship Changes:   15
Collection Changes:      6
Orphan Candidates:       1

Last Flush:
  Preparation: 2.4 ms
  Execution:   5.8 ms
  Outcome:     SUCCESS
```

---

# 220. Explainability

En desarrollo deberá poder explicarse:

```text
Why is this entity scheduled?
```

Ejemplo:

```text
OrderItem@object#71
State: NEW
Scheduled: INSERT
Reason:
  discovered by cascade persist
  from Order#100
  relationship: Order.items
```

---

# 221. Scheduling provenance

Se propone:

```text
SchedulingReason
```

---

# 222. Reason types

```text
EXPLICIT_PERSIST
CASCADE_PERSIST
EXPLICIT_REMOVE
CASCADE_REMOVE
ORPHAN_REMOVAL
CHANGE_DETECTED
RELATIONSHIP_CHANGE
COLLECTION_CHANGE
GENERATED_DEPENDENCY
EXTENSION
```

---

# 223. Provenance chain

Ejemplo:

```text
OrderItem NEW
← cascade persist
← Order.items
← explicit persist(Order)
```

---

# 224. Diagnostics value

Esto facilita depurar:

```text
Why did ORM try to insert this entity?
Why is this object being deleted?
Why did flush touch 10,000 rows?
```

---

# 225. Security

UnitOfWork no implementa Authorization.

---

# 226. persist() ≠ authorized write

Que una entidad sea scheduled no significa:

```text
user is authorized
```

---

# 227. Authorization boundary

Authorization deberá ocurrir antes o alrededor de operaciones de aplicación apropiadas.

---

# 228. Mass assignment

Tampoco pertenece al UnitOfWork.

---

# 229. Sensitive data

ChangeSets pueden contener:

```text
password hashes
tokens
personal data
financial values
```

---

# 230. Diagnostics redaction

Telemetry/logs/debug output deberán respetar:

```text
SensitiveDataPolicy
```

---

# 231. No raw ChangeSet logging by default

Nunca registrar automáticamente todos los valores `old/new` en producción.

---

# 232. Extensions

UnitOfWork deberá ser extensible mediante contratos tipados.

---

# 233. Extension points posibles

```text
ChangeTrackingStrategy
CascadePolicy
RelationshipChangeAnalyzer
CollectionChangeAnalyzer
UnitOfWorkValidator
ChangeGraphContributor
FlushPreparationListener
SchedulingDiagnosticContributor
```

---

# 234. Extension freeze

Los registries de extensiones deberán quedar:

```text
frozen
```

durante runtime normal.

---

# 235. No runtime last-wins

Dos extensiones que reclamen la misma estrategia no deberán resolverse mediante orden accidental.

---

# 236. Deterministic extension priority

Si existe prioridad:

```text
priority
+
stable registration identity
```

deberá ser explícita.

---

# 237. Extension cannot execute SQL

Un:

```text
ChangeGraphContributor
```

no deberá abrir conexiones ni ejecutar SQL como efecto lateral oculto.

---

# 238. Extension cannot hide uncertainty

Una extensión no podrá convertir:

```text
UNKNOWN
```

en:

```text
SUCCESS
```

sin evidencia semántica.

---

# 239. Testing architecture

Se requieren:

```text
Unit Tests
State Integration Tests
IdentityMap Integration Tests
Snapshot Tests
Change Tracking Tests
Relationship Tests
Cascade Tests
Orphan Tests
Persistence Planner Tests
Transaction Tests
Failure Tests
Runtime Isolation Tests
Resource Governance Tests
Determinism Tests
Extension Conformance Tests
```

---

# 240. Test: persist NEW

```text
NEW
→ persist()
→ scheduled insertion
```

sin query inmediata.

---

# 241. Test: persist idempotency

```php
$em->persist($user);
$em->persist($user);
```

no deberá duplicar scheduling.

---

# 242. Test: persist managed

No deberá crear un segundo UnitOfWork entry.

---

# 243. Test: persist detached

Deberá fallar según política V1.

---

# 244. Test: remove managed

Deberá programar removal sin DELETE inmediato.

---

# 245. Test: remove NEW

Deberá cancelar o normalizar insert pending según semántica.

---

# 246. Test: identity map coordination

Una managed entity deberá corresponder a la instancia canónica del IdentityMap.

---

# 247. Test: NEW without ID

Deberá poder entrar al graph mediante TemporaryEntityToken.

---

# 248. Test: generated ID dependency

```text
Parent NEW
Child NEW
```

deberá expresar dependencia cuando Child requiere parent generated ID.

---

# 249. Test: cascade persist

El graph deberá descubrir todos los nodes alcanzables permitidos.

---

# 250. Test: cascade cycle

```text
A → B → A
```

no deberá provocar recursión infinita.

---

# 251. Test: graph fixed point

Discovery deberá converger.

---

# 252. Test: non-convergent lifecycle

Deberá producir error explícito.

---

# 253. Test: orphan removed

Un orphan confirmado deberá quedar scheduled.

---

# 254. Test: orphan reattached

Si se reanexa antes de freeze, no deberá eliminarse.

---

# 255. Test: owning side

Solo cambios persistentes válidos deberán formar intents.

---

# 256. Test: inverse-side inconsistency

Deberá diagnosticarse si la política lo requiere.

---

# 257. Test: no-op entity

Entidad managed sin cambios no deberá generar UPDATE.

---

# 258. Test: same-value assignment

```text
name: Alice → Alice
```

no deberá producir ChangeSet semántico salvo custom type semantics.

---

# 259. Test: value object change

Cambio estructural en ValueObject deberá detectarse correctamente.

---

# 260. Test: collection add/remove

Debe producir changes tipados.

---

# 261. Test: collection reorder

Solo deberá persistirse cuando mapping declare orden persistente.

---

# 262. Test: prepare freeze

Cambios posteriores no deberán alterar `PreparedUnitOfWork`.

---

# 263. Test: reentrant flush

Deberá fallar.

---

# 264. Test: success reconciliation

Después de éxito:

```text
pending work cleared
snapshots advanced
managed state synchronized
```

---

# 265. Test: failure

No deberá marcar como sincronizado trabajo no ejecutado.

---

# 266. Test: partial execution

Debe preservar qué operaciones sí y cuáles no.

---

# 267. Test: unknown outcome

Debe taint cuando no pueda reconciliarse.

---

# 268. Test: rollback

DB rollback no deberá fingir que el object graph se revirtió.

---

# 269. Test: generated ID rollback

No deberá nulificar ID arbitrariamente.

---

# 270. Test: multiple flush cycles

Cada ciclo deberá tener ID/fingerprint independiente.

---

# 271. Test: bulk mutation policy

Managed state deberá tratarse según política explícita.

---

# 272. Test: resource limit

No deberá truncar silenciosamente graph.

---

# 273. Test: pending scope disposal

Nunca deberá ejecutar implicit flush.

---

# 274. Test: FrankenPHP

Request B no deberá observar:

```text
scheduled changes
temporary tokens
ChangeSets
```

de Request A.

---

# 275. Test: tenant switching

No deberá reutilizar UnitOfWork con otro identity namespace.

---

# 276. Test: two EntityManagers

Deberán mantener UnitOfWorks independientes.

---

# 277. Test: deterministic graph

Mismos inputs semánticos deberán producir mismo graph/fingerprint.

---

# 278. Test: extension collision

Deberá producir error determinista.

---

# 279. Exception hierarchy

```text
DatabaseOrmException
└── UnitOfWorkException
    ├── UnitOfWorkClosedException
    ├── UnitOfWorkTaintedException
    ├── UnitOfWorkStateException
    ├── UnitOfWorkRegistrationException
    ├── UnitOfWorkEntityConflictException
    ├── DetachedEntityPassedToPersistException
    ├── DetachedEntityPassedToRemoveException
    ├── CrossContextEntityException
    ├── EntitySchedulingException
    ├── EntityDependencyException
    ├── EntityDependencyCycleException
    ├── CascadeDiscoveryException
    ├── CascadeDepthExceededException
    ├── OrphanRemovalException
    ├── RelationshipConsistencyException
    ├── UnitOfWorkStabilizationException
    ├── ReentrantFlushException
    ├── UnitOfWorkFrozenException
    ├── StalePreparedUnitOfWorkException
    ├── UnitOfWorkReconciliationException
    ├── UnitOfWorkUnknownOutcomeException
    ├── UnitOfWorkResourceLimitException
    ├── UnitOfWorkRuntimeIsolationException
    ├── UnitOfWorkConcurrencyException
    └── UnitOfWorkInvariantException
```

---

# 280. Diagnostic example

```text
DB-ORM-UOW-CASCADE-001

Entity:
App\Entity\Order

Tracked Entities:
10,001

Configured Maximum:
10,000

Discovery Path:
Order
→ items
→ product
→ relatedProducts
→ ...

Reason:
Cascade discovery exceeded the configured UnitOfWork graph limit.

Action:
Review cascade mappings or persist the graph in smaller explicit batches.
```

---

# 281. Unknown outcome diagnostic

```text
DB-ORM-UOW-OUTCOME-002

Flush Cycle:
01J...

Entity:
App\Entity\User#10

Operation:
UPDATE

Execution Outcome:
UNKNOWN

Reason:
The database connection was lost after the operation was submitted,
and VoltStack cannot prove whether the update was durably applied.

UnitOfWork:
TAINTED

Action:
Do not blindly retry the flush. Reconcile database state or restart
the persistence context according to the configured recovery policy.
```

---

# 282. Reentrant diagnostic

```text
DB-ORM-UOW-FLUSH-003

Reason:
EntityManager::flush() was invoked while another flush cycle was
already preparing or executing.

Current State:
FLUSHING

Action:
Move nested persistence work to the current pre-flush phase or defer
it to a subsequent explicit flush cycle.
```

---

# 283. Arquitectura de directorios

```text
src/Quantum/Database/ORM/
│
├── UnitOfWork/
│   ├── Contract/
│   │   ├── UnitOfWork.php
│   │   ├── UnitOfWorkFactory.php
│   │   ├── UnitOfWorkValidator.php
│   │   └── UnitOfWorkReconciler.php
│   │
│   ├── Core/
│   │   ├── DefaultUnitOfWork.php
│   │   ├── UnitOfWorkId.php
│   │   ├── FlushCycleId.php
│   │   ├── UnitOfWorkState.php
│   │   ├── UnitOfWorkEntry.php
│   │   └── PreparedUnitOfWork.php
│   │
│   ├── Entity/
│   │   ├── EntityHandle.php
│   │   ├── PersistentEntityHandle.php
│   │   ├── TemporaryEntityHandle.php
│   │   ├── TemporaryEntityToken.php
│   │   └── EntityRegistration.php
│   │
│   ├── Scheduling/
│   │   ├── ScheduledEntityRegistry.php
│   │   ├── PersistenceIntent.php
│   │   ├── SchedulingReason.php
│   │   ├── SchedulingProvenance.php
│   │   └── SchedulingNormalizer.php
│   │
│   ├── Cascade/
│   │   ├── CascadeDiscovery.php
│   │   ├── CascadeTraversal.php
│   │   ├── CascadePolicy.php
│   │   └── CascadeVisitedSet.php
│   │
│   ├── Relationship/
│   │   ├── RelationshipChangeRegistry.php
│   │   ├── RelationshipChangeAnalyzer.php
│   │   └── RelationshipConsistencyValidator.php
│   │
│   ├── Collection/
│   │   ├── CollectionChangeRegistry.php
│   │   ├── CollectionChangeAnalyzer.php
│   │   └── CollectionOrderingChange.php
│   │
│   ├── Orphan/
│   │   ├── OrphanCandidate.php
│   │   ├── OrphanRegistry.php
│   │   └── OrphanRemovalResolver.php
│   │
│   ├── Graph/
│   │   ├── EntityChangeGraph.php
│   │   ├── EntityChangeNode.php
│   │   ├── EntityChangeEdge.php
│   │   ├── EntityDependencyType.php
│   │   ├── EntityChangeGraphBuilder.php
│   │   └── EntityChangeGraphValidator.php
│   │
│   ├── Flush/
│   │   ├── FlushPreparationPipeline.php
│   │   ├── UnitOfWorkStabilizer.php
│   │   ├── UnitOfWorkFreezer.php
│   │   ├── UnitOfWorkFingerprint.php
│   │   └── FlushMutationGuard.php
│   │
│   ├── Reconciliation/
│   │   ├── UnitOfWorkReconciler.php
│   │   ├── SuccessfulPersistenceReconciler.php
│   │   ├── FailedPersistenceReconciler.php
│   │   ├── PartialPersistenceReconciler.php
│   │   └── UnknownPersistenceReconciler.php
│   │
│   ├── Runtime/
│   │   ├── UnitOfWorkScope.php
│   │   ├── UnitOfWorkOwnershipGuard.php
│   │   └── UnitOfWorkRuntimeResetter.php
│   │
│   ├── Governance/
│   │   ├── UnitOfWorkResourcePolicy.php
│   │   ├── UnitOfWorkLimits.php
│   │   └── UnitOfWorkBudget.php
│   │
│   ├── Telemetry/
│   │   ├── UnitOfWorkTelemetry.php
│   │   └── UnitOfWorkProfiler.php
│   │
│   └── Exception/
│       └── ...
```

---

# 284. Dependency architecture

Permitido:

```text
UnitOfWork
   ├── Entity Model
   ├── Entity Metadata
   ├── Entity State
   ├── Identity Map
   ├── Change Tracking
   ├── Snapshot System
   └── Relationship Metadata
```

---

# 285. Downstream dependency

```text
Persistence Engine
        ↓
PreparedUnitOfWork
```

No:

```text
UnitOfWork
→ Persistence Engine internals
```

cuando pueda evitarse mediante contratos.

---

# 286. Dependencias prohibidas

```text
UnitOfWork → PDO
UnitOfWork → Driver
UnitOfWork → SQL Compiler
UnitOfWork → concrete MySQL logic
UnitOfWork → concrete PostgreSQL logic
UnitOfWork → HTTP Request globals
UnitOfWork → mutable global Tenant
```

---

# 287. Arquitectura de flush completa

```text
Application
    │
    ├── persist()
    ├── remove()
    ├── mutate entities
    └── mutate relationships
            │
            ▼
        UnitOfWork
            │
            ▼
     Cascade Discovery
            │
            ▼
      Change Tracking
            │
            ▼
    Relationship Analysis
            │
            ▼
      Orphan Analysis
            │
            ▼
    Lifecycle Stabilization
            │
            ▼
     Entity Change Graph
            │
            ▼
    PreparedUnitOfWork
            │
            ▼
     Persistence Planner
            │
            ▼
      Persistence Plan
            │
            ▼
     Persistence Engine
            │
            ▼
       Query Model/AST
            │
            ▼
      Query Architecture
            │
            ▼
      Execution Engine
            │
            ▼
         Database
            │
            ▼
    Persistence Outcome
            │
            ▼
       Reconciliation
            │
            ├── Entity State
            ├── Identity Map
            ├── Snapshots
            └── UnitOfWork
```

---

# 288. Arquitectura de responsabilidades

```text
Entity
    │
    └── domain state

Entity State System
    │
    └── lifecycle/persistence state

Identity Map
    │
    └── canonical instance

Snapshot System
    │
    └── original synchronized state

Change Tracking
    │
    └── differences

UnitOfWork
    │
    └── coordinated pending work

Persistence Planner
    │
    └── persistence strategy/order

Persistence Engine
    │
    └── synchronization execution

Transaction Manager
    │
    └── atomicity/isolation

Query Engine
    │
    └── query representation/planning

SQL Compiler
    │
    └── SQL

Execution Engine
    │
    └── physical execution
```

---

# 289. Invariantes arquitectónicas

## DB-ORM-UOW-001
Cada UnitOfWork pertenecerá a un único PersistenceContext.

## DB-ORM-UOW-002
UnitOfWork no será process-global.

## DB-ORM-UOW-003
UnitOfWork será distinto de Transaction.

## DB-ORM-UOW-004
UnitOfWork será distinto de IdentityMap.

## DB-ORM-UOW-005
UnitOfWork será distinto de Entity State.

## DB-ORM-UOW-006
UnitOfWork será distinto de Change Tracking.

## DB-ORM-UOW-007
UnitOfWork será distinto de Snapshot System.

## DB-ORM-UOW-008
UnitOfWork será distinto de Persistence Engine.

## DB-ORM-UOW-009
UnitOfWork será distinto de Persistence Planner.

## DB-ORM-UOW-010
UnitOfWork no generará SQL.

## DB-ORM-UOW-011
UnitOfWork no ejecutará queries.

## DB-ORM-UOW-012
UnitOfWork no dependerá de PDO.

## DB-ORM-UOW-013
UnitOfWork no dependerá de un driver concreto.

## DB-ORM-UOW-014
persist() no ejecutará INSERT inmediatamente.

## DB-ORM-UOW-015
remove() no ejecutará DELETE inmediatamente.

## DB-ORM-UOW-016
flush() será distinto de commit().

## DB-ORM-UOW-017
Un transaction podrá contener múltiples flushes.

## DB-ORM-UOW-018
Un flush no implicará necesariamente una transaction propia.

## DB-ORM-UOW-019
persist() sobre MANAGED será idempotente.

## DB-ORM-UOW-020
persist() sobre DETACHED será rechazado por defecto en V1.

## DB-ORM-UOW-021
remove() sobre DETACHED será rechazado por defecto.

## DB-ORM-UOW-022
UNKNOWN state no se convertirá implícitamente en MANAGED.

## DB-ORM-UOW-023
ScheduledInsertion será distinto de SQL INSERT.

## DB-ORM-UOW-024
ScheduledUpdate será distinto de SQL UPDATE.

## DB-ORM-UOW-025
ScheduledRemoval será distinto de SQL DELETE.

## DB-ORM-UOW-026
Scheduling no determinará necesariamente physical statement count.

## DB-ORM-UOW-027
Hydration podrá registrar managed entities en UnitOfWork.

## DB-ORM-UOW-028
Registration se coordinará con IdentityMap.

## DB-ORM-UOW-029
Registration se coordinará con EntityStateRegistry.

## DB-ORM-UOW-030
Registration se coordinará con SnapshotRegistry cuando aplique.

## DB-ORM-UOW-031
Los registries no deberán quedar semánticamente contradictorios.

## DB-ORM-UOW-032
Object identity podrá utilizarse internamente para tracking.

## DB-ORM-UOW-033
Object identity será distinta de persistent entity identity.

## DB-ORM-UOW-034
NEW entities sin ID deberán poder rastrearse.

## DB-ORM-UOW-035
TemporaryEntityToken será distinto de EntityKey.

## DB-ORM-UOW-036
TemporaryEntityToken no será expuesto como persistent ID.

## DB-ORM-UOW-037
TemporaryEntityToken no será durable identity.

## DB-ORM-UOW-038
ChangeSet será distinto de SQL.

## DB-ORM-UOW-039
Entidad managed sin cambios no requerirá UPDATE.

## DB-ORM-UOW-040
Dirty checking será delegable a Change Tracking.

## DB-ORM-UOW-041
Snapshots pertenecerán al Snapshot System.

## DB-ORM-UOW-042
UnitOfWork consumirá snapshots sin apropiarse de su semántica.

## DB-ORM-UOW-043
Relationship changes serán first-class.

## DB-ORM-UOW-044
Collection changes serán first-class.

## DB-ORM-UOW-045
Collection order solo será persistente cuando mapping lo declare.

## DB-ORM-UOW-046
Relationship ownership será metadata-driven.

## DB-ORM-UOW-047
Inverse-side mutation no implicará automáticamente owning-side persistence.

## DB-ORM-UOW-048
Relationship inconsistencies podrán diagnosticarse.

## DB-ORM-UOW-049
Cascade persist será distinto de DB cascade.

## DB-ORM-UOW-050
Cascade remove será distinto de DB cascade.

## DB-ORM-UOW-051
Cascade discovery deberá manejar ciclos.

## DB-ORM-UOW-052
Cascade discovery deberá mantener visited set.

## DB-ORM-UOW-053
Cascade discovery deberá alcanzar fixed point.

## DB-ORM-UOW-054
Cascade discovery estará sujeta a resource limits.

## DB-ORM-UOW-055
Resource limit nunca truncará silenciosamente el graph.

## DB-ORM-UOW-056
Orphan removal será metadata-driven.

## DB-ORM-UOW-057
Collection removal no implicará automáticamente entity deletion.

## DB-ORM-UOW-058
Orphan candidate será distinto de confirmed orphan.

## DB-ORM-UOW-059
Orphan analysis deberá estabilizarse antes de freeze.

## DB-ORM-UOW-060
EntityChangeGraph será first-class.

## DB-ORM-UOW-061
EntityChangeGraph nodes usarán entity handles.

## DB-ORM-UOW-062
EntityChangeGraph podrá contener temporary handles.

## DB-ORM-UOW-063
EntityChangeGraph edges expresarán dependencias semánticas.

## DB-ORM-UOW-064
EntityChangeGraph será distinto de physical execution order.

## DB-ORM-UOW-065
Generated-ID dependency será explícita.

## DB-ORM-UOW-066
Dependency cycles no se resolverán por orden arbitrario.

## DB-ORM-UOW-067
Persistence Planner resolverá physical cycle strategy.

## DB-ORM-UOW-068
PersistenceIntent será distinto de statement.

## DB-ORM-UOW-069
PreparedUnitOfWork será immutable.

## DB-ORM-UOW-070
Mutable UnitOfWork será distinto de PreparedUnitOfWork.

## DB-ORM-UOW-071
PreparedUnitOfWork representará un flush cycle específico.

## DB-ORM-UOW-072
Mutaciones posteriores no modificarán PreparedUnitOfWork.

## DB-ORM-UOW-073
Reentrant flush será rechazado.

## DB-ORM-UOW-074
Lifecycle hooks mutables ocurrirán antes del freeze.

## DB-ORM-UOW-075
Lifecycle mutation deberá estabilizarse.

## DB-ORM-UOW-076
Non-convergent lifecycle mutation será error.

## DB-ORM-UOW-077
Stabilization tendrá límites explícitos.

## DB-ORM-UOW-078
postPersist no significará transaction committed.

## DB-ORM-UOW-079
prepareFlush no realizará hidden DB queries.

## DB-ORM-UOW-080
Database validations necesarias serán explícitas.

## DB-ORM-UOW-081
PreparedUnitOfWork podrá tener deterministic fingerprint.

## DB-ORM-UOW-082
Semantic fingerprint no dependerá de memory address.

## DB-ORM-UOW-083
Semantic fingerprint no dependerá de random iteration order.

## DB-ORM-UOW-084
Mismos inputs lógicos producirán mismo prepared semantics.

## DB-ORM-UOW-085
UnitOfWork entregará prepared state al Persistence Planner.

## DB-ORM-UOW-086
UnitOfWork no ejecutará PersistencePlan.

## DB-ORM-UOW-087
UnitOfWork no conocerá SQL syntax.

## DB-ORM-UOW-088
UnitOfWork no contendrá vendor conditionals.

## DB-ORM-UOW-089
Capability requirements podrán expresarse abstractamente.

## DB-ORM-UOW-090
Execution outcome deberá ser estructurado.

## DB-ORM-UOW-091
Successful outcome permitirá reconciliación positiva.

## DB-ORM-UOW-092
Failed operation no será marcada synchronized.

## DB-ORM-UOW-093
UNKNOWN operation no será marcada synchronized.

## DB-ORM-UOW-094
PARTIAL outcome será first-class.

## DB-ORM-UOW-095
UNKNOWN será distinto de FAILED.

## DB-ORM-UOW-096
Unknown unreconcilable outcome podrá taint UnitOfWork.

## DB-ORM-UOW-097
Unknown unreconcilable outcome podrá taint EntityManager.

## DB-ORM-UOW-098
UnitOfWork no realizará blind retry.

## DB-ORM-UOW-099
Retry requerirá semántica de replayability/outcome certainty.

## DB-ORM-UOW-100
Transaction rollback no rebobinará object graph automáticamente.

## DB-ORM-UOW-101
Transaction rollback no implicará identifier reset.

## DB-ORM-UOW-102
Rollback reconciliation será explícita.

## DB-ORM-UOW-103
UnitOfWork no asumirá transactional execution universal.

## DB-ORM-UOW-104
One flush no será necesariamente one transaction.

## DB-ORM-UOW-105
UnitOfWork podrá sobrevivir múltiples flush cycles.

## DB-ORM-UOW-106
Cada flush cycle tendrá identidad propia.

## DB-ORM-UOW-107
Successful synchronization avanzará snapshots apropiados.

## DB-ORM-UOW-108
Unknown synchronization no avanzará snapshots como success.

## DB-ORM-UOW-109
Failed synchronization no avanzará snapshots como success.

## DB-ORM-UOW-110
Partial outcome podrá requerir selective reconciliation.

## DB-ORM-UOW-111
Bulk query no se convertirá artificialmente en millones de entity ChangeSets.

## DB-ORM-UOW-112
Bulk operations deberán declarar managed-state policy.

## DB-ORM-UOW-113
External DB mutation podrá invalidar managed assumptions.

## DB-ORM-UOW-114
UnitOfWork no fingirá detectar arbitrary external mutations.

## DB-ORM-UOW-115
Optimistic version data podrá formar parte del persistence input.

## DB-ORM-UOW-116
Pessimistic locking no pertenecerá al UnitOfWork.

## DB-ORM-UOW-117
Read-only managed entities podrán omitir dirty tracking según política.

## DB-ORM-UOW-118
Read-only será distinto de detached.

## DB-ORM-UOW-119
Immutable entities requerirán estrategia explícita.

## DB-ORM-UOW-120
Immutable replacement no reemplazará silenciosamente canonical managed instance.

## DB-ORM-UOW-121
Value Objects no serán tracked como entities por defecto.

## DB-ORM-UOW-122
Embedded Value Objects formarán parte del owning entity state.

## DB-ORM-UOW-123
UnitOfWork no será Domain Event Bus.

## DB-ORM-UOW-124
UnitOfWork events no cambiarán semántica cuando telemetry/events estén deshabilitados.

## DB-ORM-UOW-125
Frozen UnitOfWork no aceptará mutación arbitraria.

## DB-ORM-UOW-126
Nested transaction no creará nested UnitOfWork automáticamente.

## DB-ORM-UOW-127
Dos EntityManagers tendrán UnitOfWorks independientes.

## DB-ORM-UOW-128
No existirá canonical entity global entre EntityManagers.

## DB-ORM-UOW-129
Cross-manager entity usage será explícitamente gobernado.

## DB-ORM-UOW-130
Worker lifetime será distinto de UnitOfWork lifetime.

## DB-ORM-UOW-131
FrankenPHP requests no compartirán UnitOfWork mutable.

## DB-ORM-UOW-132
RoadRunner requests no compartirán UnitOfWork mutable.

## DB-ORM-UOW-133
OpenSwoole scopes no compartirán UnitOfWork mutable accidentalmente.

## DB-ORM-UOW-134
UnitOfWork mutable no será concurrency-safe por defecto.

## DB-ORM-UOW-135
Scheduled work nunca cruzará request boundaries.

## DB-ORM-UOW-136
ChangeSets nunca cruzarán request boundaries como mutable state.

## DB-ORM-UOW-137
Temporary entity tokens nunca cruzarán request boundaries.

## DB-ORM-UOW-138
Orphan candidates nunca serán worker-global.

## DB-ORM-UOW-139
Scope termination no ejecutará implicit flush.

## DB-ORM-UOW-140
Pending work al destruir scope podrá diagnosticarse.

## DB-ORM-UOW-141
Long-running jobs deberán usar explicit flush/clear boundaries.

## DB-ORM-UOW-142
Large UnitOfWork tendrá resource governance.

## DB-ORM-UOW-143
Resource governance no realizará silent semantic changes.

## DB-ORM-UOW-144
UnitOfWork telemetry será side-effect-free respecto a persistence semantics.

## DB-ORM-UOW-145
Sensitive ChangeSet values no se loguearán indiscriminadamente.

## DB-ORM-UOW-146
Scheduling provenance será diagnosticable.

## DB-ORM-UOW-147
Authorization será distinta de UnitOfWork scheduling.

## DB-ORM-UOW-148
persist() no implicará authorization.

## DB-ORM-UOW-149
Mass assignment no pertenecerá al UnitOfWork.

## DB-ORM-UOW-150
Extension points serán tipados.

## DB-ORM-UOW-151
Extension registries estarán frozen durante runtime normal.

## DB-ORM-UOW-152
Extension collisions serán deterministas.

## DB-ORM-UOW-153
UnitOfWork extensions no ejecutarán SQL ocultamente.

## DB-ORM-UOW-154
Extensions no ocultarán UNKNOWN outcome.

## DB-ORM-UOW-155
Tenant switching no reutilizará UnitOfWork con managed state incompatible.

## DB-ORM-UOW-156
Tenant contexts mantendrán UnitOfWork isolation.

## DB-ORM-UOW-157
Shard contexts mantendrán UnitOfWork isolation cuando corresponda.

## DB-ORM-UOW-158
UnitOfWork reset será determinista.

## DB-ORM-UOW-159
Cleared UnitOfWork no conservará pending persistence work.

## DB-ORM-UOW-160
UnitOfWork será la autoridad de coordinación de cambios pendientes, pero no la autoridad de ejecución física.

---

# 290. Anti-pattern: SQL inside UnitOfWork

Incorrecto:

```php
public function persist(object $entity): void
{
    $this->pdo->prepare(
        'INSERT INTO users ...'
    )->execute();
}
```

Correcto:

```text
persist()
→ register intent
→ flush preparation
→ Persistence Planner
→ Persistence Engine
→ Query Model
```

---

# 291. Anti-pattern: UnitOfWork as transaction

Incorrecto:

```php
$uow->begin();
$uow->commit();
$uow->rollback();
```

si esas operaciones pretenden reemplazar Transaction Manager.

---

# 292. Anti-pattern: persist means INSERT

Incorrecto:

```text
persist(entity)
→ database write
```

---

# 293. Anti-pattern: remove means DELETE

Incorrecto:

```text
remove(entity)
→ database delete
```

---

# 294. Anti-pattern: implicit flush

Incorrecto:

```php
public function __destruct()
{
    $this->flush();
}
```

Especialmente peligroso en persistent runtimes.

---

# 295. Anti-pattern: UnitOfWork global

Incorrecto:

```php
final class UnitOfWork
{
    private static array $pending = [];
}
```

---

# 296. Anti-pattern: snapshots inside entity

Incorrecto:

```php
class User
{
    private array $__ormOriginalState;
}
```

como requisito universal.

El ORM deberá poder mantener estado externamente.

---

# 297. Anti-pattern: SQL operation graph

Incorrecto:

```text
EntityChangeGraph
→ "INSERT users"
→ "UPDATE orders"
```

El graph debe permanecer semántico.

---

# 298. Anti-pattern: cascade everything

Incorrecto:

```text
all relationships cascade persist/remove
```

como default universal.

---

# 299. Anti-pattern: recursive cascade without visited set

Puede provocar:

```text
infinite recursion
stack overflow
duplicate scheduling
```

---

# 300. Anti-pattern: orphan = removed from array

Incorrecto:

```text
collection no longer contains entity
→ DELETE entity
```

sin metadata y estabilización.

---

# 301. Anti-pattern: lifecycle callback reentrant flush

Incorrecto:

```php
#[PreUpdate]
public function beforeUpdate(): void
{
    EntityManager::flush();
}
```

---

# 302. Anti-pattern: unknown = failed

Incorrecto:

```text
connection lost
→ mark operation failed
→ retry
```

---

# 303. Anti-pattern: rollback = restore PHP objects

Incorrecto:

```text
DB ROLLBACK
→ automatically restore all object properties
```

sin un sistema explícito de object-state restoration.

---

# 304. Anti-pattern: silent UnitOfWork clear

Incorrecto:

```text
too many entities
→ clear automatically
```

---

# 305. Anti-pattern: UnitOfWork owns authorization

Incorrecto:

```text
UnitOfWork::persist()
→ check whether HTTP user can update entity
```

El ORM core no deberá depender del contexto HTTP.

---

# 306. Fórmula de UnitOfWork

```text
UnitOfWork
=
TrackedEntities
+
PendingPersistenceIntents
+
ChangeCoordination
+
RelationshipCoordination
+
LifecycleCoordination
```

No:

```text
UnitOfWork
=
Transaction
+
SQL
```

---

# 307. Fórmula de preparación

```text
PreparedUnitOfWork
=
Stabilize(
    ExplicitRegistrations
    +
    CascadeDiscoveries
    +
    FieldChanges
    +
    RelationshipChanges
    +
    CollectionChanges
    +
    OrphanRemovals
)
```

---

# 308. Fórmula de dirty entity

```text
Dirty(E)
=
ChangeSet(E) ≠ ∅
```

bajo la estrategia de Change Tracking correspondiente.

---

# 309. Fórmula de graph closure

```text
TrackedGraph*
=
least fixed point
of
ExplicitEntities
+
CascadeReachability
+
PersistenceDependencies
```

---

# 310. Fórmula de safe flush preparation

```text
SafeFlushPreparation
=
ContextValid
∧
UnitOfWorkNotTainted
∧
CascadeClosureReached
∧
ChangesDetected
∧
RelationshipsAnalyzed
∧
OrphansStabilized
∧
LifecycleStabilized
∧
GraphValid
∧
ResourceBudgetsSatisfied
∧
PreparedStateFrozen
```

---

# 311. Fórmula de reconciliation

```text
Reconcile
=
ExecutionOutcome
+
OutcomeCertainty
+
GeneratedIdentifiers
+
ConfirmedEffects
+
FailedEffects
+
UnknownEffects
+
PreviousEntityState
+
PreviousSnapshots
```

---

# 312. Fórmula de successful synchronization

```text
Synchronize(E)
only if
PersistenceEffect(E) = CONFIRMED_SUCCESS
```

---

# 313. Fórmula de unknown safety

```text
UNKNOWN
≠
FAILED
≠
SUCCESS
```

y:

```text
UnknownUnreconciled
⇒
TaintedPersistenceContext
```

cuando no exista estrategia segura de reconciliación.

---

# 314. Fórmula de runtime isolation

```text
SafeUnitOfWorkRuntime
=
ScopedUnitOfWork
∧
ScopedIdentityMap
∧
ScopedEntityState
∧
ScopedSnapshots
∧
ScopedTemporaryTokens
∧
ImmutableSharedMetadata
∧
DeterministicReset
∧
NoImplicitFlushAtScopeEnd
```

---

# 315. Fórmula de memoria

```text
UnitOfWorkMemory
≈
TrackedEntities
+
Snapshots
+
ChangeSets
+
RelationshipState
+
CollectionState
+
GraphNodes
+
GraphEdges
+
SchedulingMetadata
```

---

# 316. Master Formula

```text
Database Unit of Work Architecture
=
Persistence Context Scoped Tracking
+
Entity Registration
+
Entity State Coordination
+
Identity Map Coordination
+
Temporary Entity Handles
+
Change Tracking Coordination
+
Snapshot Integration
+
Relationship Change Tracking
+
Collection Change Tracking
+
Cascade Discovery
+
Orphan Removal Analysis
+
Persistence Intent Scheduling
+
Entity Change Graph
+
Dependency Discovery
+
Lifecycle Stabilization
+
Immutable Flush Preparation
+
Deterministic Fingerprinting
+
Persistence Planner Handoff
+
Outcome Reconciliation
+
Generated Identity Coordination
+
Partial/Unknown Outcome Preservation
+
Transaction Boundary Separation
+
Resource Governance
+
Persistent Runtime Isolation
+
Telemetry
+
Diagnostics
+
Extension Governance
```

---

# 317. Master Rule

> **En VoltStack, UnitOfWork es la autoridad scoped que conoce y coordina qué cambios ORM están pendientes dentro de un PersistenceContext, pero delega la detección detallada de cambios, la planificación física, las transacciones, la generación de queries y la ejecución a los subsistemas especializados correspondientes.**

---

# 318. Resultado arquitectónico

Con `Identity Map + UnitOfWork` VoltStack obtiene:

```text
Entity Instances
      │
      ▼
Canonical Identity
      │
      ▼
Tracked Persistence State
      │
      ▼
Pending Changes
      │
      ▼
Entity Change Graph
      │
      ▼
Prepared Unit of Work
```

Esto establece la base para formalizar ahora:

```text
How are changes detected?
```

---

# 319. Relación con documentos siguientes

El bloque continuará:

```text
124 UnitOfWork Architecture
        │
        ▼
125 Change Tracking System
        │
        ▼
126 Entity Snapshot System
        │
        ▼
127 Persistence Engine
        │
        ▼
128 Persistence Planner
        │
        ├── 129 Insert Persistence
        ├── 130 Update Persistence
        └── 131 Delete Persistence
                │
                ▼
132 Flush System
        │
        ▼
133 Batch Persistence
        │
        ▼
134 Persistence Consistency
```

---

# 320. Siguiente documento

```text
125_DATABASE_CHANGE_TRACKING_SYSTEM.md
```

El siguiente documento deberá formalizar:

```text
Change Tracking architecture
dirty checking
snapshot comparison
explicit tracking
notification tracking
custom tracking
field-level changes
relationship changes
embedded value object changes
mutable value detection
typed equality
canonical comparison
original vs current values
ChangeSet model
ChangeSet entries
ChangeSet lifecycle
change normalization
generated/computed fields
identifier mutation detection
version fields
read-only fields
unloaded fields
partial entities
lazy properties
collection tracking boundary
change tracking policies
change detection performance
compiled change tracking plans
runtime caches
persistent worker isolation
telemetry
diagnostics
extension contracts
```

con la regla central:

> **Change Tracking determina qué estado persistente de una entidad cambió desde su baseline conocido; no decide cómo ese cambio será convertido en SQL ni si debe ejecutarse dentro de una transacción.**