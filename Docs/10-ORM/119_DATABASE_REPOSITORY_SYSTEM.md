# 119_DATABASE_REPOSITORY_SYSTEM.md

# VoltStack Quantum Database
## Database Repository System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 119 — Database Repository System  
**Bloque:** 10 — ORM  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Repository System` define la arquitectura mediante la cual VoltStack proporciona una interfaz orientada a entidades para consultar, localizar y trabajar con conjuntos lógicos de objetos persistentes.

Ejemplo:

```php
$users = $entityManager->repository(User::class);

$user = $users->find($userId);

$activeUsers = $users->findBy([
    'active' => true,
]);
```

Un Repository representa conceptualmente:

```text
Repository<User>
≈
logical collection of User entities
```

pero no es una colección materializada en memoria.

Principio central:

> **Un Repository representa una puerta de acceso orientada a entidades sobre el sistema de consultas y el Persistence Context; no genera SQL, no sustituye al EntityManager y no debe convertirse en un servicio de negocio.**

---

# 2. Posición arquitectónica

```text
Application
    │
    ▼
Repository
    │
    ├──────────────► EntityManager
    │
    ├──────────────► Entity Metadata
    │
    └──────────────► Entity Query System
                          │
                          ▼
                     Query Engine
                          │
                          ▼
                  Execution Engine
                          │
                          ▼
                       Database
```

Para resultados administrados:

```text
Database Result
      ↓
Hydration
      ↓
IdentityMap
      ↓
Entity
```

---

# 3. Repository ≠ EntityManager

El Repository está especializado en un:

```text
EntityType
```

El EntityManager coordina todo el:

```text
PersistenceContext
```

Por tanto:

```text
Repository<User>
    ↓
User queries
```

mientras:

```text
EntityManager
    ↓
all managed entity types
```

---

# 4. Repository ≠ Query Builder

Un Repository puede iniciar consultas:

```php
$users->query()
    ->where('status', UserStatus::ACTIVE)
    ->get();
```

pero no implementa internamente todo el Query Builder.

La ruta correcta será:

```text
Repository
    ↓
Entity Query Builder
    ↓
Entity Query Translation
    ↓
Query Model / AST
```

---

# 5. Repository ≠ SQL Builder

Nunca:

```php
class UserRepository
{
    public function findActive(): array
    {
        return $this->sql(
            'SELECT * FROM users WHERE active = 1'
        );
    }
}
```

como arquitectura estándar del Repository System.

Raw SQL seguirá siendo un escape hatch explícito del Database System.

---

# 6. Repository ≠ Persistence Engine

El Repository no ejecuta:

```text
INSERT
UPDATE
DELETE
```

sobre entidades por su cuenta.

Para persistencia:

```text
Repository convenience
    ↓
EntityManager
    ↓
UnitOfWork
    ↓
Persistence Engine
```

---

# 7. Repository ≠ UnitOfWork

El Repository no deberá mantener:

```text
dirty entities
snapshots
pending inserts
pending updates
pending deletes
```

Eso pertenece al UnitOfWork.

---

# 8. Repository ≠ IdentityMap

El Repository puede beneficiarse del IdentityMap mediante el EntityManager.

Pero:

```text
Repository
≠
IdentityMap
```

---

# 9. Repository ≠ Business Service

Este es uno de los límites más importantes.

Incorrecto:

```php
final class UserRepository
{
    public function registerUserAndSendWelcomeEmail(
        RegisterUserCommand $command
    ): void {
        // business workflow
    }
}
```

El Repository debe concentrarse en:

```text
finding
querying
loading
persistence-oriented collection access
```

La orquestación de negocio pertenece a:

```text
Application Service
Domain Service
Use Case
Action
```

---

# 10. Repository ≠ DAO tradicional

Aunque ambos abstraen acceso a datos, el Repository de VoltStack estará orientado a:

```text
Entity Model
```

y no a:

```text
tables
rows
SQL statements
```

Por ejemplo:

```php
$users->findByEmail($email);
```

es una consulta de entidades.

No:

```php
$users->fetchUsersTableRows();
```

---

# 11. Repository ≠ Active Record

VoltStack soportará dos experiencias:

```text
Laravel-like Model API
```

y:

```text
Repository / EntityManager API
```

pero ambas utilizarán el mismo ORM.

```text
User::find(10)
          │
          ▼
    Model API
          │
          ├────────────┐
          │            │
          ▼            ▼
 EntityManager     Repository
          │            │
          └──────┬─────┘
                 ▼
             ORM Core
```

No existirán dos motores.

---

# 12. Concepto de colección lógica

Un Repository puede verse como:

```text
Repository<T>
=
LogicalPersistentCollection<T>
```

pero:

```text
LogicalPersistentCollection
≠
LoadedCollection
```

Por ejemplo:

```php
$users = $em->repository(User::class);
```

no carga todos los usuarios.

---

# 13. Repository type

Cada Repository estará asociado a exactamente un:

```text
EntityType
```

Ejemplo:

```text
UserRepository
→ EntityType(User)
```

---

# 14. EntityType ≠ PHP class

De acuerdo con el Entity Model:

```text
EntityType
≠
EntityClass
```

aunque normalmente exista una relación:

```text
EntityType(User)
→ App\Domain\User
```

---

# 15. EntityType ≠ table

También:

```text
UserRepository
≠
users table repository
```

El Repository trabaja con:

```text
User entity
```

aunque su mapping físico cambie.

---

# 16. RepositoryDescriptor

Se propone:

```php
final readonly class RepositoryDescriptor
{
    public function __construct(
        public EntityType $entityType,
        public string $repositoryClass,
        public RepositoryKind $kind,
        public RepositoryCapabilities $capabilities,
    ) {}
}
```

---

# 17. RepositoryKind

Podrá existir:

```text
DEFAULT
CUSTOM
GENERATED
EXTENSION
```

---

# 18. Repository metadata

La asociación:

```text
EntityType
→ Repository
```

podrá provenir de:

```text
EntityMetadata
```

Ejemplo conceptual:

```php
#[Entity(
    repository: UserRepository::class
)]
final class User
{
}
```

---

# 19. Mapping ≠ repository implementation

La metadata solamente declara:

```text
repository class
```

No contiene la lógica del Repository.

---

# 20. Contrato base

Se propone conceptualmente:

```php
/**
 * @template TEntity of object
 */
interface EntityRepository
{
    /**
     * @return TEntity|null
     */
    public function find(mixed $identifier): ?object;

    /**
     * @return list<TEntity>
     */
    public function findAll(): array;

    /**
     * @return list<TEntity>
     */
    public function findBy(
        array $criteria,
        ?array $orderBy = null,
        ?int $limit = null,
        ?int $offset = null,
    ): array;

    /**
     * @return TEntity|null
     */
    public function findOneBy(array $criteria): ?object;

    /**
     * @return EntityQuery<TEntity>
     */
    public function query(): EntityQuery;
}
```

---

# 21. Generic typing

La API deberá diseñarse para herramientas estáticas como:

```text
PHPStan
Psalm
IDE inference
```

Ejemplo:

```php
/** @extends DefaultEntityRepository<User> */
final class UserRepository extends DefaultEntityRepository
{
}
```

permitiendo inferir:

```php
$user = $users->find($id);
```

como:

```text
User|null
```

---

# 22. DefaultEntityRepository

VoltStack deberá proporcionar:

```text
DefaultEntityRepository<TEntity>
```

para evitar que cada entidad requiera un Repository manual.

---

# 23. Ejemplo

```php
$users = $entityManager->repository(User::class);

$user = $users->find($id);
```

aunque no exista:

```text
UserRepository.php
```

---

# 24. Default repository responsibilities

Podrá ofrecer:

```text
find
findAll
findBy
findOneBy
count
exists
query
```

sin incluir cientos de métodos mágicos.

---

# 25. API mínima

Se recomienda mantener el Repository base relativamente pequeño.

Un Repository con:

```text
150 convenience methods
```

terminaría duplicando Entity Query.

---

# 26. `find()`

```php
$user = $repository->find($userId);
```

deberá delegar a:

```text
EntityManager.find(EntityType, Identifier)
```

---

# 27. Pipeline de `find()`

```text
Repository::find()
       ↓
EntityManager::find()
       ↓
Normalize Identifier
       ↓
EntityKey
       ↓
IdentityMap
       │
       ├── hit → managed entity
       │
       └── miss
              ↓
          EntityLoader
              ↓
          Entity Query
              ↓
          Query Engine
```

---

# 28. No duplicate find implementation

El Repository no deberá implementar un segundo algoritmo de identity lookup.

Incorrecto:

```text
EntityManager.find()
algorithm A

Repository.find()
algorithm B
```

Correcto:

```text
Repository.find()
→ EntityManager.find()
```

---

# 29. `findAll()`

Conceptualmente:

```php
$users = $repository->findAll();
```

equivale a una Entity Query sin predicates.

---

# 30. `findAll()` risk

En tablas grandes:

```text
findAll()
```

puede cargar enormes cantidades de entidades.

Por ello DX/Telemetry podrá advertir sobre su uso cuando:

```text
estimated cardinality
```

sea elevada.

---

# 31. `findBy()`

Ejemplo:

```php
$users = $repository->findBy(
    criteria: [
        'status' => UserStatus::ACTIVE,
    ],
    orderBy: [
        'createdAt' => 'DESC',
    ],
    limit: 50,
);
```

---

# 32. Criteria semantics

Los criterios deberán usar:

```text
Entity fields
```

no nombres físicos de columnas.

Ejemplo:

```php
[
    'createdAt' => $date
]
```

aunque la columna sea:

```text
created_at
```

---

# 33. Criteria translation

```text
Repository Criteria
       ↓
Entity Metadata
       ↓
Entity Query
       ↓
Query AST
```

---

# 34. Criteria ≠ raw SQL

No deberán aceptarse strings ambiguos como:

```php
[
    'condition' => 'age > 18'
]
```

dentro del criteria simple.

Para expresiones avanzadas:

```text
Entity Query API
```

---

# 35. `findOneBy()`

```php
$user = $repository->findOneBy([
    'email' => $email,
]);
```

deberá producir una consulta limitada apropiadamente.

---

# 36. Non-unique result

Si la semántica exige unicidad y aparecen múltiples resultados, podrán existir políticas:

```text
FIRST
ERROR_ON_MULTIPLE
```

---

# 37. Recommended default

Para `findOneBy()` se recomienda:

```text
LIMIT 1 semantics
```

sin afirmar unicidad lógica.

Para una consulta que exige unicidad deberá existir:

```text
one()
exactlyOne()
single()
```

en Entity Query.

---

# 38. `count()`

Podrá ofrecerse:

```php
$count = $repository->count([
    'active' => true,
]);
```

pero deberá generar:

```text
aggregate Entity Query
```

No cargar entidades y contar en PHP.

---

# 39. `exists()`

```php
if ($repository->exists([
    'email' => $email,
])) {
}
```

deberá utilizar una consulta de existencia eficiente.

No:

```php
count(...) > 0
```

obligatoriamente si el Query Engine puede representar mejor `EXISTS`.

---

# 40. Repository Query

La operación más importante para consultas avanzadas será:

```php
$query = $repository->query();
```

---

# 41. EntityQuery

```text
Repository
    ↓
EntityQuery<TEntity>
```

deberá ser entity-aware.

Ejemplo:

```php
$users
    ->query()
    ->where('status', UserStatus::ACTIVE)
    ->where('profile.country', 'MX')
    ->orderBy('createdAt', 'DESC')
    ->limit(50)
    ->get();
```

---

# 42. Entity Query ≠ SQL Query Builder

Entity Query conoce:

```text
entity fields
relationships
embedded objects
entity types
mapping metadata
```

Después traduce hacia:

```text
Database Query Model / AST
```

---

# 43. Repository no compila SQL

La ruta será:

```text
Repository
    ↓
Entity Query
    ↓
Entity Semantic Resolution
    ↓
Database Query AST
    ↓
Optimizer
    ↓
Planner
    ↓
SQL Compiler
```

---

# 44. Custom Repository

El desarrollador podrá crear:

```php
final class UserRepository extends DefaultEntityRepository
{
    public function findByEmail(
        EmailAddress $email,
    ): ?User {
        return $this
            ->query()
            ->where('email', $email)
            ->oneOrNull();
    }
}
```

---

# 45. Domain-specific query methods

Son apropiados:

```text
findByEmail()
findActiveUsers()
findPendingOrders()
findExpiredSubscriptions()
existsByExternalId()
```

porque expresan consultas sobre el conjunto de entidades.

---

# 46. Business workflows inappropriate

No deberían residir aquí:

```text
registerCustomer()
chargeSubscription()
sendInvoice()
approveLoan()
notifyAdministrator()
```

---

# 47. Regla práctica

Pregunta:

> ¿El método describe cómo seleccionar o persistir una colección de entidades, o describe un caso de uso de negocio?

Si es un caso de uso:

```text
Application Service
```

es probablemente la capa correcta.

---

# 48. Repository inheritance

Se recomienda:

```text
DefaultEntityRepository<TEntity>
       ↑
UserRepository
```

pero no exigir herencia.

---

# 49. Composition support

También deberá permitirse:

```php
final class UserRepository implements EntityRepository
{
    public function __construct(
        private EntityRepository $base,
    ) {}
}
```

---

# 50. RepositoryFactory

La creación de repositories se delegará a:

```text
RepositoryFactory
```

---

# 51. Contract

```php
interface RepositoryFactory
{
    public function create(
        RepositoryDescriptor $descriptor,
        EntityManager $entityManager,
        EntityMetadata $metadata,
    ): EntityRepository;
}
```

---

# 52. RepositoryResolver

El EntityManager utilizará:

```text
RepositoryResolver
```

para:

```text
EntityType
→ RepositoryDescriptor
→ RepositoryFactory
→ Repository
```

---

# 53. Pipeline

```text
EntityManager.repository(User)
        ↓
Resolve EntityType
        ↓
Metadata Registry
        ↓
Repository Descriptor
        ↓
Repository Cache
        │
        ├── hit
        │    ↓
        │ Repository
        │
        └── miss
             ↓
       Repository Factory
             ↓
       Repository Instance
```

---

# 54. RepositoryRegistry

Se propone un registro inmutable de definiciones:

```text
RepositoryRegistry
```

---

# 55. Registry contents

```text
EntityType
→ RepositoryDescriptor
```

---

# 56. Registry ≠ instance cache

Separar:

```text
RepositoryRegistry
```

de:

```text
RepositoryInstanceCache
```

El primero contiene definiciones.

El segundo contiene instancias runtime.

---

# 57. Shared vs scoped

Puede compartirse entre requests:

```text
RepositoryRegistry
RepositoryDescriptor
compiled repository metadata
```

si son inmutables.

Debe ser scoped:

```text
Repository instance holding EntityManager
RepositoryInstanceCache
```

---

# 58. Repository instance cache

Dentro de un EntityManager:

```php
$a = $em->repository(User::class);
$b = $em->repository(User::class);
```

podrá cumplir:

```php
$a === $b;
```

---

# 59. Cache key

El key lógico deberá ser al menos:

```text
EntityType
```

dentro del manager.

No necesita incluir tenant si el cache ya está aislado por EntityManager.

---

# 60. Cross-manager cache forbidden

Nunca:

```text
static Repository<User>
```

con manager mutable.

---

# 61. Repository constructor dependencies

Un Repository personalizado no debería recibir todo el container.

Preferible:

```php
final class UserRepository
{
    public function __construct(
        EntityManager $entityManager,
        EntityMetadata $metadata,
    ) {}
}
```

o contratos aún menores.

---

# 62. Narrow dependencies

Ejemplo:

```php
public function __construct(
    EntityQueryFactory $queries,
    EntityFinder $finder,
) {}
```

puede ser mejor si no necesita todo el EntityManager.

---

# 63. Repository Service Locator anti-pattern

Incorrecto:

```php
$this->container->get(...);
```

para dependencias arbitrarias.

---

# 64. Dependency injection

Custom repositories podrán utilizar DI para servicios legítimos relacionados con consultas.

Ejemplo:

```text
Clock
QuerySpecificationFactory
Domain-specific read policy
```

pero deberá evitarse convertir el Repository en Application Service.

---

# 65. RepositoryFactory + Container

La factory podrá integrar el Service Container para construir custom repositories.

Pipeline:

```text
RepositoryDescriptor
      ↓
RepositoryFactory
      ↓
Container-assisted construction
      ↓
Scoped Repository
```

---

# 66. Scope safety

Si el container construye repositories, deberá respetar:

```text
EntityManager scope
```

No registrar accidentalmente un custom repository como singleton.

---

# 67. Repository scope classification

```text
Repository Definition → singleton-safe if immutable
Repository Instance   → scoped if manager-bound
```

---

# 68. Repository and IdentityMap

Consultas que retornan entidades managed deberán integrarse con:

```text
IdentityMap
```

---

# 69. Example

Si:

```php
$userA = $users->find(10);

$userB = $users
    ->query()
    ->where('id', 10)
    ->one();
```

entonces normalmente:

```php
$userA === $userB;
```

dentro del mismo PersistenceContext.

---

# 70. Query does not overwrite dirty entity

Supongamos:

```php
$user = $users->find(10);

$user->rename('Local Change');

$sameUser = $users
    ->query()
    ->where('id', 10)
    ->one();
```

La segunda query no deberá sobrescribir silenciosamente:

```text
Local Change
```

con el valor almacenado en DB.

---

# 71. Refresh is explicit

Para reemplazar estado managed con estado DB:

```php
$em->refresh($user);
```

con la política correspondiente.

---

# 72. Repository and UnitOfWork

El Repository no manipula directamente ChangeSets.

Pero los resultados managed deberán registrarse correctamente mediante Hydration/UoW.

---

# 73. Save convenience

¿Debe Repository incluir?

```php
$users->save($user);
```

Puede ofrecerse como conveniencia.

Pero deberá ser exactamente:

```text
Repository.save(entity)
    ↓
EntityManager.persist(entity)
```

y opcionalmente:

```text
flush
```

según contrato explícito.

---

# 74. Recommendation

Para la API Data Mapper base se recomienda separar:

```php
$em->persist($user);
$em->flush();
```

y no colocar `save()` como requisito de todo Repository.

---

# 75. Why

Esto mantiene:

```text
Repository = collection/query abstraction
EntityManager = persistence context coordinator
```

---

# 76. Model API difference

La API estilo Laravel sí puede ofrecer:

```php
$user->save();
```

porque es una conveniencia deliberada del Model API.

---

# 77. Delete convenience

De igual manera:

```php
$repository->delete($entity);
```

no será necesario en el contrato mínimo.

Preferir:

```php
$em->remove($entity);
```

---

# 78. Bulk operations

Los Repository podrán ofrecer APIs especializadas de bulk:

```php
$users->query()
    ->where(...)
    ->bulkUpdate(...);
```

pero deben respetar las reglas de managed state.

---

# 79. Bulk bypass

Bulk update/delete puede evitar:

```text
Hydration
UnitOfWork per entity
Lifecycle per entity
```

---

# 80. Managed-state policy

Por ello deberá declarar:

```text
CLEAR_AFFECTED
REFRESH_AFFECTED
REJECT_IF_MANAGED
ALLOW_STALE_EXPLICITLY
```

---

# 81. No silent stale state

Nunca:

```text
bulk update DB
+
leave conflicting managed entities silently stale
```

---

# 82. Query specifications

VoltStack podrá soportar:

```text
Specification
```

como objeto reutilizable de criterios.

Ejemplo:

```php
$users->matching(
    ActiveUsers::inCountry($country)
);
```

---

# 83. Specification ≠ Repository

Una Specification describe:

```text
query predicate / selection intent
```

No administra entidades.

---

# 84. EntitySpecification

Posible contrato:

```php
interface EntitySpecification
{
    public function apply(
        EntityQuery $query,
    ): EntityQuery;
}
```

---

# 85. Specification composability

Podrá soportarse:

```text
AND
OR
NOT
```

si se preserva tipado y semántica.

---

# 86. No SQL in Specification

Incorrecto:

```php
return 'users.active = 1';
```

Correcto:

```text
typed Entity Query predicates
```

---

# 87. Criteria objects

Además de arrays:

```php
$repository->findBy([
    'status' => $status,
]);
```

podrá existir:

```text
EntityCriteria
```

---

# 88. EntityCriteria

Ejemplo conceptual:

```php
$criteria = EntityCriteria::create()
    ->where('status', EqualTo::value($status))
    ->orderBy('createdAt', Direction::DESC)
    ->limit(50);
```

---

# 89. Simple criteria vs EntityQuery

Jerarquía de ergonomía:

```text
find()
    ↓
findBy()
    ↓
EntityCriteria
    ↓
EntityQuery
```

De simple a avanzado.

---

# 90. Magic methods

Métodos tipo:

```php
findByEmailAndStatus(...)
```

generados dinámicamente mediante `__call()` no deberían ser la arquitectura central.

---

# 91. Why avoid magic

Problemas:

```text
weak IDE support
runtime errors
ambiguous parsing
poor refactoring
hidden behavior
harder static analysis
```

---

# 92. Code generation alternative

Si se desean métodos derivados, VoltStack podría ofrecer:

```text
compile-time/code-generation
```

en lugar de magia runtime.

---

# 93. Repository return shapes

Un Repository podrá producir:

```text
Entity
Entity Collection
Projection
Scalar
Tuple
Page
Cursor Page
Stream
```

según el método/query.

---

# 94. Entity result

```php
$user = $users->find($id);
```

usa Hydration + IdentityMap.

---

# 95. Projection result

```php
$items = $users
    ->query()
    ->select([
        'id',
        'name',
    ])
    ->as(UserSummary::class)
    ->get();
```

No deberá crear entidades parciales si no es necesario.

---

# 96. Projection preferred

Para consultas parciales:

```text
Projection
>
Partial Entity
```

como default arquitectónico.

---

# 97. Scalar query

```php
$count = $users
    ->query()
    ->count();
```

no debe hidratar entidades.

---

# 98. Streaming

Repository podrá iniciar:

```php
foreach ($users->query()->stream() as $user) {
}
```

---

# 99. Streaming modes

Integración con EntityManager:

```text
TRACKED
DETACHED_AFTER_YIELD
READ_ONLY
PROJECTION
SCALAR
```

---

# 100. Pagination

El Repository podrá delegar:

```php
$users
    ->query()
    ->paginate(...);
```

al futuro:

```text
DATABASE_PAGINATION_SYSTEM
DATABASE_CURSOR_PAGINATION_SYSTEM
```

---

# 101. Repository pagination ≠ separate paginator engine

Repository solamente inicia la Entity Query.

---

# 102. Relationship queries

Custom Repository podrá consultar relaciones:

```php
$orders
    ->query()
    ->where('customer.id', $customerId)
    ->where('items.product.category', $category)
    ->get();
```

---

# 103. Relationship path resolution

Será responsabilidad de:

```text
Entity Query
    ↓
Relationship Metadata
    ↓
Relation Resolution System
```

No del Repository directamente.

---

# 104. Join semantics

El Repository no deberá decidir:

```text
INNER JOIN
LEFT JOIN
join ordering
physical join strategy
```

salvo que la Entity Query exprese intención lógica explícita.

---

# 105. Eager loading

Podrá ofrecerse:

```php
$users
    ->query()
    ->with('profile')
    ->with('roles')
    ->get();
```

---

# 106. `with()` ≠ mandatory JOIN

La estrategia puede convertirse en:

```text
JOIN
SELECT IN
BATCH
SUBQUERY
```

según Relationship Loading System.

---

# 107. N+1

Repository no detecta N+1 por sí mismo.

La integración corresponderá a:

```text
Relationship Loading
+
Telemetry
+
N+1 Detection
```

---

# 108. Repository and lifecycle events

Cargar entidades mediante Repository deberá producir las mismas reglas de lifecycle que cualquier Entity Query equivalente.

---

# 109. Repository method must not alter semantics

```text
$repo->find(...)
```

no deberá comportarse de forma incompatible con:

```text
$em->find(...)
```

para la misma operación.

---

# 110. Soft delete integration

Una futura extensión podrá agregar:

```text
SoftDeleteScope
```

a queries del Repository.

---

# 111. Global scopes

Podrán existir scopes como:

```text
soft delete
tenant policy
application visibility
temporal query
```

pero deberán ser:

```text
typed
registered
inspectable
deterministic
explicitly disableable where policy permits
```

---

# 112. Hidden predicates warning

No deberán introducirse predicates invisibles imposibles de diagnosticar.

El Query Profiler deberá poder mostrar:

```text
Applied Entity Scopes
```

---

# 113. Tenant scopes

El core Repository no deberá asumir:

```text
tenant_id
```

---

# 114. Multitenancy integration

Cuando el paquete Multitenancy esté activo:

```text
TenantContext
    ↓
DatabaseContext
    ↓
EntityManager
    ↓
Repository
```

---

# 115. Repository tenant isolation

```text
Repository<User> Tenant A
≠
Repository<User> Tenant B
```

si están vinculados a managers distintos.

---

# 116. Tenant switching forbidden

Un Repository no deberá ofrecer:

```php
$repository->setTenant($tenant);
```

si esto cambia el PersistenceContext subyacente.

---

# 117. Correct tenant pattern

```php
$tenantAUsers = $tenantAEntityManager
    ->repository(User::class);

$tenantBUsers = $tenantBEntityManager
    ->repository(User::class);
```

---

# 118. Read/write routing

Las consultas del Repository declaran intención:

```text
READ
```

Las operaciones de persistencia siguen:

```text
EntityManager
→ WRITE
```

---

# 119. Repository does not select replica

No:

```php
$repository->useReplica(2);
```

como API ORM base.

Eso pertenece al routing system.

---

# 120. Consistency options

Entity Query podrá expresar requerimientos como:

```text
STRONG
READ_YOUR_WRITES
EVENTUAL
```

si el futuro Database Routing System los soporta.

Repository solo propaga esa intención.

---

# 121. Sharding

Repository tampoco deberá calcular:

```text
shard = hash(userId) % 16
```

El shard routing pertenece al sistema distribuido.

---

# 122. Cross-shard queries

Si una query requiere múltiples shards deberá ser una capacidad explícita.

No deberá ocurrir ocultamente desde un Repository normal.

---

# 123. Repository and cache

Repository no deberá implementar un cache paralelo arbitrario:

```php
private array $usersById;
```

para managed entities.

IdentityMap ya resuelve identidad local.

---

# 124. Query/result cache

La futura integración podrá ser:

```text
Repository
    ↓
Entity Query
    ↓
Query/Result Cache
```

---

# 125. Entity cache

Para entidades:

```text
IdentityMap
→ Entity Cache
→ Database
```

según configuración futura.

---

# 126. Repository cache semantics

Un método:

```php
findByEmail()
```

no deberá cachear resultados por su cuenta sin integrarse con el Cache System.

---

# 127. Security

Repository no es Authorization System.

```php
$user = $users->find($id);
```

no significa:

```text
actor is authorized to access user
```

---

# 128. Security scopes

Podrán existir integraciones que apliquen políticas de acceso a queries.

Pero deberán ser explícitas y auditables.

---

# 129. Authorization ≠ repository criteria

No convertir:

```text
Authorization
```

en simples filtros improvisados dentro de cada Repository.

---

# 130. Sensitive queries

Telemetry deberá poder redactar:

```text
email
tokens
credentials
personal identifiers
query parameters
```

según política.

---

# 131. Error architecture

Se propone:

```text
DatabaseOrmException
└── RepositoryException
    ├── RepositoryResolutionException
    ├── RepositoryNotFoundException
    ├── InvalidRepositoryException
    ├── RepositoryTypeMismatchException
    ├── RepositoryConstructionException
    ├── RepositoryScopeException
    ├── RepositoryQueryException
    ├── InvalidRepositoryCriteriaException
    ├── UnknownRepositoryFieldException
    ├── InvalidRepositorySpecificationException
    ├── NonUniqueRepositoryResultException
    ├── RepositoryResultTypeException
    ├── RepositoryEntityContextException
    ├── CrossContextRepositoryException
    ├── RepositoryExtensionConflictException
    └── RepositoryInvariantException
```

---

# 132. RepositoryTypeMismatchException

Ejemplo:

```text
Repository<User>
```

intentando trabajar como:

```text
Repository<Order>
```

deberá rechazarse.

---

# 133. Unknown field

```php
$users->findBy([
    'doesNotExist' => 123,
]);
```

deberá fallar durante resolución/validación.

No convertirse silenciosamente en columna SQL.

---

# 134. Invalid criteria value

Si:

```text
User.status
```

es:

```text
UserStatus
```

y recibe un tipo incompatible, el sistema deberá producir diagnóstico tipado.

---

# 135. Raw expressions

El Repository base no deberá permitir:

```php
findBy([
    'status' => Raw::sql(...)
]);
```

sin pasar por el escape hatch gobernado del Query System.

---

# 136. Repository extension system

Paquetes podrán registrar:

```text
repository decorators
repository factories
repository query extensions
repository specifications
```

mediante contratos explícitos.

---

# 137. Extension ≠ monkey patching

No se permitirá modificar métodos arbitrariamente durante runtime.

---

# 138. Extension registry

Podrá existir:

```text
RepositoryExtensionRegistry
```

inmutable después del bootstrap.

---

# 139. Collision handling

Si dos paquetes registran un Repository incompatible para el mismo EntityType:

```text
boot error
```

No:

```text
last registration wins
```

---

# 140. Decorators

Un Repository podrá decorarse:

```text
TelemetryRepositoryDecorator
```

o:

```text
PolicyAwareRepositoryDecorator
```

si preserva contratos.

---

# 141. Decoration order

El orden deberá ser:

```text
deterministic
explicit
inspectable
```

---

# 142. Repository diagnostics

Se podrá inspeccionar:

```text
EntityType
RepositoryClass
RepositoryKind
EntityManagerScopeId
DatabaseContextId
AppliedExtensions
AppliedScopes
Capabilities
```

---

# 143. Telemetry

Métricas propuestas:

```text
orm.repository.resolve.count
orm.repository.resolve.cache_hit
orm.repository.resolve.cache_miss

orm.repository.find.count
orm.repository.find_by.count
orm.repository.find_one_by.count
orm.repository.count.count
orm.repository.exists.count

orm.repository.query.created
orm.repository.query.executed
orm.repository.query.duration

orm.repository.result.entities
orm.repository.result.projections
orm.repository.result.scalars

orm.repository.stream.count
orm.repository.scope.error
```

---

# 144. Tracing

Ejemplo:

```text
Repository.findBy
├── Resolve Criteria
├── Create Entity Query
├── Translate Mapping
├── Query Engine
├── Execute
├── Hydrate
└── IdentityMap Reconciliation
```

---

# 145. Custom method tracing

Un método:

```text
UserRepository.findActiveSubscribers
```

deberá poder aparecer con un nombre lógico en telemetry.

---

# 146. Telemetry independence

Desactivar telemetry no deberá cambiar:

```text
query semantics
repository selection
IdentityMap behavior
hydration
ordering
```

---

# 147. Persistent runtime

Con FrankenPHP podrán compartirse:

```text
immutable RepositoryDescriptor
immutable RepositoryRegistry
compiled metadata
repository class definitions
```

---

# 148. Mutable state scoped

No deberán compartirse:

```text
Repository instance bound to EntityManager
RepositoryInstanceCache
EntityQuery instances
query parameters
result cursors
managed entities
```

---

# 149. FrankenPHP example

```text
Worker
│
├── Shared RepositoryRegistry
├── Shared EntityMetadataRegistry
│
├── Request A
│   ├── EntityManager A
│   └── UserRepository A
│
└── Request B
    ├── EntityManager B
    └── UserRepository B
```

---

# 150. RoadRunner/OpenSwoole

Se aplicará la misma regla:

> Los descriptors pueden compartirse si son inmutables; las instancias vinculadas a un PersistenceContext no.

---

# 151. Coroutine safety

Un Repository vinculado al EntityManager de Coroutine A no deberá utilizarse desde Coroutine B.

---

# 152. Repository state

Idealmente los custom repositories serán casi stateless excepto por referencias a servicios scoped/inmutables.

Evitar:

```php
private array $lastResults;
private ?EntityQuery $currentQuery;
```

---

# 153. Query objects

Cada:

```php
$repository->query();
```

deberá producir un query context independiente.

---

# 154. No mutable query reuse

Incorrecto:

```text
Repository
└── one shared mutable query builder
```

porque filtros de una consulta podrían filtrarse a otra.

---

# 155. Immutable EntityQuery

Si el Entity Query System adopta queries inmutables:

```php
$query = $users->query();

$active = $query->where('active', true);
$admins = $query->where('role', 'admin');
```

será especialmente seguro para runtime persistente.

---

# 156. Testing architecture

El Repository System requiere:

```text
unit tests
integration tests
mapping tests
scope tests
query tests
IdentityMap tests
persistent runtime tests
extension tests
```

---

# 157. Default repository tests

Probar:

```text
find
find missing
findAll
findBy
findOneBy
count
exists
query
```

---

# 158. Typed identifier tests

```text
UserId
UUID
ULID
integer ID
string ID
composite ID
```

---

# 159. IdentityMap tests

```php
$a = $repo->find($id);
$b = $repo->find($id);

assert($a === $b);
```

---

# 160. Cross-query identity test

```php
$a = $repo->find($id);

$b = $repo
    ->query()
    ->where('id', $id)
    ->one();

assert($a === $b);
```

---

# 161. Dirty state test

Una query repetida no deberá destruir cambios locales.

---

# 162. Criteria tests

Probar:

```text
valid field
unknown field
embedded field
relationship path
enum
value object
null
typed ID
invalid value type
```

---

# 163. Custom repository tests

Probar:

```text
custom class resolution
constructor injection
wrong entity type
invalid repository class
repository cache
repository extension
```

---

# 164. Scope tests

```text
EntityManager A
→ Repository A

EntityManager B
→ Repository B
```

y verificar:

```text
Repository A !== Repository B
```

cuando contienen manager scoped.

---

# 165. Tenant isolation tests

```text
Tenant A / User#10
Tenant B / User#10
```

deberán resolver entidades y repositories independientes.

---

# 166. Persistent worker tests

Simular:

```text
Request A
Request B
Request C
```

sobre el mismo worker y verificar:

```text
no repository instance leakage
no query state leakage
no entity leakage
no tenant leakage
```

---

# 167. Extension conflict tests

Dos registros incompatibles para el mismo EntityType deberán fallar de forma determinista.

---

# 168. Static analysis tests

La API deberá validarse con:

```text
PHPStan
Psalm
IDE inference
```

para garantizar que:

```text
Repository<User>::find()
→ User|null
```

---

# 169. Performance

La resolución de Repository deberá ser cercana a:

```text
O(1)
```

mediante:

```text
EntityType → descriptor
```

---

# 170. Repository cache

Después de la primera resolución dentro del manager:

```text
EntityType
→ Repository instance
```

deberá ser acceso directo.

---

# 171. Metadata compilation

La asociación:

```text
EntityType
→ RepositoryClass
```

deberá poder compilarse durante bootstrap.

---

# 172. Reflection

No deberá ejecutarse reflexión pesada en cada:

```php
$em->repository(User::class);
```

---

# 173. Query performance

Repository no añadirá capas innecesarias de consultas.

Idealmente:

```text
Repository convenience
→ EntityQuery
```

con overhead mínimo.

---

# 174. Memory

Repositories no deberán almacenar resultados de queries salvo que una capacidad explícita lo requiera.

---

# 175. `findAll()` governance

El framework podrá emitir diagnostics cuando:

```text
findAll()
```

sea usado sobre entidades con cardinalidad alta conocida.

No deberá prohibirse universalmente.

---

# 176. Repository API ergonomics

Objetivo:

```php
$users = $em->repository(User::class);

$user = $users->find($id);

$active = $users->findBy([
    'status' => UserStatus::ACTIVE,
]);
```

y para avanzado:

```php
$active = $users
    ->query()
    ->where('status', UserStatus::ACTIVE)
    ->where('profile.country', CountryCode::MX)
    ->orderBy('createdAt', 'DESC')
    ->limit(100)
    ->get();
```

---

# 177. Custom repository ergonomics

```php
final class UserRepository extends DefaultEntityRepository
{
    public function findActiveByCountry(
        CountryCode $country,
    ): array {
        return $this
            ->query()
            ->where('status', UserStatus::ACTIVE)
            ->where('profile.country', $country)
            ->orderBy('createdAt', 'DESC')
            ->get();
    }
}
```

---

# 178. No physical mapping leakage

Si mañana:

```text
User.email
```

cambia de:

```text
users.email
```

a:

```text
user_contacts.email_address
```

la firma:

```php
findByEmail(EmailAddress $email)
```

no debería cambiar por ese motivo.

---

# 179. Repository and domain language

Los custom repositories son un lugar apropiado para nombres de consulta cercanos al dominio:

```text
findPendingRenewals()
findOverdueInvoices()
findAvailableInventory()
```

si siguen representando selección/consulta de entidades.

---

# 180. Domain logic boundary

Un método:

```text
findInvoicesEligibleForCollection()
```

puede ser válido si expresa criterios de selección.

Pero:

```text
collectOverdueInvoices()
```

probablemente es un caso de uso.

---

# 181. Command/query distinction

Repository deberá favorecer:

```text
queries about entities
```

mientras comandos de negocio permanecen fuera.

Persistencia genérica sigue coordinada por EntityManager.

---

# 182. Repository method naming

Preferir:

```text
find...
get...
exists...
count...
query...
stream...
```

para operaciones de lectura.

Nombres específicos del dominio podrán utilizarse cuando describan selección.

---

# 183. `get()` semantics

Si se introduce:

```php
$repo->get($id);
```

deberá definirse claramente.

Posible convención:

```text
find(id) → Entity|null
get(id)  → Entity or EntityNotFoundException
```

---

# 184. Recommendation

Esto puede mejorar DX:

```php
$user = $users->get($id);
```

cuando ausencia sea excepcional.

---

# 185. `require()` alternative

También podría usarse:

```php
$users->require($id);
```

pero se recomienda escoger una sola convención pública para evitar duplicación.

---

# 186. Repository collection semantics

Repository no deberá implementar interfaces PHP como:

```text
Countable
Iterator
ArrayAccess
```

por defecto.

---

# 187. Why

Esto podría sugerir erróneamente:

```php
count($repository);
```

o:

```php
foreach ($repository as $entity)
```

como operaciones baratas.

En realidad podrían ejecutar queries enormes.

---

# 188. Explicit operations preferred

Preferir:

```php
$repository->count();
$repository->query()->stream();
```

---

# 189. Repository serialization

Una instancia Repository no deberá considerarse serializable entre requests/jobs.

---

# 190. Queue boundary

Incorrecto:

```text
Queue Payload
└── UserRepository instance
```

Correcto:

```text
Queue Payload
└── Entity identifiers / domain data
```

y reconstruir Repository dentro del job scope.

---

# 191. Repository and async jobs

```text
Job
    ↓
DatabaseContext
    ↓
EntityManager
    ↓
Repository
```

---

# 192. Repository capabilities

Podrá existir:

```php
final readonly class RepositoryCapabilities
{
    public function supportsStreaming(): bool;
    public function supportsBulkMutation(): bool;
    public function supportsEntityQuery(): bool;
}
```

pero capacidades generales deberían derivarse de componentes reales, no flags inventados.

---

# 193. Capability composition

```text
Repository Capability
=
ORM Capability
∩
Entity Mapping Capability
∩
Database Platform Capability
∩
Configured Policy
```

---

# 194. Vendor checks forbidden

No:

```php
if ($repository->database() === 'postgres') {
}
```

como arquitectura interna.

Preferir capabilities downstream.

---

# 195. Query portability

Custom repositories que utilicen solamente Entity Query portable podrán ejecutarse sobre:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 196. Platform-specific query

Si una consulta necesita una capacidad específica:

```text
full-text
JSON operators
geographic query
```

deberá declararlo mediante capabilities/extensiones.

---

# 197. Repository contract stability

El contrato base deberá evolucionar lentamente.

Funciones especializadas pertenecen a:

```text
EntityQuery
extensions
custom repositories
```

---

# 198. Arquitectura propuesta

```text
src/Quantum/Database/ORM/Repository/
│
├── Contract/
│   ├── EntityRepository.php
│   ├── RepositoryFactory.php
│   ├── RepositoryResolver.php
│   └── RepositoryExtension.php
│
├── DefaultEntityRepository.php
│
├── Definition/
│   ├── RepositoryDescriptor.php
│   ├── RepositoryKind.php
│   ├── RepositoryCapabilities.php
│   └── RepositoryDefinition.php
│
├── Registry/
│   ├── RepositoryRegistry.php
│   ├── CompiledRepositoryRegistry.php
│   └── RepositoryRegistration.php
│
├── Resolution/
│   ├── DefaultRepositoryResolver.php
│   ├── RepositoryResolution.php
│   └── RepositoryResolutionContext.php
│
├── Factory/
│   ├── DefaultRepositoryFactory.php
│   ├── ContainerRepositoryFactory.php
│   └── RepositoryConstructionPlan.php
│
├── Runtime/
│   ├── RepositoryInstanceCache.php
│   ├── RepositoryScope.php
│   └── RepositoryRuntime.php
│
├── Criteria/
│   ├── EntityCriteria.php
│   ├── EntityCriterion.php
│   ├── EntityOrder.php
│   └── CriteriaNormalizer.php
│
├── Specification/
│   ├── EntitySpecification.php
│   ├── CompositeSpecification.php
│   ├── AndSpecification.php
│   ├── OrSpecification.php
│   └── NotSpecification.php
│
├── Extension/
│   ├── RepositoryExtensionRegistry.php
│   ├── RepositoryDecorator.php
│   └── RepositoryExtensionDescriptor.php
│
├── Telemetry/
│   ├── RepositoryTelemetry.php
│   └── RepositoryDiagnostics.php
│
└── Exception/
    ├── RepositoryException.php
    ├── RepositoryResolutionException.php
    ├── RepositoryNotFoundException.php
    ├── InvalidRepositoryException.php
    ├── RepositoryTypeMismatchException.php
    ├── RepositoryConstructionException.php
    ├── RepositoryScopeException.php
    ├── RepositoryQueryException.php
    ├── InvalidRepositoryCriteriaException.php
    ├── UnknownRepositoryFieldException.php
    ├── NonUniqueRepositoryResultException.php
    └── RepositoryInvariantException.php
```

---

# 199. Relación con EntityManager

```text
EntityManager
    │
    └── repository(EntityType)
             ↓
      RepositoryResolver
             ↓
      RepositoryRegistry
             ↓
      RepositoryInstanceCache
             │
        hit ─┤
             │ miss
             ▼
      RepositoryFactory
             ↓
        Repository
```

---

# 200. Relación con Entity Query

```text
Repository
    ↓
query()
    ↓
EntityQueryFactory
    ↓
EntityQuery
    ↓
Entity Semantic Resolution
    ↓
Database Query Model
```

---

# 201. Relación con Metadata

```text
EntityMetadata
├── EntityType
├── Fields
├── Relationships
├── Identifier
└── RepositoryDescriptor
```

El Repository consume esta información.

No la redefine.

---

# 202. Relación con Mapping

```text
Repository field name
      ↓
Entity Mapping
      ↓
Database expression
```

Por ello:

```text
repository criteria
```

permanece independiente del nombre físico de columnas.

---

# 203. Relación con Hydration

```text
Repository Query
    ↓
Result
    ↓
Hydration Plan
    ↓
Entity Hydrator
    ↓
IdentityMap
    ↓
Entity
```

---

# 204. Relación con UnitOfWork

Los resultados `MANAGED` se registran en:

```text
UnitOfWork
```

mediante el pipeline de Hydration/EntityManager.

El Repository no realiza ese registro manualmente.

---

# 205. Relación con Model API

La API:

```php
User::query()
```

podrá resolver conceptualmente:

```text
Model API
    ↓
ModelContextResolver
    ↓
EntityManager
    ↓
EntityQueryFactory
```

o Repository cuando sea apropiado.

---

# 206. No circular dependency

Evitar:

```text
Repository
→ Model API
→ Repository
```

La dependencia debe dirigirse al ORM core.

---

# 207. Arquitectura consolidada del ORM

```text
                       Application
                            │
             ┌──────────────┴──────────────┐
             ▼                             ▼
        Model API                    Repository API
             │                             │
             │                             ▼
             │                       Entity Query
             │                             │
             └─────────────┬───────────────┘
                           ▼
                     EntityManager
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
     IdentityMap       UnitOfWork       EntityLoader
                           │                │
                           ▼                ▼
                  Persistence Engine    Hydration
                           │                │
                           └───────┬────────┘
                                   ▼
                              Query Engine
                                   │
                                   ▼
                            Execution Engine
                                   │
                                   ▼
                               Database
```

---

# 208. Invariantes arquitectónicas

## DB-REPOSITORY-001
Repository representará acceso lógico a un EntityType.

## DB-REPOSITORY-002
Repository no representará una tabla física.

## DB-REPOSITORY-003
Repository no será EntityManager.

## DB-REPOSITORY-004
Repository no será UnitOfWork.

## DB-REPOSITORY-005
Repository no será IdentityMap.

## DB-REPOSITORY-006
Repository no será Persistence Engine.

## DB-REPOSITORY-007
Repository no será SQL Compiler.

## DB-REPOSITORY-008
Repository no será Driver.

## DB-REPOSITORY-009
Repository no será Connection.

## DB-REPOSITORY-010
Repository no será Migration System.

## DB-REPOSITORY-011
Repository no será Business Service.

## DB-REPOSITORY-012
Repository no será Authorization System.

## DB-REPOSITORY-013
Repository estará asociado a un EntityType.

## DB-REPOSITORY-014
EntityType será distinto de table name.

## DB-REPOSITORY-015
EntityType será distinto de PHP class identity pura.

## DB-REPOSITORY-016
Repository podrá tener implementación default.

## DB-REPOSITORY-017
Una entidad no requerirá custom Repository.

## DB-REPOSITORY-018
Custom Repository será opcional.

## DB-REPOSITORY-019
Repository base tendrá API pequeña.

## DB-REPOSITORY-020
Repository no duplicará Entity Query.

## DB-REPOSITORY-021
`find()` delegará identity lookup al EntityManager.

## DB-REPOSITORY-022
Repository no mantendrá algoritmo alternativo de IdentityMap.

## DB-REPOSITORY-023
`find()` utilizará identificadores tipados/canónicos.

## DB-REPOSITORY-024
Mismo EntityKey dentro del mismo manager resolverá la misma instancia.

## DB-REPOSITORY-025
`findAll()` no implicará que el Repository sea una colección cargada.

## DB-REPOSITORY-026
`findBy()` utilizará propiedades de entidad.

## DB-REPOSITORY-027
`findBy()` no requerirá nombres físicos de columnas.

## DB-REPOSITORY-028
Criteria será traducido mediante Entity Mapping.

## DB-REPOSITORY-029
Unknown entity fields producirán error.

## DB-REPOSITORY-030
Criteria simple no aceptará SQL arbitrario.

## DB-REPOSITORY-031
Consultas complejas utilizarán Entity Query.

## DB-REPOSITORY-032
Repository no generará SQL.

## DB-REPOSITORY-033
Repository no compilará SQL.

## DB-REPOSITORY-034
Repository no ejecutará driver protocols.

## DB-REPOSITORY-035
Entity Query producirá Query Model/AST.

## DB-REPOSITORY-036
SQL Compiler permanecerá downstream.

## DB-REPOSITORY-037
`count()` no cargará entidades para contarlas.

## DB-REPOSITORY-038
`exists()` no cargará entidades innecesariamente.

## DB-REPOSITORY-039
`findOneBy()` no afirmará unicidad si no la verifica.

## DB-REPOSITORY-040
Repository soportará typing genérico para DX.

## DB-REPOSITORY-041
Repository<User>::find() deberá ser inferible como User|null.

## DB-REPOSITORY-042
Custom repositories podrán expresar consultas del dominio.

## DB-REPOSITORY-043
Custom repositories no deberán contener workflows de negocio arbitrarios.

## DB-REPOSITORY-044
RepositoryFactory será responsable de construcción.

## DB-REPOSITORY-045
RepositoryResolver será responsable de resolución.

## DB-REPOSITORY-046
RepositoryRegistry contendrá definiciones, no estado ORM mutable.

## DB-REPOSITORY-047
RepositoryRegistry podrá compartirse si es inmutable.

## DB-REPOSITORY-048
RepositoryInstanceCache será scope-local.

## DB-REPOSITORY-049
Repository bound a EntityManager no será process-global.

## DB-REPOSITORY-050
Múltiples llamadas repository(EntityType) podrán reutilizar instancia dentro del mismo manager.

## DB-REPOSITORY-051
Repository de manager A no deberá reutilizar manager B.

## DB-REPOSITORY-052
Custom Repository podrá usar DI.

## DB-REPOSITORY-053
DI no convertirá Repository en Service Locator.

## DB-REPOSITORY-054
Repository scope deberá respetar EntityManager scope.

## DB-REPOSITORY-055
Repository singleton con EntityManager mutable estará prohibido.

## DB-REPOSITORY-056
Repository query results managed respetarán IdentityMap.

## DB-REPOSITORY-057
Query repetida no sobrescribirá dirty entity silenciosamente.

## DB-REPOSITORY-058
Refresh será explícito.

## DB-REPOSITORY-059
Repository no administrará ChangeSets directamente.

## DB-REPOSITORY-060
Repository no administrará snapshots directamente.

## DB-REPOSITORY-061
Repository no programará inserts directamente.

## DB-REPOSITORY-062
Repository no programará updates directamente.

## DB-REPOSITORY-063
Repository no programará deletes directamente.

## DB-REPOSITORY-064
Persistencia genérica pertenecerá al EntityManager.

## DB-REPOSITORY-065
`save()` en Repository será opcional, no contrato fundamental.

## DB-REPOSITORY-066
Repository save convenience delegará a EntityManager.

## DB-REPOSITORY-067
Repository delete convenience delegará a EntityManager.

## DB-REPOSITORY-068
Bulk operations declararán managed-state policy.

## DB-REPOSITORY-069
Bulk mutation no dejará managed state inconsistente silenciosamente.

## DB-REPOSITORY-070
Specification será distinta de Repository.

## DB-REPOSITORY-071
Specification no contendrá SQL arbitrario.

## DB-REPOSITORY-072
Specifications deberán ser composables de forma tipada.

## DB-REPOSITORY-073
EntityCriteria será distinto de SQL criteria.

## DB-REPOSITORY-074
Magic runtime finders no serán arquitectura central.

## DB-REPOSITORY-075
Code generation será preferible a magia runtime cuando se requieran APIs derivadas.

## DB-REPOSITORY-076
Repository podrá retornar entidades.

## DB-REPOSITORY-077
Repository podrá retornar projections.

## DB-REPOSITORY-078
Repository podrá retornar scalars.

## DB-REPOSITORY-079
Repository podrá retornar streams.

## DB-REPOSITORY-080
Projection será preferida sobre Partial Entity cuando solo se necesiten campos parciales.

## DB-REPOSITORY-081
Scalar query no hidratará entidades.

## DB-REPOSITORY-082
Streaming tendrá política de tracking explícita.

## DB-REPOSITORY-083
Repository pagination delegará al Pagination System.

## DB-REPOSITORY-084
Repository no duplicará paginator engine.

## DB-REPOSITORY-085
Relationship path resolution pertenecerá al Entity Query/Relationship System.

## DB-REPOSITORY-086
Repository no decidirá physical join order.

## DB-REPOSITORY-087
Eager loading intent no significará necesariamente JOIN.

## DB-REPOSITORY-088
N+1 detection no pertenecerá directamente al Repository.

## DB-REPOSITORY-089
Repository loading respetará entity lifecycle semantics.

## DB-REPOSITORY-090
Repository find y EntityManager find compartirán semántica.

## DB-REPOSITORY-091
Global scopes deberán ser tipados.

## DB-REPOSITORY-092
Global scopes deberán ser inspeccionables.

## DB-REPOSITORY-093
Global scopes deberán ser deterministas.

## DB-REPOSITORY-094
Global scopes no deberán esconder predicates imposibles de diagnosticar.

## DB-REPOSITORY-095
Soft delete será extensión/capacidad explícita.

## DB-REPOSITORY-096
Core Repository no hardcodeará tenant_id.

## DB-REPOSITORY-097
Multitenancy llegará mediante DatabaseContext/EntityManager.

## DB-REPOSITORY-098
Repository no cambiará tenant dinámicamente.

## DB-REPOSITORY-099
Tenant A y Tenant B tendrán repositories runtime aislados.

## DB-REPOSITORY-100
Repository no elegirá replicas físicas.

## DB-REPOSITORY-101
Repository declarará intención de query y delegará routing.

## DB-REPOSITORY-102
Repository no calculará shards directamente.

## DB-REPOSITORY-103
Cross-shard query será capacidad explícita.

## DB-REPOSITORY-104
Repository no duplicará IdentityMap mediante caches privados.

## DB-REPOSITORY-105
Query Cache será sistema separado.

## DB-REPOSITORY-106
Entity Cache será sistema separado.

## DB-REPOSITORY-107
Custom repository cache deberá integrarse con Cache System.

## DB-REPOSITORY-108
Repository no concederá autorización por cargar una entidad.

## DB-REPOSITORY-109
Entity identifier no será prueba de autorización.

## DB-REPOSITORY-110
Security query scopes serán explícitos y auditables.

## DB-REPOSITORY-111
Telemetry deberá soportar redacción de datos sensibles.

## DB-REPOSITORY-112
Repository errors preservarán semántica ORM.

## DB-REPOSITORY-113
Unknown field no será tratado como nombre físico de columna.

## DB-REPOSITORY-114
Raw SQL deberá pasar por escape hatch gobernado.

## DB-REPOSITORY-115
Repository extensions estarán registradas explícitamente.

## DB-REPOSITORY-116
Repository extensions no utilizarán monkey patching.

## DB-REPOSITORY-117
Extension conflicts producirán error determinista.

## DB-REPOSITORY-118
No se utilizará last-wins silencioso para custom Repository conflictivo.

## DB-REPOSITORY-119
Repository decorators preservarán contrato.

## DB-REPOSITORY-120
Decorator order será determinista.

## DB-REPOSITORY-121
Repository diagnostics mostrarán implementación efectiva.

## DB-REPOSITORY-122
Repository diagnostics mostrarán scopes aplicados.

## DB-REPOSITORY-123
Telemetry no cambiará semántica de consultas.

## DB-REPOSITORY-124
RepositoryDescriptor inmutable podrá compartirse entre workers.

## DB-REPOSITORY-125
RepositoryRegistry inmutable podrá compartirse entre requests.

## DB-REPOSITORY-126
Repository instances manager-bound serán scope-local.

## DB-REPOSITORY-127
EntityQuery instances serán operation-local.

## DB-REPOSITORY-128
Query parameters serán operation-local.

## DB-REPOSITORY-129
Result cursors no serán process-global.

## DB-REPOSITORY-130
Repositories no mantendrán mutable query builder compartido.

## DB-REPOSITORY-131
Cada query tendrá contexto independiente.

## DB-REPOSITORY-132
FrankenPHP requests no compartirán Repository mutable.

## DB-REPOSITORY-133
RoadRunner requests no compartirán Repository mutable.

## DB-REPOSITORY-134
OpenSwoole coroutines no compartirán Repository mutable.

## DB-REPOSITORY-135
Repository manager-bound no será asumido thread-safe.

## DB-REPOSITORY-136
Repository no será serializado como payload de job.

## DB-REPOSITORY-137
Jobs reconstruirán Repository dentro de su scope.

## DB-REPOSITORY-138
Repository capabilities derivarán de capacidades reales.

## DB-REPOSITORY-139
Repository no utilizará vendor conditionals como arquitectura principal.

## DB-REPOSITORY-140
Consultas portables permanecerán independientes del vendor.

## DB-REPOSITORY-141
Capacidades platform-specific serán explícitas.

## DB-REPOSITORY-142
Repository base contract evolucionará conservadoramente.

## DB-REPOSITORY-143
Repository resolution deberá ser determinista.

## DB-REPOSITORY-144
Repository resolution deberá ser cacheable.

## DB-REPOSITORY-145
Reflection pesada no deberá ejecutarse por cada resolución.

## DB-REPOSITORY-146
Repository no almacenará resultados arbitrariamente.

## DB-REPOSITORY-147
`findAll()` podrá producir advertencias de cardinalidad sin alterar su semántica.

## DB-REPOSITORY-148
Repository no implementará Iterator por defecto.

## DB-REPOSITORY-149
Repository no implementará ArrayAccess por defecto.

## DB-REPOSITORY-150
Repository no implementará Countable por defecto si ello oculta queries.

## DB-REPOSITORY-151
Operaciones potencialmente costosas deberán ser explícitas.

## DB-REPOSITORY-152
Custom Repository deberá permanecer orientado a consultas/persistencia de entidades.

## DB-REPOSITORY-153
Cambios físicos de mapping no deberán obligar a cambiar APIs de dominio cuando la semántica permanezca.

## DB-REPOSITORY-154
Repository deberá preservar lenguaje del dominio sin absorber casos de uso.

## DB-REPOSITORY-155
Repository y Model API convergerán en el mismo ORM Core.

## DB-REPOSITORY-156
Repository no creará un segundo PersistenceContext.

## DB-REPOSITORY-157
Repository no creará un segundo UnitOfWork.

## DB-REPOSITORY-158
Repository no creará un segundo IdentityMap.

## DB-REPOSITORY-159
Repository no creará un segundo Query Engine.

## DB-REPOSITORY-160
Repository permanecerá una abstracción de acceso a colecciones lógicas de entidades.

---

# 209. Anti-patterns

## 209.1 Repository como SQL container

```php
final class UserRepository
{
    public function query(string $sql): array
    {
        // ...
    }
}
```

Problema:

```text
Repository
→ SQL abstraction
```

en vez de:

```text
Repository
→ Entity abstraction
```

---

## 209.2 Repository como Service Layer

```php
final class OrderRepository
{
    public function checkout(Order $order): void
    {
        $this->chargeCard();
        $this->sendEmail();
        $this->reserveInventory();
    }
}
```

Debe moverse a:

```text
CheckoutApplicationService
```

---

## 209.3 Repository singleton

```php
final class UserRepository
{
    private static ?EntityManager $manager;
}
```

Incompatible con runtime persistente seguro.

---

## 209.4 Query state stored in Repository

```php
final class UserRepository
{
    private EntityQuery $query;
}
```

si se reutiliza mutablemente entre operaciones.

Preferir:

```php
public function query(): EntityQuery
{
    return $this->queryFactory->for(User::class);
}
```

---

## 209.5 Physical table API

Evitar:

```php
$users->whereColumn('users.created_at', ...);
```

en la API Repository normal.

Preferir:

```php
$users
    ->query()
    ->where('createdAt', ...);
```

---

## 209.6 Hidden tenant mutation

Evitar:

```php
$users->forTenant($tenantB);
```

si modifica internamente el mismo Repository/EntityManager.

---

## 209.7 Repository-specific IdentityMap

Evitar:

```php
private array $loadedUsers = [];
```

como segunda identidad ORM.

---

# 210. Flujo completo de lectura

```text
Application
    ↓
UserRepository
    ↓
EntityQuery
    ↓
Entity Metadata
    ↓
Entity Mapping
    ↓
Query Model / AST
    ↓
Semantic Engine
    ↓
Optimizer
    ↓
Planner
    ↓
SQL Compiler
    ↓
Execution Engine
    ↓
Result
    ↓
Hydration
    ↓
IdentityMap
    ↓
UnitOfWork registration
    ↓
User Entity
```

---

# 211. Flujo `find()`

```text
UserRepository.find(UserId)
        ↓
EntityManager.find(User, UserId)
        ↓
EntityKey
        ↓
IdentityMap
        │
        ├──── HIT ────► User
        │
        └──── MISS
               ↓
          EntityLoader
               ↓
          Entity Query
               ↓
          Query Engine
               ↓
          Hydration
               ↓
          IdentityMap
               ↓
              User
```

---

# 212. Flujo custom Repository

```php
$user = $users->findByEmail($email);
```

```text
UserRepository.findByEmail()
        ↓
EntityQuery<User>
        ↓
where(User.email = EmailAddress)
        ↓
Entity Mapping
        ↓
Query AST
        ↓
Query Engine
        ↓
Hydration
        ↓
User|null
```

---

# 213. Flujo de escritura

Si se ofrece una conveniencia:

```php
$users->save($user);
```

deberá ser:

```text
Repository
    ↓
EntityManager.persist()
    ↓
UnitOfWork
```

y posteriormente:

```text
EntityManager.flush()
    ↓
Persistence Engine
```

No:

```text
Repository
→ SQL UPDATE
```

---

# 214. Fórmula del Repository

```text
Repository<TEntity>
=
EntityType Binding
+
Entity Lookup
+
Entity Query Entry Point
+
Domain-Oriented Query Methods
+
Scoped ORM Context
```

---

# 215. Fórmula de resolución

```text
resolveRepository(EntityType)
=
RepositoryRegistry[EntityType]
→ RepositoryDescriptor
→ RepositoryInstanceCache?
→ RepositoryFactory
→ Scoped Repository
```

---

# 216. Fórmula de consulta

```text
RepositoryQuery
=
EntityIntent
+
EntityMetadata
+
EntityCriteria
+
EntityQueryTranslation
→
Database Query Model
```

---

# 217. Fórmula de identidad

```text
RepositoryEntityResult
+
ManagedHydrationMode
→
EntityKey
→
IdentityMap
→
Canonical Managed Instance
```

---

# 218. Fórmula de scope

```text
SafeRepositoryScope
=
RepositoryBoundTo(EntityManagerScope)
∧
ImmutableSharedDefinitions
∧
OperationLocalQueries
∧
NoProcessGlobalManager
∧
NoCrossTenantMutation
```

---

# 219. Fórmula de responsabilidad

```text
Repository Responsibility
=
Locate Entities
+
Query Logical Entity Collection
+
Expose Domain-Specific Selection Methods
```

No:

```text
Repository Responsibility
=
Application Workflow
+
Authorization
+
SQL Compilation
+
Transaction Ownership
+
Business Orchestration
```

---

# 220. Master Formula

```text
Database Repository System
=
Typed Entity Repositories
+
Default Repository
+
Custom Repositories
+
EntityType Binding
+
Repository Descriptors
+
Repository Registry
+
Repository Resolver
+
Repository Factory
+
Scoped Repository Cache
+
EntityManager Integration
+
Identity Lookup Delegation
+
Entity Query Integration
+
Typed Criteria
+
Specifications
+
Projection Support
+
Streaming Support
+
Pagination Integration
+
Relationship Query Integration
+
Managed-State Consistency
+
Bulk Mutation Governance
+
Extension Governance
+
Multitenancy Isolation
+
Persistent Runtime Isolation
+
Diagnostics
+
Telemetry
+
Static Analysis Support
```

---

# 221. Master Rule

> **En VoltStack, un Repository representa una colección lógica y tipada de entidades dentro de un contexto ORM. Puede localizar entidades y expresar consultas del dominio, pero delega identidad al EntityManager, consultas al Entity Query System, persistencia al UnitOfWork/Persistence Engine y SQL al Query Compiler.**

---

# 222. Arquitectura resultante

Después de `119_DATABASE_REPOSITORY_SYSTEM.md`:

```text
Application
│
├──────────── Model API ──────────────┐
│                                     │
└──────────── Repository API ─────┐   │
                                 │   │
                                 ▼   ▼
                             Entity Query
                                 │
                    ┌────────────┘
                    ▼
              EntityManager
                    │
        ┌───────────┼────────────┐
        ▼           ▼            ▼
   IdentityMap   UnitOfWork   EntityLoader
                    │            │
                    ▼            ▼
            Persistence Engine Hydration
                    │            │
                    └─────┬──────┘
                          ▼
                     Query Engine
                          ▼
                   Execution Engine
                          ▼
                       Driver
```

Se preserva:

```text
Repository
≠
EntityManager
≠
EntityQuery
≠
QueryBuilder
≠
PersistenceEngine
≠
BusinessService
```

---

# 223. Estado del bloque ORM

```text
112_DATABASE_ORM_ARCHITECTURE.md
        ↓
113_DATABASE_ENTITY_MODEL.md
        ↓
114_DATABASE_MODEL_API_SYSTEM.md
        ↓
115_DATABASE_ENTITY_METADATA_SYSTEM.md
        ↓
116_DATABASE_ENTITY_MAPPING_SYSTEM.md
        ↓
117_DATABASE_ATTRIBUTE_MAPPING_SYSTEM.md
        ↓
118_DATABASE_ENTITY_MANAGER_SYSTEM.md
        ↓
119_DATABASE_REPOSITORY_SYSTEM.md
```

Ya están formalizados:

```text
ORM Architecture
Entity Model
Model API
Metadata
Mapping
Attribute Mapping
EntityManager
Repository
```

El siguiente componente debe formalizar cómo una consulta expresada en términos de entidades, propiedades y relaciones se convierte en el Query Model estructurado del Database System.

---

# 224. Siguiente documento

```text
120_DATABASE_ENTITY_QUERY_SYSTEM.md
```

Este documento deberá definir formalmente:

```text
EntityQuery
EntityQueryBuilder
EntityQueryFactory
EntityQueryContext
Entity Field References
Entity Relationship Paths
Entity Predicates
Entity Projections
Entity Ordering
Entity Aggregation
Entity Query Translation
Entity Query Semantic Resolution
Entity Query Result Modes
Entity Query Hydration Modes
Entity Scopes
Entity Query Extensions
```

con el pipeline:

```text
Repository / Model API
        ↓
Entity Query
        ↓
Entity Semantic Resolution
        ↓
Entity Mapping
        ↓
Database Query Model / AST
        ↓
Query Semantic Engine
        ↓
Optimizer
        ↓
Planner
        ↓
Compiler
        ↓
Executor
        ↓
Hydration / Projection
```

y preservando especialmente:

```text
Entity Query
≠
Repository
≠
Database Query Builder
≠
SQL
≠
Hydrator
≠
Persistence Engine
```

La regla central del siguiente documento será:

> **Entity Query expresa qué entidades, propiedades, relaciones y proyecciones desea consultar la aplicación; traduce esa intención al Query Model universal de VoltStack, pero nunca genera ni ejecuta SQL directamente.**