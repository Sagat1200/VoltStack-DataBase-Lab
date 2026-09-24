# 142_DATABASE_RELATIONSHIP_ARCHITECTURE.md

# VoltStack Quantum Database
## Database Relationship Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 142 — Database Relationship Architecture  
**Bloque:** 13 — Relationships  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Relationship Architecture` define el modelo común utilizado por VoltStack para representar, cargar, hidratar, modificar y persistir asociaciones entre entidades.

Una relación ORM representa una conexión semántica entre entidades del modelo de dominio.

Ejemplos:

```text
User
 └── Profile

User
 └── Orders[]

Order
 └── User

Post
 └── Tags[]

Comment
 └── Commentable
```

La arquitectura deberá soportar:

- one-to-one;
- one-to-many;
- many-to-one;
- many-to-many;
- relaciones polimórficas;
- relaciones unidireccionales;
- relaciones bidireccionales;
- ownership;
- inverse side;
- cascades;
- orphan removal;
- eager loading;
- lazy loading;
- batch loading;
- relationship snapshots;
- persistence planning;
- IdentityMap;
- UnitOfWork;
- proxies;
- persistent collections;
- graph hydration;
- N+1 detection.

---

# 2. Principio central

> **Una relación ORM representa una asociación semántica entre entidades. Su representación física puede utilizar foreign keys, join tables, discriminator columns u otras estrategias, pero ninguna de esas estructuras físicas constituye por sí sola la relación ORM.**

Formalmente:

```text
ORM Relationship
≠
Foreign Key
```

y:

```text
ORM Relationship
≠
Join Table
```

---

# 3. Distinciones fundamentales

La arquitectura deberá preservar permanentemente:

```text
Relationship
≠
ForeignKey

Relationship Metadata
≠
Schema Metadata

Relationship Loading
≠
Hydration

Relationship Persistence
≠
Entity Persistence

Relationship Collection
≠
Query Result Collection

Cascade Persist
≠
Database CASCADE

Cascade Remove
≠
Orphan Removal

Owning Side
≠
Parent Entity

Inverse Side
≠
Child Entity

Loaded
≠
Initialized

Initialized
≠
Complete

Relationship Graph
≠
Persistence Graph

Lazy Loading
≠
Proxy necesariamente

Eager Loading
≠
JOIN necesariamente
```

---

# 4. Posición arquitectónica

```text
                  Entity Metadata
                        │
                        ▼
              Relationship Metadata
                        │
          ┌─────────────┼──────────────┐
          │             │              │
          ▼             ▼              ▼
   Entity Query     Hydration       UnitOfWork
          │             │              │
          ▼             ▼              ▼
 Relationship     Relationship     Relationship
 Resolution       Assembly         Change Tracking
          │             │              │
          ▼             ▼              ▼
   Loading Plan    Entity Graph    Persistence Plan
          │                            │
          ▼                            ▼
 Query Engine                 Persistence Engine
```

---

# 5. Relationship System ≠ Query Engine

El Relationship System describe y coordina asociaciones.

El Query Engine decide cómo obtener datos.

Por tanto:

```text
Relationship Metadata
→ Query Semantics
→ Query Engine
```

pero nunca:

```text
Relationship System
→ manual SQL generation
```

---

# 6. Relationship System ≠ Schema System

Una relación puede corresponder a:

```text
foreign key
join table
discriminator
shared primary key
logical convention
external mapping
```

El Schema System describe estructura física.

El ORM Relationship System describe semántica de objetos.

---

# 7. Ejemplo

```php
final class Order
{
    private User $user;
}
```

puede mapearse físicamente como:

```text
orders.user_id
→ users.id
```

La relación es:

```text
Order.user
```

La FK es:

```text
orders.user_id → users.id
```

Son conceptos relacionados, no idénticos.

---

# 8. Relationship Kinds

VoltStack deberá representar tipos de relación explícitos.

```php
enum RelationshipKind
{
    case ONE_TO_ONE;
    case ONE_TO_MANY;
    case MANY_TO_ONE;
    case MANY_TO_MANY;
    case POLYMORPHIC_TO_ONE;
    case POLYMORPHIC_TO_MANY;
}
```

---

# 9. Directionality

Toda relación será:

```text
UNIDIRECTIONAL
```

o:

```text
BIDIRECTIONAL
```

---

# 10. Unidirectional

Ejemplo:

```text
Order
 └── User
```

sin:

```text
User.orders
```

---

# 11. Bidirectional

```text
User.orders
↔
Order.user
```

Ambos lados representan la misma asociación conceptual desde perspectivas distintas.

---

# 12. Owning Side

El owning side es el lado responsable de expresar el cambio persistente de la relación.

No significa:

```text
business owner
```

ni:

```text
aggregate root
```

ni necesariamente:

```text
parent object
```

---

# 13. Inverse Side

El inverse side refleja una relación cuyo estado persistente se determina desde el owning side.

---

# 14. Example

Relación:

```text
User 1 ─── N Order
```

Mapping típico:

```text
Order.user
=
MANY_TO_ONE
OWNING
```

```text
User.orders
=
ONE_TO_MANY
INVERSE
```

porque la FK está físicamente en:

```text
orders.user_id
```

---

# 15. Ownership formula

```text
PersistentRelationshipSource
=
OwningSide
```

No necesariamente:

```text
CollectionSide
```

---

# 16. Relationship Metadata

Contrato conceptual:

```php
final readonly class RelationshipMetadata
{
    public function __construct(
        public RelationshipId $id,
        public EntityType $ownerType,
        public string $property,
        public RelationshipKind $kind,
        public EntityType $targetType,
        public RelationshipDirection $direction,
        public RelationshipOwnership $ownership,
        public RelationshipFetchMode $fetch,
        public CascadePolicy $cascade,
        public OrphanRemovalPolicy $orphanRemoval,
    ) {}
}
```

---

# 17. RelationshipId

Cada relación debe tener una identidad metadata estable.

Ejemplo:

```text
User.orders
```

puede normalizarse internamente a:

```text
relationship_42
```

---

# 18. Relationship Metadata ≠ Runtime State

Metadata será:

```text
immutable
compiled
shareable
```

No almacenará:

```text
current collection
current owner
loaded state
dirty state
tenant state
```

---

# 19. Runtime Relationship State

El estado runtime estará asociado al:

```text
PersistenceContext
```

o a la colección/proxy concreta.

---

# 20. Relationship Cardinality

El tipo define expectativas conceptuales:

```text
ONE_TO_ONE
→ 0..1 / 1

MANY_TO_ONE
→ 0..1 / 1

ONE_TO_MANY
→ 0..N

MANY_TO_MANY
→ 0..N
```

según nullability y mapping.

---

# 21. Cardinality ≠ Database Constraint

Que ORM diga:

```text
Order.user required
```

no demuestra que la DB tenga:

```text
NOT NULL FK
```

La validación de schema puede comparar ambas cosas, pero no son equivalentes.

---

# 22. To-One

Una relación to-one representa:

```text
Entity
→
zero or one related entity
```

Tipos:

```text
ONE_TO_ONE
MANY_TO_ONE
POLYMORPHIC_TO_ONE
```

---

# 23. To-Many

Representa:

```text
Entity
→
collection of related entities
```

Tipos:

```text
ONE_TO_MANY
MANY_TO_MANY
POLYMORPHIC_TO_MANY
```

---

# 24. Relationship Collection

Las relaciones to-many deberán utilizar una abstracción con semántica ORM.

No una colección genérica desnuda.

---

# 25. Proposed abstraction

```php
interface PersistentCollection extends iterable, Countable
{
    public function isInitialized(): bool;

    public function isComplete(): bool;

    public function isDirty(): bool;
}
```

---

# 26. PersistentCollection ≠ ResultCollection

Una `ResultCollection` representa resultados de query.

Una `PersistentCollection` representa membresía de una asociación ORM.

---

# 27. Persistent Collection responsibilities

Puede conocer:

```text
owner EntityKey
RelationshipId
initialization state
completeness state
baseline
membership changes
loader reference
```

sin contener SQL.

---

# 28. Collection states

Propuesta:

```php
enum RelationshipCollectionState
{
    case UNINITIALIZED;
    case INITIALIZING;
    case INITIALIZED_PARTIAL;
    case INITIALIZED_COMPLETE;
}
```

---

# 29. Initialization ≠ Completeness

Una colección puede estar inicializada con un subconjunto.

Ejemplo:

```text
User.orders
```

cargada mediante:

```text
status = OPEN
```

puede ser:

```text
INITIALIZED_PARTIAL
```

---

# 30. Loaded ≠ Complete

Regla crítica:

```text
ObservedSubset
≠
CompleteRelationshipState
```

---

# 31. Why

Confundirlos podría provocar:

```text
missing related entities
→ interpreted as removed
→ accidental DELETE / FK update
```

---

# 32. Relationship Value States

Para to-one:

```text
UNINITIALIZED
NULL
ENTITY
REFERENCE
```

deben distinguirse cuando corresponda.

---

# 33. NULL ≠ UNINITIALIZED

```text
NULL
```

significa:

> Se conoce que no existe entidad relacionada.

```text
UNINITIALIZED
```

significa:

> Aún no se ha cargado.

---

# 34. Reference ≠ Loaded Entity

Una `EntityReference` puede contener:

```text
EntityKey
```

sin poseer todos los datos de la entidad.

---

# 35. Relationship Loading

La carga será una responsabilidad separada.

```text
Relationship access
      │
      ▼
Relationship Loader
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
```

---

# 36. Loading ≠ Hydration

`RelationshipLoader` decide:

```text
qué consultar
```

Hydration decide:

```text
cómo materializar el resultado
```

---

# 37. Loading Strategies

La arquitectura deberá soportar:

```text
EAGER
LAZY
BATCH
EXPLICIT
```

---

# 38. EAGER

La relación se carga como parte de la operación principal o inmediatamente asociada.

---

# 39. Eager ≠ JOIN

Puede implementarse mediante:

```text
single JOIN query
```

o:

```text
root query
+
secondary batch query
```

---

# 40. Example

```text
100 Users
+
Orders
```

podría ser mejor:

```text
Query 1: Users
Query 2: Orders WHERE user_id IN (...)
```

que un gran JOIN.

---

# 41. LAZY

La carga se dispara al acceder a una relación no inicializada.

---

# 42. Lazy ≠ Proxy necesariamente

To-one puede usar:

```text
proxy
ghost object
reference wrapper
```

To-many puede usar:

```text
PersistentCollection
```

con loader.

---

# 43. EXPLICIT

El desarrollador solicita carga:

```text
load(User.orders)
```

sin acceso mágico.

---

# 44. BATCH

Varias relaciones pendientes se cargan juntas.

Ejemplo:

```text
User#1.orders
User#2.orders
User#3.orders
```

pueden resolverse mediante:

```text
WHERE user_id IN (1,2,3)
```

---

# 45. Relationship Loader contract

```php
interface RelationshipLoader
{
    public function load(
        object $owner,
        RelationshipMetadata $relationship,
        RelationshipLoadContext $context,
    ): RelationshipLoadResult;
}
```

---

# 46. Loader does not write SQL

Debe usar:

```text
Entity Query
```

o Query Model.

---

# 47. RelationshipLoadContext

Puede incluir:

```text
PersistenceContext
load strategy
deadline
cancellation
read intent
batch context
```

---

# 48. No hidden global context

Nunca:

```text
RelationshipLoader::$currentEntityManager
```

---

# 49. IdentityMap Integration

Toda entidad relacionada cargada debe respetar:

```text
EntityKey
→ canonical object
```

---

# 50. Example

Si:

```text
User#42
```

ya está en IdentityMap y se carga:

```text
Order#10.user
```

deberá apuntar al mismo objeto `User#42`.

---

# 51. Relationship graph canonicality

```text
SameEntityKey
+
SamePersistenceContext
⇒
SameObject
```

sin importar qué relación la cargó.

---

# 52. Hydration Integration

Si el Result ya contiene relaciones por JOIN:

```text
HydrationGraphAssembler
```

puede ensamblarlas.

No se ejecuta un RelationshipLoader adicional.

---

# 53. No redundant query

Si una relación ya está:

```text
INITIALIZED_COMPLETE
```

accederla no debe generar query.

---

# 54. Partial collection access

Si está:

```text
INITIALIZED_PARTIAL
```

la política debe decidir qué significa un acceso normal.

---

# 55. Recommended policy

Una colección parcialmente cargada no debería fingirse completa.

Podrá:

```text
exposePartial()
```

o requerir:

```text
loadComplete()
```

según API.

---

# 56. Relationship Snapshot

El ORM deberá conservar baseline para relaciones persistibles.

---

# 57. To-one baseline

Ejemplo:

```text
Order.user
baseline = User#10
current  = User#11
```

Change:

```text
User#10 → User#11
```

---

# 58. To-many baseline

```text
baseline:
{10, 11, 12}

current:
{10, 12, 13}
```

ChangeSet:

```text
added:
13

removed:
11
```

---

# 59. Snapshot identity-based

No se compararán objetos completos.

Preferir:

```text
EntityKey
```

---

# 60. RelationshipSnapshot ≠ EntitySnapshot

Aunque puedan coordinarse, son conceptos diferentes.

---

# 61. Relationship Change Tracking

Debe detectar:

```text
to-one replacement
to-one nullification
collection add
collection remove
collection clear
ownership transfer
```

---

# 62. Change ≠ SQL

Ejemplo:

```text
User.orders remove Order#10
```

todavía no significa:

```sql
DELETE FROM orders ...
```

Puede significar:

```text
FK → NULL
delete orphan
delete join-table row
no persistence if inverse-only change
```

dependiendo del mapping.

---

# 63. Relationship Persistence

El Persistence Planner traduce cambios semánticos a operaciones.

---

# 64. Pipeline

```text
Relationship Change
      │
      ▼
UnitOfWork
      │
      ▼
Relationship Persistence Analysis
      │
      ▼
Persistence Planner
      │
      ▼
Persistence Operations
      │
      ▼
Query Engine
```

---

# 65. Relationship Persistence ≠ Entity Persistence

Aunque se coordinan.

Ejemplo many-to-many:

```text
Post#1.tags += Tag#5
```

puede requerir:

```text
INSERT join table
```

sin UPDATE de `Post` ni `Tag`.

---

# 66. Owning side persistence

Solo cambios relevantes en owning side determinan normalmente la operación persistente.

---

# 67. Bidirectional consistency

Si el usuario ejecuta:

```php
$user->orders->add($order);
```

pero no:

```php
$order->setUser($user);
```

el graph puede quedar inconsistente.

---

# 68. Domain synchronization

VoltStack deberá definir una política.

Opciones conceptuales:

```text
USER_MANAGED
ORM_FIXUP
STRICT
```

---

# 69. Recommended default

No ocultar por completo errores de dominio.

VoltStack puede proporcionar helpers de sincronización, pero el owning side sigue siendo autoritativo para persistence.

---

# 70. Example helper

```php
$user->addOrder($order);
```

puede internamente:

```php
$this->orders->add($order);
$order->setUser($this);
```

Eso pertenece al modelo de dominio, no al Hydrator.

---

# 71. Hydration fix-up

Durante loading sí puede ser necesario sincronizar ambos lados internamente.

---

# 72. Critical distinction

```text
Hydration Fix-Up
≠
Application Relationship Mutation
```

---

# 73. Hydration fix-up tracking

No deberá marcar automáticamente la relación dirty.

---

# 74. Application mutation tracking

Sí deberá ser detectable por UnitOfWork.

---

# 75. Cascade System

Relaciones podrán definir:

```text
PERSIST
REMOVE
DETACH
REFRESH
```

u otras semánticas permitidas.

---

# 76. Cascade ≠ database cascade

Ejemplo:

```text
cascade persist
```

significa:

```text
ORM registration propagation
```

No:

```text
ON INSERT CASCADE
```

---

# 77. Cascade Persist

Si:

```text
Order
→ new Customer
```

y mapping lo permite:

```text
persist(Order)
```

puede registrar `Customer`.

---

# 78. Cascade Remove

Puede programar eliminación de entidades relacionadas.

No es igual a:

```text
ON DELETE CASCADE
```

---

# 79. Cascade Refresh

Puede propagar refresh según policy.

---

# 80. Cascade Detach

Puede propagar detach.

Debe tener cuidado con graph size.

---

# 81. Cascade policy

```php
final readonly class CascadePolicy
{
    public function __construct(
        public bool $persist,
        public bool $remove,
        public bool $refresh,
        public bool $detach,
    ) {}
}
```

---

# 82. Cascade graph traversal

Debe prevenir:

```text
A → B → A → B ...
```

mediante visited identities/object identities.

---

# 83. Cascade traversal ≠ Loading

No deberá cargar automáticamente relaciones uninitialized solo para recorrer cascades salvo policy explícita.

---

# 84. Critical rule

```text
Cascade
⇏
ImplicitDatabaseRead
```

por defecto.

---

# 85. Orphan Removal

Orphan removal significa que una entidad dependiente removida de una asociación específica debe eliminarse según mapping.

---

# 86. Example

```text
User
 └── Address
```

Si:

```php
$user->setAddress(null);
```

puede programarse:

```text
DELETE Address
```

si `orphanRemoval = true`.

---

# 87. Orphan Removal ≠ Cascade Remove

`Cascade Remove`:

```text
remove parent
→ remove child
```

`Orphan Removal`:

```text
remove child from owning relationship
→ remove child
```

aunque parent siga existiendo.

---

# 88. Orphan detection

Debe basarse en:

```text
relationship baseline
vs
current relationship state
```

---

# 89. No orphan inference from partial collection

Regla crítica:

```text
PARTIAL collection
```

no permite concluir que elementos no observados hayan sido removidos.

---

# 90. Therefore

```text
PartialCollection
⇏
SafeOrphanDetection
```

---

# 91. Orphan removal safety

Requiere:

```text
complete baseline
```

o evidencia equivalente.

---

# 92. Many-to-many

Normalmente removing membership significa:

```text
delete join-table relation
```

no:

```text
delete target entity
```

---

# 93. Join Entity

VoltStack deberá permitir modelar una join table como entidad cuando tiene significado propio.

Ejemplo:

```text
Membership
├── User
├── Organization
├── role
├── joinedAt
```

Eso puede ser mejor que una many-to-many simple.

---

# 94. Many-to-many ≠ Join Entity

Dos modelos diferentes.

---

# 95. Polymorphic Relationships

Se soportarán como extensión de relationship metadata.

Ejemplo:

```text
Comment
→ Post | Video | Image
```

---

# 96. Polymorphic relation

Puede necesitar:

```text
type discriminator
+
identifier
```

---

# 97. Security

El discriminator deberá usar whitelist compilada.

Nunca:

```text
database class string
→ arbitrary PHP class
```

---

# 98. Referential integrity

Una polymorphic relation puede no estar respaldada por una FK tradicional.

Eso refuerza:

```text
ORM Relationship
≠
ForeignKey
```

---

# 99. Proxy Model

To-one lazy relationships podrán utilizar proxy/ghost.

---

# 100. Proxy invariant

Proxy debe compartir:

```text
canonical EntityKey
```

con la entidad concreta.

---

# 101. Proxy replacement forbidden

No:

```text
Proxy A
↓ load
Entity B
```

si `A` ya representa la identidad canónica.

Debe:

```text
initialize A
```

cuando la implementación lo permita.

---

# 102. Lazy Collection

To-many utiliza preferentemente:

```text
PersistentCollection
```

en estado:

```text
UNINITIALIZED
```

---

# 103. Lazy collection access

```text
count()
iteration
contains()
first()
```

pueden tener estrategias diferentes.

---

# 104. Extra-lazy operations

Una evolución futura puede permitir:

```text
count()
containsKey()
contains()
```

sin cargar colección completa.

---

# 105. Extra-lazy ≠ Initialized

Si se ejecuta:

```text
COUNT(*) WHERE owner_id = ?
```

la colección puede seguir:

```text
UNINITIALIZED
```

---

# 106. Count knowledge

Saber:

```text
count = 50
```

no significa conocer los 50 miembros.

---

# 107. Membership knowledge

Igualmente:

```text
contains(Order#10) = true
```

no implica inicialización completa.

---

# 108. Relationship Load Plan

Podrá existir:

```php
final readonly class RelationshipLoadPlan
{
    public function __construct(
        public RelationshipId $relationship,
        public RelationshipLoadStrategy $strategy,
        public RelationshipLoadShape $shape,
    ) {}
}
```

---

# 109. Load Strategy

```php
enum RelationshipLoadStrategy
{
    case JOIN;
    case SELECT;
    case BATCH;
    case LAZY;
    case EXPLICIT;
}
```

---

# 110. Strategy selection

Puede depender de:

```text
cardinality
expected size
pagination
streaming
query shape
runtime policy
N+1 risk
```

---

# 111. Relationship metadata default fetch

Mapping podrá declarar preferencia:

```text
LAZY
EAGER
```

pero planner podrá necesitar una implementación física concreta.

---

# 112. Fetch Mode ≠ Physical Strategy

```text
EAGER
```

es semántica de disponibilidad.

```text
JOIN
```

es estrategia física.

---

# 113. N+1

Problema:

```text
load 100 Users
for each User:
    load Orders
```

produce:

```text
1 + 100 queries
```

---

# 114. N+1 Detection

Será abordado en documento específico.

La arquitectura deberá emitir información suficiente:

```text
RelationshipId
load origin
query fingerprint
owner count
batch opportunity
```

---

# 115. Automatic batching

VoltStack podrá resolver múltiples lazy loads en batch cuando el runtime/lifecycle lo permita.

---

# 116. DataLoader-like model

Conceptualmente:

```text
Pending:
User#1.orders
User#2.orders
User#3.orders

↓ batch

SELECT ...
WHERE user_id IN (...)
```

---

# 117. Batch loading and IdentityMap

Los resultados deben reconciliarse con IdentityMap.

---

# 118. Batch loading and collection completeness

Cada owner debe recibir:

```text
COMPLETE
```

solo si la batch query realmente recuperó toda la relación según mapping.

---

# 119. Ordering

To-many relationships pueden tener:

```text
default order
```

---

# 120. Relationship ordering metadata

Ejemplo:

```text
User.orders
ORDER BY created_at DESC
```

---

# 121. Ordering ≠ collection identity

Ordenar no cambia membresía.

---

# 122. Ordered relationship baseline

Para listas donde el orden es persistente, el snapshot deberá incluir posición.

---

# 123. Set vs List

VoltStack deberá distinguir semánticas:

```text
SET
LIST
BAG
MAP
```

si las soporta.

---

# 124. SET

Misma entidad no aparece más de una vez.

---

# 125. LIST

Orden significativo.

Puede requerir columna:

```text
position
```

---

# 126. BAG

Puede permitir duplicados conceptuales.

---

# 127. MAP

Puede indexar por:

```text
key property
```

---

# 128. V1 recommendation

Soporte robusto inicial:

```text
SET
LIST
```

y extender BAG/MAP posteriormente si se desea.

---

# 129. Relationship Collection Key

Para deduplicación:

```text
OwnerEntityKey
×
RelationshipId
×
RelatedEntityKey
```

---

# 130. Different relationships

La misma entidad puede aparecer en dos relaciones distintas.

Ejemplo:

```text
User.primaryOrganization
User.organizations
```

No confundir edges.

---

# 131. Relationship Edge

Concepto:

```php
final readonly class RelationshipEdge
{
    public function __construct(
        public EntityKey $owner,
        public RelationshipId $relationship,
        public EntityKey $related,
    ) {}
}
```

---

# 132. Relationship Graph

```text
Graph
=
Entities
+
RelationshipEdges
```

---

# 133. Relationship Graph ≠ Domain Aggregate

Un graph cargado puede cruzar múltiples aggregates.

---

# 134. Relationship Graph ≠ Persistence Graph

Persistence graph incluye dependencias operativas:

```text
generated IDs
FK ordering
delete ordering
```

No es idéntico al object relationship graph.

---

# 135. Relationship Graph ≠ Hydration Graph

Hydration graph representa lo materializado por una query concreta.

El relationship graph del PersistenceContext puede ser mayor.

---

# 136. Relationship State Registry

Puede existir un registry externo:

```php
interface RelationshipStateRegistry
{
    public function stateOf(
        object $owner,
        RelationshipId $relationship,
    ): RelationshipRuntimeState;
}
```

---

# 137. Runtime State

Puede incluir:

```text
initialized
completeness
dirty
baseline
loader state
pending additions
pending removals
```

---

# 138. External state preference

Siempre que sea posible:

```text
ORM state
```

debe permanecer fuera de entidades de dominio.

---

# 139. Active Record compatibility

Los modelos tipo Laravel podrán exponer APIs convenientes:

```php
$user->orders;
$user->load('orders');
$user->relationLoaded('orders');
```

pero deberán delegar al mismo Relationship System.

---

# 140. No second relationship engine

```text
Model API
```

y:

```text
EntityManager / Repository API
```

usan la misma metadata, loaders y persistence engine.

---

# 141. Example Model API

```php
$user->orders();
```

puede devolver un:

```text
EntityRelationshipQuery
```

no SQL directo.

---

# 142. Relation query

Ejemplo:

```php
$user->orders()
    ->where('status', OrderStatus::OPEN)
    ->get();
```

Eso no significa que:

```text
$user->orders
```

quede necesariamente inicializada completa.

---

# 143. Critical distinction

```text
Querying through a relationship
≠
Initializing relationship property
```

---

# 144. Example

```php
$openOrders = $user->orders()
    ->where('status', 'open')
    ->get();
```

No deberá marcar automáticamente:

```text
$user->orders = COMPLETE
```

---

# 145. Explicit eager filtering

Si el usuario solicita:

```text
load relationship with filter
```

la colección deberá conservar:

```text
PARTIAL
```

salvo semantics especiales.

---

# 146. Relationship query context

Debe incluir automáticamente el owner constraint:

```text
Order.user = User#42
```

de manera estructurada.

---

# 147. No raw FK assumptions

No:

```text
where user_id = $user->id
```

hardcoded en la API ORM.

Metadata resuelve mapping.

---

# 148. Composite keys

Relationship resolver debe soportar:

```text
composite FK
```

y:

```text
composite entity ID
```

---

# 149. Strongly typed IDs

Debe convertir correctamente:

```text
UserId
OrderId
```

sin perder tipos.

---

# 150. Tenant isolation

Relationship loading deberá conservar:

```text
EntityIdentityNamespace
```

del PersistenceContext.

---

# 151. Cross-tenant relation

Por defecto deberá rechazarse si contextos son incompatibles.

---

# 152. Formula

```text
ValidManagedRelationship(a,b)
⇒
CompatibleIdentityNamespaces(a,b)
```

salvo una relación explícitamente diseñada para cruzar contextos.

---

# 153. Sharding

De igual forma:

```text
cross-shard relationships
```

no deberán asumirse triviales.

---

# 154. Physical cross-database relations

Pueden existir semánticamente aunque DB no soporte FK.

Loading/persistence deberá usar routing apropiado.

---

# 155. Relationship Persistence Context

Toda modificación managed deberá pertenecer a un contexto coherente.

---

# 156. Detached related entity

Asignar una entidad detached a una relación managed requiere policy.

---

# 157. Possible policies

```text
REJECT
MERGE/ATTACH explicit
REFERENCE_BY_ID
CASCADE_PERSIST if truly NEW
```

---

# 158. Default recommendation

Rechazar ambigüedad.

No asumir:

```text
detached = new
```

---

# 159. New related entity

Si se asigna entidad NEW y cascade persist está habilitado:

```text
register NEW
```

---

# 160. Without cascade persist

Debe fallar antes de flush o exigir `persist()` explícito.

---

# 161. Deleted related entity

Una relación no deberá apuntar establemente a una entidad scheduled REMOVED salvo transición válida de eliminación.

---

# 162. Flush stabilization

Lifecycle callbacks pueden modificar relaciones durante `pre*`.

Entonces:

```text
relationship changes
```

deben recomputarse antes de congelar `PersistencePlan`.

---

# 163. Relationship persistence ordering

Ejemplo:

```text
insert parent
↓ generated ID
insert child FK
```

requiere dependency.

---

# 164. Many-to-many ordering

```text
insert A
insert B
↓
insert join edge
```

---

# 165. Delete ordering

```text
delete join edges
↓
delete entity
```

según constraints/platform.

---

# 166. Relationship Planner integration

El Relationship System produce semantic operations.

Persistence Planner decide orden físico.

---

# 167. Semantic operations

Ejemplos:

```text
AssociateToOne
DissociateToOne
AddCollectionMember
RemoveCollectionMember
ReplaceRelationship
ClearRelationship
CreateJoinEdge
DeleteJoinEdge
ScheduleOrphanRemoval
```

---

# 168. Semantic operations ≠ SQL

Nunca:

```text
AddCollectionMember
=
INSERT INTO ...
```

universalmente.

---

# 169. Change normalization

Varias mutaciones pueden cancelarse.

Ejemplo:

```text
add Order#10
remove Order#10
```

antes de flush:

```text
net change = none
```

si baseline no lo contenía.

---

# 170. Conversely

Si baseline sí contenía `Order#10`:

```text
remove
add
```

puede producir:

```text
net change = none
```

salvo que ordering/edge metadata cambie.

---

# 171. Collection mutation ledger

Podrá utilizarse para optimización, pero baseline comparison sigue siendo autoridad segura.

---

# 172. Relationship Constraints

ORM puede declarar:

```text
nullable
unique
required
orphan removal
cascade
```

pero schema enforcement es separado.

---

# 173. Schema validation integration

Puede comparar:

```text
ORM expectation
vs
Schema metadata
```

y producir diagnostics.

---

# 174. Example mismatch

ORM:

```text
Order.user required
```

Schema:

```text
user_id nullable
```

No impide necesariamente runtime, pero debe reportarse.

---

# 175. Relationship Consistency

Debe integrarse con `DATABASE_PERSISTENCE_CONSISTENCY_SYSTEM`.

---

# 176. Consistency domains

Pueden incluir:

```text
relationship object graph
relationship baseline
relationship loaded state
relationship persistence state
```

---

# 177. Dirty ≠ inconsistent

Si:

```text
baseline User#10
current User#11
```

la relación está dirty, pero internamente consistente.

---

# 178. Inconsistent graph

Ejemplo:

```text
User.orders contains Order#1
```

pero:

```text
Order.user = User#2
```

en una relación bidireccional que exige simetría.

---

# 179. Consistency policy

VoltStack podrá detectar:

```text
bidirectional mismatch
```

durante:

```text
change tracking
flush preparation
debug validation
```

---

# 180. No hidden DB verification

Comprobar consistencia del object graph no debe ejecutar queries ocultas.

---

# 181. Unknown relationship state

Una colección no inicializada deberá permanecer:

```text
UNKNOWN / UNINITIALIZED
```

respecto a sus miembros.

---

# 182. Never infer empty

```text
UNINITIALIZED
⇏
EMPTY
```

---

# 183. Serialization

Serializar una entidad no deberá necesariamente disparar lazy loading.

---

# 184. Recommended policy

Serialization debe poder distinguir:

```text
loaded relationship
unloaded relationship
```

y evitar N+1 accidental.

---

# 185. HTTP integration

Pertenece a otras capas.

Relationship System solo expone estado suficiente.

---

# 186. Relationship Events

Podrán existir eventos internos:

```text
relationship.loading
relationship.loaded
relationship.changed
relationship.initialized
relationship.batch_loaded
```

pero no deben confundirse con domain events.

---

# 187. Lifecycle callbacks

ORM lifecycle puede observar relaciones, pero deberá respetar su estado de inicialización.

---

# 188. Dangerous callback

Un `postLoad` que itera una lazy collection puede generar query adicional.

Telemetry deberá poder detectarlo.

---

# 189. No automatic prevention universally

Pero puede haber:

```text
strict no-lazy-load mode
```

para testing/production policies.

---

# 190. Lazy Loading Guard

Propuesta:

```php
interface LazyLoadingGuard
{
    public function assertAllowed(
        RelationshipLoadContext $context
    ): void;
}
```

---

# 191. Strict mode

Puede lanzar:

```text
UnexpectedLazyLoadingException
```

---

# 192. Useful environments

```text
tests
API serialization
critical loops
background batch processing
```

---

# 193. N+1 telemetry

Cada lazy load podrá registrar:

```text
relationship
origin call site
owner batch
query fingerprint
```

sin PII.

---

# 194. Persistent Runtime

Relación metadata podrá compartirse.

Runtime state no.

---

# 195. Shared immutable

```text
RelationshipMetadata
CompiledRelationshipLoaderPlan
CompiledRelationshipPersistencePlan
RelationshipAccessorMetadata
```

---

# 196. Scoped mutable

```text
PersistentCollection instances
RelationshipStateRegistry
pending additions/removals
load queues
batch queues
current owner
```

---

# 197. FrankenPHP

```text
Worker
├── Shared Relationship Metadata
│
├── Request A
│   ├── PersistenceContext A
│   ├── Collections A
│   └── RelationshipState A
│
└── Request B
    ├── PersistenceContext B
    ├── Collections B
    └── RelationshipState B
```

---

# 198. No collection reuse between requests

Nunca.

---

# 199. RoadRunner/OpenSwoole

Misma regla de aislamiento.

---

# 200. Concurrent access

Un mismo PersistentCollection mutable no será concurrency-safe por defecto.

---

# 201. Concurrent lazy load

Dos fibers podrían intentar inicializar la misma colección.

Debe existir:

```text
RelationshipLoadGuard
```

o mecanismo equivalente.

---

# 202. Initialization state machine

```text
UNINITIALIZED
    │
    ▼
INITIALIZING
    │
    ├── success → INITIALIZED_*
    │
    └── failure → UNINITIALIZED / FAILED according policy
```

---

# 203. No duplicate load

Mientras está:

```text
INITIALIZING
```

otro acceso deberá coordinarse o rechazarse.

---

# 204. Failure after partial assembly

No deberá marcar colección completa.

---

# 205. Load Outcome

```php
enum RelationshipLoadOutcome
{
    case SUCCEEDED;
    case FAILED;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 206. UNKNOWN

Puede surgir si la consulta ejecutada tiene outcome incierto o el stream falla.

No se convierte a:

```text
COMPLETE
```

---

# 207. Relationship error taxonomy

```text
DatabaseRelationshipException
├── RelationshipMetadataException
├── RelationshipMappingException
├── RelationshipOwnershipException
├── RelationshipStateException
├── RelationshipLoadingException
├── RelationshipLazyLoadingException
├── RelationshipBatchLoadingException
├── RelationshipHydrationException
├── RelationshipPersistenceException
├── RelationshipChangeTrackingException
├── RelationshipCollectionException
├── RelationshipCompletenessException
├── RelationshipCascadeException
├── RelationshipOrphanRemovalException
├── RelationshipDetachedEntityException
├── RelationshipCrossContextException
├── RelationshipPolymorphicException
├── RelationshipDiscriminatorException
├── RelationshipConcurrencyException
├── RelationshipRuntimeIsolationException
└── RelationshipInvariantException
```

---

# 208. Security

El sistema deberá proteger:

- tenant isolation;
- discriminator whitelist;
- class mapping;
- hidden lazy loads;
- cross-context association;
- sensitive telemetry.

---

# 209. Mass assignment

Relacionar una entidad mediante input de usuario pertenece a:

```text
application input security
```

No a hydration.

---

# 210. Authorization

Que una relación exista no implica que el usuario pueda verla o modificarla.

---

# 211. Relationship ≠ Authorization

Regla:

```text
MappedAssociation
⇏
AuthorizedAssociationAccess
```

---

# 212. Telemetry

Métricas propuestas:

```text
orm.relationship.loads
orm.relationship.lazy_loads
orm.relationship.eager_loads
orm.relationship.batch_loads

orm.relationship.initialized
orm.relationship.partial
orm.relationship.complete

orm.relationship.changes
orm.relationship.additions
orm.relationship.removals

orm.relationship.cascade.persist
orm.relationship.cascade.remove
orm.relationship.orphan_removal

orm.relationship.load.duration
orm.relationship.batch.size

orm.relationship.n_plus_one.detected
orm.relationship.lazy_loading.prevented

orm.relationship.failures
```

---

# 213. Load amplification

```text
RelationshipLoadAmplification
=
RelationshipQueries
/
RootQueryCount
```

---

# 214. Batch effectiveness

```text
BatchCompression
=
OwnersResolved
/
BatchQueries
```

---

# 215. Collection amplification

Para eager JOIN:

```text
PhysicalRows
/
UniqueRelationshipEdges
```

puede ser útil.

---

# 216. No high cardinality metric labels

No usar:

```text
owner ID
related ID
tenant ID
```

como labels.

---

# 217. Debug Toolbar

Ejemplo:

```text
Relationships
────────────────────────────────

User.orders
  Mode:          BATCH EAGER
  Owners:        100
  Queries:       1
  Related:       2,450
  Complete:      YES

User.organization
  Mode:          LAZY
  Loads:         24

Post.tags
  Mode:          JOIN
  Root rows:     100
  Physical rows: 850

N+1 warnings:    1
```

---

# 218. Developer Experience

La API deberá ser simple.

Ejemplo:

```php
$user->orders;
```

pero la semántica interna seguirá siendo rigurosa.

---

# 219. Explicit load

```php
$entityManager->load($user, 'orders');
```

o API equivalente.

---

# 220. Batch eager

```php
User::query()
    ->with('orders')
    ->get();
```

no deberá prometer JOIN específicamente.

---

# 221. Nested relations

```php
User::query()
    ->with('orders.items.product')
    ->get();
```

debe construir un load graph.

---

# 222. Load Graph

```text
User
└── orders
    └── items
        └── product
```

---

# 223. LoadGraph ≠ EntityGraph state

Es una solicitud de carga.

---

# 224. Relationship Load Graph Planner

Puede decidir estrategia distinta en cada edge.

---

# 225. Example

```text
User.orders
→ BATCH

Order.items
→ BATCH

Item.product
→ JOIN
```

---

# 226. Why

Evita mega-JOIN cartesiano.

---

# 227. Cost hints

Planner futuro puede considerar:

```text
estimated cardinality
historical telemetry
configured thresholds
pagination
streaming
```

---

# 228. But

Relationship System no deberá volverse un optimizer paralelo.

Usará Query Planner/capability infrastructure.

---

# 229. Directory Structure

```text
src/Quantum/Database/ORM/Relationship/
│
├── Contract/
│   ├── RelationshipLoader.php
│   ├── RelationshipPersister.php
│   ├── RelationshipStateRegistry.php
│   ├── RelationshipChangeTracker.php
│   └── RelationshipCollectionFactory.php
│
├── Metadata/
│   ├── RelationshipMetadata.php
│   ├── RelationshipId.php
│   ├── RelationshipKind.php
│   ├── RelationshipDirection.php
│   ├── RelationshipOwnership.php
│   ├── RelationshipFetchMode.php
│   ├── CascadePolicy.php
│   └── OrphanRemovalPolicy.php
│
├── Collection/
│   ├── PersistentCollection.php
│   ├── PersistentSet.php
│   ├── PersistentList.php
│   ├── RelationshipCollectionState.php
│   ├── RelationshipCollectionCompleteness.php
│   └── RelationshipCollectionFactory.php
│
├── State/
│   ├── RelationshipRuntimeState.php
│   ├── DefaultRelationshipStateRegistry.php
│   └── RelationshipSnapshot.php
│
├── Loading/
│   ├── DefaultRelationshipLoader.php
│   ├── RelationshipLoadPlan.php
│   ├── RelationshipLoadContext.php
│   ├── RelationshipLoadStrategy.php
│   ├── RelationshipLoadOutcome.php
│   ├── RelationshipLoadGuard.php
│   └── LazyLoadingGuard.php
│
├── Batch/
│   ├── RelationshipBatchLoader.php
│   ├── RelationshipBatch.php
│   ├── RelationshipBatchKey.php
│   └── RelationshipBatchQueue.php
│
├── Graph/
│   ├── RelationshipGraph.php
│   ├── RelationshipEdge.php
│   ├── RelationshipLoadGraph.php
│   └── RelationshipGraphValidator.php
│
├── Hydration/
│   ├── RelationshipAssembler.php
│   ├── RelationshipFixup.php
│   └── RelationshipHydrationState.php
│
├── ChangeTracking/
│   ├── RelationshipChangeSet.php
│   ├── ToOneRelationshipChange.php
│   ├── ToManyRelationshipChange.php
│   └── RelationshipChangeNormalizer.php
│
├── Persistence/
│   ├── RelationshipPersistenceOperation.php
│   ├── RelationshipPersistencePlanner.php
│   ├── JoinEdgeOperation.php
│   └── OrphanRemovalOperation.php
│
├── Cascade/
│   ├── CascadeProcessor.php
│   ├── CascadeTraversalContext.php
│   └── CascadeVisitedSet.php
│
├── Polymorphic/
│   ├── PolymorphicRelationshipResolver.php
│   └── PolymorphicDiscriminatorMap.php
│
├── Runtime/
│   ├── RelationshipRuntimeStateResetter.php
│   └── RelationshipConcurrencyGuard.php
│
├── Telemetry/
│   ├── RelationshipTelemetry.php
│   └── RelationshipDiagnostics.php
│
└── Exception/
    └── ...
```

---

# 230. Testing Strategy

Deberá cubrir como mínimo:

```text
one-to-one
one-to-many
many-to-one
many-to-many
polymorphic

unidirectional
bidirectional

owning/inverse side

lazy loading
eager loading
batch loading

IdentityMap reuse

partial collection
complete collection
uninitialized collection

relationship change tracking
cascade persist/remove
orphan removal

cross-context failures
persistent runtime
concurrency
```

---

# 231. One-to-one test

Misma related identity produce instancia canónica.

---

# 232. One-to-many test

Colección no duplica miembros set-like.

---

# 233. Many-to-one test

Cambiar related entity produce to-one ChangeSet.

---

# 234. Many-to-many test

Membership change produce join-edge operation.

---

# 235. Bidirectional test

Fix-up inicial produce graph coherente sin dirty artificial.

---

# 236. Application mutation test

Cambio posterior sí se detecta.

---

# 237. Uninitialized test

No se trata como empty.

---

# 238. Partial test

No se trata como complete.

---

# 239. Filtered eager test

Mantiene completeness parcial.

---

# 240. Orphan test

Solo se detecta con baseline suficiente.

---

# 241. Partial orphan test

No elimina entidad no observada.

---

# 242. Cascade cycle test

No entra en loop infinito.

---

# 243. Cascade uninitialized test

No fuerza hidden loading por defecto.

---

# 244. Lazy load test

Carga una vez.

---

# 245. Concurrent lazy load test

No ejecuta dos cargas incompatibles.

---

# 246. Batch load test

N owners se resuelven con menor cantidad de queries.

---

# 247. N+1 telemetry test

Detecta repeated relationship loads.

---

# 248. Proxy test

To-one lazy mantiene canonical object.

---

# 249. Cross-tenant test

Relación incompatible se rechaza.

---

# 250. Detached entity test

No se asume NEW.

---

# 251. Strong ID test

FK/identity conversion conserva tipos.

---

# 252. Persistent runtime test

Request A no comparte collections/state con B.

---

# 253. Architectural Invariants

## DB-ORM-RELATIONSHIP-001

ORM Relationship será distinto de ForeignKey.

## DB-ORM-RELATIONSHIP-002

ORM Relationship será distinto de JoinTable.

## DB-ORM-RELATIONSHIP-003

Relationship Metadata será distinto de Schema Metadata.

## DB-ORM-RELATIONSHIP-004

Relationship Loading será distinto de Hydration.

## DB-ORM-RELATIONSHIP-005

Relationship Persistence será distinto de Entity Persistence.

## DB-ORM-RELATIONSHIP-006

Relationship Graph será distinto de Persistence Graph.

## DB-ORM-RELATIONSHIP-007

Eager Loading será distinto de JOIN.

## DB-ORM-RELATIONSHIP-008

Lazy Loading será distinto de Proxy necesariamente.

## DB-ORM-RELATIONSHIP-009

Cascade será distinto de database CASCADE.

## DB-ORM-RELATIONSHIP-010

Orphan Removal será distinto de Cascade Remove.

## DB-ORM-RELATIONSHIP-011

Relationship kinds serán explícitos.

## DB-ORM-RELATIONSHIP-012

Relationship direction será explícita.

## DB-ORM-RELATIONSHIP-013

Ownership será explícito.

## DB-ORM-RELATIONSHIP-014

Owning side será autoridad de persistence de la relación.

## DB-ORM-RELATIONSHIP-015

Owning side no significará business owner.

## DB-ORM-RELATIONSHIP-016

Inverse side no será autoridad persistente por sí sola.

## DB-ORM-RELATIONSHIP-017

Relationship metadata será immutable/compiled.

## DB-ORM-RELATIONSHIP-018

Runtime relationship state no vivirá en shared metadata.

## DB-ORM-RELATIONSHIP-019

To-one y to-many tendrán semánticas distintas.

## DB-ORM-RELATIONSHIP-020

PersistentCollection será distinta de ResultCollection.

## DB-ORM-RELATIONSHIP-021

PersistentCollection conocerá initialization state.

## DB-ORM-RELATIONSHIP-022

Initialization será distinta de completeness.

## DB-ORM-RELATIONSHIP-023

Loaded será distinto de complete.

## DB-ORM-RELATIONSHIP-024

UNINITIALIZED será distinto de EMPTY.

## DB-ORM-RELATIONSHIP-025

NULL to-one será distinto de UNINITIALIZED.

## DB-ORM-RELATIONSHIP-026

Reference será distinta de fully loaded entity.

## DB-ORM-RELATIONSHIP-027

IdentityMap canonicality aplicará a related entities.

## DB-ORM-RELATIONSHIP-028

Same EntityKey dentro del mismo context reutilizará object.

## DB-ORM-RELATIONSHIP-029

Hydrated relation no ejecutará loader redundante.

## DB-ORM-RELATIONSHIP-030

Complete collection no se recargará sin intención explícita.

## DB-ORM-RELATIONSHIP-031

Partial collection no fingirá ser complete.

## DB-ORM-RELATIONSHIP-032

Filtered relationship result no implicará completeness.

## DB-ORM-RELATIONSHIP-033

Relationship snapshot utilizará identidad semántica.

## DB-ORM-RELATIONSHIP-034

Relationship snapshot será distinto de EntitySnapshot.

## DB-ORM-RELATIONSHIP-035

Relationship change será distinto de SQL.

## DB-ORM-RELATIONSHIP-036

To-one replacement será ChangeSet semántico.

## DB-ORM-RELATIONSHIP-037

Collection add/remove serán ChangeSets semánticos.

## DB-ORM-RELATIONSHIP-038

Owning-side change gobernará persistence normal.

## DB-ORM-RELATIONSHIP-039

Inverse-only mismatch podrá diagnosticarse.

## DB-ORM-RELATIONSHIP-040

Hydration fix-up será distinto de application mutation.

## DB-ORM-RELATIONSHIP-041

Hydration fix-up no producirá dirty state artificial.

## DB-ORM-RELATIONSHIP-042

Application mutation será detectable.

## DB-ORM-RELATIONSHIP-043

Cascade Persist propagará ORM persistence intent.

## DB-ORM-RELATIONSHIP-044

Cascade Persist no será database cascade.

## DB-ORM-RELATIONSHIP-045

Cascade Remove será distinto de ON DELETE CASCADE.

## DB-ORM-RELATIONSHIP-046

Cascade traversal será cycle-safe.

## DB-ORM-RELATIONSHIP-047

Cascade no disparará hidden loads por defecto.

## DB-ORM-RELATIONSHIP-048

Orphan Removal requerirá semántica explícita.

## DB-ORM-RELATIONSHIP-049

Orphan Removal no será Cascade Remove.

## DB-ORM-RELATIONSHIP-050

Partial collection no permitirá orphan inference insegura.

## DB-ORM-RELATIONSHIP-051

Many-to-many membership removal no borrará target automáticamente.

## DB-ORM-RELATIONSHIP-052

Join Entity será distinta de simple many-to-many.

## DB-ORM-RELATIONSHIP-053

Polymorphic relationship utilizará whitelist.

## DB-ORM-RELATIONSHIP-054

Polymorphic discriminator no instanciará clases arbitrarias.

## DB-ORM-RELATIONSHIP-055

Proxy compartirá canonical EntityKey.

## DB-ORM-RELATIONSHIP-056

Proxy initialization no creará segunda managed entity.

## DB-ORM-RELATIONSHIP-057

Lazy collection podrá permanecer uninitialized después de extra-lazy count.

## DB-ORM-RELATIONSHIP-058

Count knowledge no implicará membership knowledge.

## DB-ORM-RELATIONSHIP-059

Fetch mode será distinto de physical load strategy.

## DB-ORM-RELATIONSHIP-060

EAGER podrá implementarse sin JOIN.

## DB-ORM-RELATIONSHIP-061

BATCH loading reutilizará Query Engine.

## DB-ORM-RELATIONSHIP-062

RelationshipLoader no generará SQL manualmente.

## DB-ORM-RELATIONSHIP-063

RelationshipLoader no conocerá PDO.

## DB-ORM-RELATIONSHIP-064

Relationship queries usarán metadata para owner constraints.

## DB-ORM-RELATIONSHIP-065

No se hardcodearán FK names en ORM API.

## DB-ORM-RELATIONSHIP-066

Composite identifiers serán soportables.

## DB-ORM-RELATIONSHIP-067

Strongly typed identifiers se preservarán.

## DB-ORM-RELATIONSHIP-068

Relationship identity namespace procederá del PersistenceContext.

## DB-ORM-RELATIONSHIP-069

Cross-context association incompatible se rechazará.

## DB-ORM-RELATIONSHIP-070

Detached related entity no será asumida NEW.

## DB-ORM-RELATIONSHIP-071

New related entity requerirá cascade persist o persist explícito.

## DB-ORM-RELATIONSHIP-072

Removed entity no permanecerá relacionada sin transición válida.

## DB-ORM-RELATIONSHIP-073

Lifecycle relation mutations serán reconsideradas antes de plan freeze.

## DB-ORM-RELATIONSHIP-074

Persistence Planner gobernará ordering físico.

## DB-ORM-RELATIONSHIP-075

Relationship System no será un segundo Persistence Planner.

## DB-ORM-RELATIONSHIP-076

Semantic relationship operations serán distintas de SQL.

## DB-ORM-RELATIONSHIP-077

Relationship mutation normalization podrá cancelar cambios netos.

## DB-ORM-RELATIONSHIP-078

Ordering changes podrán ser persistentes solo en mappings que lo declaren.

## DB-ORM-RELATIONSHIP-079

Collection semantics serán explícitas.

## DB-ORM-RELATIONSHIP-080

SET no permitirá duplicate membership semántico.

## DB-ORM-RELATIONSHIP-081

LIST podrá conservar ordering persistente.

## DB-ORM-RELATIONSHIP-082

Relationship edge incluirá owner, relationship y related identity.

## DB-ORM-RELATIONSHIP-083

Same related entity en diferentes relationships serán edges distintos.

## DB-ORM-RELATIONSHIP-084

Relationship Graph no será Domain Aggregate necesariamente.

## DB-ORM-RELATIONSHIP-085

Relationship Graph no será Hydration Graph necesariamente.

## DB-ORM-RELATIONSHIP-086

Relationship state podrá mantenerse fuera de domain entity.

## DB-ORM-RELATIONSHIP-087

Active Record API usará el mismo Relationship Engine.

## DB-ORM-RELATIONSHIP-088

Repository/Data Mapper API usará el mismo Relationship Engine.

## DB-ORM-RELATIONSHIP-089

No existirá segundo relationship engine para Model API.

## DB-ORM-RELATIONSHIP-090

Querying a relationship no inicializará automáticamente la relationship property.

## DB-ORM-RELATIONSHIP-091

Filtered relationship query no marcará collection complete.

## DB-ORM-RELATIONSHIP-092

Load Graph será distinto de loaded Entity Graph.

## DB-ORM-RELATIONSHIP-093

Load Graph podrá elegir estrategia por edge.

## DB-ORM-RELATIONSHIP-094

Mega-JOIN no será estrategia eager universal.

## DB-ORM-RELATIONSHIP-095

N+1 será observable.

## DB-ORM-RELATIONSHIP-096

Lazy loads repetitivos podrán agruparse en batch.

## DB-ORM-RELATIONSHIP-097

Batch loading preservará IdentityMap.

## DB-ORM-RELATIONSHIP-098

Batch loading preservará collection completeness.

## DB-ORM-RELATIONSHIP-099

Relationship ordering no cambiará entity identity.

## DB-ORM-RELATIONSHIP-100

ORM constraint será distinta de DB constraint.

## DB-ORM-RELATIONSHIP-101

Schema mismatch podrá diagnosticarse.

## DB-ORM-RELATIONSHIP-102

Dirty relationship no implicará inconsistency.

## DB-ORM-RELATIONSHIP-103

Bidirectional mismatch podrá considerarse inconsistency.

## DB-ORM-RELATIONSHIP-104

Relationship consistency check no ejecutará DB queries ocultas por defecto.

## DB-ORM-RELATIONSHIP-105

Uninitialized relationship no se interpretará como absent.

## DB-ORM-RELATIONSHIP-106

Serialization no deberá forzar lazy loads obligatoriamente.

## DB-ORM-RELATIONSHIP-107

Relationship events serán distintos de Domain Events.

## DB-ORM-RELATIONSHIP-108

Lifecycle callbacks podrán observar lazy states explícitamente.

## DB-ORM-RELATIONSHIP-109

Strict lazy-loading guard podrá existir.

## DB-ORM-RELATIONSHIP-110

Unexpected lazy load podrá fallar en strict mode.

## DB-ORM-RELATIONSHIP-111

Relationship metadata podrá compartirse entre requests.

## DB-ORM-RELATIONSHIP-112

PersistentCollection nunca se compartirá entre requests.

## DB-ORM-RELATIONSHIP-113

RelationshipStateRegistry será scoped.

## DB-ORM-RELATIONSHIP-114

Batch queue será scoped.

## DB-ORM-RELATIONSHIP-115

Current owner nunca será process-global.

## DB-ORM-RELATIONSHIP-116

FrankenPHP preservará immutable relationship metadata.

## DB-ORM-RELATIONSHIP-117

FrankenPHP limpiará runtime relationship state por request.

## DB-ORM-RELATIONSHIP-118

RoadRunner seguirá el mismo aislamiento.

## DB-ORM-RELATIONSHIP-119

OpenSwoole aislará relationship state por contexto lógico.

## DB-ORM-RELATIONSHIP-120

PersistentCollection mutable no será concurrency-safe por defecto.

## DB-ORM-RELATIONSHIP-121

Concurrent initialization deberá coordinarse.

## DB-ORM-RELATIONSHIP-122

INITIALIZING será estado distinto de INITIALIZED.

## DB-ORM-RELATIONSHIP-123

Failed initialization no marcará collection complete.

## DB-ORM-RELATIONSHIP-124

UNKNOWN load outcome no implicará complete state.

## DB-ORM-RELATIONSHIP-125

Relationship loading no hará commit.

## DB-ORM-RELATIONSHIP-126

Relationship loading no hará flush.

## DB-ORM-RELATIONSHIP-127

Relationship loading no hará rollback.

## DB-ORM-RELATIONSHIP-128

Relationship mapping no implicará authorization.

## DB-ORM-RELATIONSHIP-129

Relationship mapping no implicará input validation.

## DB-ORM-RELATIONSHIP-130

Tenant isolation se preservará durante relationship loading.

## DB-ORM-RELATIONSHIP-131

Cross-shard relationship no se asumirá local.

## DB-ORM-RELATIONSHIP-132

Physical DB limitation no eliminará necesariamente semantic relationship.

## DB-ORM-RELATIONSHIP-133

Relationship planner utilizará capability-based infrastructure.

## DB-ORM-RELATIONSHIP-134

Relationship telemetry no usará IDs sensibles como metric labels.

## DB-ORM-RELATIONSHIP-135

N+1 telemetry conservará RelationshipId y origin metadata sin PII.

## DB-ORM-RELATIONSHIP-136

Partial state será observable para debugging.

## DB-ORM-RELATIONSHIP-137

Complete state será observable para debugging.

## DB-ORM-RELATIONSHIP-138

Load strategy será observable.

## DB-ORM-RELATIONSHIP-139

Relationship persistence operations serán diagnosticables.

## DB-ORM-RELATIONSHIP-140

Orphan removal decisions serán diagnosticables.

## DB-ORM-RELATIONSHIP-141

Cascade traversal será diagnosticable.

## DB-ORM-RELATIONSHIP-142

No se borrará related entity por ausencia en partial result.

## DB-ORM-RELATIONSHIP-143

No se actualizará owning side por inverse-side hydration artifact.

## DB-ORM-RELATIONSHIP-144

Relationship snapshot no contendrá unloaded state como empty baseline.

## DB-ORM-RELATIONSHIP-145

Partial baseline no se utilizará como complete baseline.

## DB-ORM-RELATIONSHIP-146

Collection mutation sobre partial relation tendrá policy explícita.

## DB-ORM-RELATIONSHIP-147

Clear() sobre uninitialized collection no se asumirá trivial sin policy.

## DB-ORM-RELATIONSHIP-148

PersistentCollection methods deberán respetar initialization policy.

## DB-ORM-RELATIONSHIP-149

Extra-lazy operations no marcarán collection initialized automáticamente.

## DB-ORM-RELATIONSHIP-150

Relationship loader failure no corromperá IdentityMap.

## DB-ORM-RELATIONSHIP-151

Relationship load failure no inventará empty relation.

## DB-ORM-RELATIONSHIP-152

Relationship change tracking será identity-aware.

## DB-ORM-RELATIONSHIP-153

Relationship change tracking evitará deep object equality.

## DB-ORM-RELATIONSHIP-154

Same managed entity no será added dos veces a SET relation.

## DB-ORM-RELATIONSHIP-155

Join edge identity será distinta de target entity identity.

## DB-ORM-RELATIONSHIP-156

Relationship semantics no dependerán de nombres físicos de columnas.

## DB-ORM-RELATIONSHIP-157

Relationship semantics deberán poder portarse entre plataformas compatibles.

## DB-ORM-RELATIONSHIP-158

El Relationship System será una capa ORM común para loading y persistence, no un conjunto de helpers ad hoc.

## DB-ORM-RELATIONSHIP-159

Toda relación administrada deberá conservar explícitamente identidad, ownership, estado de carga, completeness y baseline suficientes para evitar inferencias destructivas.

## DB-ORM-RELATIONSHIP-160

VoltStack nunca interpretará la ausencia de información sobre una relación como evidencia de ausencia de la relación: `UNINITIALIZED`, `PARTIAL`, `NULL`, `EMPTY` y `COMPLETE` serán estados semánticamente diferentes.

---

# 254. Fórmulas fundamentales

## 254.1 Relationship

```text
Relationship
=
OwnerType
+
RelationshipId
+
TargetType
+
Cardinality
+
Ownership
+
MappingSemantics
```

---

# 255. Runtime relationship state

```text
RelationshipRuntimeState
=
Initialization
+
Completeness
+
Baseline
+
CurrentValue
+
DirtyState
```

---

# 256. To-one change

```text
ToOneChange
=
CurrentRelatedKey
-
BaselineRelatedKey
```

conceptualmente.

---

# 257. To-many change

```text
Added
=
CurrentMembers
-
BaselineMembers
```

```text
Removed
=
BaselineMembers
-
CurrentMembers
```

solo cuando baseline es suficientemente completo.

---

# 258. Safe orphan detection

```text
MayDetectOrphans
=
OrphanRemovalEnabled
∧
BaselineComplete
∧
CurrentRelationshipStateKnown
```

---

# 259. Canonical relationship graph

```text
CanonicalRelationshipGraph
=
CanonicalEntities
+
IdentityBasedEdges
```

---

# 260. Relationship load

```text
RelationshipLoad
=
RelationshipMetadata
+
OwnerIdentity
+
LoadStrategy
+
QueryEngine
+
Hydration
+
IdentityMapReconciliation
```

---

# 261. Safe collection initialization

```text
SafeCompleteInitialization
=
LoadSucceeded
∧
ResultCoverageComplete
∧
AssemblySucceeded
```

---

# 262. Eager loading

```text
EagerLoading
=
RelationshipAvailableAsPartOfDeclaredLoadOperation
```

No:

```text
EagerLoading
=
JOIN
```

---

# 263. Lazy loading

```text
LazyLoading
=
DeferredRelationshipResolution
TriggeredByExplicitOrImplicitAccess
```

---

# 264. Batch loading

```text
BatchLoadingEfficiency
=
OwnersResolved
/
QueriesExecuted
```

---

# 265. Persistence

```text
RelationshipPersistence
=
RelationshipChangeSet
+
OwnershipSemantics
+
CascadeRules
+
OrphanRules
+
PersistencePlanning
```

---

# 266. Safe runtime

```text
SafeRelationshipRuntime
=
ImmutableSharedMetadata
∧
ScopedCollections
∧
ScopedRelationshipState
∧
ScopedBatchQueues
∧
IdentityMapCanonicality
∧
NoCrossRequestGraphState
```

---

# 267. Master Formula

```text
Database Relationship Architecture
=
Relationship Metadata
+
Relationship Kinds
+
Directionality
+
Owning/Inverse Semantics
+
To-One Relations
+
To-Many Relations
+
Persistent Collections
+
Initialization States
+
Completeness States
+
Entity References
+
IdentityMap Integration
+
Relationship Snapshots
+
Relationship Change Tracking
+
Relationship Loading
+
Lazy Loading
+
Eager Loading
+
Batch Loading
+
Load Graphs
+
Hydration Graph Assembly
+
Relationship Fix-Up
+
Relationship Persistence
+
Persistence Planning
+
Cascade Processing
+
Orphan Removal
+
Many-to-Many Join Edges
+
Join Entities
+
Polymorphic Relations
+
Proxy Integration
+
Extra-Lazy Semantics
+
Collection Ordering
+
Cross-Context Protection
+
Tenant/Shard Awareness
+
N+1 Observability
+
Persistent Runtime Isolation
+
Concurrency Governance
+
Telemetry
+
Diagnostics
+
Testing
```

---

# 268. Regla maestra

> **En VoltStack, una relación ORM será una asociación tipada y metadata-driven entre identidades de entidades. Su estado runtime deberá distinguir claramente si está cargada, parcialmente cargada, completamente cargada o aún desconocida; y ninguna operación de persistencia podrá inferir eliminaciones, orphans o ausencia de relaciones a partir de información que el ORM no haya observado realmente.**

---

# 269. Arquitectura resultante

```text
                       Entity Metadata
                              │
                              ▼
                    Relationship Metadata
                              │
              ┌───────────────┼────────────────┐
              │               │                │
              ▼               ▼                ▼
        Query Resolution   Runtime State    UoW Tracking
              │               │                │
              ▼               ▼                ▼
         Load Graph      PersistentCollection RelationshipChangeSet
              │               │                │
              ▼               ▼                ▼
       Loading Strategy  Initialization     Persistence
              │          / Completeness      Planner
              ▼               │                │
          Query Engine        │                ▼
              │               │           Persistence
              ▼               │           Operations
          Execution           │                │
              │               │                ▼
              ▼               │           Query Engine
          Hydration           │
              │               │
              ▼               │
        IdentityMap           │
              │               │
              └───────┬───────┘
                      ▼
                Entity Graph
```

Esta arquitectura deja preparado el bloque para especializar cada relación sin duplicar motores.

---

# 270. Documentos del Bloque 13

```text
142_DATABASE_RELATIONSHIP_ARCHITECTURE.md
143_DATABASE_ONE_TO_ONE_RELATIONSHIP_SYSTEM.md
144_DATABASE_ONE_TO_MANY_RELATIONSHIP_SYSTEM.md
145_DATABASE_MANY_TO_ONE_RELATIONSHIP_SYSTEM.md
146_DATABASE_MANY_TO_MANY_RELATIONSHIP_SYSTEM.md
147_DATABASE_POLYMORPHIC_RELATIONSHIP_SYSTEM.md
148_DATABASE_RELATIONSHIP_METADATA_SYSTEM.md
149_DATABASE_RELATIONSHIP_PERSISTENCE_SYSTEM.md
150_DATABASE_RELATIONSHIP_LOADING_SYSTEM.md
151_DATABASE_EAGER_LOADING_SYSTEM.md
152_DATABASE_LAZY_LOADING_SYSTEM.md
153_DATABASE_BATCH_RELATION_LOADING_SYSTEM.md
154_DATABASE_N_PLUS_ONE_DETECTION_SYSTEM.md
```

---

# 271. Siguiente documento

```text
143_DATABASE_ONE_TO_ONE_RELATIONSHIP_SYSTEM.md
```

El siguiente documento deberá profundizar específicamente en:

```text
OneToOneMetadata
owning vs inverse side
shared primary key mapping
foreign-key mapping
nullable one-to-one
required one-to-one
unique FK semantics
unidirectional one-to-one
bidirectional one-to-one
identity resolution
to-one proxy/reference
lazy loading
eager loading
hydration fix-up
relationship replacement
relationship nullification
relationship snapshots
cascade persist
cascade remove
orphan removal
generated identifiers
flush ordering
cross-context safety
partial hydration
refresh semantics
persistent runtime
telemetry
testing
```

La regla central será:

> **Una relación one-to-one de VoltStack representa como máximo una identidad relacionada por cada lado semántico; la unicidad ORM deberá preservarse independientemente de si físicamente se implementa mediante foreign key única, primary key compartida u otra estrategia compatible.**