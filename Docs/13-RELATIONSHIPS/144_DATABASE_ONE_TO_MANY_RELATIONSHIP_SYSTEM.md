# 144_DATABASE_ONE_TO_MANY_RELATIONSHIP_SYSTEM.md

# VoltStack Quantum Database
## Database One-to-Many Relationship System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 144 — Database One-to-Many Relationship System  
**Bloque:** 13 — Relationships  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database One-to-Many Relationship System` define la arquitectura mediante la cual VoltStack representa, carga, hidrata, rastrea y persiste asociaciones donde una entidad origen puede relacionarse con **cero, una o múltiples entidades destino**.

Ejemplos:

```text
User 1 ───── 0..N Order

Post 1 ───── 0..N Comment

Customer 1 ─ 0..N Address

Department 1 ─ 0..N Employee
```

El sistema deberá soportar:

- one-to-many unidireccional cuando la estrategia lo permita;
- one-to-many bidireccional;
- owning/inverse semantics;
- `PersistentCollection`;
- colecciones `SET` y `LIST`;
- lazy loading;
- eager loading;
- batch loading;
- extra-lazy operations;
- hydration de JOINs;
- deduplicación;
- collection completeness;
- filtered collections;
- snapshots;
- membership change tracking;
- add/remove/replace/clear;
- ordering;
- cascade persist;
- cascade remove;
- orphan removal;
- generated identifiers;
- pagination de relaciones;
- grandes colecciones;
- persistent runtime;
- tenant/shard isolation.

---

# 2. Principio central

> **Una relación one-to-many de VoltStack será una colección ORM identity-aware cuyo estado solo podrá considerarse representación completa de la relación persistida cuando exista evidencia suficiente de cobertura.**

Por tanto:

```text
CollectionEmpty
≠
RelationshipEmpty
```

si la colección está:

```text
UNINITIALIZED
```

o:

```text
PARTIAL
```

---

# 3. One-to-Many ≠ Array PHP

Una propiedad:

```php
private array $orders = [];
```

no proporciona por sí sola semántica ORM suficiente.

El ORM necesita conocer:

```text
initialization
completeness
membership identity
baseline
dirty operations
ordering
loading strategy
ownership
```

Por ello:

```text
PHP Collection Value
≠
ORM Relationship State
```

---

# 4. One-to-Many ≠ Query Result

Una consulta:

```php
$orders = Order::query()
    ->where('user_id', 10)
    ->get();
```

no implica necesariamente que:

```text
User#10.orders
```

haya sido inicializada.

---

# 5. One-to-Many ≠ Foreign Key

La relación ORM:

```text
User.orders
```

puede corresponder físicamente a:

```text
orders.user_id
```

pero:

```text
OneToManyRelationship
≠
ForeignKey
```

---

# 6. Modelo relacional habitual

```text
users
────────
id PK

orders
────────
id PK
user_id FK → users.id
```

Semánticamente:

```text
User 1 ────────── N Order
```

---

# 7. Lado físico habitual

En el modelo relacional:

```text
orders.user_id
```

está físicamente en el lado `many`.

Por ello, una relación bidireccional típica:

```text
User.orders
↔
Order.user
```

tendrá:

```text
User.orders
=
inverse side
```

y:

```text
Order.user
=
owning side
```

---

# 8. Regla de ownership

> **En una one-to-many bidireccional convencional basada en FK, el lado many-to-one será la autoridad de persistencia de la foreign key.**

Formalmente:

```text
User.orders inverse
Order.user owning
```

---

# 9. Collection Ownership ≠ Persistence Ownership

Que `User` contenga:

```text
orders[]
```

no significa que `User.orders` sea el owning side ORM.

---

# 10. Domain Ownership ≠ ORM Ownership

DDD puede considerar:

```text
User
```

aggregate root de `Order`, pero eso no cambia automáticamente la dirección física de la FK.

---

# 11. Posición arquitectónica

```text
                   Relationship Metadata
                           │
                           ▼
                  OneToManyMetadata
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
 Entity Query        Hydration          UnitOfWork
        │                  │                  │
        ▼                  ▼                  ▼
Collection Loader   Graph Assembly   Collection Snapshot
        │                  │                  │
        ▼                  ▼                  ▼
 Query Engine        IdentityMap      Collection Diff
                                               │
                                               ▼
                                 Relationship Persistence
                                               │
                                               ▼
                                  Persistence Planner
```

---

# 12. OneToManyMetadata

Propuesta:

```php
final readonly class OneToManyMetadata
{
    public function __construct(
        public RelationshipId $id,
        public EntityType $sourceType,
        public EntityType $targetType,
        public string $property,
        public ?string $mappedBy,
        public RelationshipDirection $direction,
        public RelationshipOwnership $ownership,
        public CollectionSemantics $collectionSemantics,
        public RelationshipFetchMode $fetchMode,
        public CascadePolicy $cascade,
        public OrphanRemovalPolicy $orphanRemoval,
        public CollectionOrdering $ordering,
        public CollectionMutability $mutability,
    ) {}
}
```

---

# 13. Collection Semantics

No todas las colecciones representan lo mismo.

Propuesta:

```php
enum CollectionSemantics
{
    case SET;
    case LIST;
}
```

Una futura extensión podría incorporar:

```text
BAG
MAP
```

si existen casos de uso suficientes.

---

# 14. SET

En una colección `SET`:

```text
membership
```

es lo relevante.

No se permiten duplicados lógicos de la misma identidad.

Formalmente:

```text
∀ EntityKey k:

count(k) ≤ 1
```

---

# 15. LIST

Una colección `LIST` añade semántica de posición:

```text
EntityKey
+
Position
```

Por tanto:

```text
LIST
≠
SET with sorting
```

---

# 16. Ordered SET ≠ LIST

Una colección puede ser un `SET` cuyo resultado se carga ordenado:

```text
ORDER BY created_at
```

sin que la posición sea estado persistente.

---

# 17. PersistentCollection

VoltStack deberá utilizar una abstracción ORM para representar relaciones to-many.

Propuesta:

```php
interface PersistentCollection extends \Countable, \IteratorAggregate
{
    public function relationship(): RelationshipId;

    public function owner(): object;

    public function state(): CollectionState;

    public function completeness(): CollectionCompleteness;

    public function isInitialized(): bool;

    public function initialize(): void;

    public function contains(object $entity): bool;

    public function add(object $entity): void;

    public function remove(object $entity): bool;

    public function clear(): void;
}
```

---

# 18. PersistentCollection ≠ Repository

No deberá utilizarse para consultas arbitrarias como:

```php
$user->orders->whereRaw(...);
```

si ello convierte la colección en un Query Builder oculto.

---

# 19. PersistentCollection ≠ Entity Cache

La colección contiene referencias del `PersistenceContext` actual.

No es una cache cross-request.

---

# 20. Collection State

Propuesta:

```php
enum CollectionState
{
    case UNINITIALIZED;
    case INITIALIZING;
    case INITIALIZED;
    case FAILED;
}
```

---

# 21. Collection Completeness

Separada de initialization:

```php
enum CollectionCompleteness
{
    case UNOBSERVED;
    case PARTIAL;
    case COMPLETE;
}
```

---

# 22. Initialized ≠ Complete

Una colección puede contener entidades y estar:

```text
INITIALIZED
+
PARTIAL
```

Ejemplo:

```php
$user->orders()
    ->where('status', 'open')
    ->loadIntoCollection();
```

si se permite explícitamente.

---

# 23. Empty ≠ Complete Empty

Esto:

```text
[]
```

puede significar:

```text
UNINITIALIZED
```

o:

```text
PARTIAL with zero matches
```

o:

```text
COMPLETE and empty
```

---

# 24. Fórmula de vacío seguro

```text
RelationshipKnownEmpty
=
CollectionInitialized
∧
CollectionCompleteness = COMPLETE
∧
MemberCount = 0
```

---

# 25. Runtime State

Conceptualmente:

```text
CollectionRuntimeState
=
Initialization
+
Completeness
+
KnownMembers
+
Baseline
+
PendingOperations
+
OrderingState
```

---

# 26. Membership Identity

La membresía deberá compararse por:

```text
EntityKey
```

cuando exista identidad persistente.

No mediante:

```text
=== object
```

como única semántica.

---

# 27. Object Identity Still Matters

UnitOfWork puede necesitar object identity para NEW entities aún sin ID.

Por ello, membership interno podrá manejar:

```text
Persistent EntityKey
OR
Temporary UnitOfWork Identity
```

---

# 28. Temporary Membership Identity

Una entidad NEW puede formar parte de:

```text
User.orders
```

antes de recibir ID.

El sistema deberá representarla sin fabricar una PK.

---

# 29. Collection Entry Key

Conceptualmente:

```text
CollectionMemberKey
=
Persistent EntityKey
|
Temporary UoW Entity Identity
```

---

# 30. Identity Promotion

Después de INSERT exitoso:

```text
TemporaryMemberKey
→
EntityKey
```

sin duplicar el miembro dentro de la colección.

---

# 31. Collection Canonicality

Si una entidad con `EntityKey(Order, 100)` ya existe en IdentityMap:

```text
collection member
```

deberá ser esa misma instancia.

---

# 32. No Duplicate Canonical Members

Para `SET`:

```text
Order#100
Order#100
```

deberá colapsarse a una única membresía lógica.

---

# 33. Lazy Collection

Estado inicial:

```text
UNINITIALIZED
+
UNOBSERVED
```

---

# 34. Access Behavior

Una operación como:

```php
foreach ($user->orders as $order) {
}
```

puede provocar:

```text
initialize collection
```

si lazy loading está permitido.

---

# 35. Lazy Loading Pipeline

```text
PersistentCollection
        │
        ▼
Initialization Guard
        │
        ▼
Collection Loader
        │
        ▼
Entity Query
        │
        ▼
Query Engine
        │
        ▼
Execution
        │
        ▼
Hydration
        │
        ▼
IdentityMap
        │
        ▼
Collection Assembly
        │
        ▼
COMPLETE
```

---

# 36. Lazy Loading ≠ Always Allowed

Strict mode podrá rechazar cargas inesperadas.

Ejemplo:

```text
LazyLoadingViolationException
```

durante serialización o paths sensibles.

---

# 37. Collection Loader

Contrato conceptual:

```php
interface OneToManyLoader
{
    public function load(
        object $owner,
        OneToManyMetadata $relationship,
        RelationshipLoadContext $context,
    ): CollectionLoadResult;
}
```

---

# 38. Loader Does Not Generate SQL

Pipeline obligatorio:

```text
Relationship Metadata
→ Entity Query
→ Query Model
→ Query Engine
→ Compiler
→ Executor
→ Hydration
```

---

# 39. Loader Result

```php
final readonly class CollectionLoadResult
{
    public function __construct(
        public CollectionCompleteness $completeness,
        public int $rowsObserved,
        public int $entitiesObserved,
        public CollectionLoadOutcome $outcome,
    ) {}
}
```

---

# 40. Successful Full Load

Solo un loader cuya consulta cubra toda la relación podrá establecer:

```text
COMPLETE
```

---

# 41. Filtered Query

```php
$user->orders()
    ->where('status', 'pending')
    ->get();
```

no deberá marcar automáticamente:

```text
$user->orders
```

como `COMPLETE`.

---

# 42. Critical Rule

```text
FilteredResult
≠
CompleteRelationshipState
```

---

# 43. Relationship Query ≠ Collection Initialization

Estas APIs deberán permanecer conceptualmente distintas:

```php
$user->orders();
```

consulta relacional.

```php
$user->orders;
```

colección administrada.

---

# 44. Explicit Initialization

Podrá existir:

```php
$entityManager->initialize($user->orders);
```

o API equivalente.

---

# 45. Eager Loading

Ejemplo:

```php
User::query()
    ->with('orders')
    ->get();
```

no obliga a utilizar JOIN.

---

# 46. Eager Strategies

El planner podrá seleccionar:

```text
JOIN_FETCH
SELECT_FETCH
BATCH_FETCH
```

según:

- cardinalidad;
- pagination;
- expected rows;
- múltiples colecciones;
- platform capabilities;
- query shape.

---

# 47. JOIN Fetch

Ejemplo:

```text
User#1 | Order#10
User#1 | Order#11
User#1 | Order#12
```

deberá producir:

```text
User#1
 └── orders
     ├── Order#10
     ├── Order#11
     └── Order#12
```

---

# 48. Root Deduplication

No:

```text
User#1
User#1
User#1
```

como tres objetos root cuando el result shape solicita entidades únicas.

---

# 49. Row Count ≠ Entity Count

```text
SQL rows = 3
Users = 1
Orders = 3
```

---

# 50. Cartesian Multiplication

Con:

```text
User
├── Orders[]
└── Roles[]
```

un JOIN puede producir:

```text
orders × roles
```

rows.

Hydration deberá deduplicar cada colección por identidad.

---

# 51. Example

```text
Order#10 Role#1
Order#10 Role#2
Order#11 Role#1
Order#11 Role#2
```

deberá resultar:

```text
Orders = {10, 11}
Roles  = {1, 2}
```

---

# 52. Hydration Assembly

El sistema de hydration deberá utilizar:

```text
HydrationPlan
+
RelationshipMetadata
+
IdentityMap
+
HydrationAssemblyLedger
```

para ensamblar colecciones.

---

# 53. Hydration Does Not Perform Additional Reads

La etapa de assembly solo trabaja con datos ya presentes.

No deberá ejecutar una query escondida para completar una colección.

---

# 54. Graph Finalization

Para eager to-many:

```text
entity scalar hydration
```

puede completarse antes que:

```text
collection graph assembly
```

Por ello:

```text
MATERIALIZED
≠
GRAPH_FINALIZED
```

---

# 55. postLoad Ordering

Cuando `postLoad` necesite garantías de eager graph:

```text
postLoad
```

deberá ejecutarse en el punto de finalización definido por Hydration Architecture, no por cada row físico.

---

# 56. postLoad Once

JOIN con 100 rows del mismo User:

```text
postLoad(User#1)
```

se ejecuta una vez por materialización lógica, no 100 veces.

---

# 57. Streaming Constraint

Streaming + eager to-many requiere conocer cuándo termina el grupo de un root.

Ejemplo:

```text
User#1 rows...
User#2 rows...
```

---

# 58. Safe Streaming Boundary

Puede utilizarse:

```text
ordered root identity grouping
```

para determinar cuándo una colección eager está completa.

---

# 59. Unsafe Streaming

Si no puede determinarse la frontera:

```text
stream eager to-many
```

deberá:

- rechazarse;
- cambiar estrategia;
- usar secondary batch fetch;
- bufferizar.

No deberá entregar una entidad declarada completa antes de tiempo.

---

# 60. Collection Snapshot

Propuesta:

```php
final readonly class CollectionSnapshot
{
    public function __construct(
        public RelationshipId $relationship,
        public EntityKey $owner,
        public CollectionCompleteness $completeness,
        public CollectionMembershipSnapshot $membership,
        public ?CollectionOrderingSnapshot $ordering,
    ) {}
}
```

---

# 61. Snapshot ≠ Live Collection

El snapshot es baseline.

La colección es estado actual.

---

# 62. Complete Snapshot

Si:

```text
completeness = COMPLETE
```

puede conocerse el conjunto completo de miembros observados.

---

# 63. Partial Snapshot

Si:

```text
completeness = PARTIAL
```

el snapshot no puede utilizarse como prueba de que los miembros ausentes no existen.

---

# 64. Membership Diff

Para `SET` completo:

```text
Added
=
Current − Baseline
```

```text
Removed
=
Baseline − Current
```

---

# 65. Formula

```text
CollectionChangeSet
=
AddedMembers
+
RemovedMembers
+
OrderingChanges
+
ExplicitOperations
```

---

# 66. Partial Collection Problem

No es seguro hacer:

```text
Removed = Baseline - Current
```

si baseline/current representan solo una porción filtrada.

---

# 67. Explicit Operation Tracking

Para colecciones incompletas, operaciones explícitas son críticas.

Ejemplo:

```php
$user->orders->remove($order);
```

puede registrar:

```text
REMOVE(Order#10)
```

sin asumir conocimiento del resto.

---

# 68. Operation Log

Propuesta:

```php
enum CollectionOperationKind
{
    case ADD;
    case REMOVE;
    case CLEAR;
    case MOVE;
}
```

---

# 69. Collection Mutation Log

```php
final readonly class CollectionOperation
{
    public function __construct(
        public CollectionOperationKind $kind,
        public ?CollectionMemberKey $member,
        public ?int $position = null,
    ) {}
}
```

---

# 70. Add

```php
$user->orders->add($order);
```

deberá:

1. validar tipo;
2. validar PersistenceContext;
3. validar tenant/shard;
4. deduplicar si SET;
5. registrar mutation intent;
6. sincronizar owning side según policy.

---

# 71. Add to Uninitialized Collection

No debería requerir cargar todos los miembros.

```text
UNINITIALIZED
+
ADD(Order#10)
```

puede mantenerse como operación pendiente.

---

# 72. Critical Performance Rule

```text
add(entity)
⇏
initialize entire collection
```

---

# 73. Remove

Igualmente:

```text
remove(Order#10)
```

no deberá necesariamente cargar 100,000 Orders.

---

# 74. Remove Unknown Member

Si la colección está incompleta y no se sabe si el miembro existe:

```text
REMOVE intent
```

puede registrarse sin fabricar baseline.

---

# 75. Remove Semantics

El sistema deberá distinguir:

```text
member known present
member known absent
membership unknown
```

---

# 76. Clear

`clear()` es especialmente peligroso.

En colección completa:

```text
clear
```

puede producir removals conocidos.

---

# 77. Clear on Uninitialized Collection

No deberá forzosamente cargar toda la colección.

Puede convertirse en una operación semántica:

```text
CLEAR_RELATIONSHIP
```

si la estrategia de persistencia puede ejecutarla de forma segura.

---

# 78. Clear ≠ Enumerated Remove

```text
CLEAR
```

no es necesariamente equivalente a crear N operaciones PHP `REMOVE`.

---

# 79. Orphan Removal + Clear

Si orphan removal exige eliminar físicamente cada dependent, el planner deberá determinar si:

- puede ejecutar bulk semantic delete;
- necesita IDs;
- necesita lifecycle por entidad;
- debe materializar dependents.

---

# 80. Bulk Semantics Warning

Una operación bulk que evita materialización no podrá fingir haber ejecutado:

```text
per-entity lifecycle callbacks
```

---

# 81. Remove vs Delete

Quitar:

```text
Order#10
```

de:

```text
User.orders
```

no significa automáticamente:

```text
DELETE Order#10
```

---

# 82. Without Orphan Removal

Normalmente puede implicar:

```text
Order.user = null
```

si nullable.

---

# 83. Required Many-to-One

Si `Order.user` es required:

```text
remove from User.orders
```

sin:

- nuevo owner;
- delete;
- orphan removal;

puede ser inválido.

---

# 84. Orphan Removal

Con:

```text
orphanRemoval = true
```

un dependent removido puede programarse para eliminación.

---

# 85. Safe Orphan Removal

```text
SafeOrphanRemoval
=
ExplicitRemoval
OR
CompleteBaselineEvidence
```

más:

```text
RelationshipDependencyRulesSatisfied
```

---

# 86. Partial Collection Orphan Safety

No:

```text
not present in partial result
→ orphan
```

---

# 87. Cascade Persist

Al añadir una entidad NEW:

```php
$user->orders->add(new Order());
```

con cascade persist:

```text
Order → NEW in UoW
```

---

# 88. Without Cascade

NEW target no registrado:

```text
flush
→ TransientRelationshipEntityException
```

o error equivalente.

---

# 89. Cascade Remove

Eliminar `User` puede programar eliminación de todos sus Orders si mapping lo declara.

---

# 90. Cascade Remove Large Collection

No deberá inicializar automáticamente millones de dependents salvo que la semántica realmente lo exija.

---

# 91. Database Cascade

Cuando exista:

```text
ON DELETE CASCADE
```

el planner podrá aprovecharlo.

Pero deberá reconciliar:

```text
managed Order entities
```

que ya estén presentes en PersistenceContext.

---

# 92. ORM Cascade ≠ DB Cascade

Nunca:

```text
DB cascade
=
automatic UoW knowledge
```

---

# 93. Bidirectional Fix-Up

Para:

```text
User.orders
↔
Order.user
```

añadir un Order puede requerir:

```text
Order.user = User
```

---

# 94. Fix-Up Policy

VoltStack deberá soportar una policy explícita.

Ejemplo:

```php
enum BidirectionalRelationshipPolicy
{
    case STRICT;
    case AUTO_FIXUP;
}
```

---

# 95. STRICT

Si:

```text
$user->orders contains $order
```

pero:

```text
$order->user !== $user
```

flush puede fallar con diagnóstico.

---

# 96. AUTO_FIXUP

La colección puede sincronizar el owning side.

Pero deberá hacerlo mediante infraestructura ORM controlada.

---

# 97. Domain Methods Preferred

El dominio puede mantener coherencia:

```php
public function addOrder(Order $order): void
{
    $this->orders->add($order);
    $order->assignUser($this);
}
```

El ORM no deberá impedir este patrón.

---

# 98. Infinite Fix-Up Prevention

```text
User.orders.add(Order)
→ Order.user = User
→ User.orders.add(Order)
→ ...
```

deberá detenerse mediante ledger/context interno.

---

# 99. Ownership Conflict

```text
User#1.orders contains Order#10
Order#10.user = User#2
```

es conflicto conocido.

---

# 100. Graph Validation

Antes de plan freeze:

```text
validate relationship graph
```

para relaciones suficientemente conocidas.

---

# 101. Unknown Graph State

No deberá asumirse conflicto si el inverse collection está:

```text
UNINITIALIZED
```

---

# 102. Knowledge-Aware Graph Validation

```text
KnownContradiction
→ ERROR

UnknownState
→ no fabricated contradiction
```

---

# 103. Reparenting

Mover:

```text
Order#10
```

de:

```text
User#1
```

a:

```text
User#2
```

es una operación importante.

---

# 104. Reparenting ≠ Remove + Orphan Delete

Si el mismo flush establece:

```text
Order#10.user = User#2
```

el sistema no deberá eliminarlo como orphan de `User#1`.

---

# 105. Orphan Stabilization

Orphan removal debe evaluarse después de estabilizar el relationship graph.

---

# 106. Formula

```text
TrueOrphan(entity)
=
PreviouslyDependent
∧
NoCurrentValidOwner
∧
OrphanRemovalEnabled
```

---

# 107. Generated Owner ID

Caso:

```text
new User
├── new Order A
└── new Order B
```

requiere:

```text
Insert User
↓
generated User.id
↓
Insert Orders with User.id
```

---

# 108. Persistence Dependency

```text
Insert(User)
    │
    ├──→ Insert(Order A)
    └──→ Insert(Order B)
```

después de identity barrier.

---

# 109. Existing Owner + New Members

Puede batch insertarse:

```text
Order A
Order B
Order C
```

si:

- dependency satisfied;
- query shape compatible;
- lifecycle semantics preservadas;
- generated-value correlation segura.

---

# 110. Batch Persistence

Batching es optimización física.

No cambia:

```text
per-entity semantic operations
```

---

# 111. Collection Persistence ChangeSet

Propuesta:

```php
final readonly class OneToManyChangeSet
{
    /**
     * @param list<CollectionMemberKey> $added
     * @param list<CollectionMemberKey> $removed
     */
    public function __construct(
        public RelationshipId $relationship,
        public EntityKey $owner,
        public array $added,
        public array $removed,
        public bool $clearRequested,
        public CollectionCompleteness $baselineCompleteness,
    ) {}
}
```

---

# 112. ChangeSet ≠ SQL

Persistence layer transforma:

```text
OneToManyChangeSet
```

a operaciones semánticas.

---

# 113. Semantic Operations

Ejemplos:

```text
AssociateDependent
DissociateDependent
ReparentDependent
ClearDependents
ScheduleOrphanRemoval
UpdateCollectionPosition
```

---

# 114. Physical Persistence

Para FK-based one-to-many:

```text
User.orders ADD Order#10
```

puede terminar como modificación de:

```text
Order.user_id
```

---

# 115. Inverse Collection Does Not Write FK Directly

El owning many-to-one sigue siendo la fuente semántica de la FK.

---

# 116. Persistence Planner

Deberá reconciliar:

```text
inverse collection mutations
+
owning-side mutations
```

en un único graph de operaciones.

---

# 117. No Double Update

Si:

```text
$user->orders->add($order);
$order->setUser($user);
```

no deberán generarse dos UPDATE equivalentes.

---

# 118. Canonical Relationship Change

Ambas observaciones deberán converger a una única operación semántica.

---

# 119. Ordering

Una colección puede declarar:

```text
orderBy created_at DESC
```

como orden de carga.

---

# 120. Load Ordering ≠ Persistent Position

Esto:

```text
ORDER BY created_at
```

no implica que mover elementos en memoria deba actualizar DB.

---

# 121. Persistent List Position

Para semántica `LIST`, puede existir:

```text
position column
```

Ejemplo:

```text
playlist_items.position
```

---

# 122. Position Tracking

Entonces snapshot deberá almacenar:

```text
EntityKey → Position
```

---

# 123. Move Operation

```text
MOVE Order#10 from 4 → 1
```

será diferente de remove/add.

---

# 124. Ordering Collision

Reordenar múltiples miembros puede requerir algoritmo seguro para constraints de posición única.

---

# 125. Sparse Positioning

Una futura estrategia podrá usar:

```text
10, 20, 30, 40
```

para reducir updates.

Esto pertenece al persistence strategy, no al API semántico.

---

# 126. Extra-Lazy Operations

Grandes colecciones no deberían requerir carga completa para operaciones simples.

---

# 127. Candidate Operations

```text
count()
contains()
containsKey()
isEmpty()
slice()
```

podrán ser extra-lazy cuando sea seguro.

---

# 128. Extra-Lazy Count

```php
$count = $user->orders->count();
```

podrá producir:

```text
SELECT COUNT(...)
```

sin inicializar toda la colección.

---

# 129. Count Result ≠ Collection Initialization

Después:

```text
count = 100000
```

la colección puede seguir:

```text
UNINITIALIZED
```

---

# 130. Pending Mutations + Extra-Lazy Count

Si existen:

```text
pending ADD/REMOVE
```

el resultado deberá incorporar cambios locales conocidos o abandonar fast path.

---

# 131. Formula

```text
LogicalCount
=
DatabaseCount
+
PendingAdds
-
ConfirmedPendingRemoves
```

solo cuando pueda calcularse con certeza.

---

# 132. Unknown Membership Problem

Si no se sabe si un removed member existía en DB:

```text
DatabaseCount - 1
```

no es seguro.

Debe:

- consultar membership;
- inicializar;
- utilizar otra estrategia;
- reportar incertidumbre/error según API.

---

# 133. contains()

Podrá consultar:

```text
EXISTS
```

por identidad sin inicializar la colección.

---

# 134. Pending Add

Si target está pending ADD:

```text
contains(target)
→ true
```

sin query.

---

# 135. Pending Remove

Si target tiene confirmed pending REMOVE:

```text
contains(target)
→ false
```

---

# 136. isEmpty()

Puede usar:

```text
EXISTS
```

en lugar de COUNT.

---

# 137. Extra-Lazy ≠ Hidden Semantic Change

Estas optimizaciones no deberán marcar:

```text
collection COMPLETE
```

---

# 138. Slice

Una colección podrá obtener:

```text
offset 100
limit 20
```

sin cargar todo.

El resultado será:

```text
partial view
```

no complete collection.

---

# 139. Partial Collection View

Conviene separar:

```text
PersistentCollection
```

de:

```text
CollectionSlice
```

para evitar que una página se confunda con la relación completa.

---

# 140. Pagination

No:

```text
$page = $user->orders->paginate();
```

si ello implica que la colección administrada pasa a representar solo una página sin marcarlo.

Preferir una Relationship Query.

---

# 141. Relationship Query Pagination

```php
$user->orders()
    ->orderBy('created_at')
    ->paginate(50);
```

produce un resultado paginado separado.

---

# 142. Paginated Result ≠ Managed Collection Completeness

Las entidades individuales pueden ser managed.

La colección `User.orders` puede continuar `UNINITIALIZED`.

---

# 143. Large Collections

Una colección con millones de entidades exige evitar:

```text
full materialization
```

para operaciones que no lo requieren.

---

# 144. Memory Governance

El sistema deberá controlar:

- member index;
- snapshot size;
- pending operation log;
- hydration dedup maps;
- batch buffers.

---

# 145. Snapshot Optimization

Para colecciones completas muy grandes podrían evaluarse estrategias como:

```text
identity set
compressed identity representation
operation-based tracking
```

sin perder corrección.

---

# 146. Snapshot Optimization ≠ Hash-Only Destructive Decisions

Un hash global no basta para saber qué miembros remover.

---

# 147. Clear Optimization

Una operación:

```text
CLEAR_RELATIONSHIP
```

puede evitar enumerar miembros si:

- no se requiere lifecycle individual;
- orphan semantics lo permiten;
- DB operation es segura;
- managed affected entities pueden reconciliarse.

---

# 148. Bulk DML Distinction

```text
Collection Persistence
≠
Bulk Entity DML
```

aunque ciertas optimizaciones físicas puedan parecer similares.

---

# 149. Lifecycle Preservation

Si semántica exige:

```text
preRemove/postRemove
```

por cada orphan entity, una operación bulk que no materializa entidades no puede fingir esos eventos.

---

# 150. N+1

Caso:

```php
foreach ($users as $user) {
    foreach ($user->orders as $order) {
    }
}
```

puede generar N+1.

---

# 151. N+1 Detection

Relationship Loading deberá emitir suficiente telemetry para que:

```text
DATABASE_N_PLUS_ONE_DETECTION_SYSTEM
```

pueda identificarlo.

---

# 152. Batch Loading

Puede convertir:

```text
100 lazy collection initializations
```

en:

```text
1..N batched relationship queries
```

---

# 153. Batch Load Query

Conceptualmente:

```text
WHERE order.user_id IN (...)
```

---

# 154. Batch Assembly

Cada row deberá asociarse al owner correcto.

---

# 155. Missing Owners

Si un owner no tiene rows:

```text
complete batch load
```

deberá marcar su colección:

```text
COMPLETE + EMPTY
```

---

# 156. Batch Partial Failure

Si no existe certeza sobre todos los owners:

```text
do not mark every collection COMPLETE
```

---

# 157. Collection Load Outcome

Podrá manejar:

```text
SUCCEEDED
FAILED
CANCELLED
UNKNOWN
```

y evidencia por owner cuando sea necesario.

---

# 158. Query Failure

Una carga fallida no transforma:

```text
UNINITIALIZED
```

en:

```text
EMPTY
```

---

# 159. Cancellation

Si una colección se estaba inicializando:

```text
INITIALIZING
```

y se cancela, deberá restaurarse a un estado coherente.

---

# 160. Staged Initialization

Recomendación:

```text
load into temporary assembly
↓
validate
↓
finalize
↓
publish into PersistentCollection
```

para evitar colecciones medio cargadas.

---

# 161. Initialization Atomicity

Para una carga no streaming normal:

```text
either finalized collection
or previous state
```

cuando sea posible.

---

# 162. Streaming Exception

Si resultados ya fueron expuestos por streaming:

```text
cannot un-yield
```

pero cada entidad/segmento expuesto deberá haber sido finalizado conforme a sus garantías.

---

# 163. Refresh Collection

Podrá existir:

```php
$entityManager->refresh($user->orders);
```

o API equivalente.

---

# 164. Dirty Collection Refresh

Si existen cambios locales:

```text
ADD/REMOVE/CLEAR
```

refresh requiere policy.

---

# 165. Refresh Policies

```php
enum CollectionRefreshPolicy
{
    case REJECT_DIRTY;
    case DISCARD_LOCAL_CHANGES;
    case MERGE_IF_SAFE;
}
```

Default:

```text
REJECT_DIRTY
```

---

# 166. Normal Hydration Does Not Overwrite Dirty Collection

Una query adicional no deberá reemplazar silenciosamente pending mutations.

---

# 167. Partial Merge

Una carga parcial posterior puede añadir conocimiento:

```text
previously unobserved member
```

sin declarar la colección completa.

---

# 168. Complete Load Merge

Si existen pending local changes, un complete DB load deberá reconciliarlas explícitamente.

---

# 169. Database View vs Logical Current Collection

Con pending ADD:

```text
DB = {A, B}
local current = {A, B, C}
```

El loader no deberá eliminar `C` simplemente porque no aparece en DB.

---

# 170. Formula

```text
LogicalCurrentMembership
=
ObservedDatabaseMembership
⊕
PendingLocalOperations
```

donde `⊕` es reconciliación semántica, no simple reemplazo.

---

# 171. Removal Reconciliation

Si:

```text
DB = {A, B}
pending REMOVE B
```

tras load:

```text
logical current = {A}
```

mientras baseline puede reflejar la observación DB necesaria para change tracking.

---

# 172. Collection Baseline vs Current

Debe mantenerse separación:

```text
DatabaseObservation
Baseline
CurrentLogicalCollection
PendingOperations
```

---

# 173. Database Knowledge

Una collection snapshot es conocimiento ORM de una observación.

No verdad eterna de DB.

---

# 174. External Mutations

Raw SQL/bulk operations pueden volver una colección:

```text
STALE
```

---

# 175. Invalidation

El sistema podrá marcar:

```text
UNOBSERVED
STALE
```

sin necesariamente vaciar objetos actuales.

---

# 176. Consistency Integration

Estados generales:

```text
CONSISTENT
STALE
UNCERTAIN
INCONSISTENT
TAINTED
UNKNOWN
```

se aplican también al conocimiento de relaciones.

---

# 177. Persistence Success

Después de una operación suficientemente cierta:

```text
relationship baseline
```

puede reconciliarse.

---

# 178. UNKNOWN Persistence

Si resultado de FK update/delete es incierto:

```text
do not fabricate new clean baseline
```

---

# 179. Transaction Boundary

```text
flush success
≠
transaction commit
```

también para cambios de colección.

---

# 180. Rollback

DB rollback no rebobina automáticamente:

```text
$user->orders
```

---

# 181. Example

Baseline:

```text
{A, B}
```

Current:

```text
{A, B, C}
```

Flush inserta/asocia `C`.

Transaction rollback.

Objeto puede seguir:

```text
{A, B, C}
```

y deberá quedar dirty/reconciliable según transaction policy.

---

# 182. No Collection Time Machine

VoltStack no intentará reconstruir arbitrariamente todas las mutaciones PHP después de rollback.

---

# 183. Detached Owner

Una PersistentCollection perteneciente a entidad detached no deberá cargar DB mediante un manager histórico.

---

# 184. Detached Lazy Collection

Acceso que requiere DB podrá lanzar:

```text
DetachedCollectionInitializationException
```

---

# 185. Already Initialized Detached Collection

Podrá seguir iterándose como estado PHP conocido, pero ya no representa automáticamente estado managed actual.

---

# 186. Cross-Context Member

No deberá agregarse una entidad managed por otro PersistenceContext incompatible.

---

# 187. Tenant Isolation

Para relaciones tenant-local:

```text
Tenant(owner)
=
Tenant(member)
```

---

# 188. Shard Isolation

Igualmente:

```text
Shard(owner)
=
Shard(member)
```

salvo mapping explícito cross-shard.

---

# 189. Cross-Database Collection

Puede existir conceptualmente, pero:

```text
DB FK
```

quizá no sea posible.

Loading y persistence deberán expresar limitaciones de atomicidad.

---

# 190. No Distributed Transaction Assumption

```text
CrossDatabaseOneToMany
⇏
DistributedTransaction
```

---

# 191. Persistent Runtime

Shared immutable:

```text
OneToManyMetadata
compiled accessors
compiled load-plan templates
mapping validators
```

Scoped mutable:

```text
PersistentCollection
CollectionRuntimeState
CollectionSnapshot
pending operations
load guards
batch queues
```

---

# 192. FrankenPHP Safety

Nunca:

```php
static array $collections;
```

para almacenar colecciones managed entre requests.

---

# 193. Request A ≠ Request B

Aunque ambos carguen:

```text
User#10.orders
```

cada request posee:

```text
different PersistenceContext
different User object
different collection
different snapshot
```

---

# 194. Runtime Reset

Al finalizar scope:

```text
clear collection load queues
clear collection snapshots with UoW
clear initialization guards
clear pending collection operations
release scoped entity references
```

---

# 195. Concurrent Initialization

Dos fibers no deberán inicializar simultáneamente la misma colección sin coordinación.

---

# 196. Initialization Guard

```text
UNINITIALIZED
     │
     ▼
INITIALIZING
     │
 ┌───┴────┐
 ▼        ▼
SUCCESS   FAILURE
 │        │
 ▼        ▼
INITIALIZED  recoverable state
```

---

# 197. Recursive Initialization

Si loader/lifecycle vuelve a acceder a la misma colección durante `INITIALIZING`:

```text
detect recursive initialization
```

---

# 198. Collection Mutation During Initialization

Debe existir policy.

Default recomendado:

```text
stage mutation
```

y reconciliar al finalizar, en lugar de perderla.

---

# 199. Collection Mutation During Iteration

Para colecciones inicializadas:

```text
add/remove while iterating
```

deberá tener semántica PHP/ORM documentada.

No deberá corromper índices internos.

---

# 200. Iterator Snapshot

Una implementación puede utilizar:

```text
stable iterator snapshot
```

para evitar comportamiento no determinista.

---

# 201. Security

Relationship loading deberá respetar:

- tenant namespace;
- shard/database context;
- trusted metadata;
- validated entity types;
- parameter binding;
- connection security.

---

# 202. Authorization

Que:

```text
User.orders
```

exista no significa que el usuario HTTP actual pueda leer todos los Orders.

---

# 203. ORM Relationship ≠ Access-Control Filter

No deberán mezclarse automáticamente políticas de autorización con la definición estructural de la relación.

---

# 204. Security Scopes

Si VoltStack aplica scopes de seguridad a Entity Query, su efecto sobre:

```text
collection completeness
```

deberá ser explícito.

---

# 205. Important Security Completeness Rule

Una query filtrada por autorización puede representar:

```text
all entities visible to principal
```

pero no:

```text
all persisted relationship members
```

Son universos semánticos diferentes.

---

# 206. Completeness Domain

Una futura extensión puede asociar completeness a:

```text
RelationshipViewScope
```

para expresar:

```text
complete within query/security scope
```

sin confundirlo con integridad física global.

---

# 207. Serialization

Serializar:

```text
User
```

no deberá inicializar automáticamente `orders`.

---

# 208. API Representation

Puede representar:

```text
orders omitted
```

si no fueron solicitados.

---

# 209. Loaded Serialization

Cuando se solicite explícitamente:

```text
include=orders
```

la capa API podrá ordenar una carga controlada.

---

# 210. Avoid Serialization N+1

El serializer no deberá convertirse en un relationship loader accidental.

---

# 211. Metadata Validation

Bootstrap deberá validar:

```text
source type
target type
mappedBy
inverse mapping
collection property
collection semantics
fetch mode
cascade
orphan removal
ordering
mutability
identifier compatibility
```

---

# 212. mappedBy Validation

Si:

```text
User.orders mappedBy "user"
```

entonces `Order.user` deberá:

- existir;
- apuntar a `User`;
- tener cardinalidad compatible;
- ser owning side cuando corresponda.

---

# 213. Cardinality Compatibility

Esperado:

```text
User.orders ONE_TO_MANY
Order.user MANY_TO_ONE
```

---

# 214. Invalid Pair

```text
User.orders ONE_TO_MANY
Order.user ONE_TO_ONE
```

no deberá aceptarse como el mismo relationship mapping.

---

# 215. Collection Type Validation

La propiedad deberá aceptar el tipo de colección configurado.

---

# 216. Constructor Collection

Entidades pueden inicializar:

```php
$this->orders = new ArrayCollection();
```

pero ORM deberá adaptarla/validarla según estrategia.

---

# 217. Persistent Wrapper

VoltStack podrá envolver una colección existente:

```text
Domain Collection
↓
PersistentCollection Adapter
```

si conserva invariantes.

---

# 218. No Base Collection Requirement

No debería obligarse a toda entidad a depender directamente de una clase ORM concreta si puede evitarse.

---

# 219. Collection Adapter Contract

Podrá existir:

```php
interface CollectionAdapter
{
    public function elements(): iterable;

    public function add(object $entity): void;

    public function remove(object $entity): bool;
}
```

---

# 220. Domain Collection Support

Una colección de dominio especializada podrá integrarse mediante metadata/accessors/adapters.

---

# 221. Readonly Collection Property

Una propiedad readonly puede apuntar a un objeto colección mutable.

Esto es diferente de una colección conceptualmente inmutable.

---

# 222. Collection Mutability

Propuesta:

```php
enum CollectionMutability
{
    case MUTABLE;
    case APPEND_ONLY;
    case IMMUTABLE_AFTER_LOAD;
}
```

---

# 223. APPEND_ONLY

Puede permitir:

```text
ADD
```

pero rechazar:

```text
REMOVE
CLEAR
```

---

# 224. Immutable Collection

Después de establecer baseline:

```text
membership cannot change
```

salvo operación especializada.

---

# 225. Query Navigation

Entity Query podrá resolver:

```php
User::query()
    ->whereHas('orders', fn ($q) =>
        $q->where('status', 'pending')
    );
```

---

# 226. Semantic Query

Esto deberá convertirse a:

```text
relationship-aware Query Model
```

no SQL dentro del ORM.

---

# 227. has()

```php
User::query()->has('orders');
```

podrá convertirse a `EXISTS`, JOIN u otra forma por optimizer/compiler.

---

# 228. doesntHave()

Igualmente:

```text
NOT EXISTS
```

es una posible representación física, no una obligación del Relationship System.

---

# 229. Count Relation Query

```php
User::query()
    ->withCount('orders');
```

produce una proyección/aggregate.

No inicializa `User.orders`.

---

# 230. Critical Rule

```text
Relationship Aggregate
≠
Relationship Collection Initialization
```

---

# 231. Collection Index

PersistentCollection puede mantener:

```text
memberKey → object
```

para `SET`.

---

# 232. LIST Index

Puede mantener:

```text
position → member
memberKey → position
```

cuando sea necesario.

---

# 233. Duplicate Add

SET:

```php
$orders->add($sameOrder);
$orders->add($sameOrder);
```

debe ser idempotente respecto a membership.

---

# 234. Duplicate Canonical Identity

Dos objetos diferentes con mismo established EntityKey:

```text
must trigger IdentityMap conflict
```

no dos collection entries.

---

# 235. Remove/Re-add Stabilization

Baseline:

```text
{A, B}
```

Operations:

```text
REMOVE B
ADD B
```

puede estabilizarse a:

```text
NO NET CHANGE
```

si identidad/semántica lo permiten.

---

# 236. Add/Remove Stabilization

```text
ADD C
REMOVE C
```

para NEW/non-baseline member puede cancelarse.

---

# 237. Operation Normalization

Antes de PersistencePlan:

```text
raw collection operations
↓
normalized relationship changes
```

---

# 238. Deterministic ChangeSet

Misma baseline + mismas operaciones semánticas deberán producir mismo ChangeSet.

---

# 239. Collection Version

Puede utilizarse:

```text
CollectionMutationVersion
```

para detectar planes obsoletos.

---

# 240. Stale Plan Detection

Si colección cambia después de plan generation:

```text
PlanCollectionVersion
≠
CurrentCollectionVersion
```

→ plan inválido.

---

# 241. Flush Stabilization

Lifecycle callbacks pueden modificar colecciones antes del plan freeze.

---

# 242. Recompute

Después de `pre*` lifecycle:

```text
recompute collection changes
```

hasta estabilización bounded.

---

# 243. Frozen Workset

Después:

```text
CollectionChangeSet
```

se vuelve parte del immutable PersistencePlan.

---

# 244. Mutation After Freeze

No deberá modificar silenciosamente el plan actual.

---

# 245. post* Mutation

Será trabajo para:

```text
future flush
```

---

# 246. Error Taxonomy

```text
OneToManyRelationshipException
├── OneToManyMetadataException
├── OneToManyMappingException
├── OneToManyOwnershipException
├── OneToManyCollectionException
├── CollectionInitializationException
├── CollectionCompletenessException
├── CollectionLoadException
├── CollectionCardinalityException
├── CollectionMembershipException
├── CollectionDuplicateIdentityException
├── CollectionChangeTrackingException
├── CollectionPersistenceException
├── CollectionOrderingException
├── CollectionOrphanRemovalException
├── CollectionCascadeException
├── CollectionReparentingException
├── CollectionDetachedOwnerException
├── CollectionCrossContextException
├── CollectionTenantIsolationException
├── CollectionConcurrentInitializationException
├── CollectionRecursiveInitializationException
├── CollectionRefreshException
├── CollectionReconciliationException
├── CollectionRuntimeIsolationException
└── OneToManyInvariantException
```

---

# 247. Directory Structure

```text
src/Quantum/Database/ORM/Relationship/OneToMany/
│
├── Metadata/
│   ├── OneToManyMetadata.php
│   ├── CollectionSemantics.php
│   ├── CollectionOrdering.php
│   └── CollectionMutability.php
│
├── Collection/
│   ├── PersistentCollection.php
│   ├── ManagedPersistentCollection.php
│   ├── CollectionAdapter.php
│   ├── CollectionMemberKey.php
│   └── CollectionMemberIndex.php
│
├── State/
│   ├── CollectionState.php
│   ├── CollectionCompleteness.php
│   ├── CollectionRuntimeState.php
│   └── CollectionMutationVersion.php
│
├── Loading/
│   ├── OneToManyLoader.php
│   ├── CollectionLoadPlan.php
│   ├── CollectionLoadResult.php
│   ├── CollectionBatchLoader.php
│   └── CollectionInitializationGuard.php
│
├── Hydration/
│   ├── OneToManyAssembler.php
│   ├── CollectionHydrationLedger.php
│   └── CollectionGraphFinalizer.php
│
├── Snapshot/
│   ├── CollectionSnapshot.php
│   ├── CollectionMembershipSnapshot.php
│   └── CollectionOrderingSnapshot.php
│
├── ChangeTracking/
│   ├── OneToManyChangeSet.php
│   ├── CollectionOperation.php
│   ├── CollectionOperationKind.php
│   ├── CollectionOperationLog.php
│   └── CollectionChangeTracker.php
│
├── Persistence/
│   ├── OneToManyPersistenceOperation.php
│   ├── AssociateDependentOperation.php
│   ├── DissociateDependentOperation.php
│   ├── ReparentDependentOperation.php
│   ├── ClearDependentsOperation.php
│   └── CollectionPersistencePlanner.php
│
├── Ordering/
│   ├── CollectionPosition.php
│   └── CollectionOrderingTracker.php
│
├── Policy/
│   ├── CollectionRefreshPolicy.php
│   ├── BidirectionalRelationshipPolicy.php
│   └── ExtraLazyPolicy.php
│
├── Validation/
│   ├── OneToManyMappingValidator.php
│   └── OneToManyGraphValidator.php
│
├── Telemetry/
│   ├── OneToManyTelemetry.php
│   └── CollectionDiagnostics.php
│
└── Exception/
    └── ...
```

---

# 248. Telemetry

Métricas sugeridas:

```text
orm.relationship.one_to_many.loads
orm.relationship.one_to_many.lazy_loads
orm.relationship.one_to_many.eager_loads
orm.relationship.one_to_many.batch_loads

orm.relationship.collection.initializations
orm.relationship.collection.initialization_failures

orm.relationship.collection.members_loaded
orm.relationship.collection.members_deduplicated

orm.relationship.collection.adds
orm.relationship.collection.removes
orm.relationship.collection.clears
orm.relationship.collection.moves

orm.relationship.collection.extra_lazy.count
orm.relationship.collection.extra_lazy.contains
orm.relationship.collection.extra_lazy.exists

orm.relationship.collection.orphan_removals
orm.relationship.collection.reparentings

orm.relationship.collection.partial
orm.relationship.collection.complete

orm.relationship.collection.refresh_failures
orm.relationship.collection.cross_context_failures
```

---

# 249. Telemetry Cardinality

No utilizar:

```text
owner_id
member_id
email
tenant secrets
```

como labels de alta cardinalidad.

---

# 250. Diagnostics

Ejemplo:

```text
ONE-TO-MANY RELATIONSHIP

Relationship:
    User.orders

Target:
    Order

Mapped By:
    Order.user

Ownership:
    INVERSE

Collection:
    SET

State:
    INITIALIZED

Completeness:
    COMPLETE

Known Members:
    12

Pending Adds:
    1

Pending Removes:
    2

Dirty:
    YES

Orphan Removal:
    YES

Fetch:
    LAZY
```

---

# 251. Partial Diagnostic

```text
COLLECTION IS PARTIAL

Relationship:
    User.orders

Known members:
    20

Completeness:
    PARTIAL

Reason:
    filtered relationship load

Warning:
    Absence from this collection does not prove
    absence from the persisted relationship.
```

---

# 252. Testing Strategy

Deberá cubrir:

```text
empty complete collection
uninitialized collection
partial collection
lazy initialization
eager JOIN
batch loading
JOIN deduplication
cartesian multiplication
IdentityMap reuse
add/remove/clear
uninitialized mutations
cascade persist/remove
orphan removal
reparenting
ordering
LIST positions
extra-lazy count/contains/isEmpty
pagination
large collections
refresh
rollback
UNKNOWN persistence
tenant isolation
persistent runtime
concurrent initialization
```

---

# 253. Empty Complete Test

DB sin Orders para User#1:

```text
load full relation
```

debe producir:

```text
INITIALIZED
COMPLETE
count = 0
```

---

# 254. Uninitialized Empty Representation Test

Antes de load:

```text
collection internal member count may be zero
```

pero:

```text
must not claim relationship empty
```

---

# 255. Partial Test

Carga solo pending orders:

```text
completeness = PARTIAL
```

---

# 256. JOIN Dedup Test

Rows repetidas por segundo JOIN no duplican Orders.

---

# 257. IdentityMap Test

Order ya managed:

```text
collection member === canonical Order
```

---

# 258. Add Without Initialization Test

```text
UNINITIALIZED
+ ADD Order#10
```

no dispara full load.

---

# 259. Remove Without Initialization Test

No dispara full load cuando puede registrarse intent de forma segura.

---

# 260. Clear Without Initialization Test

Produce semantic clear o estrategia segura.

No materialización obligatoria arbitraria.

---

# 261. Orphan Partial Test

Ausencia en partial collection no elimina entidad.

---

# 262. Reparent Test

```text
User#1 → Order#10
```

a:

```text
User#2 → Order#10
```

no genera orphan delete.

---

# 263. Generated Owner Test

ID generado del owner se propaga a todos los dependents.

---

# 264. Extra-Lazy Count Test

COUNT no inicializa colección.

---

# 265. Extra-Lazy Pending Add Test

Logical count incorpora pending add cuando es seguro.

---

# 266. Extra-Lazy Unknown Remove Test

No resta ciegamente cuando membership previa es desconocida.

---

# 267. Pagination Test

Página de 50 Orders no marca colección completa.

---

# 268. Refresh Dirty Test

Default:

```text
REJECT_DIRTY
```

---

# 269. Rollback Test

Rollback DB no rebobina colección automáticamente.

---

# 270. UNKNOWN Test

Outcome incierto no produce clean baseline falso.

---

# 271. Persistent Runtime Test

Request A y Request B no comparten colección ni snapshot.

---

# 272. Architectural Invariants

## DB-ORM-ONE-TO-MANY-001

One-to-many será una relación collection-valued.

## DB-ORM-ONE-TO-MANY-002

One-to-many será distinto de array PHP.

## DB-ORM-ONE-TO-MANY-003

One-to-many será distinto de query result.

## DB-ORM-ONE-TO-MANY-004

One-to-many será distinto de foreign key.

## DB-ORM-ONE-TO-MANY-005

La semántica ORM será independiente de representación física.

## DB-ORM-ONE-TO-MANY-006

Una relación FK bidireccional convencional tendrá el many-to-one como owning side.

## DB-ORM-ONE-TO-MANY-007

Collection ownership será distinto de persistence ownership.

## DB-ORM-ONE-TO-MANY-008

DDD ownership será distinto de ORM ownership.

## DB-ORM-ONE-TO-MANY-009

OneToManyMetadata será immutable.

## DB-ORM-ONE-TO-MANY-010

Collection runtime state será scoped.

## DB-ORM-ONE-TO-MANY-011

Collection semantics será explícita.

## DB-ORM-ONE-TO-MANY-012

SET será distinto de LIST.

## DB-ORM-ONE-TO-MANY-013

Ordered SET será distinto de LIST.

## DB-ORM-ONE-TO-MANY-014

PersistentCollection será distinto de Repository.

## DB-ORM-ONE-TO-MANY-015

PersistentCollection será distinto de Query Builder.

## DB-ORM-ONE-TO-MANY-016

PersistentCollection será distinto de cache.

## DB-ORM-ONE-TO-MANY-017

Initialization será distinta de completeness.

## DB-ORM-ONE-TO-MANY-018

UNINITIALIZED será distinto de EMPTY.

## DB-ORM-ONE-TO-MANY-019

PARTIAL será distinto de COMPLETE.

## DB-ORM-ONE-TO-MANY-020

Collection empty solo probará relación vacía cuando coverage sea completa.

## DB-ORM-ONE-TO-MANY-021

Membership persistente será identity-aware.

## DB-ORM-ONE-TO-MANY-022

NEW entities podrán utilizar temporary UoW identity.

## DB-ORM-ONE-TO-MANY-023

Temporary identity no será PK ficticia.

## DB-ORM-ONE-TO-MANY-024

Identity promotion no duplicará collection membership.

## DB-ORM-ONE-TO-MANY-025

IdentityMap hit reutilizará canonical entity.

## DB-ORM-ONE-TO-MANY-026

SET no contendrá duplicados de la misma canonical identity.

## DB-ORM-ONE-TO-MANY-027

Lazy collection comenzará UNINITIALIZED/UNOBSERVED cuando no haya evidencia.

## DB-ORM-ONE-TO-MANY-028

Lazy loading podrá ser prohibido por policy.

## DB-ORM-ONE-TO-MANY-029

Collection Loader no generará SQL directamente.

## DB-ORM-ONE-TO-MANY-030

Collection Loader utilizará Entity Query/Query Engine.

## DB-ORM-ONE-TO-MANY-031

Filtered query no marcará collection COMPLETE automáticamente.

## DB-ORM-ONE-TO-MANY-032

Relationship Query será distinta de Collection Initialization.

## DB-ORM-ONE-TO-MANY-033

Eager loading será distinto de JOIN.

## DB-ORM-ONE-TO-MANY-034

Eager loading podrá usar JOIN/SELECT/BATCH.

## DB-ORM-ONE-TO-MANY-035

JOIN row count será distinto de entity count.

## DB-ORM-ONE-TO-MANY-036

Hydration deduplicará collection members por identidad.

## DB-ORM-ONE-TO-MANY-037

Cartesian multiplication no duplicará members.

## DB-ORM-ONE-TO-MANY-038

Hydration assembly no ejecutará hidden reads.

## DB-ORM-ONE-TO-MANY-039

Materialized entity será distinta de graph-finalized entity.

## DB-ORM-ONE-TO-MANY-040

postLoad no se ejecutará por cada row físico.

## DB-ORM-ONE-TO-MANY-041

Streaming no expondrá colección como completa antes de una frontera segura.

## DB-ORM-ONE-TO-MANY-042

Unsafe streaming eager collection deberá cambiar estrategia o rechazarse.

## DB-ORM-ONE-TO-MANY-043

CollectionSnapshot será baseline, no live collection.

## DB-ORM-ONE-TO-MANY-044

Partial snapshot no probará ausencia global de miembros.

## DB-ORM-ONE-TO-MANY-045

SET diff completo podrá usar set difference.

## DB-ORM-ONE-TO-MANY-046

Partial collection no utilizará ausencia como remove implícito.

## DB-ORM-ONE-TO-MANY-047

Explicit operations serán first-class.

## DB-ORM-ONE-TO-MANY-048

ADD no requerirá inicialización completa cuando sea seguro.

## DB-ORM-ONE-TO-MANY-049

REMOVE no requerirá inicialización completa cuando sea seguro.

## DB-ORM-ONE-TO-MANY-050

CLEAR será una operación semántica explícita.

## DB-ORM-ONE-TO-MANY-051

CLEAR será distinto de enumerar N removes.

## DB-ORM-ONE-TO-MANY-052

Remove membership será distinto de entity delete.

## DB-ORM-ONE-TO-MANY-053

Orphan Removal será explícito.

## DB-ORM-ONE-TO-MANY-054

Orphan Removal será distinto de Cascade Remove.

## DB-ORM-ONE-TO-MANY-055

Ausencia en colección parcial nunca generará orphan removal.

## DB-ORM-ONE-TO-MANY-056

Cascade Persist registrará NEW members según policy.

## DB-ORM-ONE-TO-MANY-057

NEW member sin cascade/explicit persist fallará antes de persistence insegura.

## DB-ORM-ONE-TO-MANY-058

Cascade Remove no requerirá materialización total si semántica puede preservarse de otra forma.

## DB-ORM-ONE-TO-MANY-059

DB Cascade será distinto de ORM Cascade.

## DB-ORM-ONE-TO-MANY-060

DB Cascade deberá reconciliar managed dependents.

## DB-ORM-ONE-TO-MANY-061

Bidirectional fix-up será cycle-safe.

## DB-ORM-ONE-TO-MANY-062

Bidirectional fix-up no marcará dirty durante hydration baseline.

## DB-ORM-ONE-TO-MANY-063

STRICT policy podrá detectar graph inconsistency.

## DB-ORM-ONE-TO-MANY-064

AUTO_FIXUP no deberá crear loops.

## DB-ORM-ONE-TO-MANY-065

Known owning/inverse contradiction será error.

## DB-ORM-ONE-TO-MANY-066

Unknown inverse collection no generará falsa contradicción.

## DB-ORM-ONE-TO-MANY-067

Reparenting será distinto de orphan removal.

## DB-ORM-ONE-TO-MANY-068

Orphan evaluation ocurrirá después de relationship stabilization.

## DB-ORM-ONE-TO-MANY-069

Generated owner ID será dependency barrier.

## DB-ORM-ONE-TO-MANY-070

Temporary IDs nunca llegarán como FKs reales.

## DB-ORM-ONE-TO-MANY-071

Batch persistence no alterará semantic operations.

## DB-ORM-ONE-TO-MANY-072

OneToManyChangeSet será distinto de SQL.

## DB-ORM-ONE-TO-MANY-073

Inverse collection changes convergerán con owning-side changes.

## DB-ORM-ONE-TO-MANY-074

La misma asociación no producirá double update.

## DB-ORM-ONE-TO-MANY-075

Load ordering será distinto de persistent list positioning.

## DB-ORM-ONE-TO-MANY-076

LIST position será estado persistente solo cuando mapping lo declare.

## DB-ORM-ONE-TO-MANY-077

MOVE será distinto de REMOVE+ADD cuando position sea semántica.

## DB-ORM-ONE-TO-MANY-078

Extra-lazy operations no inicializarán colección por definición.

## DB-ORM-ONE-TO-MANY-079

Extra-lazy count no marcará collection COMPLETE.

## DB-ORM-ONE-TO-MANY-080

Extra-lazy count respetará pending local mutations.

## DB-ORM-ONE-TO-MANY-081

Unknown pending remove no alterará count sin evidencia.

## DB-ORM-ONE-TO-MANY-082

contains podrá utilizar EXISTS sin inicialización.

## DB-ORM-ONE-TO-MANY-083

isEmpty podrá utilizar existence query.

## DB-ORM-ONE-TO-MANY-084

slice será partial view.

## DB-ORM-ONE-TO-MANY-085

Paginated result será distinto de managed collection.

## DB-ORM-ONE-TO-MANY-086

Pagination no marcará relationship COMPLETE.

## DB-ORM-ONE-TO-MANY-087

Large collections no deberán requerir full materialization para operaciones simples.

## DB-ORM-ONE-TO-MANY-088

Memory optimizations no podrán eliminar evidencia necesaria para destructive operations.

## DB-ORM-ONE-TO-MANY-089

Bulk optimization no fingirá per-entity lifecycle.

## DB-ORM-ONE-TO-MANY-090

N+1 telemetry será observable.

## DB-ORM-ONE-TO-MANY-091

Batch loading podrá inicializar múltiples owners.

## DB-ORM-ONE-TO-MANY-092

Batch load mapeará cada member al owner correcto.

## DB-ORM-ONE-TO-MANY-093

Owner sin rows en complete batch load será COMPLETE+EMPTY.

## DB-ORM-ONE-TO-MANY-094

Partial batch failure no marcará todas las collections COMPLETE.

## DB-ORM-ONE-TO-MANY-095

Failed load no convertirá colección en empty.

## DB-ORM-ONE-TO-MANY-096

Cancelled load no publicará colección medio cargada.

## DB-ORM-ONE-TO-MANY-097

Initialization deberá usar staging cuando sea necesario.

## DB-ORM-ONE-TO-MANY-098

Dirty collection refresh tendrá policy explícita.

## DB-ORM-ONE-TO-MANY-099

Default refresh policy será conservadora.

## DB-ORM-ONE-TO-MANY-100

Normal hydration no sobrescribirá pending collection mutations.

## DB-ORM-ONE-TO-MANY-101

Partial load podrá ampliar conocimiento sin declarar completeness.

## DB-ORM-ONE-TO-MANY-102

Complete load deberá reconciliar pending local operations.

## DB-ORM-ONE-TO-MANY-103

DB observation será distinta de logical current collection.

## DB-ORM-ONE-TO-MANY-104

Collection baseline será distinta de current state.

## DB-ORM-ONE-TO-MANY-105

External mutation podrá volver collection knowledge STALE.

## DB-ORM-ONE-TO-MANY-106

UNKNOWN persistence no avanzará baseline como éxito.

## DB-ORM-ONE-TO-MANY-107

Flush success será distinto de transaction commit.

## DB-ORM-ONE-TO-MANY-108

Rollback DB no rebobinará automáticamente collection state.

## DB-ORM-ONE-TO-MANY-109

Detached collection no conservará manager histórico para hidden loading.

## DB-ORM-ONE-TO-MANY-110

Cross-context member incompatible será rechazado.

## DB-ORM-ONE-TO-MANY-111

Tenant boundaries serán preservados.

## DB-ORM-ONE-TO-MANY-112

Shard boundaries serán preservados.

## DB-ORM-ONE-TO-MANY-113

Cross-database relation no implicará distributed transaction.

## DB-ORM-ONE-TO-MANY-114

Shared metadata será immutable en persistent runtime.

## DB-ORM-ONE-TO-MANY-115

Collections managed serán scoped por PersistenceContext.

## DB-ORM-ONE-TO-MANY-116

Snapshots serán scoped.

## DB-ORM-ONE-TO-MANY-117

Operation logs serán scoped.

## DB-ORM-ONE-TO-MANY-118

Initialization guards serán scoped.

## DB-ORM-ONE-TO-MANY-119

No habrá process-global mutable collection state.

## DB-ORM-ONE-TO-MANY-120

Concurrent initialization será coordinada.

## DB-ORM-ONE-TO-MANY-121

Recursive initialization será detectada.

## DB-ORM-ONE-TO-MANY-122

Mutation during initialization no será perdida.

## DB-ORM-ONE-TO-MANY-123

Collection iteration no corromperá índices internos ante mutación.

## DB-ORM-ONE-TO-MANY-124

Authorization será distinto de relationship membership.

## DB-ORM-ONE-TO-MANY-125

Security-filtered completeness será distinguible de physical completeness.

## DB-ORM-ONE-TO-MANY-126

Serialization no provocará lazy loading obligatorio.

## DB-ORM-ONE-TO-MANY-127

Serializer no será relationship loader accidental.

## DB-ORM-ONE-TO-MANY-128

Mapping será validado en bootstrap.

## DB-ORM-ONE-TO-MANY-129

mappedBy deberá apuntar a relación compatible.

## DB-ORM-ONE-TO-MANY-130

One-to-many/many-to-one pair tendrá cardinalidad coherente.

## DB-ORM-ONE-TO-MANY-131

Collection property será validada.

## DB-ORM-ONE-TO-MANY-132

Domain collections podrán integrarse mediante adapter.

## DB-ORM-ONE-TO-MANY-133

No se requerirá necesariamente una base class ORM para domain collections.

## DB-ORM-ONE-TO-MANY-134

Readonly PHP property será distinta de immutable collection semantics.

## DB-ORM-ONE-TO-MANY-135

Collection mutability será explícita cuando se restrinja.

## DB-ORM-ONE-TO-MANY-136

Relationship path queries usarán Semantic Query Engine.

## DB-ORM-ONE-TO-MANY-137

has/doesntHave no generarán SQL dentro del ORM.

## DB-ORM-ONE-TO-MANY-138

withCount no inicializará collection.

## DB-ORM-ONE-TO-MANY-139

Relationship aggregate será distinto de collection initialization.

## DB-ORM-ONE-TO-MANY-140

Duplicate SET add será idempotente.

## DB-ORM-ONE-TO-MANY-141

Duplicate established identity será IdentityMap conflict, no second member.

## DB-ORM-ONE-TO-MANY-142

Remove/re-add podrá normalizarse a no-op.

## DB-ORM-ONE-TO-MANY-143

Add/remove de member no-baseline podrá cancelarse.

## DB-ORM-ONE-TO-MANY-144

Operation normalization será determinista.

## DB-ORM-ONE-TO-MANY-145

CollectionMutationVersion podrá invalidar stale plans.

## DB-ORM-ONE-TO-MANY-146

Lifecycle mutation antes de freeze será recollected.

## DB-ORM-ONE-TO-MANY-147

Frozen PersistencePlan no será mutado silenciosamente.

## DB-ORM-ONE-TO-MANY-148

post lifecycle mutation será trabajo futuro.

## DB-ORM-ONE-TO-MANY-149

Relationship persistence no ejecutará SQL directamente.

## DB-ORM-ONE-TO-MANY-150

Persistence Planner respetará dependency graph.

## DB-ORM-ONE-TO-MANY-151

Collection completeness será una dimensión explícita de conocimiento.

## DB-ORM-ONE-TO-MANY-152

Collection initialization será una dimensión explícita distinta.

## DB-ORM-ONE-TO-MANY-153

Membership mutation intent será distinta de observed DB membership.

## DB-ORM-ONE-TO-MANY-154

Absence from partial view nunca equivaldrá a persisted absence.

## DB-ORM-ONE-TO-MANY-155

No se fabricará orphan por falta de observación.

## DB-ORM-ONE-TO-MANY-156

No se fabricará clean state después de UNKNOWN outcome.

## DB-ORM-ONE-TO-MANY-157

No se fabricará empty state después de failed load.

## DB-ORM-ONE-TO-MANY-158

No se fabricará complete state después de filtered load.

## DB-ORM-ONE-TO-MANY-159

No se fabricará authorization knowledge a partir de relationship knowledge.

## DB-ORM-ONE-TO-MANY-160

VoltStack preservará identidad, membership, completeness, mutation intent, ownership y persistence evidence como dimensiones independientes.

---

# 273. Fórmulas fundamentales

## 273.1 Relación

```text
OneToMany(A, B)
⇒
∀ b ∈ B_relationship:
owner(b) ∈ {A, null}
```

para una relación FK convencional.

---

# 274. Colección completa

```text
CompleteCollection
=
SuccessfulFullRelationshipLoad
+
AllPendingLocalOperationsReconciled
```

---

# 275. Colección vacía conocida

```text
KnownEmpty
=
COMPLETE
∧
LogicalMemberCount = 0
```

---

# 276. Membership actual

```text
LogicalMembership
=
ObservedMembership
⊕
PendingOperations
```

---

# 277. SET Diff

Solo con baseline completo:

```text
Added
=
Current − Baseline
```

```text
Removed
=
Baseline − Current
```

---

# 278. Orphan

```text
TrueOrphan(x)
=
WasDependent(x)
∧
NoCurrentOwner(x)
∧
OrphanRemovalEnabled
```

---

# 279. Reparenting

```text
Reparent(x, A, B)
=
PreviousOwner(x) = A
∧
CurrentOwner(x) = B
∧
A ≠ B
```

y:

```text
Reparent
≠
Orphan
```

---

# 280. Extra-Lazy Safety

```text
SafeExtraLazyOperation
=
DatabaseAnswer
+
LocallyKnownPendingEffects
+
NoUnresolvedMembershipAmbiguity
```

---

# 281. Runtime Safety

```text
SafeCollectionRuntime
=
ImmutableSharedMetadata
∧
ScopedPersistentCollection
∧
ScopedSnapshot
∧
ScopedOperationLog
∧
ScopedLoadGuard
∧
ScopedIdentityMap
∧
NoCrossRequestEntityReferences
```

---

# 282. Master Formula

```text
Database One-to-Many Relationship System
=
OneToMany Metadata
+
Owning/Inverse Semantics
+
PersistentCollection
+
SET/LIST Semantics
+
Initialization State
+
Completeness State
+
Membership Identity
+
Temporary Identity Support
+
IdentityMap Integration
+
Lazy Loading
+
Eager Loading
+
Batch Loading
+
Extra-Lazy Operations
+
Hydration Assembly
+
JOIN Deduplication
+
Cartesian Product Handling
+
Graph Finalization
+
Collection Snapshots
+
Membership Change Tracking
+
Operation Logs
+
Add/Remove/Clear
+
Bidirectional Fix-Up
+
Reparenting
+
Cascade Persist
+
Cascade Remove
+
Orphan Removal
+
Generated Identity Dependencies
+
Collection Ordering
+
Persistent LIST Positions
+
Pagination Semantics
+
Large Collection Governance
+
Refresh/Reconciliation
+
Transaction Awareness
+
Persistent Runtime Isolation
+
Telemetry
+
Diagnostics
+
Testing
```

---

# 283. Regla maestra final

> **VoltStack nunca interpretará una colección one-to-many no inicializada, filtrada, paginada o parcialmente observada como representación total de la relación persistida. Toda decisión destructiva —especialmente removals, clear y orphan removal— deberá estar respaldada por una mutación explícita o por evidencia suficientemente completa del estado de membresía.**

---

# 284. Arquitectura resultante

```text
OneToManyMetadata
        │
        ▼
PersistentCollection
        │
        ├─────────────── Runtime State
        │                ├── initialization
        │                ├── completeness
        │                └── pending operations
        │
        ├─────────────── Relationship Query
        │                       │
        │                       ▼
        │                  Query Engine
        │                       │
        │                       ▼
        │                    Result
        │                       │
        │                       ▼
        │                   Hydration
        │                       │
        │                       ▼
        │                  IdentityMap
        │                       │
        │                       ▼
        │               Collection Assembly
        │
        ├─────────────── Collection Snapshot
        │                       │
        │                       ▼
        │                  Change Tracking
        │                       │
        │                       ▼
        │                Normalized ChangeSet
        │                       │
        ▼                       ▼
Bidirectional Graph ───→ Relationship Persistence
                                │
                                ▼
                       Persistence Planner
                                │
                                ▼
                           Query Engine
                                │
                                ▼
                        Execution Engine
```

La arquitectura mantiene separadas:

```text
collection state
relationship query
DB observation
logical membership
pending mutation
persistence operation
physical SQL
```

evitando uno de los problemas más comunes de los ORM tradicionales: tratar una colección PHP como si siempre fuese una fotografía completa de la base de datos.

---

# 285. Siguiente documento

```text
145_DATABASE_MANY_TO_ONE_RELATIONSHIP_SYSTEM.md
```

El siguiente documento definirá el lado complementario y normalmente owning de las asociaciones basadas en foreign key:

```text
Order.user
Comment.post
Employee.department
Invoice.customer
```

incluyendo:

```text
ManyToOneMetadata
owning-side semantics
foreign-key identity
nullable/required references
entity references
lazy loading
eager loading
batch loading
IdentityMap reuse
hydration
relationship snapshots
assignment
nullification
reparenting
inverse one-to-many synchronization
cascade persist
generated identifiers
foreign-key persistence
optimistic locking interaction
tenant/shard isolation
partial hydration
filtered queries
failure reconciliation
persistent runtime
telemetry
testing
```

La regla central será:

> **Una relación many-to-one de VoltStack será una referencia identity-aware hacia como máximo una entidad target y, en el modelo FK convencional, constituirá la autoridad ORM que determina el valor persistente de la foreign key; su estado cargado, su identidad conocida y su valor `NULL` permanecerán como conceptos independientes.**