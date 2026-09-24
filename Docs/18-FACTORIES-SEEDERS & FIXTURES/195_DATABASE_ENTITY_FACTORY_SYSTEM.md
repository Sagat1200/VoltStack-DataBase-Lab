# 195_DATABASE_ENTITY_FACTORY_SYSTEM.md

# VoltStack Quantum Database
## Database Entity Factory System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 195 — Database Entity Factory System  
**Bloque:** 18 — Factories, Seeders & Fixtures  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `194_DATABASE_MODEL_FACTORY_SYSTEM.md`  
**Siguiente documento:** `196_DATABASE_SEEDER_SYSTEM.md`

---

# 1. Propósito

`Database Entity Factory System` define el mecanismo de VoltStack para construir **entidades ORM puras** dentro del modelo Data Mapper.

Su API objetivo será:

```php
$user = UserEntityFactory::new()->make();
```

o:

```php
$users = UserEntityFactory::new()
    ->count(10)
    ->make();
```

Las entidades generadas podrán persistirse posteriormente mediante:

```php
foreach ($users as $user) {
    $entityManager->persist($user);
}

$entityManager->flush();
```

VoltStack también podrá proporcionar una operación explícita de conveniencia:

```php
$users = UserEntityFactory::new()
    ->count(10)
    ->persist($entityManager);
```

La regla central será:

> **EntityFactory construye grafos de entidades compatibles con el ORM Data Mapper; no administra entidades, no decide SQL y no sustituye al EntityManager, UnitOfWork ni Persistence Engine.**

Formalmente:

```text
EntityFactory
=
Entity-oriented Factory API
+
Shared Factory Engine
+
Entity Construction Strategy
+
Relationship Graph Construction
+
optional explicit Persistence Bridge
```

Nunca:

```text
EntityFactory
=
EntityManager
```

ni:

```text
EntityFactory
=
Persistence Engine
```

---

# 2. Posición arquitectónica

Los documentos anteriores establecieron:

```text
193_DATABASE_FACTORY_SYSTEM
          │
          ▼
   Shared Factory Engine
          │
     ┌────┴────┐
     │         │
     ▼         ▼
ModelFactory EntityFactory
    194         195
```

Ambas APIs reutilizan:

```text
Factory Definition
Factory Context
Factory States
Factory Sequences
Factory Graph
Factory Random Source
Factory Clock
Factory Attribute Resolution
Factory Relationship Planning
Factory Diagnostics
```

pero tienen distinta semántica de integración.

---

# 3. ModelFactory vs EntityFactory

La separación fundamental es:

| Aspecto | ModelFactory | EntityFactory |
|---|---|---|
| Paradigma | Active Record-like DX | Data Mapper |
| Objeto | Model | Entity |
| Entrada típica | `User::factory()` | `UserEntityFactory::new()` |
| Persistence shortcut | `create()` | opcional `persist($em)` |
| EntityManager explícito | normalmente oculto por Model API | preferentemente explícito |
| `save()` | puede existir | no requerido |
| Static API | común | no necesaria |
| Factory Engine | compartido | compartido |
| SQL | nunca | nunca |
| UnitOfWork propio | no | no |

---

# 4. Regla de diseño

VoltStack deberá permitir simultáneamente:

```php
User::factory()->create();
```

y:

```php
$user = UserEntityFactory::new()->make();

$entityManager->persist($user);
$entityManager->flush();
```

sin crear dos motores ORM.

Ambos convergen en:

```text
             ModelFactory
                  │
                  │
                  ▼
             Model API Bridge
                  │
                  │
                  ├───────────────┐
                  │               │
                  ▼               │
             EntityManager ◄──────┤
                  ▲               │
                  │               │
                  │         EntityFactory
                  │               │
                  └───────────────┘
```

---

# 5. Objetivos

El sistema deberá proporcionar:

1. construcción de entidades sin DB;
2. entidades con constructor obligatorio;
3. Value Objects;
4. enums;
5. estados;
6. sequences;
7. overrides;
8. relationships;
9. aggregate graphs;
10. deterministic generation;
11. explicit persistence;
12. UnitOfWork integration;
13. IdentityMap integration durante persistence/hydration, no generación;
14. lifecycle-safe callbacks;
15. batch generation;
16. transaction compatibility;
17. tenant/shard awareness opcional;
18. persistent-runtime safety;
19. IDE/static-analysis support;
20. testability.

---

# 6. API mínima

```php
$user = UserEntityFactory::new()->make();
```

Múltiples:

```php
$users = UserEntityFactory::new()
    ->count(10)
    ->make();
```

Estados:

```php
$user = UserEntityFactory::new()
    ->verified()
    ->premium()
    ->make();
```

Overrides:

```php
$user = UserEntityFactory::new()
    ->make([
        'email' => new Email('john@example.test'),
    ]);
```

---

# 7. Entidad pura

Una entidad podrá ser:

```php
final class User
{
    public function __construct(
        private UserId $id,
        private UserName $name,
        private Email $email,
        private UserStatus $status,
    ) {}
}
```

No requiere extender:

```text
Model
```

ni usar:

```text
HasFactory
```

---

# 8. Factory explícita

```php
/**
 * @extends EntityFactory<User>
 */
final class UserEntityFactory extends EntityFactory
{
    protected function entityType(): string
    {
        return User::class;
    }
}
```

---

# 9. Base EntityFactory

Propuesta:

```php
/**
 * @template TEntity of object
 */
abstract class EntityFactory extends Factory
{
    /**
     * @return class-string<TEntity>
     */
    abstract protected function entityType(): string;

    abstract protected function definition(
        FactoryContext $context
    ): array;
}
```

---

# 10. new()

La API:

```php
UserEntityFactory::new();
```

será un convenience constructor.

Conceptualmente:

```php
public static function new(): static
{
    return FactoryRuntime::resolve(static::class);
}
```

pero no deberá depender de estado estático mutable.

---

# 11. new() ≠ singleton

Cada:

```php
UserEntityFactory::new()
```

representará una configuración independiente.

No:

```text
UserEntityFactory singleton
```

con state mutable compartido.

---

# 12. Inmutabilidad

Preferencia:

```php
$base = UserEntityFactory::new();

$verified = $base->verified();

$pending = $base->pending();
```

`$base` no deberá modificarse.

---

# 13. make()

```php
$user = UserEntityFactory::new()->make();
```

significa:

```text
Resolve Factory
      ↓
Compile/Reuse Blueprint
      ↓
Create Factory Context
      ↓
Resolve Attributes
      ↓
Construct Entity
      ↓
Construct Relationships
      ↓
Return Entity
```

No significa:

```text
persist
flush
commit
```

---

# 14. make() y database

La operación:

```php
UserEntityFactory::new()->make();
```

deberá poder ejecutarse sin conexión a base de datos.

---

# 15. Beneficio

Esto permite unit tests:

```php
$user = UserEntityFactory::new()
    ->verified()
    ->make();

$service->execute($user);
```

sin bootear:

```text
ConnectionManager
Driver
Database Server
```

---

# 16. EntityFactory ≠ ORM-managed entity

Una entidad construida no estará necesariamente:

```text
MANAGED
```

por el ORM.

Será normalmente:

```text
NEW / TRANSIENT
```

---

# 17. IdentityMap

`make()` no deberá insertar automáticamente la entidad en:

```text
IdentityMap
```

---

# 18. UnitOfWork

Tampoco deberá ejecutar:

```php
$unitOfWork->registerNew($entity);
```

directamente.

---

# 19. EntityManager

La transición a entidad administrada comienza mediante:

```php
$entityManager->persist($entity);
```

---

# 20. Persist explícito

Flujo recomendado:

```php
$user = UserEntityFactory::new()->make();

$entityManager->persist($user);
$entityManager->flush();
```

Esta será la forma arquitectónicamente más explícita.

---

# 21. persist() convenience

VoltStack podrá ofrecer:

```php
$user = UserEntityFactory::new()
    ->persist($entityManager);
```

---

# 22. Semántica de persist()

`persist($entityManager)` será conceptualmente:

```text
make()
  ↓
EntityManager::persist()
  ↓
optional flush according to explicit policy
```

---

# 23. Ambigüedad de persist()

Debido a que:

```php
$entityManager->persist($entity);
```

en VoltStack significa:

```text
register persistence intent
```

y no:

```text
INSERT immediately
```

la API Factory deberá evitar una semántica ambigua.

---

# 24. API recomendada

Se recomienda separar:

```php
$entity = UserEntityFactory::new()
    ->makeAndPersist($entityManager);
```

de:

```php
$entity = UserEntityFactory::new()
    ->create($entityManager);
```

si ambas llegan a existir.

---

# 25. Semántica canónica

Propuesta definitiva:

```text
make()
    construct only

persist($entityManager)
    construct + EntityManager::persist()
    no implicit flush

create($entityManager)
    construct + persist + flush
```

---

# 26. Diferencia crítica

Por tanto:

```text
persist()
≠
create()
```

---

# 27. Ejemplo

```php
$user = UserEntityFactory::new()
    ->persist($entityManager);
```

Estado:

```text
entity constructed
entity registered in UoW
database synchronization not guaranteed yet
```

Posteriormente:

```php
$entityManager->flush();
```

---

# 28. create()

```php
$user = UserEntityFactory::new()
    ->create($entityManager);
```

podrá ejecutar:

```text
make
 ↓
persist
 ↓
flush
```

pero nunca:

```text
commit
```

salvo que sea propietaria de una transaction creada explícitamente por policy.

---

# 29. create() ≠ commit

Regla:

> **EntityFactory::create() podrá sincronizar el UnitOfWork, pero no convierte `flush()` en `commit()`.**

---

# 30. Transaction existente

Dentro de:

```php
DB::transaction(function () use ($entityManager) {
    UserEntityFactory::new()
        ->create($entityManager);
});
```

`create()` no deberá cerrar la transacción.

---

# 31. Factory definition

Ejemplo:

```php
final class UserEntityFactory extends EntityFactory
{
    protected function entityType(): string
    {
        return User::class;
    }

    protected function definition(
        FactoryContext $context
    ): array {
        return [
            'id' => UserId::new(),

            'name' => new UserName(
                $context->data()->person()->name()
            ),

            'email' => new Email(
                $context->data()
                    ->internet()
                    ->uniqueEmail()
            ),

            'status' => UserStatus::ACTIVE,
        ];
    }
}
```

---

# 32. Value Objects first-class

EntityFactory deberá tratar como valores naturales:

```text
UserId
Email
Money
Address
DateRange
Coordinates
```

sin convertirlos prematuramente a valores DB.

---

# 33. Factory representation

Factory trabaja en:

```text
Application / Domain Representation
```

no:

```text
Database Physical Representation
```

---

# 34. Ejemplo Money

```php
'creditLimit' => Money::of(
    10000,
    Currency::MXN
),
```

No:

```php
'credit_limit' => '10000.00',
'currency' => 'MXN',
```

si el dominio espera `Money`.

---

# 35. Mapping posterior

El ORM realizará:

```text
Money
 ↓
Value Object Mapping
 ↓
Canonical Persistent Values
 ↓
Database Types
 ↓
Binding
```

---

# 36. Constructor strategy

A diferencia de Models estilo Active Record, las Entities podrán utilizar constructores estrictos.

Ejemplo:

```php
final class Invoice
{
    public function __construct(
        InvoiceId $id,
        Customer $customer,
        Money $total,
        InvoiceStatus $status,
    ) {}
}
```

---

# 37. Construction Plan

El Factory Engine deberá compilar:

```text
EntityConstructionPlan
```

con:

```text
constructor parameters
attribute sources
relationship sources
defaults
Value Object requirements
post-construction assignments
```

---

# 38. Constructor parameter matching

No deberá depender únicamente de:

```text
array key == constructor parameter
```

aunque pueda usarse como convención.

Metadata explícita podrá resolver casos complejos.

---

# 39. Explicit construction

Una Factory podrá sobrescribir:

```php
protected function construct(
    ResolvedFactoryAttributes $attributes
): User {
    return new User(
        id: $attributes->require('id'),
        name: $attributes->require('name'),
        email: $attributes->require('email'),
        status: $attributes->require('status'),
    );
}
```

---

# 40. Private constructors

Entidades con factories de dominio:

```php
final class User
{
    private function __construct(...) {}

    public static function register(...): self
    {
        // domain invariants
    }
}
```

podrán integrarse mediante:

```php
protected function construct(
    ResolvedFactoryAttributes $attributes
): User {
    return User::register(...);
}
```

---

# 41. Factory no rompe encapsulación

No deberá utilizar reflection privada automáticamente solo porque el constructor no sea público.

---

# 42. Aggregate factories

EntityFactory deberá soportar Aggregate Roots.

Ejemplo:

```text
Order
├── OrderLine
├── OrderLine
└── ShippingAddress
```

---

# 43. Aggregate example

```php
$order = OrderEntityFactory::new()
    ->withLines(3)
    ->make();
```

---

# 44. Aggregate construction

Podrá producir:

```text
Order
├── OrderLine × 3
└── Address
```

completamente en memoria.

---

# 45. Persistence independence

El aggregate podrá utilizarse:

```php
$order = OrderEntityFactory::new()
    ->withLines(3)
    ->make();

$pricingService->calculate($order);
```

sin DB.

---

# 46. Relationship factory

Ejemplo:

```php
$user = UserEntityFactory::new()
    ->has(
        PostEntityFactory::new()->count(5),
        'posts'
    )
    ->make();
```

---

# 47. Relationship metadata

La Factory deberá validar:

```text
User.posts
```

contra `Relationship Metadata System`.

---

# 48. Domain methods

Para entidades encapsuladas, la relación puede construirse mediante:

```php
$user->addPost($post);
```

en lugar de escribir una colección privada.

---

# 49. Relationship assignment strategy

Se soportarán estrategias:

```text
CONSTRUCTOR
DOMAIN_METHOD
PROPERTY
COLLECTION_ADAPTER
CUSTOM
```

---

# 50. Default recomendado

Preferencia:

```text
CONSTRUCTOR
or
DOMAIN_METHOD
```

para entidades de dominio ricas.

---

# 51. Property mutation

Solo cuando mapping/domain design lo permita.

---

# 52. Bidirectional relationships

Para:

```text
User.posts
Post.author
```

Factory Graph deberá poder establecer:

```text
User.posts contains Post
Post.author = User
```

sin producir dos entidades User.

---

# 53. Canonical graph identity

Dentro de un mismo Factory Graph:

```text
same logical generated parent
```

deberá reutilizar:

```text
same PHP object
```

cuando la relación lo requiera.

---

# 54. Factory Graph Identity ≠ IdentityMap

La deduplicación temporal del Factory Graph no es el ORM `IdentityMap`.

---

# 55. Razón

IdentityMap garantiza:

```text
persistent identity
→ canonical managed object
```

Factory Graph Identity garantiza:

```text
factory node reference
→ same generated object
```

---

# 56. Generated IDs

Una entidad podrá recibir:

```php
UserId::new()
```

antes de persistence.

---

# 57. Assigned ID ≠ persisted identity

Siempre:

```text
Identifier exists
≠
Database row confirmed
```

---

# 58. DB-generated identifiers

También deberán soportarse:

```text
id = UNASSIGNED
```

hasta persistence.

---

# 59. Factory placeholder

Internamente podrá existir:

```text
GeneratedIdentifier::databaseAssigned()
```

pero no deberá exponerse como ID real del dominio si el constructor no lo permite.

---

# 60. Constructor con DB-generated ID

Entidades que requieren ID en constructor favorecen IDs application-generated.

Para auto-increment, deberá existir una strategy explícita.

---

# 61. Entity state

Una entidad generada mediante Factory puede estar:

```text
TRANSIENT
NEW
```

sin necesidad de que la entidad misma almacene esa bandera.

---

# 62. Estado externo

`EntityState System` es responsable del estado ORM.

La Entity no necesita:

```php
$entity->exists = false;
```

---

# 63. Diferencia con ModelFactory

Esta es una diferencia importante:

```text
Model API
may expose persistence convenience state

Data Mapper Entity
does not need persistence infrastructure fields
```

---

# 64. States

Ejemplo:

```php
public function verified(): static
{
    return $this->state(
        fn (FactoryContext $context) => [
            'verifiedAt' => $context->clock()->now(),
        ]
    );
}
```

---

# 65. Domain state

También:

```php
public function suspended(): static
{
    return $this->state([
        'status' => UserStatus::SUSPENDED,
    ]);
}
```

---

# 66. State composition

```php
$user = UserEntityFactory::new()
    ->verified()
    ->premium()
    ->make();
```

---

# 67. Invalid state combinations

Si:

```text
SUSPENDED + ACTIVE
```

es inválido, el Domain Constructor o Factory Validation podrá detectarlo.

---

# 68. Factory state ≠ mutation history

States describen el resultado deseado.

No necesariamente simulan:

```text
register
verify
suspend
reactivate
```

como una secuencia temporal de comandos.

---

# 69. Historical scenario

Cuando un test necesita comportamiento histórico real deberá utilizar:

```text
Fixture
Domain Commands
Event history
```

según arquitectura.

---

# 70. Sequences

```php
$users = UserEntityFactory::new()
    ->count(3)
    ->sequence(
        ['status' => UserStatus::ACTIVE],
        ['status' => UserStatus::PENDING],
        ['status' => UserStatus::SUSPENDED],
    )
    ->make();
```

---

# 71. Explicit overrides

```php
$user = UserEntityFactory::new()
    ->verified()
    ->make([
        'email' => new Email(
            'john@example.test'
        ),
    ]);
```

---

# 72. Typed overrides

VoltStack podrá proporcionar tooling:

```php
UserEntityFactory::new()
    ->withEmail(
        new Email('john@example.test')
    )
    ->make();
```

para evitar arrays cuando se desee mayor type safety.

---

# 73. Factory DSL

Ejemplo:

```php
$order = OrderEntityFactory::new()
    ->forCustomer($customer)
    ->withProduct($product, quantity: 3)
    ->paid()
    ->make();
```

---

# 74. DSL domain-aware

Esta API puede ser preferible a:

```php
->state([...])
```

para aggregate factories complejas.

---

# 75. Existing entities

Podrán suministrarse entidades existentes:

```php
$order = OrderEntityFactory::new()
    ->forCustomer($customer)
    ->make();
```

---

# 76. Existing entity ≠ managed entity

La Factory no asumirá que `$customer` está managed.

---

# 77. Persistence validation

Al persistir el grafo, EntityManager determinará:

```text
managed?
detached?
new?
invalid?
different context?
```

---

# 78. Cross-EntityManager

Una entidad administrada por un EntityManager incompatible no deberá adjuntarse silenciosamente a otro contexto.

---

# 79. Relationship cascade

Factory graph construction no determina automáticamente ORM cascade.

---

# 80. Ejemplo

Factory puede construir:

```text
User
└── Profile
```

pero si mapping dice:

```text
cascade persist = false
```

entonces:

```php
$entityManager->persist($user);
```

no necesariamente persiste `Profile`.

---

# 81. Factory persistence graph

Si:

```php
UserEntityFactory::new()
    ->has(ProfileEntityFactory::new())
    ->create($entityManager);
```

promete persistir el grafo completo, deberá registrar explícitamente los nodos necesarios respetando metadata.

---

# 82. No cascade fabrication

No deberá cambiar metadata ORM para hacer que el ejemplo funcione.

---

# 83. Persistence graph planner

El Factory Persistence Bridge podrá transformar:

```text
FactoryGraph
```

en:

```text
Entity registration requests
```

antes del UoW.

---

# 84. UnitOfWork authority

Una vez registradas:

```text
UnitOfWork
```

determina los cambios persistentes.

---

# 85. Persistence Planner authority

Después:

```text
Persistence Planner
```

determina ordering/dependencies.

---

# 86. Factory no decide INSERT ordering

Nunca:

```text
EntityFactory:
insert user
then insert profile
then update ...
```

---

# 87. One-to-one

```php
$user = UserEntityFactory::new()
    ->has(
        ProfileEntityFactory::new(),
        'profile'
    )
    ->make();
```

---

# 88. One-to-many

```php
$user = UserEntityFactory::new()
    ->has(
        PostEntityFactory::new()->count(5),
        'posts'
    )
    ->make();
```

---

# 89. Many-to-one

```php
$post = PostEntityFactory::new()
    ->for($user, 'author')
    ->make();
```

---

# 90. Many-to-many

```php
$user = UserEntityFactory::new()
    ->hasAttached(
        RoleEntityFactory::new()->count(3),
        'roles'
    )
    ->make();
```

---

# 91. Association Entity

Cuando existen atributos propios:

```text
User
  ↓
ProjectMembership
  ↓
Project
```

deberá utilizarse:

```php
ProjectMembershipEntityFactory::new();
```

---

# 92. Polymorphic relationships

Si el ORM permite polymorphic mapping:

```php
$comment = CommentEntityFactory::new()
    ->for($post, 'commentable')
    ->make();
```

utilizará canonical relationship metadata.

---

# 93. No persisted FQCN

EntityFactory no inventará:

```text
App\Domain\Post
```

como morph discriminator.

---

# 94. Nested graph

```php
$user = UserEntityFactory::new()
    ->has(
        PostEntityFactory::new()
            ->count(5)
            ->has(
                CommentEntityFactory::new()
                    ->count(3),
                'comments'
            ),
        'posts'
    )
    ->make();
```

---

# 95. Factory graph

```text
User
│
├── Post
│   ├── Comment
│   ├── Comment
│   └── Comment
│
├── Post
│   └── ...
│
└── ...
```

---

# 96. Graph cycles

Relaciones bidireccionales pueden producir:

```text
User
 ↓
Post
 ↓
author
 ↓
User
```

El Factory Graph debe reconocer referencias existentes.

No deberá generar infinitamente:

```text
User → Post → User → Post → ...
```

---

# 97. Graph node identity

Cada nodo tendrá conceptualmente:

```text
FactoryNodeId
```

independiente del ID persistente.

---

# 98. Reference edges

El grafo podrá representar:

```text
Node A
  ↓ relation
Node B

Node B
  ↓ inverse
reference Node A
```

---

# 99. Recursion limits

Además deberá existir:

```text
maxDepth
maxNodes
maxRelationships
```

---

# 100. Graph explosion

Una configuración:

```text
10 users
× 100 posts
× 100 comments
```

produce:

```text
100,010 objects
```

aproximadamente.

El sistema deberá poder estimarlo y aplicar budgets.

---

# 101. Deterministic generation

```php
$users = UserEntityFactory::new()
    ->seed(100)
    ->count(10)
    ->make();
```

deberá ser reproducible bajo providers deterministas.

---

# 102. Factory Clock

```php
$factory = UserEntityFactory::new()
    ->clock(
        FixedClock::at(
            '2026-01-01T00:00:00Z'
        )
    );
```

---

# 103. Domain time

Una entidad podrá recibir:

```php
new RegisteredAt(
    $context->clock()->now()
)
```

---

# 104. No implicit wall clock

Para reproducibilidad, evitar dentro de Factory:

```php
new DateTimeImmutable();
```

cuando el `FactoryClock` pueda utilizarse.

---

# 105. Unique generation

Factory podrá garantizar uniqueness dentro de su execution scope.

---

# 106. DB uniqueness

No garantiza ausencia en DB.

---

# 107. Pure EntityFactory

Una Factory pura deberá ser:

```text
deterministic given context
database independent
side-effect free during construction
```

en la medida razonable.

---

# 108. I/O-aware factory

Cuando una Factory requiera información externa:

```text
database
filesystem
network
```

deberá declararlo explícitamente.

---

# 109. Capability metadata

Podrá existir:

```php
enum FactoryCapability
{
    case PURE;
    case DATABASE_READ;
    case FILESYSTEM_READ;
    case EXTERNAL_IO;
}
```

---

# 110. Default

```text
PURE
```

será preferido para EntityFactory.

---

# 111. Callbacks

EntityFactory podrá ofrecer:

```text
afterMaking
afterPersisting
afterPersisted
afterFlush
afterCommit
```

con semánticas explícitas.

---

# 112. Fases

```text
ATTRIBUTES_RESOLVED
       ↓
ENTITY_CONSTRUCTED
       ↓
GRAPH_LINKED
       ↓
AFTER_MAKING
       ↓
optional UOW_REGISTRATION
       ↓
AFTER_PERSISTING
       ↓
optional FLUSH
       ↓
AFTER_FLUSH
       ↓
optional COMMIT
       ↓
AFTER_COMMIT
```

---

# 113. Naming

Evitar un único:

```text
afterCreating
```

si resulta ambiguo en Data Mapper.

---

# 114. afterPersisting

Debe significar:

```text
entity registered for persistence
```

no:

```text
row definitely committed
```

---

# 115. afterFlush

Significa:

```text
flush completed sufficiently successfully
```

pero todavía puede existir una transacción activa.

---

# 116. afterCommit

Solo después de commit confirmado.

---

# 117. Callback failure

Un callback posterior a flush que falla:

```text
does not prove database rollback
```

---

# 118. Side effects

External side effects deberán preferir:

```text
afterCommit
Outbox
Queue
```

según necesidad.

---

# 119. Persistence policies

Propuesta:

```php
enum EntityFactoryPersistenceMode
{
    case NONE;
    case REGISTER;
    case FLUSH;
}
```

---

# 120. NONE

Usado por:

```php
->make();
```

---

# 121. REGISTER

Usado por:

```php
->persist($entityManager);
```

---

# 122. FLUSH

Usado por:

```php
->create($entityManager);
```

---

# 123. Transaction policy

Separada:

```php
enum EntityFactoryTransactionPolicy
{
    case USE_EXISTING;
    case NONE;
    case REQUIRE;
    case CREATE_IF_MISSING;
}
```

---

# 124. Factory transaction ownership

Si Factory crea una transaction, podrá ser propietaria.

Si recibe una existente:

```text
owner = caller
```

---

# 125. No foreign commit

Factory jamás hará commit de una transacción ajena.

---

# 126. Flush scope

Especial cuidado:

```php
$entityManager->persist($unrelatedEntity);

UserEntityFactory::new()
    ->create($entityManager);
```

Un `flush()` global podría persistir también `$unrelatedEntity`.

---

# 127. Riesgo de create()

Por ello `create($entityManager)` deberá documentar claramente su flush scope.

---

# 128. Alternativas

VoltStack podrá soportar:

```text
GLOBAL_UOW_FLUSH
FACTORY_GRAPH_FLUSH
EXPLICIT_ONLY
```

si el ORM puede garantizar correctamente esas semánticas.

---

# 129. Default conservador

En Data Mapper, puede ser preferible:

```text
persist()
+
caller-controlled flush()
```

como patrón recomendado.

---

# 130. Razón

Evita que una Factory controle accidentalmente el UnitOfWork completo.

---

# 131. API recomendada principal

Por ello:

```php
$users = UserEntityFactory::new()
    ->count(10)
    ->persist($entityManager);

$entityManager->flush();
```

es preferible en código avanzado.

---

# 132. Batch generation

```php
$users = UserEntityFactory::new()
    ->count(10_000)
    ->make();
```

puede consumir memoria considerable.

---

# 133. Streaming factory

Podrá existir:

```php
foreach (
    UserEntityFactory::new()
        ->count(1_000_000)
        ->stream()
    as $user
) {
    // ...
}
```

---

# 134. stream()

Deberá producir entidades progresivamente sin acumular toda la colección.

---

# 135. Graph restrictions

Streaming de grafos complejos deberá respetar dependencias y ownership.

---

# 136. Persistence streaming

Ejemplo:

```php
foreach (
    UserEntityFactory::new()
        ->count(1_000_000)
        ->stream()
    as $user
) {
    $entityManager->persist($user);

    // controlled flush/clear policy
}
```

---

# 137. Factory no limpia EntityManager

La Factory no deberá ejecutar silenciosamente:

```php
$entityManager->clear();
```

porque podría detachar entidades ajenas.

---

# 138. Explicit batch helper

Podrá existir tooling:

```php
EntityFactoryBatchRunner::run(
    factory: UserEntityFactory::new()
        ->count(1_000_000),
    entityManager: $entityManager,
    batchSize: 1000,
);
```

con políticas explícitas.

---

# 139. Bulk insert

Para datasets enormes sin necesidad de entidades:

```text
DATABASE_BULK_INSERT_SYSTEM
```

será preferible.

---

# 140. EntityFactory ≠ bulk loader

EntityFactory conserva semántica de dominio y ORM.

---

# 141. Validation

Puede existir:

```text
FactoryValidationPolicy
```

pero el constructor/domain methods siguen siendo autoridad.

---

# 142. Constructor exception

Si:

```php
new Email('invalid')
```

falla, la Factory no deberá ignorarlo.

---

# 143. Invalid entity testing

Para probar estados imposibles se necesitarán herramientas especializadas de testing.

No deberá ser el comportamiento normal de EntityFactory.

---

# 144. Entity invariants

Una EntityFactory deberá favorecer la creación de:

```text
valid domain objects
```

---

# 145. Fixtures para escenarios inválidos

Cuando se necesiten datos DB legacy inválidos:

```text
Fixture
Raw Test Data
Migration Test Tools
```

podrán ser más apropiados.

---

# 146. Persistence errors

Si:

```php
$entityManager->flush();
```

falla, la Factory no reinterpretará el error.

---

# 147. Persistence consistency

Se aplican los estados:

```text
CONSISTENT
STALE
UNCERTAIN
INCONSISTENT
TAINTED
UNKNOWN
```

definidos previamente.

---

# 148. Commit unknown

Caso:

```text
COMMIT
 ↓
network loss
 ↓
UNKNOWN
```

EntityFactory no podrá afirmar:

```text
all entities persisted
```

---

# 149. Rollback

Siempre:

```text
DatabaseRollback
≠
ObjectGraphRewind
```

---

# 150. Entity graph después de rollback

Las entidades siguen existiendo en memoria.

Sus IDs y relaciones también pueden seguir presentes.

---

# 151. UnitOfWork recovery

Será responsabilidad del ORM decidir:

```text
clear
detach
taint
rebuild
```

según contexto.

---

# 152. EntityFactory no repara UoW

No deberá intentar corregir silenciosamente el EntityManager después de un failure incierto.

---

# 153. Sharding

EntityFactory podrá generar el shard key como atributo de dominio.

Ejemplo:

```php
'region' => Region::MEXICO,
```

---

# 154. Routing posterior

```text
Entity
 ↓
Persistence
 ↓
Partition Routing
 ↓
Shard resolution
```

---

# 155. EntityFactory ≠ ShardRouter

La Factory no determina:

```text
ShardId
Endpoint
Replica
Connection
```

salvo tooling explícito de infraestructura.

---

# 156. Cross-shard graph

Si el grafo contiene entidades que pertenecen a shards diferentes:

```text
Factory Graph
      ↓
Persistence Analysis
      ↓
Cross-Shard Domain Detected
```

---

# 157. Atomicity

No deberá fabricarse:

```text
single ACID transaction
```

si el grafo cruza recursos independientes.

---

# 158. Tenant integration

Si `NeuronTenant`/Multitenancy está instalado, podrá existir:

```php
UserEntityFactory::new()
    ->forTenant($tenant)
    ->make();
```

---

# 159. Tenant semantics

Esto puede influir en:

```text
tenant-owned attributes
relationships
persistence context
routing
cache identity
```

---

# 160. Tenant ≠ EntityFactory core

La Factory base deberá funcionar sin Multitenancy.

---

# 161. Cross-tenant references

Deben validarse según domain/relationship policies.

---

# 162. Cache interaction

Una EntityFactory pura no utilizará:

```text
Entity Cache
Result Cache
Query Cache
```

para generar entidades.

---

# 163. Metadata cache

Puede beneficiarse indirectamente de:

```text
compiled ORM metadata
factory blueprint cache
relationship metadata cache
```

porque contienen estructuras, no datos de entidad generados.

---

# 164. Entity Cache distinction

Una entidad generada por Factory:

```text
must not be inserted directly
into second-level Entity Cache
```

---

# 165. Razón

Entity Cache representa:

```text
persistent state observed/confirmed
```

Factory representa:

```text
synthetic construction
```

---

# 166. IdentityMap distinction

Igualmente:

```text
Factory Graph Registry
≠
IdentityMap
```

---

# 167. Hydration distinction

EntityFactory:

```text
creates synthetic entities
```

Hydrator:

```text
reconstructs entities from executed database results
```

---

# 168. Diferencia formal

```text
EntityFactory:
SyntheticInput → Entity

Hydrator:
DatabaseResult → Entity
```

---

# 169. Serialization

Factory blueprints podrán ser serializables si contienen únicamente definiciones seguras.

---

# 170. Closures

Closures arbitrarios pueden impedir serialización/cache persistente.

---

# 171. Compiled callbacks

VoltStack deberá diferenciar:

```text
cacheable factory definition
runtime-only callback
```

---

# 172. Factory fingerprint

Podrá calcularse con:

```text
FactoryType
EntityType
FactoryDefinitionVersion
ORMMetadataGeneration
RelationshipMetadataGeneration
TypeRegistryGeneration
FrameworkVersion
```

---

# 173. Persistent runtime

Bajo FrankenPHP:

```text
Worker
├── immutable compiled factory metadata
├── immutable ORM metadata
│
├── Request A
│   └── FactoryContext A
│
└── Request B
    └── FactoryContext B
```

---

# 174. Mutable execution state

Siempre scoped:

```text
random generator
sequence index
unique value pool
graph node registry
generated entities
callbacks pending
tenant context
transaction references
```

---

# 175. No entity leakage

Una entidad generada durante Request A jamás deberá aparecer automáticamente en Request B.

---

# 176. OpenSwoole

El sistema deberá ser coroutine-safe.

---

# 177. RoadRunner

No deberá asumir process-per-request.

---

# 178. FrankenPHP

Será el runtime persistente principal considerado inicialmente.

---

# 179. Worker reset

Cualquier scoped Factory state residual deberá eliminarse al finalizar la operación/request.

---

# 180. Telemetry

Eventos:

```text
EntityFactoryResolved
EntityFactoryStarted
EntityFactoryAttributesResolved
EntityFactoryEntityConstructed
EntityFactoryGraphConstructed
EntityFactoryPersistenceRegistered
EntityFactoryFlushCompleted
EntityFactoryCompleted
EntityFactoryFailed
```

---

# 181. Métricas

```text
db.factory.entity.executions
db.factory.entity.constructed
db.factory.entity.graph_nodes
db.factory.entity.persist_registered
db.factory.entity.failures
db.factory.entity.duration
```

---

# 182. Cardinalidad

No incluir como labels:

```text
entity ID
email
tenant key raw
shard key raw
generated random values
```

---

# 183. Diagnostics

API conceptual:

```php
DB::factories()->explain(
    UserEntityFactory::new()
        ->verified()
        ->has(
            PostEntityFactory::new()
                ->count(3),
            'posts'
        )
);
```

---

# 184. Resultado de diagnóstico

```text
ENTITY FACTORY PLAN

Factory
-------
UserEntityFactory

Entity
------
App\Domain\User

Mode
----
ENTITY / DATA_MAPPER

Count
-----
1

Persistence
-----------
NONE

States
------
verified

Relationships
-------------
posts
  target: App\Domain\Post
  type: ONE_TO_MANY
  count: 3

Graph Nodes
-----------
4

Database Required
-----------------
NO

EntityManager Required
----------------------
NO

Deterministic
-------------
YES
```

---

# 185. Diagnóstico persist()

```text
Persistence Mode
----------------
REGISTER

EntityManager
-------------
EXPLICIT

Implicit Flush
--------------
NO

Implicit Commit
---------------
NO
```

---

# 186. Diagnóstico create()

```text
Persistence Mode
----------------
FLUSH

EntityManager
-------------
EXPLICIT

Flush Scope
-----------
CONFIGURED

Implicit Commit
---------------
NO
```

---

# 187. Security

EntityFactory no es una autorización para crear registros.

---

# 188. Production safety

No se deberán exponer automáticamente factories mediante:

```text
HTTP
RPC
Console unauthenticated
remote debugging
```

---

# 189. Synthetic secrets

No generar secrets de producción mediante test data providers.

---

# 190. Sensitive values

Telemetry y diagnostics deberán redacted/hash values sensibles.

---

# 191. Serialization safety

Factory cached definitions no utilizarán:

```php
unserialize($untrustedPayload);
```

con arbitrary object instantiation.

---

# 192. User-provided factory definitions

Si en el futuro existe dynamic factory configuration, deberá validarse como input no confiable.

---

# 193. Directory structure

Propuesta:

```text
src/Quantum/Database/Factory/Entity/
│
├── EntityFactory.php
├── EntityFactoryBlueprint.php
├── EntityFactoryContext.php
├── EntityFactoryRequest.php
├── EntityFactoryCollection.php
│
├── Construction/
│   ├── EntityConstructor.php
│   ├── EntityConstructionPlan.php
│   ├── EntityConstructionStrategy.php
│   ├── ConstructorStrategy.php
│   ├── DomainMethodStrategy.php
│   └── PropertyAssignmentStrategy.php
│
├── Graph/
│   ├── EntityFactoryGraph.php
│   ├── EntityFactoryGraphNode.php
│   ├── EntityFactoryGraphEdge.php
│   ├── EntityFactoryGraphPlanner.php
│   ├── EntityFactoryGraphRegistry.php
│   └── EntityFactoryGraphBudget.php
│
├── Relationship/
│   ├── EntityFactoryRelationship.php
│   ├── EntityFactoryRelationshipResolver.php
│   ├── EntityFactoryHas.php
│   ├── EntityFactoryFor.php
│   └── EntityFactoryAttached.php
│
├── Persistence/
│   ├── EntityFactoryPersistenceBridge.php
│   ├── EntityFactoryPersistenceMode.php
│   ├── EntityFactoryPersistencePlan.php
│   ├── EntityFactoryTransactionPolicy.php
│   ├── EntityFactoryFlushPolicy.php
│   └── EntityFactoryPersistenceResult.php
│
├── Callback/
│   ├── EntityFactoryCallback.php
│   ├── EntityFactoryCallbackPhase.php
│   └── EntityFactoryCallbackRegistry.php
│
├── Runtime/
│   └── EntityFactoryRuntimeState.php
│
├── Diagnostics/
│   ├── EntityFactoryInspector.php
│   └── EntityFactoryExplainer.php
│
├── Telemetry/
│   └── EntityFactoryTelemetry.php
│
└── Exception/
    ├── EntityFactoryException.php
    ├── EntityConstructionException.php
    ├── EntityFactoryGraphException.php
    ├── EntityFactoryRelationshipException.php
    └── EntityFactoryPersistenceException.php
```

---

# 194. Error hierarchy

```text
DatabaseFactoryException
└── EntityFactoryException
    ├── InvalidEntityFactoryException
    ├── EntityConstructionException
    ├── EntityFactoryStateException
    ├── EntityFactoryAttributeException
    ├── EntityFactoryGraphException
    │   ├── EntityFactoryGraphCycleException
    │   └── EntityFactoryGraphBudgetException
    ├── EntityFactoryRelationshipException
    ├── EntityFactoryPersistenceException
    └── EntityFactoryCallbackException
```

---

# 195. Constructor error

Ejemplo:

```text
Entity construction failed.

Factory:
    UserEntityFactory

Entity:
    App\Domain\User

Constructor parameter:
    email

Expected:
    App\Domain\Email

Received:
    string

Factory path:
    UserEntityFactory.email
```

---

# 196. Relationship error

```text
Invalid EntityFactory relationship.

Source:
    App\Domain\User

Relationship:
    posts

Expected target:
    App\Domain\Post

Provided factory:
    InvoiceEntityFactory

Provided target:
    App\Domain\Invoice
```

---

# 197. Persistence error

Debe conservar `previous exception` y contexto seguro.

No deberá convertir:

```text
deadlock
constraint violation
connection failure
unknown commit
```

en un genérico:

```text
Factory failed
```

perdiendo semántica.

---

# 198. Test matrix

| Área | Prueba |
|---|---|
| Pure make | sin DB |
| Constructor | parámetros correctos |
| Value Objects | preservados |
| States | composición |
| Sequence | determinismo |
| Relationships | graph correcto |
| Bidirectional | misma instancia |
| Cycles | sin recursión infinita |
| Aggregate | invariantes |
| persist | registra UoW |
| persist | no flush implícito |
| create | flush según policy |
| create | no commit ajeno |
| rollback | no graph rewind |
| UNKNOWN | no fabricated success |
| Batch | memoria acotada |
| Tenant | aislamiento |
| Shard | routing posterior |
| Worker | no leakage |
| Coroutine | isolation |
| Telemetry | sin PII |
| Diagnostics | explainable |

---

# 199. Test de pureza

```php
$user = UserEntityFactory::new()->make();
```

Assertions:

```text
Connection acquisitions = 0
Queries executed = 0
UoW registrations = 0
IdentityMap registrations = 0
```

---

# 200. Test persist()

```php
$user = UserEntityFactory::new()
    ->persist($entityManager);
```

Assertions:

```text
Entity constructed = YES
UoW registration = YES
Flush automatically = NO
Commit automatically = NO
```

---

# 201. Test create()

```php
$user = UserEntityFactory::new()
    ->create($entityManager);
```

Assertions:

```text
Entity constructed = YES
UoW registration = YES
Flush = according to FLUSH policy
Commit foreign transaction = NO
```

---

# 202. Test graph identity

```php
$user = UserEntityFactory::new()
    ->has(
        PostEntityFactory::new()
            ->count(3),
        'posts'
    )
    ->make();
```

Para cada post:

```php
assert($post->author() === $user);
```

si la relación bidireccional está configurada de esa manera.

---

# 203. Test no IdentityMap

Después de:

```php
$user = UserEntityFactory::new()->make();
```

debe cumplirse:

```text
IdentityMap.contains(user) = false
```

salvo que el caller lo haya registrado explícitamente mediante otra operación.

---

# 204. Test rollback

```php
DB::transaction(function () use ($entityManager) {
    $user = UserEntityFactory::new()
        ->create($entityManager);

    throw new RuntimeException();
});
```

La DB deberá hacer rollback según Transaction System.

Pero:

```text
$user PHP object may still exist
```

---

# 205. Test concurrency

```text
Coroutine A
UserEntityFactory seed=100

Coroutine B
UserEntityFactory seed=200
```

No deberá existir:

```text
random-state collision
sequence collision
graph-registry collision
```

---

# 206. Architectural invariants

## DB-EFAC-001
EntityFactory será una especialización del Shared Factory Engine.

## DB-EFAC-002
EntityFactory estará orientada al patrón Data Mapper.

## DB-EFAC-003
EntityFactory no será EntityManager.

## DB-EFAC-004
EntityFactory no será UnitOfWork.

## DB-EFAC-005
EntityFactory no será Persistence Engine.

## DB-EFAC-006
EntityFactory no será Query Builder.

## DB-EFAC-007
EntityFactory no será SQL Compiler.

## DB-EFAC-008
EntityFactory no será Query Executor.

## DB-EFAC-009
EntityFactory no accederá directamente al Driver.

## DB-EFAC-010
EntityFactory no administrará Connections.

## DB-EFAC-011
`make()` construirá entidades.

## DB-EFAC-012
`make()` no persistirá entidades.

## DB-EFAC-013
`make()` no ejecutará queries por defecto.

## DB-EFAC-014
`make()` no requerirá DB por defecto.

## DB-EFAC-015
`make()` no registrará UoW automáticamente.

## DB-EFAC-016
`make()` no registrará IdentityMap automáticamente.

## DB-EFAC-017
Entidad construida no implicará entidad managed.

## DB-EFAC-018
Entidad con ID no implicará row existente.

## DB-EFAC-019
`persist()` registrará persistence intent.

## DB-EFAC-020
`persist()` no implicará INSERT inmediato.

## DB-EFAC-021
`persist()` no implicará flush.

## DB-EFAC-022
`persist()` no implicará commit.

## DB-EFAC-023
`create()` podrá incluir flush según policy.

## DB-EFAC-024
`create()` no convertirá flush en commit.

## DB-EFAC-025
Factory no hará commit de transaction ajena.

## DB-EFAC-026
Factory no hará rollback de transaction ajena salvo contrato explícito de ownership.

## DB-EFAC-027
EntityManager seguirá siendo autoridad de gestión de entidades.

## DB-EFAC-028
UnitOfWork seguirá siendo autoridad de change tracking.

## DB-EFAC-029
Persistence Planner seguirá siendo autoridad de ordering.

## DB-EFAC-030
Factory no generará SQL.

## DB-EFAC-031
Factory attributes utilizarán domain/application representation.

## DB-EFAC-032
Factory no convertirá Value Objects prematuramente a DB representation.

## DB-EFAC-033
Type Conversion seguirá siendo responsabilidad del Type System.

## DB-EFAC-034
Entity constructors serán respetados.

## DB-EFAC-035
Private constructors no serán violados automáticamente.

## DB-EFAC-036
Domain factory methods podrán utilizarse como construction strategy.

## DB-EFAC-037
Reflection privada no será strategy predeterminada.

## DB-EFAC-038
Domain invariants seguirán siendo autoridad.

## DB-EFAC-039
States serán composables.

## DB-EFAC-040
States no representarán necesariamente mutation history.

## DB-EFAC-041
Sequences serán execution-scoped.

## DB-EFAC-042
Randomness será execution-scoped.

## DB-EFAC-043
Unique pools serán execution-scoped.

## DB-EFAC-044
Deterministic seeds producirán resultados reproducibles cuando providers lo permitan.

## DB-EFAC-045
FactoryClock será injectable.

## DB-EFAC-046
Pure EntityFactory será preferida.

## DB-EFAC-047
DB-aware generation será explícita.

## DB-EFAC-048
External I/O será explícito.

## DB-EFAC-049
Relationships utilizarán ORM metadata.

## DB-EFAC-050
Factory no inferirá relaciones únicamente por naming.

## DB-EFAC-051
Factory Graph será distinto de ORM Persistence Graph.

## DB-EFAC-052
Factory Graph será distinto de IdentityMap.

## DB-EFAC-053
Factory Graph será distinto de UnitOfWork.

## DB-EFAC-054
Factory Graph nodes tendrán identidad de construcción propia.

## DB-EFAC-055
Factory Graph evitará ciclos infinitos.

## DB-EFAC-056
Bidirectional relations reutilizarán el mismo graph node.

## DB-EFAC-057
Factory graph dedup no sustituirá persistent identity.

## DB-EFAC-058
Aggregate construction podrá ejecutarse sin DB.

## DB-EFAC-059
Factory podrá utilizar domain methods para enlazar aggregates.

## DB-EFAC-060
ORM cascade no será fabricado por Factory.

## DB-EFAC-061
Persistence Bridge respetará cascade metadata.

## DB-EFAC-062
Many-to-many reutilizará Relationship System.

## DB-EFAC-063
Rich associations utilizarán association entities cuando corresponda.

## DB-EFAC-064
Polymorphic relationships usarán canonical aliases.

## DB-EFAC-065
Persisted FQCN no será generado automáticamente.

## DB-EFAC-066
Generated IDs no serán prueba de persistence.

## DB-EFAC-067
DB-generated IDs tendrán strategy explícita.

## DB-EFAC-068
Factory no predecirá auto-increment IDs.

## DB-EFAC-069
Existing related entities no serán asumidas managed.

## DB-EFAC-070
Cross-EntityManager incompatibilities no serán ignoradas.

## DB-EFAC-071
Callbacks tendrán fases explícitas.

## DB-EFAC-072
`afterPersisting` será distinto de `afterFlush`.

## DB-EFAC-073
`afterFlush` será distinto de `afterCommit`.

## DB-EFAC-074
Callback failure no implicará rollback confirmado.

## DB-EFAC-075
External side effects no obtendrán atomicidad por callback.

## DB-EFAC-076
Outbox podrá utilizarse para efectos transaccionales.

## DB-EFAC-077
Flush scope deberá ser explícito.

## DB-EFAC-078
Factory no hará `EntityManager::clear()` ocultamente.

## DB-EFAC-079
Batch generation tendrá resource budgets.

## DB-EFAC-080
Streaming no acumulará necesariamente toda la colección.

## DB-EFAC-081
Bulk Insert será distinto de EntityFactory.

## DB-EFAC-082
EntityFactory conservará domain semantics.

## DB-EFAC-083
EntityFactory conservará ORM semantics.

## DB-EFAC-084
Invalid entity generation no violará invariantes por defecto.

## DB-EFAC-085
Database constraints seguirán siendo autoridad de integridad persistente.

## DB-EFAC-086
Factory uniqueness será distinta de DB uniqueness.

## DB-EFAC-087
Persistence failures conservarán exception semantics.

## DB-EFAC-088
UNKNOWN outcome permanecerá UNKNOWN.

## DB-EFAC-089
Factory no fabricará successful persistence.

## DB-EFAC-090
Rollback no rebobinará automáticamente object graph.

## DB-EFAC-091
Factory no reparará UoW silenciosamente después de unknown outcome.

## DB-EFAC-092
Sharding será una integración posterior a construcción.

## DB-EFAC-093
EntityFactory no será ShardRouter.

## DB-EFAC-094
EntityFactory no elegirá replica.

## DB-EFAC-095
EntityFactory no elegirá writer endpoint.

## DB-EFAC-096
EntityFactory no elegirá Connection.

## DB-EFAC-097
Cross-shard graph no implicará atomicidad global.

## DB-EFAC-098
Multitenancy será opcional.

## DB-EFAC-099
EntityFactory core no dependerá de tenant package.

## DB-EFAC-100
Cross-tenant relations serán validadas cuando la integración esté activa.

## DB-EFAC-101
EntityFactory no insertará directamente Entity Cache.

## DB-EFAC-102
EntityFactory no insertará directamente Result Cache.

## DB-EFAC-103
EntityFactory no insertará directamente Query Cache.

## DB-EFAC-104
Synthetic entity no será persistent cache truth.

## DB-EFAC-105
Hydration será distinta de Factory construction.

## DB-EFAC-106
Hydrator reconstruirá datos observados de DB.

## DB-EFAC-107
Factory construirá datos sintéticos.

## DB-EFAC-108
Compiled FactoryBlueprint podrá compartirse solo si es immutable.

## DB-EFAC-109
Factory runtime state será scoped.

## DB-EFAC-110
Generated entities no serán worker-global.

## DB-EFAC-111
Random state no será worker-global.

## DB-EFAC-112
Sequence state no será worker-global.

## DB-EFAC-113
Graph registry no será worker-global.

## DB-EFAC-114
Tenant context no será worker-global.

## DB-EFAC-115
Transaction references no serán worker-global.

## DB-EFAC-116
Arquitectura será segura para FrankenPHP.

## DB-EFAC-117
Arquitectura permitirá RoadRunner.

## DB-EFAC-118
Arquitectura permitirá OpenSwoole.

## DB-EFAC-119
Concurrent factory executions estarán aisladas.

## DB-EFAC-120
Worker reset eliminará scoped state residual.

## DB-EFAC-121
Telemetry evitará PII.

## DB-EFAC-122
Telemetry evitará raw generated values.

## DB-EFAC-123
Metrics tendrán bounded cardinality.

## DB-EFAC-124
Diagnostics serán explainable.

## DB-EFAC-125
Diagnostics no expondrán secrets por defecto.

## DB-EFAC-126
Factory errors conservarán causas originales.

## DB-EFAC-127
Constructor errors incluirán attribute path.

## DB-EFAC-128
Relationship errors incluirán graph path.

## DB-EFAC-129
Graph budgets limitarán depth.

## DB-EFAC-130
Graph budgets limitarán nodes.

## DB-EFAC-131
Graph budgets limitarán relationship expansion.

## DB-EFAC-132
Factory availability no implicará authorization.

## DB-EFAC-133
Factories no se expondrán remotamente por defecto.

## DB-EFAC-134
Factory definitions serán versionables.

## DB-EFAC-135
Compiled definitions serán generation-aware.

## DB-EFAC-136
ORM metadata generation podrá participar en fingerprint.

## DB-EFAC-137
Relationship metadata generation podrá participar en fingerprint.

## DB-EFAC-138
Type registry generation podrá participar en fingerprint.

## DB-EFAC-139
EntityFactory podrá funcionar sin Model API.

## DB-EFAC-140
EntityFactory podrá funcionar sin Active Record API.

## DB-EFAC-141
EntityFactory podrá funcionar sin HTTP Kernel.

## DB-EFAC-142
EntityFactory podrá funcionar sin Cache provider.

## DB-EFAC-143
EntityFactory podrá funcionar sin Multitenancy.

## DB-EFAC-144
EntityFactory podrá funcionar sin sharding.

## DB-EFAC-145
EntityFactory podrá funcionar sin active transaction.

## DB-EFAC-146
EntityFactory podrá utilizarse en unit tests.

## DB-EFAC-147
EntityFactory podrá utilizarse en integration tests.

## DB-EFAC-148
EntityFactory podrá utilizarse desde Seeders.

## DB-EFAC-149
EntityFactory podrá utilizarse desde Fixtures.

## DB-EFAC-150
EntityFactory podrá utilizarse desde CLI tooling.

## DB-EFAC-151
ModelFactory y EntityFactory compartirán Factory Engine.

## DB-EFAC-152
ModelFactory y EntityFactory conservarán APIs distintas donde sus paradigmas difieran.

## DB-EFAC-153
No se forzará semántica Active Record sobre Data Mapper.

## DB-EFAC-154
No se forzará EntityManager explícito sobre ModelFactory DX cuando no sea necesario.

## DB-EFAC-155
Ambas APIs convergerán al mismo ORM Persistence Engine.

## DB-EFAC-156
No existirán dos UnitOfWork independientes por estilo de Factory.

## DB-EFAC-157
No existirán dos IdentityMaps independientes por estilo de Factory.

## DB-EFAC-158
No existirán dos Relationship Engines independientes.

## DB-EFAC-159
No existirán dos Type Systems independientes.

## DB-EFAC-160
No existirán dos Persistence Engines independientes.

---

# 207. Modelo formal

Sea:

```text
E = Entity Type
F = EntityFactory
A = resolved attributes
S = states
R = relationship graph
C = FactoryContext
```

Entonces:

```text
EntityGraph =
Construct(E, F, A, S, R, C)
```

Para:

```text
make()
```

tenemos:

```text
make(F)
=
EntityGraph
```

Para:

```text
persist(F, EM)
```

tenemos:

```text
persist(F, EM)
=
Register(
    EM,
    make(F)
)
```

Y para:

```text
create(F, EM)
```

conceptualmente:

```text
create(F, EM)
=
Flush(
    Register(
        EM,
        make(F)
    )
)
```

Pero:

```text
Flush
≠
Commit
```

por lo que:

```text
create()
≠
TransactionCommit()
```

---

# 208. Modelo de identidad

Durante Factory:

```text
FactoryNodeId
→ PHP Object
```

Durante ORM management:

```text
EntityKey
→ canonical managed PHP Object
```

Por tanto:

```text
FactoryNodeId
≠
EntityKey
```

aunque eventualmente ambos puedan referirse al mismo objeto.

---

# 209. Modelo de estado

```text
                EntityFactory
                     │
                   make()
                     │
                     ▼
                 TRANSIENT
                     │
             EM::persist()
                     │
                     ▼
                    NEW
                     │
                   flush
                     │
                     ▼
              SYNCHRONIZED?
                     │
                   commit
                     │
                     ▼
                COMMITTED?
```

Los nombres exactos dependerán del `Entity State System`, pero la separación semántica deberá mantenerse.

---

# 210. Modelo de grafo

```text
Factory Definition
       │
       ▼
Factory Graph Plan
       │
       ▼
Entity Construction
       │
       ▼
Entity Object Graph
       │
       ├──────────────► application/test usage
       │
       └──────────────► EntityManager::persist()
                              │
                              ▼
                           UnitOfWork
                              │
                              ▼
                       Persistence Graph
```

La regla es:

```text
Factory Graph
≠
Persistence Graph
```

---

# 211. Pipeline completo

```text
UserEntityFactory::new()
          │
          ▼
   EntityFactory Definition
          │
          ├── states
          ├── overrides
          ├── sequences
          ├── relationships
          ├── clock
          └── random source
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
    Entity Construction
          │
          ▼
   Relationship Assembly
          │
          ▼
      Entity Graph
          │
       ┌──┴─────────────┐
       │                │
       ▼                ▼
     make()          persist(EM)
       │                │
       │                ▼
       │         EntityManager::persist
       │                │
       │                ▼
       │             UnitOfWork
       │
       └────────────────────────────┐
                                    │
                               optional
                                 create
                                    │
                                    ▼
                                  flush
                                    │
                                    ▼
                            Persistence Engine
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

# 212. Integración ModelFactory + EntityFactory

Arquitectura final:

```text
                         Factory Engine
                              │
                ┌─────────────┴─────────────┐
                │                           │
                ▼                           ▼
          ModelFactory                EntityFactory
                │                           │
       Laravel-like DX                 Data Mapper DX
                │                           │
                ▼                           ▼
        Model API Bridge             EntityManager API
                │                           │
                └─────────────┬─────────────┘
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
```

---

# 213. Regla maestra final

> **EntityFactory genera entidades y grafos de dominio; EntityManager decide su participación en el contexto ORM; UnitOfWork registra sus cambios; Persistence Engine decide cómo sincronizarlos; Query Engine produce las operaciones de consulta y Database Execution las ejecuta. Ninguna comodidad ofrecida por EntityFactory podrá borrar esas fronteras.**

En forma resumida:

```text
EntityFactory
     ↓
Entity Graph
     ↓
optional EntityManager registration
     ↓
UnitOfWork
     ↓
Persistence Engine
     ↓
Query Engine
     ↓
Database
```

Nunca:

```text
EntityFactory
     ↓
SQL
```

---

# 214. Resultado arquitectónico

Con los documentos `193`, `194` y `195`, VoltStack dispone de:

```text
                 DATABASE FACTORY SYSTEM
                          193
                           │
                Shared Factory Engine
                           │
             ┌─────────────┴─────────────┐
             │                           │
             ▼                           ▼
       MODEL FACTORY                ENTITY FACTORY
            194                          195
             │                           │
     Active Record-like               Data Mapper
             │                           │
             └─────────────┬─────────────┘
                           ▼
                      Canonical ORM
                           │
                           ▼
                      Persistence
```

Esto permite que VoltStack soporte dos experiencias:

```php
User::factory()
    ->verified()
    ->create();
```

y:

```php
$user = UserEntityFactory::new()
    ->verified()
    ->make();

$entityManager->persist($user);
$entityManager->flush();
```

sin duplicar la arquitectura interna.

La separación definitiva queda:

```text
Factory
    generates

EntityManager
    manages

UnitOfWork
    tracks

Persistence Engine
    synchronizes

Query Engine
    represents/plans queries

Compiler
    generates SQL

Executor
    executes

Driver
    communicates with DB
```

---

# 215. Siguiente documento

```text
196_DATABASE_SEEDER_SYSTEM.md
```

El siguiente documento definirá el sistema responsable de **orquestar datasets intencionales dentro de una base de datos**:

```text
Seeder
   │
   ├── ModelFactory
   ├── EntityFactory
   ├── explicit records
   ├── dependencies
   ├── environments
   ├── execution ordering
   └── persistence boundaries
```

manteniendo la separación:

```text
Factory
=
cómo generar una instancia

Seeder
=
qué conjunto de datos debe poblarse

Fixture
=
qué escenario reproducible necesita una prueba
```

y, especialmente:

```text
Seeder
≠
Factory
≠
Migration
≠
Fixture
≠
Persistence Engine
```