# 147_DATABASE_POLYMORPHIC_RELATIONSHIP_SYSTEM.md

# VoltStack Quantum Database
## Database Polymorphic Relationship System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 147 — Database Polymorphic Relationship System  
**Bloque:** 13 — Relationships  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Polymorphic Relationship System` define la arquitectura mediante la cual VoltStack podrá representar relaciones ORM cuyo destino puede pertenecer a más de un tipo de entidad.

Ejemplo clásico:

```text
Comment
   │
   └── commentable
          │
          ├── Post
          ├── Video
          └── Photo
```

Representación relacional:

```text
comments
────────────────
id
body
commentable_type
commentable_id
```

Ejemplo:

```text
id | commentable_type | commentable_id
───┼──────────────────┼───────────────
 1 | post             | 100
 2 | video            | 42
 3 | post             | 200
```

El sistema deberá soportar, como mínimo:

- polymorphic many-to-one / `morphTo`;
- polymorphic one-to-one;
- polymorphic one-to-many;
- polymorphic many-to-many;
- inverse polymorphic relationships;
- stable morph aliases;
- type registries;
- type-safe target resolution;
- composite identifiers;
- IdentityMap integration;
- hydration;
- lazy loading;
- eager loading;
- batch loading;
- persistence;
- change tracking;
- tenant/shard isolation;
- schema diagnostics;
- migrations;
- security;
- persistent runtimes.

---

# 2. Principio central

> **El tipo persistido de una relación polimórfica será un identificador lógico estable y controlado por metadata, nunca un nombre de clase PHP arbitrario obtenido directamente desde la base de datos.**

Por tanto:

```text
PersistedMorphType
≠
PHPClassName
```

Preferido:

```text
post
video
photo
```

No:

```text
App\Domain\Blog\Post
App\Models\Video
Vendor\Package\Photo
```

---

# 3. Problema arquitectónico

Una FK tradicional representa:

```text
target_id
    ↓
one known table
```

Una referencia polimórfica representa:

```text
(target_type, target_id)
        ↓
different possible entity types
```

Por tanto:

```text
PolymorphicReference
=
TypeDiscriminator
+
TargetIdentifier
```

---

# 4. Polymorphic Relationship ≠ Inheritance

Estos conceptos deberán permanecer separados.

```text
Polymorphic Relationship
≠
ORM Inheritance
```

Inheritance:

```text
Payment
├── CardPayment
└── BankPayment
```

Polymorphic relationship:

```text
Attachment
    │
    └── attachable
          ├── Invoice
          ├── User
          └── Message
```

---

# 5. Polymorphic Relationship ≠ Union Type

PHP puede declarar:

```php
private Post|Video $target;
```

pero esto no define por sí mismo:

- persistencia;
- discriminadores;
- columnas;
- aliases;
- query semantics;
- loading;
- IdentityMap behavior.

---

# 6. Polymorphic Relationship ≠ Arbitrary Dynamic Reference

VoltStack no permitirá que cualquier string almacenado en DB determine una clase PHP arbitraria.

Nunca:

```php
$class = $row['target_type'];

$entity = new $class();
```

Esto rompe:

- seguridad;
- refactoring;
- metadata;
- compatibilidad;
- caché;
- estabilidad de persistencia.

---

# 7. Morph Type Registry

VoltStack utilizará un registry explícito.

Ejemplo conceptual:

```text
post    → EntityType(Post)
video   → EntityType(Video)
photo   → EntityType(Photo)
```

---

# 8. Stable Morph Alias

Cada tipo polimórfico tendrá un alias estable:

```php
#[Entity]
#[MorphAlias('post')]
final class Post
{
}
```

---

# 9. Alias ≠ Class Name

Mover:

```text
App\Models\Post
```

a:

```text
Domain\Content\Post
```

no deberá modificar datos existentes si el alias continúa siendo:

```text
post
```

---

# 10. Alias Stability

Regla:

```text
Refactor(PHP namespace)
≠
Migration(polymorphic data)
```

si el `MorphAlias` permanece estable.

---

# 11. Alias Uniqueness

Dentro del dominio de resolución correspondiente:

```text
MorphAlias
→
exactly one EntityType
```

---

# 12. Duplicate Alias

Esto deberá fallar durante bootstrap:

```text
Post  → content
Video → content
```

salvo que pertenezcan explícitamente a namespaces polimórficos diferentes.

---

# 13. Morph Type Namespace

Para sistemas extensibles puede utilizarse:

```php
final readonly class MorphTypeNamespace
{
    public function __construct(
        public string $value,
    ) {}
}
```

Ejemplo:

```text
content:post
content:video
commerce:product
```

---

# 14. PolymorphicTypeId

Representación canónica:

```php
final readonly class PolymorphicTypeId
{
    public function __construct(
        public string $namespace,
        public string $alias,
    ) {}
}
```

Conceptualmente:

```text
PolymorphicTypeId
=
MorphNamespace × StableAlias
```

---

# 15. MorphTypeRegistry

Propuesta:

```php
interface MorphTypeRegistry
{
    public function resolve(
        PolymorphicTypeId $type
    ): EntityType;

    public function typeFor(
        EntityType $entity
    ): PolymorphicTypeId;

    public function contains(
        PolymorphicTypeId $type
    ): bool;
}
```

---

# 16. Registry Properties

El registry deberá ser:

- deterministic;
- immutable después del bootstrap;
- validated;
- cacheable;
- process-shareable;
- independent del request;
- independent del tenant salvo extensión explícita.

---

# 17. No Runtime Class Discovery

El hot path no deberá realizar:

```text
scan classes
reflection discovery
filesystem scanning
composer package scanning
```

para resolver cada row.

Todo deberá estar compilado previamente.

---

# 18. PolymorphicReference

Propuesta:

```php
final readonly class PolymorphicReference
{
    public function __construct(
        public PolymorphicTypeId $type,
        public EntityKey $target,
    ) {}
}
```

---

# 19. Reference Formula

```text
PolymorphicReference
=
(TypeId, EntityKey)
```

---

# 20. Type Is Part of Identity

Esto es fundamental.

```text
(post, 42)
```

y:

```text
(video, 42)
```

son referencias distintas.

Por tanto:

```text
TargetIdentifier alone
≠
Polymorphic identity
```

---

# 21. PolymorphicEntityKey

Conceptualmente:

```text
PolymorphicEntityKey
=
EntityType
+
CanonicalIdentifier
+
EffectiveIdentityContext
```

El `EntityKey` canónico existente de VoltStack ya deberá preservar el tipo.

No se creará un segundo sistema de identidad incompatible.

---

# 22. IdentityMap Integration

Después de resolver:

```text
(post, 42)
```

a:

```text
EntityType(Post) + ID(42)
```

se consulta:

```text
IdentityMap
```

con el `EntityKey` canónico normal.

---

# 23. No Separate Polymorphic IdentityMap

No existirá:

```text
PolymorphicIdentityMap
```

como segundo sistema de identidad.

---

# 24. Canonicality

Dentro de un PersistenceContext:

```text
(post, 42)
```

obtenido mediante una relación polimórfica deberá resolver a la misma instancia `Post#42` obtenida mediante:

```php
$postRepository->find(42);
```

---

# 25. Polymorphic Metadata

Propuesta:

```php
final readonly class PolymorphicRelationshipMetadata
{
    /**
     * @param list<EntityType> $allowedTargets
     */
    public function __construct(
        public RelationshipId $id,
        public EntityType $sourceType,
        public string $property,
        public PolymorphicRelationshipKind $kind,
        public PolymorphicTypeBinding $typeBinding,
        public PolymorphicIdentifierBinding $identifierBinding,
        public array $allowedTargets,
        public RelationshipFetchMode $fetchMode,
        public CascadePolicy $cascade,
    ) {}
}
```

---

# 26. Relationship Kinds

```php
enum PolymorphicRelationshipKind
{
    case TO_ONE;
    case TO_MANY;
    case MANY_TO_MANY;
}
```

La metadata especializada podrá refinar cada categoría.

---

# 27. Type Binding

```php
final readonly class PolymorphicTypeBinding
{
    public function __construct(
        public ColumnName $column,
        public MorphTypeNamespace $namespace,
        public MorphDiscriminatorType $databaseType,
    ) {}
}
```

---

# 28. Identifier Binding

```php
final readonly class PolymorphicIdentifierBinding
{
    /**
     * @param list<PolymorphicIdentifierColumnBinding> $columns
     */
    public function __construct(
        public array $columns,
    ) {}
}
```

---

# 29. Composite IDs

VoltStack no asumirá:

```text
target_id = one integer
```

Podrá existir:

```text
target_type
target_tenant_id
target_id
```

o cualquier representación compatible con el `EntityKey`.

---

# 30. Complete Polymorphic Identity

Para materializar una referencia:

```text
type discriminator
+
complete target identifier
```

deberán estar disponibles.

---

# 31. Partial Identity

Ejemplo inválido:

```text
target_type = post
target_tenant_id = 5
target_id = MISSING
```

No deberá crearse una referencia parcial.

---

# 32. Null Reference

Para relación nullable:

```text
target_type = NULL
target_id   = NULL
```

puede representar:

```text
no target
```

---

# 33. Partial Null

Esto:

```text
target_type = post
target_id   = NULL
```

es inconsistente.

También:

```text
target_type = NULL
target_id   = 42
```

---

# 34. Null Invariant

Para una referencia simple:

```text
ReferenceAbsent
⇔
TypeIsNull
∧
AllIdentifierComponentsNull
```

---

# 35. Unknown Discriminator

Si DB contiene:

```text
commentable_type = legacy_post
```

pero registry no conoce ese alias:

```text
UNKNOWN DISCRIMINATOR
```

---

# 36. Unknown Discriminator ≠ Null

Nunca:

```text
unknown type
→ null relationship
```

automáticamente.

Eso ocultaría corrupción o incompatibilidad.

---

# 37. Default Policy

```text
UNKNOWN MORPH TYPE
→
UnknownPolymorphicTypeException
```

---

# 38. Compatibility Policy

Migraciones controladas podrán usar políticas especiales:

```text
FAIL
REPORT
LEGACY_ALIAS
CUSTOM_MIGRATION_RESOLVER
```

pero nunca arbitrary class instantiation.

---

# 39. Allowed Target Types

Una relación podrá restringir targets:

```php
#[MorphTo(
    types: [
        Post::class,
        Video::class,
    ]
)]
private Post|Video|null $commentable;
```

---

# 40. Registry Membership ≠ Relationship Permission

Que:

```text
photo
```

exista en `MorphTypeRegistry` no significa que cualquier relación pueda apuntar a Photo.

---

# 41. Allowed Target Validation

Debe cumplirse:

```text
ResolvedEntityType
∈
RelationshipAllowedTargets
```

---

# 42. Open Polymorphic Relationship

Framework/extensiones pueden permitir:

```text
allowedTargets = OPEN_REGISTRY
```

pero deberá ser explícito.

---

# 43. Recommended Default

Para aplicaciones normales:

```text
closed target set
```

es más seguro y analizable.

---

# 44. Attribute Mapping

Ejemplo:

```php
#[MorphTo(
    typeColumn: 'commentable_type',
    idColumn: 'commentable_id',
    types: [
        'post' => Post::class,
        'video' => Video::class,
    ]
)]
private Post|Video|null $commentable;
```

---

# 45. Central Registry Alternative

Preferible para aliases globalmente estables:

```php
#[MorphTo(
    typeColumn: 'commentable_type',
    idColumn: 'commentable_id',
    targets: [
        Post::class,
        Video::class,
    ]
)]
```

y:

```php
#[MorphAlias('post')]
final class Post {}

#[MorphAlias('video')]
final class Video {}
```

---

# 46. Mapping Compilation

Pipeline:

```text
Attributes / Config / Extensions
              ↓
      Mapping Sources
              ↓
 Polymorphic Mapping Normalizer
              ↓
 Polymorphic Mapping Validator
              ↓
     Morph Type Registry
              ↓
Compiled Relationship Metadata
```

---

# 47. Metadata ≠ Runtime Resolver State

Compiled metadata será immutable.

Current target objects pertenecen al PersistenceContext.

---

# 48. Metadata ≠ Schema

Definir:

```text
commentable_type
commentable_id
```

en mapping no demuestra su existencia física.

Schema System es responsable de observar/comparar estructura.

---

# 49. Schema Limitation

Una FK convencional:

```text
commentable_id
```

no puede apuntar simultáneamente a:

```text
posts.id
videos.id
photos.id
```

en la mayoría de RDBMS convencionales.

---

# 50. Consequence

Las relaciones polimórficas pueden sacrificar parte de la integridad referencial declarativa proporcionada por FK tradicionales.

---

# 51. ORM Integrity Becomes More Important

VoltStack deberá compensar mediante:

- metadata validation;
- target type validation;
- identifier validation;
- persistence ordering;
- diagnostics;
- integrity checks;
- optional database-specific strategies.

---

# 52. No Fake Foreign Key

Schema System no deberá fingir que existe una FK física cuando no existe.

---

# 53. Schema Diagnostics

Podrá reportar:

```text
POLYMORPHIC REFERENCE

Entity:
    Comment

Relationship:
    commentable

Type Column:
    commentable_type

Identifier:
    commentable_id

Allowed Targets:
    post  → Post
    video → Video

Physical FK:
    NOT REPRESENTABLE AS SINGLE STANDARD FK
```

---

# 54. Morph-To

Caso:

```text
Comment
   N
   │
   1
Post | Video
```

Conceptualmente es:

```text
polymorphic many-to-one
```

---

# 55. Morph-To Metadata

```php
final readonly class MorphToMetadata
{
    public function __construct(
        public RelationshipId $id,
        public EntityType $source,
        public string $property,
        public PolymorphicTypeBinding $type,
        public PolymorphicIdentifierBinding $identifier,
        public PolymorphicTargetSet $targets,
        public bool $nullable,
        public RelationshipFetchMode $fetchMode,
    ) {}
}
```

---

# 56. Morph-One

Ejemplo:

```text
User
  1
  │
  1
Image
```

donde `Image` puede pertenecer también a:

```text
Product
Organization
```

---

# 57. Morph-Many

Ejemplo:

```text
Post
  1
  │
  N
Comment
```

y:

```text
Video
  1
  │
  N
Comment
```

utilizando:

```text
comments.commentable_type
comments.commentable_id
```

---

# 58. Inverse Side

`Post.comments` conoce implícitamente que:

```text
commentable_type = post
```

---

# 59. Inverse Query Constraint

La consulta de:

```text
Post#42.comments
```

debe incluir semánticamente:

```text
target type = post
AND
target identifier = 42
```

---

# 60. Identifier Alone Is Unsafe

No basta:

```text
commentable_id = 42
```

porque podría incluir:

```text
video#42
photo#42
```

---

# 61. Polymorphic Predicate

Formalmente:

```text
Match(reference, entity)
=
Reference.type = MorphType(entity.type)
∧
Reference.id = EntityIdentifier(entity)
```

---

# 62. Query Builder Separation

Relationship System no generará:

```sql
WHERE commentable_type = 'post'
AND commentable_id = ?
```

directamente.

---

# 63. Semantic Query

Generará un predicate semántico:

```text
PolymorphicReferencePredicate(
    relationship = Comment.commentable,
    target = EntityKey(Post#42)
)
```

---

# 64. Query Engine

Luego:

```text
Semantic Query
    ↓
Optimizer
    ↓
Planner
    ↓
Compiler
    ↓
Dialect SQL
```

---

# 65. Morph-To Query

Ejemplo:

```php
Comment::query()
    ->whereRelated('commentable', $post)
    ->get();
```

deberá usar:

```text
Post EntityType
→ stable morph alias
→ identifier
→ semantic predicate
```

---

# 66. whereMorphType

Puede existir API:

```php
Comment::query()
    ->whereMorphType('commentable', Post::class);
```

pero deberá resolver `Post::class` mediante metadata.

---

# 67. Public API ≠ Persisted Alias Knowledge

El desarrollador puede trabajar con:

```php
Post::class
```

mientras el storage usa:

```text
post
```

---

# 68. Query Across Multiple Morph Types

Ejemplo:

```php
Comment::query()
    ->whereMorphType('commentable', [
        Post::class,
        Video::class,
    ]);
```

se convierte en un conjunto tipado de discriminadores.

---

# 69. Morph Type Predicate

```text
MorphTypeSet
=
{post, video}
```

---

# 70. Query Planner

El planner podrá decidir estrategias diferentes según:

- relationship direction;
- target type count;
- table topology;
- indexes;
- cardinality;
- requested result shape.

---

# 71. Loading Morph-To

Supongamos:

```text
Comment#1 → post#100
Comment#2 → video#50
Comment#3 → post#200
```

Carga naive:

```text
3 target queries
```

es indeseable.

---

# 72. Batch Polymorphic Loading

VoltStack agrupará referencias por `EntityType`.

```text
post:
    100
    200

video:
    50
```

---

# 73. Batch Formula

```text
References
    ↓
GroupBy(EntityType)
    ↓
BatchLoad each target type
    ↓
IdentityMap
    ↓
Attach canonical entities
```

---

# 74. Query Count

Idealmente:

```text
Queries
≈
number of distinct target entity types
```

en vez de:

```text
number of source entities
```

---

# 75. Batch Loading Example

```text
100 Comments

70 → Post
20 → Video
10 → Photo
```

podrían resolverse aproximadamente con:

```text
1 source query
+
3 target batch queries
```

en lugar de 101 queries.

---

# 76. N+1 Prevention

Polymorphic batch loader deberá integrarse con N+1 detection.

---

# 77. PolymorphicBatchKey

Propuesta:

```php
final readonly class PolymorphicBatchKey
{
    public function __construct(
        public RelationshipId $relationship,
        public EntityType $targetType,
        public DatabaseContextKey $database,
    ) {}
}
```

---

# 78. Tenant-Aware Batch

Nunca mezclar:

```text
Tenant A post IDs
```

con:

```text
Tenant B post IDs
```

en un mismo batch si la estrategia de aislamiento lo prohíbe.

---

# 79. Shard-Aware Batch

Igualmente:

```text
BatchKey
=
Relationship
+
TargetType
+
Tenant/Database/Shard Context
```

---

# 80. Hydration

Hydration recibe:

```text
raw type discriminator
raw target identifier
```

desde un `Result`.

---

# 81. Hydration Pipeline

```text
Result Row
    ↓
HydrationPlan
    ↓
Read Type Binding
    ↓
Convert Morph Type
    ↓
MorphTypeRegistry
    ↓
Resolve EntityType
    ↓
Validate Allowed Target
    ↓
Read + Convert Identifier
    ↓
Build Canonical EntityKey
    ↓
IdentityMap Lookup
    ↓
Reuse / Load / Assemble
```

---

# 82. No Reflection Hot Path

Hydrator no deberá:

```text
read attribute
scan class
discover mapping
```

por cada row.

---

# 83. Hydration Plan Node

Propuesta:

```php
final readonly class PolymorphicReferenceHydrationNode
    implements HydrationNode
{
    public function __construct(
        public RelationshipId $relationship,
        public ColumnBinding $typeColumn,
        public PolymorphicIdentifierHydrationBinding $identifier,
        public MorphTypeResolverPlan $resolver,
        public PolymorphicNullabilityPlan $nullability,
    ) {}
}
```

---

# 84. Compiled Resolver

`MorphTypeResolverPlan` podrá contener una tabla immutable:

```text
post  → EntityTypeId#12
video → EntityTypeId#15
photo → EntityTypeId#20
```

---

# 85. Unknown Alias Hot Path

Lookup fallido:

```text
alias not found
```

deberá fallar determinísticamente.

No:

```text
try class_exists(alias)
```

---

# 86. Hydration Without Target Data

Si el result solo contiene:

```text
type + identifier
```

puede construirse una referencia lazy:

```text
PolymorphicEntityReference
```

sin cargar target inmediatamente.

---

# 87. PolymorphicEntityReference

```php
final readonly class PolymorphicEntityReference
{
    public function __construct(
        public EntityType $type,
        public EntityKey $key,
    ) {}
}
```

No contiene `EntityManager`.

---

# 88. Reference ≠ Proxy Necessarily

VoltStack podrá elegir:

- typed proxy;
- lazy reference wrapper;
- explicit deferred loader;
- no lazy reference.

La estrategia será independiente del identity model.

---

# 89. getReference()

Podría existir:

```php
$reference = $entityManager->getReference(
    Post::class,
    42
);
```

y ser reutilizada al resolver la relación polimórfica.

---

# 90. IdentityMap First

Si `Post#42` ya está managed:

```text
polymorphic hydration
→ reuse Post#42
```

---

# 91. Dirty Entity Protection

La carga normal de una referencia polimórfica no deberá sobrescribir campos dirty de una entidad ya managed.

---

# 92. Target Hydration

Si la consulta también contiene columnas del target, el `EntityHydrator` canónico continúa siendo responsable de materializarlo.

---

# 93. Polymorphic Hydrator ≠ Entity Hydrator

Polymorphic hydrator:

```text
resolves target type/reference
```

Entity hydrator:

```text
materializes target entity
```

---

# 94. Eager Loading Morph-To

No existe necesariamente un único JOIN simple capaz de traer todos los posibles target tables de manera eficiente.

---

# 95. Strategy

Eager loading podrá preferir:

```text
source query
+
grouped target queries
```

---

# 96. Union Strategy

En determinadas circunstancias un planner avanzado podría usar:

```text
UNION ALL
```

sobre targets compatibles.

Pero esto será una optimización del Query Planner, no una regla del Relationship System.

---

# 97. Multiple LEFT JOIN Strategy

Otra representación posible:

```text
LEFT JOIN posts
LEFT JOIN videos
LEFT JOIN photos
```

puede ser costosa y compleja.

No será default universal.

---

# 98. Capability/Cost Driven

La estrategia física deberá considerar:

```text
target count
platform
statistics
projection
cardinality
result shape
memory
streaming
```

---

# 99. Morph-Many Loading

Para:

```text
Post#42.comments
```

el target type ya es conocido:

```text
post
```

por el source.

Por tanto la query semántica conoce:

```text
(type = post, id = 42)
```

---

# 100. Morph-Many Collection

La colección sigue las mismas reglas generales del Relationship System:

```text
UNINITIALIZED
PARTIAL
COMPLETE
STALE
UNCERTAIN
```

---

# 101. UNINITIALIZED ≠ EMPTY

También aplica a relaciones polimórficas.

---

# 102. Collection Completeness

Una consulta:

```text
Post#42.comments where approved = true
```

no deberá marcar automáticamente `Post.comments` como COMPLETE.

---

# 103. Persistence

Asignar:

```php
$comment->commentable = $post;
```

deberá producir semantic relationship change.

---

# 104. Persistence Values

La persistencia deriva:

```text
type = MorphType(Post)
id   = Identifier(Post)
```

---

# 105. No Class Name Persistence

Nunca derivar automáticamente:

```text
type = $post::class
```

como representación persistida.

---

# 106. Relationship Change

Conceptualmente:

```text
PreviousReference
→
CurrentReference
```

---

# 107. PolymorphicReferenceChangeSet

```php
final readonly class PolymorphicReferenceChangeSet
{
    public function __construct(
        public RelationshipId $relationship,
        public ?PolymorphicReference $before,
        public ?PolymorphicReference $after,
    ) {}
}
```

---

# 108. ChangeSet ≠ SQL

Cambiar:

```text
post#42
→
video#42
```

es una sola mutación semántica de relación aunque físicamente cambien:

```text
type column
+
id column
```

---

# 109. Atomic Semantic Change

El planner deberá preservar la coherencia del par:

```text
(type, identifier)
```

---

# 110. Torn Reference

Nunca deberá considerarse estado lógico válido:

```text
new type
+
old identifier
```

---

# 111. Update Representation

Query Engine podrá generar un UPDATE que modifique conjuntamente los componentes necesarios.

---

# 112. Generated Identifier

Si target es NEW:

```text
Comment
→
new Post
```

y cascade persist está permitido:

```text
Insert Post
    ↓
resolve generated Post ID
    ↓
write polymorphic reference
```

---

# 113. Dependency Barrier

```text
PersistPolymorphicReference
requires
EstablishedTargetIdentity
```

---

# 114. Assigned Identity ≠ Persisted Target

Que un target tenga ID asignado no demuestra que exista en DB.

Se mantienen las reglas generales de Entity State.

---

# 115. Cascade Persist

Será explícito.

---

# 116. Cascade Remove

Default conservador:

```text
OFF
```

para `morphTo`.

---

# 117. Why Cascade Remove Is Dangerous

Un `Attachment` apuntando a `Post` no debería poder borrar el Post automáticamente salvo modelado explícito.

---

# 118. Inverse Cascade

Eliminar `Post` puede implicar:

- borrar comments;
- nullificar references;
- restringir delete;
- soft-delete;
- custom policy.

Debe definirse por relación.

---

# 119. Database FK Limitation

Como no existe FK estándar hacia múltiples tablas, ciertas acciones deberán ser gobernadas por ORM o estrategias específicas.

---

# 120. Delete Integrity

Antes de eliminar un polymorphic target, el sistema puede necesitar conocer referencias inversas.

---

# 121. Global Reverse Lookup

No deberá hacerse mediante scanning runtime de todas las entidades.

Metadata Registry deberá conocer mappings inversos.

---

# 122. Reverse Relationship Registry

Propuesta:

```php
interface PolymorphicReverseRelationshipRegistry
{
    /**
     * @return list<PolymorphicRelationshipMetadata>
     */
    public function referencing(
        EntityType $target
    ): array;
}
```

---

# 123. Registry Compiled at Bootstrap

Esto permite:

- deletion planning;
- diagnostics;
- schema analysis;
- migration analysis;
- authorization integrations futuras.

---

# 124. Morph-One Uniqueness

Una relación polymorphic one-to-one necesita garantizar:

```text
at most one source row
for
(type, target identity)
```

---

# 125. Recommended DB Constraint

Cuando sea representable:

```text
UNIQUE(target_type, target_id)
```

o equivalente para IDs compuestos.

---

# 126. Morph-Many Index

Recomendado:

```text
INDEX(target_type, target_id)
```

para resolución inversa.

---

# 127. Index Metadata ≠ Relationship Metadata

Relationship System puede recomendar/diagnosticar.

Schema System define/observa indexes.

---

# 128. Polymorphic Many-to-Many

Ejemplo:

```text
Tag
 ↑
 │
taggables
 │
 ├── Post
 ├── Video
 └── Product
```

Tabla:

```text
taggables
──────────────
tag_id
taggable_type
taggable_id
```

---

# 129. Membership Identity

Para polymorphic many-to-many:

```text
MembershipKey
=
RelationshipId
+
TagEntityKey
+
PolymorphicTargetEntityKey
```

---

# 130. Physical Uniqueness

Recomendado:

```text
UNIQUE(
    tag_id,
    taggable_type,
    taggable_id
)
```

---

# 131. Polymorphic Many-to-Many ≠ Normal Many-to-Many

Añade:

```text
target type discriminator
```

como parte necesaria de la membership.

---

# 132. Many-to-Many Rules Reused

Se reutilizan las garantías del documento 146:

- UNINITIALIZED ≠ EMPTY;
- membership snapshots;
- add/remove;
- clear;
- sync;
- collection completeness;
- extra-lazy;
- batching;
- persistence certainty.

---

# 133. No Duplicate Relationship Engine

No se creará un segundo UoW para polymorphic many-to-many.

---

# 134. Generalized Membership

La infraestructura many-to-many podrá aceptar:

```text
TargetReference
=
EntityReference
or
PolymorphicEntityReference
```

mediante contratos tipados.

---

# 135. Querying Polymorphic Many-to-Many

Ejemplo:

```php
$post->tags();
```

conoce:

```text
taggable_type = post
taggable_id   = Post#ID
```

---

# 136. Reverse Query

Desde Tag:

```php
$tag->taggables();
```

el resultado puede contener múltiples EntityTypes.

---

# 137. Heterogeneous Result

Ejemplo:

```text
Post#10
Video#7
Product#50
```

---

# 138. Heterogeneous Collection

Una colección polimórfica puede contener entidades de distintos tipos.

---

# 139. Collection Identity

Dedup deberá utilizar:

```text
EntityKey
```

incluyendo `EntityType`.

---

# 140. Same Numeric ID

```text
Post#10
Video#10
```

son dos miembros diferentes.

---

# 141. Ordering Heterogeneous Targets

Ordenar por un campo que no existe uniformemente en todos los tipos puede ser inválido.

---

# 142. Semantic Validation

Ejemplo:

```text
orderBy('title')
```

solo será válido si la semántica de todos los targets seleccionados lo soporta o existe una proyección común explícita.

---

# 143. Common Polymorphic Projection

Puede existir:

```text
PolymorphicProjection
├── type
├── id
└── label
```

con mappings específicos por EntityType.

---

# 144. Projection ≠ Entity Interface Automatically

Que varias entidades implementen una interfaz PHP no implica que sus columnas relacionales sean uniformes.

---

# 145. Result Shape

Una consulta polimórfica podrá producir:

```text
ENTITY
POLYMORPHIC_ENTITY
TUPLE
DTO
PROJECTION
SCALAR
```

según contrato.

---

# 146. Hydration Plan Requirements

El plan deberá conocer:

- type column;
- identifier columns;
- allowed aliases;
- alias→EntityType mapping;
- target-specific subplans;
- nullability;
- grouping;
- collection semantics;
- result shape.

---

# 147. Target-Specific Subplans

Ejemplo:

```text
PolymorphicHydrationNode
├── post  → PostEntityHydrationPlan
├── video → VideoEntityHydrationPlan
└── photo → PhotoEntityHydrationPlan
```

---

# 148. Dynamic Subplan Selection

La selección se realiza mediante discriminador validado.

No mediante reflexión.

---

# 149. Plan Validation

Antes del hot path se validará:

```text
all allowed aliases resolve
all target metadata exists
identifier bindings compatible
subplans valid
no duplicate aliases
null semantics valid
target restrictions valid
```

---

# 150. Hydration Plan Cache

El cache key deberá considerar:

```text
Relationship Metadata Generation
Morph Registry Generation
Entity Metadata Generation
Type Registry Generation
Hydration Compiler Version
```

---

# 151. Morph Registry Generation

Propuesta:

```php
final readonly class MorphRegistryGeneration
{
    public function __construct(
        public int|string $value,
    ) {}
}
```

---

# 152. Cache Invalidation

Cambiar:

```text
post → Post
```

por otra resolución deberá producir nueva generación/fingerprint.

---

# 153. No Mutable Registry Behind Cached Plan

Un plan compilado no deberá observar silenciosamente un registry mutado.

---

# 154. Persistent Runtime

Compartible entre requests:

```text
MorphTypeRegistry
PolymorphicRelationshipMetadata
compiled alias maps
hydration plan templates
reverse relationship registry
mapping validators
```

si son immutable.

---

# 155. Scoped State

Debe ser request/operation scoped:

```text
EntityManager
IdentityMap
UnitOfWork
loaded polymorphic targets
lazy loading queues
batch queues
collection state
pending relationship changes
HydrationSession
```

---

# 156. FrankenPHP Safety

Nunca:

```php
static array $resolvedPolymorphicTargets;
```

conteniendo entidades de requests anteriores.

---

# 157. RoadRunner/OpenSwoole

Se aplican las mismas garantías:

```text
immutable metadata shared
+
mutable ORM state scoped
```

---

# 158. Coroutine Safety

Un `HydrationSession` no deberá compartirse concurrentemente entre operaciones incompatibles.

---

# 159. Batch Queue Isolation

Batch loaders deberán separar por:

```text
PersistenceContext
DatabaseContext
Tenant
Shard
Relationship
Target EntityType
```

según corresponda.

---

# 160. Tenant Isolation

Una referencia:

```text
(post, 42)
```

no es suficiente en un sistema tenant-aware.

Conceptualmente:

```text
Tenant A / Post#42
≠
Tenant B / Post#42
```

---

# 161. EntityKey Carries Effective Context

La resolución deberá producir el `EntityKey` correcto dentro del tenant/database namespace.

---

# 162. Cross-Tenant Reference

Default:

```text
REJECT
```

---

# 163. Cross-Shard Reference

Default:

```text
REJECT
```

salvo extensión distribuida explícita.

---

# 164. Type Discriminator Does Not Select Tenant

Nunca utilizar:

```text
target_type
```

para determinar arbitrariamente credenciales/database.

---

# 165. Database Routing

DatabaseContext se determina mediante las reglas de infraestructura/tenant/shard.

El discriminador únicamente resuelve el tipo lógico permitido.

---

# 166. Security Threat: Class Injection

Entrada maliciosa:

```text
target_type =
"Some\\Unexpected\\Class"
```

no deberá causar:

```text
class_exists()
new $class()
reflection
autoload
```

---

# 167. Security Threat: Alias Injection

Incluso un alias:

```text
admin_user
```

solo será válido si:

```text
registry contains alias
∧
relationship allows target
```

---

# 168. Security Threat: Type Confusion

Un atacante no deberá poder convertir:

```text
Post#42
```

en:

```text
User#42
```

modificando únicamente el discriminador sin pasar por validation/authorization de capas superiores.

---

# 169. Database Security ≠ Authorization

Relationship System valida integridad estructural.

Authorization System decide si el actor puede cambiar la relación.

---

# 170. Mass Assignment

Payload:

```json
{
  "commentable_type": "post",
  "commentable_id": 42
}
```

no deberá asignarse directamente al persistence model por default.

---

# 171. Preferred Application Input

Capas superiores deberían resolver:

```text
validated domain target
→ entity/reference
→ ORM relationship assignment
```

---

# 172. Alias Exposure

El alias persistente puede o no coincidir con un API discriminator público.

No deberán acoplarse obligatoriamente.

---

# 173. API Type ≠ DB Morph Type

Ejemplo:

```text
API: article
DB:  post
ORM: EntityType(Post)
```

puede ser válido con mapping explícito.

---

# 174. Rename Safety

Supongamos:

```text
post → article
```

como cambio del alias persistido.

Eso sí requiere estrategia de migración.

---

# 175. Alias Migration

No bastará cambiar metadata.

Debe contemplarse:

```text
old alias data
+
new alias mapping
+
migration
+
compatibility window
```

---

# 176. Alias Compatibility Window

Puede existir temporalmente:

```text
post-v1 → Post
post    → Post
```

donde uno es alias legacy de lectura.

---

# 177. Canonical Write Alias

Durante transición:

```text
read:
    post-v1
    post

write:
    post
```

---

# 178. Legacy Alias

Propuesta:

```php
#[MorphAlias('post')]
#[LegacyMorphAlias('post-v1')]
final class Post
{
}
```

---

# 179. Legacy Alias ≠ Permanent Duplicate Alias

Debe tener lifecycle/deprecation explícito.

---

# 180. Migration Completion

Después de migrar datos:

```text
legacy alias
→ removed
```

en una versión compatible planificada.

---

# 181. Versioning

Cambios en Morph Registry son cambios de persistencia potencialmente incompatibles.

Deberán formar parte de:

```text
Database compatibility governance
```

---

# 182. Schema Migration Detection

Schema Diff no detectará necesariamente un cambio semántico:

```text
'post' → 'article'
```

porque estructura de columnas puede ser idéntica.

---

# 183. Data Migration

Este cambio requiere:

```text
data migration
```

no únicamente schema migration.

---

# 184. Migration Safety

Renombrar un alias en producción puede requerir:

```text
expand
dual-read
canonical-write
backfill
verify
contract
```

---

# 185. Zero-Downtime Alias Migration

Ejemplo:

```text
Phase 1:
read old + new
write old

Phase 2:
read old + new
write new

Phase 3:
backfill old → new

Phase 4:
verify no old aliases

Phase 5:
remove old alias support
```

---

# 186. Observability

Telemetry deberá poder detectar:

```text
legacy alias reads
unknown aliases
type resolution failures
batch grouping
polymorphic lazy loads
```

---

# 187. Metrics

Sugeridas:

```text
orm.relationship.polymorphic.resolutions
orm.relationship.polymorphic.resolution_failures
orm.relationship.polymorphic.unknown_types
orm.relationship.polymorphic.legacy_alias_reads

orm.relationship.polymorphic.lazy_loads
orm.relationship.polymorphic.eager_loads
orm.relationship.polymorphic.batch_loads
orm.relationship.polymorphic.batch_target_types

orm.relationship.polymorphic.persistence_changes
orm.relationship.polymorphic.persistence_failures

orm.relationship.polymorphic.cross_tenant_rejections
orm.relationship.polymorphic.cross_shard_rejections
```

---

# 188. Metric Cardinality

No usar labels como:

```text
target_id
source_id
tenant_id
```

---

# 189. Safe Labels

Pueden utilizarse:

```text
relationship kind
fetch strategy
resolution outcome
platform
```

y EntityType solo si el cardinality budget lo permite.

---

# 190. Diagnostics

Ejemplo:

```text
POLYMORPHIC RELATIONSHIP

Source:
    Comment

Property:
    commentable

Kind:
    MORPH_TO

Type Column:
    commentable_type

Identifier:
    commentable_id

Allowed Targets:
    post  → Post
    video → Video
    photo → Photo

Nullable:
    YES

Fetch:
    LAZY

Unknown Type Policy:
    FAIL
```

---

# 191. Resolution Diagnostics

```text
POLYMORPHIC RESOLUTION FAILURE

Relationship:
    Comment.commentable

Observed Alias:
    legacy_article

Namespace:
    content

Registry Generation:
    42

Status:
    UNKNOWN_ALIAS

Target Instantiated:
    NO
```

No incluir datos sensibles innecesarios.

---

# 192. Explain Batch

```text
POLYMORPHIC BATCH LOAD

Relationship:
    Comment.commentable

Sources:
    100

References:
    100

Groups:
    Post     70
    Video    20
    Photo    10

Strategy:
    GROUP_BY_ENTITY_TYPE

Expected Target Queries:
    3
```

---

# 193. Error Taxonomy

```text
PolymorphicRelationshipException
├── PolymorphicMappingException
├── PolymorphicMetadataException
├── MorphAliasException
├── DuplicateMorphAliasException
├── UnknownPolymorphicTypeException
├── DisallowedPolymorphicTargetException
├── InvalidPolymorphicReferenceException
├── PartialPolymorphicReferenceException
├── PolymorphicIdentifierException
├── PolymorphicTypeConversionException
├── PolymorphicHydrationException
├── PolymorphicLoadingException
├── PolymorphicBatchLoadingException
├── PolymorphicPersistenceException
├── PolymorphicReferenceConflictException
├── PolymorphicMigrationCompatibilityException
├── CrossTenantPolymorphicReferenceException
├── CrossShardPolymorphicReferenceException
├── PolymorphicRuntimeIsolationException
├── PolymorphicResourceLimitException
└── PolymorphicInvariantException
```

---

# 194. Resource Governance

Una consulta con cientos de tipos registrados no deberá intentar cargarlos todos indiscriminadamente.

---

# 195. Runtime Target Set

Solo los discriminadores realmente observados o explícitamente solicitados deberán generar target load groups.

---

# 196. Max Target Types

Puede existir:

```php
final readonly class PolymorphicLoadPolicy
{
    public function __construct(
        public int $maxTargetTypesPerBatch,
        public int $maxReferencesPerBatch,
        public int $maxBufferedReferences,
    ) {}
}
```

---

# 197. Open Registry Risk

Una relación abierta con cientos de tipos puede producir:

```text
query fan-out
```

---

# 198. Query Fan-Out

Estimación:

```text
TargetQueries
≈
DistinctTargetTypes
```

para grouped batch loading.

---

# 199. Planner Threshold

Cuando:

```text
DistinctTargetTypes
```

sea alto, diagnostics/planner podrá recomendar otra estrategia/modelado.

---

# 200. Architectural Modeling Guidance

Las relaciones polimórficas ofrecen ergonomía, pero tienen trade-offs:

```text
+ flexible target types
+ reusable relation tables
+ convenient domain navigation

- weaker DB FK enforcement
- more complex query planning
- more complex migrations
- potential query fan-out
- harder cross-type analytics
```

---

# 201. Prefer Normal FK When Closed and Stable

Si una relación siempre apunta únicamente a:

```text
User
```

deberá utilizarse una relación normal.

No una polimórfica.

---

# 202. Prefer Explicit Association Model When Rich

Si la referencia polimórfica posee:

- lifecycle;
- metadata;
- permissions;
- state;
- temporal history;

puede ser mejor una entidad explícita.

---

# 203. Avoid Universal Entity Reference

Anti-pattern:

```text
entity_type
entity_id
```

utilizado para conectar absolutamente todo con todo.

---

# 204. Why Universal Reference Is Dangerous

Puede destruir:

- domain boundaries;
- schema clarity;
- static analysis;
- referential integrity;
- query optimization;
- maintainability.

---

# 205. Polymorphism Must Be Intentional

Cada relación deberá declarar:

```text
what target universe is valid
```

en lugar de usar un universal morph registry sin restricciones.

---

# 206. Extension System

Paquetes podrán registrar nuevos aliases mediante API controlada durante bootstrap.

---

# 207. Extension Registration

Ejemplo conceptual:

```php
$morphRegistry->register(
    namespace: 'content',
    alias: 'article',
    entity: Article::class,
);
```

---

# 208. Registry Freeze

Después de bootstrap:

```text
register()
→ forbidden
```

en runtime normal.

---

# 209. Conflict Detection

Dos paquetes registrando:

```text
content:article
```

para tipos distintos deberán fallar determinísticamente.

---

# 210. Extension Cannot Bypass Security

Un plugin no deberá convertir datos arbitrarios de DB en nombres de clases dinámicos.

---

# 211. Metadata Generation

Registrar una extensión cambia:

```text
MorphRegistryGeneration
```

antes de que los planes se compilen.

---

# 212. Dependency Direction

```text
Polymorphic Relationship
        ↓
Entity Metadata
        ↓
Entity Query
        ↓
Query Engine
        ↓
Execution
```

y para materialización:

```text
Result
  ↓
Hydration Plan
  ↓
Polymorphic Resolver
  ↓
Entity Hydrator
  ↓
IdentityMap
```

---

# 213. Forbidden Dependency

Nunca:

```text
Driver
→
Polymorphic Relationship
```

---

# 214. Compiler Independence

SQL Compiler conoce:

```text
AST
dialect
bindings
```

no:

```text
Comment.commentable
```

como objeto ORM runtime.

---

# 215. Executor Independence

Executor devuelve:

```text
Result
```

No decide:

```text
'post' means Post entity
```

---

# 216. Resolver Responsibility

La interpretación:

```text
'post'
→
EntityType(Post)
```

pertenece al Polymorphic Relationship/Hydration metadata layer.

---

# 217. Testing Strategy

La suite deberá cubrir:

```text
morphTo
morphOne
morphMany
polymorphic many-to-many
stable aliases
class namespace refactor
unknown aliases
legacy aliases
closed target sets
open registries
nullable references
partial null references
composite identifiers
IdentityMap reuse
lazy loading
eager loading
batch loading
N+1 prevention
persistence
generated identifiers
cascade policies
inverse relationships
tenant isolation
shard isolation
migration compatibility
persistent workers
concurrency
telemetry
```

---

# 218. Core Test — Same ID, Different Type

```text
post#42
video#42
```

deben producir dos referencias diferentes.

---

# 219. Core Test — Canonical Identity

Dos references:

```text
post#42
post#42
```

dentro del mismo PersistenceContext resuelven a la misma instancia managed.

---

# 220. Core Test — Unknown Alias

```text
target_type = unsupported
```

deberá fallar sin intentar cargar una clase dinámica.

---

# 221. Core Test — Disallowed Known Alias

Si registry conoce:

```text
user
```

pero relación solo permite:

```text
post
video
```

entonces:

```text
user
→
DisallowedPolymorphicTargetException
```

---

# 222. Core Test — Refactor

Cambiar:

```text
App\Models\Post
```

a:

```text
Domain\Content\Post
```

manteniendo:

```text
MorphAlias = post
```

no modifica representación persistida.

---

# 223. Core Test — Partial Null

```text
type = post
id   = NULL
```

deberá fallar.

---

# 224. Core Test — Complete Null

```text
type = NULL
id   = NULL
```

en relación nullable produce `null`.

---

# 225. Core Test — Batch Grouping

100 references de tres EntityTypes deberán agruparse en tres grupos lógicos.

---

# 226. Core Test — Tenant Isolation

```text
TenantA/Post#42
```

nunca reutiliza:

```text
TenantB/Post#42
```

desde IdentityMap.

---

# 227. Core Test — Persistence

Asignar `Post#42` deberá persistir:

```text
canonical alias for Post
+
canonical identifier for Post
```

---

# 228. Core Test — Alias Migration

Legacy alias deberá resolverse únicamente cuando policy/generation lo permita.

---

# 229. Core Test — Persistent Runtime

Dos requests en el mismo worker podrán compartir metadata pero nunca target entity instances.

---

# 230. Architectural Invariants

## DB-ORM-POLYMORPHIC-001

Persisted morph type será distinto de PHP class name.

## DB-ORM-POLYMORPHIC-002

Morph aliases serán estables.

## DB-ORM-POLYMORPHIC-003

Morph aliases serán validados durante bootstrap.

## DB-ORM-POLYMORPHIC-004

Un alias resolverá determinísticamente a un EntityType.

## DB-ORM-POLYMORPHIC-005

Unknown alias no será convertido a class name.

## DB-ORM-POLYMORPHIC-006

Unknown alias no será convertido a null.

## DB-ORM-POLYMORPHIC-007

Registry será immutable durante runtime normal.

## DB-ORM-POLYMORPHIC-008

Registry mutation cambiará generation.

## DB-ORM-POLYMORPHIC-009

Polymorphic relationship será distinta de inheritance.

## DB-ORM-POLYMORPHIC-010

Polymorphic relationship será distinta de PHP union type.

## DB-ORM-POLYMORPHIC-011

Polymorphic reference contendrá type + identifier.

## DB-ORM-POLYMORPHIC-012

Target identifier solo no definirá polymorphic identity.

## DB-ORM-POLYMORPHIC-013

Same numeric ID across different types será distinto.

## DB-ORM-POLYMORPHIC-014

EntityKey canónico será reutilizado.

## DB-ORM-POLYMORPHIC-015

No existirá segundo IdentityMap polimórfico.

## DB-ORM-POLYMORPHIC-016

Polymorphic resolution preservará IdentityMap canonicality.

## DB-ORM-POLYMORPHIC-017

Allowed target set será independiente del global registry.

## DB-ORM-POLYMORPHIC-018

Known alias podrá ser disallowed para una relación.

## DB-ORM-POLYMORPHIC-019

Closed target set será default recomendado.

## DB-ORM-POLYMORPHIC-020

Open target registry será explícito.

## DB-ORM-POLYMORPHIC-021

Composite target identifiers serán soportados.

## DB-ORM-POLYMORPHIC-022

Partial target identifier será inválido.

## DB-ORM-POLYMORPHIC-023

Null reference requerirá null type + null complete identifier.

## DB-ORM-POLYMORPHIC-024

Partial null reference será inconsistente.

## DB-ORM-POLYMORPHIC-025

Metadata será distinta de Schema.

## DB-ORM-POLYMORPHIC-026

ORM no fabricará FK física inexistente.

## DB-ORM-POLYMORPHIC-027

Schema diagnostics reconocerán limitaciones de FK.

## DB-ORM-POLYMORPHIC-028

Inverse lookup incluirá type discriminator.

## DB-ORM-POLYMORPHIC-029

Identifier-only inverse lookup será inválido.

## DB-ORM-POLYMORPHIC-030

Relationship System no generará SQL.

## DB-ORM-POLYMORPHIC-031

Polymorphic predicates serán semantic query nodes.

## DB-ORM-POLYMORPHIC-032

Compiler resolverá dialect representation downstream.

## DB-ORM-POLYMORPHIC-033

Public EntityType API no requerirá conocer persisted alias.

## DB-ORM-POLYMORPHIC-034

Batch loading agrupará por target EntityType.

## DB-ORM-POLYMORPHIC-035

Batch loading respetará tenant/database/shard context.

## DB-ORM-POLYMORPHIC-036

N+1 detection observará polymorphic lazy loads.

## DB-ORM-POLYMORPHIC-037

Hydration utilizará HydrationPlan.

## DB-ORM-POLYMORPHIC-038

Hydration no realizará reflection discovery por row.

## DB-ORM-POLYMORPHIC-039

Hydration no analizará SQL.

## DB-ORM-POLYMORPHIC-040

Discriminator resolution ocurrirá antes de target hydration.

## DB-ORM-POLYMORPHIC-041

Target EntityHydrator seguirá siendo canónico.

## DB-ORM-POLYMORPHIC-042

Polymorphic resolver será distinto de EntityHydrator.

## DB-ORM-POLYMORPHIC-043

Existing managed target será reutilizado.

## DB-ORM-POLYMORPHIC-044

Normal hydration no sobrescribirá dirty managed entity.

## DB-ORM-POLYMORPHIC-045

Eager loading será distinto de single JOIN.

## DB-ORM-POLYMORPHIC-046

Physical eager strategy será planner-driven.

## DB-ORM-POLYMORPHIC-047

Morph-many collection preservará collection completeness.

## DB-ORM-POLYMORPHIC-048

UNINITIALIZED será distinto de EMPTY.

## DB-ORM-POLYMORPHIC-049

Filtered morph-many load será PARTIAL cuando corresponda.

## DB-ORM-POLYMORPHIC-050

Assignment producirá semantic relationship change.

## DB-ORM-POLYMORPHIC-051

Relationship ChangeSet será distinto de SQL.

## DB-ORM-POLYMORPHIC-052

Type + identifier se tratarán como referencia semántica coherente.

## DB-ORM-POLYMORPHIC-053

Torn polymorphic reference no será estado lógico válido.

## DB-ORM-POLYMORPHIC-054

Generated target ID será dependency barrier.

## DB-ORM-POLYMORPHIC-055

Assigned ID no demostrará DB existence.

## DB-ORM-POLYMORPHIC-056

Cascade persist será explícito.

## DB-ORM-POLYMORPHIC-057

Cascade remove estará OFF por default para morph-to.

## DB-ORM-POLYMORPHIC-058

Reverse relationship metadata será compilable.

## DB-ORM-POLYMORPHIC-059

Reverse lookup no escaneará classes en hot path.

## DB-ORM-POLYMORPHIC-060

Morph-one requerirá uniqueness semántica.

## DB-ORM-POLYMORPHIC-061

Schema constraint será distinta de ORM uniqueness semantics.

## DB-ORM-POLYMORPHIC-062

Polymorphic many-to-many reutilizará membership engine.

## DB-ORM-POLYMORPHIC-063

Polymorphic many-to-many incluirá target type en membership identity.

## DB-ORM-POLYMORPHIC-064

Same ID different type serán memberships diferentes.

## DB-ORM-POLYMORPHIC-065

Heterogeneous collection dedup utilizará EntityKey.

## DB-ORM-POLYMORPHIC-066

Cross-type ordering será semanticamente validado.

## DB-ORM-POLYMORPHIC-067

PHP interface no implicará relational projection compatibility.

## DB-ORM-POLYMORPHIC-068

Hydration plan tendrá target-specific subplans cuando sean necesarios.

## DB-ORM-POLYMORPHIC-069

Subplan selection utilizará validated discriminator.

## DB-ORM-POLYMORPHIC-070

Hydration plan cache dependerá de MorphRegistryGeneration.

## DB-ORM-POLYMORPHIC-071

Cached plan no observará mutable registry.

## DB-ORM-POLYMORPHIC-072

Immutable registry podrá compartirse entre workers.

## DB-ORM-POLYMORPHIC-073

Managed targets serán scope-local.

## DB-ORM-POLYMORPHIC-074

Batch queues serán scope-local.

## DB-ORM-POLYMORPHIC-075

HydrationSession será scope-local.

## DB-ORM-POLYMORPHIC-076

Persistent workers no conservarán target entities entre requests.

## DB-ORM-POLYMORPHIC-077

Cross-tenant reference será rechazada por default.

## DB-ORM-POLYMORPHIC-078

Cross-shard reference será rechazada por default.

## DB-ORM-POLYMORPHIC-079

Morph discriminator no seleccionará credenciales/database arbitrariamente.

## DB-ORM-POLYMORPHIC-080

Class injection será imposible mediante DB discriminator.

## DB-ORM-POLYMORPHIC-081

Alias injection será validada contra registry y relationship target set.

## DB-ORM-POLYMORPHIC-082

Relationship integrity será distinta de authorization.

## DB-ORM-POLYMORPHIC-083

Mass assignment de type/id no será seguro por default.

## DB-ORM-POLYMORPHIC-084

API discriminator podrá ser distinto del DB morph alias.

## DB-ORM-POLYMORPHIC-085

PHP namespace refactor no requerirá data migration si alias permanece.

## DB-ORM-POLYMORPHIC-086

Persisted alias rename requerirá compatibility strategy.

## DB-ORM-POLYMORPHIC-087

Legacy aliases serán explícitos.

## DB-ORM-POLYMORPHIC-088

Legacy alias podrá ser read-only mientras canonical alias se usa para writes.

## DB-ORM-POLYMORPHIC-089

Alias migration podrá requerir data migration.

## DB-ORM-POLYMORPHIC-090

Schema Diff no sustituirá semantic data migration detection.

## DB-ORM-POLYMORPHIC-091

Zero-downtime alias migration podrá utilizar dual-read/canonical-write.

## DB-ORM-POLYMORPHIC-092

Telemetry podrá detectar legacy alias usage.

## DB-ORM-POLYMORPHIC-093

Telemetry no expondrá target IDs.

## DB-ORM-POLYMORPHIC-094

Resource governance limitará polymorphic query fan-out.

## DB-ORM-POLYMORPHIC-095

Batch query count estará relacionado con distinct target types.

## DB-ORM-POLYMORPHIC-096

Open registries no implicarán cargar todos los tipos.

## DB-ORM-POLYMORPHIC-097

Normal FK será preferido cuando target type sea único y estable.

## DB-ORM-POLYMORPHIC-098

Universal entity reference será considerado anti-pattern por default.

## DB-ORM-POLYMORPHIC-099

Polymorphic relationships deberán ser intencionales y acotadas.

## DB-ORM-POLYMORPHIC-100

Extensions registrarán aliases durante bootstrap.

## DB-ORM-POLYMORPHIC-101

Alias conflicts de extensiones fallarán determinísticamente.

## DB-ORM-POLYMORPHIC-102

Plugins no podrán habilitar arbitrary class instantiation desde DB.

## DB-ORM-POLYMORPHIC-103

Driver no conocerá polymorphic relationships.

## DB-ORM-POLYMORPHIC-104

Connection no conocerá morph aliases.

## DB-ORM-POLYMORPHIC-105

Executor no interpretará discriminadores ORM.

## DB-ORM-POLYMORPHIC-106

SQL Compiler no resolverá EntityType desde DB strings.

## DB-ORM-POLYMORPHIC-107

Entity Query traducirá relationship semantics a Query Model.

## DB-ORM-POLYMORPHIC-108

Result Hydrator delegará entity materialization al EntityHydrator.

## DB-ORM-POLYMORPHIC-109

Persistence Planner respetará generated identity dependencies.

## DB-ORM-POLYMORPHIC-110

Persistence outcome UNKNOWN permanecerá UNKNOWN.

## DB-ORM-POLYMORPHIC-111

Flush success será distinto de transaction commit.

## DB-ORM-POLYMORPHIC-112

Commit uncertainty no será convertida en durable reference.

## DB-ORM-POLYMORPHIC-113

Rollback no rebobinará automáticamente object graph.

## DB-ORM-POLYMORPHIC-114

Refresh de dirty relationship será policy-driven.

## DB-ORM-POLYMORPHIC-115

Raw external mutations podrán marcar relationship knowledge STALE.

## DB-ORM-POLYMORPHIC-116

Cancellation no fabricará complete relationship state.

## DB-ORM-POLYMORPHIC-117

Partial batch failure preservará per-group certainty cuando sea conocida.

## DB-ORM-POLYMORPHIC-118

Failure de un target type no se convertirá automáticamente en failure conocido de otros grupos exitosos.

## DB-ORM-POLYMORPHIC-119

Target type resolution será deterministic.

## DB-ORM-POLYMORPHIC-120

Morph aliases utilizarán canonical comparison definida.

## DB-ORM-POLYMORPHIC-121

Alias normalization no será locale-dependent.

## DB-ORM-POLYMORPHIC-122

Alias values tendrán límites de longitud configurables.

## DB-ORM-POLYMORPHIC-123

Malformed discriminator será distinto de unknown valid discriminator.

## DB-ORM-POLYMORPHIC-124

Malformed identifiers serán distintos de missing targets.

## DB-ORM-POLYMORPHIC-125

Known type + nonexistent ID será distinto de unknown type.

## DB-ORM-POLYMORPHIC-126

Missing target policy será distinta de discriminator policy.

## DB-ORM-POLYMORPHIC-127

Target-not-found no será convertido automáticamente a null.

## DB-ORM-POLYMORPHIC-128

Soft-deleted target semantics serán policy-driven.

## DB-ORM-POLYMORPHIC-129

Temporal target semantics serán policy-driven.

## DB-ORM-POLYMORPHIC-130

Type resolution no ejecutará hidden DB I/O.

## DB-ORM-POLYMORPHIC-131

Morph registry lookup será memory-local sobre metadata compilada.

## DB-ORM-POLYMORPHIC-132

Relationship load podrá ejecutar I/O únicamente mediante Query Engine.

## DB-ORM-POLYMORPHIC-133

Entity reference object no retendrá EntityManager.

## DB-ORM-POLYMORPHIC-134

Entity reference object no será process-global mutable state.

## DB-ORM-POLYMORPHIC-135

Persistent runtime reset liberará batch/load state.

## DB-ORM-POLYMORPHIC-136

Metadata generations no serán mutadas in-place.

## DB-ORM-POLYMORPHIC-137

Old compiled plans podrán terminar su scope sin observar nueva generation.

## DB-ORM-POLYMORPHIC-138

New scopes utilizarán la generación vigente.

## DB-ORM-POLYMORPHIC-139

Mapping validation fallará temprano ante alias incompatibles.

## DB-ORM-POLYMORPHIC-140

Relationship target types deberán ser entidades ORM válidas.

## DB-ORM-POLYMORPHIC-141

Polymorphic type metadata no almacenará tenant mutable state.

## DB-ORM-POLYMORPHIC-142

Morph registry no almacenará entity instances.

## DB-ORM-POLYMORPHIC-143

Reverse registry no almacenará EntityManager.

## DB-ORM-POLYMORPHIC-144

Batch grouping no mezclará incompatible DatabaseContexts.

## DB-ORM-POLYMORPHIC-145

Heterogeneous result cardinality será gobernada por ResultHydrator.

## DB-ORM-POLYMORPHIC-146

Tuple cardinality no será deduplicada por target entity automáticamente.

## DB-ORM-POLYMORPHIC-147

Polymorphic collection completeness será independiente de target-type count.

## DB-ORM-POLYMORPHIC-148

Loaded one target type no probará completeness de una heterogeneous collection.

## DB-ORM-POLYMORPHIC-149

Collection snapshot preservará target EntityType dentro de EntityKey.

## DB-ORM-POLYMORPHIC-150

Polymorphic relationship system preservará explícitamente type, identity, context, loading state y persistence certainty.

---

# 231. Missing Target Semantics

Existe otra situación distinta:

```text
type = post
id   = 42
```

el alias es válido, pero:

```text
Post#42 does not exist
```

---

# 232. Missing Target ≠ Unknown Type

```text
UnknownType
≠
MissingTarget
```

---

# 233. Missing Target Policy

Propuesta:

```php
enum MissingPolymorphicTargetPolicy
{
    case FAIL;
    case RETURN_NULL;
    case RETURN_REFERENCE;
    case REPORT_STALE_REFERENCE;
}
```

---

# 234. Recommended Default

Para relaciones obligatorias:

```text
FAIL
```

Para nullable relationships, no deberá asumirse automáticamente que dangling reference equivale a null.

---

# 235. Referential Integrity Diagnostics

VoltStack podrá ofrecer una operación de diagnóstico que detecte:

```text
known alias
+
identifier
+
missing target
```

---

# 236. Integrity Scanner

Esto pertenece a diagnostics/maintenance, no al hot path normal.

---

# 237. Soft Deletes

Si `Post#42` está soft-deleted:

```text
exists physically
but hidden by default scope
```

no es necesariamente un dangling reference.

---

# 238. Soft Delete Policy

Relationship loading deberá distinguir:

```text
NOT_FOUND
FILTERED
SOFT_DELETED
FORBIDDEN
UNKNOWN
```

cuando las capas disponibles puedan proporcionar dicha evidencia.

---

# 239. Authorization ≠ Not Found

Relationship System no deberá interpretar una denegación de autorización como ausencia física.

---

# 240. Temporal Targets

Con temporal/versioned entities:

```text
(type, id)
```

puede no ser suficiente para identificar una versión histórica.

Una extensión temporal podrá ampliar la referencia.

---

# 241. Extended Reference

Conceptualmente:

```text
PolymorphicReference
=
Type
+
EntityIdentity
+
OptionalTemporalQualifier
```

cuando el mapping lo declare.

No será default.

---

# 242. Performance Model

Resolución de alias debería aproximarse a:

```text
O(1)
```

mediante lookup compilado.

---

# 243. Batch Grouping Complexity

Para `N` referencias:

```text
Tgroup ≈ O(N)
```

---

# 244. Query Complexity

Con `K` tipos de destino distintos:

```text
TargetQueryCount ≈ O(K)
```

para estrategia grouped-select.

---

# 245. Memory

```text
Mbatch ≈ O(N)
```

si todas las referencias se bufferizan.

Batching deberá permitir límites.

---

# 246. Streaming Morph-To

Puede procesarse por ventanas:

```text
read source batch
→ group target references
→ load target groups
→ hydrate
→ yield completed roots
```

---

# 247. Streaming Trade-Off

Batch demasiado pequeño:

```text
more target queries
```

Batch demasiado grande:

```text
more memory
```

---

# 248. Adaptive Batch Size

Podrá existir posteriormente una política basada en:

- memory budget;
- target type distribution;
- deadline;
- parameter limits.

No deberá modificar semántica.

---

# 249. Directory Structure

```text
src/Quantum/Database/ORM/Relationship/Polymorphic/
│
├── Contract/
│   ├── MorphTypeRegistry.php
│   ├── PolymorphicTypeResolver.php
│   ├── PolymorphicRelationshipLoader.php
│   └── PolymorphicReverseRelationshipRegistry.php
│
├── Metadata/
│   ├── PolymorphicRelationshipMetadata.php
│   ├── MorphToMetadata.php
│   ├── MorphOneMetadata.php
│   ├── MorphManyMetadata.php
│   ├── PolymorphicManyToManyMetadata.php
│   ├── PolymorphicTypeBinding.php
│   └── PolymorphicIdentifierBinding.php
│
├── Type/
│   ├── PolymorphicTypeId.php
│   ├── MorphTypeNamespace.php
│   ├── MorphAlias.php
│   ├── LegacyMorphAlias.php
│   └── MorphRegistryGeneration.php
│
├── Reference/
│   ├── PolymorphicReference.php
│   ├── PolymorphicEntityReference.php
│   └── PolymorphicTargetSet.php
│
├── Registry/
│   ├── CompiledMorphTypeRegistry.php
│   ├── CompiledReverseRelationshipRegistry.php
│   └── MorphRegistryBuilder.php
│
├── Hydration/
│   ├── PolymorphicReferenceHydrationNode.php
│   ├── PolymorphicIdentifierHydrationBinding.php
│   ├── MorphTypeResolverPlan.php
│   └── PolymorphicNullabilityPlan.php
│
├── Loading/
│   ├── PolymorphicBatchLoader.php
│   ├── PolymorphicBatchKey.php
│   ├── PolymorphicLoadPlan.php
│   └── PolymorphicLoadPolicy.php
│
├── ChangeTracking/
│   └── PolymorphicReferenceChangeSet.php
│
├── Persistence/
│   ├── PolymorphicReferencePersistenceOperation.php
│   └── PolymorphicPersistencePlanner.php
│
├── Migration/
│   ├── MorphAliasCompatibilityMap.php
│   ├── MorphAliasMigrationPlan.php
│   └── LegacyMorphAliasPolicy.php
│
├── Validation/
│   ├── PolymorphicMappingValidator.php
│   ├── MorphAliasValidator.php
│   └── PolymorphicReferenceValidator.php
│
├── Diagnostics/
│   ├── PolymorphicRelationshipDiagnostics.php
│   └── PolymorphicIntegrityScanner.php
│
├── Telemetry/
│   └── PolymorphicRelationshipTelemetry.php
│
└── Exception/
    └── ...
```

---

# 250. Dependency Model

```text
               Entity Metadata
                     │
                     ▼
             Morph Type Registry
                     │
                     ▼
      Polymorphic Relationship Metadata
          │                      │
          ▼                      ▼
     Entity Query          Hydration Plan
          │                      │
          ▼                      ▼
     Query Engine       Polymorphic Resolver
          │                      │
          ▼                      ▼
      Execution            Entity Hydrator
                                 │
                                 ▼
                            IdentityMap
```

Persistence:

```text
Entity Mutation
      ↓
Relationship Change Tracking
      ↓
UnitOfWork
      ↓
Persistence Planner
      ↓
Polymorphic Reference Operation
      ↓
Query Model
      ↓
Query Engine
      ↓
Compiler
      ↓
Executor
```

---

# 251. Core Separation

```text
Morph Alias
≠
PHP Class Name
≠
EntityType
≠
EntityKey
≠
Target Identifier
≠
Relationship Metadata
≠
Hydration Plan
≠
Runtime Entity Instance
```

---

# 252. Knowledge Model

Una relación polimórfica puede tener distintos grados de conocimiento:

```text
TYPE_UNKNOWN
TYPE_KNOWN
REFERENCE_KNOWN
TARGET_REFERENCE_ONLY
TARGET_LOADED
TARGET_MISSING
TARGET_STALE
TARGET_UNCERTAIN
```

---

# 253. Type Knowledge ≠ Target Knowledge

Resolver:

```text
post
→ Post
```

no demuestra que:

```text
Post#42
```

exista.

---

# 254. Loaded Target ≠ Durable Relationship

Tener target en memoria tampoco demuestra que la referencia haya sido committeada.

---

# 255. Persistence Certainty

Se reutilizará el modelo del documento 134:

```text
SUCCESS
FAILED
UNKNOWN
```

y sus dominios de consistencia.

---

# 256. Unknown Persistence

Si un UPDATE que cambia:

```text
post#42
→
video#7
```

tiene outcome desconocido:

```text
DatabaseRelationshipReality = UNKNOWN
```

---

# 257. No Fabricated Reconciliation

No deberá avanzarse snapshot a `video#7` como hecho conocido.

---

# 258. Taint

Un resultado suficientemente ambiguo podrá marcar:

```text
EntityManager = TAINTED
```

según Persistence Consistency Policy.

---

# 259. Final Formula — Resolution

```text
ResolvedTargetType
=
MorphRegistry.Resolve(
    Canonicalize(RawDiscriminator)
)
```

con:

```text
ResolvedTargetType
∈
AllowedTargetSet
```

---

# 260. Final Formula — Reference

```text
PolymorphicReference
=
(
    ResolvedEntityType,
    CanonicalIdentifier,
    EffectiveIdentityContext
)
```

---

# 261. Final Formula — Null

```text
ReferenceAbsent
⇔
TypeNull
∧
IdentifierFullyNull
```

---

# 262. Final Formula — Validity

```text
ValidReference
=
KnownMorphType
∧
AllowedTargetType
∧
CompleteIdentifier
∧
CompatibleIdentifierType
∧
CompatibleDatabaseContext
∧
TenantShardPolicySatisfied
```

---

# 263. Final Formula — Batch Loading

```text
BatchGroups
=
GroupBy(
    References,
    TargetEntityType
    × DatabaseContext
    × TenantContext
    × ShardContext
)
```

---

# 264. Final Formula — Persistence

```text
PersistedReference(entity)
=
(
    CanonicalMorphAlias(EntityType(entity)),
    CanonicalPersistentIdentifier(entity)
)
```

---

# 265. Final Formula — Safe Hydration

```text
SafePolymorphicHydration
=
ValidatedDiscriminator
∧
ResolvedEntityType
∧
AllowedTarget
∧
CompleteCanonicalIdentifier
∧
IdentityMapCanonicality
∧
HydrationPlanCompliance
```

---

# 266. Final Formula — Runtime Safety

```text
SafePersistentRuntime
=
ImmutableMorphRegistry
∧
ImmutableRelationshipMetadata
∧
ImmutableHydrationPlans
∧
ScopedIdentityMap
∧
ScopedUnitOfWork
∧
ScopedLoadQueues
∧
ScopedHydrationState
∧
DeterministicReset
```

---

# 267. Master Formula

```text
Database Polymorphic Relationship System
=
Stable Morph Aliases
+
Morph Type Registry
+
Polymorphic Type Namespaces
+
Allowed Target Sets
+
Type Discriminator Bindings
+
Identifier Bindings
+
Composite Identity Support
+
Polymorphic References
+
Canonical EntityKey Integration
+
IdentityMap Reuse
+
MorphTo
+
MorphOne
+
MorphMany
+
Polymorphic ManyToMany
+
Inverse Relationship Resolution
+
Semantic Query Predicates
+
Hydration Plans
+
Target-Specific Hydration
+
Lazy Loading
+
Eager Loading
+
Type-Grouped Batch Loading
+
N+1 Prevention
+
Relationship Change Tracking
+
Persistence Planning
+
Generated Identity Barriers
+
Cascade Policies
+
Missing Target Policies
+
Schema Diagnostics
+
Index/Uniqueness Recommendations
+
Alias Migration Compatibility
+
Legacy Alias Resolution
+
Zero-Downtime Alias Migration
+
Tenant Isolation
+
Shard Isolation
+
Security
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

# 268. Regla maestra final

> **VoltStack deberá tratar una referencia polimórfica como una referencia tipada y controlada por metadata, no como la combinación insegura de un nombre de clase almacenado en texto y un identificador arbitrario.**

La arquitectura preservará:

```text
Persisted Morph Alias
        ↓
Validated Morph Registry
        ↓
Canonical EntityType
        ↓
Canonical EntityKey
        ↓
IdentityMap
        ↓
Canonical Managed Entity
```

Nunca:

```text
Database String
      ↓
class_exists()
      ↓
new $class()
```

---

# 269. Resultado arquitectónico

La relación:

```text
Comment
    ↓
(post, 42)
```

será interpretada como:

```text
Raw Discriminator
    ↓
Canonicalization
    ↓
MorphTypeRegistry
    ↓
EntityType(Post)
    ↓
AllowedTarget Validation
    ↓
Canonical Identifier(42)
    ↓
EntityKey(Post#42, Context)
    ↓
IdentityMap
    ↓
Post#42
```

y para múltiples targets:

```text
Polymorphic References
        ↓
Group by EntityType + Context
        │
        ├── Post IDs
        ├── Video IDs
        └── Photo IDs
        ↓
Entity Query
        ↓
Query Engine
        ↓
Execution
        ↓
Entity Hydration
        ↓
IdentityMap
        ↓
Relationship Assembly
```

De esta manera VoltStack obtiene ergonomía similar a sistemas ORM modernos sin sacrificar:

- estabilidad frente a refactors;
- seguridad;
- identidad canónica;
- type safety;
- arquitectura Query Engine;
- persistencia consistente;
- aislamiento multi-tenant;
- compatibilidad con persistent runtimes.

---

# 270. Siguiente documento

```text
148_DATABASE_RELATIONSHIP_METADATA_SYSTEM.md
```

El siguiente documento consolidará el modelo canónico de metadata utilizado por todas las relaciones:

```text
one-to-one
one-to-many
many-to-one
many-to-many
polymorphic
```

y definirá:

```text
RelationshipId
RelationshipMetadata
RelationshipKind
RelationshipOwnership
RelationshipDirection
Source/Target EntityType
Join Metadata
Foreign Key Bindings
Collection Semantics
Fetch Strategy
Cascade Policy
Orphan Policy
Nullability
Ordering
Filtering
Inverse Mapping
Polymorphic Metadata
Loading Requirements
Persistence Requirements
Hydration Bindings
Generation/Fingerprinting
Compilation
Validation
Registry
Caching
Extensions
Persistent Runtime Safety
```

con la regla central:

> **`RelationshipMetadata` será la descripción ORM canónica, immutable y compilada de una relación; loaders, hydrators, UnitOfWork y persistence planners consumirán esa metadata, pero ninguno deberá redescubrir mappings mediante reflection o convenciones durante el hot path.**