# 143_DATABASE_ONE_TO_ONE_RELATIONSHIP_SYSTEM.md

# VoltStack Quantum Database
## Database One-to-One Relationship System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 143 — Database One-to-One Relationship System  
**Bloque:** 13 — Relationships  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database One-to-One Relationship System` define la arquitectura mediante la cual VoltStack representa, carga, hidrata, rastrea y persiste asociaciones ORM donde una entidad puede relacionarse con **como máximo una entidad del otro lado**.

Ejemplos:

```text
User 1 ───── 0..1 Profile

Person 1 ─── 0..1 Passport

Account 1 ── 1 Settings

Order 1 ──── 0..1 Invoice
```

El sistema deberá soportar:

- one-to-one unidireccional;
- one-to-one bidireccional;
- owning side;
- inverse side;
- relaciones opcionales;
- relaciones obligatorias;
- foreign key única;
- shared primary key;
- referencias lazy;
- eager loading;
- explicit loading;
- hydration fix-up;
- replacement;
- nullification;
- cascade persist;
- cascade remove;
- orphan removal;
- generated identifiers;
- optimistic locking;
- relationship snapshots;
- persistent runtime;
- tenant/shard isolation.

---

# 2. Principio central

> **Una relación one-to-one de VoltStack representa como máximo una identidad relacionada por cada lado semántico; la unicidad ORM deberá preservarse independientemente de si físicamente se implementa mediante foreign key única, primary key compartida u otra estrategia compatible.**

Formalmente:

```text
∀ A ∈ EntityTypeA:

|relationship(A)| ≤ 1
```

y, cuando la asociación sea realmente uno-a-uno en ambas direcciones:

```text
∀ B ∈ EntityTypeB:

|inverseRelationship(B)| ≤ 1
```

---

# 3. One-to-One ≠ Foreign Key

La relación:

```text
User.profile
```

es semántica ORM.

Su representación física puede ser:

```text
profiles.user_id UNIQUE
```

o:

```text
profiles.id
=
users.id
```

o alguna estrategia especializada.

Por tanto:

```text
OneToOneRelationship
≠
UniqueForeignKey
```

aunque una FK única sea una implementación común.

---

# 4. One-to-One ≠ One-to-Many con límite

No deberá modelarse conceptualmente como:

```text
User
 └── Profiles[]
     LIMIT 1
```

Una relación one-to-one posee semántica propia:

```text
User.profile
→ Profile | null
```

No una colección.

---

# 5. Posición arquitectónica

```text
                    Relationship Metadata
                             │
                             ▼
                    OneToOneMetadata
                             │
             ┌───────────────┼────────────────┐
             │               │                │
             ▼               ▼                ▼
      Entity Query       Hydration        UnitOfWork
             │               │                │
             ▼               ▼                ▼
       To-One Loader     To-One Fix-Up   To-One Snapshot
             │               │                │
             ▼               ▼                ▼
        Query Engine      IdentityMap     Change Tracking
                                              │
                                              ▼
                                  Relationship Persistence
                                              │
                                              ▼
                                     Persistence Planner
```

---

# 6. Cardinalidad

La cardinalidad conceptual puede ser:

```text
1 → 0..1
```

o:

```text
1 → 1
```

dependiendo del mapping.

Ejemplo opcional:

```php
private ?Profile $profile = null;
```

Ejemplo requerido:

```php
private Profile $profile;
```

---

# 7. Required ≠ Database NOT NULL

Si ORM declara:

```text
User.profile required
```

eso no prueba automáticamente que la base tenga:

```text
NOT NULL
```

ni siquiera que la FK esté físicamente en `users`.

La compatibilidad de Schema deberá verificarse por separado.

---

# 8. Dirección

Una relación one-to-one puede ser:

```text
UNIDIRECTIONAL
```

o:

```text
BIDIRECTIONAL
```

---

# 9. Unidirectional One-to-One

Ejemplo:

```text
User
 └── Profile
```

sin:

```text
Profile.user
```

La asociación solo es navegable desde `User`.

---

# 10. Bidirectional One-to-One

```text
User.profile
↔
Profile.user
```

Ambas propiedades representan la misma asociación conceptual.

---

# 11. Owning Side

El owning side será el lado cuya información determina la persistencia física de la asociación.

Ejemplo:

```text
profiles.user_id
```

Entonces típicamente:

```text
Profile.user
=
OWNING
```

y:

```text
User.profile
=
INVERSE
```

---

# 12. Ownership ≠ Direction

Una relación puede ser unidireccional y owning.

También puede ser bidireccional y tener exactamente un owning side.

---

# 13. Ownership ≠ Domain Ownership

No confundir:

```text
ORM owning side
```

con:

```text
DDD aggregate ownership
```

ni con:

```text
business ownership
```

---

# 14. Exactly One Persistence Authority

Para una asociación bidireccional normal:

```text
OwningSides = 1
```

No:

```text
OwningSides = 2
```

porque generaría ambigüedad sobre qué lado determina la relación persistida.

---

# 15. Inverse Side

El inverse side representa la asociación desde la dirección opuesta.

Ejemplo:

```text
User.profile
```

puede derivarse conceptualmente de:

```text
Profile.user
```

cuando `Profile.user` es owning.

---

# 16. OneToOneMetadata

Propuesta:

```php
final readonly class OneToOneMetadata
{
    public function __construct(
        public RelationshipId $id,
        public EntityType $sourceType,
        public EntityType $targetType,
        public string $property,
        public RelationshipDirection $direction,
        public RelationshipOwnership $ownership,
        public RelationshipOptionality $optionality,
        public OneToOneStorageStrategy $storageStrategy,
        public RelationshipFetchMode $fetchMode,
        public CascadePolicy $cascade,
        public OrphanRemovalPolicy $orphanRemoval,
    ) {}
}
```

---

# 17. Storage Strategy

Propuesta:

```php
enum OneToOneStorageStrategy
{
    case UNIQUE_FOREIGN_KEY;
    case SHARED_PRIMARY_KEY;
    case CUSTOM;
}
```

La estrategia física deberá permanecer separada de la semántica ORM.

---

# 18. Unique Foreign Key

Ejemplo:

```text
users
─────
id

profiles
────────
id
user_id UNIQUE NOT NULL
```

Relación:

```text
User 1 ───── 1 Profile
```

---

# 19. Optional Unique Foreign Key

```text
profiles.user_id UNIQUE NULL
```

puede representar una relación opcional.

Pero la semántica exacta de múltiples `NULL` depende de plataforma.

Por ello:

```text
Platform Capability
```

deberá gobernar compatibilidad física.

---

# 20. No Vendor Conditionals

No:

```php
if ($driver === 'mysql') {
}
```

Preferir:

```php
$platform->capabilities()
    ->uniqueNullableConstraintSemantics();
```

---

# 21. Shared Primary Key

Ejemplo:

```text
users
────────
id PK

profiles
────────
user_id PK FK → users.id
```

Entonces:

```text
Profile identity
```

puede derivarse directamente de:

```text
User identity
```

---

# 22. Shared PK semantics

Formalmente:

```text
ProfileId = UserId
```

dentro de esa estrategia.

Pero:

```text
Profile
≠
User
```

Son tipos de entidad distintos aunque compartan valor identificador.

---

# 23. Identity Namespace

Debe preservarse:

```text
EntityKey(User, 42)
≠
EntityKey(Profile, 42)
```

aunque ambos utilicen `42`.

---

# 24. Derived Identity

Shared primary key puede implicar:

```text
DependentIdentity
=
PrincipalIdentity
```

La asignación deberá ser controlada por el Persistence Planner.

---

# 25. Principal / Dependent

Para ciertas relaciones one-to-one será útil distinguir:

```text
PRINCIPAL
DEPENDENT
```

Esto es diferente de:

```text
OWNING
INVERSE
```

aunque frecuentemente estén relacionados.

---

# 26. Principal ≠ Owning Side

Una entidad principal puede no ser el owning side de navegación ORM.

Los conceptos deberán mantenerse separados.

---

# 27. Optionality

Propuesta:

```php
enum RelationshipOptionality
{
    case OPTIONAL;
    case REQUIRED;
}
```

---

# 28. OPTIONAL

```text
User.profile
=
Profile | null
```

---

# 29. REQUIRED

Conceptualmente:

```text
Account.settings
=
Settings
```

pero durante construcción/hydration/persistence pueden existir estados transitorios internos.

---

# 30. Required Relation ≠ Always Initialized

Una relación required puede ser lazy.

Entonces:

```text
required
```

no significa:

```text
currently initialized
```

---

# 31. Runtime To-One State

Debe distinguirse:

```php
enum ToOneRelationshipState
{
    case UNINITIALIZED;
    case REFERENCE;
    case NULL;
    case ENTITY;
    case INITIALIZING;
}
```

---

# 32. UNINITIALIZED

No existe conocimiento runtime suficiente sobre el valor actual.

---

# 33. NULL

El ORM conoce que:

```text
related entity = absent
```

según la observación/mutación actual.

---

# 34. REFERENCE

Existe una identidad relacionada conocida:

```text
EntityKey(Profile, 10)
```

pero no necesariamente estado completo de `Profile`.

---

# 35. ENTITY

Existe instancia canónica materializada.

---

# 36. NULL ≠ UNINITIALIZED

Regla fundamental:

```text
NULL
≠
UNINITIALIZED
```

---

# 37. REFERENCE ≠ ENTITY

Igualmente:

```text
KnownIdentity
≠
LoadedState
```

---

# 38. One-to-One Reference

Propuesta:

```php
interface ToOneReference
{
    public function relationship(): RelationshipId;

    public function targetKey(): EntityKey;

    public function isInitialized(): bool;

    public function get(): object;
}
```

---

# 39. Reference does not own EntityManager

No deberá almacenar un EntityManager global.

El loader/context será resuelto de forma scoped.

---

# 40. IdentityMap Integration

Si:

```text
Profile#10
```

ya está managed:

```text
User.profile
```

deberá resolver al mismo objeto.

Formalmente:

```text
IdentityMapHit(Profile#10)
⇒
ReuseCanonicalProfile
```

---

# 41. No Duplicate Entity

Nunca:

```text
$user->profile === $profileA

IdentityMap(Profile#10) === $profileB

$profileA !== $profileB
```

dentro del mismo PersistenceContext.

---

# 42. Lazy Loading

Una relación one-to-one podrá ser lazy.

Ejemplo conceptual:

```php
$user->profile;
```

puede provocar resolución si aún está:

```text
UNINITIALIZED
```

---

# 43. Lazy Loading ≠ Query necesariamente

Si la identidad relacionada ya está disponible y la entidad está en IdentityMap:

```text
lazy resolution
→ IdentityMap hit
→ zero DB query
```

---

# 44. Lazy Loading Pipeline

```text
Access relationship
        │
        ▼
ToOne State
        │
        ├── NULL ────────→ return null
        │
        ├── ENTITY ──────→ return canonical entity
        │
        ├── REFERENCE ───→ IdentityMap / Loader
        │
        └── UNINITIALIZED
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
             Hydration
                 │
                 ▼
            IdentityMap
```

---

# 45. Lazy Loading Guard

Strict mode podrá impedir:

```text
unexpected lazy loading
```

por ejemplo durante:

```text
serialization
loops
API responses
critical paths
```

---

# 46. Eager Loading

Una one-to-one podrá declararse eager.

Eso no obliga:

```text
JOIN
```

Puede utilizar:

```text
JOIN
```

o:

```text
secondary SELECT
```

o:

```text
batch SELECT
```

---

# 47. Join Hydration

Ejemplo físico:

```text
User#1 | Profile#10
User#2 | Profile#11
User#3 | NULL
```

Hydration deberá producir:

```text
User#1.profile → Profile#10
User#2.profile → Profile#11
User#3.profile → null
```

---

# 48. Optional Join Absence

Si todas las columnas necesarias para la identidad target son `NULL`:

```text
TargetAbsent
```

No deberá crearse:

```text
Profile(id: null)
```

---

# 49. Formula

```text
OptionalTargetAbsent
⇔
AllRequiredTargetIdentityColumnsAreNull
```

según mapping.

---

# 50. Partial Composite Identifier

Si una identidad compuesta produce:

```text
part1 = 10
part2 = NULL
```

cuando ambos son requeridos:

```text
PartialIdentifierHydrationException
```

No deberá interpretarse automáticamente como ausencia.

---

# 51. Hydration Fix-Up

En asociación bidireccional:

```text
User.profile
↔
Profile.user
```

si hydration materializa ambos:

```text
$user->profile = $profile
```

y:

```text
$profile->user = $user
```

deben quedar coherentes.

---

# 52. Fix-Up ≠ User Mutation

La sincronización realizada por hydration:

```text
must not mark relationship dirty
```

---

# 53. Fix-Up Loop Prevention

No:

```text
set User.profile
→ set Profile.user
→ set User.profile
→ ...
```

Se utilizará un contexto interno/ledger.

---

# 54. Relationship Fix-Up Ledger

Conceptualmente:

```text
RelationshipEdge
+
HydrationSession
```

permite detectar asociaciones ya sincronizadas.

---

# 55. Baseline

Después de hydration:

```text
baseline User.profile = Profile#10
current  User.profile = Profile#10
```

Entonces:

```text
dirty = false
```

---

# 56. One-to-One Snapshot

Propuesta:

```php
final readonly class ToOneRelationshipSnapshot
{
    public function __construct(
        public RelationshipId $relationship,
        public EntityKey $owner,
        public ?EntityKey $related,
        public RelationshipKnowledge $knowledge,
    ) {}
}
```

---

# 57. RelationshipKnowledge

Puede distinguir:

```text
KNOWN_NULL
KNOWN_ENTITY
UNOBSERVED
```

No usar:

```text
related = null
```

para representar simultáneamente NULL y desconocido.

---

# 58. Snapshot formula

```text
Snapshot
=
ObservedRelatedIdentity
+
KnowledgeState
```

---

# 59. Change Detection

Ejemplo:

```text
baseline = Profile#10
current  = Profile#11
```

produce:

```text
REPLACED
```

---

# 60. Change Types

Propuesta:

```php
enum ToOneChangeKind
{
    case NONE;
    case ASSIGNED;
    case REPLACED;
    case NULLIFIED;
}
```

---

# 61. ASSIGNED

```text
baseline = null
current  = Profile#10
```

---

# 62. REPLACED

```text
baseline = Profile#10
current  = Profile#11
```

---

# 63. NULLIFIED

```text
baseline = Profile#10
current  = null
```

---

# 64. NONE

```text
baseline = Profile#10
current  = Profile#10
```

---

# 65. Unobserved Baseline

Si:

```text
baseline = UNOBSERVED
```

el ORM no deberá fabricar un cambio destructivo simplemente comparándolo con `null`.

---

# 66. Safe Change Detection

```text
SafeToOneDiff
=
KnownBaseline
∧
KnownCurrentState
```

o evidencia semántica equivalente derivada de una mutación explícita.

---

# 67. Explicit Mutation Intent

Incluso con relación lazy no inicializada:

```php
$user->setProfile(null);
```

es una intención explícita.

El ORM puede registrar:

```text
ExplicitNullificationIntent
```

sin necesitar cargar primero el Profile.

---

# 68. No Forced Read Before Write

Regla importante:

```text
ExplicitRelationshipMutation
⇏
MandatoryRelationshipLoad
```

si la operación puede persistirse con seguridad usando identidad/metadata disponible.

---

# 69. Mutation Intent ≠ Baseline

Sin embargo, el sistema deberá conservar que:

```text
explicitly changed
```

no es lo mismo que:

```text
full prior state observed
```

Esto importa para orphan removal.

---

# 70. Relationship ChangeSet

Propuesta:

```php
final readonly class ToOneRelationshipChangeSet
{
    public function __construct(
        public RelationshipId $relationship,
        public EntityKey $owner,
        public ToOneChangeKind $kind,
        public ?EntityKey $before,
        public ?EntityKey $after,
        public RelationshipChangeEvidence $evidence,
    ) {}
}
```

---

# 71. Relationship Persistence

Un ChangeSet no genera SQL directamente.

Pipeline:

```text
ToOne ChangeSet
      │
      ▼
Relationship Persistence System
      │
      ▼
Semantic Persistence Operation
      │
      ▼
Persistence Planner
      │
      ▼
Query Model
      │
      ▼
Query Engine
```

---

# 72. Assignment Operation

Ejemplo:

```text
User.profile
null → Profile#10
```

podría convertirse en:

```text
AssociateOneToOne
```

---

# 73. Replacement Operation

```text
Profile#10 → Profile#11
```

puede requerir:

```text
dissociate old
associate new
```

o una única FK update.

La decisión física pertenece al Persistence Planner.

---

# 74. Nullification Operation

```text
Profile#10 → null
```

puede requerir:

```text
UPDATE FK = NULL
```

o:

```text
DELETE dependent
```

si orphan removal aplica.

---

# 75. Nullification ≠ Delete

Sin orphan removal:

```text
relationship = null
```

no significa automáticamente:

```text
DELETE related entity
```

---

# 76. Orphan Removal

Con:

```text
orphanRemoval = true
```

la eliminación de la relación puede programar eliminación del dependent.

---

# 77. Safe Orphan Removal

Debe existir evidencia suficiente de:

```text
old related entity
```

antes de programar su eliminación.

---

# 78. Formula

```text
ScheduleOrphanRemoval
=
OrphanRemovalEnabled
∧
PreviousRelatedIdentityKnown
∧
RelationshipNoLongerReferencesPrevious
∧
OwnershipRulesSatisfied
```

---

# 79. Unknown Old Identity

Si el ORM no conoce qué entidad estaba relacionada:

```text
do not invent orphan
```

Puede:

- requerir lectura;
- usar operación DB segura que obtenga evidencia;
- rechazar;
- aplicar policy especializada.

---

# 80. Cascade Persist

Ejemplo:

```php
$user->setProfile(new Profile());
$entityManager->persist($user);
```

con:

```text
cascade persist = true
```

puede registrar `Profile` como NEW.

---

# 81. Without Cascade Persist

Si target es NEW:

```text
flush
→ RelationshipCascadeException
```

o error de entidad transitoria equivalente.

No deberá insertarse silenciosamente.

---

# 82. Cascade Remove

```php
$entityManager->remove($user);
```

puede programar:

```text
remove Profile
```

si mapping lo declara.

---

# 83. Cascade Remove ≠ Orphan Removal

Reiteración:

```text
remove(User)
→ remove(Profile)
```

es cascade remove.

```text
User.profile = null
→ remove(Profile)
```

es orphan removal.

---

# 84. Database ON DELETE CASCADE

Si DB posee:

```text
ON DELETE CASCADE
```

el ORM puede aprovechar esa capability/configuración.

Pero deberá reconciliar correctamente su PersistenceContext.

---

# 85. ORM Cascade vs DB Cascade

Nunca asumir:

```text
DB deleted row
⇒
ORM automatically knows every affected entity
```

Se requiere metadata/reconciliation.

---

# 86. Generated Identifiers

Caso:

```text
new User
new Profile
Profile.user = User
```

si `User.id` es generado:

```text
INSERT User
↓
obtain UserId
↓
INSERT Profile with UserId
```

---

# 87. Generated ID Barrier

Persistence Plan deberá incluir dependencia:

```text
Insert(User)
      │
      ▼
GeneratedIdentity(User)
      │
      ▼
Insert(Profile)
```

---

# 88. Shared PK Barrier

En shared primary key:

```text
User.id generated
```

deberá resolverse antes de:

```text
Profile.id = User.id
```

---

# 89. No Temporary ID Leakage

Una identidad temporal de UnitOfWork no deberá persistirse accidentalmente como PK/FK real.

---

# 90. Flush Ordering

Dependiendo de mapping:

```text
principal insert
→ dependent insert
```

para creación.

En eliminación puede requerirse:

```text
dependent delete
→ principal delete
```

---

# 91. Database Cascade Optimization

Si plataforma/schema garantiza cascade, el planner puede reducir operaciones.

Pero:

```text
Optimization
```

no deberá alterar semántica ORM observable.

---

# 92. Replacement Ordering

Con constraint `UNIQUE`, reemplazar:

```text
User#1.profile:
Profile#10 → Profile#11
```

puede requerir orden específico para evitar violación temporal de unicidad.

---

# 93. Example

Si `Profile#11` actualmente pertenece a otro owner, el sistema no deberá simplemente reasignarlo sin validar semantics.

---

# 94. One-to-One Uniqueness

La asociación exige:

```text
AtMostOneOwnerPerTarget
```

cuando mapping lo establece.

---

# 95. Identity-Level Uniqueness

Dentro del PersistenceContext:

```text
User#1.profile = Profile#10
User#2.profile = Profile#10
```

puede ser una violación semántica si la relación es one-to-one estricta.

---

# 96. In-Memory Uniqueness Validation

El UoW podrá detectar conflictos conocidos antes de DB execution.

---

# 97. Database Remains Authority

La validación in-memory no reemplaza:

```text
UNIQUE constraint
```

cuando se requiere integridad concurrente.

---

# 98. Concurrency

Dos transacciones pueden intentar:

```text
User#1 → Profile#10
User#2 → Profile#10
```

simultáneamente.

Solo el object graph local no puede prevenirlo globalmente.

---

# 99. Database Constraint Recommendation

Cuando sea posible:

```text
one-to-one
→ database uniqueness enforcement
```

además de invariantes ORM.

---

# 100. Optimistic Locking

Si cambiar la relación debe formar parte de la versión de una entidad:

```text
version predicate
```

puede integrarse con Update Persistence.

La policy deberá ser explícita.

---

# 101. Relationship Change and Versioning

No toda asociación requiere incrementar automáticamente la versión del owner.

Mapping/persistence policy deberá decidirlo.

---

# 102. Pessimistic Locking

Operaciones críticas de reasignación podrán utilizar locking mediante Transaction/Concurrency System.

Relationship System no implementará locks directamente.

---

# 103. Refresh

`refresh(User)` deberá definir qué ocurre con:

```text
User.profile
```

---

# 104. Refresh Policy

Posibles políticas:

```text
REFRESH_LOADED_RELATIONS
INVALIDATE_RELATIONS
PRESERVE_UNLOADED
EXPLICIT_RELATION_REFRESH
```

---

# 105. Recommended Default

Refresh de entidad no debería provocar una explosión de queries sobre todo el graph.

Preferir:

```text
refresh entity scalar state
+
invalidate/reconcile relation state according metadata
```

sin cargar relaciones no solicitadas.

---

# 106. Dirty Relationship Refresh

Si:

```text
User.profile
```

está dirty, refresh deberá seguir una policy explícita:

```text
REJECT
DISCARD_LOCAL_CHANGE
MERGE_IF_SAFE
```

Default seguro:

```text
REJECT
```

salvo intención explícita.

---

# 107. Refresh ≠ Hydration Normal

Una query normal que encuentra un `User` ya managed no deberá sobrescribir ciegamente una relación dirty.

---

# 108. Normal Hydration Policy

```text
IdentityMapHit
+
ExistingRelationshipDirty
⇒
PreserveManagedRelationship
```

por defecto.

---

# 109. Explicit Refresh

Solo una operación explícita de refresh puede reemplazar estado managed conforme a policy.

---

# 110. Partial Entity Hydration

Una entidad parcialmente hidratada puede tener la relación:

```text
UNINITIALIZED
```

aunque el property PHP tenga algún valor por construcción.

El estado ORM externo será autoridad.

---

# 111. PHP Property Value ≠ ORM Knowledge

Especialmente para partial entities:

```text
PropertyDefault
≠
ObservedRelationshipState
```

---

# 112. Partial Query

Ejemplo:

```text
SELECT user.id, user.name
```

sin datos de `profile`.

No deberá inferirse:

```text
User.profile = null
```

---

# 113. Projection

Una DTO/projection que incluya:

```text
userId
profileId
```

no necesariamente inicializa la relación managed `User.profile`.

---

# 114. Query Result ≠ Relationship Initialization

Solo un HydrationPlan que declare explícitamente ensamblaje de la relación podrá actualizar su runtime state.

---

# 115. Join Filtering

Considérese:

```text
LEFT JOIN profile
  ON profile.user_id = user.id
 AND profile.active = true
```

Si no aparece Profile:

```text
NULL joined row
```

no demuestra necesariamente que:

```text
User.profile = null
```

en la relación completa.

---

# 116. Critical Rule

```text
FilteredJoinAbsence
≠
RelationshipAbsence
```

---

# 117. Completeness Semantics

Para to-one también se requiere conocimiento de cobertura.

Propuesta:

```php
enum ToOneRelationshipCompleteness
{
    case UNOBSERVED;
    case PARTIAL;
    case COMPLETE;
}
```

---

# 118. Why PARTIAL for To-One?

Aunque cardinalidad sea uno, una query filtrada puede observar solo:

```text
related entity satisfying condition
```

no la relación completa.

---

# 119. Safe Null

Solo podrá marcarse:

```text
KNOWN_NULL + COMPLETE
```

cuando la query/loader cubra toda la relación.

---

# 120. Formula

```text
SafeKnownNull
=
TargetNotFound
∧
LoadCoverageComplete
```

---

# 121. Explicit Lazy Loader

El loader normal de `User.profile` consulta la relación completa.

Por tanto, ausencia exitosa puede producir:

```text
KNOWN_NULL
+
COMPLETE
```

---

# 122. Filtered Relation Query

```php
$user->profile()
    ->where('active', true)
    ->first();
```

si retorna `null`, no modifica necesariamente el estado de:

```text
$user->profile
```

---

# 123. Relationship Query API

Debe diferenciar:

```text
query relationship
```

de:

```text
initialize relationship
```

---

# 124. Detached Entities

Asignar:

```text
detached Profile#10
```

a:

```text
managed User#1
```

no deberá asumir que Profile es NEW.

---

# 125. Default Detached Policy

Preferible:

```text
REJECT
```

hasta que el desarrollador:

- reobtenga entidad managed;
- utilice referencia canónica;
- ejecute operación explícita de attach/merge si VoltStack la soporta.

---

# 126. Cross PersistenceContext

No:

```text
User managed by Context A
Profile managed by Context B
```

en una relación administrada ordinaria.

---

# 127. Formula

```text
ManagedAssociation(A, B)
⇒
CompatiblePersistenceContext(A, B)
```

---

# 128. Tenant Isolation

Debe cumplirse:

```text
TenantNamespace(User)
=
TenantNamespace(Profile)
```

para relaciones tenant-local normales.

---

# 129. Cross-Tenant One-to-One

Solo se permitirá si existe mapping explícito que lo soporte.

No por accidente.

---

# 130. Shard Awareness

Misma regla para:

```text
ShardNamespace
```

---

# 131. Cross-Database Relationship

Una one-to-one puede existir lógicamente entre bases distintas.

En ese caso:

- DB FK quizá no exista;
- loading puede requerir otro connection route;
- atomic persistence puede no estar disponible;
- transaction semantics deberán ser explícitas.

---

# 132. Relationship System Does Not Invent Distributed Transactions

```text
CrossDatabaseRelationship
⇏
DistributedAtomicTransaction
```

---

# 133. Entity Removal

Si target está `REMOVED`, no deberá asignarse como relación activa sin policy explícita.

---

# 134. Owner Removal

Eliminar owner puede interactuar con:

```text
cascade remove
orphan removal
DB cascade
FK nullification
```

Persistence Planner decide operaciones.

---

# 135. New Owner + Existing Target

Ejemplo:

```text
new User
→ existing Profile#10
```

puede ser válido si mapping permite reassignment.

El planner deberá manejar generated owner ID.

---

# 136. Existing Owner + New Target

```text
existing User#1
→ new Profile
```

requiere:

```text
cascade persist
```

o `persist(Profile)` explícito.

---

# 137. New Owner + New Target

Puede requerir un dependency graph completo.

---

# 138. Replacement with Orphan Removal

```text
Profile#10 → Profile#11
```

con orphan removal:

```text
associate Profile#11
+
schedule Profile#10 removal
```

según ordering seguro.

---

# 139. Replacement and Unique Constraint

El planner debe considerar:

```text
UNIQUE FK
```

para evitar estados intermedios inválidos.

---

# 140. Relationship Persistence Operation

Propuesta:

```php
interface OneToOnePersistenceOperation
{
    public function relationship(): RelationshipId;

    public function owner(): EntityKey;
}
```

Especializaciones:

```text
AssociateOneToOneOperation
DissociateOneToOneOperation
ReplaceOneToOneOperation
ScheduleOneToOneOrphanRemoval
```

---

# 141. Operation ≠ Statement

Una operación puede producir:

```text
0
1
N
```

statements físicos dependiendo de estrategia/capabilities.

---

# 142. Shared PK Persistence

Shared PK puede no requerir `UPDATE FK`.

La asociación puede establecerse mediante la identidad del dependent durante INSERT.

---

# 143. Unique FK Persistence

Puede requerir:

```text
INSERT dependent with FK
```

o:

```text
UPDATE owning row SET fk = ?
```

---

# 144. Inverse-Side Mutation

Si solo cambia inverse side:

```text
$user->profile = $profile;
```

pero owning side permanece sin cambio, la política deberá evitar persistencia ambigua.

---

# 145. Strict Ownership Mode

Recomendado:

```text
inverse-only mutation
→ diagnostic/error during flush
```

si produce inconsistencia con owning side.

---

# 146. Optional Fix-Up Mode

Una API de dominio correctamente configurada puede sincronizar ambos lados.

Pero ORM no deberá adivinar arbitrariamente intención.

---

# 147. Relationship Graph Validation

Antes del plan:

```text
validate known bidirectional one-to-one edges
```

---

# 148. Example Violation

```text
User#1.profile = Profile#10
Profile#10.user = User#2
```

Debe producir diagnóstico.

---

# 149. Another Violation

```text
User#1.profile = Profile#10
User#2.profile = Profile#10
```

si ambas relaciones son conocidas y one-to-one:

```text
OneToOneUniquenessViolation
```

---

# 150. Unknown State

No deberá producir falsos positivos si:

```text
User#2.profile = UNINITIALIZED
```

---

# 151. Knowledge-Aware Validation

```text
KnownConflict
→ ERROR

UnknownPotentialConflict
→ no fabricated error
```

La DB constraint seguirá protegiendo concurrencia/global state.

---

# 152. Lifecycle

`postLoad` deberá ejecutarse conforme a Hydration Architecture una vez que la entidad y graph requerido estén suficientemente finalizados.

---

# 153. Relationship Mutation in postLoad

Si `postLoad` cambia:

```text
User.profile
```

después de baseline:

```text
relationship becomes dirty
```

para un flush futuro.

---

# 154. preUpdate

Si lifecycle modifica una one-to-one durante `preUpdate`/stabilization:

```text
RelationshipChangeSet
```

deberá recomputarse antes del plan freeze.

---

# 155. postUpdate

Una mutación posterior a ejecución:

```text
becomes future work
```

No genera recursive hidden update.

---

# 156. Recursive Flush

Sigue prohibido.

---

# 157. Failure Semantics

Debe distinguirse:

```text
relationship operation succeeded
```

de:

```text
transaction committed
```

---

# 158. Statement Success ≠ Commit

Una FK update exitosa dentro de una transacción todavía puede ser revertida.

---

# 159. UNKNOWN Outcome

Si se pierde conexión después de enviar una operación:

```text
Outcome = UNKNOWN
```

No deberá actualizarse baseline como éxito confirmado.

---

# 160. Relationship Snapshot Reconciliation

Solo tras outcome suficientemente cierto:

```text
baseline ← current relationship
```

según transaction/reconciliation policy.

---

# 161. Rollback

Database rollback no significa automáticamente que el object graph vuelva al estado anterior.

---

# 162. Example

Antes:

```text
baseline Profile#10
```

Objeto modificado:

```text
current Profile#11
```

Flush ejecuta update.

Luego transaction rollback.

El objeto puede seguir:

```text
current Profile#11
```

y deberá quedar correctamente dirty/stale según reconciliation policy.

---

# 163. No Object Time Machine

VoltStack no intentará mágicamente reconstruir todas las referencias PHP tras rollback.

---

# 164. Persistence Consistency Integration

Estados relevantes:

```text
CONSISTENT
STALE
UNCERTAIN
INCONSISTENT
TAINTED
UNKNOWN
```

---

# 165. Unknown Relationship Persistence

Un outcome desconocido puede:

```text
taint PersistenceContext
```

si no existe forma segura de reconciliarlo.

---

# 166. Recovery

Opciones:

```text
refresh affected entities
clear context
close EntityManager
new PersistenceContext
application reconciliation
```

---

# 167. Serialization

Serializar `User` no deberá necesariamente inicializar `profile`.

---

# 168. Safe Serialization

Puede representar:

```text
profile omitted
```

o:

```text
relationship not loaded
```

según API serialization policy.

---

# 169. Avoid N+1

No:

```text
serialize 1,000 users
→ 1,000 profile queries
```

por accidente.

---

# 170. Batch Loading

Aunque cada owner tenga máximo un target, múltiples owners pueden cargarse juntos.

Ejemplo:

```text
User#1.profile
User#2.profile
User#3.profile
```

→

```text
SELECT profiles ...
WHERE user_id IN (1,2,3)
```

---

# 171. Batch Result Mapping

Debe mapear cada target a exactamente su owner.

---

# 172. Duplicate Target Result

Si DB retorna dos rows para una relación one-to-one que debería ser única:

```text
OneToOneCardinalityViolationException
```

No escoger arbitrariamente:

```text
first()
```

---

# 173. Critical Rule

```text
ExpectedAtMostOne
+
ObservedMoreThanOne
⇒
CardinalityViolation
```

---

# 174. Database Corruption Detection

Esto permite detectar:

- falta de UNIQUE constraint;
- datos legacy inconsistentes;
- mapping incorrecto;
- query incorrecta.

---

# 175. Loader Result

Propuesta:

```php
final readonly class ToOneLoadResult
{
    public function __construct(
        public RelationshipLoadOutcome $outcome,
        public RelationshipKnowledge $knowledge,
        public ?object $entity,
        public ToOneRelationshipCompleteness $completeness,
    ) {}
}
```

---

# 176. Zero Rows

En load completo:

```text
0 rows
→ KNOWN_NULL
```

si optional.

Si required:

```text
0 rows
→ RequiredRelationshipMissingException
```

según strictness/mapping.

---

# 177. One Row

```text
1 row
→ canonical related entity
```

---

# 178. More Than One

```text
>1 rows
→ cardinality violation
```

---

# 179. Required Missing Relationship

Puede indicar:

- corrupted DB;
- stale replica;
- cross-database inconsistency;
- mapping mismatch;
- transient migration state.

El error deberá conservar contexto diagnóstico.

---

# 180. Replica Reads

Una required relation ausente en replica podría deberse a lag.

El Relationship System no deberá adivinar.

Read/write routing posterior podrá definir retry-on-primary policy.

---

# 181. Relationship Loader Does Not Own Failover

El loader expresa:

```text
read intent
```

Connection routing decide origen físico.

---

# 182. Cache Integration

Entity cache/result cache son sistemas separados.

Una cached target entity deberá reconciliarse con IdentityMap antes de asociación.

---

# 183. Relationship Cache

No deberá existir un cache global mutable de:

```text
User#1.profile object instance
```

---

# 184. Cached Relationship Identity

Una futura cache podría guardar:

```text
User#1.profile → Profile#10
```

con invalidation/consistency explícita.

Pero no object references request-scoped.

---

# 185. Persistent Runtime

Shared:

```text
OneToOneMetadata
compiled accessors
compiled load plans
immutable persistence descriptors
```

Scoped:

```text
ToOne runtime states
relationship snapshots
references/proxies
load guards
change sets
```

---

# 186. FrankenPHP

```text
Worker
├── immutable OneToOne metadata
│
├── Request A
│   ├── User#1
│   ├── Profile#10
│   └── relationship state A
│
└── Request B
    ├── User#1
    ├── Profile#10
    └── relationship state B
```

Los objetos de A y B no son los mismos.

---

# 187. Request Reset

Al terminar scope:

```text
clear relationship runtime state
clear references/proxies owned by context
clear load queues
clear relationship snapshots with UoW
```

---

# 188. No Process-Global Relationship State

Nunca:

```php
static array $loadedOneToOneRelations;
```

---

# 189. Concurrency

Una referencia lazy mutable no será thread/fiber-safe por defecto.

---

# 190. Concurrent Initialization

Debe evitar:

```text
Fiber A → load Profile
Fiber B → load Profile simultaneously
```

sobre la misma reference sin coordinación.

---

# 191. Initialization Guard

Estado:

```text
UNINITIALIZED
      │
      ▼
INITIALIZING
      │
      ├── success → ENTITY / NULL
      │
      └── failure → recoverable state
```

---

# 192. Recursive Lazy Initialization

Si durante inicialización se vuelve a acceder a la misma referencia:

```text
detect recursion
```

y evitar loop infinito.

---

# 193. Security

One-to-one deberá respetar:

- identity namespaces;
- tenant boundaries;
- discriminator policies si target es polimórfico;
- connection routing;
- metadata whitelist;
- no arbitrary class instantiation.

---

# 194. Authorization

```text
User.profile exists
```

no significa:

```text
current application user may access Profile
```

---

# 195. Relationship Loading ≠ Authorization

Authorization deberá aplicarse en la capa correspondiente.

---

# 196. Mass Assignment

Hydration de DB no utiliza reglas de mass assignment.

Asignación desde HTTP/input sí deberá pasar por APIs de aplicación correspondientes.

---

# 197. Internal Field Writer

Hydration puede usar compiled accessors internos.

No deben quedar disponibles como mecanismo público para saltarse invariantes de dominio.

---

# 198. Readonly Properties

Una one-to-one almacenada en propiedad `readonly` requiere estrategia compatible.

Opciones:

```text
constructor/factory hydration
write-once compiled accessor where legal
mapping rejection
```

---

# 199. Replacement of readonly relation

Si el dominio declara una relación realmente inmutable:

```text
replacement
```

deberá rechazarse.

---

# 200. Immutable Relationship Metadata

Podrá existir:

```text
mutable = false
```

como semántica mapping/domain.

---

# 201. Relationship Mutability

Propuesta:

```php
enum RelationshipMutability
{
    case MUTABLE;
    case IMMUTABLE_AFTER_ESTABLISHMENT;
}
```

---

# 202. Immutable Relationship

Una vez baseline:

```text
Profile#10
```

no podrá cambiar a:

```text
Profile#11
```

sin operación especializada.

---

# 203. Uniqueness and Soft Delete

Si target utiliza soft delete, una unique FK puede interactuar con registros eliminados lógicamente.

Esto deberá resolverse entre:

```text
Relationship
Soft Delete
Schema
Platform Capabilities
```

No mediante hacks del loader.

---

# 204. Temporal Entities

Igualmente, relaciones temporales/versionadas pueden requerir semánticas futuras específicas.

El one-to-one base no asumirá:

```text
one physical row forever
```

---

# 205. Relationship Metadata Validation

Durante bootstrap deberá comprobar:

- source entity existe;
- target entity existe;
- property/accessor válido;
- owning/inverse coherentes;
- mappedBy/inversedBy coherentes;
- cardinalidad compatible;
- storage strategy válida;
- cascade válida;
- orphan removal válida;
- optionality válida;
- identity mapping válida.

---

# 206. Bidirectional Pair Validation

Si:

```text
User.profile mappedBy Profile.user
```

entonces `Profile.user` deberá existir y apuntar a `User`.

---

# 207. Cardinality Pair Validation

No aceptar accidentalmente:

```text
User.profile = ONE_TO_ONE
Profile.user = ONE_TO_MANY
```

como dos lados de la misma relación.

---

# 208. Ownership Validation

No aceptar:

```text
User.profile OWNED
Profile.user OWNED
```

para el mismo mapping ordinario.

---

# 209. Orphan Removal Validation

Orphan removal deberá permitirse solo donde la semántica de dependencia sea suficientemente clara.

---

# 210. Shared PK Validation

Debe verificar compatibilidad entre:

```text
principal identifier type
dependent identifier type
```

---

# 211. Mapping Compilation

Attributes/configuración fuente:

```text
#[OneToOne(...)]
```

se normalizan a:

```text
OneToOneMetadata
```

inmutable.

---

# 212. No Reflection Hot Path

Reflection se usa durante compilación.

No por cada:

```text
$user->profile
```

---

# 213. Attribute Example

Conceptualmente:

```php
#[OneToOne(
    target: Profile::class,
    mappedBy: 'user',
    fetch: 'lazy',
    orphanRemoval: true,
)]
private ?Profile $profile = null;
```

---

# 214. Owning Example

```php
#[OneToOne(
    target: User::class,
    inversedBy: 'profile',
)]
#[JoinColumn(
    name: 'user_id',
    referencedColumn: 'id',
    unique: true,
    nullable: false,
)]
private User $user;
```

Los atributos son fuente declarativa, no runtime engine.

---

# 215. No SQL in Metadata

No almacenar:

```text
SELECT ...
UPDATE ...
```

dentro de `OneToOneMetadata`.

---

# 216. Query Resolution

Entity Query puede resolver:

```php
User::query()
    ->where('profile.country', 'MX');
```

mediante Relationship Metadata.

---

# 217. Query Relation Navigation

Semantic Query Engine traduce:

```text
User.profile.country
```

a un graph relacional lógico.

No el OneToOne System directamente a SQL.

---

# 218. Null Predicates

Ejemplo:

```php
User::query()
    ->whereRelationNull('profile');
```

deberá traducirse semánticamente según mapping.

---

# 219. Existence Queries

```php
User::query()
    ->has('profile');
```

podrá usar:

```text
EXISTS
JOIN
FK predicate
```

según optimizer/compiler.

---

# 220. Relationship System Does Not Pick SQL

Reiteración:

```text
has(profile)
→ Query Model
→ Optimizer
→ Planner
→ Compiler
```

---

# 221. Telemetry

Métricas sugeridas:

```text
orm.relationship.one_to_one.loads
orm.relationship.one_to_one.lazy_loads
orm.relationship.one_to_one.eager_loads
orm.relationship.one_to_one.batch_loads

orm.relationship.one_to_one.identity_map_hits
orm.relationship.one_to_one.identity_map_misses

orm.relationship.one_to_one.assigned
orm.relationship.one_to_one.replaced
orm.relationship.one_to_one.nullified

orm.relationship.one_to_one.cascade_persist
orm.relationship.one_to_one.cascade_remove
orm.relationship.one_to_one.orphan_removals

orm.relationship.one_to_one.cardinality_violations
orm.relationship.one_to_one.required_missing
orm.relationship.one_to_one.cross_context_failures
orm.relationship.one_to_one.load_failures
```

---

# 222. Diagnostics

Ejemplo:

```text
One-to-One Relationship
────────────────────────────────

Relationship: User.profile
Target:       Profile
Direction:    BIDIRECTIONAL
Ownership:    INVERSE
Mapped By:    Profile.user

Fetch:        LAZY
State:        ENTITY
Completeness: COMPLETE

Owner:        managed
Target:       managed
IdentityMap:  canonical

Dirty:        NO
OrphanRemoval:YES
```

---

# 223. Cardinality Diagnostic

```text
ONE-TO-ONE CARDINALITY VIOLATION

Relationship:
    User.profile

Owner:
    User

Expected:
    0..1 Profile

Observed:
    2 Profile rows

Possible causes:
    - missing UNIQUE constraint
    - legacy inconsistent data
    - incorrect relationship mapping
    - malformed query
```

---

# 224. Error Taxonomy

```text
OneToOneRelationshipException
├── OneToOneMetadataException
├── OneToOneMappingException
├── OneToOneOwnershipException
├── OneToOneInverseMappingException
├── OneToOneCardinalityViolationException
├── RequiredOneToOneMissingException
├── OneToOneIdentityException
├── OneToOneSharedPrimaryKeyException
├── OneToOneLoadException
├── OneToOneLazyLoadException
├── OneToOneInitializationException
├── OneToOneChangeTrackingException
├── OneToOnePersistenceException
├── OneToOneOrphanRemovalException
├── OneToOneCascadeException
├── OneToOneDetachedEntityException
├── OneToOneCrossContextException
├── OneToOneTenantIsolationException
├── OneToOneConcurrencyException
├── OneToOneReconciliationException
├── OneToOneRuntimeIsolationException
└── OneToOneInvariantException
```

---

# 225. Directory Structure

```text
src/Quantum/Database/ORM/Relationship/OneToOne/
│
├── Metadata/
│   ├── OneToOneMetadata.php
│   ├── OneToOneStorageStrategy.php
│   ├── RelationshipOptionality.php
│   └── RelationshipMutability.php
│
├── State/
│   ├── ToOneRelationshipState.php
│   ├── ToOneRelationshipCompleteness.php
│   ├── ToOneRelationshipSnapshot.php
│   └── ToOneRelationshipRuntimeState.php
│
├── Reference/
│   ├── ToOneReference.php
│   ├── LazyToOneReference.php
│   └── ToOneInitializationGuard.php
│
├── Loading/
│   ├── OneToOneLoader.php
│   ├── OneToOneLoadPlan.php
│   └── ToOneLoadResult.php
│
├── Hydration/
│   ├── OneToOneAssembler.php
│   ├── OneToOneFixUp.php
│   └── OneToOneFixUpLedger.php
│
├── ChangeTracking/
│   ├── ToOneRelationshipChangeSet.php
│   ├── ToOneChangeKind.php
│   └── OneToOneChangeTracker.php
│
├── Persistence/
│   ├── OneToOnePersistenceOperation.php
│   ├── AssociateOneToOneOperation.php
│   ├── DissociateOneToOneOperation.php
│   ├── ReplaceOneToOneOperation.php
│   └── OneToOnePersistencePlanner.php
│
├── Validation/
│   ├── OneToOneGraphValidator.php
│   └── OneToOneMappingValidator.php
│
├── Telemetry/
│   ├── OneToOneTelemetry.php
│   └── OneToOneDiagnostics.php
│
└── Exception/
    └── ...
```

---

# 226. Testing Strategy

El sistema deberá probar como mínimo:

```text
unidirectional one-to-one
bidirectional one-to-one

owning side
inverse side

optional relationship
required relationship

unique FK
shared primary key

lazy loading
eager loading
batch loading

IdentityMap reuse
hydration fix-up

assignment
replacement
nullification

cascade persist
cascade remove
orphan removal

generated identifiers
flush ordering

partial hydration
filtered joins
refresh

cross-context
tenant isolation

cardinality violations
persistent runtime
concurrency
```

---

# 227. Basic Hydration Test

DB:

```text
User#1
Profile#10 → User#1
```

Expected:

```text
$user->profile === $profile
$profile->user === $user
```

---

# 228. IdentityMap Test

Si `Profile#10` ya está managed:

```text
load User#1.profile
```

debe reutilizar la misma instancia.

---

# 229. Duplicate Row Test

Si query retorna accidentalmente dos rows target distintas:

```text
Profile#10
Profile#11
```

para el mismo one-to-one owner:

```text
cardinality violation
```

---

# 230. Optional Absence Test

Load completo sin target:

```text
state = NULL
completeness = COMPLETE
```

---

# 231. Filtered Absence Test

Filtered JOIN sin target:

```text
must not infer COMPLETE NULL
```

---

# 232. Required Missing Test

Load completo de relación required sin target:

```text
RequiredOneToOneMissingException
```

según strict policy.

---

# 233. Lazy Test

Primer acceso carga.

Segundo acceso:

```text
no additional query
```

---

# 234. IdentityMap Lazy Test

Target ya managed:

```text
zero DB query
```

si identity reference es suficiente.

---

# 235. Fix-Up Test

Hydration bidireccional sincroniza ambos lados sin dirty.

---

# 236. Mutation Test

Después de baseline:

```php
$user->setProfile($newProfile);
```

produce `REPLACED`.

---

# 237. Nullification Test

```php
$user->setProfile(null);
```

produce `NULLIFIED`.

---

# 238. Orphan Removal Test

Con baseline conocido:

```text
Profile#10 → null
```

programa orphan removal.

---

# 239. Unknown Orphan Test

Sin identidad previa conocida:

```text
must not invent orphan delete
```

---

# 240. Cascade Persist Test

NEW target con cascade persist:

```text
registered in UoW
```

---

# 241. No Cascade Test

NEW target sin cascade:

```text
flush fails before unsafe persistence
```

---

# 242. Shared PK Test

Generated principal ID se propaga correctamente al dependent.

---

# 243. Temporary ID Test

Temporary UoW ID nunca llega a DB.

---

# 244. Replacement Ordering Test

Unique constraint no se viola por ordering incorrecto.

---

# 245. Bidirectional Conflict Test

```text
User#1.profile = Profile#10
Profile#10.user = User#2
```

se detecta.

---

# 246. Local Uniqueness Test

Dos owners conocidos no pueden poseer el mismo target en una one-to-one estricta.

---

# 247. Detached Test

Detached target no se interpreta como NEW.

---

# 248. Cross-Context Test

Context A owner + Context B target:

```text
rejected
```

---

# 249. Tenant Test

Tenant A owner + Tenant B target:

```text
rejected
```

por default.

---

# 250. Dirty Hydration Test

Query normal no sobrescribe relación dirty.

---

# 251. Explicit Refresh Test

Refresh aplica policy declarada.

---

# 252. postLoad Test

postLoad mutation se convierte en future dirty state.

---

# 253. UNKNOWN Persistence Test

Outcome incierto no actualiza snapshot como éxito.

---

# 254. Rollback Test

Rollback no simula object graph rewind.

---

# 255. Persistent Runtime Test

Dos requests en mismo worker no comparten:

```text
references
runtime state
snapshots
entities
```

---

# 256. Concurrency Test

Dos initializations concurrentes se coordinan.

---

# 257. Architectural Invariants

## DB-ORM-ONE-TO-ONE-001

One-to-one será una relación to-one, no una colección limitada.

## DB-ORM-ONE-TO-ONE-002

One-to-one será distinto de ForeignKey.

## DB-ORM-ONE-TO-ONE-003

One-to-one será distinto de UniqueConstraint.

## DB-ORM-ONE-TO-ONE-004

La representación física no definirá por sí sola la semántica ORM.

## DB-ORM-ONE-TO-ONE-005

Cada owner tendrá como máximo un target dentro de la relación.

## DB-ORM-ONE-TO-ONE-006

Una one-to-one bidireccional tendrá un único owning side normal.

## DB-ORM-ONE-TO-ONE-007

Owning side será distinto de principal.

## DB-ORM-ONE-TO-ONE-008

Owning side será distinto de aggregate owner.

## DB-ORM-ONE-TO-ONE-009

Inverse side no será persistence authority por sí solo.

## DB-ORM-ONE-TO-ONE-010

Directionality será explícita.

## DB-ORM-ONE-TO-ONE-011

Optionality será explícita.

## DB-ORM-ONE-TO-ONE-012

Fetch mode será explícito.

## DB-ORM-ONE-TO-ONE-013

Storage strategy será explícita.

## DB-ORM-ONE-TO-ONE-014

Unique FK y shared PK serán estrategias distintas.

## DB-ORM-ONE-TO-ONE-015

Shared identifier value no fusionará EntityTypes distintos.

## DB-ORM-ONE-TO-ONE-016

EntityKey conservará namespace de tipo.

## DB-ORM-ONE-TO-ONE-017

Derived identity será coordinada por Persistence Planner.

## DB-ORM-ONE-TO-ONE-018

Temporary UoW identity no se persistirá como ID real.

## DB-ORM-ONE-TO-ONE-019

NULL será distinto de UNINITIALIZED.

## DB-ORM-ONE-TO-ONE-020

REFERENCE será distinto de ENTITY.

## DB-ORM-ONE-TO-ONE-021

Known identity será distinta de loaded entity state.

## DB-ORM-ONE-TO-ONE-022

Required será distinto de initialized.

## DB-ORM-ONE-TO-ONE-023

To-one runtime state será scoped.

## DB-ORM-ONE-TO-ONE-024

OneToOneMetadata será immutable.

## DB-ORM-ONE-TO-ONE-025

Runtime state no vivirá en shared metadata.

## DB-ORM-ONE-TO-ONE-026

IdentityMap canonicality será preservada.

## DB-ORM-ONE-TO-ONE-027

IdentityMap hit reutilizará canonical target.

## DB-ORM-ONE-TO-ONE-028

Lazy resolution no requerirá query cuando IdentityMap sea suficiente.

## DB-ORM-ONE-TO-ONE-029

Lazy loading será distinto de proxy.

## DB-ORM-ONE-TO-ONE-030

Eager loading será distinto de JOIN.

## DB-ORM-ONE-TO-ONE-031

Batch loading será válido para múltiples owners.

## DB-ORM-ONE-TO-ONE-032

Relationship Loader no generará SQL directamente.

## DB-ORM-ONE-TO-ONE-033

Relationship Loader utilizará Query Engine.

## DB-ORM-ONE-TO-ONE-034

Optional joined absence no creará entidad con ID null.

## DB-ORM-ONE-TO-ONE-035

Partial composite identifier no será tratado como ausencia automáticamente.

## DB-ORM-ONE-TO-ONE-036

Hydration fix-up sincronizará ambos lados cuando corresponda.

## DB-ORM-ONE-TO-ONE-037

Hydration fix-up no marcará dirty artificialmente.

## DB-ORM-ONE-TO-ONE-038

Fix-up será cycle-safe.

## DB-ORM-ONE-TO-ONE-039

Hydrated baseline coincidirá con relación observada.

## DB-ORM-ONE-TO-ONE-040

Relationship snapshot conservará knowledge state.

## DB-ORM-ONE-TO-ONE-041

UNOBSERVED no será representado como KNOWN_NULL.

## DB-ORM-ONE-TO-ONE-042

Change tracking será identity-based.

## DB-ORM-ONE-TO-ONE-043

Deep object equality no determinará cambios de relación.

## DB-ORM-ONE-TO-ONE-044

ASSIGNED será distinto de REPLACED.

## DB-ORM-ONE-TO-ONE-045

REPLACED será distinto de NULLIFIED.

## DB-ORM-ONE-TO-ONE-046

Relationship ChangeSet será distinto de SQL.

## DB-ORM-ONE-TO-ONE-047

Explicit mutation podrá registrarse sin hidden load cuando sea seguro.

## DB-ORM-ONE-TO-ONE-048

Explicit mutation intent será distinto de observed baseline.

## DB-ORM-ONE-TO-ONE-049

Persistence operation será semántica.

## DB-ORM-ONE-TO-ONE-050

Persistence Planner decidirá statements físicos.

## DB-ORM-ONE-TO-ONE-051

Nullification no implicará delete automáticamente.

## DB-ORM-ONE-TO-ONE-052

Orphan Removal será explícito.

## DB-ORM-ONE-TO-ONE-053

Orphan Removal será distinto de Cascade Remove.

## DB-ORM-ONE-TO-ONE-054

Orphan Removal no inventará previous identity desconocida.

## DB-ORM-ONE-TO-ONE-055

Cascade Persist será distinto de DB cascade.

## DB-ORM-ONE-TO-ONE-056

Cascade Remove será distinto de orphan removal.

## DB-ORM-ONE-TO-ONE-057

NEW target requerirá cascade persist o registro explícito.

## DB-ORM-ONE-TO-ONE-058

Detached target no será asumido NEW.

## DB-ORM-ONE-TO-ONE-059

Removed target no será asociación activa válida por defecto.

## DB-ORM-ONE-TO-ONE-060

Generated ID dependencies aparecerán en PersistencePlan.

## DB-ORM-ONE-TO-ONE-061

Shared PK respetará generated ID barriers.

## DB-ORM-ONE-TO-ONE-062

Insert ordering será dependency-aware.

## DB-ORM-ONE-TO-ONE-063

Delete ordering será constraint-aware.

## DB-ORM-ONE-TO-ONE-064

Replacement ordering respetará unique constraints.

## DB-ORM-ONE-TO-ONE-065

Known duplicate ownership podrá detectarse antes de execution.

## DB-ORM-ONE-TO-ONE-066

In-memory uniqueness no reemplazará DB uniqueness.

## DB-ORM-ONE-TO-ONE-067

Concurrency global no se resolverá solo con object graph validation.

## DB-ORM-ONE-TO-ONE-068

DB uniqueness será recomendada donde sea representable.

## DB-ORM-ONE-TO-ONE-069

Locking pertenecerá al Transaction/Concurrency System.

## DB-ORM-ONE-TO-ONE-070

Refresh tendrá relationship policy explícita.

## DB-ORM-ONE-TO-ONE-071

Refresh no cargará todo el graph implícitamente.

## DB-ORM-ONE-TO-ONE-072

Dirty relationship no será sobrescrita por hydration normal.

## DB-ORM-ONE-TO-ONE-073

Explicit refresh podrá reemplazar dirty state solo conforme a policy.

## DB-ORM-ONE-TO-ONE-074

Partial entity no implicará relation null.

## DB-ORM-ONE-TO-ONE-075

PHP property default no será ORM knowledge.

## DB-ORM-ONE-TO-ONE-076

Projection no inicializará relación managed automáticamente.

## DB-ORM-ONE-TO-ONE-077

Query result no será relationship initialization sin HydrationPlan explícito.

## DB-ORM-ONE-TO-ONE-078

Filtered JOIN absence no implicará relationship absence.

## DB-ORM-ONE-TO-ONE-079

To-one tendrá completeness semantics.

## DB-ORM-ONE-TO-ONE-080

Safe known null requerirá complete coverage.

## DB-ORM-ONE-TO-ONE-081

Filtered relationship query no cambiará canonical relationship state automáticamente.

## DB-ORM-ONE-TO-ONE-082

Querying relationship será distinto de initializing relationship.

## DB-ORM-ONE-TO-ONE-083

Managed association requerirá compatible PersistenceContext.

## DB-ORM-ONE-TO-ONE-084

Cross-context association incompatible será rechazada.

## DB-ORM-ONE-TO-ONE-085

Tenant identity namespace será preservado.

## DB-ORM-ONE-TO-ONE-086

Cross-tenant association será explícita o rechazada.

## DB-ORM-ONE-TO-ONE-087

Shard identity namespace será preservado.

## DB-ORM-ONE-TO-ONE-088

Cross-database relation no implicará distributed transaction.

## DB-ORM-ONE-TO-ONE-089

Lifecycle mutation será recollected antes de plan freeze.

## DB-ORM-ONE-TO-ONE-090

post-lifecycle mutation no provocará recursive hidden persistence.

## DB-ORM-ONE-TO-ONE-091

Recursive flush seguirá prohibido.

## DB-ORM-ONE-TO-ONE-092

Statement success será distinto de transaction commit.

## DB-ORM-ONE-TO-ONE-093

UNKNOWN outcome será preservado.

## DB-ORM-ONE-TO-ONE-094

UNKNOWN outcome no avanzará baseline como success.

## DB-ORM-ONE-TO-ONE-095

Rollback DB no rebobinará mágicamente object graph.

## DB-ORM-ONE-TO-ONE-096

Relationship consistency se integrará con Persistence Consistency.

## DB-ORM-ONE-TO-ONE-097

Uncertain relationship outcome podrá taint context.

## DB-ORM-ONE-TO-ONE-098

Recovery será explícita.

## DB-ORM-ONE-TO-ONE-099

Serialization no deberá disparar lazy loading obligatoriamente.

## DB-ORM-ONE-TO-ONE-100

Batch loading preservará canonical entities.

## DB-ORM-ONE-TO-ONE-101

Batch loading mapeará como máximo un target por owner.

## DB-ORM-ONE-TO-ONE-102

Más de un target observado será cardinality violation.

## DB-ORM-ONE-TO-ONE-103

El loader no elegirá arbitrariamente first row ante violation.

## DB-ORM-ONE-TO-ONE-104

Required missing relation será distinguible de optional null.

## DB-ORM-ONE-TO-ONE-105

Replica inconsistency no será ocultada por Relationship System.

## DB-ORM-ONE-TO-ONE-106

Connection routing será externo al loader.

## DB-ORM-ONE-TO-ONE-107

Entity cache se reconciliará con IdentityMap.

## DB-ORM-ONE-TO-ONE-108

No se cachearán object references scoped globalmente.

## DB-ORM-ONE-TO-ONE-109

OneToOneMetadata podrá compartirse en persistent workers.

## DB-ORM-ONE-TO-ONE-110

Relationship runtime state no podrá compartirse entre requests.

## DB-ORM-ONE-TO-ONE-111

Lazy references serán scoped.

## DB-ORM-ONE-TO-ONE-112

Load guards serán scoped.

## DB-ORM-ONE-TO-ONE-113

No habrá process-global mutable relationship state.

## DB-ORM-ONE-TO-ONE-114

Concurrent initialization será coordinada.

## DB-ORM-ONE-TO-ONE-115

Recursive initialization será detectada.

## DB-ORM-ONE-TO-ONE-116

Failed initialization no fabricará NULL.

## DB-ORM-ONE-TO-ONE-117

Security boundaries serán preservados.

## DB-ORM-ONE-TO-ONE-118

Relationship existence no implicará authorization.

## DB-ORM-ONE-TO-ONE-119

DB hydration no usará mass-assignment rules.

## DB-ORM-ONE-TO-ONE-120

Internal hydration accessors no serán public bypass APIs.

## DB-ORM-ONE-TO-ONE-121

Readonly relationship requerirá mapping compatible.

## DB-ORM-ONE-TO-ONE-122

Immutable relationship no podrá reemplazarse silenciosamente.

## DB-ORM-ONE-TO-ONE-123

Soft-delete uniqueness será capability/schema-aware.

## DB-ORM-ONE-TO-ONE-124

Temporal semantics no serán asumidas por core one-to-one.

## DB-ORM-ONE-TO-ONE-125

Mapping será validado durante bootstrap.

## DB-ORM-ONE-TO-ONE-126

Bidirectional mappings deberán apuntarse coherentemente.

## DB-ORM-ONE-TO-ONE-127

Mismatched cardinalities serán mapping errors.

## DB-ORM-ONE-TO-ONE-128

Double owning-side mapping será rechazado cuando sea ambiguo.

## DB-ORM-ONE-TO-ONE-129

Shared PK identifier types deberán ser compatibles.

## DB-ORM-ONE-TO-ONE-130

Attributes serán mapping sources, no runtime engine.

## DB-ORM-ONE-TO-ONE-131

Reflection no estará en el hot path.

## DB-ORM-ONE-TO-ONE-132

OneToOneMetadata no almacenará SQL.

## DB-ORM-ONE-TO-ONE-133

Relationship path queries usarán Semantic Query Engine.

## DB-ORM-ONE-TO-ONE-134

Existence queries no tendrán SQL hardcoded en ORM.

## DB-ORM-ONE-TO-ONE-135

Platform differences serán capability-driven.

## DB-ORM-ONE-TO-ONE-136

No habrá vendor conditionals en relationship semantics.

## DB-ORM-ONE-TO-ONE-137

One-to-one state será observable mediante diagnostics.

## DB-ORM-ONE-TO-ONE-138

Cardinality violations serán diagnosticables.

## DB-ORM-ONE-TO-ONE-139

Orphan decisions serán diagnosticables.

## DB-ORM-ONE-TO-ONE-140

Generated identity barriers serán diagnosticables.

## DB-ORM-ONE-TO-ONE-141

Telemetry no expondrá IDs sensibles como labels.

## DB-ORM-ONE-TO-ONE-142

Relationship loading podrá medir IdentityMap hits.

## DB-ORM-ONE-TO-ONE-143

Relationship mutation telemetry distinguirá assign/replace/nullify.

## DB-ORM-ONE-TO-ONE-144

Required relationship missing tendrá error específico.

## DB-ORM-ONE-TO-ONE-145

Local graph conflict será distinto de DB constraint conflict.

## DB-ORM-ONE-TO-ONE-146

Known graph conflict no se ignorará.

## DB-ORM-ONE-TO-ONE-147

Unknown graph state no generará falsa certeza.

## DB-ORM-ONE-TO-ONE-148

Hydration completeness gobernará inferencia de NULL.

## DB-ORM-ONE-TO-ONE-149

Persistence consistency gobernará baseline reconciliation.

## DB-ORM-ONE-TO-ONE-150

Flush no equivaldrá a commit para relationship persistence.

## DB-ORM-ONE-TO-ONE-151

A successful relation operation dentro de transacción no implicará durability.

## DB-ORM-ONE-TO-ONE-152

One-to-one replacement podrá producir múltiples physical operations.

## DB-ORM-ONE-TO-ONE-153

One semantic operation no será equivalente a one SQL statement.

## DB-ORM-ONE-TO-ONE-154

DB cascade optimization no alterará observable ORM semantics.

## DB-ORM-ONE-TO-ONE-155

Inverse-only mutation conflict tendrá policy explícita.

## DB-ORM-ONE-TO-ONE-156

ORM no adivinará intención de sincronización bidireccional.

## DB-ORM-ONE-TO-ONE-157

One-to-one graph validation será knowledge-aware.

## DB-ORM-ONE-TO-ONE-158

Cardinality validation no tratará UNINITIALIZED como NULL.

## DB-ORM-ONE-TO-ONE-159

La arquitectura preservará identidad, cardinalidad, ownership, completeness y persistence evidence como dimensiones separadas.

## DB-ORM-ONE-TO-ONE-160

VoltStack nunca convertirá falta de conocimiento sobre el target de una one-to-one en evidencia de que el target no existe.

---

# 258. Fórmulas fundamentales

## 258.1 Cardinalidad

```text
OneToOne(A, B)
⇒
∀ a ∈ A: |relatedB(a)| ≤ 1
```

Para relación estrictamente bidireccional:

```text
∀ b ∈ B: |relatedA(b)| ≤ 1
```

---

# 259. Canonical Target

```text
KnownTargetKey
+
IdentityMapHit
⇒
CanonicalTargetInstance
```

---

# 260. Safe Null Resolution

```text
KnownNull
=
SuccessfulCompleteRelationshipLoad
∧
NoTargetObserved
```

No:

```text
NoTargetInFilteredQuery
⇒
KnownNull
```

---

# 261. Relationship State

```text
ToOneRuntimeState
=
Knowledge
+
Completeness
+
Initialization
+
CanonicalTarget
+
Baseline
+
MutationIntent
```

---

# 262. Assignment

```text
ASSIGNED
=
BaselineKnownNull
∧
CurrentTargetKnown
```

---

# 263. Replacement

```text
REPLACED
=
BaselineTarget ≠ CurrentTarget
∧
BothStatesSufficientlyKnown
```

---

# 264. Nullification

```text
NULLIFIED
=
BaselineTargetKnown
∧
CurrentKnownNull
```

---

# 265. Safe Orphan Removal

```text
SafeOrphanRemoval
=
OrphanRemovalEnabled
∧
OldTargetIdentityKnown
∧
RelationshipMutationConfirmed
∧
OwnershipSemanticsSatisfied
```

---

# 266. Shared Primary Key

```text
DependentIdentifier
=
Canonicalize(PrincipalIdentifier)
```

con tipos compatibles.

---

# 267. Safe Bidirectional Graph

```text
User.profile = Profile
⇔
Profile.user = User
```

cuando ambos lados están suficientemente conocidos y mapping declara esa asociación bidireccional.

---

# 268. Safe Persistence

```text
SafeOneToOnePersistence
=
ValidMetadata
∧
CompatiblePersistenceContexts
∧
ValidIdentityNamespaces
∧
KnownMutationIntent
∧
OwnershipResolved
∧
DependenciesPlanned
∧
PlatformCapabilitiesValidated
```

---

# 269. Safe Runtime

```text
SafeOneToOneRuntime
=
ImmutableSharedMetadata
∧
ScopedRelationshipState
∧
ScopedIdentityMap
∧
ScopedReferences
∧
ScopedLoadGuards
∧
NoCrossRequestObjectGraph
```

---

# 270. Master Formula

```text
Database One-to-One Relationship System
=
OneToOne Metadata
+
Cardinality Semantics
+
Directionality
+
Owning/Inverse Semantics
+
Principal/Dependent Semantics
+
Optionality
+
Unique Foreign Key Mapping
+
Shared Primary Key Mapping
+
Identity Canonicalization
+
IdentityMap Integration
+
To-One Runtime State
+
Relationship Knowledge
+
Completeness Semantics
+
Entity References
+
Lazy Loading
+
Eager Loading
+
Batch Loading
+
Hydration Assembly
+
Bidirectional Fix-Up
+
Relationship Snapshots
+
Change Tracking
+
Assignment
+
Replacement
+
Nullification
+
Cascade Persist
+
Cascade Remove
+
Orphan Removal
+
Generated Identity Dependencies
+
Persistence Planning
+
Uniqueness Validation
+
Concurrency Governance
+
Refresh Semantics
+
Partial Hydration Safety
+
Tenant/Shard Isolation
+
Persistent Runtime Isolation
+
Failure Reconciliation
+
Telemetry
+
Diagnostics
+
Testing
```

---

# 271. Regla maestra final

> **VoltStack tratará una relación one-to-one como una asociación de identidad con cardinalidad máxima uno, gobernada por metadata, ownership y evidencia explícita del estado de la relación. La ausencia de datos, una relación no inicializada o una query filtrada nunca serán convertidas automáticamente en evidencia de `NULL`; y ninguna optimización física podrá romper la identidad canónica, la unicidad semántica o las garantías del PersistenceContext.**

---

# 272. Integración con la arquitectura general

```text
Entity Metadata
      │
      ▼
OneToOne Metadata
      │
      ├──────────────────────────────┐
      │                              │
      ▼                              ▼
Entity Query                   Runtime State
      │                              │
      ▼                              ├── UNINITIALIZED
Query Engine                    ├── REFERENCE
      │                              ├── NULL
      ▼                              └── ENTITY
Execution                           │
      │                              ▼
      ▼                        IdentityMap
Hydration                           │
      │                              ▼
      ├──── Identity Resolution ─→ Canonical Entity
      │                              │
      └──── Bidirectional Fix-Up ────┘
                                     │
                                     ▼
                              Relationship Snapshot
                                     │
                                     ▼
                               Change Tracking
                                     │
                         ┌───────────┼────────────┐
                         ▼           ▼            ▼
                      Assign      Replace      Nullify
                         │           │            │
                         └───────────┼────────────┘
                                     ▼
                         Relationship Persistence
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

---

# 273. Resultado arquitectónico

Con este sistema VoltStack obtiene una one-to-one que no depende de convenciones frágiles como:

```text
"si existe una FK UNIQUE entonces es one-to-one"
```

sino de una definición explícita:

```text
Relationship Semantics
        ↓
Compiled Metadata
        ↓
Runtime Knowledge
        ↓
Canonical Identity
        ↓
Change Tracking
        ↓
Persistence Intent
        ↓
Persistence Planning
```

Esto permite utilizar la misma arquitectura tanto para:

```text
Laravel-like Model API
```

como para:

```text
EntityManager / Repository
```

sin crear dos motores de relaciones diferentes.

---

# 274. Siguiente documento

```text
144_DATABASE_ONE_TO_MANY_RELATIONSHIP_SYSTEM.md
```

El siguiente documento deberá definir la arquitectura de colecciones one-to-many, incluyendo:

```text
OneToManyMetadata
inverse-side semantics
owning many-to-one counterpart
PersistentCollection
SET/LIST semantics
collection initialization
collection completeness
collection snapshots
membership identity
add/remove operations
collection diffing
hydration deduplication
JOIN row multiplication
lazy collections
eager loading
batch loading
extra-lazy operations
filtered collections
ordering
orphan removal
cascade persist/remove
collection clear semantics
partial collections
generated owner IDs
persistence planning
N+1 integration
large collections
persistent runtime
telemetry
testing
```

La regla central será:

> **Una relación one-to-many de VoltStack será una colección ORM identity-aware cuyo estado de membresía solo podrá considerarse completo cuando exista evidencia suficiente de cobertura; una colección no inicializada o parcialmente cargada jamás será interpretada como representación total de la relación persistida.**