# 145_DATABASE_MANY_TO_ONE_RELATIONSHIP_SYSTEM.md

# VoltStack Quantum Database
## Database Many-to-One Relationship System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 145 — Database Many-to-One Relationship System  
**Bloque:** 13 — Relationships  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Many-to-One Relationship System` define la arquitectura mediante la cual VoltStack representa, carga, hidrata, rastrea y persiste asociaciones donde múltiples entidades origen pueden referenciar a una misma entidad destino.

Ejemplos:

```text
Order    N ───── 1 User
Comment  N ───── 1 Post
Employee N ───── 1 Department
Invoice  N ───── 1 Customer
```

Desde la perspectiva de una entidad individual:

```text
Order
 └── user → User#10
```

la relación contiene:

```text
0..1 target
```

dependiendo de su nulabilidad.

El sistema deberá soportar:

- relaciones required;
- relaciones nullable;
- asociaciones unidireccionales;
- asociaciones bidireccionales;
- owning-side semantics;
- inverse one-to-many synchronization;
- references conocidas sin objeto materializado;
- lazy loading;
- eager loading;
- batch loading;
- JOIN hydration;
- IdentityMap;
- EntityReference;
- generated identifiers;
- foreign-key persistence;
- assignment;
- nullification;
- reparenting;
- snapshots;
- change tracking;
- cascade persist;
- refresh;
- optimistic locking interaction;
- tenant/shard isolation;
- persistent runtimes.

---

# 2. Principio central

> **Una relación many-to-one de VoltStack será una referencia identity-aware hacia como máximo una entidad target y, en el modelo convencional basado en foreign key, constituirá la autoridad ORM que determina la identidad persistente de dicha referencia.**

Esto significa:

```text
Order.user
```

es normalmente la fuente ORM de:

```text
orders.user_id
```

---

# 3. Regla fundamental

```text
ManyToOne
=
Relationship Reference
+
Target Identity
+
Load State
+
Ownership Semantics
+
Persistence Knowledge
```

No simplemente:

```text
PHP object property
```

---

# 4. Many-to-One ≠ Foreign Key

Una foreign key:

```text
orders.user_id
```

es representación relacional.

La asociación:

```text
Order.user
```

es representación ORM.

Por tanto:

```text
ManyToOneRelationship
≠
ForeignKey
```

---

# 5. Many-to-One ≠ PHP Object Reference

Esto:

```php
private ?User $user;
```

no describe por sí solo:

- owning side;
- nullable semantics;
- target identity;
- loading state;
- cascade;
- tenant scope;
- join columns;
- persistence behavior.

---

# 6. Many-to-One ≠ One-to-One

Aunque ambas propiedades puedan contener:

```text
0..1 object
```

la cardinalidad global es diferente.

```text
Many-to-One

Order#1 ─┐
Order#2 ─┼──→ User#10
Order#3 ─┘
```

frente a:

```text
One-to-One

Profile#1 ───→ User#10
```

---

# 7. Modelo relacional habitual

```text
users
────────────
id PK

orders
────────────
id PK
user_id FK → users.id
```

Semántica:

```text
User 1 ←──── N Order
```

y desde `Order`:

```text
Order N ────→ 1 User
```

---

# 8. Owning Side

En este modelo:

```text
Order.user
```

es normalmente:

```text
OWNING
```

mientras:

```text
User.orders
```

es:

```text
INVERSE
```

---

# 9. Regla de ownership

> **El lado que controla semánticamente la foreign key será la autoridad de persistencia de la asociación.**

Normalmente:

```text
ManyToOne = OWNING
OneToMany = INVERSE
```

---

# 10. Ownership ≠ Object Location

Que una entidad contenga una propiedad no determina ownership.

---

# 11. Ownership ≠ Aggregate Ownership

DDD puede establecer:

```text
User = Aggregate Root
```

sin modificar:

```text
Order.user = owning side ORM
```

---

# 12. Arquitectura general

```text
                     ManyToOneMetadata
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
      Hydration        Entity Query       UnitOfWork
          │                                   │
          ▼                                   ▼
 Foreign-Key Identity                  Relationship Snapshot
          │                                   │
          ▼                                   ▼
      IdentityMap                       Change Tracking
          │                                   │
     ┌────┴────┐                              ▼
     │         │                       Persistence Planner
     ▼         ▼                              │
 Existing   Reference                         ▼
 Entity     / Hydration                  Query Engine
```

---

# 13. ManyToOneMetadata

Propuesta:

```php
final readonly class ManyToOneMetadata
{
    /**
     * @param list<JoinColumnMetadata> $joinColumns
     */
    public function __construct(
        public RelationshipId $id,
        public EntityType $sourceType,
        public EntityType $targetType,
        public string $property,
        public array $joinColumns,
        public RelationshipOwnership $ownership,
        public RelationshipNullability $nullability,
        public RelationshipFetchMode $fetchMode,
        public CascadePolicy $cascade,
        public ?string $inversedBy,
    ) {}
}
```

---

# 14. JoinColumnMetadata

```php
final readonly class JoinColumnMetadata
{
    public function __construct(
        public ColumnName $localColumn,
        public ColumnName $referencedColumn,
        public bool $nullable,
        public ?ForeignKeyAction $onDelete,
        public ?ForeignKeyAction $onUpdate,
    ) {}
}
```

---

# 15. Metadata ≠ Schema

`JoinColumnMetadata` describe mapping ORM.

No significa que el constraint exista realmente.

```text
ORM Mapping
≠
Observed Database Schema
```

---

# 16. Required Relationship

Ejemplo:

```php
#[ManyToOne(target: User::class)]
#[JoinColumn(nullable: false)]
private User $user;
```

Semánticamente:

```text
Order.user ∈ User
```

y no:

```text
Order.user ∈ User ∪ NULL
```

---

# 17. Nullable Relationship

```php
#[ManyToOne(target: User::class)]
#[JoinColumn(nullable: true)]
private ?User $user = null;
```

Semánticamente:

```text
Order.user ∈ User ∪ NULL
```

---

# 18. NULL ≠ Unloaded

Regla crítica:

```text
relationship = NULL
```

no deberá confundirse con:

```text
relationship not loaded
```

---

# 19. Unknown ≠ NULL

Igualmente:

```text
UNKNOWN
≠
NULL
```

---

# 20. Relationship Reference State

VoltStack necesita distinguir al menos:

```php
enum ToOneReferenceState
{
    case UNLOADED;
    case REFERENCE_KNOWN;
    case LOADED;
    case NULL;
}
```

---

# 21. UNLOADED

Significa:

```text
No relationship value has been established
```

No prueba que la FK sea NULL.

---

# 22. REFERENCE_KNOWN

Significa:

```text
Target identity known
Object not necessarily materialized
```

Ejemplo:

```text
user_id = 10
```

permite conocer:

```text
User#10
```

sin cargar toda la fila de User.

---

# 23. LOADED

Significa:

```text
Target identity known
+
Canonical target object available
```

---

# 24. NULL

Significa que existe evidencia suficiente de:

```text
relationship has no target
```

dentro de la observación actual.

---

# 25. State Model

```text
             FK not observed
                   │
                   ▼
               UNLOADED
                   │
           FK observed = 10
                   ▼
            REFERENCE_KNOWN
                   │
              materialize
                   ▼
                LOADED
```

Alternativa:

```text
FK observed = NULL
        │
        ▼
       NULL
```

---

# 26. Reference Identity

Una relación many-to-one podrá conocerse mediante:

```text
EntityKey
```

sin requerir target materializado.

---

# 27. EntityReference

Propuesta conceptual:

```php
interface EntityReference
{
    public function type(): EntityType;

    public function key(): EntityKey;

    public function isInitialized(): bool;

    public function resolve(): object;
}
```

---

# 28. EntityReference ≠ EntityManager

La referencia no deberá almacenar un EntityManager global o histórico.

---

# 29. EntityReference ≠ Proxy Necessarily

VoltStack podrá implementar referencias mediante:

- proxy;
- lazy object;
- reference wrapper;
- compiled accessor strategy.

La semántica no deberá depender de una única técnica.

---

# 30. getReference()

API conceptual:

```php
$user = $entityManager->getReference(User::class, 10);
```

podrá obtener una referencia sin ejecutar SELECT.

---

# 31. Reference ≠ Database Existence Proof

```text
getReference(User#10)
```

no prueba:

```text
users.id = 10 exists
```

---

# 32. IdentityMap Integration

Antes de crear una nueva representación de:

```text
User#10
```

deberá consultarse:

```text
IdentityMap
```

---

# 33. Canonical Instance

Si `User#10` ya está managed:

```php
$order->getUser() === $managedUser;
```

deberá ser verdadero después de resolución apropiada.

---

# 34. Proxy Canonicalization

Proxy/reference y entidad materializada no deberán convertirse en dos identidades ORM distintas.

---

# 35. Canonical EntityType

La identidad deberá usar:

```text
canonical EntityType
```

no la clase concreta de un proxy generado.

---

# 36. Identity Namespace

La key deberá incorporar el contexto requerido:

```text
Entity Type
+
Identifier
+
Tenant
+
Shard/Database Identity Namespace
```

cuando aplique.

---

# 37. Hydration

Supongamos:

```text
orders.id      = 100
orders.user_id = 10
```

sin JOIN de `users`.

Hydration puede establecer:

```text
Order#100.user
=
REFERENCE_KNOWN(User#10)
```

sin cargar User.

---

# 38. Foreign-Key Hydration

Pipeline:

```text
Result Row
   │
   ▼
Join Column Binding
   │
   ▼
DB→PHP Type Conversion
   │
   ▼
Target Identifier
   │
   ▼
Canonical EntityKey
   │
   ▼
IdentityMap Lookup
   │
   ├── hit ─────→ canonical User
   │
   └── miss ────→ EntityReference
```

---

# 39. No Hidden SELECT During Hydration

Hydratar:

```text
user_id = 10
```

no deberá provocar automáticamente:

```sql
SELECT * FROM users WHERE id = 10
```

---

# 40. JOIN Hydration

Si la consulta contiene:

```text
Order
JOIN User
```

Hydration puede materializar ambos.

---

# 41. JOIN Result

```text
Order#100 | User#10
Order#101 | User#10
Order#102 | User#10
```

deberá producir:

```text
Order#100.user ─┐
Order#101.user ─┼──→ same canonical User#10
Order#102.user ─┘
```

---

# 42. Row Count ≠ Target Count

Tres rows no significan tres Users.

---

# 43. IdentityMap Deduplication

```text
same EntityKey
⇒
same managed object
```

dentro del mismo PersistenceContext.

---

# 44. Optional JOIN

Para:

```text
LEFT JOIN users
```

si todas las columnas necesarias de identidad del target son NULL:

```text
relationship = NULL
```

cuando mapping permita NULL.

---

# 45. All-Null Identifier Rule

Conceptualmente:

```text
OptionalTargetAbsent
⇔
AllRequiredTargetIdentifierBindingsAreNull
```

---

# 46. Partial Composite Identifier

Si un ID compuesto produce:

```text
partA = 10
partB = NULL
```

cuando ambos son required:

```text
PartialRelationshipIdentifierException
```

---

# 47. NULL FK

Si todas las join columns nullable están NULL:

```text
relationship = NULL
```

---

# 48. Required FK NULL

Si mapping dice required pero resultado contiene NULL:

```text
RequiredRelationshipNullException
```

o inconsistencia equivalente.

---

# 49. Lazy Loading

Una referencia conocida:

```text
User#10
```

puede permanecer sin materializar.

---

# 50. Lazy Resolution

Acceder a datos no disponibles puede producir:

```text
EntityReference
     │
     ▼
Reference Resolver
     │
     ▼
IdentityMap
     │
     ├── hit
     │
     └── miss
           │
           ▼
      Entity Loader
           │
           ▼
       Entity Query
           │
           ▼
       Query Engine
```

---

# 51. Lazy Loading ≠ Required Behavior

Strict mode podrá prohibir:

```text
implicit database access
```

---

# 52. Detached Lazy Reference

Una referencia detached no deberá conservar acceso escondido al EntityManager anterior.

---

# 53. Detached Resolution

Si necesita DB:

```text
DetachedRelationshipReferenceException
```

---

# 54. Eager Loading

```php
Order::query()
    ->with('user')
    ->get();
```

podrá usar:

```text
JOIN_FETCH
SELECT_FETCH
BATCH_FETCH
```

---

# 55. Eager Loading ≠ JOIN

El planner podrá escoger la estrategia.

---

# 56. Batch Loading

Para:

```text
100 Orders
```

con Users:

```text
10
10
20
20
30
...
```

el batch loader puede cargar:

```text
User#10
User#20
User#30
...
```

una sola vez por identidad.

---

# 57. Batch Identity Dedup

Antes de consultar:

```text
requested target identities
↓
canonicalize
↓
deduplicate
↓
remove IdentityMap hits
↓
batch query remaining IDs
```

---

# 58. Formula

```text
BatchLoadSet
=
UniqueRequestedEntityKeys
-
IdentityMapKnownKeys
```

---

# 59. Batch Result

Cada source deberá apuntar a la instancia canónica correcta.

---

# 60. Missing Batch Target

Si FK dice:

```text
user_id = 10
```

pero batch load no encuentra `User#10`, existe una inconsistencia potencial.

---

# 61. Missing Target Policy

Propuesta:

```php
enum MissingRelationshipTargetPolicy
{
    case THROW;
    case MARK_BROKEN_REFERENCE;
    case NULL_IF_MAPPING_PERMITS;
}
```

Default recomendado:

```text
THROW
```

para una FK no-null observada.

---

# 62. Broken Reference ≠ NULL

Una referencia a entidad inexistente:

```text
BROKEN(User#10)
```

no deberá convertirse silenciosamente en:

```text
NULL
```

---

# 63. Database FK Constraint ≠ Assumed Existence

Incluso si metadata declara FK, el ORM no deberá asumir que la DB siempre está íntegra cuando una lectura demuestre lo contrario.

---

# 64. Assignment

Ejemplo:

```php
$order->setUser($user);
```

produce una mutación de relación.

---

# 65. Assignment Semantics

Conceptualmente:

```text
PreviousReference
→
CurrentReference
```

---

# 66. Relationship Snapshot

Propuesta:

```php
final readonly class ToOneRelationshipSnapshot
{
    public function __construct(
        public RelationshipId $relationship,
        public EntityKey $source,
        public ToOneSnapshotValue $value,
        public RelationshipKnowledge $knowledge,
    ) {}
}
```

---

# 67. Snapshot Value

Podrá representar:

```text
NULL
EntityKey(User#10)
UNOBSERVED
```

---

# 68. Snapshot ≠ Live Reference

Snapshot:

```text
baseline
```

Current:

```text
current object/reference
```

---

# 69. Change Detection

Ejemplo:

```text
Baseline:
User#10

Current:
User#20
```

produce:

```text
RelationshipChange(
    old = User#10,
    new = User#20
)
```

---

# 70. Reference Equality

Cambio de relación deberá compararse por identidad canónica.

No por igualdad completa de atributos.

---

# 71. Same Target Different Object

Dos objetos distintos con:

```text
EntityKey(User, 10)
```

no representan dos valores relacionales distintos.

En un contexto correcto, además, IdentityMap deberá evitar esa duplicación.

---

# 72. Nullification

```php
$order->setUser(null);
```

produce:

```text
User#10
→
NULL
```

---

# 73. Nullability Validation

Si relationship es required:

```text
NULL
```

deberá rechazarse antes de ejecutar SQL.

---

# 74. Required Relationship Invariant

```text
RequiredManyToOne
⇒
CurrentTarget ≠ NULL
```

al llegar a un persistence boundary válido.

---

# 75. Temporary Invalid Domain State

VoltStack puede permitir temporalmente durante mutaciones:

```text
old owner removed
new owner not yet assigned
```

si el graph queda válido antes del plan freeze.

---

# 76. Validation Boundary

Por ello:

```text
temporary in-memory state
≠
final persistence-valid state
```

---

# 77. Reparenting

Ejemplo:

```text
Order#100.user
User#10 → User#20
```

es:

```text
REPARENT
```

---

# 78. Reparenting ≠ Delete

Cambiar owner no implica eliminar Order.

---

# 79. Reparenting Persistence

Normalmente:

```text
orders.user_id
10 → 20
```

---

# 80. Reparenting + One-to-Many

Si existe:

```text
User.orders
```

deberá reconciliarse:

```text
User#10.orders remove Order#100
User#20.orders add Order#100
```

según la política bidireccional.

---

# 81. Owning Side Authority

Si existe contradicción:

```text
Order#100.user = User#20
```

pero:

```text
User#10.orders contains Order#100
```

la foreign-key persistence no deberá inferirse del inverse side ignorando el owning side.

---

# 82. Strict Graph Policy

Puede fallar:

```text
BidirectionalRelationshipInconsistencyException
```

antes del plan freeze.

---

# 83. Auto Fix-Up

Alternativamente:

```text
Order.user = User#20
```

puede provocar:

```text
remove from User#10.orders
add to User#20.orders
```

cuando las colecciones estén disponibles/conocidas y policy lo permita.

---

# 84. Unknown Inverse Collection

Si:

```text
User#10.orders = UNINITIALIZED
```

no será necesario cargarla solo para remover Order.

---

# 85. Critical Performance Rule

```text
ManyToOneAssignment
⇏
InitializeInverseOneToMany
```

---

# 86. Inverse Pending Mutation

En lugar de cargarla, el ORM puede registrar:

```text
inverse REMOVE intent
```

si la infraestructura de colección lo soporta.

---

# 87. Cascade Persist

Caso:

```php
$order->setUser(new User());
```

con cascade persist:

```text
User → NEW
```

en UoW.

---

# 88. Without Cascade Persist

Target transient no registrado:

```text
flush
→ TransientRelationshipEntityException
```

---

# 89. Generated Target Identifier

Caso:

```text
new User
new Order → new User
```

requiere:

```text
INSERT User
↓
generated User.id
↓
INSERT Order(user_id)
```

---

# 90. Dependency Graph

```text
Insert(User)
    │
    ▼
Identity Barrier
    │
    ▼
Insert(Order)
```

---

# 91. No Temporary FK Leakage

La identidad temporal de UoW nunca deberá enviarse como valor real:

```text
orders.user_id
```

---

# 92. Existing Source Reassigned to New Target

Caso:

```text
existing Order#100
→ new User
```

puede requerir:

```text
INSERT User
↓
generated id
↓
UPDATE Order.user_id
```

---

# 93. Persistence Change

Propuesta:

```php
final readonly class ManyToOneChange
{
    public function __construct(
        public RelationshipId $relationship,
        public EntityKey $source,
        public ToOneReferenceValue $previous,
        public ToOneReferenceValue $current,
        public ManyToOneChangeKind $kind,
    ) {}
}
```

---

# 94. Change Kinds

```php
enum ManyToOneChangeKind
{
    case ASSIGN;
    case REASSIGN;
    case NULLIFY;
}
```

---

# 95. Change ≠ SQL

`REASSIGN` no significa directamente:

```sql
UPDATE ...
```

Persistence Planner decide la representación.

---

# 96. Persistence Pipeline

```text
Relationship Change
      │
      ▼
UnitOfWork
      │
      ▼
Persistence Operation
      │
      ▼
Persistence Planner
      │
      ▼
Update Persistence System
      │
      ▼
Query Model
      │
      ▼
Query Engine
      │
      ▼
Compiler
      │
      ▼
Execution
```

---

# 97. Foreign-Key Assignment

Para una relación convencional:

```text
CurrentTarget EntityKey
```

se transforma mediante metadata/type system en:

```text
join column value(s)
```

---

# 98. Entity Object ≠ Bound Parameter

Nunca:

```text
bind($userObject)
```

como FK.

Debe extraerse:

```text
canonical identifier
```

y convertirse mediante type system.

---

# 99. Composite Foreign Key

Una relación puede usar:

```text
tenant_id
customer_id
```

como join compuesto.

---

# 100. Composite Reference

Target identity:

```text
CustomerKey(
    tenant = 5,
    customer = 100
)
```

puede mapear a múltiples columnas.

---

# 101. Composite Assignment Atomicity

Todas las partes de una referencia compuesta deberán tratarse como una unidad semántica.

---

# 102. Partial Composite Assignment

No:

```text
tenant_id = 5
customer_id = NULL
```

para identidad required compuesta.

---

# 103. Foreign-Key Type Conversion

Cada componente utilizará el `Database Type System`.

No casting ad hoc en Relationship System.

---

# 104. Target Identifier Mutation

Si identificadores establecidos son immutable por defecto:

```text
target ID mutation
```

no deberá convertirse silenciosamente en cambio de relación.

---

# 105. Query Navigation

Ejemplo:

```php
Order::query()
    ->where('user.email', 'alice@example.com');
```

deberá resolverse mediante:

```text
Relationship Metadata
→ Semantic Query Engine
→ Join/Exists Resolution
```

---

# 106. Relationship System Does Not Generate JOIN SQL

No:

```php
if ($relation instanceof ManyToOne) {
    $sql .= ' JOIN ...';
}
```

dentro del ORM.

---

# 107. Entity Query

Podrá ofrecer:

```php
Order::query()
    ->whereBelongsTo($user);
```

---

# 108. whereBelongsTo Semantics

Debe resolverse usando:

```text
relationship metadata
+
target canonical identifier
```

---

# 109. No Hardcoded `user_id`

El API no deberá asumir convenciones físicas una vez compilado el mapping.

---

# 110. Relationship Predicate

Conceptualmente:

```text
Order.user == User#10
```

se transforma a un predicate semántico.

---

# 111. Optimizer Freedom

El Query Engine podrá decidir:

- FK predicate directo;
- JOIN;
- EXISTS;
- otra forma equivalente.

---

# 112. Hydration Snapshot

Después de cargar:

```text
Order#100.user = User#10
```

el baseline deberá registrar:

```text
relationship target identity = User#10
```

---

# 113. Snapshot Does Not Require Target Fully Loaded

Puede almacenarse:

```text
EntityKey(User#10)
```

aunque `User#10` sea solo referencia.

---

# 114. Successful Initial Hydration

Antes de `postLoad`:

```text
Current Relationship
=
Snapshot Relationship
```

y:

```text
RelationshipChangeSet = empty
```

---

# 115. postLoad Mutation

Si `postLoad` hace:

```php
$order->setUser($otherUser);
```

entonces:

```text
current != baseline
```

y será dirty para futuro flush.

---

# 116. Hydration Must Not Mark Dirty

Asignaciones internas durante materialización deberán ejecutarse bajo modo de hydration.

---

# 117. Setter Side Effects

Si hydration usa setters de dominio indiscriminadamente, podría disparar:

- events;
- validation;
- inverse mutations;
- dirty tracking.

Por ello la estrategia de field/reference assignment deberá ser explícita.

---

# 118. Compiled Relationship Writer

Propuesta:

```php
interface RelationshipWriter
{
    public function writeHydratedReference(
        object $source,
        ToOneReferenceValue $value,
        HydrationAssignmentContext $context,
    ): void;
}
```

---

# 119. Internal Assignment ≠ Domain Assignment

VoltStack deberá distinguir:

```text
HYDRATION_ASSIGNMENT
DOMAIN_MUTATION
REFRESH_ASSIGNMENT
ROLLBACK_RECONCILIATION
```

---

# 120. Assignment Context

```php
enum RelationshipAssignmentMode
{
    case HYDRATE;
    case DOMAIN;
    case REFRESH;
    case RECONCILE;
}
```

---

# 121. Refresh

`refresh(Order#100)` podrá reemplazar la relación con el valor observado en DB.

---

# 122. Dirty Relationship Refresh

Si relación tiene cambios locales, refresh deberá seguir una policy.

---

# 123. Refresh Policies

```php
enum ToOneRefreshPolicy
{
    case REJECT_DIRTY;
    case DISCARD_LOCAL_CHANGE;
    case MERGE_IF_SAFE;
}
```

Default:

```text
REJECT_DIRTY
```

---

# 124. Normal Query ≠ Refresh

Una consulta normal que encuentra nuevamente `Order#100` no deberá sobrescribir ciegamente:

```text
Order.user
```

si la entidad ya está managed y dirty.

---

# 125. IdentityMap Hit Policy

```text
normal hydration
+
existing managed source
⇒
reuse canonical source
```

sin blind overwrite.

---

# 126. Partial Entity Hydration

Si la join column no fue seleccionada:

```text
Order.user
```

queda:

```text
UNLOADED
```

no `NULL`.

---

# 127. Absent Column ≠ NULL

Regla:

```text
JoinColumnAbsent
≠
JoinColumnValue(NULL)
```

---

# 128. Loaded Field/Relationship Knowledge

El ORM deberá registrar externamente qué relaciones fueron observadas.

---

# 129. Partial Entity Safety

Un flush de una entidad parcial no deberá convertir:

```text
UNLOADED relationship
```

en:

```text
NULL FK
```

---

# 130. Critical Rule

```text
UnknownRelationshipValue
⇏
NullifyRelationship
```

---

# 131. Subsequent Hydration

Una consulta posterior puede llenar una relación previamente `UNLOADED`.

---

# 132. Safe Fill

Si la relación no tiene mutación local:

```text
UNLOADED
→
REFERENCE_KNOWN / LOADED / NULL
```

puede realizarse.

---

# 133. Dirty Partial Entity

Si application code asignó relación antes de que DB baseline fuese conocida, el ORM deberá preservar la mutación y no sobrescribirla con una query normal.

---

# 134. Relationship Knowledge

Puede requerirse:

```php
enum RelationshipKnowledge
{
    case UNOBSERVED;
    case OBSERVED;
    case STALE;
    case UNCERTAIN;
}
```

---

# 135. OBSERVED ≠ Fresh Forever

Una relación observada puede volverse obsoleta por:

- external SQL;
- bulk DML;
- otro transaction;
- replica lag.

---

# 136. External Mutation

Si raw SQL modifica:

```text
orders.user_id
```

la referencia managed puede quedar:

```text
STALE
```

---

# 137. Bulk Update

Una operación bulk:

```text
UPDATE orders SET user_id = ...
```

no actualizará mágicamente todos los objetos managed.

---

# 138. Bulk Invalidation

El sistema deberá poder invalidar:

```text
relationship knowledge
```

por:

- entity;
- entity type;
- relationship;
- table;
- PersistenceContext.

---

# 139. Persistence Outcome

Estados:

```text
SUCCEEDED
FAILED
CONFLICT
NOT_EXECUTED
UNKNOWN
```

deberán preservarse.

---

# 140. Successful Reassignment

Solo con evidencia suficiente:

```text
baseline User#10
→
baseline User#20
```

---

# 141. UNKNOWN Update

Si:

```text
UPDATE sent
connection lost
```

no:

```text
baseline = User#20
```

automáticamente.

---

# 142. UNKNOWN Rule

```text
UnknownPersistenceOutcome
⇒
NoFabricatedRelationshipBaseline
```

---

# 143. Transaction Boundary

```text
statement success
≠
transaction commit
```

---

# 144. Flush Boundary

```text
flush success
≠
durable relationship change
```

si existe transacción activa.

---

# 145. Rollback

Caso:

```text
baseline User#10
current User#20
```

flush actualiza DB a User#20.

Después:

```text
ROLLBACK
```

DB vuelve a User#10.

El objeto puede seguir:

```text
current User#20
```

---

# 146. No Object Graph Time Machine

VoltStack no deberá asumir que rollback puede deshacer automáticamente toda mutación de objetos.

---

# 147. Post-Rollback State

Puede quedar:

```text
current = User#20
database = User#10
```

y deberá reconciliarse mediante la política transaccional establecida.

---

# 148. Optimistic Locking

Si source `Order` tiene:

```text
version
```

un cambio de relación puede formar parte del mismo UPDATE versionado.

---

# 149. Example

Conceptualmente:

```text
SET user_id = 20,
    version = 8

WHERE id = 100
AND version = 7
```

---

# 150. Relationship System Does Not Generate This SQL

Solo aporta el cambio semántico al `Update Persistence System`.

---

# 151. Optimistic Conflict

Si update afecta cero rows por versión:

```text
OptimisticLockConflict
```

No deberá avanzarse el relationship baseline.

---

# 152. Version Increment

El nuevo version value solo se reconcilia cuando el update es suficientemente cierto.

---

# 153. Source Deletion

Eliminar `Order` hace irrelevante persistir una reassignment de `Order.user` por separado si el planner determina que la entidad será eliminada.

---

# 154. Planner Normalization

```text
UPDATE relationship
+
DELETE source
```

puede normalizarse eliminando el UPDATE innecesario.

---

# 155. Target Deletion

Eliminar `User` cuando Orders todavía lo referencian requiere resolver:

- restrict;
- cascade delete;
- nullification;
- reassignment;
- DB cascade.

---

# 156. Relationship Metadata + FK Policy

La decisión no deberá inferirse únicamente del SQL constraint.

ORM cascade y DB referential action permanecen separadas.

---

# 157. ON DELETE SET NULL

Si DB hace:

```text
ON DELETE SET NULL
```

managed Orders pueden quedar stale.

---

# 158. DB Side Effect Reconciliation

El Persistence System deberá saber que una operación puede modificar indirectamente relaciones managed.

---

# 159. ON DELETE CASCADE

Igualmente puede eliminar Orders sin que UoW haya ejecutado per-entity delete.

---

# 160. DB Cascade ≠ Per-Entity Lifecycle

No deberán fabricarse callbacks de entidades nunca materializadas.

---

# 161. Cascade Remove on Many-to-One

Por defecto, eliminar un Order no debería eliminar automáticamente su User.

---

# 162. Dangerous Cascade

```text
Order.user cascade REMOVE
```

puede ser válido en casos especiales, pero deberá considerarse configuración de alto riesgo.

---

# 163. Mapping Safety

VoltStack puede advertir sobre cascade remove desde many-side hacia shared target.

---

# 164. Shared Target

Ejemplo:

```text
Order#1 ─┐
Order#2 ─┼── User#10
Order#3 ─┘
```

eliminar Order#1 no debe eliminar User#10 por accidente.

---

# 165. Safe Default

Para many-to-one:

```text
cascade remove = OFF
```

por defecto.

---

# 166. Cascade Persist Default

También debería ser explícito, aunque podrá existir una configuración ergonómica en Model API.

---

# 167. Relationship Existence Validation

Asignar:

```text
getReference(User#999)
```

no obliga a verificar existencia inmediatamente.

---

# 168. Validation Timing

La existencia podrá comprobarse:

- mediante FK al persistir;
- mediante explicit validation;
- mediante load;
- mediante domain rule.

---

# 169. ORM ≠ Authorization

Poder asignar:

```text
Order.user = User#10
```

no significa que el usuario HTTP tenga permiso para hacerlo.

---

# 170. ORM ≠ Input Validation

Mass assignment de:

```text
user_id
```

es una preocupación diferente.

---

# 171. Object Assignment Preferred

API de dominio:

```php
$order->setUser($user);
```

es conceptualmente más segura que exponer directamente:

```php
$order->user_id = 10;
```

---

# 172. Foreign-Key Shadow Property

VoltStack podrá soportar una propiedad FK shadow/read-only para optimización, pero deberá evitar dos fuentes independientes de verdad.

---

# 173. Critical Rule

```text
RelationshipReference
+
WritableForeignKeyScalar
```

no deberán competir como autoridades.

---

# 174. Dual Mapping Conflict

Si se permite:

```php
$order->user = $user;
$order->userId = 20;
```

y ambas son writable, aparece ambigüedad.

Default recomendado:

```text
single persistence authority
```

---

# 175. Shadow Identifier

Puede exponerse:

```php
$order->userId();
```

como vista derivada del reference state.

---

# 176. No Forced Target Load

Obtener la identidad conocida:

```text
$order->userId()
```

no debería materializar `User`.

---

# 177. Reference Identifier API

Conceptualmente:

```php
$entityManager->referenceIdentity(
    $order,
    'user'
);
```

puede devolver `EntityKey` sin lazy load.

---

# 178. Serialization

Serializar `Order` no deberá resolver `User` automáticamente.

---

# 179. API Include

Solo cuando se solicite:

```text
include=user
```

la capa superior podrá ordenar carga.

---

# 180. N+1

Caso:

```php
foreach ($orders as $order) {
    echo $order->user->name;
}
```

puede generar N+1.

---

# 181. N+1 Telemetry

Relationship Loader deberá registrar:

```text
relationship identity
source query/session
load strategy
batch opportunities
```

sin PII.

---

# 182. Batch Optimization

Si múltiples references se resuelven dentro de una ventana compatible, VoltStack podrá batch-load.

---

# 183. Batch Resolver

Conceptualmente:

```text
Reference Requests
      │
      ▼
Identity Dedup
      │
      ▼
IdentityMap Hits Removed
      │
      ▼
Entity Query IN (...)
      │
      ▼
Hydration
      │
      ▼
Resolve Waiting References
```

---

# 184. Batch Loader ≠ DataLoader Requirement

La arquitectura permite batching, pero no obliga a una implementación específica.

---

# 185. Read/Write Routing

Una relationship load es una lectura.

La decisión primary/replica pertenece a:

```text
Read/Write Routing System
```

no al Many-to-One System.

---

# 186. Read-Your-Writes

Si target fue creado/reasignado durante el contexto actual, IdentityMap/UoW puede satisfacer parte del acceso sin replica query.

---

# 187. Replica Staleness

Una referencia observada desde replica puede ser válida como:

```text
observed replica state
```

sin implicar estado primario más reciente.

---

# 188. Tenant Isolation

Para relaciones tenant-local:

```text
Tenant(Order)
=
Tenant(User)
```

---

# 189. Tenant Key

Si identidad lógica depende de tenant:

```text
EntityKey(User#10, Tenant#5)
```

deberá distinguirse de:

```text
EntityKey(User#10, Tenant#6)
```

---

# 190. Tenant Context ≠ User Input

El namespace efectivo proviene del `DatabaseContext`/Tenant Context confiable.

No de un campo arbitrario recibido del cliente.

---

# 191. Cross-Tenant Assignment

Default:

```text
REJECT
```

---

# 192. Cross-Shard Assignment

Igualmente:

```text
Shard(Order)
≠
Shard(User)
```

deberá rechazarse salvo mapping explícito.

---

# 193. Cross-Database Relationship

Puede existir como relación lógica.

Pero:

```text
database foreign key
```

puede no existir.

---

# 194. Cross-Database Persistence

La atomicidad deberá declararse explícitamente.

No asumir:

```text
distributed ACID
```

---

# 195. Relationship Consistency

La relación puede tener estados:

```text
CONSISTENT
STALE
UNCERTAIN
INCONSISTENT
TAINTED
UNKNOWN
```

según el sistema general de consistencia.

---

# 196. Consistent Relationship

Dentro del conocimiento ORM actual:

```text
current reference
baseline
owning-side state
inverse-side known state
```

no presentan contradicciones conocidas.

---

# 197. STALE

Ejemplo:

```text
bulk update may have changed FK
```

pero objeto managed conserva una representación internamente válida.

---

# 198. UNCERTAIN

Ejemplo:

```text
UPDATE outcome unknown
```

---

# 199. INCONSISTENT

Ejemplo:

```text
Order.user = User#20
User#10.orders known-complete and still claims Order
User#20.orders known-complete and claims absence
```

cuando el graph debería estar sincronizado.

---

# 200. UNKNOWN Inverse Side

No:

```text
uninitialized User.orders
→ inconsistency
```

---

# 201. Relationship Consistency Formula

```text
KnownRelationshipConsistency
=
OwningReferenceValidity
∧
IdentityValidity
∧
ContextCompatibility
∧
KnownInverseCompatibility
∧
PersistenceEvidenceCompatibility
```

---

# 202. Runtime Scope

Mutable relationship state pertenece al:

```text
PersistenceContext
```

---

# 203. Persistent Runtime

Compartible:

```text
ManyToOneMetadata
compiled accessors
join bindings
load plan templates
mapping validators
```

No compartible:

```text
EntityReference runtime state
managed entities
relationship snapshots
pending changes
lazy resolution state
batch request queues
```

---

# 204. FrankenPHP Safety

No:

```php
static array $references;
```

con entidades de requests previos.

---

# 205. Request Isolation

```text
Request A User#10
≠
Request B User#10 object identity
```

aunque compartan persistent identifier.

---

# 206. Worker Reset

Al terminar scope:

```text
clear reference resolver queues
clear relationship snapshots
release entity references
clear load guards
clear scoped batch state
```

---

# 207. Concurrent Resolution

Dos fibers intentando resolver la misma referencia deberán coordinarse.

---

# 208. Reference Resolution State

Propuesta:

```php
enum ReferenceResolutionState
{
    case UNRESOLVED;
    case RESOLVING;
    case RESOLVED;
    case FAILED;
}
```

---

# 209. Duplicate Resolution

Una segunda resolución concurrente podrá:

- esperar;
- reutilizar resultado;
- recibir el mismo error.

No deberá ejecutar arbitrariamente dos loads incompatibles.

---

# 210. Recursive Resolution

Si resolución de `Order.user` vuelve a resolver la misma referencia antes de terminar:

```text
recursive resolution guard
```

deberá evitar loops.

---

# 211. Circular Graphs

Ejemplo:

```text
Order.user
User.primaryOrder
```

puede formar ciclos.

IdentityMap reservation/hydration architecture deberá resolverlos sin duplicación.

---

# 212. Relationship Resolution Does Not Own Circular Hydration

El soporte base pertenece a:

```text
Hydration Architecture
+
IdentityMap
```

Many-to-One consume esas garantías.

---

# 213. Mapping Validation

Bootstrap deberá validar:

```text
source entity
target entity
join columns
referenced columns
identifier compatibility
nullability
ownership
inversedBy
cascade
fetch mode
type compatibility
tenant/shard constraints
```

---

# 214. inversedBy Validation

Si:

```text
Order.user inversedBy "orders"
```

entonces:

```text
User.orders
```

deberá existir y apuntar a `Order`.

---

# 215. Cardinality Pair

Esperado:

```text
Order.user       MANY_TO_ONE
User.orders      ONE_TO_MANY
```

---

# 216. Invalid Pair

No aceptar como misma asociación:

```text
Order.user       MANY_TO_ONE
User.orders      MANY_TO_MANY
```

---

# 217. Join Column Count

Mapping compuesto deberá tener suficiente información para reconstruir el target identifier.

---

# 218. Referenced Columns

No deberán apuntar arbitrariamente a columnas incompatibles con la identidad/unique-key semantics declarada.

---

# 219. Non-PK Unique Reference

VoltStack podrá soportar referencias a unique keys.

Ejemplo:

```text
orders.customer_code
→
customers.code UNIQUE
```

---

# 220. Entity Identity ≠ Referenced Column Necessarily

Sin embargo, el ORM deberá poder resolver esa referencia hacia el `EntityKey` canónico.

---

# 221. Alternate Key Resolution

Si join utiliza alternate key pero IdentityMap usa PK:

```text
alternate key
→ canonical entity identity
```

puede requerir resolución adicional.

---

# 222. V1 Recommendation

Para mantener invariantes fuertes y evitar hidden lookups:

```text
ManyToOne join → target canonical identifier
```

deberá ser el default y la ruta optimizada.

Alternate-key references pueden quedar como capability avanzada.

---

# 223. Foreign-Key Constraint Optionality

El ORM podrá mapear una asociación aunque DB no tenga FK física.

Pero diagnostics deberá distinguir:

```text
mapped relationship
```

de:

```text
database-enforced relationship
```

---

# 224. Schema Validation

Una herramienta de validación podrá comparar:

```text
ORM Mapping
↔
Schema Metadata
```

sin convertir mapping en schema definition.

---

# 225. Developer API

Ejemplo Data Mapper:

```php
$order = $orders->find(100);

$user = $users->find(10);

$order->setUser($user);

$entityManager->flush();
```

---

# 226. Model API

Ejemplo ergonómico:

```php
$order->user()->associate($user);

$order->save();
```

pero:

```text
associate()
```

deberá delegar al mismo relationship/UoW engine.

---

# 227. Model API ≠ Second Relationship Engine

No habrá implementación independiente para Active Record.

---

# 228. Dissociate

```php
$order->user()->dissociate();
```

solo será válido si mapping nullable/policy lo permite.

---

# 229. associateById

Podría existir:

```php
$order->user()->associateId(10);
```

mediante `EntityReference`.

No deberá ejecutar SELECT obligatoriamente.

---

# 230. associateId Safety

Deberá:

1. canonicalizar ID;
2. construir EntityKey;
3. validar context namespace;
4. consultar IdentityMap;
5. usar canonical object/reference;
6. registrar mutation.

---

# 231. No Fake Entity

No:

```php
$user = new User();
$user->id = 10;
```

como implementación interna de `associateId()`.

---

# 232. Reference Factory

Utilizar:

```text
EntityReferenceFactory
```

o `EntityManager::getReference()`.

---

# 233. Relationship Query API

```php
$order->user();
```

puede devolver un relationship query object.

---

# 234. Relationship Query ≠ Current Relationship Value

Debe distinguirse:

```php
$order->user;
```

de:

```php
$order->user();
```

si Model API adopta sintaxis Laravel-like.

---

# 235. Query Object

Podrá permitir:

```php
$order->user()
    ->select(...)
    ->first();
```

sin alterar necesariamente el current managed relationship.

---

# 236. Query Result ≠ Assignment

Consultar un User compatible no significa:

```text
Order.user = queried User
```

automáticamente.

---

# 237. Relationship Loading API

Debe existir una operación explícita para cargar el mapping en la entidad cuando eso sea el objetivo.

---

# 238. Telemetry

Métricas sugeridas:

```text
orm.relationship.many_to_one.loads
orm.relationship.many_to_one.lazy_loads
orm.relationship.many_to_one.eager_loads
orm.relationship.many_to_one.batch_loads

orm.relationship.many_to_one.references.created
orm.relationship.many_to_one.references.resolved
orm.relationship.many_to_one.references.identity_map_hits

orm.relationship.many_to_one.assignments
orm.relationship.many_to_one.reassignments
orm.relationship.many_to_one.nullifications

orm.relationship.many_to_one.broken_references
orm.relationship.many_to_one.missing_targets

orm.relationship.many_to_one.cross_context_failures
orm.relationship.many_to_one.tenant_violations

orm.relationship.many_to_one.refreshes
orm.relationship.many_to_one.refresh_conflicts

orm.relationship.many_to_one.persistence.unknown
```

---

# 239. Telemetry Privacy

No utilizar como labels:

```text
user_id
order_id
email
tenant secrets
identifier values
```

---

# 240. Diagnostics

Ejemplo:

```text
MANY-TO-ONE RELATIONSHIP

Relationship:
    Order.user

Target:
    User

Ownership:
    OWNING

Join:
    orders.user_id
    → users.id

Nullability:
    REQUIRED

Fetch:
    LAZY

Reference State:
    REFERENCE_KNOWN

Target Identity:
    [redacted]

Target Loaded:
    NO

Dirty:
    NO

Inverse:
    User.orders
```

---

# 241. Dirty Diagnostic

```text
RELATIONSHIP CHANGE

Relationship:
    Order.user

Change:
    REASSIGN

Baseline:
    User#[redacted]

Current:
    User#[redacted]

Inverse State:
    PARTIALLY KNOWN

Persistence:
    PENDING
```

---

# 242. Broken Reference Diagnostic

```text
BROKEN RELATIONSHIP REFERENCE

Relationship:
    Order.user

Observed FK:
    [redacted]

Target:
    User

Load Outcome:
    TARGET_NOT_FOUND

Mapping:
    REQUIRED

Action:
    Relationship integrity must be reconciled.
```

---

# 243. Error Taxonomy

```text
ManyToOneRelationshipException
├── ManyToOneMetadataException
├── ManyToOneMappingException
├── ManyToOneOwnershipException
├── ManyToOneNullabilityException
├── RequiredRelationshipNullException
├── RelationshipReferenceException
├── RelationshipReferenceResolutionException
├── DetachedRelationshipReferenceException
├── BrokenRelationshipReferenceException
├── MissingRelationshipTargetException
├── PartialRelationshipIdentifierException
├── RelationshipAssignmentException
├── RelationshipReassignmentException
├── RelationshipNullificationException
├── RelationshipChangeTrackingException
├── RelationshipPersistenceException
├── RelationshipRefreshException
├── RelationshipInverseSynchronizationException
├── BidirectionalRelationshipInconsistencyException
├── RelationshipCrossContextException
├── RelationshipTenantIsolationException
├── RelationshipShardIsolationException
├── RelationshipConcurrentResolutionException
├── RelationshipRecursiveResolutionException
├── RelationshipRuntimeIsolationException
└── ManyToOneInvariantException
```

---

# 244. Directory Structure

```text
src/Quantum/Database/ORM/Relationship/ManyToOne/
│
├── Metadata/
│   ├── ManyToOneMetadata.php
│   ├── JoinColumnMetadata.php
│   └── RelationshipNullability.php
│
├── Reference/
│   ├── EntityReference.php
│   ├── ManagedEntityReference.php
│   ├── EntityReferenceFactory.php
│   ├── ReferenceResolutionState.php
│   └── RelationshipReferenceResolver.php
│
├── State/
│   ├── ToOneReferenceState.php
│   ├── ToOneReferenceValue.php
│   └── RelationshipKnowledge.php
│
├── Loading/
│   ├── ManyToOneLoader.php
│   ├── ManyToOneBatchLoader.php
│   ├── RelationshipLoadPlan.php
│   └── ReferenceResolutionGuard.php
│
├── Hydration/
│   ├── ManyToOneHydrationBinding.php
│   ├── ManyToOneHydrator.php
│   └── RelationshipWriter.php
│
├── Snapshot/
│   ├── ToOneRelationshipSnapshot.php
│   └── ToOneSnapshotValue.php
│
├── ChangeTracking/
│   ├── ManyToOneChange.php
│   ├── ManyToOneChangeKind.php
│   └── ManyToOneChangeTracker.php
│
├── Persistence/
│   ├── ManyToOnePersistenceOperation.php
│   ├── AssociateReferenceOperation.php
│   ├── ReassignReferenceOperation.php
│   └── NullifyReferenceOperation.php
│
├── Synchronization/
│   ├── InverseCollectionSynchronizer.php
│   └── RelationshipSynchronizationLedger.php
│
├── Policy/
│   ├── MissingRelationshipTargetPolicy.php
│   ├── ToOneRefreshPolicy.php
│   └── RelationshipAssignmentPolicy.php
│
├── Validation/
│   ├── ManyToOneMappingValidator.php
│   └── ManyToOneGraphValidator.php
│
├── Telemetry/
│   ├── ManyToOneTelemetry.php
│   └── ManyToOneDiagnostics.php
│
└── Exception/
    └── ...
```

---

# 245. Testing Strategy

Deberá cubrir al menos:

```text
required relationship
nullable relationship
unloaded reference
known identity reference
loaded reference
NULL reference
lazy resolution
eager JOIN
batch loading
IdentityMap reuse
reference without SELECT
broken reference
composite identifier
assignment
nullification
reparenting
inverse synchronization
cascade persist
generated target ID
partial hydration
refresh
optimistic locking
rollback
UNKNOWN outcome
tenant isolation
persistent runtime
concurrent resolution
```

---

# 246. Required Relationship Test

```text
Order.user = NULL
```

antes de persistence boundary:

```text
validation failure
```

---

# 247. Nullable Test

```text
User#10 → NULL
```

produce nullification válida.

---

# 248. Unloaded Test

Join column no seleccionada:

```text
state = UNLOADED
```

no `NULL`.

---

# 249. Reference-Known Test

FK observada sin target columns:

```text
state = REFERENCE_KNOWN
```

sin SELECT adicional.

---

# 250. IdentityMap Hit Test

Si User#10 ya está managed:

```text
Order.user === User#10 canonical instance
```

---

# 251. getReference Test

```php
$entityManager->getReference(User::class, 10);
```

no ejecuta query.

---

# 252. Broken Reference Test

FK conocida pero target ausente:

```text
MissingRelationshipTargetException
```

según default policy.

---

# 253. JOIN Dedup Test

100 Orders referenciando User#10 producen un solo User#10 managed.

---

# 254. Composite ID Test

Todas las join columns son canonicalizadas correctamente.

---

# 255. Partial Composite ID Test

Parte NULL inesperada:

```text
PartialRelationshipIdentifierException
```

---

# 256. Assignment Test

```text
User#10 → User#20
```

produce un único `REASSIGN`.

---

# 257. Same Identity Assignment Test

Asignar nuevamente canonical User#10:

```text
no relationship change
```

---

# 258. Nullification Required Test

Required mapping rechaza:

```text
User#10 → NULL
```

antes de SQL.

---

# 259. Reparenting Inverse Test

Mover Order entre Users actualiza inverse collections conocidas sin inicializarlas obligatoriamente.

---

# 260. Cascade Persist Test

NEW User se registra cuando mapping lo permite.

---

# 261. Generated ID Test

Order no se persiste con FK temporal.

---

# 262. Partial Entity Flush Test

UNLOADED relationship no se convierte en NULL.

---

# 263. postLoad Mutation Test

Cambio realizado en `postLoad` queda dirty.

---

# 264. Normal Requery Test

Entidad managed dirty no es sobrescrita por hydration normal.

---

# 265. Refresh Dirty Test

Default:

```text
REJECT_DIRTY
```

---

# 266. Optimistic Lock Test

Conflict no avanza relationship snapshot.

---

# 267. UNKNOWN Test

Outcome incierto no fabrica baseline nuevo.

---

# 268. Rollback Test

Rollback DB no reescribe automáticamente object reference.

---

# 269. Tenant Isolation Test

Target de otro tenant es rechazado.

---

# 270. Cross-Context Test

Target managed por PersistenceContext incompatible es rechazado o explícitamente merged según policy; nunca aceptado silenciosamente.

---

# 271. Persistent Runtime Test

Dos requests no comparten EntityReference runtime state.

---

# 272. Concurrent Resolution Test

Dos fibers no generan dos canonical targets.

---

# 273. Architectural Invariants

## DB-ORM-MANY-TO-ONE-001

Many-to-one será una relación to-one identity-aware.

## DB-ORM-MANY-TO-ONE-002

Many-to-one será distinto de foreign key.

## DB-ORM-MANY-TO-ONE-003

Many-to-one será distinto de PHP object reference.

## DB-ORM-MANY-TO-ONE-004

Many-to-one será distinto de one-to-one.

## DB-ORM-MANY-TO-ONE-005

En mapping FK convencional, many-to-one será owning side.

## DB-ORM-MANY-TO-ONE-006

Ownership será distinto de object location.

## DB-ORM-MANY-TO-ONE-007

ORM ownership será distinto de aggregate ownership.

## DB-ORM-MANY-TO-ONE-008

ManyToOneMetadata será immutable.

## DB-ORM-MANY-TO-ONE-009

Mapping metadata será distinto de schema metadata.

## DB-ORM-MANY-TO-ONE-010

Required y nullable serán semánticas explícitas.

## DB-ORM-MANY-TO-ONE-011

NULL será distinto de UNLOADED.

## DB-ORM-MANY-TO-ONE-012

UNKNOWN será distinto de NULL.

## DB-ORM-MANY-TO-ONE-013

REFERENCE_KNOWN será distinto de LOADED.

## DB-ORM-MANY-TO-ONE-014

Target identity podrá conocerse sin target materializado.

## DB-ORM-MANY-TO-ONE-015

EntityReference será distinto de EntityManager.

## DB-ORM-MANY-TO-ONE-016

EntityReference no requerirá una implementación proxy específica.

## DB-ORM-MANY-TO-ONE-017

getReference no probará existencia en DB.

## DB-ORM-MANY-TO-ONE-018

IdentityMap será consultado antes de crear representación target.

## DB-ORM-MANY-TO-ONE-019

Canonical target será único por EntityKey dentro del scope.

## DB-ORM-MANY-TO-ONE-020

Proxy type no alterará canonical EntityType.

## DB-ORM-MANY-TO-ONE-021

Identity namespace preservará tenant/shard context.

## DB-ORM-MANY-TO-ONE-022

FK hydration no ejecutará hidden SELECT.

## DB-ORM-MANY-TO-ONE-023

JOIN hydration reutilizará canonical target.

## DB-ORM-MANY-TO-ONE-024

Row count será distinto de target entity count.

## DB-ORM-MANY-TO-ONE-025

All-null optional target identity podrá representar NULL.

## DB-ORM-MANY-TO-ONE-026

Partial composite target identity inválida será rechazada.

## DB-ORM-MANY-TO-ONE-027

Required relationship con NULL observado será inconsistencia/error.

## DB-ORM-MANY-TO-ONE-028

Lazy resolution utilizará Entity Query/Query Engine.

## DB-ORM-MANY-TO-ONE-029

Lazy loading podrá prohibirse.

## DB-ORM-MANY-TO-ONE-030

Detached reference no conservará hidden manager access.

## DB-ORM-MANY-TO-ONE-031

Eager loading será distinto de JOIN.

## DB-ORM-MANY-TO-ONE-032

Batch loading deduplicará target identities.

## DB-ORM-MANY-TO-ONE-033

IdentityMap hits serán excluidos del batch load cuando sea posible.

## DB-ORM-MANY-TO-ONE-034

Missing target será distinto de NULL relationship.

## DB-ORM-MANY-TO-ONE-035

Broken reference no será silenciado por default.

## DB-ORM-MANY-TO-ONE-036

Assignment será relationship mutation.

## DB-ORM-MANY-TO-ONE-037

Relationship snapshot será distinto de current reference.

## DB-ORM-MANY-TO-ONE-038

Change detection comparará canonical target identity.

## DB-ORM-MANY-TO-ONE-039

Same target identity no producirá reassignment.

## DB-ORM-MANY-TO-ONE-040

Nullification validará nullability.

## DB-ORM-MANY-TO-ONE-041

Temporary invalid graph podrá existir antes de stabilization.

## DB-ORM-MANY-TO-ONE-042

Persistence-valid graph deberá existir antes de plan freeze.

## DB-ORM-MANY-TO-ONE-043

Reparenting será distinto de delete.

## DB-ORM-MANY-TO-ONE-044

Reparenting actualizará owning-side reference.

## DB-ORM-MANY-TO-ONE-045

Owning side será autoridad ante persistence.

## DB-ORM-MANY-TO-ONE-046

Known bidirectional contradiction podrá fallar en strict mode.

## DB-ORM-MANY-TO-ONE-047

Auto-fixup será cycle-safe.

## DB-ORM-MANY-TO-ONE-048

Many-to-one assignment no inicializará inverse collection obligatoriamente.

## DB-ORM-MANY-TO-ONE-049

Cascade persist será explícito.

## DB-ORM-MANY-TO-ONE-050

Transient target sin persist/cascade será rechazado.

## DB-ORM-MANY-TO-ONE-051

Generated target ID será dependency barrier.

## DB-ORM-MANY-TO-ONE-052

Temporary UoW identity nunca será FK física.

## DB-ORM-MANY-TO-ONE-053

Relationship Change será distinto de SQL.

## DB-ORM-MANY-TO-ONE-054

Persistence utilizará canonical target identifier.

## DB-ORM-MANY-TO-ONE-055

Entity object no será bound como FK.

## DB-ORM-MANY-TO-ONE-056

Composite reference será tratada como unidad semántica.

## DB-ORM-MANY-TO-ONE-057

Partial composite assignment inválida será rechazada.

## DB-ORM-MANY-TO-ONE-058

FK conversion utilizará Database Type System.

## DB-ORM-MANY-TO-ONE-059

Target identifier mutation no será relationship mutation implícita.

## DB-ORM-MANY-TO-ONE-060

Relationship navigation utilizará Semantic Query Engine.

## DB-ORM-MANY-TO-ONE-061

Relationship System no generará JOIN SQL.

## DB-ORM-MANY-TO-ONE-062

whereBelongsTo utilizará metadata, no hardcoded column names.

## DB-ORM-MANY-TO-ONE-063

Hydration snapshot podrá almacenar EntityKey sin target materializado.

## DB-ORM-MANY-TO-ONE-064

Initial successful hydration producirá empty relationship changeset.

## DB-ORM-MANY-TO-ONE-065

postLoad mutation será future dirty work.

## DB-ORM-MANY-TO-ONE-066

Hydration assignment no será domain mutation.

## DB-ORM-MANY-TO-ONE-067

Assignment mode será explícito internamente.

## DB-ORM-MANY-TO-ONE-068

Dirty refresh tendrá policy.

## DB-ORM-MANY-TO-ONE-069

Normal query será distinta de refresh.

## DB-ORM-MANY-TO-ONE-070

Normal hydration no sobrescribirá blindamente managed dirty relationship.

## DB-ORM-MANY-TO-ONE-071

Absent join column será distinto de NULL join column.

## DB-ORM-MANY-TO-ONE-072

Partial entity conservará relationship loaded knowledge.

## DB-ORM-MANY-TO-ONE-073

UNLOADED relationship nunca generará nullification implícita.

## DB-ORM-MANY-TO-ONE-074

Subsequent hydration podrá completar relationship knowledge cuando sea seguro.

## DB-ORM-MANY-TO-ONE-075

Application mutation tendrá prioridad sobre normal rehydration overwrite.

## DB-ORM-MANY-TO-ONE-076

OBSERVED relationship podrá volverse STALE.

## DB-ORM-MANY-TO-ONE-077

Raw/bulk mutations no actualizarán mágicamente managed references.

## DB-ORM-MANY-TO-ONE-078

Bulk operations podrán invalidar relationship knowledge.

## DB-ORM-MANY-TO-ONE-079

Persistence outcome UNKNOWN será first-class.

## DB-ORM-MANY-TO-ONE-080

UNKNOWN outcome no avanzará relationship baseline.

## DB-ORM-MANY-TO-ONE-081

Statement success será distinto de transaction commit.

## DB-ORM-MANY-TO-ONE-082

Flush success será distinto de durable relationship change.

## DB-ORM-MANY-TO-ONE-083

Rollback no rebobinará object graph automáticamente.

## DB-ORM-MANY-TO-ONE-084

Optimistic relationship update no avanzará baseline ante conflict.

## DB-ORM-MANY-TO-ONE-085

Version increment será reconciliado solo con outcome suficientemente cierto.

## DB-ORM-MANY-TO-ONE-086

Planner podrá eliminar relationship update eclipsado por source delete.

## DB-ORM-MANY-TO-ONE-087

Target delete deberá considerar referential semantics.

## DB-ORM-MANY-TO-ONE-088

ORM cascade será distinto de DB referential action.

## DB-ORM-MANY-TO-ONE-089

DB-side SET NULL podrá volver managed references stale.

## DB-ORM-MANY-TO-ONE-090

DB cascade no fabricará per-entity lifecycle.

## DB-ORM-MANY-TO-ONE-091

Cascade remove desde many-side será off por default.

## DB-ORM-MANY-TO-ONE-092

Shared target será protegido de cascade remove accidental.

## DB-ORM-MANY-TO-ONE-093

EntityReference existence no requerirá validación inmediata.

## DB-ORM-MANY-TO-ONE-094

ORM relationship será distinto de authorization.

## DB-ORM-MANY-TO-ONE-095

ORM relationship será distinto de input validation.

## DB-ORM-MANY-TO-ONE-096

Writable FK scalar no competirá con relationship authority por default.

## DB-ORM-MANY-TO-ONE-097

Shadow FK identity podrá obtenerse sin target load.

## DB-ORM-MANY-TO-ONE-098

Serialization no resolverá relationship obligatoriamente.

## DB-ORM-MANY-TO-ONE-099

N+1 será observable.

## DB-ORM-MANY-TO-ONE-100

Batch resolution preservará canonical identity.

## DB-ORM-MANY-TO-ONE-101

Read/write routing no pertenecerá al Relationship System.

## DB-ORM-MANY-TO-ONE-102

Replica observation no implicará primary freshness.

## DB-ORM-MANY-TO-ONE-103

Tenant-local relationship preservará tenant identity.

## DB-ORM-MANY-TO-ONE-104

Tenant namespace no provendrá de input no confiable.

## DB-ORM-MANY-TO-ONE-105

Cross-tenant assignment será rechazado por default.

## DB-ORM-MANY-TO-ONE-106

Cross-shard assignment será rechazado salvo mapping explícito.

## DB-ORM-MANY-TO-ONE-107

Cross-database relationship no implicará distributed transaction.

## DB-ORM-MANY-TO-ONE-108

Relationship consistency preservará uncertainty.

## DB-ORM-MANY-TO-ONE-109

Unknown inverse collection no será inconsistency.

## DB-ORM-MANY-TO-ONE-110

Known contradiction sí será inconsistency.

## DB-ORM-MANY-TO-ONE-111

Mutable relationship runtime state será PersistenceContext-scoped.

## DB-ORM-MANY-TO-ONE-112

ManyToOneMetadata podrá compartirse entre workers solo si es immutable.

## DB-ORM-MANY-TO-ONE-113

EntityReference runtime state no será process-global.

## DB-ORM-MANY-TO-ONE-114

Relationship snapshots no cruzarán requests.

## DB-ORM-MANY-TO-ONE-115

Batch resolution queues serán scoped.

## DB-ORM-MANY-TO-ONE-116

Persistent workers liberarán entity references al reset.

## DB-ORM-MANY-TO-ONE-117

Concurrent resolution será coordinada.

## DB-ORM-MANY-TO-ONE-118

Recursive resolution será detectada.

## DB-ORM-MANY-TO-ONE-119

Circular graphs utilizarán IdentityMap/Hydration reservation.

## DB-ORM-MANY-TO-ONE-120

Mapping será validado durante bootstrap.

## DB-ORM-MANY-TO-ONE-121

inversedBy deberá apuntar a relación compatible.

## DB-ORM-MANY-TO-ONE-122

Many-to-one/one-to-many pair tendrá cardinalidad compatible.

## DB-ORM-MANY-TO-ONE-123

Join column types serán compatibles con target identity.

## DB-ORM-MANY-TO-ONE-124

Composite joins serán completos y deterministas.

## DB-ORM-MANY-TO-ONE-125

Non-PK alternate-key relationships requerirán semántica explícita.

## DB-ORM-MANY-TO-ONE-126

Canonical-ID joins serán la ruta recomendada por default.

## DB-ORM-MANY-TO-ONE-127

ORM mapping no asumirá existencia de DB FK física.

## DB-ORM-MANY-TO-ONE-128

Schema validation será una preocupación separada.

## DB-ORM-MANY-TO-ONE-129

Model API delegará al mismo relationship engine.

## DB-ORM-MANY-TO-ONE-130

associate no será segundo persistence mechanism.

## DB-ORM-MANY-TO-ONE-131

dissociate respetará nullability.

## DB-ORM-MANY-TO-ONE-132

associateId utilizará EntityReference, no fake entity.

## DB-ORM-MANY-TO-ONE-133

Relationship Query será distinto de current relationship value.

## DB-ORM-MANY-TO-ONE-134

Relationship Query result no implicará assignment.

## DB-ORM-MANY-TO-ONE-135

Relationship loading será explícito cuando modifique managed state.

## DB-ORM-MANY-TO-ONE-136

No se fabricará NULL a partir de UNLOADED.

## DB-ORM-MANY-TO-ONE-137

No se fabricará target existence a partir de reference identity.

## DB-ORM-MANY-TO-ONE-138

No se fabricará clean baseline después de UNKNOWN.

## DB-ORM-MANY-TO-ONE-139

No se fabricará inverse consistency a partir de inverse state desconocido.

## DB-ORM-MANY-TO-ONE-140

No se fabricará authorization a partir de relationship mapping.

## DB-ORM-MANY-TO-ONE-141

No se fabricará distributed atomicity para cross-database references.

## DB-ORM-MANY-TO-ONE-142

Relationship persistence respetará UoW stabilization.

## DB-ORM-MANY-TO-ONE-143

Lifecycle mutations antes de plan freeze serán recollected.

## DB-ORM-MANY-TO-ONE-144

Frozen plan no cambiará por relationship mutation silenciosa.

## DB-ORM-MANY-TO-ONE-145

post lifecycle relationship mutation será trabajo del siguiente flush.

## DB-ORM-MANY-TO-ONE-146

Relationship change tracking será identity-based.

## DB-ORM-MANY-TO-ONE-147

Relationship state será distinto de entity state.

## DB-ORM-MANY-TO-ONE-148

Relationship snapshot será distinto de entity snapshot aunque pueda integrarse con él.

## DB-ORM-MANY-TO-ONE-149

Relationship loader no interpretará persistence operations.

## DB-ORM-MANY-TO-ONE-150

Persistence system no realizará lazy loading para descubrir arbitrariamente la relación.

## DB-ORM-MANY-TO-ONE-151

Planner trabajará con relationship changes semánticamente estabilizados.

## DB-ORM-MANY-TO-ONE-152

Generated identity dependencies serán explícitas.

## DB-ORM-MANY-TO-ONE-153

Context compatibility será validada antes de persistence.

## DB-ORM-MANY-TO-ONE-154

Relationship reference resolution no modificará ownership semántico.

## DB-ORM-MANY-TO-ONE-155

IdentityMap canonicalization será preservada durante lazy/eager/batch loading.

## DB-ORM-MANY-TO-ONE-156

Target hydration failure no dejará una referencia falsamente RESOLVED.

## DB-ORM-MANY-TO-ONE-157

Failed resolution no convertirá reference en NULL.

## DB-ORM-MANY-TO-ONE-158

Cancellation no fabricará target absence.

## DB-ORM-MANY-TO-ONE-159

Relationship diagnostics preservarán conocimiento y uncertainty.

## DB-ORM-MANY-TO-ONE-160

VoltStack mantendrá separadas target identity, target materialization, NULL, loaded state, ownership, inverse knowledge y persistence outcome.

---

# 274. Fórmulas fundamentales

## 274.1 Cardinalidad

Para source `S` y target type `T`:

```text
ManyToOne(S, T)
⇒
Target(S) ∈ T ∪ {NULL}
```

cuando nullable.

Para required:

```text
Target(S) ∈ T
```

---

# 275. Reference Knowledge

```text
RelationshipReferenceKnowledge
=
TargetIdentityKnowledge
+
MaterializationState
+
PersistenceKnowledge
```

---

# 276. Canonical Reference

```text
EntityKeyKnown
∧
IdentityMapHit
⇒
UseCanonicalTargetInstance
```

---

# 277. Reference Without Load

```text
FKObserved
∧
ValidCanonicalIdentifier
∧
IdentityMapMiss
⇒
REFERENCE_KNOWN
```

no:

```text
Mandatory SELECT
```

---

# 278. Null Semantics

```text
KnownNullRelationship
=
JoinColumnsObserved
∧
AllRequiredJoinValuesNull
∧
MappingAllowsNull
```

---

# 279. Change Detection

```text
RelationshipDirty
=
Canonical(CurrentTarget)
≠
Canonical(BaselineTarget)
```

con semántica especial para:

```text
NULL
UNOBSERVED
```

---

# 280. Reparenting

```text
Reparent
=
BaselineTarget = A
∧
CurrentTarget = B
∧
A ≠ B
∧
A ≠ NULL
∧
B ≠ NULL
```

---

# 281. Nullification

```text
Nullify
=
BaselineTarget ≠ NULL
∧
CurrentTarget = NULL
```

---

# 282. Assignment

```text
Assign
=
BaselineTarget = NULL
∧
CurrentTarget ≠ NULL
```

---

# 283. Generated Identity Dependency

```text
Persist(Source → NewTarget)
=
Persist(Target)
→
ResolveTargetIdentity
→
PersistSourceForeignKey
```

---

# 284. Partial Safety

```text
RelationshipUnobserved
⇒
NoImplicitForeignKeyMutation
```

---

# 285. Persistence Certainty

```text
AdvanceRelationshipBaseline
⇒
OperationOutcomeSufficientlyCertain
```

---

# 286. Runtime Safety

```text
SafeManyToOneRuntime
=
ImmutableSharedMetadata
∧
ScopedReferences
∧
ScopedSnapshots
∧
ScopedLoadState
∧
ScopedIdentityMap
∧
NoCrossRequestEntityReferences
```

---

# 287. Master Formula

```text
Database Many-to-One Relationship System
=
ManyToOne Metadata
+
Owning-Side Semantics
+
Join Column Mapping
+
Required/Nullable Semantics
+
Reference State
+
Target Identity Knowledge
+
EntityReference
+
IdentityMap Canonicalization
+
Foreign-Key Hydration
+
JOIN Hydration
+
Lazy Resolution
+
Eager Loading
+
Batch Loading
+
Missing Target Detection
+
Relationship Snapshots
+
Identity-Based Change Tracking
+
Assignment
+
Reassignment
+
Nullification
+
Reparenting
+
Inverse One-to-Many Synchronization
+
Cascade Persist
+
Generated Identity Dependencies
+
Composite References
+
Partial Hydration Safety
+
Refresh Policies
+
Optimistic Locking Integration
+
Persistence Outcome Certainty
+
Transaction Awareness
+
Tenant/Shard Isolation
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

# 288. Regla maestra final

> **VoltStack nunca confundirá “no cargado”, “identidad conocida”, “entidad materializada”, “NULL” y “target inexistente”. La relación many-to-one conservará estas condiciones como estados semánticamente diferentes, y solamente el owning side estabilizado podrá determinar la foreign key que será entregada al Persistence Engine.**

Esto evita errores como:

```text
UNLOADED → NULL
```

```text
reference → assumed existence
```

```text
inverse collection → FK authority
```

```text
UNKNOWN persistence → clean baseline
```

---

# 289. Arquitectura resultante

```text
                         ManyToOneMetadata
                                │
                 ┌──────────────┴──────────────┐
                 │                             │
                 ▼                             ▼
          Relationship State             Query Navigation
                 │                             │
     ┌───────────┼───────────┐                 ▼
     │           │           │          Semantic Query Engine
     ▼           ▼           ▼
 UNLOADED    REFERENCE      NULL
               KNOWN
                 │
                 ▼
            IdentityMap
                 │
          ┌──────┴──────┐
          │             │
          ▼             ▼
      canonical       lazy/batch
       entity          resolution
          │             │
          └──────┬──────┘
                 ▼
               LOADED
                 │
                 ▼
        Relationship Snapshot
                 │
                 ▼
          Change Tracking
                 │
                 ▼
       Stabilized Relationship
                 │
                 ▼
             UnitOfWork
                 │
                 ▼
       Persistence Planner
                 │
                 ▼
      Update/Insert Persistence
                 │
                 ▼
            Query Engine
                 │
                 ▼
         Execution Engine
```

La separación fundamental queda establecida como:

```text
Target Identity
≠
Target Object
≠
Relationship Load State
≠
Foreign Key
≠
Inverse Collection
≠
Persistence Outcome
```

---

# 290. Siguiente documento

```text
146_DATABASE_MANY_TO_MANY_RELATIONSHIP_SYSTEM.md
```

El siguiente documento definirá relaciones:

```text
User N ───── N Role
Student N ── N Course
Post N ───── N Tag
Product N ── N Category
```

incluyendo:

```text
ManyToManyMetadata
join tables
association identity
owning/inverse semantics
PersistentCollection integration
pivot/association rows
pure join tables
association entities
composite join keys
lazy loading
eager loading
batch loading
JOIN hydration
deduplication
collection completeness
add/remove/sync/clear
relationship snapshots
membership change tracking
cascade persist
generated identifiers
ordering
extra-lazy operations
large collections
duplicate membership prevention
join-row persistence
transaction consistency
tenant/shard isolation
persistent runtime
telemetry
testing
```

La regla central será:

> **Una relación many-to-many de VoltStack representará una asociación de membresía entre dos conjuntos de identidades; la fila de unión será una representación persistente de esa membresía, pero no será confundida con una entidad de dominio salvo que el mapping la modele explícitamente como Association Entity.**