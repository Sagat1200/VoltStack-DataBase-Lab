# 114_DATABASE_MODEL_API_SYSTEM.md

# VoltStack Quantum Database
## Database Model API System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 114 — Database Model API System  
**Bloque:** 10 — ORM  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Model API System` define la API de alto nivel orientada al desarrollador para trabajar con entidades persistentes mediante una experiencia similar a Active Record, sin convertir `Model` en un segundo ORM.

VoltStack deberá permitir una experiencia como:

```php
$user = User::find(42);

$user->name = 'Ana';

$user->save();
```

o:

```php
$users = User::query()
    ->where('active', true)
    ->orderBy('name')
    ->get();
```

manteniendo internamente:

```text
Model API
    ↓
ORM Core
    ↓
EntityManager
    ↓
UnitOfWork
    ↓
Persistence Engine
    ↓
Query Engine
```

El principio central será:

> **Model es una API ergonómica sobre el ORM Core; no es un segundo ORM ni un Persistence Engine alternativo.**

---

# 2. Objetivos

El sistema deberá proporcionar:

- API sencilla para aplicaciones;
- consultas mediante `Model::query()`;
- `find()`;
- `findOrFail()`;
- `create()`;
- `save()`;
- `delete()`;
- `refresh()`;
- `fresh()`;
- `replicate()`;
- acceso ergonómico a atributos;
- mass assignment controlado;
- integración con casts;
- acceso a relaciones;
- scopes;
- timestamps;
- soft-delete integration;
- serialización;
- acceso estático seguro;
- integración con EntityManager;
- integración con UnitOfWork;
- soporte para persistent runtimes;
- extensibilidad.

---

# 3. No objetivos

`Model API` no deberá:

- generar SQL;
- ejecutar SQL directamente;
- administrar conexiones;
- implementar IdentityMap propio;
- implementar UnitOfWork propio;
- mantener un ORM alternativo;
- interpretar Schema;
- controlar transacciones directamente;
- almacenar EntityManager global;
- mantener estado mutable estático entre requests.

---

# 4. Arquitectura

```text
Application
    │
    ▼
Model API
    │
    ├── Static API
    ├── Instance API
    ├── Attribute API
    ├── Relationship API
    ├── Scope API
    └── Serialization API
            │
            ▼
      Model Context
            │
            ├── Query Bridge
            ├── Persistence Bridge
            ├── Metadata Bridge
            └── Relationship Bridge
                    │
                    ▼
               ORM Core
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
 EntityManager  UnitOfWork  Metadata
        │
        ▼
 Persistence Engine
        │
        ▼
 Query Engine
```

---

# 5. Model ≠ ORM

Regla fundamental:

```text
Model API
≠
ORM Core
```

`Model` será una fachada orientada a objetos sobre:

```text
EntityManager
Repository
EntityQuery
UnitOfWork
Metadata
Persistence Engine
```

---

# 6. Model ≠ EntityManager

Incorrecto:

```php
class Model
{
    public static array $identityMap = [];

    public function save()
    {
        // custom persistence engine
    }
}
```

Correcto:

```text
Model::save()
      ↓
ModelPersistenceBridge
      ↓
EntityManager
      ↓
UnitOfWork
```

---

# 7. Model como Entity

Una clase:

```php
final class User extends Model
{
}
```

continúa siendo una:

```text
Entity
```

desde la perspectiva del ORM.

Por tanto:

```text
Model
⊂
Entity-capable objects
```

pero:

```text
Entity
⊄
Model
```

porque también se soportarán entidades POPO.

---

# 8. Dos estilos sobre un ORM

VoltStack permitirá:

```text
Style A
────────────────────────────

class User extends Model

User::find(42);
$user->save();


Style B
────────────────────────────

#[Entity]
class User {}

$entityManager->find(User::class, 42);
$entityManager->persist($user);
$entityManager->flush();
```

Ambos convergen a:

```text
same EntityMetadata
same IdentityMap
same UnitOfWork
same Persistence Engine
same Query Engine
```

---

# 9. Regla de convergencia

Formalmente:

```text
Persistence(ModelEntity)
=
Persistence(PlainEntity)
=
ORM Persistence Engine
```

---

# 10. Clase base

Se propone:

```php
abstract class Model
{
    // ergonomic API only
}
```

Namespace:

```text
VoltStack\Quantum\Database\ORM\Model
```

o una exposición pública simplificada:

```text
VoltStack\Database\Model
```

---

# 11. Responsabilidad de Model

`Model` podrá ofrecer:

```text
query
find
findOrFail
create
save
delete
refresh
fresh
replicate
fill
forceFill
toArray
toJson
relationship helpers
scopes
```

pero deberá delegar operaciones ORM reales.

---

# 12. Model Context

Una instancia `Model` necesita acceder al contexto ORM actual sin almacenar permanentemente el EntityManager.

Se introduce:

```text
ModelContext
```

---

# 13. ModelContext

Conceptualmente:

```php
interface ModelContext
{
    public function entityManager(): EntityManager;

    public function metadata(): EntityMetadataRegistry;

    public function query(): ModelQueryBridge;

    public function persistence(): ModelPersistenceBridge;
}
```

---

# 14. ModelContext ≠ global singleton

Nunca:

```php
Model::$entityManager = $manager;
```

como estado global mutable.

---

# 15. Model Context Resolver

Se propone:

```php
interface ModelContextResolver
{
    public function resolve(): ModelContext;
}
```

---

# 16. Runtime resolution

En una petición:

```text
User::query()
     ↓
ModelContextResolver
     ↓
Current Request Scope
     ↓
EntityManager
```

---

# 17. FrankenPHP

En FrankenPHP:

```text
Worker
│
├── Request A
│   └── ModelContext A
│
├── RESET
│
└── Request B
    └── ModelContext B
```

Nunca:

```text
ModelContext A
→ Request B
```

---

# 18. Static API

La API estática será únicamente:

```text
ergonomic dispatch
```

hacia servicios scoped.

Ejemplo:

```php
User::query();
```

conceptualmente:

```text
User::query()
    ↓
ModelContextResolver
    ↓
ModelQueryBridge
    ↓
EntityQuery
```

---

# 19. Static ≠ static state

Regla:

> **VoltStack permitirá métodos estáticos sin permitir contexto ORM estático persistente.**

---

# 20. ModelQueryBridge

Contrato:

```php
interface ModelQueryBridge
{
    /**
     * @template TModel of Model
     *
     * @param class-string<TModel> $model
     *
     * @return ModelQuery<TModel>
     */
    public function query(string $model): ModelQuery;
}
```

---

# 21. query()

Ejemplo:

```php
$query = User::query();
```

retorna:

```text
ModelQuery<User>
```

No:

```text
SQL string
```

---

# 22. ModelQuery

`ModelQuery` será una API ORM-aware sobre el Query Engine.

```text
ModelQuery
    ↓
EntityQuery
    ↓
Entity Metadata
    ↓
Query Model / AST
```

---

# 23. ModelQuery ≠ QueryBuilder

Aunque compartan ergonomía:

```text
ModelQuery
≠
Low-Level QueryBuilder
```

porque `ModelQuery` conoce:

```text
entities
mapped fields
relationships
scopes
hydration
```

mientras QueryBuilder conoce:

```text
query structures
tables
columns
expressions
```

---

# 24. Ejemplo

```php
$users = User::query()
    ->where('status', UserStatus::ACTIVE)
    ->where('age', '>=', 18)
    ->orderBy('name')
    ->limit(50)
    ->get();
```

Pipeline:

```text
ModelQuery
   ↓
EntityQuery
   ↓
Entity Metadata
   ↓
Semantic Query Model
   ↓
Query AST
   ↓
Optimizer
   ↓
Planner
   ↓
Compiler
   ↓
Executor
   ↓
Hydrator
   ↓
IdentityMap
   ↓
User entities
```

---

# 25. find()

```php
$user = User::find(42);
```

equivale conceptualmente a:

```php
$user = $entityManager->find(
    User::class,
    42,
);
```

---

# 26. IdentityMap first

`find()` deberá consultar:

```text
IdentityMap
```

antes de cargar desde DB cuando corresponda.

```text
User::find(42)
      ↓
EntityManager
      ↓
EntityKey(User,42)
      ↓
IdentityMap
      ├── HIT
      │    ↓
      │  Entity
      │
      └── MISS
           ↓
         Query
```

---

# 27. find() consistency

Dentro del mismo scope:

```php
$a = User::find(42);
$b = User::find(42);

assert($a === $b);
```

cuando ambos representan la misma `EntityKey`.

---

# 28. findOrFail()

```php
$user = User::findOrFail(42);
```

si no existe:

```text
ModelNotFoundException
```

---

# 29. findOrNull()

Opcionalmente:

```php
$user = User::findOrNull(42);
```

puede ser alias explícito de semántica nullable.

---

# 30. findMany()

```php
$users = User::findMany([
    10,
    20,
    30,
]);
```

deberá utilizar una consulta agrupada cuando sea posible.

No:

```text
N independent SELECTs
```

por default.

---

# 31. first()

```php
$user = User::query()
    ->where('email', $email)
    ->first();
```

retorna:

```text
User|null
```

---

# 32. firstOrFail()

```php
$user = User::query()
    ->where('email', $email)
    ->firstOrFail();
```

---

# 33. exists()

```php
$exists = User::query()
    ->where('email', $email)
    ->exists();
```

deberá generar semántica de existencia eficiente.

No necesariamente:

```text
SELECT *
```

---

# 34. count()

```php
$count = User::query()
    ->where('active', true)
    ->count();
```

deberá permanecer en Query Engine.

No deberá hidratar entidades.

---

# 35. create()

```php
$user = User::create([
    'name' => 'Ana',
    'email' => 'ana@example.com',
]);
```

pipeline:

```text
Input
  ↓
Mass Assignment
  ↓
Model Construction
  ↓
Casts / Value Conversion
  ↓
EntityManager::persist()
  ↓
UnitOfWork
  ↓
Flush Policy
```

---

# 36. create() flush semantics

Debe decidirse explícitamente si:

```php
User::create(...)
```

hace `flush()` automáticamente.

Para ergonomía tipo Laravel, VoltStack puede hacerlo por default en `Model API`.

Pero internamente:

```text
create()
=
construct
+
persist
+
flush according to ModelWritePolicy
```

---

# 37. ModelWritePolicy

Se propone:

```text
AUTO_FLUSH
DEFERRED_FLUSH
EXPLICIT_FLUSH
```

---

# 38. Default

Para Model API:

```text
AUTO_FLUSH
```

puede ser el default amigable.

Para EntityManager:

```text
EXPLICIT_FLUSH
```

será el modelo natural.

---

# 39. Distinción importante

Por tanto:

```text
User::create()
```

puede ser conveniente.

Mientras:

```php
$entityManager->persist($user);
```

no deberá implicar necesariamente ejecución inmediata.

---

# 40. save()

```php
$user->save();
```

no deberá decidir directamente:

```text
INSERT vs UPDATE
```

basándose únicamente en `$id`.

---

# 41. save() pipeline

```text
Model::save()
      ↓
ModelPersistenceBridge
      ↓
EntityManager
      ↓
Entity State
      ↓
UnitOfWork
      ↓
Persistence Planner
      ↓
INSERT / UPDATE strategy
```

---

# 42. save() ≠ upsert

Regla:

```text
save()
≠
if id == null INSERT else UPDATE
```

---

# 43. New model

Para una entidad NEW:

```php
$user = new User();

$user->name = 'Ana';

$user->save();
```

el bridge registra la entidad con UnitOfWork.

---

# 44. Managed model

Para una entidad MANAGED:

```php
$user->name = 'Beatriz';

$user->save();
```

UnitOfWork determina cambios.

---

# 45. Detached model

`save()` sobre una entidad DETACHED deberá seguir una política explícita.

No deberá asumir automáticamente UPDATE.

---

# 46. DetachedSavePolicy

Opciones:

```text
REJECT
REATTACH_EXPLICITLY
MERGE_EXPLICITLY
```

Default recomendado:

```text
REJECT
```

---

# 47. delete()

```php
$user->delete();
```

pipeline:

```text
Model
 ↓
PersistenceBridge
 ↓
EntityManager::remove()
 ↓
UnitOfWork
 ↓
Persistence Engine
```

---

# 48. delete() result

Se recomienda:

```php
$user->delete();
```

retorne un resultado tipado:

```text
DeleteResult
```

o `bool` en la API simplificada.

Internamente deberá existir un resultado rico.

---

# 49. delete() ≠ immediate row deletion

Dependiendo de write policy:

```text
delete()
```

puede significar:

```text
schedule removal
+
flush
```

---

# 50. forceDelete()

Con Soft Delete:

```php
$user->forceDelete();
```

será una operación explícita.

---

# 51. restore()

Para modelos soft-deletable:

```php
$user->restore();
```

---

# 52. Soft delete capability

No todo Model deberá incluir realmente soft delete.

Será capability/trait/configuration.

---

# 53. refresh()

```php
$user->refresh();
```

significa:

> Recargar el estado persistente dentro de la misma instancia managed.

---

# 54. refresh pipeline

```text
$user
 ↓
EntityKey
 ↓
EntityManager::refresh()
 ↓
DB
 ↓
Hydration
 ↓
same object instance
```

---

# 55. refresh identity

Después:

```php
$before = $user;

$user->refresh();

assert($before === $user);
```

---

# 56. refresh dirty state

Por default, `refresh()` reemplazará cambios locales no persistidos.

Esto deberá ser explícito en documentación y tooling.

---

# 57. Safe refresh

Podrá existir:

```php
$user->refresh(
    policy: RefreshPolicy::REJECT_IF_DIRTY
);
```

---

# 58. fresh()

```php
$fresh = $user->fresh();
```

conceptualmente obtiene una representación actual desde persistencia.

---

# 59. fresh() y IdentityMap

Existe una diferencia importante respecto a Laravel tradicional.

Dentro de un ORM con IdentityMap fuerte:

```text
same EntityKey
→ same managed object
```

Por tanto `fresh()` no deberá crear silenciosamente una segunda managed instance.

---

# 60. Fresh strategies

Se proponen:

```text
REFRESH_MANAGED
DETACHED_COPY
SNAPSHOT
```

---

# 61. Default fresh()

La opción más coherente:

```text
fresh()
→ detached refreshed representation
```

o devolver un snapshot.

Pero deberá evitar romper IdentityMap.

---

# 62. API recomendada

Para reducir ambigüedad:

```php
$user->refresh();

$snapshot = $user->freshSnapshot();

$copy = $user->freshDetached();
```

`fresh()` podrá mantenerse como convenience alias documentado.

---

# 63. replicate()

```php
$copy = $user->replicate();
```

deberá producir:

```text
new entity
+
copied replicable state
-
persistent identity
-
ORM management state
```

---

# 64. replicate() identity rule

Nunca:

```text
copy EntityKey
```

---

# 65. Replication policy

No todos los campos deben copiarse.

Ejemplo:

```text
id                NO
version           NO
created_at        NO
updated_at        NO
tenant_id         CONTEXT/POLICY
business fields   YES
relationships     POLICY
```

---

# 66. replicate() ≠ clone

`clone` conserva arbitrariamente estado PHP.

`replicate()` aplica metadata y política ORM.

---

# 67. Attribute API

Model podrá ofrecer acceso ergonómico:

```php
$user->name;
$user->email;
```

pero la arquitectura deberá distinguir:

```text
PHP property
Mapped attribute
Computed attribute
Relationship
Virtual attribute
```

---

# 68. Attribute descriptor

Metadata será la fuente autoritativa.

```text
Model
 ↓
EntityMetadata
 ↓
AttributeMetadata
```

---

# 69. AttributeBag

Podrá existir una abstracción interna:

```text
ModelAttributeBag
```

pero no será obligatoria para todas las entidades.

---

# 70. Typed properties

VoltStack deberá favorecer:

```php
private string $name;
private Email $email;
```

sobre un único:

```php
protected array $attributes = [];
```

cuando sea posible.

---

# 71. Dynamic attributes

Para DX/legacy podrán soportarse modelos dinámicos.

Pero:

```text
Dynamic Model
```

será una capacidad adicional.

No el único modelo ORM.

---

# 72. Property access

Podrán coexistir:

```text
DIRECT_PROPERTY
ACCESSOR_METHOD
COMPILED_ACCESSOR
MAGIC_ACCESS
CUSTOM
```

---

# 73. Magic access

`__get()` y `__set()` podrán ofrecer ergonomía.

No deberán convertirse en el único mecanismo interno.

---

# 74. Attribute resolution pipeline

```text
$user->name
      ↓
Model Attribute Resolver
      ├── mapped property
      ├── accessor
      ├── cast
      ├── virtual attribute
      └── relationship
```

---

# 75. Ambiguity detection

Si:

```text
name
```

es simultáneamente:

```text
mapped field
relationship
virtual attribute
```

sin política clara:

```text
MetadataConflictException
```

---

# 76. Accessors

Ejemplo:

```php
protected function displayName(): Attribute
{
    return Attribute::get(
        fn () => "{$this->firstName} {$this->lastName}"
    );
}
```

La forma exacta podrá evolucionar.

---

# 77. Mutators

```php
protected function email(): Attribute
{
    return Attribute::make(
        set: fn (string $value) => Email::fromString($value),
    );
}
```

---

# 78. Accessor ≠ database type conversion

Se distinguirá:

```text
ORM Type Conversion
```

de:

```text
Model Accessor/Mutator
```

---

# 79. Type conversion

Ejemplo:

```text
VARCHAR
→ Email Value Object
```

pertenece principalmente al Type/Mapping System.

---

# 80. Presentation accessor

Ejemplo:

```text
first_name + last_name
→ display_name
```

pertenece a Model API.

---

# 81. Casts

Model API podrá exponer:

```php
protected function casts(): array
{
    return [
        'active' => 'bool',
        'settings' => 'json',
        'status' => UserStatus::class,
    ];
}
```

---

# 82. Cast architecture

```text
Model Cast Declaration
       ↓
Entity Metadata
       ↓
Type / Casting System
```

No deberá existir un cast engine totalmente separado.

---

# 83. Cast ≠ accessor

```text
Cast
```

convierte representación tipada.

```text
Accessor
```

puede calcular/componer comportamiento de presentación.

---

# 84. Mass assignment

```php
User::create($request->all());
```

no deberá permitir asignación arbitraria.

---

# 85. MassAssignmentPolicy

Se propone:

```text
ALLOW_LIST
DENY_LIST
STRICT_EXPLICIT
DISABLED
```

---

# 86. Default seguro

Recomendado:

```text
STRICT_EXPLICIT
```

o:

```text
ALLOW_LIST
```

para inputs externos.

---

# 87. fillable()

Ejemplo:

```php
protected function fillable(): array
{
    return [
        'name',
        'email',
    ];
}
```

---

# 88. guarded()

También podrá existir:

```php
protected function guarded(): array
{
    return [
        'id',
        'role',
        'tenant_id',
    ];
}
```

pero `fillable` explícito es más seguro.

---

# 89. fill()

```php
$user->fill([
    'name' => 'Ana',
    'role' => 'admin',
]);
```

si `role` no está permitido:

```text
MassAssignmentException
```

en strict mode.

---

# 90. Silent discard

No se recomienda descartar campos prohibidos silenciosamente por default.

---

# 91. forceFill()

```php
$user->forceFill([...]);
```

será un escape hatch explícito.

---

# 92. forceFill() ≠ authorization bypass

Aunque omita mass-assignment policy:

```text
forceFill
≠
permission granted
```

---

# 93. Mass assignment ≠ validation

Regla:

```text
Mass Assignment
≠
Validation
≠
Authorization
```

---

# 94. Input pipeline

Idealmente:

```text
HTTP Input
   ↓
Validation
   ↓
Authorization
   ↓
Application DTO / Command
   ↓
Model fill/create
```

---

# 95. Dirty API

Model podrá ofrecer:

```php
$user->isDirty();
$user->isDirty('email');
$user->isClean();
$user->getDirty();
$user->getOriginal('email');
```

---

# 96. Source of truth

Estos métodos consultarán:

```text
UnitOfWork
```

o su snapshot/change-tracking infrastructure.

No mantendrán un segundo dirty tracker.

---

# 97. getOriginal()

```php
$user->getOriginal('email');
```

deberá leer la snapshot administrada.

---

# 98. Dirty API on detached entity

Deberá existir semántica clara.

Posiblemente:

```text
DetachedEntityStateUnavailableException
```

si no existe snapshot válida.

---

# 99. wasChanged()

Podrá representar cambios confirmados durante el último flush del scope.

No deberá depender de flags eternos en la instancia.

---

# 100. timestamps

Model API podrá ofrecer:

```text
created_at
updated_at
```

automáticos.

---

# 101. Timestamp policy

Será una capability:

```text
TimestampBehavior
```

No una obligación universal de Entity.

---

# 102. Timestamps ≠ ORM state

Los timestamps son campos persistentes.

No sustituyen snapshots/versioning.

---

# 103. Clock abstraction

Nunca:

```php
new DateTimeImmutable();
```

disperso por Model internals.

Usar:

```text
Clock
```

inyectable.

---

# 104. updated_at

El valor deberá determinarse de forma consistente dentro de la operación/flush.

---

# 105. created_at

Se asignará según política cuando la entidad entra a persistencia por primera vez.

---

# 106. Database timestamps

Podrá soportarse:

```text
APPLICATION_GENERATED
DATABASE_GENERATED
```

como estrategias diferentes.

---

# 107. Scopes

Ejemplo:

```php
User::query()
    ->active()
    ->verified()
    ->get();
```

---

# 108. Local scope

Podrá declararse:

```php
public function scopeActive(ModelQuery $query): void
{
    $query->where('active', true);
}
```

o mediante una API/attribute más tipada.

---

# 109. Scope ≠ raw SQL macro

Un scope deberá modificar:

```text
EntityQuery / Query Model
```

no concatenar SQL.

---

# 110. Global scopes

Ejemplos:

```text
SoftDeleteScope
TenantScope
VisibilityScope
```

---

# 111. Global scope ordering

Será:

```text
deterministic
explicit
inspectable
```

---

# 112. Global scope bypass

Deberá ser explícito:

```php
User::withoutGlobalScope(SoftDeleteScope::class);
```

---

# 113. Security scopes

Un global scope no deberá considerarse automáticamente una frontera suficiente de autorización.

---

# 114. Scope registry

```text
ModelScopeRegistry
```

deberá congelarse después de bootstrap.

---

# 115. Query macros

La extensión de queries podrá hacerse mediante registry.

Nunca mediante mutación global impredecible durante requests.

---

# 116. Relationships

Model API ofrecerá ergonomía:

```php
$user->posts;
```

o:

```php
$user->posts();
```

---

# 117. Relationship definition

Ejemplo:

```php
public function posts(): HasMany
{
    return $this->hasMany(Post::class);
}
```

Pero internamente deberá convertirse en:

```text
Relationship Metadata
```

---

# 118. Relationship method ≠ query execution

Declarar:

```php
posts()
```

no deberá ejecutar inmediatamente la consulta.

---

# 119. Relationship access

```php
$user->posts
```

puede activar lazy loading si la política lo permite.

---

# 120. Explicit loading

También:

```php
$user->load('posts');
```

---

# 121. Lazy loading policy

Se proponen:

```text
ALLOW
WARN
FORBID
```

---

# 122. Production default

Podrá permitirse:

```text
ALLOW
```

con N+1 telemetry.

Para aplicaciones estrictas:

```text
FORBID
```

---

# 123. Detached relationship access

Una entidad detached no deberá encontrar mágicamente un EntityManager global para lazy-load.

---

# 124. Relation query

```php
$user->posts()
    ->where('published', true)
    ->get();
```

deberá utilizar `EntityQuery`.

---

# 125. Relationship collections

`$user->posts` podrá devolver:

```text
EntityCollection<Post>
```

---

# 126. Collection ≠ array

La colección podrá mantener información de:

```text
loading state
relationship identity
batch loading
change tracking
```

---

# 127. Collection API

Ejemplos:

```php
$user->posts->count();
$user->posts->contains($post);
$user->posts->add($post);
$user->posts->remove($post);
```

---

# 128. Relationship mutation

Modificar una colección deberá notificar al UnitOfWork mediante infraestructura ORM.

No deberá ejecutar SQL inmediatamente.

---

# 129. with()

Eager loading:

```php
$users = User::query()
    ->with('roles', 'profile')
    ->get();
```

---

# 130. Nested eager loading

```php
User::query()
    ->with('posts.comments.author')
    ->get();
```

deberá compilarse a un:

```text
Entity Loading Plan
```

---

# 131. N+1 integration

Model API deberá integrarse con:

```text
DATABASE_N_PLUS_ONE_DETECTION_SYSTEM
```

posterior.

---

# 132. Serialization

```php
$user->toArray();
$user->toJson();
```

serán funciones de presentación.

No persistence serialization.

---

# 133. Serialization profile

Se propone:

```text
SerializationProfile
```

para controlar:

```text
visible
hidden
relationships
computed fields
depth
sensitive fields
```

---

# 134. hidden()

Ejemplo:

```php
protected function hidden(): array
{
    return [
        'passwordHash',
        'rememberToken',
    ];
}
```

---

# 135. visible()

También:

```php
protected function visible(): array
{
    return [
        'id',
        'name',
        'email',
    ];
}
```

---

# 136. Hidden ≠ unmapped

Un campo oculto puede continuar persistido.

---

# 137. Serialization ≠ authorization

Que un campo no esté hidden no implica que cualquier usuario pueda verlo.

---

# 138. Circular relationships

Serialización deberá detectar:

```text
User
→ Posts
→ User
→ Posts
...
```

---

# 139. Serialization depth

Deberán existir límites explícitos.

---

# 140. Lazy loading during serialization

Default recomendado:

> La serialización no deberá activar relaciones lazy de forma accidental.

---

# 141. Serialization loading policy

```text
LOADED_ONLY
EXPLICIT
ALLOW_LAZY
```

Default:

```text
LOADED_ONLY
```

---

# 142. JSON

`JsonSerializable` podrá integrarse, pero deberá respetar perfiles de seguridad.

---

# 143. toArray() ≠ debug dump

Debug tooling tendrá una representación diferente.

---

# 144. Model events

Model API podrá exponer ergonomía:

```text
creating
created
updating
updated
saving
saved
deleting
deleted
restoring
restored
```

---

# 145. Event mapping

Internamente deberán mapearse a:

```text
Entity Lifecycle Events
Persistence Events
```

según corresponda.

---

# 146. No duplicate event engine

Model no implementará otro Event Bus.

---

# 147. Event timing

Se distinguirá:

```text
before scheduling
before flush
after SQL execution
after transaction commit
```

No todos significan `saved`.

---

# 148. saved event

La semántica deberá especificar si:

```text
saved
```

significa:

```text
persistence executed
```

o:

```text
transaction committed
```

VoltStack deberá evitar nombres ambiguos internamente.

---

# 149. Public ergonomic aliases

Puede ofrecer:

```text
saved
```

pero mapear a eventos internos precisos.

---

# 150. Model observers

```php
#[Observe(UserObserver::class)]
class User extends Model
{
}
```

podrá integrarse con EventSystem.

---

# 151. Model boot methods

Patrones como:

```php
protected static function booted(): void
```

podrán ofrecerse por compatibilidad ergonómica.

Pero deberán ejecutarse durante bootstrap/metadata registration.

---

# 152. booted() ≠ every request mutation

No deberá modificar registries congelados después de bootstrap.

---

# 153. Model traits

Podrán existir:

```text
HasTimestamps
SoftDeletes
HasFactory
HasUuids
HasUlids
HasEvents
HasScopes
```

---

# 154. Trait architecture

Los traits serán:

```text
declarative convenience
```

sobre capabilities.

No sistemas paralelos.

---

# 155. HasFactory

`HasFactory` deberá integrar:

```text
Database Factory System
```

posterior.

---

# 156. SoftDeletes

`SoftDeletes` deberá integrar:

```text
Soft Delete System
```

posterior.

---

# 157. HasUuids

Deberá configurar:

```text
IdentifierGenerationStrategy
```

no generar IDs desde un segundo subsystem.

---

# 158. Model metadata hooks

Traits podrán contribuir metadata mediante:

```text
ModelMetadataContributor
```

durante bootstrap.

---

# 159. Trait collision

Dos traits que declaren configuración incompatible deberán producir error.

Nunca:

```text
last trait wins silently
```

---

# 160. Model configuration

Una clase podrá declarar:

```php
final class User extends Model
{
    protected static string $table = 'users';

    protected static string $primaryKey = 'id';
}
```

para DX.

---

# 161. Static configuration

Estos valores deberán tratarse como:

```text
immutable class metadata
```

después de compilación.

No como request state.

---

# 162. Preferred configuration

A largo plazo se favorecerá metadata tipada:

```php
#[Entity]
#[Table('users')]
final class User extends Model
{
}
```

pero se podrán soportar propiedades estáticas por ergonomía/migración.

---

# 163. table()

Podrá existir:

```php
User::tableName();
```

para introspección.

No deberá permitir cambiar la tabla globalmente durante runtime.

---

# 164. Dynamic table mutation

Evitar:

```php
User::setTable('users_' . $tenantId);
```

como mecanismo multitenant.

---

# 165. Tenant routing

La selección tenant/table/database pertenece a:

```text
Tenant Context
Connection Resolution
Mapping Context
Query Context
```

no a mutación estática de Model.

---

# 166. Connection selection

Podrá declararse un logical connection:

```php
#[Connection('analytics')]
```

pero:

```text
logical connection name
≠
live Connection object
```

---

# 167. getConnection()

Una API de compatibilidad puede devolver contexto resuelto.

Pero el Model no deberá almacenar la conexión.

---

# 168. Transaction API

No se recomienda:

```php
User::beginTransaction();
```

como API primaria.

Preferir:

```php
Database::transaction(function () {
    // ...
});
```

o:

```php
$entityManager->transactional(...);
```

---

# 169. Query shortcuts

Podrán existir:

```php
User::where(...)
User::orderBy(...)
User::with(...)
```

como forwarding hacia:

```php
User::query()
```

---

# 170. __callStatic()

Puede utilizarse como convenience.

Pero la API pública importante deberá ser discoverable por IDE mediante:

```text
explicit methods
generated stubs
generic annotations
```

---

# 171. Static forwarding rule

```text
User::where(...)
```

deberá equivaler a:

```text
User::query()->where(...)
```

sin mantener query state en la clase.

---

# 172. No static query mutation

Incorrecto:

```php
User::$currentWhere[] = ...
```

---

# 173. Query immutability

Idealmente:

```php
$query2 = $query1->where(...);
```

podrá utilizar query objects inmutables.

La API fluent puede ocultar esta implementación.

---

# 174. Query reuse

Reutilizar un ModelQuery no deberá contaminar otras consultas accidentalmente.

---

# 175. Pagination

```php
User::query()->paginate(25);
```

delegará al futuro:

```text
DATABASE_PAGINATION_SYSTEM
```

---

# 176. Cursor pagination

```php
User::query()->cursorPaginate(100);
```

delegará al sistema correspondiente.

---

# 177. chunk()

```php
User::query()->chunk(500, function ($users) {
});
```

deberá integrarse con:

```text
DATABASE_CHUNK_PROCESSING_SYSTEM
```

---

# 178. lazy()

```php
foreach (User::query()->lazy() as $user) {
}
```

deberá considerar IdentityMap/memory governance.

---

# 179. Streaming caution

Un IdentityMap sin eviction puede derrotar streaming.

Por tanto:

```text
ModelQuery streaming
```

deberá integrarse con políticas de:

```text
clear
detach
windowed identity scope
```

---

# 180. Bulk operations

```php
User::query()
    ->where(...)
    ->update([...]);
```

no equivale a modificar entidades managed una por una.

---

# 181. Bulk update semantics

Debe documentarse:

```text
Bulk Update
→ Query Engine
→ DB
```

puede omitir:

```text
entity hydration
per-entity UnitOfWork events
individual dirty tracking
```

---

# 182. IdentityMap invalidation

Después de bulk update:

```text
managed entities may be stale
```

Por tanto se requerirá política explícita.

---

# 183. BulkMutationPolicy

Posibles:

```text
REJECT_IF_AFFECTED_MANAGED
INVALIDATE_AFFECTED
CLEAR_ENTITY_TYPE
ALLOW_STALE_EXPLICITLY
```

---

# 184. Default bulk safety

Recomendado:

```text
CLEAR_ENTITY_TYPE
```

o invalidación controlada.

---

# 185. increment()/decrement()

```php
$user->increment('loginCount');
```

deberá decidir entre:

```text
entity mutation + UnitOfWork
```

y:

```text
atomic DB expression
```

---

# 186. Atomic field operation

Se recomienda una API explícita:

```php
User::query()
    ->whereKey($id)
    ->increment('loginCount');
```

para operación DB atómica.

---

# 187. Instance increment

Si existe:

```php
$user->increment(...)
```

deberá sincronizar la entidad managed o invalidarla/refresh según política.

---

# 188. update()

```php
$user->update([
    'name' => 'Ana',
]);
```

será:

```text
fill
+
save
```

con mass assignment policy.

---

# 189. forceUpdate()

Podrá existir como escape hatch, pero no como bypass de UnitOfWork.

---

# 190. updateOrCreate()

```php
User::updateOrCreate(
    ['email' => $email],
    ['name' => $name],
);
```

será una operación de conveniencia.

---

# 191. Race condition

`updateOrCreate()` implementado como:

```text
SELECT
then INSERT
```

no garantiza atomicidad.

---

# 192. Upsert capability

Cuando se requiera garantía DB:

```text
native UPSERT
```

deberá utilizar Query Engine + Platform Capability.

---

# 193. firstOrCreate()

Misma consideración:

```text
convenience semantics
≠
atomic uniqueness guarantee
```

---

# 194. Database constraints remain authoritative

La integridad concurrente deberá apoyarse en:

```text
UNIQUE
FK
transactions
locking
platform capabilities
```

no solo en helpers Model.

---

# 195. touch()

```php
$user->touch();
```

podrá actualizar timestamp mediante UnitOfWork.

---

# 196. touchQuietly()

Si existe, deberá definir exactamente qué eventos omite.

---

# 197. quiet operations

Métodos como:

```text
saveQuietly
deleteQuietly
```

podrán suprimir eventos de Model API.

No deberán suprimir:

```text
security audit
transaction integrity
repository invariants
```

cuando estos sean obligatorios.

---

# 198. Event suppression scope

La supresión será:

```text
operation-scoped
```

nunca:

```text
global static flag
```

---

# 199. withoutEvents()

Incorrecto:

```php
Model::$eventsEnabled = false;
```

Correcto:

```text
EventSuppressionContext
```

scoped.

---

# 200. Model context hierarchy

```text
RequestScope
    │
    ▼
DatabaseContext
    │
    ▼
ORMContext
    │
    ▼
ModelContext
```

---

# 201. Context contents

`ModelContext` puede contener referencias scoped a:

```text
EntityManager
ModelQueryBridge
ModelPersistenceBridge
SerializationContext
Clock
```

---

# 202. ModelPersistenceBridge

Contrato conceptual:

```php
interface ModelPersistenceBridge
{
    public function save(Model $model): ModelSaveResult;

    public function delete(Model $model): ModelDeleteResult;

    public function refresh(Model $model): void;
}
```

---

# 203. Bridge responsibility

El bridge traduce:

```text
Model convenience operation
```

a:

```text
ORM operation
```

No implementa persistencia física.

---

# 204. ModelSaveResult

```php
final readonly class ModelSaveResult
{
    public function __construct(
        public bool $persisted,
        public bool $inserted,
        public bool $updated,
        public EntityIdentifierState $identifier,
    ) {}
}
```

La forma concreta podrá refinarse.

---

# 205. Public save return

La API pública puede mantener:

```php
bool
```

por simplicidad.

Internamente deberá conservarse resultado rico.

---

# 206. Exceptions

Jerarquía propuesta:

```text
DatabaseOrmException
└── ModelApiException
    ├── ModelContextUnavailableException
    ├── InvalidModelException
    ├── ModelNotFoundException
    ├── ModelPersistenceException
    ├── DetachedModelException
    ├── ModelRefreshException
    ├── ModelReplicationException
    ├── ModelAttributeException
    ├── UndefinedModelAttributeException
    ├── ModelAttributeConflictException
    ├── ModelCastException
    ├── MassAssignmentException
    ├── GuardedAttributeException
    ├── ModelScopeException
    ├── ModelRelationshipException
    ├── LazyLoadingViolationException
    ├── ModelSerializationException
    ├── ModelEventException
    ├── ModelBulkMutationException
    ├── ModelRuntimeScopeException
    └── ModelApiInvariantException
```

---

# 207. Undefined attributes

Acceder:

```php
$user->doesNotExist;
```

en strict mode deberá lanzar:

```text
UndefinedModelAttributeException
```

en lugar de devolver `null` silenciosamente.

---

# 208. Strictness profiles

Se podrán ofrecer:

```text
STRICT
BALANCED
COMPATIBILITY
```

---

# 209. Recommended development profile

```text
STRICT
```

para detectar:

```text
undefined attributes
lazy loading
discarded mass assignment
invalid casts
ambiguous relationships
```

---

# 210. Production behavior

Production no deberá ocultar errores arquitectónicos.

Solo podrá reducir diagnóstico/telemetry visible.

---

# 211. Model API metadata

La API dependerá fuertemente de metadata compilada:

```text
ModelMetadata
├── EntityMetadata
├── Attributes
├── Casts
├── Relationships
├── Scopes
├── MassAssignment
├── Serialization
├── Timestamps
└── Behaviors
```

---

# 212. ModelMetadata ≠ EntityMetadata

Puede existir:

```text
EntityMetadata
```

como metadata ORM universal.

Y:

```text
ModelApiMetadata
```

para capacidades específicas de `Model`.

---

# 213. Composition

```text
ModelMetadata
=
EntityMetadata
+
ModelApiMetadata
```

conceptualmente.

---

# 214. Compiled Model metadata

Después de bootstrap:

```text
Model metadata
→ validated
→ normalized
→ compiled
→ frozen
```

---

# 215. No runtime reflection hot path

Reflection podrá utilizarse durante bootstrap/development.

Producción deberá favorecer metadata compilada.

---

# 216. Model API cache

Se podrá cachear:

```text
metadata
accessors
mutators
scope descriptors
serialization plans
relationship descriptors
```

---

# 217. Cache safety

Nunca cachear process-wide:

```text
current model
current EntityManager
current tenant
current query
dirty attributes
```

---

# 218. Persistent runtime architecture

```text
Worker Shared
├── Frozen Model Metadata
├── Compiled Accessors
├── Scope Registry
├── Cast Registry
└── Serialization Metadata

Request Scoped
├── ModelContext
├── EntityManager
├── IdentityMap
├── UnitOfWork
├── Query Context
├── Tenant Context
└── Event Suppression Context
```

---

# 219. Model static calls under persistent runtime

```text
User::find(42)
```

deberá resolver el contexto en cada operación.

Nunca reutilizar una referencia stale.

---

# 220. Coroutine safety

Para OpenSwoole:

```text
static mutable current context
```

será especialmente peligroso.

El resolver deberá ser:

```text
request/coroutine aware
```

---

# 221. Async considerations

Si VoltStack incorpora ejecución async:

```text
ModelContext
```

deberá propagarse explícitamente.

---

# 222. Testing

Model API deberá ser testeable sin servidor web.

---

# 223. Context test

```php
$context = TestModelContext::create();

ModelRuntime::within($context, function () {
    $user = User::find(42);
});
```

La API concreta podrá cambiar, pero el scope deberá ser explícito.

---

# 224. Static API tests

Verificar:

```text
User::query()
User::find()
User::create()
```

resuelven el mismo EntityManager scoped.

---

# 225. Identity tests

```php
$a = User::find(42);
$b = User::find(42);

assert($a === $b);
```

---

# 226. Save tests

Cubrir:

```text
NEW
MANAGED clean
MANAGED dirty
DETACHED
REMOVED
generated identifier
assigned identifier
```

---

# 227. Mass assignment tests

Cubrir:

```text
allowed
guarded
unknown
strict rejection
forceFill
nested input
sensitive fields
```

---

# 228. Cast tests

Cubrir:

```text
primitive
enum
JSON
DateTime
value object
custom cast
null handling
invalid conversion
```

---

# 229. Relationship tests

Cubrir:

```text
lazy
eager
explicit
detached
missing related entity
batch loading
N+1 detection
```

---

# 230. Serialization tests

Cubrir:

```text
hidden
visible
computed
loaded relationships
unloaded relationships
cycles
depth
sensitive fields
```

---

# 231. Persistent runtime tests

Simular:

```text
Request A
User::find(42)
reset
Request B
User::find(42)
```

y verificar:

```text
different managed instances
same persistent identity
```

---

# 232. Concurrency tests

Dos concurrent requests nunca deberán compartir:

```text
ModelContext
EntityManager
UnitOfWork
IdentityMap
TenantContext
```

---

# 233. Bulk mutation tests

Verificar invalidación de managed entities después de:

```text
bulk update
bulk delete
increment
decrement
```

---

# 234. Property-based tests

La API fluent podrá probar:

```text
same query operations
+
same metadata
=
same semantic query
```

independientemente de object allocation.

---

# 235. Model API invariants

## DB-MODEL-API-001
Model API será una capa sobre ORM Core.

## DB-MODEL-API-002
Model API no implementará un segundo ORM.

## DB-MODEL-API-003
Model API no implementará un segundo UnitOfWork.

## DB-MODEL-API-004
Model API no implementará un segundo IdentityMap.

## DB-MODEL-API-005
Model API no generará SQL.

## DB-MODEL-API-006
Model API no ejecutará SQL directamente.

## DB-MODEL-API-007
Model API no administrará Driver.

## DB-MODEL-API-008
Model API no almacenará live Connection global.

## DB-MODEL-API-009
Model entities y plain entities compartirán Persistence Engine.

## DB-MODEL-API-010
Model entities y plain entities compartirán EntityManager.

## DB-MODEL-API-011
Model entities y plain entities compartirán UnitOfWork.

## DB-MODEL-API-012
Model entities y plain entities compartirán IdentityMap.

## DB-MODEL-API-013
Static API no implicará static ORM state.

## DB-MODEL-API-014
Cada static operation resolverá contexto scoped.

## DB-MODEL-API-015
ModelContext no será process-global mutable.

## DB-MODEL-API-016
ModelContext no sobrevivirá request reset.

## DB-MODEL-API-017
User::query() producirá Entity-aware query.

## DB-MODEL-API-018
ModelQuery será distinto de low-level QueryBuilder.

## DB-MODEL-API-019
ModelQuery no generará SQL.

## DB-MODEL-API-020
find() respetará IdentityMap.

## DB-MODEL-API-021
Misma EntityKey tendrá máximo una managed instance por scope.

## DB-MODEL-API-022
findOrFail() tendrá failure explícito.

## DB-MODEL-API-023
findMany() evitará N queries cuando sea posible.

## DB-MODEL-API-024
exists() no requerirá hidratar entidad.

## DB-MODEL-API-025
count() no hidratará entidades.

## DB-MODEL-API-026
create() delegará a EntityManager/UnitOfWork.

## DB-MODEL-API-027
create() tendrá flush policy explícita.

## DB-MODEL-API-028
save() no inferirá INSERT/UPDATE únicamente por nullabilidad del ID.

## DB-MODEL-API-029
save() será distinto de upsert.

## DB-MODEL-API-030
Detached save tendrá política explícita.

## DB-MODEL-API-031
delete() delegará a ORM persistence.

## DB-MODEL-API-032
delete() será distinto de direct SQL delete.

## DB-MODEL-API-033
refresh() preservará managed object identity.

## DB-MODEL-API-034
refresh() tendrá semántica explícita respecto a dirty state.

## DB-MODEL-API-035
fresh() no romperá IdentityMap.

## DB-MODEL-API-036
replicate() no copiará persistent identity.

## DB-MODEL-API-037
replicate() será distinto de clone.

## DB-MODEL-API-038
Attribute metadata será autoritativa.

## DB-MODEL-API-039
Magic access no será el único mecanismo interno.

## DB-MODEL-API-040
Undefined attributes serán detectables.

## DB-MODEL-API-041
Cast será distinto de accessor.

## DB-MODEL-API-042
Cast engine convergerá al Type System.

## DB-MODEL-API-043
Mass assignment será distinto de validation.

## DB-MODEL-API-044
Mass assignment será distinto de authorization.

## DB-MODEL-API-045
Mass assignment de campos prohibidos podrá fallar en strict mode.

## DB-MODEL-API-046
forceFill será explícito.

## DB-MODEL-API-047
forceFill no otorgará autorización.

## DB-MODEL-API-048
Dirty API utilizará UnitOfWork.

## DB-MODEL-API-049
Model no mantendrá un segundo original-state tracker.

## DB-MODEL-API-050
Timestamps serán capability opcional.

## DB-MODEL-API-051
Clock será abstraído.

## DB-MODEL-API-052
Scopes modificarán EntityQuery, no SQL strings.

## DB-MODEL-API-053
Global scopes tendrán orden determinista.

## DB-MODEL-API-054
Global scope bypass será explícito.

## DB-MODEL-API-055
Global scopes no sustituirán authorization.

## DB-MODEL-API-056
Relationship declarations no ejecutarán queries.

## DB-MODEL-API-057
Lazy loading tendrá política explícita.

## DB-MODEL-API-058
Detached models no lazy-loadearán mediante global EntityManager.

## DB-MODEL-API-059
Relationship collections podrán integrarse con UnitOfWork.

## DB-MODEL-API-060
Relationship mutation no ejecutará SQL inmediatamente por default.

## DB-MODEL-API-061
Eager loading producirá loading plans.

## DB-MODEL-API-062
Serialization será distinta de persistence mapping.

## DB-MODEL-API-063
Serialization será distinta de authorization.

## DB-MODEL-API-064
Hidden fields podrán seguir persistidos.

## DB-MODEL-API-065
Serialization detectará ciclos.

## DB-MODEL-API-066
Serialization no lazy-loadeará relaciones accidentalmente por default.

## DB-MODEL-API-067
Model events usarán EventSystem/ORM lifecycle.

## DB-MODEL-API-068
Model no tendrá segundo Event Bus.

## DB-MODEL-API-069
Event timing será semánticamente explícito.

## DB-MODEL-API-070
Model boot metadata se congelará tras bootstrap.

## DB-MODEL-API-071
Traits serán convenience capabilities.

## DB-MODEL-API-072
Traits no crearán subsistemas paralelos.

## DB-MODEL-API-073
Trait conflicts no usarán last-wins silencioso.

## DB-MODEL-API-074
Static model configuration será immutable metadata tras compilación.

## DB-MODEL-API-075
Dynamic tenant table mutation mediante static state estará prohibida.

## DB-MODEL-API-076
Logical connection será distinta de live Connection.

## DB-MODEL-API-077
Transactions no serán responsabilidad primaria de Model.

## DB-MODEL-API-078
Static query shortcuts crearán nuevas query contexts.

## DB-MODEL-API-079
No habrá static mutable query state.

## DB-MODEL-API-080
Pagination delegará al Pagination System.

## DB-MODEL-API-081
Chunk processing delegará al Chunk Processing System.

## DB-MODEL-API-082
Streaming considerará IdentityMap memory growth.

## DB-MODEL-API-083
Bulk updates serán distintos de per-entity persistence.

## DB-MODEL-API-084
Bulk operations tendrán política de IdentityMap invalidation.

## DB-MODEL-API-085
Atomic increment utilizará Query Engine cuando corresponda.

## DB-MODEL-API-086
update() respetará mass-assignment policy.

## DB-MODEL-API-087
updateOrCreate convenience no implicará atomicidad automáticamente.

## DB-MODEL-API-088
Database constraints seguirán siendo autoridad de integridad concurrente.

## DB-MODEL-API-089
Event suppression será operation-scoped.

## DB-MODEL-API-090
Event suppression no será static global flag.

## DB-MODEL-API-091
Quiet operations no podrán desactivar invariantes obligatorios.

## DB-MODEL-API-092
ModelPersistenceBridge traducirá API, no implementará DB persistence.

## DB-MODEL-API-093
ModelQueryBridge traducirá API, no generará SQL.

## DB-MODEL-API-094
Model metadata podrá compilarse.

## DB-MODEL-API-095
Reflection no será necesaria en hot path de producción.

## DB-MODEL-API-096
Shared caches solo contendrán metadata immutable.

## DB-MODEL-API-097
Shared caches no contendrán managed Models.

## DB-MODEL-API-098
FrankenPHP requests no compartirán ModelContext.

## DB-MODEL-API-099
RoadRunner requests no compartirán ModelContext.

## DB-MODEL-API-100
OpenSwoole coroutines no compartirán ModelContext incorrectamente.

## DB-MODEL-API-101
Static calls resolverán contexto actual en cada operación.

## DB-MODEL-API-102
Stale EntityManager references no se almacenarán en Model static state.

## DB-MODEL-API-103
TenantContext será scoped.

## DB-MODEL-API-104
EventSuppressionContext será scoped.

## DB-MODEL-API-105
SerializationContext será scoped.

## DB-MODEL-API-106
Model inspection no modificará estado.

## DB-MODEL-API-107
Model API será compatible con PHPStan/Psalm.

## DB-MODEL-API-108
Runtime no dependerá de PHPStan/Psalm.

## DB-MODEL-API-109
Public API importante será IDE-discoverable.

## DB-MODEL-API-110
Model shortcuts no ocultarán transacciones implícitas inesperadas.

## DB-MODEL-API-111
Flush semantics serán documentadas.

## DB-MODEL-API-112
Dirty state pertenecerá al UnitOfWork.

## DB-MODEL-API-113
Original state pertenecerá al snapshot system.

## DB-MODEL-API-114
Model state no se filtrará entre requests.

## DB-MODEL-API-115
Model relationship state no se filtrará entre requests.

## DB-MODEL-API-116
Model scopes no se registrarán dinámicamente durante hot request path por default.

## DB-MODEL-API-117
Model metadata registry será congelable.

## DB-MODEL-API-118
Duplicate Model metadata declarations serán errores detectables.

## DB-MODEL-API-119
Ambiguous attribute/relationship names serán detectables.

## DB-MODEL-API-120
Unknown casts serán errores.

## DB-MODEL-API-121
Invalid relationship definitions fallarán durante metadata validation cuando sea posible.

## DB-MODEL-API-122
Generated ID strategy será reutilizada desde Entity Model.

## DB-MODEL-API-123
Model no tendrá semántica de identidad diferente a plain Entity.

## DB-MODEL-API-124
Model::find() y EntityManager::find() convergerán a la misma IdentityMap.

## DB-MODEL-API-125
Model::save() y EntityManager persistence convergerán al mismo UnitOfWork.

## DB-MODEL-API-126
Model::delete() y EntityManager removal convergerán al mismo Persistence Engine.

## DB-MODEL-API-127
Model query hydration reutilizará Entity Hydration System.

## DB-MODEL-API-128
Model API no hidratará entidades mediante implementación paralela.

## DB-MODEL-API-129
Model relationships reutilizarán Relationship System.

## DB-MODEL-API-130
Model casts reutilizarán Type/Casting System.

## DB-MODEL-API-131
Model factories reutilizarán Factory System.

## DB-MODEL-API-132
Model soft deletes reutilizarán Soft Delete System.

## DB-MODEL-API-133
Model pagination reutilizará Pagination System.

## DB-MODEL-API-134
Model telemetry reutilizará Database Telemetry.

## DB-MODEL-API-135
Model security no dependerá de IDs difíciles de adivinar.

## DB-MODEL-API-136
Mass assignment no será frontera de authorization.

## DB-MODEL-API-137
Hidden serialization fields no serán frontera de authorization.

## DB-MODEL-API-138
Model API permitirá strict developer mode.

## DB-MODEL-API-139
Production no convertirá errores estructurales en comportamiento silencioso.

## DB-MODEL-API-140
Model API podrá evolucionar sin cambiar ORM Core semantics.

## DB-MODEL-API-141
Model convenience methods tendrán equivalentes ORM explícitos cuando sea razonable.

## DB-MODEL-API-142
Convenience API no ocultará incertidumbre de concurrencia.

## DB-MODEL-API-143
firstOrCreate no garantizará atomicidad salvo estrategia DB explícita.

## DB-MODEL-API-144
updateOrCreate no garantizará atomicidad salvo estrategia DB explícita.

## DB-MODEL-API-145
Bulk operations documentarán sus efectos sobre managed entities.

## DB-MODEL-API-146
Refresh no creará identidad duplicada.

## DB-MODEL-API-147
Replication producirá entidad NEW.

## DB-MODEL-API-148
Replication no copiará ORM management state.

## DB-MODEL-API-149
Model API será request-safe.

## DB-MODEL-API-150
Model será una capa de Developer Experience y nunca una segunda arquitectura de persistencia.

---

# 236. Anti-patterns

## 236.1 Segundo ORM dentro de Model

```php
public function save(): void
{
    $sql = 'UPDATE users ...';

    PDO::exec($sql);
}
```

Prohibido.

---

## 236.2 EntityManager estático

```php
abstract class Model
{
    protected static EntityManager $manager;
}
```

Prohibido en persistent runtime.

---

## 236.3 Static query state

```php
User::$where[] = ['active', true];
```

Prohibido.

---

## 236.4 save() por null ID

```php
if ($this->id === null) {
    $this->insert();
} else {
    $this->update();
}
```

No será la semántica ORM.

---

## 236.5 Lazy loading detached

```text
Detached Model
    ↓
global container
    ↓
new EntityManager
    ↓
hidden query
```

Prohibido.

---

## 236.6 Mass assignment como validation

```php
$user->fill($request->all());
```

sin validación no vuelve seguros los datos.

---

## 236.7 Hidden como autorización

```text
hidden = password
⇒ API is secure
```

Incorrecto.

---

## 236.8 Tenant mediante setTable()

```php
User::setTable("tenant_{$tenant}_users");
```

como static mutation.

Prohibido.

---

## 236.9 Bulk update ignorando IdentityMap

```php
User::where(...)->update(...);
```

y continuar usando managed entities stale sin política.

---

## 236.10 fresh() creando identidad duplicada

```text
IdentityMap:
User#42 → A

fresh():
User#42 → B
```

con ambos managed.

Prohibido.

---

## 236.11 Trait last-wins

Configuraciones ORM incompatibles no deberán resolverse silenciosamente por orden de traits.

---

## 236.12 Reflection por cada acceso

Evitar:

```text
$user->name
→ reflection
→ metadata discovery
```

en cada lectura.

Metadata deberá compilarse/cachearse.

---

# 237. Ejemplo completo

```php
final class User extends Model
{
    protected static string $table = 'users';

    protected function fillable(): array
    {
        return [
            'name',
            'email',
        ];
    }

    protected function casts(): array
    {
        return [
            'email' => Email::class,
            'status' => UserStatus::class,
        ];
    }

    public function posts(): HasMany
    {
        return $this->hasMany(Post::class);
    }
}
```

Uso:

```php
$user = User::create([
    'name' => 'Ana',
    'email' => 'ana@example.com',
]);

$user->name = 'Ana María';

$user->save();
```

Internamente:

```text
User::create()
      ↓
ModelContextResolver
      ↓
ModelPersistenceBridge
      ↓
EntityManager
      ↓
EntityMetadata
      ↓
UnitOfWork
      ↓
Persistence Planner
      ↓
Query Engine
      ↓
Database
```

---

# 238. Ejemplo de consulta

```php
$users = User::query()
    ->where('status', UserStatus::ACTIVE)
    ->with('posts')
    ->orderBy('name')
    ->limit(100)
    ->get();
```

Internamente:

```text
ModelQuery<User>
       ↓
EntityQuery
       ↓
Metadata Resolution
       ↓
Semantic Query
       ↓
Query AST
       ↓
Optimizer
       ↓
Planner
       ↓
SQL Compiler
       ↓
Executor
       ↓
Result
       ↓
Hydration
       ↓
IdentityMap
       ↓
EntityCollection<User>
```

---

# 239. Ejemplo IdentityMap

```php
$a = User::find(42);

$b = User::query()
    ->whereKey(42)
    ->first();

assert($a === $b);
```

Aunque hayan sido obtenidos mediante APIs diferentes:

```text
Model::find()
```

y:

```text
ModelQuery
```

convergen a:

```text
EntityKey(User,42)
       ↓
IdentityMap
       ↓
same object
```

---

# 240. Ejemplo POPO vs Model

### Model API

```php
$user = User::find(42);

$user->rename('Ana');

$user->save();
```

### EntityManager API

```php
$user = $entityManager->find(UserEntity::class, 42);

$user->rename('Ana');

$entityManager->flush();
```

Ambos:

```text
same ORM Core
```

---

# 241. Fórmula de Model

```text
Model
=
Entity
+
Developer Experience API
+
Model Metadata
+
Context Resolution
+
Query Bridge
+
Persistence Bridge
```

No:

```text
Model
=
Entity
+
Second ORM
```

---

# 242. Fórmula de consulta

```text
ModelQuery
=
EntityQuery
+
Model Ergonomics
+
Scopes
+
Relationship Loading
+
Model Result API
```

---

# 243. Fórmula de persistencia

```text
Model::save()
=
ResolveCurrentModelContext
+
DelegateToORM
+
UnitOfWorkRegistration
+
FlushAccordingToWritePolicy
+
ReturnDeveloperFriendlyResult
```

---

# 244. Fórmula de seguridad runtime

```text
SafeModelRuntime
=
ImmutableSharedMetadata
+
ScopedModelContext
+
ScopedEntityManager
+
ScopedIdentityMap
+
ScopedUnitOfWork
+
NoStaticMutableRequestState
+
DeterministicReset
```

---

# 245. Fórmula de convergencia

```text
Model API
        ┐
        │
Repository API
        ├──→ EntityManager
        │       ↓
Entity API
        ┘    UnitOfWork
                ↓
        Persistence Engine
                ↓
           Query Engine
```

---

# 246. Regla maestra

> **VoltStack Model proporciona la productividad y ergonomía de una API Active Record sin adoptar las debilidades arquitectónicas de mantener persistencia, queries, identidad y estado dentro de cada modelo.**

En otras palabras:

```text
Laravel-like ergonomics
+
Doctrine-like identity/persistence architecture
+
VoltStack Query Engine
+
Persistent-runtime safety
```

---

# 247. Resultado arquitectónico

VoltStack podrá ofrecer:

```php
$user = User::find(42);

$user->name = 'Ana';

$user->save();
```

sin sacrificar:

```text
IdentityMap
UnitOfWork
typed metadata
query AST
semantic analysis
query optimization
persistence planning
persistent-runtime isolation
testability
```

La experiencia externa será simple:

```text
Model
```

mientras la arquitectura interna permanecerá:

```text
Model
  ↓
ORM
  ↓
Persistence
  ↓
Query Engine
  ↓
Execution
  ↓
Database
```

---

# 248. Relación con los siguientes documentos

La API definida aquí depende de información que todavía debe formalizarse:

```text
115_DATABASE_ENTITY_METADATA_SYSTEM.md
    ↓
qué sabe VoltStack sobre una Entity

116_DATABASE_ENTITY_MAPPING_SYSTEM.md
    ↓
cómo se relaciona Entity ↔ database model

117_DATABASE_ATTRIBUTE_MAPPING_SYSTEM.md
    ↓
cómo PHP Attributes producen mapping

118_DATABASE_ENTITY_MANAGER_SYSTEM.md
    ↓
quién coordina las entidades managed

119_DATABASE_REPOSITORY_SYSTEM.md
    ↓
cómo se accede a colecciones de entidades

120_DATABASE_ENTITY_QUERY_SYSTEM.md
    ↓
cómo ModelQuery se traduce a consultas ORM

121_DATABASE_ENTITY_STATE_SYSTEM.md
    ↓
cómo se representa NEW/MANAGED/REMOVED/DETACHED

122_DATABASE_ENTITY_LIFECYCLE_SYSTEM.md
    ↓
cómo evolucionan las entidades durante el runtime
```

Posteriormente:

```text
123_DATABASE_IDENTITY_MAP_SYSTEM.md
124_DATABASE_UNIT_OF_WORK_ARCHITECTURE.md
```

implementarán las garantías que permiten que:

```php
User::find(42);
$user->save();
```

sean operaciones sencillas externamente pero rigurosas internamente.

---

# 249. Siguiente documento

```text
115_DATABASE_ENTITY_METADATA_SYSTEM.md
```

El siguiente documento deberá definir formalmente:

```text
EntityMetadata
EntityMetadataRegistry
EntityMetadataBuilder
EntityMetadataCompiler
EntityMetadataValidator
EntityMetadataCache
EntityType metadata
Identifier metadata
Field metadata
Relationship metadata
Inheritance metadata
Lifecycle metadata
Instantiation metadata
Change tracking metadata
Persistence metadata
Hydration metadata
Model API metadata bridge
metadata fingerprints
metadata freezing
persistent-runtime metadata sharing
```

manteniendo la regla:

> **Metadata describe cómo el ORM entiende una entidad; nunca contiene la instancia viva de la entidad ni ejecuta operaciones de persistencia.**