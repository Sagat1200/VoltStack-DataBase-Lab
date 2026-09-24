# 194_DATABASE_MODEL_FACTORY_SYSTEM.md

# VoltStack Quantum Database
## Database Model Factory System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 194 — Database Model Factory System  
**Bloque:** 18 — Factories, Seeders & Fixtures  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `193_DATABASE_FACTORY_SYSTEM.md`  
**Siguiente documento:** `195_DATABASE_ENTITY_FACTORY_SYSTEM.md`

---

# 1. Propósito

`Database Model Factory System` define la API de factories orientada al **Model API Laravel-like de VoltStack**.

Su objetivo es proporcionar una experiencia como:

```php
$user = User::factory()->make();

$users = User::factory()
    ->count(10)
    ->create();
```

y permitir composición avanzada:

```php
$users = User::factory()
    ->count(10)
    ->verified()
    ->has(
        Post::factory()
            ->count(5)
            ->published()
    )
    ->create();
```

sin convertir `ModelFactory` en:

- un segundo ORM;
- un Persistence Engine;
- un EntityManager;
- un UnitOfWork;
- un Query Builder;
- un SQL generator.

La regla central será:

> **ModelFactory es una API de conveniencia para construir Models y, opcionalmente, solicitar su persistencia mediante el motor canónico de Factory y ORM de VoltStack.**

Formalmente:

```text
ModelFactory
=
Model-oriented Factory API
+
Shared Factory Engine
+
Model Construction Adapter
+
optional Model Persistence Bridge
```

Nunca:

```text
ModelFactory
=
Second ORM
```

---

# 2. Relación con el Factory System

El documento anterior estableció:

```text
Factory Engine
├── Definitions
├── States
├── Sequences
├── Attributes
├── Relationships
├── Construction
├── Random Data
├── Validation
└── Persistence Bridge
```

`ModelFactory` será una especialización:

```text
                 Factory Engine
                       │
             ┌─────────┴─────────┐
             │                   │
             ▼                   ▼
      ModelFactory          EntityFactory
             │
             ▼
       Model API Adapter
             │
             ▼
          Model
```

Por tanto, `ModelFactory` no duplicará el motor de `193_DATABASE_FACTORY_SYSTEM.md`.

---

# 3. Objetivos

El sistema deberá proporcionar:

1. `Model::factory()`;
2. `make()`;
3. `create()`;
4. `count()`;
5. named states;
6. inline states;
7. sequences;
8. explicit overrides;
9. `has()`;
10. `for()`;
11. many-to-many relationships;
12. nested factory graphs;
13. callbacks;
14. deterministic generation;
15. batch persistence;
16. transaction integration;
17. tenant awareness opcional;
18. shard awareness;
19. strong typing;
20. persistent-runtime safety.

---

# 4. API objetivo

Ejemplo mínimo:

```php
$user = User::factory()->make();
```

Persistencia:

```php
$user = User::factory()->create();
```

Múltiples:

```php
$users = User::factory()
    ->count(10)
    ->create();
```

Estados:

```php
$users = User::factory()
    ->count(10)
    ->verified()
    ->active()
    ->create();
```

Relaciones:

```php
$user = User::factory()
    ->has(
        Post::factory()->count(5)
    )
    ->create();
```

---

# 5. Principio de ergonomía

VoltStack deberá conservar una experiencia cercana a Laravel:

```php
User::factory()
    ->count(20)
    ->verified()
    ->create();
```

pero internamente utilizar una arquitectura más explícita:

```text
Model::factory()
      ↓
ModelFactoryResolver
      ↓
ModelFactory
      ↓
FactoryRequest
      ↓
Shared Factory Engine
      ↓
Model Construction
      ↓
optional PersistenceBridge
      ↓
EntityManager / UnitOfWork
      ↓
Persistence Engine
```

---

# 6. ModelFactory ≠ Active Record Persistence

Aunque la API parezca Active Record:

```php
User::factory()->create();
```

la Factory no deberá ejecutar internamente:

```php
INSERT INTO users ...
```

directamente.

La operación deberá converger al sistema canónico:

```text
ModelFactory
    ↓
FactoryPersistenceBridge
    ↓
ORM
    ↓
UnitOfWork
    ↓
Persistence Planner
    ↓
Query Engine
    ↓
Execution Engine
```

---

# 7. Model::factory()

Los Models podrán exponer:

```php
final class User extends Model
{
    use HasFactory;
}
```

y:

```php
User::factory();
```

---

# 8. HasFactory

Propuesta:

```php
trait HasFactory
{
    public static function factory(
        ?int $count = null
    ): ModelFactory {
        return ModelFactoryResolver::for(
            static::class,
            $count,
        );
    }
}
```

La implementación real no deberá depender de estado estático mutable.

---

# 9. Static syntax ≠ static runtime state

La sintaxis:

```php
User::factory();
```

es válida.

Pero esto no implica:

```php
static FactoryContext $current;
```

ni:

```php
static EntityManager $manager;
```

---

# 10. Resolución contextual

`HasFactory` deberá utilizar un resolver seguro:

```text
User::factory()
      ↓
ModelFactoryContextResolver
      ↓
Scoped Framework Context
      ↓
FactoryRegistry
```

---

# 11. ModelFactoryResolver

Contrato conceptual:

```php
interface ModelFactoryResolver
{
    public function resolve(
        string $modelType
    ): ModelFactory;
}
```

---

# 12. Factory discovery

Para:

```text
App\Models\User
```

podrá resolverse:

```text
Database\Factories\UserFactory
```

según configuración.

---

# 13. Resolución explícita

Un Model podrá declarar:

```php
#[UseFactory(UserFactory::class)]
final class User extends Model
{
}
```

---

# 14. Prioridad de resolución

Orden propuesto:

```text
Explicit Model Metadata
        ↓
Factory Registry
        ↓
Configured Convention
        ↓
Application Resolver
        ↓
Not Found
```

---

# 15. No runtime guessing ilimitado

VoltStack no recorrerá arbitrariamente namespaces por request buscando una Factory.

---

# 16. Factory not found

Ejemplo:

```php
User::factory();
```

sin Factory deberá producir:

```text
ModelFactoryNotFoundException
```

con diagnóstico útil.

---

# 17. Base ModelFactory

Propuesta:

```php
abstract class ModelFactory extends Factory
{
    abstract protected function model(): string;

    abstract protected function definition(
        FactoryContext $context
    ): array;
}
```

---

# 18. Variante inferida

Si el tipo se declara genéricamente:

```php
/**
 * @extends ModelFactory<User>
 */
final class UserFactory extends ModelFactory
{
    protected string $model = User::class;
}
```

---

# 19. Generics para análisis estático

Aunque PHP no tenga generics nativos completos, VoltStack podrá documentar:

```php
/**
 * @template TModel of Model
 */
abstract class ModelFactory
{
}
```

para mejorar soporte de:

- IDE;
- PHPStan;
- Psalm;
- tooling propio de VoltStack.

---

# 20. Retorno tipado

Conceptualmente:

```text
ModelFactory<User>::make()
→ User

ModelFactory<User>::count(10)->make()
→ ModelCollection<User>
```

---

# 21. Singular vs collection

Debe existir una semántica clara.

```php
User::factory()->make();
```

retorna:

```text
User
```

mientras:

```php
User::factory()->count(10)->make();
```

retorna:

```text
ModelCollection<User>
```

---

# 22. count(1)

`count(1)` deberá seguir representando una colección solicitada explícitamente:

```php
User::factory()
    ->count(1)
    ->make();
```

→

```text
ModelCollection<User>
```

Esto evita que el tipo de retorno dependa del valor numérico.

---

# 23. make()

Regla:

```text
make()
=
construct Model
without persistence
```

Ejemplo:

```php
$user = User::factory()->make();
```

Estado esperado:

```text
PHP object exists
Database row does not necessarily exist
```

---

# 24. make() y UnitOfWork

Por defecto:

```text
make()
→ no persist()
→ no flush()
→ no INSERT
```

---

# 25. make() y IdentityMap

El Model generado tampoco deberá entrar automáticamente al `IdentityMap`.

---

# 26. Estado del Model

Un Model generado mediante `make()` podrá representar:

```text
NEW / TRANSIENT
```

dependiendo de la terminología final del Model API.

---

# 27. exists

Si VoltStack expone compatibilidad Laravel-like:

```php
$model->exists
```

entonces:

```php
User::factory()->make()->exists
```

deberá ser:

```text
false
```

---

# 28. create()

```php
$user = User::factory()->create();
```

representará:

```text
Factory Build
      ↓
Model Persistence Bridge
      ↓
Canonical ORM Persistence
```

---

# 29. create() ≠ direct insert

Prohibido:

```text
ModelFactory
→ Connection
→ INSERT
```

Debe utilizarse:

```text
ModelFactory
→ PersistenceBridge
→ ORM
```

---

# 30. create() y UnitOfWork

La implementación podrá realizar conceptualmente:

```php
$model = $factory->make();

$entityManager->persist($model);
$entityManager->flush();
```

según `FactoryFlushPolicy`.

---

# 31. Flush policy

Podrá existir:

```text
IMMEDIATE
DEFERRED
MANUAL
BATCH
```

---

# 32. API create()

El comportamiento por defecto deberá favorecer ergonomía:

```php
User::factory()->create();
```

debe producir un Model sincronizado con DB cuando la operación finalice correctamente.

---

# 33. createQuietly()

Podrá existir:

```php
User::factory()->createQuietly();
```

pero su significado deberá ser preciso.

---

# 34. Quiet ≠ bypass ORM

`createQuietly()` podría deshabilitar determinados lifecycle application events.

Nunca deberá significar:

```text
skip security
skip validation automatically
skip constraints
skip UnitOfWork
direct SQL
```

---

# 35. makeQuietly()

También podría existir:

```php
User::factory()->makeQuietly();
```

si el Model Construction System genera eventos.

---

# 36. save() compatibility

Un Model construido:

```php
$user = User::factory()->make();
```

podrá posteriormente hacer:

```php
$user->save();
```

si el Model API soporta esta sintaxis.

Internamente:

```text
Model::save()
→ ModelContextResolver
→ canonical EntityManager/Persistence Engine
```

---

# 37. Factory definition

Ejemplo:

```php
final class UserFactory extends ModelFactory
{
    protected string $model = User::class;

    protected function definition(
        FactoryContext $context
    ): array {
        return [
            'name' => $context
                ->data()
                ->person()
                ->name(),

            'email' => $context
                ->data()
                ->internet()
                ->email(),

            'status' => UserStatus::ACTIVE,
        ];
    }
}
```

---

# 38. Definition values

Podrán ser:

```text
scalar
enum
value object
closure
sequence
factory reference
relationship reference
typed generator
```

---

# 39. No database access by default

Evitar:

```php
protected function definition(): array
{
    return [
        'role_id' => Role::query()->first()->id,
    ];
}
```

porque convierte una Factory pura en una operación DB implícita.

---

# 40. Preferir relationships

Mejor:

```php
User::factory()
    ->for(
        Role::factory(),
        'role'
    );
```

cuando semánticamente sea una relación.

---

# 41. Existing Model

Si se necesita un Model existente:

```php
User::factory()
    ->for($existingRole, 'role');
```

---

# 42. Explicit database resolver

Cuando realmente sea necesario consultar DB durante construcción deberá utilizarse un mecanismo explícito:

```text
ExistingModelReference
```

o integración equivalente.

---

# 43. States

Ejemplo:

```php
final class UserFactory extends ModelFactory
{
    public function inactive(): static
    {
        return $this->state([
            'status' => UserStatus::INACTIVE,
        ]);
    }
}
```

Uso:

```php
User::factory()
    ->inactive()
    ->create();
```

---

# 44. State methods

Las factories podrán ofrecer métodos de dominio:

```php
->admin()
->verified()
->suspended()
->premium()
```

---

# 45. State ≠ persistence operation

```php
->verified()
```

solo modifica el plan de generación.

No ejecuta:

```text
UPDATE
```

---

# 46. State composition

```php
User::factory()
    ->admin()
    ->verified()
    ->active()
    ->make();
```

---

# 47. State ordering

Se aplicará el orden definido por el Factory Engine.

---

# 48. Explicit override

```php
$user = User::factory()
    ->verified()
    ->make([
        'email' => 'john@example.test',
    ]);
```

El override explícito tendrá precedencia por defecto.

---

# 49. Inline state

```php
User::factory()
    ->state([
        'status' => UserStatus::PENDING,
    ])
    ->make();
```

---

# 50. state() inmutable

Preferentemente:

```php
$base = User::factory();

$active = $base->active();

$inactive = $base->inactive();
```

no mutará `$base`.

---

# 51. Sequence

Ejemplo:

```php
User::factory()
    ->count(3)
    ->sequence(
        ['role' => 'admin'],
        ['role' => 'editor'],
        ['role' => 'customer'],
    )
    ->make();
```

---

# 52. Indexed sequence

```php
User::factory()
    ->count(100)
    ->sequence(
        fn (FactorySequence $sequence) => [
            'email' =>
                "user{$sequence->index}@example.test",
        ]
    )
    ->make();
```

---

# 53. Sequence isolation

Dos llamadas:

```php
User::factory()->count(10)->make();
User::factory()->count(10)->make();
```

tendrán secuencias independientes.

---

# 54. Relationship API

ModelFactory deberá ofrecer una API orientada a relaciones ORM.

Base:

```text
has()
for()
hasAttached()
```

y aliases tipados generados opcionalmente.

---

# 55. has()

Ejemplo:

```php
User::factory()
    ->has(
        Post::factory()->count(5),
        'posts'
    )
    ->create();
```

---

# 56. Resolución

```text
UserFactory
   │
   └── has('posts')
          ↓
Relationship Metadata
          ↓
User.posts
          ↓
OneToMany
          ↓
PostFactory
```

---

# 57. No FK guessing

No asumir simplemente:

```text
posts.user_id
```

La relación debe resolverse desde ORM metadata.

---

# 58. Magic relationship methods

VoltStack podrá ofrecer:

```php
User::factory()
    ->hasPosts(5)
    ->create();
```

como DX opcional.

Pero deberá compilarse a:

```text
has(PostFactory, relationship='posts')
```

---

# 59. No runtime reflection magic excesiva

Los métodos dinámicos podrán generarse mediante tooling/metadata compilada.

No se recomienda depender de parsing dinámico de nombres en cada request.

---

# 60. for()

Ejemplo:

```php
Post::factory()
    ->for(
        User::factory(),
        'author'
    )
    ->create();
```

---

# 61. Existing parent

```php
Post::factory()
    ->for($user, 'author')
    ->create();
```

---

# 62. Parent generated once

Para:

```php
Post::factory()
    ->count(10)
    ->for(
        User::factory(),
        'author'
    )
```

la semántica deberá ser explícita.

Default recomendado:

```text
one generated User
shared by 10 Posts
```

---

# 63. Per-instance parent

Cuando se requiera:

```text
10 Posts
10 different Users
```

deberá solicitarse explícitamente.

Ejemplo conceptual:

```php
Post::factory()
    ->count(10)
    ->forEach(
        User::factory(),
        'author'
    );
```

---

# 64. Shared relationship semantics

Nunca depender de una casualidad de implementación.

---

# 65. One-to-one

```php
User::factory()
    ->has(
        Profile::factory(),
        'profile'
    )
    ->create();
```

---

# 66. One-to-many

```php
User::factory()
    ->has(
        Post::factory()->count(10),
        'posts'
    )
    ->create();
```

---

# 67. Many-to-one

```php
Post::factory()
    ->for(
        User::factory(),
        'author'
    )
    ->create();
```

---

# 68. Many-to-many

```php
User::factory()
    ->hasAttached(
        Role::factory()->count(3),
        relationship: 'roles',
    )
    ->create();
```

---

# 69. Pivot attributes

Para asociaciones simples:

```php
User::factory()
    ->hasAttached(
        Role::factory()->count(2),
        relationship: 'roles',
        attributes: [
            'assigned_at' => FactoryValue::now(),
        ],
    )
    ->create();
```

---

# 70. Rich associations

Si la relación contiene identidad/comportamiento propio:

```text
User
↔
ProjectMembership
↔
Project
```

usar:

```text
ProjectMembershipFactory
```

en lugar de tratarla como pivot anónimo.

---

# 71. Polymorphic relationships

Ejemplo conceptual:

```php
Comment::factory()
    ->for(
        Post::factory(),
        'commentable'
    )
    ->create();
```

La Factory deberá reutilizar `Polymorphic Relationship Metadata`.

---

# 72. Persisted morph aliases

Factory no escribirá FQCN arbitrarios como discriminator.

Usará el canonical morph registry.

---

# 73. Nested relationships

```php
User::factory()
    ->has(
        Post::factory()
            ->count(5)
            ->has(
                Comment::factory()->count(3),
                'comments'
            ),
        'posts'
    )
    ->create();
```

---

# 74. Graph result

```text
User
├── Post
│   ├── Comment
│   ├── Comment
│   └── Comment
├── Post
│   └── ...
└── ...
```

---

# 75. Graph construction first

Conceptualmente:

```text
Factory Graph
     ↓
Construct Models
     ↓
Link Relationships
     ↓
Persistence
```

---

# 76. Persistence ordering

La Factory no decidirá SQL order.

El `Persistence Planner` determinará:

```text
parent inserts
generated IDs
child inserts
membership inserts
updates
```

según dependencies.

---

# 77. Generated IDs

Ejemplo:

```text
User.id generated by DB
Post.user_id depends on User.id
```

La Factory deberá expresar la relación.

No intentar predecir el ID.

---

# 78. Application-generated IDs

Con UUID/ULID:

```text
ID may exist before persistence
```

pero sigue sin significar que el Model exista en DB.

---

# 79. Model state after make()

Ejemplo:

```php
$user = User::factory()->make();
```

puede tener:

```text
id = UUID
exists = false
```

---

# 80. Model state after successful create()

```php
$user = User::factory()->create();
```

podrá resultar:

```text
id = established
exists = true
managed = according to ORM policy
```

solo después de confirmación suficiente.

---

# 81. UNKNOWN outcome

Si ocurre:

```text
COMMIT sent
connection lost
```

no deberá asumirse:

```text
exists = true
```

sin evidencia suficiente.

---

# 82. Model state uncertainty

El Model/PersistenceContext podrá quedar:

```text
TAINTED / UNKNOWN
```

según las reglas de consistencia del ORM.

---

# 83. createMany()

Podrá existir:

```php
User::factory()->createMany([
    ['name' => 'Alice'],
    ['name' => 'Bob'],
]);
```

---

# 84. Semántica

Cada entrada representa overrides sobre la Factory base.

---

# 85. createMany() ≠ bulk insert

Debe conservar:

- Model semantics;
- casts;
- relationships;
- UnitOfWork;
- lifecycle;
- persistence consistency.

---

# 86. Bulk alternatives

Para inserciones masivas donde no se necesitan Models:

```text
DATABASE_BULK_INSERT_SYSTEM
```

será más apropiado.

---

# 87. makeMany()

También podrá existir:

```php
$users = User::factory()->makeMany([
    ['name' => 'Alice'],
    ['name' => 'Bob'],
]);
```

sin persistencia.

---

# 88. afterMaking()

Ejemplo:

```php
protected function configure(): static
{
    return $this->afterMaking(
        function (User $user): void {
            // local object preparation
        }
    );
}
```

---

# 89. afterCreating()

API familiar:

```php
->afterCreating(
    function (User $user): void {
        // persisted-stage callback
    }
);
```

---

# 90. Naming interno

Aunque la API pública utilice:

```text
afterCreating
```

internamente deberá mapearse a una fase precisa:

```text
afterPersistenceSynchronization
```

y no confundirse con `afterCommit`.

---

# 91. afterCommit()

Si se necesita semántica real de commit:

```php
->afterCommit(...)
```

deberá integrarse con `Transaction Event System`.

---

# 92. Callback order

Orden conceptual:

```text
attributes resolved
      ↓
Model constructed
      ↓
relationships linked
      ↓
afterMaking
      ↓
optional persist
      ↓
flush/synchronize
      ↓
afterCreating
      ↓
optional transaction commit
      ↓
afterCommit
```

---

# 93. afterCreating failure

Si DB ya fue modificada y un callback falla:

```text
callback failure
≠
automatic proof of DB rollback
```

La transaction policy decidirá.

---

# 94. Callback transaction participation

Callbacks ejecutados dentro de transaction deberán conocer su fase semántica.

---

# 95. Side effects

Evitar:

```php
->afterCreating(
    fn (User $user) => $mailer->send(...)
);
```

si se pretende atomicidad con DB.

Preferir:

```text
Outbox
afterCommit
Queue
```

según caso.

---

# 96. Deterministic ModelFactory

```php
$users = User::factory()
    ->seed(4201)
    ->count(100)
    ->make();
```

deberá reutilizar `FactoryRandomSource`.

---

# 97. Static Model::factory() safety

El seed nunca se almacenará en:

```text
User::$factorySeed
```

---

# 98. Factory configuration ownership

Cada:

```php
User::factory()
```

produce una configuración independiente.

---

# 99. Reuse

```php
$base = User::factory()
    ->verified();

$admins = $base
    ->admin()
    ->count(5);

$customers = $base
    ->customer()
    ->count(50);
```

deberá ser seguro.

---

# 100. ModelFactoryCollection

Podrá existir un resultado especializado:

```php
/**
 * @template TModel of Model
 */
final class ModelFactoryCollection
{
}
```

---

# 101. Collection responsibilities

Solo utilidades de colección.

No deberá convertirse en EntityManager.

---

# 102. Model constructor

No todos los Models deberán requerir constructor vacío.

Ejemplo:

```php
final class User extends Model
{
    public function __construct(
        UserName $name,
        Email $email,
    ) {}
}
```

ModelFactory deberá soportarlo.

---

# 103. Construction metadata

La Factory podrá declarar:

```php
protected function construct(
    ResolvedFactoryAttributes $attributes
): User {
    return new User(
        $attributes->get('name'),
        $attributes->get('email'),
    );
}
```

---

# 104. Mass assignment

No deberá asumirse que todos los Models se construyen con:

```php
new User($attributes);
```

---

# 105. Factory mass assignment policy

Factories son código interno confiable, pero no deben destruir las reglas de dominio del Model.

---

# 106. fillable/guarded

Si VoltStack implementa compatibilidad Laravel-like:

```text
fillable
guarded
```

la Factory deberá definir si:

```text
A) respeta mass assignment
B) usa trusted construction
```

Default recomendado:

```text
trusted factory construction
```

pero sin omitir invariantes del dominio.

---

# 107. Trusted ≠ unrestricted reflection

No se deberán escribir propiedades privadas arbitrariamente salvo strategy explícita y segura.

---

# 108. Model casts

Factory input deberá atravesar la semántica del Model API cuando corresponda.

Ejemplo:

```php
[
    'status' => UserStatus::ACTIVE,
]
```

debe conservar el enum como representación de aplicación hasta la fase de persistencia.

---

# 109. Factory ≠ DB conversion

No convertir:

```text
UserStatus::ACTIVE
→ "active"
```

a representación DB prematuramente.

Eso pertenece al:

```text
Casting / Value Conversion System
```

---

# 110. Value Objects

Ejemplo:

```php
[
    'email' => EmailFactory::new()->make(),
]
```

deberá ser válido si el Model mapping soporta `Email`.

---

# 111. DateTime

Factories utilizarán el `FactoryClock`.

Ejemplo:

```php
'createdAt' => fn (FactoryContext $context) =>
    $context->clock()->now(),
```

---

# 112. timestamps()

Si Model API maneja:

```text
created_at
updated_at
```

deberá existir una policy clara.

---

# 113. Timestamp policy

Propuesta:

```php
enum ModelFactoryTimestampPolicy
{
    case MODEL_DEFAULT;
    case FACTORY_DEFINED;
    case FIXED;
    case DISABLED;
}
```

---

# 114. MODEL_DEFAULT

Permite que el Model/Persistence System aplique su comportamiento normal.

---

# 115. FACTORY_DEFINED

Factory proporciona timestamps explícitos.

---

# 116. Fixed clock

Tests podrán utilizar:

```php
User::factory()
    ->at('2026-01-01T00:00:00Z')
    ->create();
```

como sugar sobre `FactoryClock`.

---

# 117. Soft deletes

Para Models con soft delete:

```php
User::factory()
    ->trashed()
    ->create();
```

podrá ser un state.

---

# 118. trashed()

No deberá ejecutar un DELETE.

Simplemente construirá el estado persistente correspondiente.

---

# 119. Model lifecycle events

Debe diferenciarse:

```text
Factory Events
Model Lifecycle Events
Persistence Events
Transaction Events
```

---

# 120. Event ordering

Ejemplo conceptual:

```text
FactoryGenerating
        ↓
ModelConstructed
        ↓
FactoryAfterMaking
        ↓
ORM Persist
        ↓
Persistence Events
        ↓
FactoryAfterCreating
        ↓
Transaction Commit
        ↓
Transaction Events
```

---

# 121. Event suppression

`createQuietly()` deberá especificar qué categorías silencia.

---

# 122. No universal silence

No deberá silenciar automáticamente:

- database errors;
- transaction failures;
- security auditing;
- mandatory telemetry;
- constraint enforcement.

---

# 123. Validation integration

ModelFactory podrá ejecutar:

```text
STRUCTURAL
DOMAIN
FULL
NONE
```

según `FactoryValidationPolicy`.

---

# 124. Invalid Models for tests

Ejemplo:

```php
$user = User::factory()
    ->withoutValidation()
    ->make([
        'email' => 'invalid',
    ]);
```

debe ser explícito.

---

# 125. Invalid Model construction

Si el constructor del dominio rechaza el estado, `withoutValidation()` no deberá utilizar reflection para violarlo automáticamente.

---

# 126. Domain invariants remain authoritative

Si:

```php
new Email('invalid')
```

lanza excepción, Factory no deberá saltarla salvo que exista tooling especializado para tests de bajo nivel.

---

# 127. Unique values

```php
User::factory()
    ->count(100)
```

podrá garantizar emails únicos dentro de la ejecución.

---

# 128. Database uniqueness

No podrá garantizar por sí sola que no exista ya:

```text
same email
```

en DB.

---

# 129. Existing database uniqueness

Si se requiere:

```text
generate value not currently in DB
```

deberá utilizarse un explicit DB-aware generation strategy.

---

# 130. DB-aware generation

Debe estar marcado como:

```text
I/O-dependent
non-pure
possibly non-deterministic
```

---

# 131. Concurrency race

Incluso:

```text
SELECT email does not exist
```

seguido de generación no garantiza uniqueness.

La DB constraint sigue siendo autoridad.

---

# 132. recycle()

Una API útil podrá ser:

```php
$roles = Role::factory()
    ->count(3)
    ->create();

User::factory()
    ->count(100)
    ->recycle($roles)
    ->create();
```

---

# 133. recycle semantics

Permite reutilizar Models existentes para relaciones.

---

# 134. Recycle pool

```text
Existing Models
      ↓
Recycle Pool
      ↓
Relationship Resolver
```

---

# 135. Recycle selection

Podrá ser:

```text
RANDOM
ROUND_ROBIN
SEQUENTIAL
CUSTOM
```

---

# 136. Deterministic recycling

Con seed determinista, la selección deberá ser reproducible cuando la colección de entrada sea estable.

---

# 137. recycle() ≠ IdentityMap

Recycle Pool es una herramienta de generación.

IdentityMap es una garantía ORM de identidad canónica.

---

# 138. suspended Models

Factories no deberán mantener referencias globales a Models reciclados entre requests.

---

# 139. Lazy attributes

Podrá existir:

```php
User::factory()
    ->lazy([
        'token' => fn (...) => ...
    ]);
```

aunque las definitions ya sean lazy por defecto.

---

# 140. Derived attributes

Ejemplo:

```php
User::factory()
    ->state(fn (FactoryAttributes $attributes) => [
        'slug' => Slug::from($attributes->get('name')),
    ]);
```

---

# 141. Attribute dependency graph

El sistema deberá detectar:

```text
name
 ↓
slug
```

y ordenar resolución correctamente.

---

# 142. Cycles

```text
username depends email
email depends username
```

deberá fallar.

---

# 143. ModelFactory configure()

Ejemplo:

```php
final class UserFactory extends ModelFactory
{
    protected function configure(): static
    {
        return $this
            ->afterMaking(...)
            ->afterCreating(...);
    }
}
```

---

# 144. configure() execution

`configure()` deberá formar parte de la compilación/configuración de Factory, no ejecutarse repetidamente por cada atributo si no es necesario.

---

# 145. Factory inheritance

Podrá soportarse:

```text
BaseUserFactory
   ↓
CustomerFactory
```

pero composición mediante states deberá preferirse cuando sea suficiente.

---

# 146. Deep inheritance

Debe evitarse una jerarquía como:

```text
UserFactory
→ ActiveUserFactory
→ VerifiedUserFactory
→ AdminUserFactory
```

cuando states resuelven mejor el problema.

---

# 147. Composition over inheritance

Preferido:

```php
User::factory()
    ->active()
    ->verified()
    ->admin();
```

---

# 148. Custom factory methods

Factories podrán implementar métodos:

```php
public function withSubscription(
    SubscriptionPlan $plan
): static
{
    // configure graph
}
```

---

# 149. Domain-specific factory DSL

Esto permite:

```php
User::factory()
    ->premium()
    ->withSubscription(SubscriptionPlan::PRO)
    ->create();
```

---

# 150. No hidden queries

Los DSL methods no deberán hacer DB I/O durante configuración salvo que su nombre/contrato lo indique explícitamente.

---

# 151. Persistence modes

ModelFactory podrá exponer:

```text
make
create
stream
createInChunks
```

---

# 152. makeOne()

Podrá existir como alias explícito:

```php
User::factory()->makeOne();
```

---

# 153. createOne()

Igualmente:

```php
User::factory()->createOne();
```

---

# 154. Batch persistence

```php
User::factory()
    ->count(10_000)
    ->create();
```

deberá poder utilizar:

```text
Factory Generation Chunks
+
Batch Persistence
```

---

# 155. Memory management

No deberá acumular obligatoriamente:

```text
10,000 Models
+
10,000 Snapshots
+
10,000 IdentityMap Entries
```

si el caller no requiere conservarlos.

---

# 156. createInChunks()

Ejemplo conceptual:

```php
User::factory()
    ->count(1_000_000)
    ->createInChunks(1000);
```

---

# 157. Chunk result policy

Podrá retornar:

```text
summary
iterator
IDs
objects
custom collector
```

según API.

---

# 158. Large Model creation warning

Crear millones de Models tiene un costo semántico.

Para importaciones puramente tabulares:

```text
Bulk Insert System
```

puede ser mejor.

---

# 159. Transaction integration

Ejemplo:

```php
DB::transaction(function () {
    User::factory()
        ->count(10)
        ->create();
});
```

deberá participar en la transacción actual.

---

# 160. Transaction ownership

La Factory no deberá ejecutar:

```text
commit
```

sobre una transaction que no posee.

---

# 161. Existing transaction

```text
USE_EXISTING
```

será la semántica natural.

---

# 162. No active transaction

La Factory podrá utilizar su policy configurada.

---

# 163. Transaction wrapper

Podrá existir:

```php
User::factory()
    ->count(100)
    ->transactional()
    ->create();
```

como convenience API.

---

# 164. transactional() ≠ distributed atomicity

Si el graph cruza shards:

```text
transactional()
```

no deberá prometer atomicidad global.

---

# 165. Sharding

Un Model podrá poseer:

```text
shard key
```

generado por la Factory.

---

# 166. Factory does not choose endpoint

Pipeline:

```text
ModelFactory
    ↓
Model
    ↓
Persistence Planner
    ↓
Partition Routing
    ↓
Shard
    ↓
Read/Write Routing
    ↓
Endpoint
```

---

# 167. Cross-shard graph

Si un Factory Graph produce:

```text
User → shard A
Order → shard B
```

el Persistence System deberá detectar el dominio distribuido.

---

# 168. No fabricated transaction

La Factory no presentará esto como una transacción atómica normal.

---

# 169. Shard key override

```php
User::factory()
    ->make([
        'region' => 'mx',
    ]);
```

puede afectar routing posterior.

---

# 170. Hardcoded ShardId

No se recomienda:

```php
->shard('shard-7')
```

como mecanismo normal de dominio.

Preferir:

```text
shard key
```

porque `ShardMap` puede cambiar.

---

# 171. Testing infrastructure

Tooling de infraestructura sí podrá forzar un shard explícito bajo APIs controladas.

---

# 172. Multitenancy

Con paquete Multitenancy:

```php
User::factory()
    ->forTenant($tenant)
    ->create();
```

podrá contribuir contexto.

---

# 173. Tenant context

Deberá propagarse a:

```text
Factory Generation
Persistence Context
Partition Routing
Cache Identity
Telemetry
```

cuando corresponda.

---

# 174. Tenant field

Si tenant identity forma parte del Model:

```text
tenant_id
```

podrá ser resuelto por integración de metadata.

---

# 175. No automatic cross-tenant relation

```php
User::factory()
    ->forTenant($tenantA)
    ->has(
        Order::factory()->forTenant($tenantB)
    );
```

deberá rechazarse cuando viole las reglas de dominio.

---

# 176. ModelFactory no depende de Multitenancy

La integración será opcional.

---

# 177. Persistent runtimes

Especialmente bajo FrankenPHP:

```text
Worker
├── Request A
│   └── UserFactory execution A
│
├── Request B
│   └── UserFactory execution B
│
└── Request C
    └── UserFactory execution C
```

---

# 178. Shared state permitido

Podrá compartirse:

```text
immutable FactoryBlueprint
immutable Model Metadata
immutable Type Metadata
compiled constructor accessors
```

---

# 179. Shared state prohibido

No compartir:

```text
FactoryContext
RandomSource state
Sequence counter
Unique pool
Generated Models
Relationship graph runtime
Tenant context
Transaction context
```

---

# 180. FrankenPHP

La implementación deberá funcionar correctamente con workers persistentes.

---

# 181. RoadRunner

El mismo modelo scoped deberá permitir integración futura.

---

# 182. OpenSwoole

Se deberá asumir posible concurrencia:

```text
coroutines
```

por lo que los factories no podrán depender de mutable globals.

---

# 183. Concurrent use

Esto deberá ser seguro:

```text
Coroutine A
→ UserFactory seed 100

Coroutine B
→ UserFactory seed 200
```

sin contaminación.

---

# 184. Security

ModelFactory será principalmente tooling de desarrollo/testing.

Pero el framework no deberá asumir que siempre corre fuera de producción.

---

# 185. Production exposure

Rutas como:

```text
/debug/create-users
```

no deberán existir automáticamente.

---

# 186. Authorization

La existencia de una Factory no autoriza a un usuario a crear Models.

---

# 187. Factory ≠ authorization

```text
Factory Capability
≠
Application Permission
```

---

# 188. Sensitive fields

Factories deberán tratar cuidadosamente:

```text
password
token
secret
API key
payment data
PII
```

---

# 189. Password example

```php
'password' => TestPassword::known(),
```

podrá ser procesado por el Hashing System.

---

# 190. No plaintext telemetry

Nunca registrar el password generado.

---

# 191. Synthetic emails

Preferir dominios reservados para ejemplos/tests:

```text
example.test
example.invalid
```

según provider.

---

# 192. Cache interaction

ModelFactory no deberá insertar directamente objetos en:

```text
Entity Cache
Result Cache
Query Cache
```

---

# 193. Cache population

Si Persistence Engine provoca invalidación/populación de caches, se seguirán las reglas de los documentos 186–192.

---

# 194. Uncommitted Model cache

Nunca publicar un Model no committed como estado compartido confirmado.

---

# 195. Rollback

Si `create()` participa en una transaction que luego hace rollback:

```text
DB rollback
≠
automatic PHP Model graph rewind
```

---

# 196. Model `exists`

Por ello, cualquier compatibilidad `exists` deberá integrarse cuidadosamente con `Persistence Consistency System`.

---

# 197. Unknown commit

No deberá dejar silenciosamente un Model marcado como definitivamente persisted si el resultado es UNKNOWN.

---

# 198. ModelFactory diagnostics

API conceptual:

```php
DB::factories()
    ->explain(
        User::factory()
            ->verified()
            ->has(
                Post::factory()->count(3)
            )
    );
```

---

# 199. Explain example

```text
MODEL FACTORY PLAN

Target
------
App\Models\User

Factory
-------
Database\Factories\UserFactory

Mode
----
MODEL

Count
-----
1

States
------
verified

Relationships
-------------
posts
  type: ONE_TO_MANY
  factory: PostFactory
  count: 3

Construction
------------
MODEL_CONSTRUCTOR

Persistence
-----------
NONE

Randomness
----------
DETERMINISTIC

Runtime Scope
-------------
factory-execution

Estimated Objects
-----------------
4
```

---

# 200. Persistence explain

Para:

```php
User::factory()
    ->count(100)
    ->create();
```

podrá mostrar:

```text
Requested Models:       100
Persistence:            ENABLED
Flush Policy:           BATCH
Transaction:            USE_EXISTING
Estimated Graph Nodes:  100
Direct SQL from Factory: NO
```

---

# 201. Telemetry

Eventos específicos:

```text
ModelFactoryResolved
ModelFactoryGenerationStarted
ModelFactoryModelConstructed
ModelFactoryRelationshipLinked
ModelFactoryPersistenceRequested
ModelFactoryPersistenceCompleted
ModelFactoryExecutionCompleted
ModelFactoryExecutionFailed
```

---

# 202. Metrics

```text
db.factory.model.executions
db.factory.model.constructed
db.factory.model.persisted
db.factory.model.relationships
db.factory.model.failures
db.factory.model.duration
```

---

# 203. Cardinality

No utilizar:

```text
model ID
email
tenant ID raw
generated values
```

como metric labels.

---

# 204. Tracing

Span:

```text
database.factory.model
```

Atributos seguros:

```text
model.type
factory.type
count
persistence.enabled
relationship.count
state.count
```

---

# 205. Error hierarchy

```text
DatabaseFactoryException
└── ModelFactoryException
    ├── ModelFactoryNotFoundException
    ├── InvalidModelFactoryException
    ├── ModelFactoryConstructionException
    ├── ModelFactoryStateException
    ├── ModelFactoryRelationshipException
    ├── ModelFactoryPersistenceException
    ├── ModelFactoryCallbackException
    └── ModelFactoryRuntimeException
```

---

# 206. Invalid target

Si:

```text
UserFactory
```

declara:

```text
Order::class
```

cuando está registrada para `User`, deberá fallar durante compilación/bootstrap cuando sea detectable.

---

# 207. Relationship mismatch

Ejemplo:

```php
User::factory()
    ->has(
        Invoice::factory(),
        'posts'
    );
```

deberá producir un error claro.

---

# 208. Error message

Ejemplo:

```text
Invalid ModelFactory relationship.

Factory:
    UserFactory

Model:
    App\Models\User

Relationship:
    posts

Expected target:
    App\Models\Post

Factory target:
    App\Models\Invoice
```

---

# 209. Directory structure

Propuesta:

```text
src/Quantum/Database/Factory/Model/
│
├── ModelFactory.php
├── ModelFactoryResolver.php
├── ModelFactoryRegistry.php
├── ModelFactoryBlueprint.php
├── ModelFactoryRequest.php
├── ModelFactoryContext.php
├── ModelFactoryCollection.php
│
├── Concerns/
│   ├── HasStates.php
│   ├── HasSequences.php
│   ├── HasModelRelationships.php
│   ├── HasFactoryCallbacks.php
│   ├── HasFactoryOverrides.php
│   └── HasFactoryPersistence.php
│
├── Attribute/
│   ├── ModelFactoryAttributeResolver.php
│   └── ModelFactoryAttributePlan.php
│
├── Relationship/
│   ├── ModelFactoryRelationship.php
│   ├── ModelFactoryRelationshipResolver.php
│   ├── ModelFactoryHas.php
│   ├── ModelFactoryFor.php
│   ├── ModelFactoryAttached.php
│   ├── ModelFactoryRecyclePool.php
│   └── ModelFactoryGraphAdapter.php
│
├── Construction/
│   ├── ModelFactoryConstructor.php
│   ├── ModelFactoryConstructionPlan.php
│   └── ModelFactoryConstructionStrategy.php
│
├── Persistence/
│   ├── ModelFactoryPersistenceBridge.php
│   ├── ModelFactoryPersistencePolicy.php
│   ├── ModelFactoryFlushPolicy.php
│   └── ModelFactoryPersistenceResult.php
│
├── Callback/
│   ├── ModelFactoryCallback.php
│   ├── ModelFactoryCallbackRegistry.php
│   └── ModelFactoryCallbackPhase.php
│
├── Timestamp/
│   └── ModelFactoryTimestampPolicy.php
│
├── Runtime/
│   └── ModelFactoryRuntimeState.php
│
├── Diagnostics/
│   ├── ModelFactoryInspector.php
│   └── ModelFactoryExplainer.php
│
├── Telemetry/
│   └── ModelFactoryTelemetry.php
│
└── Exception/
    ├── ModelFactoryException.php
    ├── ModelFactoryNotFoundException.php
    ├── InvalidModelFactoryException.php
    ├── ModelFactoryConstructionException.php
    ├── ModelFactoryRelationshipException.php
    └── ModelFactoryPersistenceException.php
```

Integración Model API:

```text
src/Quantum/Database/ORM/Model/
└── Concerns/
    └── HasFactory.php
```

---

# 210. Ejemplo completo

```php
final class UserFactory extends ModelFactory
{
    protected string $model = User::class;

    protected function definition(
        FactoryContext $context
    ): array {
        return [
            'name' => $context
                ->data()
                ->person()
                ->name(),

            'email' => $context
                ->data()
                ->internet()
                ->uniqueEmail(),

            'status' => UserStatus::ACTIVE,

            'emailVerifiedAt' => null,
        ];
    }

    public function verified(): static
    {
        return $this->state(
            fn (FactoryContext $context) => [
                'emailVerifiedAt' =>
                    $context->clock()->now(),
            ]
        );
    }

    public function suspended(): static
    {
        return $this->state([
            'status' => UserStatus::SUSPENDED,
        ]);
    }
}
```

Uso:

```php
$user = User::factory()
    ->verified()
    ->make();
```

---

# 211. Ejemplo con relaciones

```php
$user = User::factory()
    ->verified()
    ->has(
        Profile::factory(),
        'profile'
    )
    ->has(
        Post::factory()
            ->count(5)
            ->published()
            ->has(
                Comment::factory()->count(3),
                'comments'
            ),
        'posts'
    )
    ->create();
```

Grafo:

```text
User
│
├── Profile
│
└── Posts × 5
    │
    └── Comments × 3
```

Total:

```text
1 User
1 Profile
5 Posts
15 Comments
----------------
22 Models
```

---

# 212. Pipeline completo

```text
User::factory()
      │
      ▼
HasFactory
      │
      ▼
ModelFactoryResolver
      │
      ▼
UserFactory
      │
      ▼
Factory Configuration
      │
      ├── count
      ├── states
      ├── overrides
      ├── sequences
      ├── relationships
      └── callbacks
      │
      ▼
Shared Factory Compiler
      │
      ▼
FactoryBlueprint
      │
      ▼
FactoryGraphPlanner
      │
      ▼
Attribute Resolution
      │
      ▼
Model Construction
      │
      ▼
Relationship Linking
      │
      ▼
afterMaking
      │
      ├────────────────────────────┐
      │                            │
    make()                       create()
      │                            │
      ▼                            ▼
   Return Model        ModelPersistenceBridge
                                   │
                                   ▼
                             EntityManager
                                   │
                                   ▼
                              UnitOfWork
                                   │
                                   ▼
                          Persistence Planner
                                   │
                                   ▼
                              Query Engine
                                   │
                                   ▼
                          Partition Routing
                                   │
                                   ▼
                           Execution Engine
                                   │
                                   ▼
                               Database
```

---

# 213. Anti-pattern: Factory SQL

Incorrecto:

```php
public function create(): User
{
    DB::insert(
        'INSERT INTO users ...'
    );
}
```

Razón:

```text
bypasses ORM
bypasses UoW
bypasses Persistence Planner
bypasses relationships
bypasses type conversion
bypasses transaction semantics
```

---

# 214. Anti-pattern: global Faker

Incorrecto:

```php
static Faker $faker;
```

bajo persistent runtime si mantiene mutable generation state.

---

# 215. Anti-pattern: global sequence

Incorrecto:

```php
private static int $sequence = 0;
```

---

# 216. Anti-pattern: hidden query

Incorrecto:

```php
public function definition(): array
{
    return [
        'role_id' => Role::first()->id,
    ];
}
```

sin declarar que la Factory requiere DB I/O.

---

# 217. Anti-pattern: implicit commit

Incorrecto:

```text
Factory create()
→ flush
→ assume committed
```

---

# 218. Anti-pattern: bypass domain constructor

Incorrecto:

```text
Reflection
→ force invalid private state
```

como comportamiento normal.

---

# 219. Anti-pattern: factory as seeder

Incorrecto:

```php
UserFactory::seedProductionDatabase();
```

Factory describe cómo generar Users.

Seeder decide qué dataset generar.

---

# 220. Anti-pattern: factory as fixture

Una Factory:

```text
UserFactory
```

no deberá representar por sí sola un escenario como:

```text
CustomerWithThreeOverdueInvoices
```

Ese escenario puede componerse mediante Fixture.

---

# 221. Testing matrix

| Área | Prueba |
|---|---|
| Resolution | `User::factory()` resuelve `UserFactory` |
| Missing Factory | error explícito |
| make | cero escrituras DB |
| create | utiliza Persistence Engine |
| count | cardinalidad correcta |
| states | composición determinista |
| sequence | aislamiento |
| overrides | precedencia correcta |
| has | relación correcta |
| for | parent correcto |
| many-to-many | memberships correctas |
| polymorphic | alias correcto |
| callbacks | orden correcto |
| transactions | ownership correcto |
| rollback | no fabricated state |
| UNKNOWN | conserva incertidumbre |
| tenant | aislamiento |
| shard | routing correcto |
| worker | sin leakage |
| coroutine | sin state collision |
| deterministic | mismo seed |
| memory | bounded generation |
| telemetry | no sensitive values |

---

# 222. Test de make()

```php
$user = User::factory()->make();

assert($user instanceof User);
assert($user->exists === false);
```

y:

```text
Database writes = 0
```

---

# 223. Test create()

```php
$user = User::factory()->create();
```

deberá verificar:

```text
Factory
→ ORM
→ Persistence Engine
```

y no acceso directo a Driver.

---

# 224. Test count

```php
$users = User::factory()
    ->count(10)
    ->make();
```

Debe cumplirse:

```text
count($users) = 10
```

---

# 225. Test state isolation

```php
$base = User::factory();

$admin = $base->admin()->make();
$customer = $base->customer()->make();
```

`admin()` no deberá contaminar `customer()`.

---

# 226. Test deterministic

```php
$a = User::factory()
    ->seed(123)
    ->count(10)
    ->make();

$b = User::factory()
    ->seed(123)
    ->count(10)
    ->make();
```

Deberá cumplirse, bajo providers deterministas:

```text
CanonicalFactoryValues(A)
=
CanonicalFactoryValues(B)
```

---

# 227. Test persistent worker

```text
Request A
seed = 10

Request B
seed = 20
```

El resultado B no deberá depender del estado de A.

---

# 228. Test transaction

```php
DB::transaction(function () {
    User::factory()->create();

    throw new RuntimeException();
});
```

DB deberá seguir transaction semantics normales.

Factory no deberá interceptar ni fabricar commit.

---

# 229. Test graph cardinality

```php
User::factory()
    ->count(10)
    ->has(
        Post::factory()
            ->count(5)
            ->has(
                Comment::factory()->count(3)
            )
    );
```

esperado:

```text
Users     10
Posts     50
Comments 150
Total    210
```

---

# 230. Test budget

Si:

```text
maxObjects = 100
```

el grafo anterior deberá rechazarse antes o durante planificación de forma segura.

---

# 231. Architectural invariants

## DB-MFAC-001
ModelFactory será una especialización del Factory Engine.

## DB-MFAC-002
ModelFactory no será un segundo ORM.

## DB-MFAC-003
ModelFactory no será Persistence Engine.

## DB-MFAC-004
ModelFactory no generará SQL.

## DB-MFAC-005
ModelFactory no ejecutará Driver directamente.

## DB-MFAC-006
ModelFactory no administrará Connection directamente.

## DB-MFAC-007
`Model::factory()` será una API de conveniencia.

## DB-MFAC-008
Static factory syntax no implicará static mutable state.

## DB-MFAC-009
Cada Factory execution tendrá contexto scoped.

## DB-MFAC-010
`make()` no persistirá.

## DB-MFAC-011
`make()` no ejecutará INSERT.

## DB-MFAC-012
`make()` no ejecutará flush.

## DB-MFAC-013
`make()` no registrará automáticamente en UoW.

## DB-MFAC-014
`make()` no registrará automáticamente en IdentityMap.

## DB-MFAC-015
`create()` utilizará PersistenceBridge.

## DB-MFAC-016
PersistenceBridge utilizará el ORM canónico.

## DB-MFAC-017
`create()` no realizará direct SQL.

## DB-MFAC-018
`flush()` será distinto de `commit()`.

## DB-MFAC-019
`afterCreating()` será distinto de `afterCommit()`.

## DB-MFAC-020
UNKNOWN persistence outcome permanecerá UNKNOWN.

## DB-MFAC-021
ModelFactory no fabricará `exists=true` ante resultado incierto.

## DB-MFAC-022
Factory configuration será preferentemente immutable.

## DB-MFAC-023
Named states serán composables.

## DB-MFAC-024
State ordering será determinista.

## DB-MFAC-025
Explicit overrides tendrán precedencia definida.

## DB-MFAC-026
Sequences serán execution-scoped.

## DB-MFAC-027
No existirán global mutable sequence counters.

## DB-MFAC-028
Randomness será execution-scoped.

## DB-MFAC-029
Deterministic generation utilizará explicit seed.

## DB-MFAC-030
Factory definitions no harán DB I/O por defecto.

## DB-MFAC-031
DB-aware generation será explícita.

## DB-MFAC-032
Relationships se resolverán mediante ORM metadata.

## DB-MFAC-033
ModelFactory no inferirá relationships únicamente por FK naming.

## DB-MFAC-034
`has()` representará child graph intent.

## DB-MFAC-035
`for()` representará parent/reference intent.

## DB-MFAC-036
Many-to-many reutilizará Relationship System.

## DB-MFAC-037
Polymorphic factories utilizarán canonical morph aliases.

## DB-MFAC-038
Rich associations utilizarán association entity cuando corresponda.

## DB-MFAC-039
FactoryGraphPlan será distinto de PersistencePlan.

## DB-MFAC-040
Generated DB IDs no serán predichos.

## DB-MFAC-041
Application-generated IDs no implicarán persistence.

## DB-MFAC-042
ModelFactory no determinará persistence ordering SQL.

## DB-MFAC-043
Persistence Planner determinará dependency ordering.

## DB-MFAC-044
`createMany()` será distinto de Bulk Insert.

## DB-MFAC-045
Large factory generation tendrá budgets.

## DB-MFAC-046
Chunking no ejecutará hidden EntityManager clear sin policy.

## DB-MFAC-047
Model constructor semantics serán respetadas.

## DB-MFAC-048
Factory trusted construction no significará arbitrary reflection.

## DB-MFAC-049
Domain invariants seguirán siendo autoridad.

## DB-MFAC-050
Model casts no serán DB-converted prematuramente.

## DB-MFAC-051
Value Objects conservarán application representation.

## DB-MFAC-052
FactoryClock controlará temporal generation cuando se requiera.

## DB-MFAC-053
Soft-delete state no será DELETE operation.

## DB-MFAC-054
Factory events serán distintos de Model events.

## DB-MFAC-055
Model events serán distintos de Persistence events.

## DB-MFAC-056
Persistence events serán distintos de Transaction events.

## DB-MFAC-057
Quiet creation no omitirá DB correctness.

## DB-MFAC-058
Invalid testing será explícito.

## DB-MFAC-059
Factory no romperá constructores para generar estado inválido por defecto.

## DB-MFAC-060
Factory uniqueness será distinta de DB uniqueness.

## DB-MFAC-061
DB constraints seguirán siendo autoridad final.

## DB-MFAC-062
Recycle pool será distinto de IdentityMap.

## DB-MFAC-063
Recycle pool será execution-scoped.

## DB-MFAC-064
Existing Models no se almacenarán globalmente por Factory.

## DB-MFAC-065
Attribute dependency cycles serán detectados.

## DB-MFAC-066
Factory inheritance no sustituirá composición por states.

## DB-MFAC-067
Domain-specific Factory DSL no deberá ocultar DB queries.

## DB-MFAC-068
Transactions serán coordinadas por Transaction System.

## DB-MFAC-069
Factory no hará commit de transaction ajena.

## DB-MFAC-070
`transactional()` no prometerá distributed atomicity.

## DB-MFAC-071
Shard routing pertenecerá a Partition Routing.

## DB-MFAC-072
Factory no elegirá endpoint.

## DB-MFAC-073
Factory no elegirá replica.

## DB-MFAC-074
Factory no elegirá writer físico.

## DB-MFAC-075
Shard key podrá formar parte del Model generado.

## DB-MFAC-076
Hardcoded ShardId no será mecanismo de dominio predeterminado.

## DB-MFAC-077
Cross-shard persistence no fabricará atomicidad.

## DB-MFAC-078
Multitenancy será integración opcional.

## DB-MFAC-079
No habrá static current tenant.

## DB-MFAC-080
Cross-tenant relationships serán validadas.

## DB-MFAC-081
Compiled FactoryBlueprint podrá compartirse entre requests si es immutable.

## DB-MFAC-082
Generated Models no se compartirán entre requests.

## DB-MFAC-083
FactoryContext no se compartirá entre requests.

## DB-MFAC-084
Random state no se compartirá entre requests.

## DB-MFAC-085
Sequence state no se compartirá entre requests.

## DB-MFAC-086
Unique pools no se compartirán accidentalmente entre requests.

## DB-MFAC-087
Tenant context no se compartirá entre requests.

## DB-MFAC-088
Transaction context no se compartirá entre requests.

## DB-MFAC-089
ModelFactory será segura bajo FrankenPHP.

## DB-MFAC-090
Arquitectura permitirá RoadRunner.

## DB-MFAC-091
Arquitectura permitirá OpenSwoole.

## DB-MFAC-092
Concurrent Factory executions estarán aisladas.

## DB-MFAC-093
Factory availability no implicará authorization.

## DB-MFAC-094
Factories no expondrán production endpoints automáticamente.

## DB-MFAC-095
Sensitive generated values no aparecerán en telemetry.

## DB-MFAC-096
Passwords no aparecerán en logs.

## DB-MFAC-097
ModelFactory no escribirá directamente Entity Cache.

## DB-MFAC-098
ModelFactory no escribirá directamente Result Cache.

## DB-MFAC-099
ModelFactory no escribirá directamente Query Cache.

## DB-MFAC-100
Cache effects seguirán Persistence/Transaction rules.

## DB-MFAC-101
Rollback no implicará automatic PHP object graph rewind.

## DB-MFAC-102
Factory diagnostics serán explainable.

## DB-MFAC-103
Telemetry tendrá bounded cardinality.

## DB-MFAC-104
Errors incluirán relationship paths útiles.

## DB-MFAC-105
ModelFactory target deberá ser validado.

## DB-MFAC-106
Factory resolution será determinista.

## DB-MFAC-107
Runtime no hará filesystem scanning ilimitado.

## DB-MFAC-108
`count(1)` explícito conservará collection semantics.

## DB-MFAC-109
Singular factory sin `count()` devolverá singular Model.

## DB-MFAC-110
Collection factories devolverán typed Model collection.

## DB-MFAC-111
`createQuietly()` no omitirá security constraints.

## DB-MFAC-112
`createQuietly()` no omitirá transaction failures.

## DB-MFAC-113
Callbacks tendrán phase semantics explícitas.

## DB-MFAC-114
External side effects no obtendrán atomicidad por estar en callback.

## DB-MFAC-115
After-commit effects usarán Transaction Event integration.

## DB-MFAC-116
Factory graph cardinality deberá poder estimarse.

## DB-MFAC-117
Recursive relationships tendrán límites.

## DB-MFAC-118
Factory graph explosion deberá poder rechazarse.

## DB-MFAC-119
Factory persistence conservará ORM lifecycle semantics.

## DB-MFAC-120
Factory persistence conservará relationship semantics.

## DB-MFAC-121
Factory persistence conservará casting semantics.

## DB-MFAC-122
Factory persistence conservará type conversion semantics.

## DB-MFAC-123
Factory persistence conservará transaction semantics.

## DB-MFAC-124
Factory persistence conservará partition routing semantics.

## DB-MFAC-125
Factory persistence conservará cache consistency semantics.

## DB-MFAC-126
Factory persistence conservará telemetry semantics.

## DB-MFAC-127
ModelFactory podrá utilizarse sin Seeder System.

## DB-MFAC-128
ModelFactory podrá utilizarse sin Fixture System.

## DB-MFAC-129
ModelFactory podrá utilizarse en unit tests sin DB.

## DB-MFAC-130
ModelFactory podrá utilizarse en integration tests.

## DB-MFAC-131
ModelFactory podrá utilizarse desde CLI.

## DB-MFAC-132
ModelFactory podrá utilizarse desde Seeders.

## DB-MFAC-133
ModelFactory podrá utilizarse desde Fixtures.

## DB-MFAC-134
ModelFactory API será IDE-friendly.

## DB-MFAC-135
ModelFactory API será compatible con static analysis.

## DB-MFAC-136
ModelFactory no requerirá PDO.

## DB-MFAC-137
ModelFactory no requerirá Redis.

## DB-MFAC-138
ModelFactory no requerirá HTTP.

## DB-MFAC-139
ModelFactory no requerirá Multitenancy.

## DB-MFAC-140
ModelFactory no requerirá Sharding.

## DB-MFAC-141
Optional integrations no contaminarán Factory core.

## DB-MFAC-142
Factory state será disposable al finalizar execution.

## DB-MFAC-143
Worker reset eliminará cualquier scoped mutable Factory state restante.

## DB-MFAC-144
Factory compiler cache solo almacenará estructuras reutilizables seguras.

## DB-MFAC-145
Compiled factory artifacts serán generation-aware.

## DB-MFAC-146
Model metadata generation podrá formar parte del blueprint fingerprint.

## DB-MFAC-147
Relationship metadata generation podrá formar parte del blueprint fingerprint.

## DB-MFAC-148
Type registry generation podrá formar parte del blueprint fingerprint.

## DB-MFAC-149
ModelFactory no será fuente de database truth.

## DB-MFAC-150
Database continuará siendo autoridad del estado persistente confirmado.

## DB-MFAC-151
`make()` y `create()` tendrán límites semánticos diferentes.

## DB-MFAC-152
Convenience API nunca deberá borrar esos límites.

## DB-MFAC-153
Laravel-like syntax no implicará Laravel-like internals.

## DB-MFAC-154
Toda API conveniente deberá converger al motor canónico de VoltStack.

## DB-MFAC-155
ModelFactory y EntityFactory compartirán Factory Engine.

## DB-MFAC-156
ModelFactory y EntityFactory no crearán dos sistemas de generación incompatibles.

## DB-MFAC-157
ModelFactory será una capa de DX sobre arquitectura explícita.

## DB-MFAC-158
ModelFactory deberá ser reemplazable/extensible mediante contratos.

## DB-MFAC-159
Custom ModelFactory extensions respetarán los mismos boundaries.

## DB-MFAC-160
Ninguna extensión podrá convertir Factory en direct SQL executor sin salir explícitamente del contrato Factory.

---

# 232. Modelo formal

Sea:

```text
M = Model type
F = ModelFactory definition
S = State set
O = Overrides
R = Relationship graph
G = Generation context
```

La construcción será:

```text
ModelGraph
=
Build(M, F, S, O, R, G)
```

Mientras:

```text
make()
=
Build(...)
```

y:

```text
create()
=
Persist(
    Build(...)
)
```

Por tanto:

```text
create()
=
Persistence ∘ Construction
```

pero:

```text
Construction
≠
Persistence
```

---

# 233. Modelo de identidad

Para:

```php
$user = User::factory()->make();
```

existe:

```text
ObjectIdentity(user)
```

pero puede no existir:

```text
PersistentIdentity(user)
```

Incluso si existe un UUID preasignado:

```text
IdentifierAssigned
≠
RowExists
```

---

# 234. Modelo de estado

Simplificado:

```text
             make()
               │
               ▼
         TRANSIENT MODEL
               │
          persist/create
               ▼
           SCHEDULED
               │
             flush
               ▼
         SYNCHRONIZED?
               │
             commit
               ▼
          COMMITTED?
```

Cada `?` representa que el resultado puede requerir evidencia.

---

# 235. Modelo de relación

Factory declara:

```text
UserFactory
    has posts
```

ORM metadata determina:

```text
User.posts
=
OneToMany<Post>
```

Factory Graph construye:

```text
User
 ├── Post
 ├── Post
 └── Post
```

Persistence Planner transforma después este grafo en operaciones persistentes.

---

# 236. Modelo de dependencias

```text
ModelFactory API
       │
       ▼
Shared Factory Engine
       │
       ├────────► ORM Metadata
       ├────────► Type System
       ├────────► Relationship Metadata
       │
       ▼
Model Construction
       │
       ▼
optional PersistenceBridge
       │
       ▼
EntityManager
       │
       ▼
UnitOfWork
       │
       ▼
Persistence Engine
       │
       ▼
Query Engine
       │
       ▼
Execution
```

Dependencia inversa prohibida:

```text
Query Engine
→ ModelFactory
```

---

# 237. Experiencia Laravel-like

VoltStack buscará conservar:

```php
User::factory()
    ->count(10)
    ->has(
        Post::factory()->count(5)
    )
    ->create();
```

porque proporciona una excelente experiencia de desarrollo.

Pero internamente la arquitectura será:

```text
Simple Public API
        ↓
Typed Factory Request
        ↓
Compiled Factory Blueprint
        ↓
Factory Graph
        ↓
Model Construction
        ↓
Canonical ORM
```

Esto permite combinar:

```text
Laravel-like DX
+
Doctrine-like persistence discipline
+
VoltStack runtime architecture
```

---

# 238. Integración con el Model API

El flujo general será:

```text
                   MODEL API
                       │
              User::factory()
                       │
                       ▼
                 HasFactory
                       │
                       ▼
            ModelFactoryResolver
                       │
                       ▼
                  UserFactory
                       │
                       ▼
             Shared Factory Engine
                       │
                ┌──────┴──────┐
                │             │
              make          create
                │             │
                ▼             ▼
              User       PersistenceBridge
                              │
                              ▼
                       Canonical ORM Engine
```

---

# 239. Decisión arquitectónica

VoltStack adoptará:

```text
Laravel-like ModelFactory API
```

sobre:

```text
VoltStack Shared Factory Engine
```

y no copiará el acoplamiento interno de ningún framework existente.

Esto permite que:

```text
ModelFactory
EntityFactory
Seeder
Fixture
TestDataGenerator
```

compartan infraestructura sin convertirse en el mismo concepto.

---

# 240. Regla maestra final

> **`Model::factory()` será azúcar sintáctico de alto nivel para el Factory Engine de VoltStack; `make()` construirá Models y `create()` solicitará persistencia a través del ORM canónico, sin permitir que ModelFactory se convierta en un segundo motor de persistencia.**

La secuencia definitiva será:

```text
Model::factory()
      ↓
ModelFactory
      ↓
Factory Definition
      ↓
States / Sequences / Overrides
      ↓
Relationship Graph
      ↓
Model Construction
      ↓
      ├──────────── make()
      │
      └──────────── create()
                         ↓
                 PersistenceBridge
                         ↓
                    EntityManager
                         ↓
                     UnitOfWork
                         ↓
                Persistence Engine
                         ↓
                    Query Engine
                         ↓
                      Database
```

---

# 241. Resultado del documento

Con `194_DATABASE_MODEL_FACTORY_SYSTEM.md` queda definida la primera especialización del Factory Engine:

```text
193 DATABASE FACTORY SYSTEM
             │
             ├─────────────────────────────┐
             ▼                             ▼
194 MODEL FACTORY SYSTEM          195 ENTITY FACTORY SYSTEM
 Laravel-like API                   Data Mapper API
             │                             │
             └──────────────┬──────────────┘
                            ▼
                    Shared Factory Engine
```

La arquitectura garantiza que:

```text
Laravel-like convenience
≠
architectural coupling
```

y que VoltStack puede ofrecer:

```php
User::factory()
    ->verified()
    ->has(
        Post::factory()->count(5)
    )
    ->create();
```

manteniendo todos los invariantes del ORM, Transaction System, Distribution System y Persistent Runtime.

---

# 242. Siguiente documento

```text
195_DATABASE_ENTITY_FACTORY_SYSTEM.md
```

El siguiente documento definirá la segunda especialización:

```text
EntityFactory
```

orientada al modelo Data Mapper:

```php
$user = UserEntityFactory::new()->make();

$users = UserEntityFactory::new()
    ->count(10)
    ->make();

foreach ($users as $user) {
    $entityManager->persist($user);
}

$entityManager->flush();
```

así como una API opcional:

```php
$users = UserEntityFactory::new()
    ->count(10)
    ->persist($entityManager);
```

manteniendo:

```text
ModelFactory ──┐
               ├── Shared Factory Engine
EntityFactory ─┘
```

y la regla fundamental:

```text
EntityFactory
≠
EntityManager
≠
UnitOfWork
≠
Persistence Engine
```