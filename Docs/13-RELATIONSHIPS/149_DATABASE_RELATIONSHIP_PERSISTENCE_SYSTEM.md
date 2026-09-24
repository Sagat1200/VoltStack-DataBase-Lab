# 149_DATABASE_RELATIONSHIP_PERSISTENCE_SYSTEM.md

# VoltStack Quantum Database
## Database Relationship Persistence System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 149 — Database Relationship Persistence System  
**Bloque:** 13 — Relationships  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Relationship Persistence System` define cómo VoltStack transforma cambios semánticos realizados sobre asociaciones entre entidades en operaciones de persistencia tipadas, ordenables, verificables y reconciliables.

El sistema conecta:

```text
Entity Relationship State
        +
RelationshipMetadata
        +
Relationship Snapshots
        +
Change Tracking
        +
UnitOfWork
        ↓
Relationship ChangeSets
        ↓
Relationship Persistence Operations
        ↓
Persistence Planner
        ↓
PersistencePlan
        ↓
Query Models
        ↓
Execution Engine
```

sin convertir las relaciones ORM en SQL directo.

El sistema deberá soportar:

- One-to-One;
- One-to-Many;
- Many-to-One;
- Many-to-Many;
- relaciones polimórficas;
- owning/inverse sides;
- cascade persist;
- cascade remove;
- orphan removal;
- join tables;
- generated identifiers;
- relationship ordering;
- optimistic locking;
- partial collections;
- flush stabilization;
- batch persistence;
- transaction participation;
- rollback reconciliation;
- outcomes parciales;
- outcomes `UNKNOWN`;
- persistent runtimes.

---

# 2. Regla central

> **Relationship Persistence transforma cambios semánticos confirmados de asociaciones en operaciones de persistencia tipadas; nunca convierte directamente mutaciones de objetos en SQL ni permite que el lado inverso cree una segunda fuente de verdad persistente.**

Formalmente:

```text
Object Relationship Mutation
        ↓
Change Tracking
        ↓
RelationshipChangeSet
        ↓
Relationship Persistence
        ↓
Typed PersistenceOperation
        ↓
Persistence Planner
        ↓
Query Model
        ↓
Query Engine
        ↓
SQL Compiler
```

Nunca:

```text
$post->author = $user
        ↓
"UPDATE posts SET author_id = ..."
```

---

# 3. Posición arquitectónica

```text
Entity / Model API
        ↓
Entity State
        ↓
Relationship State
        ↓
Change Tracking
        ↓
UnitOfWork
        ↓
┌─────────────────────────────────────┐
│ Relationship Persistence System     │
│                                     │
│ To-One Changes                      │
│ To-Many Changes                     │
│ Join Table Changes                  │
│ Polymorphic Changes                 │
│ Cascade Resolution                  │
│ Orphan Resolution                   │
│ Dependency Extraction               │
└─────────────────────────────────────┘
        ↓
Persistence Planner
        ↓
Persistence Engine
        ↓
Query Engine
        ↓
Execution Engine
```

---

# 4. Relationship Persistence ≠ Relationship Metadata

`RelationshipMetadata` describe:

```text
Post.author
    kind       = MANY_TO_ONE
    owning     = true
    target     = User
    join       = author_id → users.id
```

Relationship Persistence procesa un cambio concreto:

```text
Post#42.author

User#5
   ↓
User#9
```

Por tanto:

```text
RelationshipMetadata
≠
RelationshipChange
```

---

# 5. Relationship Persistence ≠ Change Tracking

Change Tracking responde:

> ¿Qué cambió?

Relationship Persistence responde:

> ¿Qué operaciones semánticas de persistencia corresponden a ese cambio?

---

# 6. Relationship Persistence ≠ UnitOfWork

UnitOfWork coordina el trabajo ORM completo.

Relationship Persistence se especializa en asociaciones.

```text
UnitOfWork
├── Entity field changes
├── Inserts
├── Updates
├── Deletes
└── Relationship changes
```

---

# 7. Relationship Persistence ≠ Persistence Planner

Relationship Persistence produce operaciones y dependencias.

Planner decide:

```text
ordering
barriers
batching
cycle handling
execution phases
```

---

# 8. Relationship Persistence ≠ SQL Compiler

Nunca deberá contener:

```php
$sql = 'INSERT INTO role_user ...';
```

Debe producir algo como:

```php
new AddJoinTableMembershipOperation(...);
```

---

# 9. Relationship Persistence ≠ Database Transaction

Una relación puede persistirse dentro de una transacción, pero:

```text
Relationship Persistence
≠
Transaction Manager
```

---

# 10. Relationship Persistence ≠ Commit

```text
Relationship persistence success
≠
transaction committed
```

---

# 11. Relationship Persistence ≠ In-Memory Synchronization

Actualizar:

```php
$post->setAuthor($user);

$user->posts->add($post);
```

es sincronización del object graph.

Persistir la asociación es otro proceso.

---

# 12. Relationship Persistence ≠ Relationship Loading

Loading:

```text
Database
→ Object Graph
```

Persistence:

```text
Object Graph Changes
→ Database Operations
```

Son direcciones opuestas.

---

# 13. Fuente de verdad persistente

Cada relación deberá tener una semántica persistente no ambigua.

Para asociaciones bidireccionales:

```text
Owning Side
=
Persistent Source of Truth
```

Ejemplo:

```text
User.posts
    ↕
Post.author
```

si `Post.author` posee `author_id`:

```text
Post.author = OWNING
User.posts  = INVERSE
```

---

# 14. Inverse Side

El lado inverso puede:

- mejorar navegación;
- mantener consistencia del object graph;
- ayudar a detectar cambios;
- participar en diagnostics.

Pero no crea una segunda operación persistente independiente.

---

# 15. Regla de ownership

```text
PersistentRelationshipDecision
=
OwningSideState
```

por defecto.

---

# 16. Relationship Synchronization Policy

De acuerdo con el documento 148:

```php
enum RelationshipSynchronizationPolicy
{
    case OWNING_SIDE_AUTHORITATIVE;
    case SYNCHRONIZE_BOTH_SIDES;
    case STRICT_BIDIRECTIONAL;
}
```

---

# 17. Política recomendada V1

VoltStack utilizará:

```text
OWNING_SIDE_AUTHORITATIVE
```

como comportamiento base.

Podrá ofrecer helpers que mantengan ambos lados sincronizados.

---

# 18. Contradicciones bidireccionales

Ejemplo:

```php
$post->setAuthor($alice);

$bob->posts->add($post);
```

El estado es contradictorio.

VoltStack no deberá generar:

```text
two competing persistence operations
```

---

# 19. Resolución de contradicción

Según política:

```text
OWNING_SIDE_AUTHORITATIVE
    → Post.author wins
    → diagnostic optional

STRICT_BIDIRECTIONAL
    → flush fails

SYNCHRONIZE_BOTH_SIDES
    → controlled reconciliation
```

---

# 20. Modelo general de cambio

```php
interface RelationshipChangeSet
{
    public function relationship(): RelationshipId;

    public function owner(): EntityReference;

    public function kind(): RelationshipChangeKind;
}
```

---

# 21. RelationshipChangeKind

```php
enum RelationshipChangeKind
{
    case REFERENCE_ASSIGNED;
    case REFERENCE_REPLACED;
    case REFERENCE_CLEARED;

    case MEMBER_ADDED;
    case MEMBER_REMOVED;

    case COLLECTION_REPLACED;
    case COLLECTION_REORDERED;

    case ORPHANED;
}
```

---

# 22. ChangeSet ≠ SQL

Ejemplo:

```text
REFERENCE_REPLACED

relationship:
    Post.author

owner:
    Post#42

old:
    User#5

new:
    User#9
```

No contiene SQL.

---

# 23. To-One ChangeSet

Propuesta:

```php
final readonly class ToOneRelationshipChangeSet
    implements RelationshipChangeSet
{
    public function __construct(
        public RelationshipId $relationship,
        public EntityReference $owner,
        public ?EntityReference $before,
        public ?EntityReference $after,
    ) {}
}
```

---

# 24. To-Many ChangeSet

```php
final readonly class ToManyRelationshipChangeSet
    implements RelationshipChangeSet
{
    /**
     * @param list<EntityReference> $added
     * @param list<EntityReference> $removed
     */
    public function __construct(
        public RelationshipId $relationship,
        public EntityReference $owner,
        public array $added,
        public array $removed,
        public CollectionCoverage $coverage,
    ) {}
}
```

---

# 25. Membership identity

Los miembros deberán compararse mediante identidad ORM:

```text
EntityKey
```

y no mediante:

```php
$a == $b;
```

---

# 26. Temporary identity

Entidades `NEW` todavía sin ID persistente necesitan identidad temporal dentro del UoW.

Conceptualmente:

```text
RelationshipMemberKey
=
Established EntityKey
OR
UnitOfWorkTemporaryIdentity
```

---

# 27. Temporary identity ≠ persistent identity

Nunca deberá escribirse una identidad temporal en DB.

Solo sirve para:

- graph traversal;
- deduplication;
- dependency tracking;
- membership tracking.

---

# 28. Relationship Snapshot

El baseline puede almacenar:

### To-One

```text
EntityKey|null
```

### To-Many

```text
Set<EntityKey>
```

o representación equivalente.

---

# 29. Snapshot ≠ live collection

El snapshot representa el baseline ORM conocido.

No una referencia mutable a la colección actual.

---

# 30. Mutable collection snapshot

Anti-pattern:

```php
$snapshot = $entity->roles;
```

si ambas referencias apuntan al mismo objeto mutable.

---

# 31. Canonical snapshot

Preferido:

```text
MembershipSnapshot
=
CanonicalMembershipKeys
```

---

# 32. Loaded coverage

Toda colección deberá conservar información de cobertura.

Ejemplo:

```php
enum CollectionCoverage
{
    case UNINITIALIZED;
    case PARTIAL;
    case COMPLETE;
}
```

---

# 33. COMPLETE

Significa:

> El ORM conoce todos los miembros pertenecientes a la definición estructural de esta relación para el contexto correspondiente.

---

# 34. PARTIAL

Significa:

> El ORM conoce algunos miembros, pero no puede afirmar que el conjunto observado sea completo.

---

# 35. UNINITIALIZED

Significa:

> El contenido persistente de la relación no ha sido materializado como colección.

---

# 36. EMPTY ≠ UNINITIALIZED

```text
EMPTY + COMPLETE
```

significa colección conocida sin elementos.

```text
UNINITIALIZED
```

significa desconocimiento.

---

# 37. Persistencia segura de colecciones parciales

VoltStack nunca interpretará:

```text
member not present in partial collection
```

como:

```text
member removed
```

---

# 38. Regla formal

```text
AbsenceInPartialCollection
≠
ConfirmedRemoval
```

---

# 39. Explicit mutation tracking

Una colección parcial sí puede registrar:

```text
explicit add
explicit remove
```

sin afirmar conocimiento completo.

---

# 40. Relationship PersistentCollection

Puede existir:

```php
interface PersistentRelationshipCollection
{
    public function coverage(): CollectionCoverage;

    public function additions(): iterable;

    public function removals(): iterable;

    public function isDirty(): bool;
}
```

---

# 41. PersistentCollection ≠ metadata

Es estado scoped runtime.

---

# 42. PersistentCollection ≠ generic Collection

Una colección ORM persistente añade conocimiento de:

```text
baseline
coverage
explicit mutations
initialization
owner
relationship
```

---

# 43. Dirty collection

Una colección puede estar dirty aunque la entidad no tenga scalar field changes.

---

# 44. Relationship dirtiness

Por tanto:

```text
EntityScalarDirty
≠
RelationshipDirty
```

---

# 45. Relationship Change Detection

Pipeline:

```text
Relationship Metadata
        +
Relationship Snapshot
        +
Current Relationship State
        +
Explicit Mutation Log
        ↓
Relationship Change Detector
        ↓
RelationshipChangeSet
```

---

# 46. Detection strategies

VoltStack podrá soportar:

```text
SNAPSHOT_DIFF
MUTATION_TRACKING
HYBRID
```

---

# 47. Snapshot Diff

Adecuado principalmente para colecciones `COMPLETE`.

```text
added   = Current - Snapshot
removed = Snapshot - Current
```

---

# 48. Mutation Tracking

Registra directamente:

```text
add(User#10)
remove(User#4)
```

---

# 49. Hybrid

Puede utilizar:

```text
explicit mutation log
+
snapshot validation
```

cuando la colección está completamente cargada.

---

# 50. ChangeSet canonicalization

Operaciones redundantes deberán estabilizarse.

Ejemplo:

```text
add X
remove X
```

dentro del mismo baseline puede cancelarse según estado inicial.

---

# 51. Membership state machine

Para un miembro:

```text
ABSENT
  │ add
  ▼
ADDED
  │ remove
  ▼
ABSENT
```

mientras:

```text
PRESENT
  │ remove
  ▼
REMOVED
  │ add
  ▼
PRESENT
```

---

# 52. Mutation normalization

Esto evita producir operaciones innecesarias.

---

# 53. Relationship Persistence Operations

El sistema deberá producir operaciones tipadas.

Jerarquía conceptual:

```text
RelationshipPersistenceOperation
├── SetToOneReferenceOperation
├── ClearToOneReferenceOperation
├── AddRelationshipMemberOperation
├── RemoveRelationshipMemberOperation
├── AddJoinTableMembershipOperation
├── RemoveJoinTableMembershipOperation
├── UpdateJoinTableMembershipOperation
├── ReorderRelationshipOperation
├── OrphanRemovalOperation
└── PolymorphicRelationshipOperation
```

---

# 54. PersistenceOperation ≠ QueryModel

La operación todavía expresa intención ORM.

Después:

```text
RelationshipPersistenceOperation
        ↓
Persistence Planner / Operation Translator
        ↓
Query Model
```

---

# 55. SetToOneReferenceOperation

```php
final readonly class SetToOneReferenceOperation
    implements RelationshipPersistenceOperation
{
    public function __construct(
        public RelationshipId $relationship,
        public EntityReference $owner,
        public EntityReference $target,
    ) {}
}
```

---

# 56. ClearToOneReferenceOperation

```php
final readonly class ClearToOneReferenceOperation
    implements RelationshipPersistenceOperation
{
    public function __construct(
        public RelationshipId $relationship,
        public EntityReference $owner,
    ) {}
}
```

---

# 57. To-One nullability

`ClearToOneReferenceOperation` solo será válido si:

```text
RelationshipNullability = NULLABLE
```

o si otra operación del plan elimina/reemplaza la entidad de manera que el estado intermedio sea válido según constraints/capabilities.

---

# 58. Many-to-One example

Estado inicial:

```text
Post#42.author = User#5
```

Estado final:

```text
Post#42.author = User#9
```

ChangeSet:

```text
before = User#5
after  = User#9
```

Operación:

```text
SetToOneReference(
    Post#42,
    Post.author,
    User#9
)
```

---

# 59. Query translation

Posteriormente:

```text
UpdateQueryModel(
    table = posts,
    set = author_id → User#9.id,
    where = Post#42 identity predicate
)
```

El SQL Compiler decidirá SQL concreto.

---

# 60. Many-to-One with NEW target

Ejemplo:

```php
$newUser = new User(...);

$post->setAuthor($newUser);

$em->persist($newUser);
$em->flush();
```

Si `User.id` es generado:

```text
Insert User
    ↓
Generated User ID
    ↓
Set Post.author reference
```

---

# 61. Identity dependency

Planner deberá representar:

```text
Post.author update
depends on
User insert identity resolution
```

---

# 62. Generated identity barrier

```text
TargetIdentityRequired
∧
TargetIdentityNotEstablished
→
DependencyBarrier
```

---

# 63. Cascade Persist

Si metadata declara:

```text
cascade persist
```

entonces:

```php
$post->setAuthor($newUser);

$em->persist($post);
```

podrá registrar automáticamente `$newUser` como `NEW`.

---

# 64. Cascade Persist ≠ Insert

Cascade persist significa:

```text
propagate persistence registration
```

no:

```text
execute INSERT now
```

---

# 65. Non-cascaded NEW entity

Si:

```text
Post.author
does not cascade persist
```

y referencia una entidad NEW no registrada:

```text
flush
→ error
```

por defecto.

---

# 66. Error recomendado

```text
TransientRelationshipTargetException
```

con diagnóstico:

```text
Post#42.author references NEW User
but cascade persist is disabled
and User is not scheduled for persistence.
```

---

# 67. One-to-One persistence

One-to-One utiliza to-one reference semantics más constraints de uniqueness/ownership.

---

# 68. Owning one-to-one

Ejemplo:

```text
User.profile
    FK users.profile_id
```

Cambio:

```text
Profile#5
→
Profile#8
```

genera una actualización de referencia del owner.

---

# 69. Inverse one-to-one

Cambiar únicamente el inverse side no deberá generar una segunda actualización independiente.

---

# 70. One-to-one uniqueness

El planner deberá considerar que el target puede estar limitado a un owner.

Ejemplo:

```text
UserA.profile = Profile#1
UserB.profile = Profile#1
```

puede violar la semántica one-to-one antes incluso del SQL.

---

# 71. In-memory uniqueness detection

Cuando el UoW conoce ambos cambios podrá detectar conflictos tempranamente.

---

# 72. Database uniqueness remains authoritative

La validación ORM no sustituye:

```text
UNIQUE constraint
```

en DB cuando la integridad debe garantizarse concurrentemente.

---

# 73. One-to-Many persistence

Un `OneToMany` suele ser inverse de un `ManyToOne`.

Ejemplo:

```text
User.posts
↔
Post.author
```

---

# 74. Adding to inverse collection

```php
$user->posts->add($post);
```

por sí solo no deberá crear una persistencia ambigua.

---

# 75. Helper synchronization

El API de dominio puede implementar:

```php
public function addPost(Post $post): void
{
    $this->posts->add($post);
    $post->setAuthor($this);
}
```

Entonces el owning side queda correctamente actualizado.

---

# 76. Unsynchronized inverse mutation

Si solo:

```text
User.posts += Post#42
```

pero:

```text
Post#42.author ≠ User
```

la política decidirá:

```text
warning
error
controlled synchronization
```

---

# 77. One-to-Many direct ownership

VoltStack podrá soportar mappings donde el one-to-many tenga representación persistente propia si el modelo de metadata lo permite.

Pero no deberá asumirse como default.

---

# 78. Many-to-Many persistence

Many-to-many normalmente persiste membership en una join table.

Ejemplo:

```text
User.roles
↔
Role.users
```

---

# 79. Membership addition

```php
$user->roles->add($admin);
```

ChangeSet:

```text
added = Role#1
```

Operación:

```text
AddJoinTableMembership(
    User#42,
    User.roles,
    Role#1
)
```

---

# 80. Membership removal

```text
RemoveJoinTableMembership(
    User#42,
    User.roles,
    Role#1
)
```

---

# 81. Join-table query model

Posteriormente:

```text
InsertQueryModel
    role_user
    user_id = 42
    role_id = 1
```

o:

```text
DeleteQueryModel
    role_user
    where user_id = 42
      and role_id = 1
```

---

# 82. Duplicate membership

Para collection semantics `SET_LIKE`:

```text
add same EntityKey twice
```

deberá normalizarse a una sola membership.

---

# 83. Database uniqueness

Se recomienda además:

```text
UNIQUE(user_id, role_id)
```

cuando corresponda.

---

# 84. Join-table membership identity

Conceptualmente:

```text
MembershipKey
=
RelationshipId
×
OwnerEntityKey
×
TargetEntityKey
```

---

# 85. MembershipKey

```php
final readonly class RelationshipMembershipKey
{
    public function __construct(
        public RelationshipId $relationship,
        public EntityKey $owner,
        public EntityKey $target,
    ) {}
}
```

---

# 86. Composite identifiers

MembershipKey deberá soportar:

```text
composite owner ID
+
composite target ID
```

sin concatenaciones inseguras de strings.

---

# 87. NEW owner/target in many-to-many

Si owner o target tiene generated identity:

```text
Insert Owner
Insert Target
      ↓
Resolve identities
      ↓
Insert join-table membership
```

---

# 88. Many-to-many dependency graph

```text
Owner Insert ─────┐
                  ├──→ Join Membership Insert
Target Insert ────┘
```

---

# 89. Removing membership

Normalmente no requiere eliminar target.

```text
remove Role from User.roles
≠
delete Role
```

---

# 90. Cascade Remove and Many-to-Many

Debe utilizarse con extrema precaución.

Eliminar un `User` puede eliminar:

```text
join memberships
```

pero no necesariamente:

```text
Role entities
```

---

# 91. Join cleanup ≠ cascade target deletion

Diferenciar:

```text
DELETE relationship rows
```

de:

```text
DELETE target entities
```

---

# 92. Association Entity

Si join table contiene dominio propio:

```text
user_id
role_id
assigned_at
assigned_by
expires_at
```

se recomienda:

```text
UserRoleAssignment entity
```

en lugar de ocultarlo como many-to-many simple.

---

# 93. Pivot payload

Si VoltStack permite payload simple de pivot, sus cambios deberán convertirse en operaciones tipadas independientes.

Nunca manipular arrays arbitrarios sin metadata.

---

# 94. Polymorphic To-One Persistence

Ejemplo:

```text
Comment.commentable
→ Post#10
```

persistirá conceptualmente:

```text
commentable_type = "post"
commentable_id   = 10
```

---

# 95. Morph alias

Nunca:

```text
commentable_type = Post::class
```

por default.

Debe utilizarse:

```text
MorphTypeRegistry
```

---

# 96. Polymorphic operation

```php
final readonly class SetPolymorphicReferenceOperation
    implements RelationshipPersistenceOperation
{
    public function __construct(
        public RelationshipId $relationship,
        public EntityReference $owner,
        public MorphType $type,
        public EntityReference $target,
    ) {}
}
```

---

# 97. Morph type and identity atomicity

El par:

```text
(type, id)
```

forma una referencia lógica.

No deberá reconciliarse parcialmente.

---

# 98. Polymorphic reference formula

```text
PolymorphicReference
=
MorphType
×
CanonicalTargetIdentifier
```

---

# 99. Unknown morph type

Nunca se persistirá un alias no registrado.

---

# 100. Polymorphic target validation

Antes del plan:

```text
Target EntityType
∈
AllowedPolymorphicTargets
```

---

# 101. Polymorphic Many-to-Many

Membership puede requerir:

```text
owner identity
target identity
target morph type
```

como clave.

---

# 102. Polymorphic MembershipKey

Conceptualmente:

```text
RelationshipId
×
OwnerEntityKey
×
MorphType
×
TargetEntityKey
```

---

# 103. Cascade traversal

Cascade persist/remove requiere recorrer relationship graph.

---

# 104. Cascade graph

Ejemplo:

```text
Order
├── items
│   ├── Product
│   └── adjustments
└── payment
```

---

# 105. Cycle protection

El traversal deberá mantener:

```text
VisitedObjectIdentitySet
```

y/o canonical EntityKeys.

---

# 106. Cascade ≠ recursion without bounds

Nunca:

```php
foreach ($relations as $relation) {
    cascade($relation); // without visited set
}
```

---

# 107. Cascade scheduling

Cascade deberá registrar trabajo en UoW.

No ejecutarlo.

---

# 108. Cascade Persist algorithm

Conceptualmente:

```text
persist(root)
    ↓
visit cascade-persist relationships
    ↓
for each target:
    if UNTRACKED:
        register NEW/MANAGED according to identity semantics
    ↓
continue traversal
```

---

# 109. Cascade Remove algorithm

```text
remove(root)
    ↓
visit cascade-remove relationships
    ↓
schedule semantic removal
```

sin SQL inmediato.

---

# 110. Cascade timing

Debe definirse claramente si traversal ocurre:

- durante `persist/remove`;
- durante flush collection;
- o ambos mediante stabilization.

VoltStack puede registrar inicialmente y volver a estabilizar durante flush.

---

# 111. Flush stabilization

Lifecycle callbacks o domain-level ORM hooks pueden modificar relaciones durante `preFlush`/`preUpdate`.

Por ello:

```text
Collect
→ Detect relationship changes
→ Lifecycle callbacks
→ Recompute
→ Stabilize
→ Freeze workset
```

---

# 112. Bounded stabilization

Nunca permitir:

```text
infinite callback mutation loop
```

---

# 113. Stabilization formula

```text
Stable
⇔
NoNewEntityWork
∧
NoNewRelationshipChanges
∧
NoRequiredRecomputation
```

---

# 114. Maximum passes

Una policy configurable deberá limitar el número de iteraciones.

Si no converge:

```text
FlushStabilizationException
```

---

# 115. Orphan Removal

Orphan removal responde a:

> ¿La eliminación de una asociación implica que una entidad dependiente debe ser eliminada?

---

# 116. Example

```text
Order
└── OrderItem
```

si:

```php
$order->items->remove($item);
```

y:

```text
orphanRemoval = true
```

entonces puede generarse:

```text
RemoveEntity(OrderItem#5)
```

---

# 117. Orphan removal ≠ collection removal

Sin orphan removal:

```text
remove membership
```

solo cambia asociación.

Con orphan removal:

```text
remove membership
+
schedule entity removal
```

según mapping.

---

# 118. Orphan detection

Solo deberá ejecutarse con evidencia suficiente.

---

# 119. Partial collection danger

Si colección está `PARTIAL`:

```text
missing child
```

no significa orphan.

---

# 120. Explicit removal from partial collection

Sí puede ser evidencia de orphan si:

```text
removal was explicitly recorded
```

y metadata permite orphan removal.

---

# 121. Shared orphan protection

Si target puede estar asociado a múltiples owners, orphan removal automático puede ser inseguro.

Metadata validation deberá impedir configuraciones ambiguas.

---

# 122. Orphan scheduling

Orphan removal produce:

```text
semantic RemoveEntityOperation
```

que entra al mismo Persistence Planner.

---

# 123. Orphan removal and NEW entity

Si un child `NEW` es añadido y luego removido antes del flush:

```text
insert may be cancelled
```

en lugar de:

```text
INSERT
DELETE
```

---

# 124. Cancellation optimization

Esto es una optimización semánticamente segura cuando ninguna otra dependencia requiere el child.

---

# 125. Relationship ordering

Algunas relaciones `LIST_LIKE` pueden persistir posición.

Ejemplo:

```text
Playlist.tracks
position
```

---

# 126. Ordered relationship metadata

Debe declarar explícitamente:

```text
order column / association order semantics
```

si el orden es persistente.

---

# 127. Display ordering ≠ persisted ordering

```text
ORDER BY created_at
```

no significa que reordenar la colección deba escribir posiciones.

---

# 128. Persisted ordering

Si existe:

```text
position column
```

un reorder puede producir:

```text
RelationshipReorderChangeSet
```

---

# 129. Reorder operation

```php
final readonly class ReorderRelationshipOperation
{
    /**
     * @param list<RelationshipMemberPosition> $positions
     */
    public function __construct(
        public RelationshipId $relationship,
        public EntityReference $owner,
        public array $positions,
    ) {}
}
```

---

# 130. Reorder optimization

Solo las posiciones realmente modificadas deberían persistirse cuando sea seguro.

---

# 131. Reorder with partial collection

No se puede inferir orden global desde una colección parcial.

---

# 132. Rule

```text
GlobalReorder
requires
CompleteOrderingKnowledge
```

salvo operación explícita especializada con semántica diferente.

---

# 133. Relationship Persistence Planner Input

El subsistema produce:

```php
final readonly class RelationshipPersistenceWork
{
    /**
     * @param list<RelationshipPersistenceOperation> $operations
     * @param list<PersistenceDependency> $dependencies
     */
    public function __construct(
        public array $operations,
        public array $dependencies,
    ) {}
}
```

---

# 134. Dependency types

Ejemplos:

```text
REQUIRES_OWNER_IDENTITY
REQUIRES_TARGET_IDENTITY
REQUIRES_PARENT_INSERT
REQUIRES_CHILD_INSERT
REQUIRES_REFERENCE_CLEAR
REQUIRES_JOIN_DELETE
REQUIRES_ENTITY_DELETE
REQUIRES_VERSION_CHECK
```

---

# 135. Planner responsibilities

Planner decidirá orden final.

Ejemplo:

```text
clear old FK
→ delete old dependent
→ insert new dependent
→ assign new FK
```

si mapping/capabilities lo requieren.

---

# 136. No universal ordering

Nunca asumir:

```text
all inserts
→ all relationship updates
→ all deletes
```

para todos los grafos.

---

# 137. Dependency DAG

```text
Relationship Operations
        ↓
Dependency Graph
        ↓
Topological Planning
        ↓
PersistencePlan
```

---

# 138. Cycles

Ejemplo:

```text
A.b → B
B.a → A
```

con IDs generados y FKs no-null.

Puede ser imposible sin:

- deferred constraints;
- assigned IDs;
- nullable intermediate FK;
- two-phase persistence;
- platform-specific capability.

---

# 139. Cycle handling

Relationship Persistence describe dependencias.

Planner + Platform Capabilities deciden si son satisfacibles.

---

# 140. Unsatisfiable cycle

Debe producir error explícito.

Nunca romper arbitrariamente una relación.

---

# 141. Transaction requirement

Algunos relationship plans deberán indicar:

```text
transaction recommended/required
```

pero este sistema no inicia/commitea arbitrariamente una transacción.

---

# 142. Flush and transactions

Si EntityManager ya participa en transacción:

```text
flush
→ uses transaction
→ does not commit
```

---

# 143. Atomicity expectations

Una operación relationship compleja puede requerir múltiples statements.

Ejemplo:

```text
remove old one-to-one reference
insert new entity
assign new reference
```

El plan puede requerir atomic execution.

---

# 144. Statement success ≠ relationship durable

```text
join row INSERT succeeded
≠
transaction committed
```

---

# 145. Relationship operation outcomes

```php
enum RelationshipPersistenceOutcomeStatus
{
    case SUCCESS;
    case FAILED;
    case UNKNOWN;
    case SKIPPED;
    case NOT_EXECUTED;
}
```

---

# 146. Outcome object

```php
final readonly class RelationshipPersistenceOutcome
{
    public function __construct(
        public RelationshipPersistenceOperationId $operation,
        public RelationshipPersistenceOutcomeStatus $status,
        public PersistenceEvidence $evidence,
    ) {}
}
```

---

# 147. SUCCESS

Significa que la operación tiene evidencia suficiente de éxito en el nivel correspondiente.

No significa commit durable.

---

# 148. FAILED

Existe evidencia suficiente de que la operación no se completó exitosamente.

---

# 149. UNKNOWN

No existe evidencia suficiente para afirmar:

```text
success
```

ni:

```text
failure
```

---

# 150. UNKNOWN example

```text
join-table INSERT sent
connection lost
```

No deberá asumirse:

```text
membership absent
```

ni:

```text
membership present
```

---

# 151. UNKNOWN rule

> **VoltStack nunca fabricará conocimiento de relationship state cuando el resultado de persistencia sea incierto.**

---

# 152. Partial outcomes

Un flush puede ejecutar:

```text
Operation A = SUCCESS
Operation B = SUCCESS
Operation C = UNKNOWN
Operation D = NOT_EXECUTED
```

El sistema debe preservar esta información.

---

# 153. No fake rollback

Si DB no confirmó rollback:

```text
UNKNOWN
```

permanece `UNKNOWN`.

---

# 154. Relationship consistency

Después de outcomes inciertos:

```text
RelationshipKnowledgeStatus
```

podrá ser:

```php
enum RelationshipKnowledgeStatus
{
    case CONSISTENT;
    case STALE;
    case UNCERTAIN;
    case UNKNOWN;
}
```

---

# 155. PersistenceContext taint

Un `UNKNOWN` significativo puede:

```text
TAINT EntityManager/PersistenceContext
```

para evitar continuar como si el baseline fuera fiable.

---

# 156. Snapshot reconciliation

Solo se actualizarán relationship snapshots cuando exista evidencia suficiente.

---

# 157. Successful to-one reconciliation

Después de éxito suficientemente cierto:

```text
Snapshot(Post.author)
=
EntityKey(User#9)
```

---

# 158. Successful membership addition

```text
MembershipSnapshot
+= Role#1
```

solo cuando la operación correspondiente sea reconciliable.

---

# 159. UNKNOWN reconciliation

Nunca:

```text
UNKNOWN outcome
→ update snapshot as success
```

---

# 160. FAILED reconciliation

Tampoco se deberá actualizar baseline como si hubiera éxito.

---

# 161. In-memory state after failure

VoltStack no deberá intentar una "máquina del tiempo" sobre el object graph.

Si:

```php
$post->setAuthor($newUser);
```

y flush falla, el objeto puede continuar apuntando a `$newUser`.

---

# 162. Database state vs object state

```text
DatabaseReality
≠
CurrentObjectGraph
≠
ORMKnowledge
```

deben mantenerse conceptualmente separados.

---

# 163. Rollback

Si una transacción se revierte:

```text
database relationship state
```

puede regresar al baseline previo.

Pero el object graph no se rebobina automáticamente.

---

# 164. Rollback reconciliation policy

Opciones futuras:

```text
MARK_DIRTY
MARK_STALE
REQUIRE_REFRESH
DETACH_AFFECTED
TAINT_CONTEXT
```

según tipo de rollback/outcome.

---

# 165. Recommended V1

Con rollback conocido:

```text
preserve object graph
+
restore/retain appropriate known DB baseline
+
mark differences dirty/stale
```

cuando pueda hacerse sin fabricar conocimiento.

---

# 166. Unknown transaction completion

Si:

```text
COMMIT sent
connection lost
```

las relaciones afectadas deberán quedar:

```text
UNCERTAIN / UNKNOWN
```

y el contexto probablemente `TAINTED`.

---

# 167. Optimistic locking

Relationship persistence puede interactuar con versioning de la entidad owner.

---

# 168. To-One owner update

Si cambiar:

```text
Post.author
```

modifica la fila `posts`, el update podrá incluir:

```text
WHERE id = ?
AND version = ?
```

y actualizar version.

---

# 169. Join-table mutation

Una membership many-to-many no necesariamente modifica la version del owner.

La policy deberá ser explícita.

---

# 170. Relationship version policy

Posible:

```php
enum RelationshipVersioningPolicy
{
    case NONE;
    case OWNER;
    case TARGET;
    case BOTH;
    case ASSOCIATION;
}
```

---

# 171. Default

No deberá inventarse incremento de version para toda relación.

Debe derivarse de mapping/persistence semantics.

---

# 172. Concurrent collection changes

Ejemplo:

```text
Transaction A adds Role#1
Transaction B adds Role#2
```

pueden coexistir si join table lo permite.

Un "replace all collection" ingenuo puede perder cambios.

---

# 173. Delta persistence preferred

Para many-to-many:

```text
INSERT added memberships
DELETE removed memberships
```

es preferible a:

```text
DELETE ALL
INSERT CURRENT
```

---

# 174. Why replace-all is dangerous

Puede:

- perder cambios concurrentes;
- generar write amplification;
- romper audit;
- incrementar locks;
- generar transient inconsistency.

---

# 175. Collection replacement

`COLLECTION_REPLACED` deberá normalizarse a deltas cuando exista baseline completo.

---

# 176. Partial replacement

No se podrá calcular delta total desde colección parcial sin información adicional.

---

# 177. Direct bulk relationship mutation

VoltStack podrá ofrecer APIs futuras:

```text
attach
detach
sync
```

pero deberán distinguirse de manipular una colección managed normal.

---

# 178. Bulk Relationship API

Ejemplo conceptual:

```php
$user->rolesRelation()
    ->attach($roleId);
```

podrá producir operación directa.

---

# 179. Direct operation implications

Debe definir qué ocurre con:

```text
loaded User.roles collection
relationship snapshot
UoW state
```

---

# 180. Managed-state staleness

Una operación directa que omita la colección managed puede marcarla:

```text
STALE
```

---

# 181. Never silently keep stale complete collection

Si se ejecuta:

```text
direct attach Role#5
```

y `User.roles` estaba `COMPLETE`, el ORM deberá:

- reconciliarla; o
- invalidarla/marcarla stale.

Nunca dejarla silenciosamente como `COMPLETE` incorrecta.

---

# 182. Raw SQL

Raw SQL externo que modifique relaciones puede invalidar conocimiento ORM.

---

# 183. Raw SQL rule

```text
ExternalMutation
→
ManagedRelationshipState may become STALE
```

---

# 184. Database triggers

Triggers también pueden modificar relationship-relevant columns.

Si el ORM depende de ellos, metadata/persistence policy deberá declarar generated/refresh requirements.

---

# 185. Generated relationship values

Algunas DBs pueden asignar valores de asociación mediante:

- triggers;
- defaults;
- generated columns.

Estos casos requerirán result reconciliation explícita.

---

# 186. Batch persistence

Operaciones relationship compatibles podrán agruparse.

Ejemplo:

```text
100 role memberships
```

---

# 187. Batch ≠ Bulk semantic bypass

Batch preserva:

```text
individual semantic operations
individual outcome attribution
```

cuando sea necesario.

---

# 188. Batch eligibility

Depende de:

```text
same operation kind
same relationship mapping
same physical query shape
identity availability
outcome correlation
platform capabilities
parameter limits
```

---

# 189. Dependency preservation

Batching nunca deberá violar:

```text
PersistencePlan DAG
```

---

# 190. Generated identity barriers and batching

No agrupar join-table memberships antes de resolver IDs requeridos.

---

# 191. Batch failure

Si el driver no permite atribuir qué memberships se aplicaron:

```text
batch outcome
→ PARTIAL/UNKNOWN
```

según evidencia.

---

# 192. Retry semantics

Retry de relationship operations deberá considerar idempotencia.

---

# 193. Membership insert retry

Con unique membership constraint:

```text
INSERT role_user(...)
```

repetido puede producir duplicate-key.

Eso no significa automáticamente que el primer intento haya sido exitoso.

---

# 194. Idempotency semantics

El sistema deberá distinguir:

```text
operation desired final state
```

de:

```text
statement replay
```

---

# 195. State-oriented retry

Una futura estrategia puede verificar:

```text
membership exists?
```

antes de reconciliar un outcome incierto.

Pero esto es recovery explícito, no supuesto automático.

---

# 196. Cancellation

Relationship persistence deberá respetar:

```text
deadline
cancellation token
request cancellation
worker shutdown
```

---

# 197. Cancellation during plan execution

Operaciones no ejecutadas deberán permanecer:

```text
NOT_EXECUTED
```

---

# 198. Cancellation after statement dispatch

Puede resultar:

```text
UNKNOWN
```

si no puede determinarse el outcome.

---

# 199. Flush freeze

Una vez congelado el workset:

```text
relationship persistence plan
```

no deberá mutarse concurrentemente.

---

# 200. Post-lifecycle mutation

Si `postUpdate` modifica una relación:

```text
mutation belongs to future flush
```

salvo reglas específicas previamente definidas.

---

# 201. Recursive flush

Un relationship callback no podrá invocar recursivamente el mismo EntityManager flush.

---

# 202. Relationship lifecycle events

Relationship Persistence no deberá crear un segundo sistema de lifecycle incompatible.

Podrá integrarse con:

```text
Persistence Events
Entity Lifecycle Events
Transaction Synchronization
```

---

# 203. Relationship event timing

Eventos deberán distinguir:

```text
relationship change detected
relationship operation planned
relationship operation executed
relationship reconciled
transaction committed
```

---

# 204. Executed ≠ committed

Nunca llamar:

```text
RelationshipCommitted
```

al terminar un statement.

---

# 205. External side effects

Si una relationship change debe disparar:

```text
email
webhook
message
```

utilizar:

```text
afterCommit
Outbox
Job After Commit
```

---

# 206. RelationshipPersistenceContext

Propuesta:

```php
final readonly class RelationshipPersistenceContext
{
    public function __construct(
        public PersistenceContextId $persistenceContext,
        public FlushId $flush,
        public RelationshipMetadataRegistry $metadata,
        public RelationshipPersistencePolicy $policy,
    ) {}
}
```

---

# 207. Context ≠ global state

Nunca:

```php
RelationshipPersistence::currentContext();
```

como static mutable global.

---

# 208. RelationshipPersistenceCoordinator

```php
interface RelationshipPersistenceCoordinator
{
    public function collect(
        UnitOfWork $unitOfWork,
        RelationshipPersistenceContext $context,
    ): RelationshipPersistenceWork;
}
```

---

# 209. Specialized persisters

Arquitectura propuesta:

```text
RelationshipPersistenceCoordinator
├── OneToOneRelationshipPersister
├── OneToManyRelationshipPersister
├── ManyToOneRelationshipPersister
├── ManyToManyRelationshipPersister
└── PolymorphicRelationshipPersister
```

---

# 210. Persister naming caveat

Estos `Persister` no ejecutan SQL directamente.

Son traductores ORM de:

```text
relationship changes
→ persistence operations
```

---

# 211. Alternative naming

Para evitar ambigüedad también puede usarse:

```text
RelationshipOperationFactory
```

o:

```text
RelationshipPersistenceTranslator
```

---

# 212. Recommended naming

Preferencia:

```text
RelationshipPersistenceCoordinator
+
RelationshipOperationFactory
```

para mantener clara la separación con Execution Engine.

---

# 213. RelationshipOperationFactory

```php
interface RelationshipOperationFactory
{
    /**
     * @return list<RelationshipPersistenceOperation>
     */
    public function create(
        RelationshipChangeSet $changeSet,
        RelationshipMetadata $metadata,
        RelationshipPersistenceContext $context,
    ): array;
}
```

---

# 214. RelationshipDependencyExtractor

```php
interface RelationshipDependencyExtractor
{
    /**
     * @return list<PersistenceDependency>
     */
    public function dependencies(
        RelationshipPersistenceOperation $operation,
        PersistencePlanningContext $context,
    ): array;
}
```

---

# 215. RelationshipReconciler

Después de execution:

```php
interface RelationshipPersistenceReconciler
{
    public function reconcile(
        RelationshipPersistenceOperation $operation,
        RelationshipPersistenceOutcome $outcome,
        RelationshipReconciliationContext $context,
    ): void;
}
```

---

# 216. Reconciler responsibility

Puede actualizar:

```text
relationship snapshot
collection mutation log
knowledge status
UoW bookkeeping
```

según evidencia.

---

# 217. Reconciler must not fabricate success

`UNKNOWN` permanece incierto.

---

# 218. Reconciliation atomicity

Cambios internos coordinados:

```text
Snapshot
MutationLog
RelationshipState
UoW bookkeeping
```

deberán actualizarse lógicamente como una unidad.

---

# 219. Internal reconciliation ≠ DB transaction

Es atomicidad del estado ORM interno, no una transacción SQL.

---

# 220. Error taxonomy

```text
DatabaseRelationshipPersistenceException
├── RelationshipChangeDetectionException
├── InvalidRelationshipChangeException
├── RelationshipOwnershipViolationException
├── RelationshipSynchronizationException
├── RelationshipCardinalityViolationException
├── RelationshipNullabilityViolationException
├── TransientRelationshipTargetException
├── DetachedRelationshipTargetException
├── RelationshipIdentityUnavailableException
├── RelationshipMembershipException
├── DuplicateRelationshipMembershipException
├── PartialCollectionPersistenceException
├── RelationshipOrderingPersistenceException
├── RelationshipCascadeException
├── RelationshipCascadeCycleException
├── RelationshipOrphanRemovalException
├── SharedOrphanException
├── RelationshipDependencyException
├── UnsatisfiedRelationshipDependencyException
├── RelationshipPersistenceCycleException
├── RelationshipVersionConflictException
├── PolymorphicRelationshipPersistenceException
├── UnknownMorphTypePersistenceException
├── RelationshipBatchPersistenceException
├── RelationshipOutcomeException
├── RelationshipReconciliationException
├── RelationshipConsistencyException
├── RelationshipPersistenceCancellationException
├── RelationshipRuntimeIsolationException
└── RelationshipPersistenceInvariantException
```

---

# 221. Validation before planning

Cada ChangeSet deberá validar:

```text
relationship exists
owner is compatible
target is compatible
ownership semantics
cardinality
nullability
entity state
identity requirements
cascade requirements
collection coverage
polymorphic type
```

---

# 222. Detached targets

Una referencia a entidad `DETACHED` requiere política explícita.

Default recomendado:

```text
reject
```

en vez de reattach implícito.

---

# 223. Managed target

Target `MANAGED` es el caso normal.

---

# 224. NEW target

Permitido si:

```text
already persisted/scheduled
OR
cascade persist enabled
```

---

# 225. REMOVED target

Asignar una entidad `REMOVED` como nuevo target normalmente deberá fallar.

---

# 226. Cross-context entities

Una entidad administrada por otro `PersistenceContext` incompatible no deberá insertarse silenciosamente en una relación managed.

---

# 227. Context identity rule

```text
RelationshipOwnerContext
must be compatible with
RelationshipTargetContext
```

cuando ambos están managed.

---

# 228. Tenant isolation

Relationship persistence deberá preservar tenant/database/shard identity domain.

---

# 229. Cross-tenant reference

Por defecto:

```text
TenantA.Post
→ TenantB.User
```

deberá rechazarse.

---

# 230. Metadata is not enough

Que los EntityTypes sean compatibles no significa que sus runtime identity domains lo sean.

---

# 231. Relationship EntityKey

La validación deberá considerar:

```text
EntityIdentityNamespace
+
Tenant/Database Context
+
Canonical Identifier
```

según arquitectura del IdentityMap.

---

# 232. Sharding

Cross-shard relationships deberán estar:

- prohibidas;
- virtualizadas;
- o soportadas explícitamente por extensión.

Nunca asumidas.

---

# 233. Foreign-key feasibility

Una FK física no puede apuntar arbitrariamente entre bases/shards.

El planner/capability model deberá conocer estas restricciones.

---

# 234. Relationship persistence policy

```php
final readonly class RelationshipPersistencePolicy
{
    public function __construct(
        public RelationshipSynchronizationPolicy $synchronization,
        public bool $rejectDetachedTargets,
        public bool $rejectCrossContextReferences,
        public bool $strictPartialCollections,
        public bool $detectBidirectionalConflicts,
    ) {}
}
```

---

# 235. Secure/default-safe policy

Recomendado:

```text
owning side authoritative
detached target rejection
cross-context rejection
partial collection conservative handling
bidirectional conflict diagnostics
unknown outcome preservation
```

---

# 236. Performance architecture

Relationship persistence debe evitar:

```text
full graph scan
```

si no existe trabajo relationship relevante.

---

# 237. Dirty relationship registry

UoW puede mantener:

```text
dirty to-one relationships
dirty collections
cascade roots
orphan candidates
```

---

# 238. Complexity

Para `C` cambios relationship confirmados:

```text
Tchange ≈ O(C)
```

más:

```text
cascade traversal
dependency analysis
```

---

# 239. Snapshot diff complexity

Para una colección completa de `N` miembros:

```text
Tdiff ≈ O(N)
```

usando hash/set por canonical membership key.

---

# 240. Avoid O(N²)

Nunca comparar cada elemento con todos los demás mediante equality de objetos.

---

# 241. Large collections

Para colecciones enormes se preferirá:

```text
explicit mutation tracking
EXTRA_LAZY operations
direct relationship operations
```

sobre cargar todo para calcular un diff.

---

# 242. Memory governance

Mutation logs deberán poder compactarse:

```text
add X + remove X
→ no-op
```

cuando sea semánticamente correcto.

---

# 243. Persistent runtime architecture

Estado compartible:

```text
RelationshipMetadata
Operation definitions
immutable policies
compiled accessors
```

Estado scoped:

```text
Relationship snapshots
PersistentCollections
mutation logs
orphan candidates
cascade visited sets
RelationshipChangeSets
PersistenceOperations
outcomes
reconciliation state
```

---

# 244. FrankenPHP

Nunca dejar:

```text
dirty collections from request A
```

accesibles durante request B.

---

# 245. RoadRunner/OpenSwoole

Misma regla.

---

# 246. Scope cleanup

Al finalizar un scope deberán descartarse:

```text
relationship worksets
mutation logs
cascade traversal state
temporary membership keys
operation outcomes
```

según lifecycle del PersistenceContext.

---

# 247. Cleanup ≠ flush

Nunca:

```text
worker reset
→ automatically persist relationship changes
```

---

# 248. Cleanup with pending changes

Los cambios no flushed se descartan junto con el PersistenceContext.

No se persisten implícitamente.

---

# 249. Concurrency

Un mismo mutable `PersistenceContext` no será concurrent-mutation-safe por defecto.

---

# 250. Shared entity mutation

No deberá permitirse que dos coroutines manipulen simultáneamente la misma collection managed dentro del mismo context sin coordinación explícita.

---

# 251. Telemetry

Métricas posibles:

```text
database.orm.relationship.changes
database.orm.relationship.to_one_changes
database.orm.relationship.members_added
database.orm.relationship.members_removed
database.orm.relationship.join_operations
database.orm.relationship.cascade_traversals
database.orm.relationship.orphans
database.orm.relationship.partial_collection_mutations
database.orm.relationship.persistence_failures
database.orm.relationship.persistence_unknown
database.orm.relationship.version_conflicts
database.orm.relationship.reconciliation_failures
```

---

# 252. Tracing

Span conceptual:

```text
database.orm.relationship.persistence
```

atributos de baja cardinalidad:

```text
relationship.kind
operation.kind
collection.coverage
outcome
cascade.enabled
orphan_removal.enabled
```

---

# 253. PII

No registrar automáticamente:

```text
entity field values
raw identifiers
tenant secrets
relationship payload data
```

---

# 254. Diagnostics

Ejemplo:

```text
RELATIONSHIP PERSISTENCE

Relationship:
    Post.author

Kind:
    MANY_TO_ONE

Owner:
    Post

Change:
    REFERENCE_REPLACED

Before:
    <entity reference>

After:
    <entity reference>

Owning Side:
    YES

Target Identity:
    ESTABLISHED

Operation:
    SET_TO_ONE_REFERENCE

Dependencies:
    none

Outcome:
    SUCCESS

Transaction:
    ACTIVE / NOT YET COMMITTED
```

---

# 255. Many-to-many diagnostic

```text
Relationship:
    User.roles

Coverage:
    PARTIAL

Explicit Additions:
    3

Explicit Removals:
    1

Inferred Removals:
    0

Join Operations:
    INSERT × 3
    DELETE × 1
```

La ausencia de miembros no observados no genera deletes.

---

# 256. Explainability

Debe ser posible explicar:

```text
why operation exists
which ChangeSet produced it
which metadata rule authorized it
which dependency delayed it
how outcome was reconciled
```

---

# 257. Directory Structure

```text
src/Quantum/Database/ORM/Relationship/Persistence/
│
├── Contract/
│   ├── RelationshipPersistenceCoordinator.php
│   ├── RelationshipOperationFactory.php
│   ├── RelationshipDependencyExtractor.php
│   ├── RelationshipPersistenceReconciler.php
│   └── RelationshipChangeDetector.php
│
├── Change/
│   ├── RelationshipChangeSet.php
│   ├── RelationshipChangeKind.php
│   ├── ToOneRelationshipChangeSet.php
│   ├── ToManyRelationshipChangeSet.php
│   ├── RelationshipReorderChangeSet.php
│   └── RelationshipChangeSetCollection.php
│
├── Membership/
│   ├── RelationshipMembershipKey.php
│   ├── RelationshipMemberKey.php
│   ├── MembershipSnapshot.php
│   ├── MembershipMutationLog.php
│   └── MembershipState.php
│
├── Collection/
│   ├── PersistentRelationshipCollection.php
│   ├── CollectionCoverage.php
│   ├── CollectionMutation.php
│   └── CollectionChangeDetector.php
│
├── Operation/
│   ├── RelationshipPersistenceOperation.php
│   ├── RelationshipPersistenceOperationId.php
│   ├── SetToOneReferenceOperation.php
│   ├── ClearToOneReferenceOperation.php
│   ├── AddRelationshipMemberOperation.php
│   ├── RemoveRelationshipMemberOperation.php
│   ├── AddJoinTableMembershipOperation.php
│   ├── RemoveJoinTableMembershipOperation.php
│   ├── UpdateJoinTableMembershipOperation.php
│   ├── SetPolymorphicReferenceOperation.php
│   ├── ClearPolymorphicReferenceOperation.php
│   ├── ReorderRelationshipOperation.php
│   └── OrphanRemovalOperation.php
│
├── Factory/
│   ├── OneToOneOperationFactory.php
│   ├── OneToManyOperationFactory.php
│   ├── ManyToOneOperationFactory.php
│   ├── ManyToManyOperationFactory.php
│   └── PolymorphicOperationFactory.php
│
├── Cascade/
│   ├── RelationshipCascadePlanner.php
│   ├── CascadeTraversal.php
│   ├── CascadeVisitedSet.php
│   └── CascadeWork.php
│
├── Orphan/
│   ├── OrphanDetector.php
│   ├── OrphanCandidate.php
│   ├── OrphanRemovalPlanner.php
│   └── OrphanRemovalPolicyEvaluator.php
│
├── Dependency/
│   ├── RelationshipDependencyExtractor.php
│   ├── RelationshipIdentityDependency.php
│   ├── RelationshipOrderingDependency.php
│   └── RelationshipConstraintDependency.php
│
├── Polymorphic/
│   ├── PolymorphicReferenceResolver.php
│   ├── PolymorphicMembershipKey.php
│   └── PolymorphicPersistenceValidator.php
│
├── Outcome/
│   ├── RelationshipPersistenceOutcome.php
│   ├── RelationshipPersistenceOutcomeStatus.php
│   └── RelationshipOutcomeCollection.php
│
├── Reconciliation/
│   ├── RelationshipPersistenceReconciler.php
│   ├── RelationshipReconciliationContext.php
│   └── RelationshipKnowledgeStatus.php
│
├── Policy/
│   └── RelationshipPersistencePolicy.php
│
├── Diagnostics/
│   ├── RelationshipPersistenceDiagnostics.php
│   └── RelationshipPersistenceExplainer.php
│
└── Exception/
    └── ...
```

---

# 258. Integration with UnitOfWork

Flujo completo:

```text
Entity Mutation
        ↓
UoW tracking
        ↓
Relationship dirty registry
        ↓
Change detection
        ↓
RelationshipChangeSets
        ↓
Cascade / Orphan analysis
        ↓
RelationshipPersistenceWork
        ↓
Persistence Planner
```

---

# 259. Integration with Persistence Engine

`Persistence Engine` recibe trabajo combinado:

```text
Entity Insert Operations
Entity Update Operations
Entity Delete Operations
Relationship Operations
```

y lo transforma mediante el planner en un plan coherente.

---

# 260. Unified Persistence DAG

Ejemplo:

```text
Insert User#NEW
       │
       ▼
Resolve User ID
       │
       ▼
Update Post.author
       │
       ▼
Insert UserRole Membership
```

---

# 261. Integration with Query Engine

Relationship operation:

```text
SetToOneReferenceOperation
```

se traduce a:

```text
Update Query Model
```

No SQL.

---

# 262. Integration with Type System

Identifiers y pivot values deberán utilizar:

```text
Database Type System
```

para conversiones.

---

# 263. Integration with IdentityMap

Relationship references a entidades managed deberán preservar canonical entity identity.

---

# 264. IdentityMap ≠ relationship snapshot

IdentityMap responde:

> ¿Cuál es la instancia canónica de EntityKey X?

Relationship snapshot responde:

> ¿Qué asociación conocía el ORM como baseline?

---

# 265. Integration with Lifecycle

Cambios de relaciones producidos durante lifecycle callbacks deberán seguir reglas de estabilización del Flush System.

---

# 266. Integration with Transaction Manager

Transaction Manager controla:

```text
begin
commit
rollback
savepoint
```

Relationship Persistence recibe eventos de completion para reconciliación final cuando sea necesario.

---

# 267. Integration with Consistency System

`DATABASE_PERSISTENCE_CONSISTENCY_SYSTEM` deberá poder representar consistencia específica de relaciones.

---

# 268. Relationship consistency domain

Agregar conceptualmente:

```text
ConsistencyDomain::RELATIONSHIP
```

con evidencia por relación/operación cuando sea necesario.

---

# 269. Integration with Telemetry

Telemetry observa:

```text
changes
planning
execution
outcomes
reconciliation
```

sin modificar semántica.

---

# 270. Testing Strategy

La suite deberá cubrir al menos:

- to-one assign;
- to-one replace;
- nullable clear;
- non-nullable clear rejection;
- one-to-one uniqueness;
- owning/inverse semantics;
- unsynchronized inverse;
- strict bidirectional policy;
- one-to-many add/remove;
- many-to-many attach/detach;
- duplicate membership;
- composite identifiers;
- generated IDs;
- cascade persist;
- cascade remove;
- orphan removal;
- NEW orphan cancellation;
- partial collections;
- explicit partial removal;
- collection replacement;
- reorder;
- polymorphic to-one;
- polymorphic many-to-many;
- unknown morph type;
- cross-context references;
- tenant isolation;
- optimistic locking;
- batch persistence;
- UNKNOWN outcomes;
- rollback;
- transaction commit uncertainty;
- persistent workers.

---

# 271. Test — owning side

```text
User.posts changed
Post.author unchanged
```

Con `OWNING_SIDE_AUTHORITATIVE`:

```text
no independent FK persistence from inverse side
```

---

# 272. Test — strict bidirectional

Estado contradictorio:

```text
Post.author = UserA
UserB.posts contains Post
```

deberá fallar antes de ejecutar DB operations.

---

# 273. Test — partial collection

Snapshot desconocido + explicit addition:

```text
PARTIAL collection
add Role#5
```

deberá generar solo:

```text
ADD Role#5
```

No deletes inferidos.

---

# 274. Test — partial absence

Un Role no presente en colección parcial no deberá generar `REMOVE`.

---

# 275. Test — generated target identity

```text
NEW User
Post.author = User
```

deberá producir dependency:

```text
Insert User
→ Resolve ID
→ Update/Insert Post relationship
```

---

# 276. Test — many-to-many generated identities

```text
NEW User
NEW Role
User.roles += Role
```

deberá ordenar:

```text
Insert User
Insert Role
→
Join membership
```

---

# 277. Test — add/remove normalization

Baseline absent:

```text
add X
remove X
```

deberá producir no-op.

---

# 278. Test — remove/add normalization

Baseline present:

```text
remove X
add X
```

deberá producir no-op si no existe semántica adicional de ordering/pivot payload.

---

# 279. Test — orphan NEW

Child NEW añadido y removido antes de flush deberá poder cancelar insert si no tiene otras dependencias.

---

# 280. Test — UNKNOWN

Connection failure después de dispatch deberá conservar outcome `UNKNOWN`.

---

# 281. Test — snapshot safety

`UNKNOWN` nunca deberá actualizar relationship snapshot como success.

---

# 282. Test — rollback

Rollback conocido no deberá rebobinar mágicamente el object graph.

---

# 283. Test — direct mutation staleness

Operación directa de attach sobre una colección managed completa deberá reconciliarla o marcarla stale.

---

# 284. Test — no SQL

Ningún componente de `ORM/Relationship/Persistence` deberá generar SQL strings.

---

# 285. Test — no direct driver

El subsystem no dependerá de:

```text
PDO
Driver
Connection
```

para producir relationship operations.

---

# 286. Test — persistent runtime isolation

Request A:

```text
User#1.roles dirty
```

Request B:

```text
User#1.roles clean
```

deberán usar estados totalmente independientes aunque compartan metadata.

---

# 287. Architectural Invariants

## DB-ORM-REL-PERSIST-001

Relationship Persistence transformará cambios semánticos en operaciones tipadas.

## DB-ORM-REL-PERSIST-002

Relationship Persistence no generará SQL.

## DB-ORM-REL-PERSIST-003

Relationship Persistence no ejecutará queries directamente.

## DB-ORM-REL-PERSIST-004

Relationship Persistence será distinta de Change Tracking.

## DB-ORM-REL-PERSIST-005

Relationship Persistence será distinta de UnitOfWork.

## DB-ORM-REL-PERSIST-006

Relationship Persistence será distinta de Persistence Planner.

## DB-ORM-REL-PERSIST-007

Relationship Persistence será distinta de Transaction Manager.

## DB-ORM-REL-PERSIST-008

Relationship Persistence success no implicará transaction commit.

## DB-ORM-REL-PERSIST-009

Owning side será la fuente persistente autoritativa por defecto.

## DB-ORM-REL-PERSIST-010

Inverse side no generará una segunda fuente persistente independiente.

## DB-ORM-REL-PERSIST-011

Bidirectional contradictions serán policy-driven.

## DB-ORM-REL-PERSIST-012

RelationshipChangeSet será distinto de SQL.

## DB-ORM-REL-PERSIST-013

To-one ChangeSet preservará before/after identity.

## DB-ORM-REL-PERSIST-014

To-many ChangeSet preservará added/removed membership.

## DB-ORM-REL-PERSIST-015

Entity membership utilizará canonical identity.

## DB-ORM-REL-PERSIST-016

Temporary UoW identity no será persistida.

## DB-ORM-REL-PERSIST-017

Relationship snapshots serán independientes del live collection object.

## DB-ORM-REL-PERSIST-018

EMPTY será distinto de UNINITIALIZED.

## DB-ORM-REL-PERSIST-019

PARTIAL será distinto de COMPLETE.

## DB-ORM-REL-PERSIST-020

Absence in partial collection no significará removal.

## DB-ORM-REL-PERSIST-021

Explicit mutations podrán persistirse sobre colecciones parciales cuando sean seguras.

## DB-ORM-REL-PERSIST-022

Relationship dirtiness será distinta de scalar entity dirtiness.

## DB-ORM-REL-PERSIST-023

Snapshot diff solo inferirá removals cuando exista cobertura suficiente.

## DB-ORM-REL-PERSIST-024

Mutation logs serán canonicalizados.

## DB-ORM-REL-PERSIST-025

Redundant add/remove podrá colapsarse a no-op cuando semánticamente corresponda.

## DB-ORM-REL-PERSIST-026

Persistence operations serán typed.

## DB-ORM-REL-PERSIST-027

PersistenceOperation será distinta de QueryModel.

## DB-ORM-REL-PERSIST-028

Clear to-one respetará nullability.

## DB-ORM-REL-PERSIST-029

Generated target identity creará dependency barrier.

## DB-ORM-REL-PERSIST-030

Cascade persist registrará trabajo; no ejecutará INSERT inmediato.

## DB-ORM-REL-PERSIST-031

NEW non-cascaded target no será persistido implícitamente.

## DB-ORM-REL-PERSIST-032

REMOVED target no será asignado silenciosamente.

## DB-ORM-REL-PERSIST-033

DETACHED target requerirá política explícita.

## DB-ORM-REL-PERSIST-034

One-to-one uniqueness podrá validarse tempranamente cuando sea observable.

## DB-ORM-REL-PERSIST-035

ORM uniqueness checks no sustituirán DB constraints.

## DB-ORM-REL-PERSIST-036

One-to-many inverse mutation no controlará FK independientemente.

## DB-ORM-REL-PERSIST-037

Many-to-many membership utilizará join-table operations.

## DB-ORM-REL-PERSIST-038

Removing many-to-many membership no eliminará target por defecto.

## DB-ORM-REL-PERSIST-039

Join cleanup será distinto de cascade target delete.

## DB-ORM-REL-PERSIST-040

SET_LIKE relationships deduplicarán membership por canonical key.

## DB-ORM-REL-PERSIST-041

MembershipKey soportará composite IDs.

## DB-ORM-REL-PERSIST-042

Join membership con generated IDs esperará ambas identities requeridas.

## DB-ORM-REL-PERSIST-043

Rich association podrá modelarse como entity explícita.

## DB-ORM-REL-PERSIST-044

Polymorphic persistence utilizará Morph Registry.

## DB-ORM-REL-PERSIST-045

Polymorphic persistence no almacenará arbitrary class names por defecto.

## DB-ORM-REL-PERSIST-046

Morph type e identifier se tratarán como referencia lógica coordinada.

## DB-ORM-REL-PERSIST-047

Unknown morph aliases serán rechazados.

## DB-ORM-REL-PERSIST-048

Polymorphic target deberá pertenecer al allowed target set.

## DB-ORM-REL-PERSIST-049

Cascade traversal tendrá cycle protection.

## DB-ORM-REL-PERSIST-050

Cascade traversal no ejecutará SQL.

## DB-ORM-REL-PERSIST-051

Cascade work será registrado en UoW.

## DB-ORM-REL-PERSIST-052

Flush stabilization será bounded.

## DB-ORM-REL-PERSIST-053

Workset se congelará antes de ejecución.

## DB-ORM-REL-PERSIST-054

Recursive same-manager flush será rechazado.

## DB-ORM-REL-PERSIST-055

Orphan removal será distinto de membership removal.

## DB-ORM-REL-PERSIST-056

Orphan removal requerirá evidencia suficiente.

## DB-ORM-REL-PERSIST-057

Partial collection absence no generará orphan.

## DB-ORM-REL-PERSIST-058

Explicit partial removal podrá generar orphan cuando metadata lo autorice.

## DB-ORM-REL-PERSIST-059

Shared targets no usarán orphan removal ambiguo.

## DB-ORM-REL-PERSIST-060

NEW orphan no requerirá INSERT+DELETE innecesarios.

## DB-ORM-REL-PERSIST-061

Persisted ordering será explícito.

## DB-ORM-REL-PERSIST-062

Display ordering no implicará persisted ordering.

## DB-ORM-REL-PERSIST-063

Global reorder requerirá conocimiento suficiente del orden.

## DB-ORM-REL-PERSIST-064

Relationship persistence producirá dependency information.

## DB-ORM-REL-PERSIST-065

Planner decidirá ordering final.

## DB-ORM-REL-PERSIST-066

No existirá universal persistence ordering.

## DB-ORM-REL-PERSIST-067

Unsatisfiable relationship cycles fallarán explícitamente.

## DB-ORM-REL-PERSIST-068

Platform capabilities participarán en cycle resolution.

## DB-ORM-REL-PERSIST-069

Relationship Persistence no hará commit.

## DB-ORM-REL-PERSIST-070

Flush no hará commit de user transaction.

## DB-ORM-REL-PERSIST-071

Statement success será distinto de durable relationship state.

## DB-ORM-REL-PERSIST-072

Relationship outcomes soportarán SUCCESS.

## DB-ORM-REL-PERSIST-073

Relationship outcomes soportarán FAILED.

## DB-ORM-REL-PERSIST-074

Relationship outcomes soportarán UNKNOWN.

## DB-ORM-REL-PERSIST-075

Relationship outcomes soportarán SKIPPED/NOT_EXECUTED.

## DB-ORM-REL-PERSIST-076

UNKNOWN nunca se convertirá automáticamente en SUCCESS.

## DB-ORM-REL-PERSIST-077

UNKNOWN nunca se convertirá automáticamente en FAILED.

## DB-ORM-REL-PERSIST-078

Partial outcomes serán preservados.

## DB-ORM-REL-PERSIST-079

Unknown significant outcomes podrán taint PersistenceContext.

## DB-ORM-REL-PERSIST-080

Relationship snapshots solo avanzarán con evidencia suficiente.

## DB-ORM-REL-PERSIST-081

Failed outcome no actualizará baseline como éxito.

## DB-ORM-REL-PERSIST-082

Unknown outcome no actualizará baseline como éxito.

## DB-ORM-REL-PERSIST-083

Object graph no será automáticamente rebobinado después de failure.

## DB-ORM-REL-PERSIST-084

DatabaseReality será distinta de CurrentObjectGraph.

## DB-ORM-REL-PERSIST-085

ORMKnowledge será distinto de DatabaseReality.

## DB-ORM-REL-PERSIST-086

Rollback no implicará automatic object graph rewind.

## DB-ORM-REL-PERSIST-087

Unknown commit completion preservará uncertainty.

## DB-ORM-REL-PERSIST-088

Optimistic locking podrá cubrir relationship-driven owner updates.

## DB-ORM-REL-PERSIST-089

Join-table mutation no incrementará owner version automáticamente sin policy.

## DB-ORM-REL-PERSIST-090

Many-to-many persistence preferirá deltas.

## DB-ORM-REL-PERSIST-091

Replace-all no será estrategia universal.

## DB-ORM-REL-PERSIST-092

Direct relationship mutations deberán reconciliar managed state.

## DB-ORM-REL-PERSIST-093

Direct mutation no dejará collection incorrectamente COMPLETE.

## DB-ORM-REL-PERSIST-094

Raw external mutations podrán marcar relationship state stale.

## DB-ORM-REL-PERSIST-095

Trigger-generated relationship values requerirán reconciliation explícita.

## DB-ORM-REL-PERSIST-096

Batching no cambiará semántica ORM.

## DB-ORM-REL-PERSIST-097

Batching preservará dependency DAG.

## DB-ORM-REL-PERSIST-098

Batching respetará generated identity barriers.

## DB-ORM-REL-PERSIST-099

Ambiguous batch outcome se representará conservadoramente.

## DB-ORM-REL-PERSIST-100

Retry requerirá análisis de idempotencia.

## DB-ORM-REL-PERSIST-101

Duplicate-key durante retry no probará automáticamente previous success.

## DB-ORM-REL-PERSIST-102

Cancellation será first-class.

## DB-ORM-REL-PERSIST-103

Cancellation después de dispatch podrá producir UNKNOWN.

## DB-ORM-REL-PERSIST-104

Post-lifecycle mutations podrán pertenecer al siguiente flush.

## DB-ORM-REL-PERSIST-105

Relationship execution events serán distintos de commit events.

## DB-ORM-REL-PERSIST-106

External irreversible effects utilizarán afterCommit/Outbox.

## DB-ORM-REL-PERSIST-107

RelationshipPersistenceContext será scoped.

## DB-ORM-REL-PERSIST-108

No existirá static mutable current relationship context.

## DB-ORM-REL-PERSIST-109

Operation factories no ejecutarán SQL.

## DB-ORM-REL-PERSIST-110

Dependency extractors no ejecutarán SQL.

## DB-ORM-REL-PERSIST-111

Reconciler preservará outcome evidence.

## DB-ORM-REL-PERSIST-112

Reconciler no fabricará success.

## DB-ORM-REL-PERSIST-113

Internal reconciliation será distinta de DB transaction.

## DB-ORM-REL-PERSIST-114

Owner y target deberán ser EntityTypes compatibles con metadata.

## DB-ORM-REL-PERSIST-115

Cross-context managed references serán rechazadas por defecto.

## DB-ORM-REL-PERSIST-116

Cross-tenant relationships serán rechazadas por defecto.

## DB-ORM-REL-PERSIST-117

Cross-shard relationships requerirán soporte explícito.

## DB-ORM-REL-PERSIST-118

Entity identity domain participará en relationship validation.

## DB-ORM-REL-PERSIST-119

Relationship metadata será immutable y compartible.

## DB-ORM-REL-PERSIST-120

Relationship snapshots serán scope-local.

## DB-ORM-REL-PERSIST-121

Persistent collections serán scope-local.

## DB-ORM-REL-PERSIST-122

Mutation logs serán scope-local.

## DB-ORM-REL-PERSIST-123

Cascade visited sets serán scope-local.

## DB-ORM-REL-PERSIST-124

Orphan candidates serán scope-local.

## DB-ORM-REL-PERSIST-125

Relationship operations serán flush/workset-local.

## DB-ORM-REL-PERSIST-126

Relationship outcomes serán operation/flush-local.

## DB-ORM-REL-PERSIST-127

Worker cleanup no implicará automatic flush.

## DB-ORM-REL-PERSIST-128

Pending changes podrán descartarse al destruir PersistenceContext.

## DB-ORM-REL-PERSIST-129

Same mutable PersistenceContext no será concurrent-mutation-safe por defecto.

## DB-ORM-REL-PERSIST-130

Relationship persistence evitará full graph scans cuando no exista trabajo.

## DB-ORM-REL-PERSIST-131

Collection diff deberá evitar O(N²) cuando sea posible.

## DB-ORM-REL-PERSIST-132

Large collections podrán utilizar explicit mutation tracking.

## DB-ORM-REL-PERSIST-133

Telemetry no alterará relationship semantics.

## DB-ORM-REL-PERSIST-134

Telemetry evitará PII por defecto.

## DB-ORM-REL-PERSIST-135

Diagnostics deberán explicar el origen de cada operation.

## DB-ORM-REL-PERSIST-136

Relationship persistence será testeable sin SQL compiler concreto.

## DB-ORM-REL-PERSIST-137

Relationship persistence será testeable sin driver concreto.

## DB-ORM-REL-PERSIST-138

Relationship persistence será independiente de PDO.

## DB-ORM-REL-PERSIST-139

Relationship persistence respetará EntityState.

## DB-ORM-REL-PERSIST-140

Relationship persistence respetará IdentityMap canonicality.

## DB-ORM-REL-PERSIST-141

Relationship persistence respetará RelationshipMetadata ownership.

## DB-ORM-REL-PERSIST-142

Relationship persistence respetará loaded coverage.

## DB-ORM-REL-PERSIST-143

Relationship persistence respetará Persistence Consistency evidence.

## DB-ORM-REL-PERSIST-144

Relationship persistence no asumirá DB state no observado.

## DB-ORM-REL-PERSIST-145

Relationship persistence no confundirá missing knowledge con absence.

## DB-ORM-REL-PERSIST-146

Relationship persistence no confundirá association removal con entity deletion.

## DB-ORM-REL-PERSIST-147

Relationship persistence no confundirá cascade remove con DB cascade.

## DB-ORM-REL-PERSIST-148

Relationship persistence no confundirá join cleanup con target deletion.

## DB-ORM-REL-PERSIST-149

Relationship persistence no confundirá flush success con commit.

## DB-ORM-REL-PERSIST-150

Relationship persistence mantendrá separación ORM → Query Engine → Compiler → Executor.

## DB-ORM-REL-PERSIST-151

Relationship persistence nunca accederá directamente al Driver.

## DB-ORM-REL-PERSIST-152

Relationship persistence nunca dependerá de dialect-specific SQL.

## DB-ORM-REL-PERSIST-153

Relationship persistence será capability-driven cuando existan diferencias de plataforma.

## DB-ORM-REL-PERSIST-154

Relationship persistence será compatible con persistent runtimes mediante scope isolation.

## DB-ORM-REL-PERSIST-155

Relationship state mutable nunca se almacenará en process-global metadata.

## DB-ORM-REL-PERSIST-156

Operation planning será deterministic para el mismo workset y metadata generation.

## DB-ORM-REL-PERSIST-157

Relationship operation IDs permitirán outcome attribution.

## DB-ORM-REL-PERSIST-158

Operation outcomes deberán poder rastrearse hasta ChangeSet y RelationshipId.

## DB-ORM-REL-PERSIST-159

Un relationship persistence failure no borrará evidencia válida de operaciones anteriores.

## DB-ORM-REL-PERSIST-160

VoltStack preservará explícitamente incertidumbre antes que fabricar consistencia.

---

# 288. Anti-Patterns

## 288.1 SQL desde la entidad

```php
public function setAuthor(User $user): void
{
    DB::update(...);
}
```

**Rechazado.**

---

## 288.2 SQL desde RelationshipMetadata

```php
$metadata->insertPivotSql();
```

**Rechazado.**

---

## 288.3 Persistir desde inverse side independientemente

```text
User.posts
and
Post.author
```

produciendo dos operaciones competidoras.

**Rechazado.**

---

## 288.4 Inferir removals desde colección parcial

```text
not loaded
→ assume deleted
```

**Rechazado.**

---

## 288.5 Delete-all/insert-all universal

```text
DELETE all memberships
INSERT current memberships
```

**Rechazado como estrategia general.**

---

## 288.6 Cascade mediante recursion sin visited set

**Rechazado.**

---

## 288.7 Orphan removal desde ausencia no observada

**Rechazado.**

---

## 288.8 Asumir success después de connection loss

**Rechazado.**

---

## 288.9 Actualizar snapshot antes de conocer outcome

**Rechazado.**

---

## 288.10 Automatic object rewind

```text
rollback
→ mutate entire object graph back automatically
```

**Rechazado como comportamiento general.**

---

## 288.11 Cross-tenant FK silenciosa

**Rechazado.**

---

## 288.12 Global mutable relationship queues

```php
static array $dirtyRelationships;
```

**Rechazado.**

---

# 289. Ejemplo integral — Many-to-One

Código:

```php
$post = $posts->find(42);
$user = $users->find(9);

$post->setAuthor($user);

$entityManager->flush();
```

Pipeline:

```text
Post.author mutation
        ↓
Change Tracking
        ↓
ToOneRelationshipChangeSet
        │
        ├── before = User#5
        └── after  = User#9
        ↓
RelationshipMetadata(Post.author)
        │
        ├── MANY_TO_ONE
        ├── OWNING
        └── author_id → users.id
        ↓
SetToOneReferenceOperation
        ↓
Persistence Planner
        ↓
Update Query Model
        ↓
SQL Compiler
        ↓
Statement Execution
        ↓
Operation Outcome
        ↓
Relationship Reconciliation
        ↓
Snapshot(Post.author) = User#9
```

solo si existe evidencia suficiente de éxito.

---

# 290. Ejemplo integral — Many-to-Many

```php
$user->roles->add($admin);
$user->roles->remove($editor);

$entityManager->flush();
```

Pipeline:

```text
PersistentCollection mutation log
        ↓
Normalize
        ↓
added:
    Admin

removed:
    Editor
        ↓
ToManyRelationshipChangeSet
        ↓
ManyToMany Metadata
        ↓
┌──────────────────────────────┐
│ AddJoinTableMembership       │
│ RemoveJoinTableMembership    │
└──────────────────────────────┘
        ↓
Persistence Planner
        ↓
Insert/Delete Query Models
        ↓
Execution
        ↓
Per-operation outcomes
        ↓
Membership Snapshot Reconciliation
```

---

# 291. Ejemplo integral — Cascade + Generated ID

```php
$post = new Post();

$author = new User();

$post->setAuthor($author);

$entityManager->persist($post);
$entityManager->flush();
```

Mapping:

```text
Post.author
cascade persist
```

Plan conceptual:

```text
persist(Post)
      ↓
cascade persist(User)
      ↓
UoW:
    Post = NEW
    User = NEW
      ↓
Persistence Planner

Insert User
      ↓
Resolve User.id
      ↓
Insert Post(author_id = User.id)
```

---

# 292. Ejemplo integral — Orphan Removal

```php
$order->removeItem($item);

$entityManager->flush();
```

Con:

```text
Order.items
orphanRemoval = true
```

Flujo:

```text
Explicit Membership Removal
        ↓
Relationship ChangeSet
        ↓
Orphan Analysis
        ↓
Confirmed Orphan
        ↓
RemoveEntityOperation(OrderItem)
        ↓
Persistence Planner
        ↓
Delete Persistence System
```

---

# 293. Ejemplo integral — Partial Collection

```text
User.roles
coverage = PARTIAL

Loaded:
    Admin
    Editor
```

Existe además en DB:

```text
Viewer
Auditor
```

Usuario ejecuta:

```php
$user->roles->remove($admin);
```

VoltStack genera:

```text
REMOVE Admin
```

No:

```text
REMOVE Viewer
REMOVE Auditor
```

porque:

```text
NotObserved
≠
Absent
```

---

# 294. Ejemplo integral — UNKNOWN

Plan:

```text
Insert role_user(User#42, Role#1)
```

Secuencia:

```text
statement dispatched
        ↓
connection lost
        ↓
no reliable result
```

Outcome:

```text
UNKNOWN
```

Entonces:

```text
MembershipSnapshot
≠ automatically updated
```

y:

```text
PersistenceContext
→ potentially TAINTED
```

---

# 295. Fórmulas fundamentales

## Relationship Change

```text
RelationshipChangeSet
=
Detect(
    RelationshipMetadata,
    RelationshipSnapshot,
    CurrentRelationshipState,
    ExplicitMutations
)
```

## Membership Delta

Para colección completa:

```text
Added
=
CurrentMembership
-
BaselineMembership
```

```text
Removed
=
BaselineMembership
-
CurrentMembership
```

## Safe Partial Collection

```text
SafePartialPersistence
=
ExplicitAdditions
+
ExplicitRemovals
```

sin inferir:

```text
UnknownAbsence
→ Removal
```

## Operation Generation

```text
RelationshipOperations
=
Translate(
    RelationshipChangeSets,
    RelationshipMetadata,
    EntityState,
    PersistencePolicy
)
```

## Planning

```text
PersistencePlan
=
Plan(
    EntityOperations
    +
    RelationshipOperations
    +
    Dependencies
)
```

## Reconciliation

```text
NewRelationshipBaseline
=
Reconcile(
    PreviousBaseline,
    OperationOutcome,
    PersistenceEvidence
)
```

## Unknown Rule

```text
InsufficientEvidence
→
UNKNOWN
```

nunca:

```text
InsufficientEvidence
→
AssumedSuccess
```

---

# 296. Master Formula

```text
Relationship Persistence System
=
Relationship Metadata
+
Relationship State
+
Relationship Snapshots
+
Collection Coverage
+
Explicit Mutation Tracking
+
Relationship Change Detection
+
Canonical Membership Identity
+
Owning-Side Authority
+
Bidirectional Consistency Policy
+
To-One Operations
+
To-Many Operations
+
Join-Table Operations
+
Polymorphic Operations
+
Cascade Traversal
+
Orphan Detection
+
Generated Identity Dependencies
+
Ordering Persistence
+
Optimistic Locking
+
Persistence DAG Integration
+
Batch Compatibility
+
Execution Outcomes
+
Uncertainty Preservation
+
Snapshot Reconciliation
+
Rollback Awareness
+
Consistency Integration
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

# 297. Regla maestra final

> **Una mutación en el object graph expresa intención; un `RelationshipChangeSet` expresa el cambio ORM confirmado; una `RelationshipPersistenceOperation` expresa el trabajo persistente; un `QueryModel` expresa la operación de datos; el SQL Compiler expresa SQL; el Executor ejecuta; y solo la evidencia obtenida posteriormente permite reconciliar el conocimiento ORM. Ninguna de estas capas deberá sustituir a las demás.**

La cadena completa queda:

```text
Object Graph
     ↓
Relationship State
     ↓
Change Tracking
     ↓
RelationshipChangeSet
     ↓
Relationship Persistence
     ↓
RelationshipPersistenceOperation
     ↓
Persistence Planner
     ↓
PersistencePlan
     ↓
Query Model
     ↓
Query Engine
     ↓
SQL Compiler
     ↓
Execution Engine
     ↓
Persistence Evidence
     ↓
Relationship Reconciliation
     ↓
Updated ORM Knowledge
```

---

# 298. Resultado arquitectónico

Con este sistema, VoltStack podrá manejar:

```php
$post->setAuthor($user);
$user->roles->add($admin);
$order->items->remove($item);
$comment->setCommentable($video);
```

como cambios de dominio/ORM sin permitir que las entidades conozcan:

```text
SQL
PDO
drivers
dialects
transactions
join-table statements
```

Al ejecutar:

```php
$entityManager->flush();
```

VoltStack podrá construir un único grafo coherente:

```text
Entity Inserts
Entity Updates
Entity Deletes
Relationship Reference Changes
Join Table Membership Changes
Cascade Operations
Orphan Removals
Generated Identity Dependencies
Optimistic Lock Dependencies
```

que posteriormente será planificado y ejecutado por la infraestructura canónica de persistencia.

Esto mantiene la regla fundamental de VoltStack:

```text
ORM
↓
Persistence Engine
↓
Query Engine
↓
Compiler
↓
Executor
↓
Connection
↓
Driver
```

y nunca en dirección inversa.

---

# 299. Siguiente documento

```text
150_DATABASE_RELATIONSHIP_LOADING_SYSTEM.md
```

El siguiente documento deberá definir el camino inverso:

```text
RelationshipMetadata
+
Owner Entity Identity
+
Loading Context
+
Fetch Policy
        ↓
Relationship Load Request
        ↓
Loading Strategy
        ↓
Entity Query / Query Model
        ↓
Execution
        ↓
Hydration
        ↓
Relationship Assembly
        ↓
Collection / Reference State
```

incluyendo:

- to-one loading;
- to-many loading;
- owning/inverse navigation;
- lazy loading;
- eager loading;
- explicit loading;
- batch loading;
- polymorphic loading;
- join-table loading;
- collection initialization;
- `UNINITIALIZED / PARTIAL / COMPLETE`;
- IdentityMap reuse;
- HydrationPlan integration;
- N+1 prevention hooks;
- recursion/cycle protection;
- read/write routing implications;
- transaction consistency;
- stale managed entities;
- cancellation;
- streaming;
- resource governance;
- persistent runtime isolation.

La regla central del siguiente sistema será:

> **Relationship Loading materializa conocimiento de asociaciones desde resultados obtenidos por el Query Engine y lo integra al object graph preservando IdentityMap, coverage y hydration semantics; nunca convierte el acceso a una relación en SQL oculto fuera de una estrategia de carga explícitamente gobernada.**