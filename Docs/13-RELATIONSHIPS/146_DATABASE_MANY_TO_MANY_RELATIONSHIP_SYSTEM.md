# 146_DATABASE_MANY_TO_MANY_RELATIONSHIP_SYSTEM.md

# VoltStack Quantum Database
## Database Many-to-Many Relationship System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 146 — Database Many-to-Many Relationship System  
**Bloque:** 13 — Relationships  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Many-to-Many Relationship System` define la arquitectura mediante la cual VoltStack representa, carga, hidrata, rastrea y persiste asociaciones donde múltiples entidades de un tipo pueden relacionarse con múltiples entidades de otro tipo.

Ejemplos:

```text
User    N ───── N Role
Student N ───── N Course
Post    N ───── N Tag
Product N ───── N Category
Author  N ───── N Book
```

Relacionalmente, esta cardinalidad normalmente requiere una tabla intermedia:

```text
users
─────
id PK

roles
─────
id PK

user_roles
──────────
user_id FK → users.id
role_id FK → roles.id
```

VoltStack deberá tratar esta asociación como una abstracción ORM de membresía y no simplemente como una tabla SQL oculta detrás de una colección PHP.

El sistema deberá soportar:

- relaciones many-to-many unidireccionales;
- relaciones bidireccionales;
- owning/inverse sides;
- join tables;
- join columns compuestas;
- colecciones persistentes;
- lazy loading;
- eager loading;
- batch loading;
- JOIN hydration;
- collection completeness;
- snapshots;
- membership change tracking;
- `add`;
- `remove`;
- `attach`;
- `detach`;
- `sync`;
- `clear`;
- cascade persist;
- ordering;
- duplicate prevention;
- extra-lazy operations;
- association rows;
- association entities;
- large collections;
- transaction consistency;
- tenant/shard isolation;
- persistent runtimes.

---

# 2. Principio central

> **Una relación many-to-many de VoltStack representa una asociación de membresía entre dos conjuntos de identidades; la fila de unión es la representación persistente de esa membresía, pero no se convierte automáticamente en una entidad de dominio.**

Formalmente:

```text
ManyToMany(A, B)
=
Membership ⊆ Identity(A) × Identity(B)
```

Ejemplo:

```text
(User#10, Role#1)
(User#10, Role#2)
(User#20, Role#1)
```

representa tres membresías.

---

# 3. Regla fundamental

```text
ManyToMany
=
Relationship Metadata
+
Source Identity
+
Target Identity Set
+
Membership Knowledge
+
Collection State
+
Ownership
+
Persistence Semantics
```

No simplemente:

```text
array<object>
```

---

# 4. Many-to-Many ≠ Join Table

La relación:

```text
User.roles
```

es una abstracción ORM.

La tabla:

```text
user_roles
```

es una representación relacional.

Por tanto:

```text
ManyToManyRelationship
≠
JoinTable
```

---

# 5. Many-to-Many ≠ PersistentCollection

`PersistentCollection` mantiene estado runtime de una colección.

La relación many-to-many define:

- cardinalidad;
- ownership;
- target;
- mapping;
- join representation;
- cascade;
- loading;
- persistence semantics.

Por tanto:

```text
Relationship Metadata
≠
Collection Runtime State
```

---

# 6. Many-to-Many ≠ Association Entity

Una tabla:

```text
user_roles
──────────
user_id
role_id
```

puede representar únicamente membresía.

Pero:

```text
user_roles
──────────
id
user_id
role_id
assigned_at
assigned_by
expires_at
status
```

puede representar un concepto de dominio independiente.

En ese caso podría modelarse como:

```text
User
  1
  │
  N
UserRole
  N
  │
  1
Role
```

y no como many-to-many puro.

---

# 7. Regla de modelado

> **Cuando la asociación posee identidad, ciclo de vida, estado o comportamiento propio relevante para el dominio, VoltStack recomendará modelarla como Association Entity en lugar de esconderla como una relación many-to-many pura.**

---

# 8. Pure Join Table

Ejemplo:

```text
post_tags
─────────
post_id
tag_id

PRIMARY KEY(post_id, tag_id)
```

representa:

```text
Post ↔ Tag membership
```

sin identidad ORM propia.

---

# 9. Association Row ≠ Entity

La fila:

```text
post_id = 100
tag_id  = 5
```

no entra automáticamente en:

- IdentityMap;
- EntityManager;
- UnitOfWork como entidad;
- Entity lifecycle callbacks.

La membresía sí será rastreada semánticamente por Relationship System.

---

# 10. Association Entity

Si el desarrollador define:

```php
final class PostTag
{
    private Post $post;
    private Tag $tag;
    private DateTimeImmutable $assignedAt;
}
```

entonces deja de ser un many-to-many puro desde el punto de vista del persistence model.

---

# 11. Arquitectura general

```text
                     ManyToManyMetadata
                            │
            ┌───────────────┼────────────────┐
            │               │                │
            ▼               ▼                ▼
      Collection State   Loading       Persistence
            │               │                │
            ▼               ▼                ▼
   PersistentCollection  Query Engine   Membership Snapshot
            │                                │
            ▼                                ▼
       IdentityMap                     Change Tracking
            │                                │
            └──────────────┬─────────────────┘
                           ▼
                    Membership Changes
                           │
                           ▼
                  Persistence Planner
                           │
                           ▼
                   Join Row Operations
                           │
                           ▼
                      Query Engine
```

---

# 12. ManyToManyMetadata

Propuesta:

```php
final readonly class ManyToManyMetadata
{
    /**
     * @param list<JoinColumnMetadata> $sourceJoinColumns
     * @param list<JoinColumnMetadata> $targetJoinColumns
     */
    public function __construct(
        public RelationshipId $id,
        public EntityType $sourceType,
        public EntityType $targetType,
        public string $property,
        public JoinTableMetadata $joinTable,
        public array $sourceJoinColumns,
        public array $targetJoinColumns,
        public RelationshipOwnership $ownership,
        public RelationshipFetchMode $fetchMode,
        public CascadePolicy $cascade,
        public CollectionSemantics $collectionSemantics,
        public ?string $mappedBy,
        public ?string $inversedBy,
    ) {}
}
```

---

# 13. JoinTableMetadata

```php
final readonly class JoinTableMetadata
{
    public function __construct(
        public TableName $table,
        public ?SchemaName $schema,
        public JoinTableUniqueness $uniqueness,
    ) {}
}
```

---

# 14. Metadata ≠ Schema

Declarar:

```php
#[JoinTable('user_roles')]
```

no demuestra que la tabla exista.

Tampoco demuestra que contenga:

- PK;
- unique constraint;
- FK;
- indexes.

Esto pertenece a Schema System.

---

# 15. Owning Side

En una relación bidireccional deberá existir una autoridad de persistencia.

Ejemplo:

```text
User.roles     OWNING
Role.users     INVERSE
```

---

# 16. Ownership

El owning side determina la membresía que deberá persistirse.

El inverse side proporciona navegación y sincronización del object graph.

---

# 17. Owning Side ≠ Domain Owner

El aggregate root de DDD puede ser otro.

---

# 18. Owning Side ≠ Join Table Owner

La tabla no "pertenece" físicamente al objeto owning.

Ownership es semántica ORM.

---

# 19. Bidirectional Mapping

Ejemplo:

```php
final class User
{
    #[ManyToMany(
        target: Role::class,
        inversedBy: 'users'
    )]
    private Collection $roles;
}

final class Role
{
    #[ManyToMany(
        target: User::class,
        mappedBy: 'roles'
    )]
    private Collection $users;
}
```

---

# 20. Unidirectional Mapping

También:

```text
User.roles
```

sin:

```text
Role.users
```

deberá ser válido.

---

# 21. Collection Semantics

Una relación many-to-many necesita declarar qué significa su colección.

Propuesta:

```php
enum CollectionSemantics
{
    case SET;
    case ORDERED_SET;
    case LIST;
}
```

Para many-to-many puro, el default recomendado es:

```text
SET
```

---

# 22. Razón del SET Default

Una membresía matemática no debería contener:

```text
(User#10, Role#1)
(User#10, Role#1)
```

dos veces.

---

# 23. Duplicate Membership

Por defecto:

```text
Add(User#10.roles, Role#1)
```

cuando ya existe:

```text
(User#10, Role#1)
```

será idempotente.

---

# 24. Formula

```text
MembershipSet
=
Unique(TargetEntityKey)
```

por source entity.

---

# 25. PHP Object Equality ≠ Membership Equality

La comparación deberá usar:

```text
canonical EntityKey
```

no:

```php
$a == $b
```

ni igualdad completa de atributos.

---

# 26. PersistentCollection

La colección runtime podrá representar:

```text
UNINITIALIZED
PARTIALLY_INITIALIZED
INITIALIZED
STALE
UNCERTAIN
```

además de cambios pendientes.

---

# 27. Collection State

Propuesta conceptual:

```php
enum RelationshipCollectionState
{
    case UNINITIALIZED;
    case PARTIALLY_INITIALIZED;
    case INITIALIZED;
    case STALE;
    case UNCERTAIN;
}
```

---

# 28. UNINITIALIZED ≠ Empty

Regla crítica:

```text
UNINITIALIZED
≠
[]
```

---

# 29. PARTIALLY_INITIALIZED ≠ Complete

Una colección cargada mediante:

```text
LIMIT
filter
slice
extra-lazy query
```

no deberá considerarse completa.

---

# 30. Collection Completeness

Propuesta:

```php
enum CollectionCompleteness
{
    case UNKNOWN;
    case PARTIAL;
    case COMPLETE;
}
```

---

# 31. Completeness Invariant

Solo:

```text
COMPLETE
```

permite afirmar:

```text
target not in collection
⇒
membership absent
```

dentro del snapshot observado.

---

# 32. Unknown Membership

Si colección está uninitialized:

```text
Role#5 not currently in memory
```

no significa:

```text
Role#5 is not associated
```

---

# 33. PersistentCollection Responsibilities

Puede encargarse de:

- iteration;
- count semantics;
- initialization state;
- pending additions;
- pending removals;
- known membership;
- snapshot reference;
- loading delegation.

No deberá generar SQL.

---

# 34. Collection ≠ Repository

La colección representa una asociación concreta de una source entity.

Repository representa acceso al conjunto de entidades de un tipo.

---

# 35. Collection ≠ Query Builder

Aunque pueda exponer operaciones como:

```php
$roles->contains($role);
```

no deberá convertirse en un SQL builder.

---

# 36. Membership Identity

Propuesta:

```php
final readonly class MembershipKey
{
    public function __construct(
        public RelationshipId $relationship,
        public EntityKey $source,
        public EntityKey $target,
    ) {}
}
```

---

# 37. MembershipKey

Conceptualmente:

```text
MembershipKey
=
RelationshipId
×
SourceEntityKey
×
TargetEntityKey
```

---

# 38. MembershipKey ≠ Join Row PK

La DB puede usar:

```text
PRIMARY KEY(user_id, role_id)
```

pero `MembershipKey` es identidad semántica ORM.

---

# 39. MembershipKey ≠ EntityKey

La membresía no es automáticamente una entidad.

---

# 40. Lazy Loading

Una colección:

```text
User#10.roles
```

puede comenzar:

```text
UNINITIALIZED
```

---

# 41. Lazy Initialization

Acceder a:

```php
foreach ($user->roles as $role) {
    // ...
}
```

puede provocar:

```text
Relationship Loader
      ↓
Entity Query
      ↓
Query Engine
      ↓
Execution
      ↓
Hydration
      ↓
IdentityMap
      ↓
PersistentCollection
```

---

# 42. Lazy Loading Policy

Podrá existir:

```php
enum LazyRelationshipPolicy
{
    case ALLOW;
    case WARN;
    case FORBID;
}
```

---

# 43. Strict Mode

Con:

```text
FORBID
```

acceder a una colección no cargada podrá producir:

```text
UninitializedRelationshipAccessException
```

---

# 44. Lazy Load ≠ Ownership Change

Inicializar una colección no modifica membresías.

---

# 45. Lazy Load ≠ Dirty

La carga interna deberá establecer baseline sin generar change set.

---

# 46. Eager Loading

Ejemplo:

```php
User::query()
    ->with('roles')
    ->get();
```

---

# 47. Eager Loading Strategy

El planner podrá utilizar:

```text
JOIN_FETCH
SELECT_FETCH
BATCH_FETCH
SUBSELECT_FETCH
```

según capacidades y costo.

---

# 48. Eager Loading ≠ JOIN

No se fijará una estrategia física en el API.

---

# 49. JOIN Row Multiplication

Ejemplo:

```text
User#10 | Role#1
User#10 | Role#2
User#10 | Role#3
```

representa:

```text
User#10.roles = {Role#1, Role#2, Role#3}
```

---

# 50. Multiple Roots

```text
User#10 | Role#1
User#10 | Role#2
User#20 | Role#1
```

debe producir:

```text
User#10.roles = {Role#1, Role#2}
User#20.roles = {Role#1}
```

---

# 51. Target Identity Deduplication

`Role#1` deberá ser la misma instancia canónica para ambos Users dentro del PersistenceContext.

---

# 52. Root Deduplication

La multiplicación física de rows no deberá duplicar roots cuando el `ResultShape` sea una colección de entidades root.

---

# 53. Result Shape Matters

Si la consulta pide tuples:

```text
(User, Role)
```

las filas pueden conservar cardinalidad tuple.

No deberá aplicarse dedup de root indiscriminadamente.

---

# 54. Hydration Plan

La semántica de ensamblaje deberá venir precompilada desde:

```text
HydrationPlan
```

incluyendo:

- root key;
- relationship key;
- target key;
- collection assembler;
- completeness;
- grouping;
- deduplication.

---

# 55. No SQL Parsing During Hydration

Hydrators no deberán analizar:

```text
JOIN user_roles
```

para descubrir la relación.

---

# 56. Collection Assembly

Pipeline:

```text
Result Row
    │
    ├── Source EntityKey
    ├── Target EntityKey
    │
    ▼
IdentityMap
    │
    ▼
Canonical Source + Target
    │
    ▼
MembershipKey
    │
    ▼
HydrationSession Membership Registry
    │
    ▼
PersistentCollection Assembly
```

---

# 57. Duplicate Physical Rows

Una consulta compleja puede devolver:

```text
User#10 | Role#1
User#10 | Role#1
```

varias veces.

La colección SET deberá contener una sola membresía.

---

# 58. Hydration Dedup

```text
same MembershipKey
⇒
one logical membership
```

---

# 59. Hydration Registry

Propuesta:

```php
interface MembershipAssemblyRegistry
{
    public function contains(MembershipKey $key): bool;

    public function register(MembershipKey $key): void;
}
```

---

# 60. Registry Scope

Este registry será:

```text
HydrationSession-scoped
```

no process-global.

---

# 61. Empty Collection via LEFT JOIN

Ejemplo:

```text
User#10 | Role identifier = NULL
```

podrá significar:

```text
User#10.roles = empty
```

si:

- source fue observada;
- relación fue fetch-joined completamente;
- target identity está completamente NULL;
- plan establece collection coverage COMPLETE.

---

# 62. NULL Target ≠ Null Collection

Una relación many-to-many no es:

```text
roles = null
```

sino:

```text
roles = empty set
```

cuando se conoce que no existen memberships.

---

# 63. Collection Presence

La propiedad deberá conceptualmente representar una colección siempre.

---

# 64. Batch Loading

Para múltiples Users:

```text
User#10
User#20
User#30
```

puede cargarse:

```text
user_roles
JOIN roles
WHERE user_id IN (...)
```

conceptualmente.

La generación SQL pertenece al Query Engine.

---

# 65. Batch Loader

Pipeline:

```text
Source EntityKeys
      ↓
Canonicalize
      ↓
Remove already-complete collections
      ↓
Batch Relationship Query
      ↓
Hydration
      ↓
Group memberships by source
      ↓
Mark appropriate collections COMPLETE
```

---

# 66. Batch Completeness

Una colección solo se marcará COMPLETE si la consulta cubrió todas sus memberships según el plan.

---

# 67. Filtered Batch

Si carga únicamente:

```text
active roles
```

entonces:

```text
collection completeness = PARTIAL
```

salvo que la relación misma esté definida semánticamente con ese filtro.

---

# 68. Mapping Filter vs Query Filter

Distinguir:

```text
relationship-level permanent filter
```

de:

```text
ad hoc query filter
```

---

# 69. Relationship Universe

La completitud siempre se evalúa respecto al universo semántico de la relación.

---

# 70. Add Operation

Ejemplo:

```php
$user->roles->add($adminRole);
```

produce intención:

```text
ADD Membership(User#10, Role#1)
```

---

# 71. Add ≠ Immediate INSERT

No:

```text
add()
→ INSERT user_roles
```

inmediatamente.

Debe registrarse en UoW/Relationship Change Tracking.

---

# 72. Remove Operation

```php
$user->roles->remove($adminRole);
```

produce:

```text
REMOVE Membership(User#10, Role#1)
```

---

# 73. Remove ≠ Target Delete

Eliminar una membresía no elimina `Role`.

---

# 74. Clear

```php
$user->roles->clear();
```

significa:

```text
remove all memberships for source
```

no:

```text
delete all Role entities
```

---

# 75. Clear on Uninitialized Collection

Este caso es crítico.

No deberá forzar obligatoriamente:

```text
load all roles
```

antes de borrar las memberships.

---

# 76. Bulk Membership Intent

Podrá registrarse:

```text
CLEAR_ALL_MEMBERSHIPS(source)
```

como intención semántica.

---

# 77. Clear ≠ Bulk Entity Delete

La operación afecta association rows.

No target rows.

---

# 78. attach()

Model API podrá ofrecer:

```php
$user->roles()->attach($role);
```

como fachada sobre:

```text
ADD membership
```

---

# 79. detach()

```php
$user->roles()->detach($role);
```

delega a:

```text
REMOVE membership
```

---

# 80. sync()

Ejemplo:

```php
$user->roles()->sync([$admin, $editor]);
```

expresa:

```text
DesiredMembershipSet
=
{Admin, Editor}
```

---

# 81. sync Semantics

Conceptualmente:

```text
ToAdd
=
Desired - KnownCurrent

ToRemove
=
KnownCurrent - Desired
```

---

# 82. sync Requires Knowledge

Para calcular diferencias exactas necesita conocer el membership set actual o usar una estrategia de persistence capaz de expresar reemplazo semántico sin fabricar conocimiento.

---

# 83. Unsafe sync

No:

```text
UNINITIALIZED
→
assume empty
→
insert desired
```

---

# 84. Sync Planner

Podrá elegir:

```text
LOAD_AND_DIFF
DATABASE_DIFF
DELETE_AND_REINSERT
MERGE_UPSERT
CUSTOM
```

según:

- transaction;
- lifecycle semantics;
- join-table payload;
- DB capabilities;
- expected cardinality;
- concurrency policy.

---

# 85. Delete-and-Reinsert

No será universalmente seguro.

Puede:

- causar unnecessary writes;
- afectar audit;
- producir races;
- romper temporalmente memberships;
- disparar DB effects.

---

# 86. Default Sync Rule

VoltStack deberá preferir una estrategia semánticamente correcta y explicable antes que una optimización destructiva.

---

# 87. Membership Snapshot

Propuesta:

```php
final readonly class ManyToManySnapshot
{
    /**
     * @param set<EntityKey> $knownMembers
     */
    public function __construct(
        public RelationshipId $relationship,
        public EntityKey $source,
        public array $knownMembers,
        public CollectionCompleteness $completeness,
        public RelationshipKnowledge $knowledge,
    ) {}
}
```

---

# 88. Snapshot ≠ Current Collection

Snapshot:

```text
baseline memberships
```

Current:

```text
baseline
+
pending additions
-
pending removals
```

---

# 89. Snapshot Completeness

Una snapshot PARTIAL no permite inferir la ausencia de miembros no observados.

---

# 90. Membership Change Set

Propuesta:

```php
final readonly class MembershipChangeSet
{
    /**
     * @param list<EntityKey> $added
     * @param list<EntityKey> $removed
     */
    public function __construct(
        public RelationshipId $relationship,
        public EntityKey $source,
        public array $added,
        public array $removed,
        public bool $clearAll,
    ) {}
}
```

---

# 91. Added/Removed Sets

Para colección COMPLETE:

```text
Added
=
Current - Baseline

Removed
=
Baseline - Current
```

---

# 92. Pending Mutations Without Initialization

VoltStack deberá permitir:

```text
UNINITIALIZED collection
+
ADD Role#5
```

sin cargar necesariamente toda la colección.

---

# 93. Pending Add

Estado conceptual:

```text
Known Collection: UNKNOWN
Pending Adds: {Role#5}
Pending Removes: {}
```

---

# 94. contains() With Pending Add

Aunque colección no esté inicializada:

```text
contains(Role#5)
```

puede devolver:

```text
TRUE
```

si existe pending add.

---

# 95. contains() Unknown Case

Para Role#8 sin información:

```text
contains(Role#8)
```

no siempre puede responder correctamente sin DB.

---

# 96. Tri-State Membership

Internamente puede requerirse:

```php
enum MembershipKnowledge
{
    case PRESENT;
    case ABSENT;
    case UNKNOWN;
}
```

---

# 97. Public contains()

El API público puede:

- resolver lazy;
- usar extra-lazy query;
- lanzar en strict mode.

Pero internamente deberá conservar UNKNOWN.

---

# 98. Pending Remove

Una remoción conocida:

```text
Pending Removes: {Role#1}
```

permite responder ABSENT desde la perspectiva del estado deseado actual aunque baseline DB todavía contenga la membresía.

---

# 99. Current Semantic State ≠ Database State

Durante UoW:

```text
ApplicationCurrentMembership
≠
DatabaseMembership
```

puede ser perfectamente válido.

---

# 100. Cascade Persist

Si:

```php
$user->roles->add(new Role());
```

y mapping tiene cascade persist:

```text
Role → NEW
```

se registra en UoW.

---

# 101. Without Cascade

Un target transient no registrado deberá provocar error antes de ejecutar association row insert.

---

# 102. Generated Target Identifier

Para:

```text
existing User
+
new Role
```

se requiere:

```text
INSERT Role
    ↓
resolve Role ID
    ↓
INSERT user_roles(user_id, role_id)
```

---

# 103. Generated Source Identifier

Para:

```text
new User
+
existing Role
```

se requiere:

```text
INSERT User
    ↓
resolve User ID
    ↓
INSERT membership
```

---

# 104. Both New

```text
new User
+
new Role
```

requiere:

```text
Insert User ──┐
              ├──→ Membership Insert
Insert Role ──┘
```

---

# 105. Dependency Graph

```text
Insert(Source) ───┐
                  ▼
              Identity Barrier
                  ▲
Insert(Target) ───┘
                  │
                  ▼
         Insert Membership
```

---

# 106. Temporary IDs

Nunca deberán escribirse:

```text
temporary UoW identifiers
```

en join table.

---

# 107. Membership Persistence Operation

Propuesta:

```php
interface MembershipPersistenceOperation
{
    public function relationship(): RelationshipId;

    public function source(): EntityKey;
}
```

Implementaciones:

```text
AddMembershipOperation
RemoveMembershipOperation
ClearMembershipsOperation
ReplaceMembershipSetOperation
```

---

# 108. Persistence Operation ≠ SQL

`AddMembershipOperation` no es:

```sql
INSERT INTO ...
```

El Query Engine determina representación física.

---

# 109. Add Membership

Semánticamente:

```text
ensure membership exists
```

puede representarse mediante distintas estrategias según concurrency/capabilities.

---

# 110. Duplicate Prevention

La protección debe existir en dos niveles:

```text
ORM
+
Database when possible
```

---

# 111. ORM Duplicate Prevention

`PersistentCollection` y UoW no deberán planificar dos inserts equivalentes para el mismo `MembershipKey`.

---

# 112. Database Duplicate Prevention

Recomendado:

```text
UNIQUE(source_fk..., target_fk...)
```

o PK equivalente.

---

# 113. ORM Check ≠ Concurrency Protection

Aunque ORM haya comprobado ausencia:

```text
another transaction
```

puede insertar simultáneamente.

Por ello la restricción DB sigue siendo importante.

---

# 114. Concurrent Add

Dos transacciones:

```text
T1 add Role#5
T2 add Role#5
```

deberán manejar el unique conflict según policy.

---

# 115. Idempotent Add Policy

Puede tratarse un duplicate-key confirmado para exactamente la misma membership como:

```text
already satisfied
```

solo si la operación fue declarada idempotente y la evidencia es suficiente.

---

# 116. Generic Duplicate-Key ≠ Success

No deberá tragarse cualquier unique violation.

Debe identificarse el constraint/semántica correspondiente.

---

# 117. Remove Missing Membership

Eliminar una membresía ya inexistente puede ser:

```text
idempotent success
```

según policy.

---

# 118. Strict Remove

También podrá existir policy:

```text
require membership to exist
```

para dominios que lo necesiten.

---

# 119. Ordering

Una relación many-to-many puede requerir:

```text
ORDER BY role.name
```

sin que el orden forme parte de la membresía persistente.

---

# 120. Derived Ordering

```text
orderBy target field
```

es orden derivado.

No requiere columna de posición.

---

# 121. Persistent Ordering

Si el usuario necesita:

```text
position
```

en join row:

```text
user_roles
──────────
user_id
role_id
position
```

la relación ya posee payload de asociación.

---

# 122. Association Payload

VoltStack deberá distinguir:

```text
technical payload
```

de:

```text
domain association state
```

---

# 123. V1 Recommendation

Si join row tiene campos modificables significativos:

```text
position
assigned_at
status
permissions
metadata
```

recomendar:

```text
Association Entity
```

---

# 124. Pivot Concept

Para compatibilidad ergonómica Laravel-like, Model API podrá exponer una abstracción `pivot`.

Pero:

```text
Pivot API
≠
Core ORM Identity Model
```

---

# 125. Pivot Data

Si se soporta para join tables simples, deberá existir un modelo claro de:

```text
association attributes
```

sin convertir implícitamente cada pivot en una entidad managed.

---

# 126. Architectural Preference

VoltStack debería priorizar:

```text
Simple ManyToMany
```

para membership pura,

y:

```text
Association Entity
```

para asociación rica.

Esto reduce ambigüedad.

---

# 127. Extra-Lazy Operations

Grandes colecciones no deberían inicializarse para operaciones simples.

Ejemplos:

```text
count()
contains()
containsKey()
slice()
```

---

# 128. Extra-Lazy Count

```php
$user->roles->count();
```

podrá ejecutar un count relationship query sin cargar todos los Roles.

---

# 129. Count + Pending Mutations

El resultado deberá reconciliar:

```text
database count
+
pending adds
-
pending removes
```

sin doble contar memberships.

---

# 130. Extra-Lazy contains()

Puede preguntar:

```text
does MembershipKey exist?
```

mediante Entity Query/Query Engine.

---

# 131. Pending State First

Antes de DB:

```text
pending add
→ PRESENT

pending remove
→ ABSENT
```

---

# 132. Extra-Lazy Query ≠ Collection Initialization

Después de:

```text
contains(Role#5)
```

la colección puede continuar:

```text
UNINITIALIZED
```

---

# 133. Extra-Lazy Slice

```php
$user->roles->slice(0, 20);
```

produce una vista parcial.

No:

```text
CollectionCompleteness = COMPLETE
```

---

# 134. Large Collections

El sistema deberá evitar requerir:

```text
O(N)
```

memory para toda operación.

---

# 135. Large Collection Strategies

Podrán incluir:

- extra-lazy queries;
- streaming;
- pagination;
- batch operations;
- database-side membership diff;
- bounded snapshots;
- delta tracking.

---

# 136. Full Snapshot Problem

Para millones de memberships:

```text
snapshot = all target IDs
```

puede ser inviable.

---

# 137. Snapshot Strategy

Propuesta:

```php
enum CollectionSnapshotStrategy
{
    case FULL;
    case DELTA;
    case DATABASE_ASSISTED;
    case NONE_FOR_UNINITIALIZED;
}
```

---

# 138. DELTA

Con colección no inicializada:

```text
pending adds
pending removes
clear intent
```

puede ser suficiente sin snapshot completa.

---

# 139. Change Tracking ≠ Full Initialization

Regla importante:

> **Registrar una mutación many-to-many no deberá implicar cargar la colección completa.**

---

# 140. clear() + add()

Caso:

```text
clear()
add(Role#1)
add(Role#2)
```

puede expresarse:

```text
Desired state after clear:
{Role#1, Role#2}
```

sin leer baseline.

---

# 141. Persistence Planner

Podrá producir:

```text
ClearMemberships(User#10)
AddMembership(User#10, Role#1)
AddMembership(User#10, Role#2)
```

---

# 142. Ordering of Membership Operations

Planner deberá respetar dependencias:

```text
entity identity generation
before
membership insertion
```

y:

```text
membership deletion
before
entity deletion
```

cuando FK lo requiera.

---

# 143. Source Delete

Eliminar `User` puede requerir borrar join rows.

Esto puede lograrse mediante:

- ORM membership cleanup;
- DB `ON DELETE CASCADE`;
- otra estrategia capability-aware.

---

# 144. DB Cascade ≠ ORM Lifecycle

Si DB elimina join rows automáticamente, no existen "join entities" a las que fabricar lifecycle events.

---

# 145. Target Delete

Eliminar `Role` requiere resolver memberships que lo referencian.

---

# 146. Cascade Remove

Eliminar un User no deberá eliminar Roles compartidos por default.

---

# 147. Safe Default

```text
ManyToMany cascade REMOVE target = OFF
```

---

# 148. Shared Target

```text
User#10 ─┐
User#20 ─┼── Role#1
User#30 ─┘
```

demuestra por qué cascade remove puede ser peligroso.

---

# 149. Orphan Removal

`orphanRemoval` no tiene la misma semántica natural en many-to-many que en one-to-many owned-child.

---

# 150. Default

```text
orphanRemoval = unsupported/off
```

para many-to-many puro.

---

# 151. Why

Quitar:

```text
Role#1
```

de:

```text
User#10.roles
```

no demuestra que Role#1 no tenga otros owners.

---

# 152. Query Navigation

Ejemplo:

```php
User::query()
    ->whereHas('roles', fn ($q) =>
        $q->where('name', 'admin')
    );
```

---

# 153. Semantic Resolution

Se transforma conceptualmente en:

```text
Relationship Path
→
Relationship Metadata
→
Semantic Query Graph
→
Query Planner
```

---

# 154. No Hardcoded Join Table in Entity Query

Entity Query no deberá concatenar:

```sql
JOIN user_roles ...
```

manualmente.

---

# 155. Query Planner Freedom

Puede usar:

```text
JOIN
EXISTS
SEMI-JOIN
SUBQUERY
```

según dialect/cost/capabilities.

---

# 156. contains Relationship Predicate

Ejemplo:

```php
User::query()
    ->whereRelated('roles', $admin);
```

se resuelve mediante target identity.

---

# 157. Collection Query

Model API podrá ofrecer:

```php
$user->roles()
    ->where('active', true)
    ->get();
```

---

# 158. Collection Query ≠ Collection State

Consultar:

```text
active roles
```

no significa reemplazar:

```text
$user->roles
```

por esa subset.

---

# 159. Explicit Load

Para modificar el managed relationship state deberá existir:

```text
load relationship
```

con coverage explícita.

---

# 160. Relationship Loader

Propuesta:

```php
interface ManyToManyLoader
{
    public function load(
        object $source,
        ManyToManyMetadata $relationship,
        RelationshipLoadPlan $plan,
        RelationshipLoadContext $context,
    ): RelationshipLoadResult;
}
```

---

# 161. Load Result

```php
final readonly class RelationshipLoadResult
{
    /**
     * @param list<object> $targets
     */
    public function __construct(
        public array $targets,
        public CollectionCompleteness $completeness,
        public RelationshipKnowledge $knowledge,
    ) {}
}
```

---

# 162. Hydration vs Loading

```text
Loading
=
decide/request data acquisition

Hydration
=
materialize already returned data
```

---

# 163. Relationship System Does Not Execute SQL Directly

Loader utiliza Entity Query/Query Engine.

---

# 164. Inverse Synchronization

Para bidireccional:

```php
$user->roles->add($role);
```

puede sincronizar:

```php
$role->users
```

cuando sea seguro.

---

# 165. Inverse Collection Uninitialized

No deberá cargarse toda `Role.users` solo para reflejar una adición conocida.

---

# 166. Pending Inverse Mutation

Puede registrarse:

```text
Role#1.users
pending add User#10
```

sin inicialización.

---

# 167. Remove Synchronization

Igualmente:

```text
User.roles remove Role#1
```

puede registrar:

```text
Role.users pending remove User#10
```

---

# 168. Infinite Recursion Prevention

No:

```text
User.roles.add(Role)
→ Role.users.add(User)
→ User.roles.add(Role)
→ ...
```

---

# 169. Synchronization Token

Puede utilizarse:

```php
final readonly class RelationshipMutationToken
{
    public function __construct(
        public RelationshipId $origin,
        public MutationId $mutation,
    ) {}
}
```

---

# 170. Owning Authority

Aunque inverse side se sincronice:

```text
owning side
```

continúa siendo autoridad para persistence.

---

# 171. Inverse-Only Mutation

Si application modifica únicamente inverse side en strict mode:

```text
Role.users.add(User)
```

pero owning side es `User.roles`, puede:

- sincronizar owning side automáticamente;
- fallar;
- registrar inconsistency.

La policy deberá ser explícita.

---

# 172. Recommended Policy

Para APIs de alto nivel:

```text
synchronize both sides
```

sin hidden collection initialization.

Para internals:

```text
owning side remains authoritative
```

---

# 173. Relationship Lifecycle

Una membership no es entidad, por tanto no recibe:

```text
prePersist entity
postPersist entity
```

---

# 174. Relationship Events

Podrán existir eventos ORM específicos:

```text
preMembershipAdd
postMembershipAdd
preMembershipRemove
postMembershipRemove
```

si el Event Architecture los define.

---

# 175. Event Timing

`postMembershipAdd` después de statement success no significa:

```text
transaction committed
```

---

# 176. External Effects

Side effects irreversibles deberán utilizar:

```text
afterCommit
Outbox
JobAfterCommit
```

---

# 177. Batch Membership Inserts

Múltiples adds compatibles podrán agruparse.

---

# 178. Batch ≠ Bulk Semantic Bypass

Batch deberá preservar:

- membership identity;
- failure attribution cuando sea necesaria;
- transaction semantics;
- consistency.

---

# 179. Example

```text
Add User#10→Role#1
Add User#10→Role#2
Add User#10→Role#3
```

puede convertirse físicamente en un multi-row insert.

---

# 180. Planner Still Sees Three Memberships

La optimización física no destruye la semántica individual.

---

# 181. Batch Delete

Removals compatibles también podrán agruparse.

---

# 182. Parameter Limits

Batch sizing respetará:

- max parameters;
- max SQL length;
- driver limits;
- memory;
- deadline.

---

# 183. Persistence Outcome

Cada operación podrá terminar:

```text
SUCCESS
FAILED
CONFLICT
UNKNOWN
NOT_EXECUTED
```

---

# 184. Batch UNKNOWN

Si un batch insert pierde conexión sin evidencia suficiente:

```text
which memberships were inserted?
```

puede ser desconocido.

---

# 185. No Fabricated Snapshot

No deberá avanzarse toda la snapshot como si el batch hubiese tenido éxito.

---

# 186. Consistency State

La colección puede quedar:

```text
UNCERTAIN
```

y el PersistenceContext:

```text
TAINTED
```

según severidad.

---

# 187. Transaction Semantics

```text
membership statement success
≠
transaction commit
```

---

# 188. Flush Semantics

```text
flush success
≠
durable membership
```

si existe transacción externa activa.

---

# 189. Rollback

Después de:

```text
add Role#5
flush
rollback
```

el objeto puede continuar mostrando:

```text
Role#5 present
```

aunque DB haya restaurado la ausencia.

---

# 190. No Collection Time Machine

VoltStack no deberá rebobinar automáticamente todo el object graph.

---

# 191. Rollback Reconciliation

El sistema transaccional podrá:

- marcar stale;
- conservar dirty intent;
- requerir refresh;
- restaurar snapshot cuando sea demostrablemente seguro.

---

# 192. UNKNOWN Commit

Caso:

```text
COMMIT sent
connection lost
```

puede dejar:

```text
membership durability = UNKNOWN
```

---

# 193. No Retry Blind

Reintentar membership insert sin considerar idempotency puede producir duplicate-key.

---

# 194. Retry Safety

```text
RetrySafe
=
OperationIdempotent
∨
DuplicateCanBeProvenEquivalent
∨
TransactionDefinitelyRolledBack
```

---

# 195. Tenant Isolation

Para relación tenant-local:

```text
Tenant(Source)
=
Tenant(Target)
```

por default.

---

# 196. MembershipKey Context

La identidad semántica deberá incorporar namespaces efectivos mediante los `EntityKey`.

---

# 197. Cross-Tenant Membership

Default:

```text
REJECT
```

---

# 198. Cross-Shard Membership

Default:

```text
REJECT
```

salvo mapping explícito.

---

# 199. Cross-Database Join Table

Una DB join table normalmente no puede representar directamente entidades alojadas en databases independientes.

---

# 200. Logical Distributed Many-to-Many

Podría existir una extensión distribuida, pero no deberá reutilizar silenciosamente las garantías del many-to-many local.

---

# 201. Distributed Association

Requeriría semánticas explícitas de:

- location;
- routing;
- consistency;
- compensation;
- durability;
- partial failure.

Fuera del core many-to-many convencional.

---

# 202. Security

Una membership no implica authorization.

Ejemplo:

```text
User.roles contains AdminRole
```

puede alimentar Authorization System, pero Database Relationship System no decide políticas de acceso.

---

# 203. Mass Assignment

```text
role_ids = [...]
```

desde HTTP no deberá aplicarse directamente sin validation/authorization.

---

# 204. Tenant Input

Nunca confiar en:

```text
tenant_id
```

proporcionado por el cliente para construir relationship namespace.

---

# 205. Join Table Injection

Nombres de:

- join table;
- columns;
- referenced columns;

provienen de metadata compilada, no de input runtime.

---

# 206. Mapping Validation

Bootstrap deberá validar:

```text
source entity exists
target entity exists
join table metadata valid
source join columns complete
target join columns complete
identifier type compatibility
owning/inverse consistency
mappedBy/inversedBy correctness
collection semantics
cascade policies
tenant/shard compatibility
association payload restrictions
```

---

# 207. mappedBy Validation

Si:

```text
Role.users mappedBy roles
```

deberá resolverse:

```text
User.roles
```

como relación compatible hacia `Role`.

---

# 208. Same Logical Association

Ambos lados deberán compartir:

```text
RelationshipAssociationId
```

o referencia semántica equivalente.

---

# 209. Join Mapping Authority

Solo owning side deberá definir autoridad física completa de join table en mapping bidireccional.

---

# 210. Inverse Mapping Duplication

No permitir dos configuraciones contradictorias:

```text
User.roles → user_roles
Role.users → role_user
```

para la misma asociación bidireccional.

---

# 211. Composite Source Identifier

Ejemplo:

```text
tenant_id
user_id
```

puede producir:

```text
join table source columns:
tenant_id
user_id
```

---

# 212. Composite Target Identifier

También:

```text
organization_id
role_id
```

---

# 213. Composite Membership

```text
MembershipKey
=
Source EntityKey
+
Target EntityKey
```

continúa siendo una unidad semántica aunque físicamente use cuatro o más columnas.

---

# 214. Partial Composite Key

No deberá planificarse membership con identidad incompleta.

---

# 215. Type Conversion

Cada join value utilizará:

```text
Database Type System
```

no casts locales.

---

# 216. Join Column Order

La metadata compilada deberá determinar correspondencia explícita.

No depender de orden accidental de reflection.

---

# 217. Join Table Unique Constraint

Para semántica SET se recomienda:

```text
UNIQUE(source identity columns, target identity columns)
```

---

# 218. Schema Diagnostics

VoltStack podrá advertir:

```text
Many-to-many SET mapping has no observed unique constraint
```

sin afirmar que mapping sea schema.

---

# 219. Collection Initialization Snapshot

Cuando una colección completa se carga exitosamente:

```text
Current Memberships
=
Baseline Memberships
```

antes de lifecycle/application mutation posterior.

---

# 220. Hydration Does Not Dirty Collection

Las adiciones internas durante assembly:

```text
HYDRATE
```

no serán `ADD` domain mutations.

---

# 221. Mutation Modes

```php
enum CollectionMutationMode
{
    case HYDRATE;
    case DOMAIN;
    case REFRESH;
    case RECONCILE;
    case INVERSE_SYNC;
}
```

---

# 222. Hydration Mode

No:

```text
hydrate Role#1
→ pending add Role#1
```

---

# 223. Refresh

Una relación puede refrescarse explícitamente.

---

# 224. Dirty Refresh Policy

Si existen pending changes:

```text
REJECT_DIRTY
DISCARD_LOCAL_CHANGES
MERGE_IF_SAFE
```

---

# 225. Default Refresh Policy

```text
REJECT_DIRTY
```

---

# 226. Normal Requery ≠ Refresh

Una query que vuelve a encontrar `User#10` no deberá reemplazar ciegamente `User#10.roles` si contiene cambios locales.

---

# 227. Partial Relationship Load

Una query filtrada:

```text
roles where active = true
```

no deberá sobrescribir una colección COMPLETE existente con una subset.

---

# 228. Merge Policy

Una subset puede enriquecer conocimiento:

```text
known members += hydrated members
```

sin afirmar completitud.

---

# 229. Completeness Monotonicity

No siempre es estrictamente monotónica debido a staleness.

Pero dentro de una hydration operation consistente:

```text
PARTIAL
→
COMPLETE
```

solo con evidencia suficiente.

---

# 230. STALE Complete Collection

Una colección puede haber sido COMPLETE cuando se cargó y luego convertirse en STALE.

---

# 231. External Mutations

Raw SQL sobre join table puede invalidar:

```text
membership knowledge
```

---

# 232. Bulk Relationship Operations

Operaciones directas sobre join table deberán marcar colecciones managed afectadas como:

```text
STALE
```

cuando sea posible determinar alcance.

---

# 233. Cache Interaction

Relationship state managed no deberá confundirse con second-level cache.

---

# 234. Cached Membership Data

Si posteriormente existe relationship cache:

```text
cache hit
```

deberá pasar por las mismas reglas de:

- identity;
- completeness;
- staleness;
- context;
- hydration.

---

# 235. Serialization

Serializar User no deberá inicializar Roles automáticamente.

---

# 236. Recursive Serialization

Bidirectional:

```text
User.roles
Role.users
```

puede producir ciclos.

Serialization System deberá tener su propia política.

---

# 237. N+1

Caso:

```php
foreach ($users as $user) {
    foreach ($user->roles as $role) {
        // ...
    }
}
```

puede generar N+1.

---

# 238. N+1 Detection

Telemetry deberá asociar:

```text
same relationship
multiple source identities
same execution context
repeated lazy loads
```

---

# 239. Batch Recommendation

Diagnostics podrá sugerir:

```text
with('roles')
batch load
```

sin cambiar automáticamente semántica.

---

# 240. Persistent Runtime

Compartible:

```text
ManyToManyMetadata
JoinTableMetadata
compiled accessors
compiled collection factories
mapping validators
load-plan templates
```

Scoped:

```text
PersistentCollection
managed entities
membership snapshots
pending adds/removes
assembly registries
lazy load state
batch queues
refresh state
```

---

# 241. FrankenPHP

No deberá existir:

```php
static array $relationshipCollections;
```

con entidades de requests anteriores.

---

# 242. Worker Reset

Al terminar scope:

```text
clear relationship load queues
clear membership assembly registries
clear snapshots
release PersistentCollections
clear pending inverse synchronization
clear extra-lazy operation state
```

---

# 243. Concurrent Collection Initialization

Dos fibers no deberán inicializar la misma colección de manera incompatible.

---

# 244. Initialization State

Propuesta:

```php
enum CollectionInitializationState
{
    case NOT_STARTED;
    case INITIALIZING;
    case INITIALIZED;
    case FAILED;
}
```

---

# 245. Concurrent Access

Una segunda operación podrá:

- esperar;
- reutilizar initialization result;
- recibir mismo failure.

---

# 246. Recursive Initialization

Bidirectional graph loading puede intentar volver a la misma colección.

Deberá existir guard.

---

# 247. Cancellation

Una carga cancelada no deberá marcar:

```text
collection = COMPLETE
```

---

# 248. Partial Cancellation

Si algunos targets ya fueron hidratados:

```text
known members
```

pueden existir,

pero:

```text
completeness ≠ COMPLETE
```

---

# 249. Resource Governance

Policies sugeridas:

```php
final readonly class ManyToManyResourcePolicy
{
    public function __construct(
        public int $maxEagerCollectionSize,
        public int $maxBatchSources,
        public int $maxBatchMemberships,
        public int $maxBufferedMemberships,
    ) {}
}
```

---

# 250. Huge Eager Join

Una consulta:

```text
1000 Users × 1000 Roles
```

puede producir un resultado enorme.

Planner/diagnostics deberán detectar riesgos de row explosion.

---

# 251. Join Explosion

Estimación:

```text
PhysicalRows
≈
Σ memberships(root_i)
```

y con múltiples to-many joins puede multiplicarse aún más.

---

# 252. Multiple Collection Fetch Joins

Ejemplo:

```text
User
JOIN roles
JOIN groups
```

puede producir:

```text
roles × groups
```

por root.

---

# 253. Planner Warning

Puede recomendar:

```text
split fetch
batch fetch
```

---

# 254. Streaming

Many-to-many fetch join requiere conocer cuándo una root collection está completa antes de yield si se promete graph completo.

---

# 255. Safe Yield

```text
SafeYield(root)
=
RootGraphComplete
∧
NoFutureRowCanExtendFetchedCollections
```

---

# 256. Required Ordering

Streaming grouped hydration puede requerir rows ordenadas por root identity.

---

# 257. No Guaranteed Group Boundary

Si execution no garantiza orden:

```text
buffer
```

o:

```text
reject graph streaming
```

---

# 258. Collection Query Streaming

Streaming targets individuales de una relationship query es distinto de inicializar la colección completa.

---

# 259. Memory

Incluso si rows se streamean, managed target entities pueden permanecer en IdentityMap.

---

# 260. Large Dataset Guidance

Para procesamiento masivo se combinará con:

```text
Chunk Processing
Lazy Collection
EntityManager clear boundaries
```

documentados posteriormente.

---

# 261. Telemetry

Métricas sugeridas:

```text
orm.relationship.many_to_many.loads
orm.relationship.many_to_many.lazy_loads
orm.relationship.many_to_many.eager_loads
orm.relationship.many_to_many.batch_loads

orm.relationship.many_to_many.memberships.hydrated
orm.relationship.many_to_many.memberships.deduplicated

orm.relationship.many_to_many.adds
orm.relationship.many_to_many.removes
orm.relationship.many_to_many.clears
orm.relationship.many_to_many.syncs

orm.relationship.many_to_many.extra_lazy.count
orm.relationship.many_to_many.extra_lazy.contains
orm.relationship.many_to_many.extra_lazy.slice

orm.relationship.many_to_many.join_explosion_warnings
orm.relationship.many_to_many.broken_mappings
orm.relationship.many_to_many.persistence.unknown
```

---

# 262. Low-Cardinality Labels

Permitidos:

```text
relationship type
fetch strategy
collection state
outcome
driver/platform
```

Evitar:

```text
source ID
target ID
tenant ID
email
role name
```

---

# 263. Diagnostics

Ejemplo:

```text
MANY-TO-MANY RELATIONSHIP

Relationship:
    User.roles

Target:
    Role

Ownership:
    OWNING

Join Table:
    user_roles

Source Join:
    user_id → users.id

Target Join:
    role_id → roles.id

Semantics:
    SET

Collection:
    INITIALIZED

Completeness:
    COMPLETE

Known Members:
    5

Pending Adds:
    1

Pending Removes:
    2
```

---

# 264. Explain Membership Plan

```text
MEMBERSHIP PERSISTENCE PLAN

Relationship:
    User.roles

Source:
    User#[redacted]

Operations:
    REMOVE Role#[redacted]
    REMOVE Role#[redacted]
    ADD    Role#[redacted]

Requires:
    Source identity: RESOLVED
    Target identities: RESOLVED

Transaction:
    REQUIRED

Batch Eligible:
    YES
```

---

# 265. Error Taxonomy

```text
ManyToManyRelationshipException
├── ManyToManyMetadataException
├── ManyToManyMappingException
├── ManyToManyOwnershipException
├── JoinTableMappingException
├── JoinColumnMappingException
├── DuplicateMembershipException
├── UnknownMembershipException
├── PartialMembershipIdentityException
├── CollectionInitializationException
├── CollectionCompletenessException
├── CollectionMutationException
├── CollectionRefreshException
├── CollectionConcurrentInitializationException
├── CollectionRecursiveInitializationException
├── MembershipPersistenceException
├── MembershipConflictException
├── MembershipSynchronizationException
├── AssociationEntityRequiredException
├── CrossTenantMembershipException
├── CrossShardMembershipException
├── ManyToManyStreamingException
├── ManyToManyResourceLimitException
├── ManyToManyRuntimeIsolationException
└── ManyToManyInvariantException
```

---

# 266. Directorio propuesto

```text
src/Quantum/Database/ORM/Relationship/ManyToMany/
│
├── Metadata/
│   ├── ManyToManyMetadata.php
│   ├── JoinTableMetadata.php
│   ├── JoinColumnMetadata.php
│   ├── CollectionSemantics.php
│   └── RelationshipOwnership.php
│
├── Collection/
│   ├── PersistentCollection.php
│   ├── RelationshipCollectionState.php
│   ├── CollectionCompleteness.php
│   ├── CollectionInitializationState.php
│   ├── MembershipKnowledge.php
│   └── CollectionMutationMode.php
│
├── Membership/
│   ├── MembershipKey.php
│   ├── MembershipSet.php
│   └── MembershipAssemblyRegistry.php
│
├── Loading/
│   ├── ManyToManyLoader.php
│   ├── ManyToManyBatchLoader.php
│   ├── ManyToManyExtraLazyLoader.php
│   ├── RelationshipLoadPlan.php
│   └── RelationshipLoadResult.php
│
├── Hydration/
│   ├── ManyToManyHydrationBinding.php
│   ├── ManyToManyCollectionAssembler.php
│   └── MembershipHydrationContext.php
│
├── Snapshot/
│   ├── ManyToManySnapshot.php
│   └── CollectionSnapshotStrategy.php
│
├── ChangeTracking/
│   ├── MembershipChangeSet.php
│   ├── MembershipChangeTracker.php
│   └── PendingMembershipMutations.php
│
├── Persistence/
│   ├── MembershipPersistenceOperation.php
│   ├── AddMembershipOperation.php
│   ├── RemoveMembershipOperation.php
│   ├── ClearMembershipsOperation.php
│   └── ReplaceMembershipSetOperation.php
│
├── Synchronization/
│   ├── BidirectionalCollectionSynchronizer.php
│   ├── RelationshipMutationToken.php
│   └── PendingInverseMutation.php
│
├── Policy/
│   ├── LazyRelationshipPolicy.php
│   ├── CollectionRefreshPolicy.php
│   ├── MembershipConflictPolicy.php
│   └── ManyToManyResourcePolicy.php
│
├── Validation/
│   ├── ManyToManyMappingValidator.php
│   ├── JoinTableMappingValidator.php
│   └── ManyToManyGraphValidator.php
│
├── Diagnostics/
│   ├── ManyToManyDiagnostics.php
│   └── MembershipPlanExplainer.php
│
├── Telemetry/
│   └── ManyToManyTelemetry.php
│
└── Exception/
    └── ...
```

---

# 267. Testing Strategy

La suite deberá cubrir al menos:

```text
unidirectional mapping
bidirectional mapping
owning/inverse validation
empty collection
uninitialized collection
partial collection
complete collection
lazy initialization
eager JOIN
batch loading
membership dedup
IdentityMap reuse
add
duplicate add
remove
clear
sync
pending mutation without initialization
cascade persist
generated IDs
composite IDs
extra-lazy count
extra-lazy contains
extra-lazy slice
inverse synchronization
refresh
rollback
UNKNOWN outcome
tenant isolation
persistent workers
concurrent initialization
streaming graph assembly
large collections
```

---

# 268. Core Test — UNINITIALIZED ≠ EMPTY

```text
collection state = UNINITIALIZED
```

deberá impedir inferir:

```text
membership set = {}
```

---

# 269. Core Test — Empty Complete Collection

Una carga completa sin targets:

```text
state = INITIALIZED
completeness = COMPLETE
members = {}
```

---

# 270. Duplicate JOIN Test

Múltiples physical rows con mismo `MembershipKey` producen una sola membership.

---

# 271. IdentityMap Test

El mismo Role presente en múltiples Users reutiliza canonical instance.

---

# 272. Add Without Initialization

```text
UNINITIALIZED
+
add Role#5
```

no ejecuta SELECT completo.

---

# 273. Remove Without Initialization

Cuando API permite una remoción identity-aware:

```text
UNINITIALIZED
+
remove Role#5
```

puede registrar pending remove sin cargar toda la colección.

---

# 274. Clear Without Initialization

`clear()` puede producir semantic clear intent sin SELECT previo.

---

# 275. sync Test

No asume colección vacía cuando baseline es desconocida.

---

# 276. Generated ID Test

Membership insert espera source y target IDs reales.

---

# 277. Partial Hydration Test

Filtered load no marca colección COMPLETE.

---

# 278. Refresh Dirty Test

Default rechaza refresh destructivo de pending mutations.

---

# 279. Rollback Test

Rollback no rebobina automáticamente la colección PHP.

---

# 280. UNKNOWN Test

Outcome desconocido no actualiza snapshot como éxito.

---

# 281. Persistent Runtime Test

Collections y snapshots no cruzan requests.

---

# 282. Concurrent Initialization Test

Dos fibers no producen dos inicializaciones incompatibles.

---

# 283. Architectural Invariants

## DB-ORM-MANY-TO-MANY-001

Many-to-many representará membership entre identidades.

## DB-ORM-MANY-TO-MANY-002

Many-to-many será distinto de join table.

## DB-ORM-MANY-TO-MANY-003

Many-to-many será distinto de PersistentCollection.

## DB-ORM-MANY-TO-MANY-004

Join row será distinto de Entity por default.

## DB-ORM-MANY-TO-MANY-005

Association Entity será explícita.

## DB-ORM-MANY-TO-MANY-006

Asociaciones con lifecycle/state propio deberán favorecer Association Entity.

## DB-ORM-MANY-TO-MANY-007

Mapping metadata será distinto de Schema metadata.

## DB-ORM-MANY-TO-MANY-008

Bidirectional relationship tendrá una autoridad de persistence.

## DB-ORM-MANY-TO-MANY-009

Owning side será distinto de aggregate ownership.

## DB-ORM-MANY-TO-MANY-010

Unidirectional many-to-many será válido.

## DB-ORM-MANY-TO-MANY-011

SET será la semántica default recomendada.

## DB-ORM-MANY-TO-MANY-012

Duplicate logical membership será idempotente bajo SET semantics.

## DB-ORM-MANY-TO-MANY-013

Membership equality utilizará EntityKey.

## DB-ORM-MANY-TO-MANY-014

UNINITIALIZED será distinto de empty.

## DB-ORM-MANY-TO-MANY-015

PARTIAL será distinto de COMPLETE.

## DB-ORM-MANY-TO-MANY-016

Unknown absence no será known absence.

## DB-ORM-MANY-TO-MANY-017

PersistentCollection no generará SQL.

## DB-ORM-MANY-TO-MANY-018

PersistentCollection será distinta de Repository.

## DB-ORM-MANY-TO-MANY-019

MembershipKey será distinto de EntityKey.

## DB-ORM-MANY-TO-MANY-020

MembershipKey será distinto de join-row PK física.

## DB-ORM-MANY-TO-MANY-021

Lazy initialization no producirá dirty state.

## DB-ORM-MANY-TO-MANY-022

Lazy loading podrá prohibirse.

## DB-ORM-MANY-TO-MANY-023

Eager loading será distinto de JOIN.

## DB-ORM-MANY-TO-MANY-024

JOIN row multiplication no duplicará memberships SET.

## DB-ORM-MANY-TO-MANY-025

Target IdentityMap canonicalization será preservada.

## DB-ORM-MANY-TO-MANY-026

Root dedup dependerá de ResultShape.

## DB-ORM-MANY-TO-MANY-027

Hydration utilizará HydrationPlan.

## DB-ORM-MANY-TO-MANY-028

Hydration no analizará SQL.

## DB-ORM-MANY-TO-MANY-029

Membership assembly registry será scoped.

## DB-ORM-MANY-TO-MANY-030

All-null joined target no producirá fake target.

## DB-ORM-MANY-TO-MANY-031

Known empty collection requerirá coverage suficiente.

## DB-ORM-MANY-TO-MANY-032

Many-to-many collection no será NULL.

## DB-ORM-MANY-TO-MANY-033

Batch load marcará COMPLETE solo con cobertura completa.

## DB-ORM-MANY-TO-MANY-034

Ad hoc filtered load será PARTIAL.

## DB-ORM-MANY-TO-MANY-035

Relationship-level filter y query filter serán distintos.

## DB-ORM-MANY-TO-MANY-036

add() no ejecutará INSERT inmediatamente.

## DB-ORM-MANY-TO-MANY-037

remove() no eliminará target entity.

## DB-ORM-MANY-TO-MANY-038

clear() no eliminará target entities.

## DB-ORM-MANY-TO-MANY-039

clear() no requerirá full initialization obligatoria.

## DB-ORM-MANY-TO-MANY-040

attach/detach serán API sobre membership engine canónico.

## DB-ORM-MANY-TO-MANY-041

sync no asumirá baseline vacío.

## DB-ORM-MANY-TO-MANY-042

Sync strategy será explícita/capability-aware.

## DB-ORM-MANY-TO-MANY-043

Delete-and-reinsert no será default universal.

## DB-ORM-MANY-TO-MANY-044

Snapshot conservará completeness.

## DB-ORM-MANY-TO-MANY-045

Partial snapshot no probará ausencia global.

## DB-ORM-MANY-TO-MANY-046

Membership ChangeSet será distinto de SQL.

## DB-ORM-MANY-TO-MANY-047

Pending mutation podrá existir sin initialization.

## DB-ORM-MANY-TO-MANY-048

Membership knowledge conservará UNKNOWN.

## DB-ORM-MANY-TO-MANY-049

Application current membership será distinto de DB state durante UoW.

## DB-ORM-MANY-TO-MANY-050

Cascade persist será explícito.

## DB-ORM-MANY-TO-MANY-051

Transient target sin cascade/persist será rechazado.

## DB-ORM-MANY-TO-MANY-052

Generated source ID será dependency barrier.

## DB-ORM-MANY-TO-MANY-053

Generated target ID será dependency barrier.

## DB-ORM-MANY-TO-MANY-054

Temporary UoW IDs nunca serán join values.

## DB-ORM-MANY-TO-MANY-055

Membership operation será semántica, no SQL.

## DB-ORM-MANY-TO-MANY-056

Duplicate prevention existirá en ORM y preferentemente DB.

## DB-ORM-MANY-TO-MANY-057

ORM duplicate check no reemplazará DB concurrency protection.

## DB-ORM-MANY-TO-MANY-058

Generic unique violation no será automáticamente idempotent success.

## DB-ORM-MANY-TO-MANY-059

Remove-missing semantics serán policy-driven.

## DB-ORM-MANY-TO-MANY-060

Derived ordering será distinto de persistent association ordering.

## DB-ORM-MANY-TO-MANY-061

Rich association payload favorecerá Association Entity.

## DB-ORM-MANY-TO-MANY-062

Pivot API no definirá el core identity model.

## DB-ORM-MANY-TO-MANY-063

Extra-lazy operation no inicializará colección completa.

## DB-ORM-MANY-TO-MANY-064

Extra-lazy count reconciliará pending mutations.

## DB-ORM-MANY-TO-MANY-065

Pending membership state tendrá prioridad sobre extra-lazy DB observation.

## DB-ORM-MANY-TO-MANY-066

Extra-lazy slice producirá partial knowledge.

## DB-ORM-MANY-TO-MANY-067

Large collections no requerirán full snapshot universal.

## DB-ORM-MANY-TO-MANY-068

Change tracking no requerirá full initialization.

## DB-ORM-MANY-TO-MANY-069

clear + add podrá expresarse como delta sin baseline read.

## DB-ORM-MANY-TO-MANY-070

Planner respetará identity dependencies.

## DB-ORM-MANY-TO-MANY-071

Membership cleanup precederá entity delete cuando sea requerido.

## DB-ORM-MANY-TO-MANY-072

DB cascade será distinto de ORM lifecycle.

## DB-ORM-MANY-TO-MANY-073

Cascade remove target estará OFF por default.

## DB-ORM-MANY-TO-MANY-074

Orphan removal estará OFF/unsupported por default.

## DB-ORM-MANY-TO-MANY-075

Relationship query utilizará Semantic Query Engine.

## DB-ORM-MANY-TO-MANY-076

Relationship System no generará JOIN SQL.

## DB-ORM-MANY-TO-MANY-077

Query Planner podrá escoger JOIN/EXISTS/etc.

## DB-ORM-MANY-TO-MANY-078

Collection Query será distinto de managed collection state.

## DB-ORM-MANY-TO-MANY-079

Filtered query no reemplazará collection state implícitamente.

## DB-ORM-MANY-TO-MANY-080

Loading será distinto de hydration.

## DB-ORM-MANY-TO-MANY-081

Loader no ejecutará SQL directamente.

## DB-ORM-MANY-TO-MANY-082

Bidirectional synchronization no forzará inverse initialization.

## DB-ORM-MANY-TO-MANY-083

Inverse mutations podrán registrarse como deltas.

## DB-ORM-MANY-TO-MANY-084

Bidirectional synchronization será recursion-safe.

## DB-ORM-MANY-TO-MANY-085

Owning side permanecerá persistence authority.

## DB-ORM-MANY-TO-MANY-086

Inverse-only mutation tendrá policy explícita.

## DB-ORM-MANY-TO-MANY-087

Membership no recibirá entity lifecycle por default.

## DB-ORM-MANY-TO-MANY-088

Relationship event success será distinto de transaction commit.

## DB-ORM-MANY-TO-MANY-089

External irreversible effects usarán after-commit semantics.

## DB-ORM-MANY-TO-MANY-090

Batching no destruirá membership semantics.

## DB-ORM-MANY-TO-MANY-091

Batch sizing respetará platform/resource limits.

## DB-ORM-MANY-TO-MANY-092

UNKNOWN batch outcome será first-class.

## DB-ORM-MANY-TO-MANY-093

UNKNOWN no fabricará successful snapshot.

## DB-ORM-MANY-TO-MANY-094

Statement success será distinto de commit.

## DB-ORM-MANY-TO-MANY-095

Flush success será distinto de durable membership.

## DB-ORM-MANY-TO-MANY-096

Rollback no rebobinará collection object graph automáticamente.

## DB-ORM-MANY-TO-MANY-097

UNKNOWN commit no será convertido a committed.

## DB-ORM-MANY-TO-MANY-098

Retry respetará idempotency.

## DB-ORM-MANY-TO-MANY-099

Tenant-local memberships preservarán tenant namespace.

## DB-ORM-MANY-TO-MANY-100

Cross-tenant membership será rechazado por default.

## DB-ORM-MANY-TO-MANY-101

Cross-shard membership será rechazado por default.

## DB-ORM-MANY-TO-MANY-102

Distributed many-to-many no heredará garantías locales implícitamente.

## DB-ORM-MANY-TO-MANY-103

Relationship membership será distinto de authorization.

## DB-ORM-MANY-TO-MANY-104

HTTP role IDs no se aplicarán sin capas superiores de seguridad.

## DB-ORM-MANY-TO-MANY-105

Runtime table/column names provendrán de metadata confiable.

## DB-ORM-MANY-TO-MANY-106

Mapping será validado durante bootstrap.

## DB-ORM-MANY-TO-MANY-107

mappedBy/inversedBy deberán ser compatibles.

## DB-ORM-MANY-TO-MANY-108

Owning side será única autoridad física del join mapping bidireccional.

## DB-ORM-MANY-TO-MANY-109

Composite identities serán soportadas explícitamente.

## DB-ORM-MANY-TO-MANY-110

Partial composite membership identity será inválida.

## DB-ORM-MANY-TO-MANY-111

Join values usarán Database Type System.

## DB-ORM-MANY-TO-MANY-112

Join column mapping no dependerá de reflection order.

## DB-ORM-MANY-TO-MANY-113

SET semantics recomendará DB uniqueness.

## DB-ORM-MANY-TO-MANY-114

Schema diagnostics serán distintos de mapping validation.

## DB-ORM-MANY-TO-MANY-115

Hydration assembly no generará pending domain adds.

## DB-ORM-MANY-TO-MANY-116

Collection mutation mode será explícito.

## DB-ORM-MANY-TO-MANY-117

Dirty refresh será policy-driven.

## DB-ORM-MANY-TO-MANY-118

Normal requery será distinto de refresh.

## DB-ORM-MANY-TO-MANY-119

Partial load no sobrescribirá complete collection con subset.

## DB-ORM-MANY-TO-MANY-120

Partial hydration podrá enriquecer known membership sin fabricar completeness.

## DB-ORM-MANY-TO-MANY-121

COMPLETE podrá volverse STALE.

## DB-ORM-MANY-TO-MANY-122

External join-table mutations podrán invalidar relationship knowledge.

## DB-ORM-MANY-TO-MANY-123

Managed relationship state será distinto de second-level cache.

## DB-ORM-MANY-TO-MANY-124

Serialization no inicializará collections por default.

## DB-ORM-MANY-TO-MANY-125

Serialization cycles pertenecerán a Serialization System.

## DB-ORM-MANY-TO-MANY-126

N+1 será observable.

## DB-ORM-MANY-TO-MANY-127

ManyToManyMetadata será shareable solo si immutable.

## DB-ORM-MANY-TO-MANY-128

PersistentCollection será scope-local.

## DB-ORM-MANY-TO-MANY-129

Membership snapshots serán scope-local.

## DB-ORM-MANY-TO-MANY-130

Batch queues serán scope-local.

## DB-ORM-MANY-TO-MANY-131

Persistent workers no conservarán entity collections entre requests.

## DB-ORM-MANY-TO-MANY-132

Concurrent initialization será coordinada.

## DB-ORM-MANY-TO-MANY-133

Recursive initialization será detectada.

## DB-ORM-MANY-TO-MANY-134

Cancelled initialization no marcará COMPLETE.

## DB-ORM-MANY-TO-MANY-135

Partial cancelled hydration conservará partial knowledge explícito.

## DB-ORM-MANY-TO-MANY-136

Resource governance limitará pathological relationship loads.

## DB-ORM-MANY-TO-MANY-137

Planner podrá detectar join explosion.

## DB-ORM-MANY-TO-MANY-138

Multiple to-many fetch joins no se asumirán baratos.

## DB-ORM-MANY-TO-MANY-139

Streaming graph hydration requerirá safe group boundaries.

## DB-ORM-MANY-TO-MANY-140

Streaming rows no implicará bounded IdentityMap memory.

## DB-ORM-MANY-TO-MANY-141

Membership telemetry no expondrá identifiers sensibles.

## DB-ORM-MANY-TO-MANY-142

Collection diagnostics preservarán completeness.

## DB-ORM-MANY-TO-MANY-143

Collection diagnostics preservarán pending deltas.

## DB-ORM-MANY-TO-MANY-144

Unknown membership no será convertido a absent.

## DB-ORM-MANY-TO-MANY-145

Unknown persistence outcome no será convertido a success.

## DB-ORM-MANY-TO-MANY-146

Unknown inverse state no será convertido a inconsistency.

## DB-ORM-MANY-TO-MANY-147

Known bidirectional contradiction podrá ser inconsistency.

## DB-ORM-MANY-TO-MANY-148

Membership mutations serán estabilizadas antes de plan freeze.

## DB-ORM-MANY-TO-MANY-149

Lifecycle mutation previa al freeze será recollected.

## DB-ORM-MANY-TO-MANY-150

Post-persistence lifecycle mutation será future work.

## DB-ORM-MANY-TO-MANY-151

Frozen membership plan será immutable.

## DB-ORM-MANY-TO-MANY-152

Persistence Planner será distinto de Relationship Change Tracker.

## DB-ORM-MANY-TO-MANY-153

Relationship Change Tracker será distinto de Query Engine.

## DB-ORM-MANY-TO-MANY-154

Query Engine no conocerá PersistentCollection runtime state.

## DB-ORM-MANY-TO-MANY-155

Compiler no conocerá Entity objects.

## DB-ORM-MANY-TO-MANY-156

Executor no interpretará membership semantics.

## DB-ORM-MANY-TO-MANY-157

Connection no conocerá relationship mappings.

## DB-ORM-MANY-TO-MANY-158

Driver no conocerá ORM relationships.

## DB-ORM-MANY-TO-MANY-159

Active Record y Data Mapper utilizarán el mismo membership engine.

## DB-ORM-MANY-TO-MANY-160

VoltStack preservará separadamente membership, materialization, collection completeness, persistence intent y database certainty.

---

# 284. Fórmulas fundamentales

## 284.1 Membership

```text
Membership(A, B)
=
(EntityKey(A), EntityKey(B))
```

---

# 285. Membership Set

Para source `S`:

```text
Members(S)
=
{ T | Membership(S, T) }
```

---

# 286. SET Semantics

```text
∀ T:
Count(Membership(S, T)) ≤ 1
```

---

# 287. Add

```text
CurrentDesired
=
PreviousDesired ∪ {Target}
```

---

# 288. Remove

```text
CurrentDesired
=
PreviousDesired - {Target}
```

---

# 289. Complete Collection Diff

Cuando baseline es COMPLETE:

```text
Added
=
Current - Baseline
```

```text
Removed
=
Baseline - Current
```

---

# 290. Partial Collection Rule

```text
Completeness ≠ COMPLETE
⇒
AbsenceInMemory
≠
KnownMembershipAbsence
```

---

# 291. Pending Mutation Rule

```text
UninitializedCollection
+
ExplicitMutation
⇒
TrackDelta
```

no:

```text
InitializeEverything
```

---

# 292. Identity Dependency

```text
InsertMembership(S, T)
requires
EstablishedIdentity(S)
∧
EstablishedIdentity(T)
```

---

# 293. Hydration Dedup

```text
SameMembershipKey
⇒
OneLogicalMembership
```

---

# 294. Canonical Target

```text
SameTargetEntityKey
⇒
SameManagedTargetInstance
```

dentro del PersistenceContext.

---

# 295. Known Empty Collection

```text
KnownEmpty
=
ObservedRelationshipUniverse
∧
NoMembershipRows
∧
CoverageComplete
```

---

# 296. Extra-Lazy Safety

```text
ExtraLazyOperation
⇒
NoImplicitFullInitialization
```

salvo que la semántica solicitada lo requiera.

---

# 297. Snapshot Advancement

```text
AdvanceMembershipBaseline
⇒
PersistenceOutcomeSufficientlyCertain
```

---

# 298. Safe Streaming

```text
SafeYield(Source)
=
RequiredRelationshipGraphComplete
∧
NoFutureRowCanExtendSourceCollections
```

---

# 299. Runtime Safety

```text
SafeManyToManyRuntime
=
ImmutableSharedMetadata
∧
ScopedCollections
∧
ScopedSnapshots
∧
ScopedMembershipDeltas
∧
ScopedAssemblyState
∧
ScopedIdentityMap
∧
DeterministicReset
```

---

# 300. Master Formula

```text
Database Many-to-Many Relationship System
=
ManyToMany Metadata
+
Owning/Inverse Semantics
+
Join Table Mapping
+
Join Column Mapping
+
Collection Semantics
+
PersistentCollection
+
Membership Identity
+
Collection State
+
Collection Completeness
+
Lazy Loading
+
Eager Loading
+
Batch Loading
+
JOIN Hydration
+
Membership Deduplication
+
IdentityMap Canonicalization
+
Membership Snapshots
+
Delta Change Tracking
+
Add/Remove/Clear
+
Attach/Detach/Sync
+
Extra-Lazy Operations
+
Large Collection Strategies
+
Cascade Persist
+
Generated Identity Barriers
+
Membership Persistence Operations
+
Duplicate Prevention
+
Concurrency Handling
+
Inverse Synchronization
+
Association Entity Boundary
+
Transaction Awareness
+
Persistence Certainty
+
Tenant/Shard Isolation
+
Streaming Safety
+
Resource Governance
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

# 301. Regla maestra final

> **VoltStack nunca confundirá una colección no inicializada con una colección vacía, una entidad ausente en memoria con una membresía inexistente, una fila de unión con una entidad de dominio, ni un cambio de colección con una operación SQL inmediata.**

La arquitectura preservará explícitamente:

```text
Membership Identity
≠
Target Entity Identity
≠
Join Row
≠
PersistentCollection
≠
Collection Completeness
≠
Pending Mutation
≠
Database State
≠
Persistence Outcome
```

y mantendrá como principio:

```text
Unknown Membership
≠
Absent Membership
```

```text
UNINITIALIZED
≠
EMPTY
```

```text
ADD
≠
INSERT
```

```text
REMOVE
≠
DELETE TARGET
```

```text
flush success
≠
transaction commit
```

---

# 302. Arquitectura resultante

```text
                       ManyToManyMetadata
                              │
            ┌─────────────────┼──────────────────┐
            │                 │                  │
            ▼                 ▼                  ▼
       Collection API      Loading           Query Navigation
            │                 │                  │
            ▼                 ▼                  ▼
   PersistentCollection   Entity Query      Semantic Query
            │                 │                  │
     ┌──────┼──────┐          ▼                  ▼
     │      │      │       Hydration          Query Engine
     ▼      ▼      ▼          │
 UNINIT  PARTIAL COMPLETE     ▼
     │      │      │      IdentityMap
     └──────┼──────┘          │
            ▼                 ▼
      Membership State ← Canonical Targets
            │
     ┌──────┼────────┐
     │      │        │
     ▼      ▼        ▼
   ADD    REMOVE    CLEAR
     │      │        │
     └──────┼────────┘
            ▼
      Membership ChangeSet
            │
            ▼
         UnitOfWork
            │
            ▼
    Persistence Planner
            │
      ┌─────┴──────┐
      ▼            ▼
 Entity Identity   Membership
 Dependencies      Operations
      │            │
      └─────┬──────┘
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
         Executor
```

La separación arquitectónica final queda:

```text
Relationship
    ↓
Membership Semantics
    ↓
UnitOfWork
    ↓
Persistence Planner
    ↓
Query Model
    ↓
Query Engine
    ↓
SQL Compiler
    ↓
Execution Engine
    ↓
Connection
    ↓
Driver
```

Nunca en sentido inverso.

---

# 303. Siguiente documento

```text
147_DATABASE_POLYMORPHIC_RELATIONSHIP_SYSTEM.md
```

El siguiente documento definirá la arquitectura de relaciones polimórficas, incluyendo:

```text
morph-to
morph-one
morph-many
polymorphic many-to-many
type discriminators
stable morph aliases
target type resolution
polymorphic EntityKey
polymorphic references
IdentityMap integration
type-safe metadata
query resolution
hydration
lazy/eager/batch loading
persistence
security
tenant isolation
schema implications
migration compatibility
renaming safety
unknown discriminator handling
extension registration
persistent runtime
telemetry
testing
```

con una regla central:

> **El tipo persistido de una relación polimórfica será un identificador lógico estable y controlado por metadata, nunca un nombre de clase PHP arbitrario obtenido directamente desde la base de datos.**