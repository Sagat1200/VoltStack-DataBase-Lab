# 148_DATABASE_RELATIONSHIP_METADATA_SYSTEM.md

# VoltStack Quantum Database
## Database Relationship Metadata System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 148 — Database Relationship Metadata System  
**Bloque:** 13 — Relationships  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Relationship Metadata System` define la representación canónica, immutable, validada y compilada de las relaciones ORM de VoltStack.

El sistema deberá describir de manera uniforme:

```text
OneToOne
OneToMany
ManyToOne
ManyToMany
Polymorphic
```

sin convertir la metadata en:

- SQL;
- Query AST;
- Hydration runtime state;
- UnitOfWork state;
- PersistencePlan;
- Entity instance;
- database schema;
- lazy loader;
- collection state.

La metadata será el contrato estructural compartido por los subsistemas que necesitan conocer la semántica de una relación.

---

# 2. Regla central

> **`RelationshipMetadata` será la descripción ORM canónica, immutable y compilada de una relación; loaders, hydrators, Entity Query, UnitOfWork y Persistence Planner podrán consumirla, pero ninguno deberá redescubrir mappings mediante reflection, atributos o convenciones durante el hot path.**

Por tanto:

```text
Mapping Sources
      ↓
Normalization
      ↓
Validation
      ↓
Compilation
      ↓
RelationshipMetadata
      ↓
Runtime Consumers
```

y nunca:

```text
Runtime operation
      ↓
Reflection
      ↓
Discover relationship
      ↓
Guess semantics
```

---

# 3. Objetivos

El sistema deberá proporcionar:

1. identidad estable de relaciones;
2. representación uniforme de relationship kinds;
3. source y target EntityTypes;
4. ownership;
5. direction;
6. cardinality;
7. nullability;
8. join metadata;
9. foreign-key bindings;
10. pivot/join-table metadata;
11. polymorphic metadata;
12. collection semantics;
13. fetch strategy;
14. cascade policy;
15. orphan-removal policy;
16. ordering;
17. filtering;
18. inverse mappings;
19. hydration requirements;
20. loading requirements;
21. persistence requirements;
22. generation/fingerprinting;
23. registry;
24. validation;
25. compilation;
26. extension metadata;
27. diagnostics;
28. persistent-runtime safety.

---

# 4. Relationship Metadata ≠ Mapping Source

Un atributo PHP:

```php
#[ManyToOne(target: User::class)]
private User $author;
```

es una fuente declarativa.

No es la metadata runtime.

Formalmente:

```text
Attribute Mapping
≠
RelationshipMetadata
```

---

# 5. Relationship Metadata ≠ Reflection

Reflection puede utilizarse durante bootstrap para descubrir:

```text
property
type
attributes
visibility
```

pero no deberá conservarse como mecanismo primario de resolución runtime.

---

# 6. Relationship Metadata ≠ Schema

La metadata ORM puede declarar:

```text
Post.author
→ User
```

pero eso no demuestra que la base tenga:

```sql
FOREIGN KEY (author_id) REFERENCES users(id)
```

Por tanto:

```text
ORM Relationship Metadata
≠
Database Schema Model
```

---

# 7. Relationship Metadata ≠ Foreign Key

Una FK es una restricción relacional.

Una relación ORM incluye además:

```text
navigation
ownership
inverse side
fetch semantics
cascade
orphan removal
hydration
collection behavior
persistence semantics
```

---

# 8. Relationship Metadata ≠ Runtime State

Metadata:

```text
Post.comments is OneToMany
```

Runtime state:

```text
Post#42.comments = PARTIAL
```

Son conceptos diferentes.

---

# 9. Relationship Metadata ≠ Relationship Value

Esto:

```php
$post->author
```

es un valor runtime.

Esto:

```text
RelationshipMetadata(Post.author)
```

describe cómo interpretar ese valor.

---

# 10. Relationship Metadata ≠ Query

La metadata podrá ayudar a Entity Query a resolver:

```text
post.author.name
```

pero no contiene un query concreto.

---

# 11. Relationship Metadata ≠ Hydration Plan

Metadata describe la relación en general.

`HydrationPlan` describe cómo materializar un resultado concreto.

```text
RelationshipMetadata
        ↓
HydrationPlanCompiler
        ↓
RelationshipHydrationBinding
```

---

# 12. Relationship Metadata ≠ Persistence Plan

Metadata proporciona reglas.

Persistence Planner decide operaciones concretas.

---

# 13. Relationship Metadata ≠ Collection

Una relación `OneToMany` puede describirse una sola vez y utilizarse para millones de colecciones runtime.

---

# 14. Canonical Relationship Model

Modelo conceptual:

```text
RelationshipMetadata
├── RelationshipId
├── Name
├── Kind
├── Cardinality
├── Source EntityType
├── Target EntityType / TargetSet
├── Ownership
├── Direction
├── Nullability
├── Join Model
├── Fetch Policy
├── Cascade Policy
├── Orphan Policy
├── Collection Policy
├── Ordering
├── Filtering
├── Inverse Mapping
├── Hydration Requirements
├── Loading Requirements
├── Persistence Requirements
└── Extensions
```

---

# 15. RelationshipId

Toda relación tendrá identidad lógica estable.

Propuesta:

```php
final readonly class RelationshipId
{
    public function __construct(
        public EntityTypeId $source,
        public string $name,
    ) {}
}
```

Ejemplo:

```text
Post.author
Post.comments
User.profile
Role.permissions
Comment.commentable
```

---

# 16. RelationshipId ≠ Property Name

`comments` por sí solo no es globalmente único.

Debe existir namespace lógico:

```text
EntityType + RelationshipName
```

---

# 17. RelationshipId Formula

```text
RelationshipId
=
SourceEntityType
×
CanonicalRelationshipName
```

---

# 18. Relationship Name Stability

Cambiar internamente el accessor PHP puede requerir mapping explícito si se desea conservar un identificador lógico estable entre generaciones.

Para V1 podrá utilizarse el nombre canónico de propiedad mientras el sistema de metadata genere nuevos fingerprints ante cambios.

---

# 19. RelationshipKind

```php
enum RelationshipKind
{
    case ONE_TO_ONE;
    case ONE_TO_MANY;
    case MANY_TO_ONE;
    case MANY_TO_MANY;
    case POLYMORPHIC_TO_ONE;
    case POLYMORPHIC_TO_MANY;
    case POLYMORPHIC_MANY_TO_MANY;
}
```

---

# 20. Kind ≠ Cardinality

`RelationshipKind` expresa estructura ORM.

`Cardinality` expresa multiplicidad.

---

# 21. RelationshipCardinality

```php
enum RelationshipCardinality
{
    case TO_ONE;
    case TO_MANY;
}
```

---

# 22. Cardinality Invariant

```text
ONE_TO_ONE  → TO_ONE
MANY_TO_ONE → TO_ONE

ONE_TO_MANY  → TO_MANY
MANY_TO_MANY → TO_MANY
```

Las variantes polimórficas siguen la misma regla.

---

# 23. Source EntityType

Toda relación tiene un source:

```text
Post.author

source = Post
```

---

# 24. Target EntityType

Relaciones no polimórficas tienen target único:

```text
Post.author

target = User
```

---

# 25. Target Set

Una relación polimórfica puede tener:

```text
Comment.commentable

targets =
    Post
    Video
    Photo
```

Por tanto la abstracción general podrá ser:

```php
interface RelationshipTarget
{
}
```

con:

```text
SingleEntityTarget
PolymorphicEntityTargetSet
```

---

# 26. SingleEntityTarget

```php
final readonly class SingleEntityTarget
    implements RelationshipTarget
{
    public function __construct(
        public EntityType $entity,
    ) {}
}
```

---

# 27. PolymorphicTargetSet

Utilizará la infraestructura definida en el documento 147.

---

# 28. Relationship Ownership

Ownership determina qué lado contiene la representación persistente principal de la asociación.

---

# 29. Ownership ≠ Domain Ownership

No debe confundirse con Aggregate Root/domain ownership.

---

# 30. Ownership Model

```php
enum RelationshipOwnership
{
    case OWNING;
    case INVERSE;
}
```

---

# 31. Owning Side

Ejemplo:

```text
posts.author_id
```

Entonces:

```text
Post.author
=
owning side
```

si esa relación administra `author_id`.

---

# 32. Inverse Side

```text
User.posts
```

puede ser:

```text
inverse side
```

de `Post.author`.

---

# 33. Ownership Invariant

Para una asociación bidireccional convencional deberá existir una interpretación persistente no ambigua del owning side.

---

# 34. Both Sides Cannot Independently Own Same FK

No deberá ocurrir:

```text
Post.author owns author_id
AND
User.posts independently owns same association
```

como dos persistencias separadas.

---

# 35. Direction

Direction describe navegabilidad ORM.

```php
enum RelationshipDirection
{
    case UNIDIRECTIONAL;
    case BIDIRECTIONAL;
}
```

---

# 36. Bidirectional Relationship

Ejemplo:

```text
User.posts
    ↕
Post.author
```

---

# 37. Inverse RelationshipId

Metadata deberá poder enlazar:

```text
Post.author
↔
User.posts
```

mediante IDs compilados.

---

# 38. No Runtime Name Guessing

Nunca:

```text
Post.author target User
→ guess inverse is "posts"
```

en runtime.

El inverse mapping será explícito/compilado.

---

# 39. Inverse Binding

```php
final readonly class InverseRelationshipBinding
{
    public function __construct(
        public RelationshipId $inverse,
    ) {}
}
```

---

# 40. Unidirectional Relationship

Puede no tener inverse binding:

```text
AuditLog.actor → User
```

sin:

```text
User.auditLogs
```

---

# 41. Nullability

Para relaciones `TO_ONE`:

```php
enum RelationshipNullability
{
    case REQUIRED;
    case NULLABLE;
}
```

---

# 42. Nullability ≠ PHP Type Alone

PHP:

```php
?User
```

puede ser una fuente de información.

Pero metadata compilada es la autoridad ORM.

---

# 43. Nullability ≠ Database Constraint

Metadata ORM:

```text
NULLABLE
```

y schema:

```text
author_id NOT NULL
```

representan inconsistencia detectable.

Pero uno no reemplaza al otro.

---

# 44. To-Many Nullability

Una colección no se modelará como:

```text
nullable collection
```

La ausencia de elementos será:

```text
EMPTY
```

mientras el conocimiento de carga será:

```text
UNINITIALIZED
PARTIAL
COMPLETE
```

---

# 45. Collection Semantics

Toda relación `TO_MANY` deberá declarar sus semantics.

---

# 46. CollectionMetadata

```php
final readonly class RelationshipCollectionMetadata
{
    public function __construct(
        public CollectionKind $kind,
        public CollectionUniqueness $uniqueness,
        public CollectionOrdering $ordering,
        public CollectionFetchSemantics $fetch,
    ) {}
}
```

---

# 47. Collection Kind

Ejemplos:

```text
SET_LIKE
LIST_LIKE
MAP_LIKE
BAG_LIKE
```

V1 podrá soportar principalmente:

```text
SET_LIKE
LIST_LIKE
```

---

# 48. Collection Uniqueness

Default para relaciones de entidades:

```text
EntityKey uniqueness
```

no:

```text
PHP == comparison
```

---

# 49. Collection State Is Not Metadata

Metadata puede declarar:

```text
SET_LIKE
```

pero:

```text
COMPLETE
PARTIAL
UNINITIALIZED
```

es estado runtime.

---

# 50. Fetch Strategy

```php
enum RelationshipFetchMode
{
    case LAZY;
    case EAGER;
    case EXTRA_LAZY;
}
```

---

# 51. Fetch Mode ≠ Query Execution

`LAZY` significa política.

No significa que metadata ejecute query.

---

# 52. EAGER ≠ JOIN

`EAGER` significa:

```text
target should be loaded as part of operation semantics
```

No necesariamente:

```text
SQL JOIN
```

Puede utilizar:

- join;
- secondary query;
- batch query;
- subselect;
- polymorphic grouped loading.

---

# 53. Fetch Strategy Is Logical

La estrategia física corresponde al planner/loader.

---

# 54. Extra Lazy

`EXTRA_LAZY` permite operaciones como:

```text
count
contains
exists
slice
```

sin inicializar toda la colección cuando sea semánticamente seguro.

---

# 55. Cascade Policy

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

# 56. Cascade ≠ Database CASCADE

ORM:

```text
cascade remove
```

no es lo mismo que:

```sql
ON DELETE CASCADE
```

---

# 57. Cascade Persist

Indica propagación de intención ORM.

No significa insertar inmediatamente.

---

# 58. Cascade Remove

Indica propagación semántica de `remove()`.

No implica SQL inmediato.

---

# 59. Cascade Graph

Persistence Planner deberá usar metadata para construir dependencias.

---

# 60. Infinite Cascade Protection

Metadata graph puede contener ciclos:

```text
A → B
B → A
```

La traversal runtime deberá utilizar visited identity/object sets.

---

# 61. Orphan Removal

```php
enum OrphanRemovalPolicy
{
    case DISABLED;
    case REMOVE_ORPHAN;
}
```

---

# 62. Orphan Removal ≠ Cascade Remove

Son conceptos diferentes.

```text
Cascade Remove:
remove parent → remove child

Orphan Removal:
remove relationship membership → remove orphaned child
```

---

# 63. Orphan Semantics Require Ownership

No todas las relaciones pueden soportar orphan removal de forma segura.

---

# 64. Shared Target Restriction

Si una entidad puede ser compartida por múltiples parents, orphan removal deberá rechazarse o requerir semántica explícita.

---

# 65. Join Model

Las relaciones deberán describir cómo se representa la asociación relacionalmente.

---

# 66. JoinMetadata

Abstracción:

```php
interface RelationshipJoinMetadata
{
}
```

Implementaciones:

```text
ForeignKeyJoinMetadata
JoinTableMetadata
PolymorphicJoinMetadata
CustomJoinMetadata
```

---

# 67. ForeignKeyJoinMetadata

```php
final readonly class ForeignKeyJoinMetadata
    implements RelationshipJoinMetadata
{
    /**
     * @param list<JoinColumnBinding> $columns
     */
    public function __construct(
        public array $columns,
    ) {}
}
```

---

# 68. JoinColumnBinding

```php
final readonly class JoinColumnBinding
{
    public function __construct(
        public FieldId $sourceField,
        public FieldId $targetField,
    ) {}
}
```

---

# 69. Field Binding ≠ Raw SQL Column

Preferentemente metadata ORM referencia campos/mappings compilados.

La traducción a columnas físicas se resuelve mediante field mapping.

---

# 70. Composite Foreign Keys

Se soportará:

```text
(source_a, source_b)
→
(target_a, target_b)
```

---

# 71. Join Binding Order

El orden deberá ser deterministic y formar parte del fingerprint.

---

# 72. JoinTableMetadata

Para many-to-many:

```php
final readonly class JoinTableMetadata
    implements RelationshipJoinMetadata
{
    /**
     * @param list<JoinColumnBinding> $ownerColumns
     * @param list<JoinColumnBinding> $inverseColumns
     */
    public function __construct(
        public TableReference $table,
        public array $ownerColumns,
        public array $inverseColumns,
    ) {}
}
```

---

# 73. Join Table ≠ Pivot Entity

Si la tabla intermedia posee dominio propio:

```text
role
status
created_at
approved_by
```

puede requerir entidad de asociación explícita.

---

# 74. Pivot Metadata

Para pivots simples podrán existir metadata adicional:

```php
final readonly class PivotMetadata
{
    public function __construct(
        public TableReference $table,
        public PivotIdentityPolicy $identity,
        public ?PivotPayloadMetadata $payload,
    ) {}
}
```

---

# 75. Polymorphic Join Metadata

Utilizará:

```text
type discriminator
+
target identifier bindings
```

definidos por el documento 147.

---

# 76. Relationship Mapping Sources

Las fuentes podrán ser:

```text
PHP Attributes
configuration
programmatic metadata
package extensions
generated metadata
```

---

# 77. Source Precedence

Debe existir precedencia deterministic.

Ejemplo:

```text
Generated/Explicit Override
        >
Programmatic Mapping
        >
Attributes
        >
Conventions
```

La política exacta deberá documentarse/configurarse.

---

# 78. Convention Mapping

VoltStack puede ofrecer ergonomía mediante convenciones.

Ejemplo:

```text
author
→ author_id
```

pero la convención se resuelve durante metadata compilation.

Nunca durante cada query.

---

# 79. Convention ≠ Runtime Guess

Regla:

> **Las convenciones son una fuente de metadata, no un sustituto permanente de metadata compilada.**

---

# 80. Relationship Metadata Compilation Pipeline

```text
Entity Mapping Sources
        ↓
Relationship Discovery
        ↓
Relationship Definition
        ↓
Normalization
        ↓
Cross-Entity Resolution
        ↓
Validation
        ↓
Inverse Resolution
        ↓
Join Resolution
        ↓
Policy Compilation
        ↓
Fingerprint
        ↓
Immutable RelationshipMetadata
        ↓
RelationshipMetadataRegistry
```

---

# 81. Relationship Definition

Antes de compilar podrá existir:

```php
final class RelationshipDefinition
{
    // mutable only during bootstrap/build phase
}
```

---

# 82. Definition ≠ Metadata

```text
RelationshipDefinition
=
build-time representation

RelationshipMetadata
=
compiled runtime representation
```

---

# 83. Mutable Build Phase

Definitions pueden ser mutables mientras:

```text
bootstrap
package extensions
configuration
```

se están aplicando.

---

# 84. Freeze Boundary

Después de compilación:

```text
mutable definition
      ↓
freeze
      ↓
immutable metadata
```

---

# 85. RelationshipMetadata Interface

Propuesta:

```php
interface RelationshipMetadata
{
    public function id(): RelationshipId;

    public function kind(): RelationshipKind;

    public function cardinality(): RelationshipCardinality;

    public function source(): EntityType;

    public function target(): RelationshipTarget;

    public function ownership(): RelationshipOwnership;

    public function direction(): RelationshipDirection;

    public function fetchMode(): RelationshipFetchMode;

    public function cascade(): CascadePolicy;

    public function join(): RelationshipJoinMetadata;
}
```

---

# 86. Specialized Metadata

No conviene colocar todos los campos posibles en una mega clase nullable.

Preferido:

```text
RelationshipMetadata
├── OneToOneRelationshipMetadata
├── OneToManyRelationshipMetadata
├── ManyToOneRelationshipMetadata
├── ManyToManyRelationshipMetadata
├── PolymorphicToOneRelationshipMetadata
├── PolymorphicToManyRelationshipMetadata
└── PolymorphicManyToManyRelationshipMetadata
```

---

# 87. Avoid Nullable Metadata Blob

Anti-pattern:

```php
final class RelationshipMetadata
{
    public ?string $pivotTable;
    public ?string $typeColumn;
    public ?string $inverse;
    public ?string $foreignKey;
    public ?array $morphTypes;
    // ...
}
```

Esto permite combinaciones inválidas.

---

# 88. Make Invalid States Unrepresentable

Preferido:

```text
ManyToManyMetadata
requires JoinTableMetadata

ManyToOneMetadata
requires ForeignKeyJoinMetadata

PolymorphicToOneMetadata
requires PolymorphicJoinMetadata
```

---

# 89. Shared Metadata Components

Los componentes comunes pueden ser value objects:

```text
RelationshipId
RelationshipTarget
FetchPolicy
CascadePolicy
OrderingPolicy
FilterPolicy
InverseBinding
CollectionPolicy
```

---

# 90. Relationship Registry

Propuesta:

```php
interface RelationshipMetadataRegistry
{
    public function get(
        RelationshipId $id
    ): RelationshipMetadata;

    /**
     * @return list<RelationshipMetadata>
     */
    public function forEntity(
        EntityType $entity
    ): array;

    public function has(
        RelationshipId $id
    ): bool;
}
```

---

# 91. Registry By Source

Lookup principal:

```text
EntityType
→ relationships
```

---

# 92. Registry By Target

También puede existir índice inverso:

```text
Target EntityType
→ relationships referencing target
```

útil para:

- delete planning;
- diagnostics;
- dependency analysis;
- polymorphic reverse lookup.

---

# 93. Registry Indexes

Conceptualmente:

```text
byRelationshipId
bySourceEntityType
byTargetEntityType
byKind
```

Todos immutable después del bootstrap.

---

# 94. Registry ≠ Service Locator

Solo resuelve metadata.

No:

```text
EntityManager
Repository
Connection
QueryExecutor
```

---

# 95. Metadata Generation

```php
final readonly class RelationshipMetadataGeneration
{
    public function __construct(
        public int|string $value,
    ) {}
}
```

---

# 96. Generation Purpose

Permite detectar:

```text
compiled hydration plan
compiled entity query plan
metadata cache
```

creados contra metadata antigua.

---

# 97. Metadata Fingerprint

Cada relación tendrá fingerprint deterministic.

---

# 98. Fingerprint Formula

Conceptualmente:

```text
RelationshipFingerprint
=
H(
    RelationshipId,
    Kind,
    Source,
    Target,
    Ownership,
    Direction,
    Nullability,
    JoinModel,
    FetchPolicy,
    CascadePolicy,
    OrphanPolicy,
    CollectionPolicy,
    Ordering,
    Filters,
    InverseBinding,
    Extensions,
    CompilerVersion
)
```

---

# 99. Runtime Values Not in Fingerprint

No incluir:

```text
entity IDs
tenant IDs
request IDs
loaded collection state
current EntityManager
```

---

# 100. Deterministic Compilation

Mismo input lógico + mismas generaciones:

```text
same fingerprint
```

---

# 101. Metadata Cacheability

```text
Cacheable
⇔
Immutable
∧
Deterministic
∧
NoScopedMutableState
```

---

# 102. Process Sharing

Compiled relationship metadata podrá compartirse entre requests en FrankenPHP.

---

# 103. Metadata Must Not Hold Entities

Nunca:

```php
$metadata->currentOwner = $post;
```

---

# 104. Metadata Must Not Hold EntityManager

Nunca:

```php
$metadata->entityManager = $manager;
```

---

# 105. Metadata Must Not Hold Tenant Runtime Object

Tenant-specific routing pertenece al DatabaseContext/runtime.

---

# 106. Tenant-Specific Mapping

Si una arquitectura futura requiere mapping distinto por tenant, deberá existir una generación/namespace de metadata explícita.

No mutación del mismo objeto global.

---

# 107. Relationship Resolution

`EntityMetadata` deberá poder resolver:

```php
$postMetadata->relationship('author');
```

a metadata compilada.

---

# 108. EntityMetadata Integration

Puede existir:

```php
final readonly class EntityRelationshipSet
{
    /**
     * @param array<string, RelationshipId> $relationships
     */
    public function __construct(
        public array $relationships,
    ) {}
}
```

---

# 109. Avoid Circular Object Graph Metadata

Para facilitar cache/serialization:

```text
EntityMetadata
→ RelationshipId

RelationshipMetadata
→ EntityTypeId
```

puede ser preferible a enormes referencias circulares entre objetos metadata.

---

# 110. Stable Metadata IDs

El uso de IDs/value objects facilita:

- serialization;
- cache;
- fingerprinting;
- debugging;
- generation validation.

---

# 111. Entity Query Integration

Consulta:

```php
Post::query()
    ->where('author.name', 'John');
```

requiere:

```text
Post metadata
      ↓
Post.author RelationshipMetadata
      ↓
target User
      ↓
User.name FieldMetadata
      ↓
Semantic Relationship Path
```

---

# 112. Relationship Path

Propuesta:

```php
final readonly class RelationshipPath
{
    /**
     * @param list<RelationshipId> $segments
     */
    public function __construct(
        public array $segments,
    ) {}
}
```

---

# 113. Relationship Path ≠ SQL Join

El path:

```text
Post.author.organization
```

es semántico.

El planner decide representación física.

---

# 114. Join Resolution

Metadata proporciona:

```text
how association is represented
```

pero Query Planner decide:

```text
whether/how to join
```

---

# 115. Query Optimizer

Optimizer podrá utilizar metadata para conocer:

- cardinality;
- nullability;
- uniqueness;
- inverse semantics.

Pero no deberá modificar metadata.

---

# 116. Hydration Integration

HydrationPlanCompiler consume relationship metadata para construir:

```text
RelationshipHydrationBinding
```

---

# 117. RelationshipHydrationBinding

Conceptualmente:

```php
final readonly class RelationshipHydrationBinding
{
    public function __construct(
        public RelationshipId $relationship,
        public RelationshipHydrationMode $mode,
        public HydrationNodeId $targetNode,
        public RelationshipAssemblyPlan $assembly,
    ) {}
}
```

---

# 118. Hydration Metadata ≠ Runtime Assembly

Metadata dice:

```text
Post.comments = OneToMany
```

Hydration session decide:

```text
row #17 adds Comment#5 to Post#42.comments
```

---

# 119. Loaded State

`LoadedFieldMask` y collection completeness son runtime state.

No se almacenan en RelationshipMetadata.

---

# 120. Relationship Assembly

Metadata podrá producir/ayudar a compilar:

```text
RelationshipAssemblyPlan
```

pero no ejecuta la assembly.

---

# 121. UnitOfWork Integration

UoW usa metadata para saber:

```text
is to-one?
is to-many?
owning side?
cascade persist?
orphan removal?
membership identity?
```

---

# 122. UoW Does Not Rediscover Mapping

Nunca:

```text
if property is array → probably one-to-many
```

---

# 123. Change Tracking Integration

Relationship metadata determina qué propiedades participan en:

```text
relationship snapshots
relationship changesets
```

---

# 124. To-One Snapshot

Podrá guardar:

```text
EntityKey|null
```

---

# 125. To-Many Snapshot

Podrá guardar membership baseline basado en:

```text
EntityKey
```

o `MembershipKey` para many-to-many.

---

# 126. Snapshot ≠ Metadata

Metadata define cómo interpretar snapshot.

Snapshot contiene estado runtime.

---

# 127. Persistence Planner Integration

Planner utiliza:

```text
ownership
join bindings
cascade
orphan removal
target identity
nullability
```

---

# 128. Planner Does Not Generate SQL

Relationship metadata tampoco.

El resultado será:

```text
PersistenceOperation
→ Query Model
```

---

# 129. Many-to-One Persistence

Metadata:

```text
Post.author
owning
FK Post.author_id → User.id
```

permite derivar semantic operation:

```text
SetRelationshipReference(
    Post#10,
    Post.author,
    User#5
)
```

---

# 130. One-to-Many Persistence

Si `User.posts` es inverse de `Post.author`:

```text
add Post to User.posts
```

deberá reconciliarse con owning side según relationship synchronization policy.

---

# 131. Relationship Synchronization Policy

Puede existir:

```php
enum RelationshipSynchronizationPolicy
{
    case OWNING_SIDE_AUTHORITATIVE;
    case SYNCHRONIZE_BOTH_SIDES;
    case STRICT_BIDIRECTIONAL;
}
```

---

# 132. Recommended V1

```text
owning side authoritative
+
optional helper synchronization
```

es más predecible.

---

# 133. In-Memory Inverse Synchronization

Actualizar ambos lados en memoria es distinto de determinar persistencia.

---

# 134. Bidirectional Consistency

Ejemplo inconsistente:

```text
$post->author = $userA

$userB->posts contains $post
```

Diagnostics/UoW podrán detectarlo según policy.

---

# 135. Strict Policy

Podría fallar antes del flush.

---

# 136. Balanced Policy

Podría advertir y usar owning side.

---

# 137. No Silent Ambiguity

No deberán persistirse decisiones contradictorias sin política definida.

---

# 138. Ordering Metadata

To-many podrá declarar:

```php
final readonly class RelationshipOrdering
{
    /**
     * @param list<RelationshipOrderTerm> $terms
     */
    public function __construct(
        public array $terms,
    ) {}
}
```

---

# 139. Default Ordering ≠ Guaranteed DB Order

Sin ordering explícito:

```text
collection order
```

no deberá depender accidentalmente del orden físico retornado por DB.

---

# 140. Relationship Order Term

```php
final readonly class RelationshipOrderTerm
{
    public function __construct(
        public FieldPath $field,
        public SortDirection $direction,
        public ?NullOrdering $nulls,
    ) {}
}
```

---

# 141. Ordering Validation

El field deberá pertenecer a un path válido dentro del target semantics.

---

# 142. Filtering Metadata

Una relación podrá tener filtro estructural.

Ejemplo:

```text
User.activeSessions
```

con:

```text
Session.active = true
```

---

# 143. RelationshipFilter

Debe ser representación semántica, no SQL raw.

---

# 144. Filter ≠ Global Scope

Relationship filter pertenece a la relación.

Global scope pertenece al Entity Query environment.

Ambos pueden combinarse downstream.

---

# 145. Filter and Collection Completeness

Una colección filtrada puede ser COMPLETE respecto de su definición de relación.

Ejemplo:

```text
User.activeSessions
```

si `active = true` forma parte permanente de metadata.

---

# 146. Ad-Hoc Filter

Pero:

```php
$user->sessions()
    ->where('ip', '...')
```

no convierte `User.sessions` en COMPLETE.

---

# 147. Structural Filter vs Query Filter

```text
Structural Relationship Filter
≠
Ad-hoc Query Filter
```

---

# 148. Static Relationship Constraints

Toda restricción estructural deberá ser:

- deterministic;
- serializable/compilable;
- query-model based;
- free of request state.

---

# 149. Request-Dependent Relationship Filter

Evitar:

```text
relationship filter captures current user
```

dentro de metadata global.

---

# 150. Contextual Policies

Si se requieren filtros contextuales:

```text
tenant
authorization
soft delete
temporal context
```

se aplican mediante QueryContext/policy layers.

No mutando metadata.

---

# 151. Soft Delete Integration

Relationship metadata puede declarar que target soporta soft delete.

Pero incluir/excluir deleted rows depende de query/loading policy.

---

# 152. Temporal Integration

Igualmente:

```text
temporal relationship semantics
```

podrán extender metadata posteriormente sin incrustar timestamps runtime.

---

# 153. Relationship Loading Requirements

Propuesta:

```php
final readonly class RelationshipLoadingRequirements
{
    public function __construct(
        public bool $requiresOwnerIdentity,
        public bool $requiresTargetIdentity,
        public bool $batchable,
        public bool $supportsExtraLazy,
        public bool $mayRequireGrouping,
    ) {}
}
```

---

# 154. Requirements ≠ Loader

Metadata describe capacidades/requisitos.

Loader realiza operación.

---

# 155. Batchability

No toda relación personalizada tiene que ser batchable.

Debe declararse/derivarse durante compilation.

---

# 156. Persistence Requirements

```php
final readonly class RelationshipPersistenceRequirements
{
    public function __construct(
        public bool $requiresOwnerIdentity,
        public bool $requiresTargetIdentity,
        public bool $requiresJoinTableMutation,
        public bool $requiresOrphanAnalysis,
    ) {}
}
```

---

# 157. Generated IDs

Metadata permite al planner conocer que:

```text
relationship persistence
requires target identity
```

El planner crea dependency barrier.

---

# 158. Relationship Capability Model

Puede existir:

```php
final readonly class RelationshipCapabilities
{
    public function __construct(
        public bool $lazyLoadable,
        public bool $eagerLoadable,
        public bool $batchLoadable,
        public bool $extraLazy,
        public bool $cascadePersist,
        public bool $cascadeRemove,
        public bool $orphanRemoval,
    ) {}
}
```

---

# 159. Capabilities Derived

Preferiblemente capabilities se derivan de metadata validada en compilation.

No se configuran redundantemente si pueden inferirse.

---

# 160. Validation Architecture

```text
Relationship Definition
        ↓
Local Validation
        ↓
Entity Resolution
        ↓
Cross-Relationship Validation
        ↓
Join Validation
        ↓
Inverse Validation
        ↓
Policy Validation
        ↓
Capability Validation
        ↓
Compile
```

---

# 161. Local Validation

Comprueba:

- relationship name;
- kind;
- source;
- target declaration;
- basic policy combinations.

---

# 162. Entity Resolution Validation

Comprueba:

```text
source exists
target exists
target is entity
```

---

# 163. Inverse Validation

Para:

```text
Post.author ↔ User.posts
```

validará:

```text
source/target compatibility
cardinality compatibility
ownership compatibility
inverse symmetry
```

---

# 164. Cardinality Compatibility

Ejemplo válido:

```text
ManyToOne ↔ OneToMany
```

Ejemplo potencialmente inválido:

```text
ManyToOne ↔ ManyToMany
```

---

# 165. One-to-One Compatibility

```text
OneToOne ↔ OneToOne
```

con ownership definido.

---

# 166. Many-to-Many Compatibility

```text
ManyToMany ↔ ManyToMany
```

referenciando la misma asociación lógica/join table.

---

# 167. Polymorphic Validation

Usará reglas del documento 147:

```text
known morph aliases
allowed target set
identifier completeness
registry generation
```

---

# 168. Join Validation

Comprueba:

- field existence;
- compatible types;
- identifier compatibility;
- duplicate mappings;
- composite binding completeness.

---

# 169. Type Compatibility

No necesariamente requiere tipos PHP idénticos.

Debe existir compatibilidad mediante Database Type System.

---

# 170. Example

```text
User.id = Uuid
Post.author_id = UUID database representation
```

puede ser compatible.

---

# 171. Nullability Validation

Puede comprobar coherencia ORM interna.

La coherencia con DB schema se verifica mediante schema validation/integration.

---

# 172. Cascade Validation

Combinaciones peligrosas podrán generar:

```text
ERROR
WARNING
DIAGNOSTIC
```

según severidad.

---

# 173. Orphan Validation

`orphanRemoval` sobre una relación compartida deberá fallar o requerir override explícito.

---

# 174. Fetch Validation

`EXTRA_LAZY` solo será válido cuando la relación tenga representación consultable sin inicialización completa.

---

# 175. Ordering Validation

Paths deberán ser resolubles semánticamente.

---

# 176. Structural Filter Validation

Filtros deberán ser compilables al Query Model.

No closures arbitrarios que capturen runtime state.

---

# 177. Error Taxonomy

```text
RelationshipMetadataException
├── RelationshipDefinitionException
├── DuplicateRelationshipException
├── UnknownRelationshipException
├── InvalidRelationshipKindException
├── InvalidRelationshipTargetException
├── RelationshipCardinalityException
├── RelationshipOwnershipException
├── RelationshipInverseException
├── RelationshipJoinException
├── RelationshipJoinTypeException
├── RelationshipNullabilityException
├── RelationshipCascadeException
├── RelationshipOrphanPolicyException
├── RelationshipFetchPolicyException
├── RelationshipCollectionPolicyException
├── RelationshipOrderingException
├── RelationshipFilterException
├── RelationshipCompilationException
├── RelationshipFingerprintException
├── RelationshipGenerationMismatchException
├── RelationshipExtensionConflictException
├── RelationshipRuntimeIsolationException
└── RelationshipMetadataInvariantException
```

---

# 178. Duplicate Relationship

Dentro de un EntityType:

```text
two mappings named "author"
```

deberán fallar durante bootstrap.

---

# 179. Unknown Relationship

Query:

```text
Post.unknownRelation
```

deberá fallar durante semantic resolution.

No llegar al SQL Compiler.

---

# 180. Compilation Architecture

Propuesta:

```php
interface RelationshipMetadataCompiler
{
    public function compile(
        RelationshipDefinition $definition,
        RelationshipCompilationContext $context,
    ): RelationshipMetadata;
}
```

---

# 181. Compilation Context

Puede contener:

```text
EntityMetadataRegistry
TypeRegistry
MorphTypeRegistry
RelationshipDefinitionRegistry
ExtensionRegistry
CompilerVersion
```

pero no:

```text
EntityManager
Connection
current tenant
current request
```

---

# 182. Two-Pass Compilation

Las relaciones bidireccionales hacen conveniente:

```text
Pass 1:
compile entity-local relationship definitions

Pass 2:
resolve cross-entity/inverse references
```

---

# 183. Multi-Pass Model

Pipeline más robusto:

```text
Discover
   ↓
Normalize
   ↓
Assign IDs
   ↓
Resolve Entity Targets
   ↓
Resolve Inverses
   ↓
Validate Joins
   ↓
Compile Policies
   ↓
Freeze
```

---

# 184. Cyclic Metadata Graphs

Ejemplo:

```text
User.posts → Post
Post.author → User
```

no debe causar recursión infinita durante compilation.

IDs permiten resolver ciclos.

---

# 185. Registry Freeze

Una vez completado:

```text
RelationshipMetadataRegistry::freeze()
```

conceptualmente impedirá mutaciones.

---

# 186. Package Extensions

Un paquete podrá contribuir:

- relationship mapping sources;
- metadata validators;
- metadata extension blocks;
- custom relationship kind, si se habilita;
- custom loading/persistence capability.

---

# 187. Extension Boundary

Una extensión no podrá:

- ejecutar SQL durante metadata compilation;
- almacenar EntityManager en metadata;
- capturar request state;
- modificar metadata ya frozen;
- romper IdentityMap semantics.

---

# 188. RelationshipMetadataExtension

Propuesta:

```php
interface RelationshipMetadataExtension
{
    public function id(): string;

    public function compile(
        RelationshipDefinition $definition,
        RelationshipCompilationContext $context,
    ): ?RelationshipMetadataContribution;
}
```

---

# 189. Extension Contributions

Deben ser:

```text
immutable
deterministic
fingerprinted
validated
```

---

# 190. Extension Conflicts

Dos extensiones que asignen valores incompatibles deberán producir error determinista.

No:

```text
last plugin silently wins
```

---

# 191. Metadata Serialization

Compiled metadata deberá ser apta para cache precompilado.

---

# 192. Serialization Requirements

Evitar objetos no serializables como:

```text
live ReflectionProperty
Closure capturing container
PDO
EntityManager
Request
```

---

# 193. Compiled Accessors

Si se utilizan accessors optimizados:

```text
CompiledPropertyReaderId
CompiledPropertyWriterId
```

podrán referenciar un registry immutable de accessors.

---

# 194. Reflection-Free Hot Path

Objetivo:

```text
ReflectionCallsPerRelationshipOperation ≈ 0
```

---

# 195. Metadata Lookup Complexity

Lookup por `RelationshipId` deberá aproximarse a:

```text
O(1)
```

---

# 196. Memory Complexity

Para `R` relaciones:

```text
Mmetadata ≈ O(R)
```

con índices secundarios controlados.

---

# 197. Metadata Deduplication

Value objects immutable compartibles pueden reutilizarse cuando sea seguro:

```text
CascadePolicy
FetchPolicy
empty ordering
```

---

# 198. Premature Optimization

No deberá sacrificarse claridad semántica por ahorrar pequeños objetos durante V1.

---

# 199. Diagnostics

Debe ser posible inspeccionar:

```text
RELATIONSHIP METADATA

ID:
    Post.author

Kind:
    MANY_TO_ONE

Source:
    Post

Target:
    User

Cardinality:
    TO_ONE

Ownership:
    OWNING

Direction:
    BIDIRECTIONAL

Inverse:
    User.posts

Nullable:
    NO

Fetch:
    LAZY

Cascade:
    persist

Join:
    Post.author_id → User.id
```

---

# 200. Many-to-Many Diagnostics

```text
RELATIONSHIP METADATA

ID:
    User.roles

Kind:
    MANY_TO_MANY

Source:
    User

Target:
    Role

Join Table:
    role_user

Owner Bindings:
    role_user.user_id → User.id

Inverse Bindings:
    role_user.role_id → Role.id

Inverse:
    Role.users

Collection:
    SET_LIKE
```

---

# 201. Polymorphic Diagnostics

```text
RELATIONSHIP METADATA

ID:
    Comment.commentable

Kind:
    POLYMORPHIC_TO_ONE

Targets:
    post  → Post
    video → Video

Type Binding:
    commentable_type

Identifier:
    commentable_id

Unknown Type:
    FAIL
```

---

# 202. Explainability

Metadata diagnostics no deberá mostrar:

- credentials;
- entity field values;
- current user;
- tenant secrets.

---

# 203. Telemetry

Metadata compilation podrá medir:

```text
database.orm.metadata.relationships
database.orm.metadata.relationship_compile_duration
database.orm.metadata.relationship_validation_failures
database.orm.metadata.relationship_cache_hits
database.orm.metadata.relationship_cache_misses
```

---

# 204. Low Cardinality

No etiquetar métricas con cada `RelationshipId` en instalaciones enormes salvo telemetry policy explícita.

---

# 205. Boot-Time Fail Fast

Errores estructurales deberán detectarse preferentemente durante bootstrap:

```text
duplicate relationship
unknown target
invalid inverse
invalid join
invalid cascade combination
unknown morph alias
```

---

# 206. Development vs Production

Development puede proporcionar diagnostics extensos.

Production puede utilizar metadata precompilada/cacheada.

Las invariantes serán iguales.

---

# 207. Metadata Precompilation

VoltStack podrá ofrecer:

```text
database:metadata:compile
```

en el futuro.

---

# 208. Precompiled Artifact

Conceptualmente:

```text
EntityMetadata
+
RelationshipMetadata
+
Type Metadata
+
Mapping Generation Vector
```

---

# 209. Generation Vector

Propuesta:

```php
final readonly class RelationshipMetadataGenerationVector
{
    public function __construct(
        public EntityMetadataGeneration $entity,
        public TypeRegistryGeneration $types,
        public RelationshipMetadataGeneration $relationships,
        public ?MorphRegistryGeneration $morphs,
        public ExtensionGeneration $extensions,
        public MetadataCompilerVersion $compiler,
    ) {}
}
```

---

# 210. Plan Compatibility

Un HydrationPlan generado con:

```text
RelationshipGeneration = 15
```

no deberá reutilizarse silenciosamente contra:

```text
RelationshipGeneration = 16
```

si el plan depende de dicha metadata.

---

# 211. Generation Compatibility Formula

```text
PlanCompatible
⇔
RequiredMetadataGeneration
is compatible with
CurrentMetadataGeneration
```

---

# 212. Immutable Generations

Una generación existente no se muta.

Se crea una nueva.

---

# 213. Hot Reload

En development:

```text
mapping changes
      ↓
new metadata generation
```

Los scopes existentes podrán finalizar con su generación original según runtime policy.

---

# 214. No Mid-Scope Metadata Mutation

Un mismo PersistenceContext no deberá observar:

```text
relationship metadata V1
```

y posteriormente:

```text
same relationship object mutated into V2
```

---

# 215. Persistent Runtime Model

```text
Process Shared
──────────────────────────────
EntityMetadata
RelationshipMetadata
Metadata Registry
Type Registry
Morph Registry
Compiled Accessors
Fingerprints

Scope Local
──────────────────────────────
EntityManager
IdentityMap
UnitOfWork
Snapshots
Relationship State
Collections
Lazy Load State
Batch Load State
HydrationSession
```

---

# 216. FrankenPHP

La metadata immutable puede permanecer caliente entre requests.

Esto es deseable.

---

# 217. RoadRunner

Misma regla:

```text
shared immutable definitions
+
scoped mutable persistence state
```

---

# 218. OpenSwoole

La metadata puede compartirse si es immutable/thread/coroutine-safe.

El runtime ORM mutable no.

---

# 219. Concurrency

Dos requests pueden leer simultáneamente la misma metadata.

No deberán modificarla.

---

# 220. Metadata Thread Safety

```text
Immutable Metadata
→ naturally read-concurrent
```

si sus dependencias también son immutable.

---

# 221. No Lazy Mutation

Evitar:

```php
public function target(): EntityMetadata
{
    return $this->target ??= resolve(...);
}
```

sobre objeto compartido mutable.

---

# 222. Pre-Resolution Preferred

Preferir referencias/IDs compilados o caches immutable construidos antes del freeze.

---

# 223. Relationship Metadata and Security

Metadata es información estructural interna.

No debe exponerse automáticamente a clientes HTTP.

---

# 224. Dynamic Relationship Names

Input externo:

```text
?include=author
```

deberá validarse contra una allowlist/API policy.

Que exista metadata no significa que sea públicamente navegable.

---

# 225. Relationship Metadata ≠ Authorization

Metadata puede decir:

```text
User.roles exists
```

pero no:

```text
current actor may read roles
```

---

# 226. Security Boundary

```text
Metadata validity
≠
Access permission
```

---

# 227. Testing Architecture

La suite deberá incluir:

```text
relationship identity
kind
cardinality
ownership
direction
target resolution
inverse resolution
join compilation
composite joins
nullability
fetch modes
cascade
orphan removal
collection semantics
ordering
structural filters
polymorphic metadata
registry
fingerprints
generation compatibility
extensions
serialization
persistent runtime
concurrency
```

---

# 228. Test — Deterministic Fingerprint

Mismo mapping:

```text
compile A
compile B
```

deberá producir:

```text
Fingerprint(A) = Fingerprint(B)
```

---

# 229. Test — Different Join

Cambiar:

```text
author_id
```

por:

```text
owner_id
```

deberá modificar fingerprint.

---

# 230. Test — Different Cascade

Cambiar cascade policy deberá modificar fingerprint.

---

# 231. Test — Runtime Values

Cambiar:

```text
current tenant
current request
entity ID
```

no deberá modificar fingerprint global.

---

# 232. Test — Invalid Inverse

```text
Post.author → User.posts

User.posts → Comment
```

deberá fallar.

---

# 233. Test — Duplicate Relationship

Dos definiciones para:

```text
Post.author
```

deberán fallar.

---

# 234. Test — Composite Join

Todos los componentes deberán preservarse y validarse.

---

# 235. Test — Polymorphic Generation

Cambio en Morph Registry deberá invalidar metadata/planes dependientes según generation vector.

---

# 236. Test — No Reflection Hot Path

Operaciones repetidas sobre relaciones no deberán volver a descubrir atributos.

---

# 237. Test — Persistent Worker

Dos requests reutilizan la misma metadata immutable pero no comparten:

```text
collections
snapshots
entities
load queues
```

---

# 238. Test — Concurrent Reads

Metadata deberá poder consultarse concurrentemente sin locks de mutación.

---

# 239. Test — Extension Conflict

Dos contribuciones incompatibles deberán fallar durante compilation.

---

# 240. Test — Serialization

Compiled metadata deberá poder cachearse/restaurarse sin runtime scoped objects.

---

# 241. Proposed Directory Structure

```text
src/Quantum/Database/ORM/Relationship/Metadata/
│
├── Contract/
│   ├── RelationshipMetadata.php
│   ├── RelationshipMetadataRegistry.php
│   ├── RelationshipMetadataCompiler.php
│   └── RelationshipMetadataExtension.php
│
├── Definition/
│   ├── RelationshipDefinition.php
│   ├── OneToOneDefinition.php
│   ├── OneToManyDefinition.php
│   ├── ManyToOneDefinition.php
│   ├── ManyToManyDefinition.php
│   └── PolymorphicRelationshipDefinition.php
│
├── Compiled/
│   ├── OneToOneRelationshipMetadata.php
│   ├── OneToManyRelationshipMetadata.php
│   ├── ManyToOneRelationshipMetadata.php
│   ├── ManyToManyRelationshipMetadata.php
│   ├── PolymorphicToOneRelationshipMetadata.php
│   ├── PolymorphicToManyRelationshipMetadata.php
│   └── PolymorphicManyToManyRelationshipMetadata.php
│
├── Identity/
│   ├── RelationshipId.php
│   ├── RelationshipFingerprint.php
│   ├── RelationshipMetadataGeneration.php
│   └── RelationshipMetadataGenerationVector.php
│
├── Target/
│   ├── RelationshipTarget.php
│   ├── SingleEntityTarget.php
│   └── PolymorphicEntityTargetSet.php
│
├── Join/
│   ├── RelationshipJoinMetadata.php
│   ├── ForeignKeyJoinMetadata.php
│   ├── JoinColumnBinding.php
│   ├── JoinTableMetadata.php
│   ├── PivotMetadata.php
│   └── PolymorphicJoinMetadata.php
│
├── Policy/
│   ├── RelationshipFetchMode.php
│   ├── CascadePolicy.php
│   ├── OrphanRemovalPolicy.php
│   ├── RelationshipNullability.php
│   └── RelationshipSynchronizationPolicy.php
│
├── Collection/
│   ├── RelationshipCollectionMetadata.php
│   ├── CollectionKind.php
│   ├── CollectionUniqueness.php
│   └── CollectionOrdering.php
│
├── Query/
│   ├── RelationshipOrdering.php
│   ├── RelationshipOrderTerm.php
│   └── RelationshipFilter.php
│
├── Requirement/
│   ├── RelationshipLoadingRequirements.php
│   ├── RelationshipPersistenceRequirements.php
│   └── RelationshipCapabilities.php
│
├── Registry/
│   ├── CompiledRelationshipMetadataRegistry.php
│   └── RelationshipDefinitionRegistry.php
│
├── Compilation/
│   ├── RelationshipCompilationContext.php
│   ├── RelationshipDefinitionNormalizer.php
│   ├── RelationshipTargetResolver.php
│   ├── RelationshipInverseResolver.php
│   ├── RelationshipJoinResolver.php
│   └── RelationshipFingerprintGenerator.php
│
├── Validation/
│   ├── RelationshipMetadataValidator.php
│   ├── RelationshipInverseValidator.php
│   ├── RelationshipJoinValidator.php
│   ├── RelationshipCascadeValidator.php
│   └── RelationshipCollectionValidator.php
│
├── Extension/
│   ├── RelationshipMetadataContribution.php
│   └── RelationshipExtensionRegistry.php
│
├── Diagnostics/
│   └── RelationshipMetadataDiagnostics.php
│
└── Exception/
    └── ...
```

---

# 242. Architectural Invariants

## DB-ORM-REL-META-001

Toda relación tendrá `RelationshipId` canónico.

## DB-ORM-REL-META-002

RelationshipId estará namespaced por source EntityType.

## DB-ORM-REL-META-003

Relationship metadata será immutable después de compilation.

## DB-ORM-REL-META-004

Mapping source será distinto de compiled metadata.

## DB-ORM-REL-META-005

Reflection no será el mecanismo runtime primario.

## DB-ORM-REL-META-006

Relationship metadata será distinta de Schema Model.

## DB-ORM-REL-META-007

Relationship metadata será distinta de FK metadata.

## DB-ORM-REL-META-008

Relationship metadata será distinta de runtime relationship state.

## DB-ORM-REL-META-009

Relationship metadata será distinta de HydrationPlan.

## DB-ORM-REL-META-010

Relationship metadata será distinta de PersistencePlan.

## DB-ORM-REL-META-011

Relationship kind y cardinality serán conceptos separados.

## DB-ORM-REL-META-012

Toda relación tendrá source EntityType.

## DB-ORM-REL-META-013

Relación no polimórfica tendrá target EntityType conocido.

## DB-ORM-REL-META-014

Relación polimórfica tendrá target set explícito/compilado.

## DB-ORM-REL-META-015

Ownership será distinto de domain ownership.

## DB-ORM-REL-META-016

Bidirectional relationship tendrá inverse binding explícito.

## DB-ORM-REL-META-017

Inverse relationship no será inferida en hot path.

## DB-ORM-REL-META-018

Owning side será persistently authoritative según policy.

## DB-ORM-REL-META-019

Dos lados no persistirán independientemente la misma FK.

## DB-ORM-REL-META-020

Nullability ORM será distinta de DB nullability.

## DB-ORM-REL-META-021

To-many collection no será nullable.

## DB-ORM-REL-META-022

EMPTY será distinto de UNINITIALIZED.

## DB-ORM-REL-META-023

Collection state no residirá en metadata.

## DB-ORM-REL-META-024

Fetch mode será política lógica.

## DB-ORM-REL-META-025

EAGER no implicará JOIN.

## DB-ORM-REL-META-026

LAZY metadata no ejecutará queries.

## DB-ORM-REL-META-027

EXTRA_LAZY requerirá capacidades válidas.

## DB-ORM-REL-META-028

Cascade ORM será distinto de database cascade.

## DB-ORM-REL-META-029

Cascade persist no significará immediate insert.

## DB-ORM-REL-META-030

Cascade remove no significará immediate delete.

## DB-ORM-REL-META-031

Orphan removal será distinto de cascade remove.

## DB-ORM-REL-META-032

Orphan removal requerirá semántica de ownership compatible.

## DB-ORM-REL-META-033

Join metadata será typed.

## DB-ORM-REL-META-034

Composite joins serán soportados.

## DB-ORM-REL-META-035

Join bindings tendrán ordering deterministic.

## DB-ORM-REL-META-036

Many-to-many utilizará join-table metadata explícita.

## DB-ORM-REL-META-037

Rich association podrá modelarse como entidad explícita.

## DB-ORM-REL-META-038

Polymorphic join metadata reutilizará Morph Registry semantics.

## DB-ORM-REL-META-039

Conventions serán fuentes de metadata.

## DB-ORM-REL-META-040

Conventions no serán runtime guesses.

## DB-ORM-REL-META-041

Relationship definitions podrán ser mutables solo durante build phase.

## DB-ORM-REL-META-042

Compiled metadata estará frozen.

## DB-ORM-REL-META-043

Invalid metadata combinations deberán ser irrepresentables cuando sea razonable.

## DB-ORM-REL-META-044

No se utilizará una mega metadata class nullable para todos los kinds.

## DB-ORM-REL-META-045

Registry será immutable después del bootstrap.

## DB-ORM-REL-META-046

Registry no será service locator.

## DB-ORM-REL-META-047

Registry no resolverá EntityManager.

## DB-ORM-REL-META-048

Registry podrá indexar por source.

## DB-ORM-REL-META-049

Registry podrá indexar por target.

## DB-ORM-REL-META-050

Metadata tendrá generation explícita.

## DB-ORM-REL-META-051

Metadata tendrá fingerprint deterministic.

## DB-ORM-REL-META-052

Runtime entity values no formarán parte del fingerprint.

## DB-ORM-REL-META-053

Tenant runtime values no formarán parte del fingerprint global.

## DB-ORM-REL-META-054

Metadata cacheable no contendrá mutable scoped state.

## DB-ORM-REL-META-055

Metadata no contendrá entity instances.

## DB-ORM-REL-META-056

Metadata no contendrá EntityManager.

## DB-ORM-REL-META-057

Metadata no contendrá Connection runtime.

## DB-ORM-REL-META-058

Metadata no contendrá current request.

## DB-ORM-REL-META-059

Metadata no contendrá current tenant object.

## DB-ORM-REL-META-060

Entity Query consumirá metadata mediante IDs/registry.

## DB-ORM-REL-META-061

Relationship path será distinto de SQL join.

## DB-ORM-REL-META-062

Query Planner decidirá representación física.

## DB-ORM-REL-META-063

Hydration compiler consumirá metadata.

## DB-ORM-REL-META-064

Hydration runtime state no se almacenará en metadata.

## DB-ORM-REL-META-065

LoadedFieldMask no será metadata.

## DB-ORM-REL-META-066

Collection completeness no será metadata.

## DB-ORM-REL-META-067

UoW no redescubrirá relationships mediante property inspection.

## DB-ORM-REL-META-068

Relationship snapshots serán distintos de metadata.

## DB-ORM-REL-META-069

Persistence Planner consumirá ownership/join/policy metadata.

## DB-ORM-REL-META-070

Persistence Planner no generará SQL.

## DB-ORM-REL-META-071

Relationship metadata no generará SQL.

## DB-ORM-REL-META-072

Bidirectional synchronization será policy-driven.

## DB-ORM-REL-META-073

Owning side será default authoritative.

## DB-ORM-REL-META-074

Contradictory bidirectional state no será resuelto silenciosamente sin policy.

## DB-ORM-REL-META-075

Relationship ordering será semantic metadata.

## DB-ORM-REL-META-076

No ordering implicará no guaranteed collection order.

## DB-ORM-REL-META-077

Structural relationship filters serán distintos de ad-hoc query filters.

## DB-ORM-REL-META-078

Structural filters podrán definir completeness domain.

## DB-ORM-REL-META-079

Ad-hoc filtered load no marcará colección completa automáticamente.

## DB-ORM-REL-META-080

Structural filters no capturarán request state.

## DB-ORM-REL-META-081

Contextual policies se aplicarán mediante runtime QueryContext.

## DB-ORM-REL-META-082

Loading requirements serán distintos del loader.

## DB-ORM-REL-META-083

Persistence requirements serán distintos del Persistence Engine.

## DB-ORM-REL-META-084

Generated identity dependencies serán representables.

## DB-ORM-REL-META-085

Capabilities serán derivadas de metadata cuando sea posible.

## DB-ORM-REL-META-086

Unknown target EntityType fallará durante compilation.

## DB-ORM-REL-META-087

Invalid inverse mapping fallará durante compilation.

## DB-ORM-REL-META-088

Invalid cardinality pairing fallará durante compilation.

## DB-ORM-REL-META-089

Invalid join field fallará durante compilation.

## DB-ORM-REL-META-090

Join type compatibility utilizará Database Type System.

## DB-ORM-REL-META-091

Invalid orphan policy fallará durante compilation.

## DB-ORM-REL-META-092

Invalid extra-lazy configuration fallará durante compilation.

## DB-ORM-REL-META-093

Relationship compilation no ejecutará DB queries.

## DB-ORM-REL-META-094

Compilation context no contendrá current EntityManager.

## DB-ORM-REL-META-095

Cross-entity resolution podrá utilizar multi-pass compilation.

## DB-ORM-REL-META-096

Cyclic metadata graphs no causarán infinite compilation recursion.

## DB-ORM-REL-META-097

Package extensions contribuirán antes del freeze.

## DB-ORM-REL-META-098

Extensions no modificarán frozen metadata.

## DB-ORM-REL-META-099

Extension contributions serán deterministic.

## DB-ORM-REL-META-100

Extension contributions serán fingerprinted.

## DB-ORM-REL-META-101

Extension conflicts no utilizarán silent last-write-wins.

## DB-ORM-REL-META-102

Compiled metadata será serializable/cacheable cuando sea posible.

## DB-ORM-REL-META-103

Compiled metadata no conservará live ReflectionProperty como requisito hot-path.

## DB-ORM-REL-META-104

Reflection calls por relationship operation tenderán a cero.

## DB-ORM-REL-META-105

Relationship lookup por ID tenderá a O(1).

## DB-ORM-REL-META-106

Metadata diagnostics no expondrá sensitive values.

## DB-ORM-REL-META-107

Structural errors deberán fallar temprano.

## DB-ORM-REL-META-108

Production podrá utilizar metadata precompilada.

## DB-ORM-REL-META-109

Development y production compartirán invariantes.

## DB-ORM-REL-META-110

Generation vector gobernará compatibilidad con planes compilados.

## DB-ORM-REL-META-111

Metadata generation existente no será mutada in-place.

## DB-ORM-REL-META-112

Hot reload creará nueva generation.

## DB-ORM-REL-META-113

PersistenceContext no observará metadata mutada a mitad de scope.

## DB-ORM-REL-META-114

Immutable metadata podrá compartirse entre requests.

## DB-ORM-REL-META-115

IdentityMap permanecerá scope-local.

## DB-ORM-REL-META-116

UnitOfWork permanecerá scope-local.

## DB-ORM-REL-META-117

Relationship collections permanecerán scope-local.

## DB-ORM-REL-META-118

Lazy loading queues permanecerán scope-local.

## DB-ORM-REL-META-119

Batch loading state permanecerá scope-local.

## DB-ORM-REL-META-120

Metadata será segura para FrankenPHP bajo immutability.

## DB-ORM-REL-META-121

Metadata será reusable en RoadRunner bajo las mismas garantías.

## DB-ORM-REL-META-122

Metadata será read-concurrent en OpenSwoole cuando dependencias sean immutable.

## DB-ORM-REL-META-123

Shared metadata no utilizará lazy mutable resolution.

## DB-ORM-REL-META-124

Dynamic relationship input externo requerirá API validation.

## DB-ORM-REL-META-125

Metadata existence no implicará authorization.

## DB-ORM-REL-META-126

Relationship metadata no se expondrá automáticamente a HTTP clients.

## DB-ORM-REL-META-127

Polymorphic aliases permanecerán controlados por Morph Registry.

## DB-ORM-REL-META-128

Relationship metadata no convertirá DB discriminator en class name.

## DB-ORM-REL-META-129

Schema validation podrá comparar ORM relationship metadata con physical schema.

## DB-ORM-REL-META-130

Schema validation no modificará metadata silenciosamente.

## DB-ORM-REL-META-131

Database introspection no será requisito para cada metadata lookup.

## DB-ORM-REL-META-132

Metadata compilation será deterministic respecto de sus inputs.

## DB-ORM-REL-META-133

Same logical metadata producirá same fingerprint.

## DB-ORM-REL-META-134

Semantic metadata change deberá alterar fingerprint cuando afecte runtime behavior.

## DB-ORM-REL-META-135

Query parameter values no afectarán relationship metadata fingerprint.

## DB-ORM-REL-META-136

Entity instance state no afectará relationship metadata fingerprint.

## DB-ORM-REL-META-137

Fetch strategy será parte de metadata semantics.

## DB-ORM-REL-META-138

Cascade strategy será parte de metadata semantics.

## DB-ORM-REL-META-139

Orphan policy será parte de metadata semantics.

## DB-ORM-REL-META-140

Join representation será parte de metadata semantics.

## DB-ORM-REL-META-141

Inverse binding será parte de metadata semantics.

## DB-ORM-REL-META-142

Collection semantics serán parte de metadata semantics.

## DB-ORM-REL-META-143

Polymorphic target restrictions serán parte de metadata semantics.

## DB-ORM-REL-META-144

Runtime relationship changes nunca modificarán compiled metadata.

## DB-ORM-REL-META-145

Metadata registry será reemplazado por generación, no mutado arbitrariamente.

## DB-ORM-REL-META-146

Relationship metadata será la única fuente ORM canónica después de compilation.

## DB-ORM-REL-META-147

Hydrators no redescubrirán relationship mapping.

## DB-ORM-REL-META-148

Loaders no redescubrirán relationship mapping.

## DB-ORM-REL-META-149

UnitOfWork no redescubrirá relationship mapping.

## DB-ORM-REL-META-150

Persistence Engine no redescubrirá relationship mapping.

## DB-ORM-REL-META-151

Entity Query no redescubrirá relationship mapping.

## DB-ORM-REL-META-152

Compiler SQL permanecerá independiente de ORM metadata runtime.

## DB-ORM-REL-META-153

Driver permanecerá independiente de ORM relationship metadata.

## DB-ORM-REL-META-154

Connection permanecerá independiente de ORM relationship metadata.

## DB-ORM-REL-META-155

Relationship metadata deberá preservar separation of concerns.

## DB-ORM-REL-META-156

Relationship metadata deberá ser inspectable.

## DB-ORM-REL-META-157

Relationship metadata deberá ser testable sin conexión a DB.

## DB-ORM-REL-META-158

Relationship metadata deberá ser extension-safe.

## DB-ORM-REL-META-159

Relationship metadata deberá ser persistent-runtime-safe.

## DB-ORM-REL-META-160

Invalid relationship semantics deberán fallar antes del hot path siempre que puedan determinarse durante bootstrap.

---

# 243. Anti-Patterns

## 243.1 Reflection por cada acceso

```php
$reflection = new ReflectionProperty(
    $entity,
    'author'
);
```

en cada operación ORM.

**Rechazado.**

---

## 243.2 SQL dentro de metadata

```php
$relationship->joinSql();
```

**Rechazado.**

Debe producirse semántica estructural, no SQL dialect-specific.

---

## 243.3 EntityManager dentro de metadata

```php
$metadata->manager = $entityManager;
```

**Rechazado.**

---

## 243.4 Current Tenant dentro de metadata

```php
$metadata->tenant = TenantContext::current();
```

**Rechazado.**

---

## 243.5 Closures capturando request state

```php
filter: fn ($query) =>
    $query->where('tenant_id', currentTenant()->id)
```

dentro de metadata global.

**Rechazado.**

---

## 243.6 Guessing inverse names

```text
author
→ guess authors/posts/user
```

**Rechazado en runtime.**

---

## 243.7 Nullable Mega Metadata Object

Una clase con decenas de propiedades opcionales incompatibles.

**Rechazado.**

---

## 243.8 Treating EAGER as JOIN

```text
EAGER = always JOIN
```

**Rechazado.**

---

## 243.9 Treating Cascade as DB Cascade

```text
cascadeRemove = ON DELETE CASCADE
```

**Rechazado.**

---

## 243.10 Treating ORM Metadata as Physical Truth

```text
mapping says FK exists
→ assume database has FK
```

**Rechazado.**

---

# 244. Core Compilation Example

Mapping:

```php
#[Entity]
final class Post
{
    #[ManyToOne(
        target: User::class,
        inversedBy: 'posts',
        nullable: false,
        cascade: ['persist'],
    )]
    private User $author;
}
```

Build-time definition:

```text
RelationshipDefinition
    source       = Post
    name         = author
    kind         = MANY_TO_ONE
    target       = User
    inverse      = posts
    nullable     = false
    cascade      = persist
```

Después de resolution:

```text
RelationshipId:
    Post.author

Target:
    EntityType(User)

Inverse:
    User.posts

Join:
    Post.author_id
        →
    User.id
```

Finalmente:

```text
Immutable ManyToOneRelationshipMetadata
```

---

# 245. Runtime Example

Consulta:

```php
Post::query()
    ->with('author')
    ->get();
```

Flujo:

```text
Entity Query
    ↓
Post EntityMetadata
    ↓
RelationshipId(Post.author)
    ↓
RelationshipMetadataRegistry
    ↓
ManyToOneRelationshipMetadata
    ↓
Semantic Query Model
    ↓
Planner
    ↓
Loading Strategy
    ↓
Hydration Plan
    ↓
EntityHydrator
    ↓
Relationship Assembly
```

En ningún punto se necesita redescubrir:

```text
#[ManyToOne]
```

---

# 246. Persistence Example

```php
$post->setAuthor($newAuthor);

$entityManager->flush();
```

Flujo:

```text
Object mutation
      ↓
Change Tracking
      ↓
RelationshipMetadata(Post.author)
      ↓
Relationship ChangeSet
      ↓
UnitOfWork
      ↓
Persistence Planner
      ↓
Set Relationship Reference
      ↓
Update Query Model
      ↓
Query Engine
      ↓
Compiler
      ↓
Executor
```

---

# 247. Architectural Formula

```text
RelationshipMetadata
=
Identity
+
Relationship Kind
+
Cardinality
+
Source
+
Target
+
Ownership
+
Direction
+
Nullability
+
Join Semantics
+
Fetch Semantics
+
Cascade Semantics
+
Orphan Semantics
+
Collection Semantics
+
Ordering
+
Structural Filtering
+
Inverse Binding
+
Loading Requirements
+
Persistence Requirements
+
Capabilities
+
Generation
+
Fingerprint
+
Extensions
```

---

# 248. Compilation Formula

```text
RelationshipMetadata
=
Compile(
    Validate(
        Resolve(
            Normalize(
                RelationshipDefinition
            )
        )
    )
)
```

---

# 249. Runtime Formula

```text
RuntimeRelationshipOperation
=
ExecuteUsing(
    CompiledRelationshipMetadata,
    ScopedPersistenceContext
)
```

no:

```text
RuntimeRelationshipOperation
=
RediscoverMappingWithReflection()
```

---

# 250. Cacheability Formula

```text
RelationshipMetadataCacheable
⇔
Immutable
∧
Deterministic
∧
GenerationBound
∧
NoEntityInstances
∧
NoPersistenceContext
∧
NoRequestState
```

---

# 251. Persistent Runtime Formula

```text
SafeRelationshipRuntime
=
SharedImmutableMetadata
+
ScopedMutableRelationshipState
+
GenerationIsolation
+
DeterministicReset
```

---

# 252. Master Architectural Rule

> **Relationship Metadata es conocimiento estructural del ORM, no estado de ejecución.**

Por tanto:

```text
Shared:
    Relationship definitions compiled
    Relationship IDs
    Join bindings
    Fetch policies
    Cascade policies
    Inverse bindings
    Fingerprints

Scoped:
    actual entities
    collection contents
    loaded state
    relationship snapshots
    changesets
    lazy-load queues
    batch queues
    persistence outcomes
```

---

# 253. Resultado arquitectónico

Con este sistema, VoltStack podrá transformar:

```php
#[ManyToOne(
    target: User::class,
    inversedBy: 'posts'
)]
private User $author;
```

una sola vez durante bootstrap en:

```text
RelationshipDefinition
        ↓
Normalize
        ↓
Resolve
        ↓
Validate
        ↓
Compile
        ↓
ManyToOneRelationshipMetadata
        ↓
RelationshipMetadataRegistry
```

y posteriormente reutilizar esa representación en:

```text
Entity Query
Hydration
Lazy Loading
Eager Loading
Batch Loading
Change Tracking
UnitOfWork
Persistence Planning
Diagnostics
Schema Validation
Telemetry
```

sin acoplar esos sistemas a:

```text
Reflection
PHP Attributes
SQL
PDO
Driver
Current Request
Current Tenant
```

---

# 254. Relación con documentos anteriores

Este documento consolida las reglas definidas por:

```text
142_DATABASE_RELATIONSHIP_ARCHITECTURE.md
143_DATABASE_ONE_TO_ONE_RELATIONSHIP_SYSTEM.md
144_DATABASE_ONE_TO_MANY_RELATIONSHIP_SYSTEM.md
145_DATABASE_MANY_TO_ONE_RELATIONSHIP_SYSTEM.md
146_DATABASE_MANY_TO_MANY_RELATIONSHIP_SYSTEM.md
147_DATABASE_POLYMORPHIC_RELATIONSHIP_SYSTEM.md
```

Todos ellos convergen ahora sobre:

```text
RelationshipMetadataRegistry
```

como fuente canónica runtime.

---

# 255. Siguiente documento

```text
149_DATABASE_RELATIONSHIP_PERSISTENCE_SYSTEM.md
```

El siguiente documento deberá definir cómo:

```text
RelationshipMetadata
+
Relationship Snapshots
+
Entity State
+
Change Tracking
+
UnitOfWork
```

se convierten en:

```text
Relationship ChangeSets
        ↓
Relationship Persistence Operations
        ↓
Dependency Graph
        ↓
Persistence Planner
        ↓
Query Models
```

incluyendo:

- to-one reference changes;
- one-to-many synchronization;
- many-to-many membership changes;
- polymorphic references;
- owning/inverse side semantics;
- cascade persist/remove;
- orphan removal;
- generated identifier dependencies;
- join-table mutations;
- relationship ordering;
- partial collections;
- optimistic locking interaction;
- flush stabilization;
- execution outcomes;
- rollback;
- UNKNOWN outcomes;
- consistency reconciliation;
- persistent runtime isolation.

La regla central será:

> **Relationship Persistence transforma cambios semánticos confirmados de asociaciones en operaciones de persistencia tipadas; nunca convierte directamente mutaciones de objetos en SQL ni permite que el lado inverso cree una segunda fuente de verdad persistente.**