# 112_DATABASE_ORM_ARCHITECTURE.md

# VoltStack Quantum Database
## Database ORM Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 112 — Database ORM Architecture  
**Bloque:** 10 — ORM  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database ORM Architecture` define la arquitectura general del **Object-Relational Mapping System** de VoltStack.

El ORM será la capa responsable de representar, mapear, consultar, hidratar, rastrear y coordinar la persistencia de objetos de dominio sobre el Database System sin convertir las entidades en generadores de SQL ni duplicar la infraestructura del Query Engine.

Su posición conceptual será:

```text
Application / Domain
        │
        ▼
┌───────────────────────────────┐
│        VoltStack ORM          │
│                               │
│ Model API     Repository API  │
│      │              │         │
│      └──────┬───────┘         │
│             ▼                 │
│       Entity Manager          │
│             │                 │
│   ┌─────────┼──────────┐      │
│   ▼         ▼          ▼      │
│ Metadata  IdentityMap  UoW    │
│   │                    │      │
│   └──────────┬─────────┘      │
│              ▼                │
│      Persistence Engine       │
└──────────────┼────────────────┘
               ▼
          Query Engine
               │
               ▼
        Execution Engine
               │
               ▼
          Connection
               │
               ▼
            Driver
               │
               ▼
           Database
```

El principio fundamental será:

> **El ORM conoce entidades y su persistencia; el Query Engine conoce consultas; el Compiler conoce SQL; el Execution Engine conoce ejecución; el Driver conoce el protocolo de base de datos.**

---

# 2. Objetivos

La arquitectura ORM deberá proporcionar:

1. Entity Mapping fuertemente tipado.
2. Metadata compilable y cacheable.
3. Entity Manager.
4. Repository Pattern.
5. Model API de ergonomía similar a Laravel.
6. Unit of Work.
7. Identity Map.
8. Change Tracking.
9. Persistence Planning.
10. Hydration.
11. Relationship Mapping.
12. Lazy/Eager Loading.
13. lifecycle hooks.
14. integración con Transaction System.
15. integración con Query Engine.
16. soporte de Value Objects.
17. casting y tipos.
18. optimistic/pessimistic locking.
19. eventos de persistencia.
20. extensibilidad.
21. seguridad bajo runtimes persistentes.
22. compatibilidad con FrankenPHP.
23. integración opcional con Multitenancy.
24. alta capacidad de optimización.
25. bajo acoplamiento entre Domain Model y Database Engine.

---

# 3. No objetivos

El ORM no será responsable de:

- generar SQL directamente;
- manejar sockets;
- implementar drivers;
- implementar Connection Pooling;
- compilar Schema AST;
- administrar migrations;
- reemplazar Query Builder;
- reemplazar Transaction Manager;
- implementar cache genérico;
- implementar Authorization;
- implementar Validation de aplicación;
- definir reglas de negocio;
- implementar Multitenancy dentro del núcleo;
- convertirse en un Service Container;
- mantener estado global de entidades entre requests.

---

# 4. Separaciones fundamentales

VoltStack preservará:

```text
ORM
≠
Query Builder

ORM
≠
SQL Compiler

ORM
≠
Execution Engine

ORM
≠
Driver

ORM
≠
Schema System

ORM
≠
Migration System

ORM
≠
Entity Manager

ORM
≠
Repository

ORM
≠
Active Record

ORM
≠
Unit of Work

ORM
≠
Identity Map

ORM
≠
Hydrator

Entity
≠
Database Row

Entity
≠
Query

Entity
≠
SQL

Model API
≠
Persistence Engine

Repository
≠
Query Builder

UnitOfWork
≠
Transaction

IdentityMap
≠
Cache

Hydration
≠
Persistence

Mapping
≠
Schema

ORM Relationship
≠
Foreign Key

ORM Validation
≠
Database Constraint

Entity Lifecycle
≠
Request Lifecycle

flush()
≠
commit()

persist()
≠
INSERT

remove()
≠
DELETE immediately
```

---

# 5. Regla arquitectónica principal

La dependencia será estrictamente descendente:

```text
ORM
 ↓
Query Engine
 ↓
Execution Engine
 ↓
Connection
 ↓
Driver
```

Nunca:

```text
Query Engine
    ↓
   ORM
```

ni:

```text
Driver
 ↓
ORM
```

---

# 6. ORM sobre el Query Engine

El ORM no construirá SQL.

En cambio:

```text
Entity Operation
      ↓
ORM Persistence Planner
      ↓
Query Model / Query AST
      ↓
Semantic Engine
      ↓
Optimizer
      ↓
Query Planner
      ↓
SQL Compiler
      ↓
Execution Engine
```

Por ejemplo:

```php
$user->name = 'Ana';

$entityManager->flush();
```

conceptualmente producirá:

```text
Entity Change
    ↓
UnitOfWork
    ↓
ChangeSet
    ↓
Persistence Planner
    ↓
Update Query Model
    ↓
Query Engine
```

Nunca:

```php
$sql = "UPDATE users SET name = ...";
```

dentro del ORM.

---

# 7. Arquitectura macro

```text
                         Application
                             │
            ┌────────────────┴────────────────┐
            │                                 │
            ▼                                 ▼
       Model API                        Repository API
            │                                 │
            └────────────────┬────────────────┘
                             ▼
                       Entity Manager
                             │
          ┌──────────────────┼───────────────────┐
          │                  │                   │
          ▼                  ▼                   ▼
       Metadata          Entity Query        Entity State
          │                  │                   │
          │                  ▼                   │
          │            Query Translation         │
          │                  │                   │
          └────────────┬─────┴────────────┐      │
                       ▼                  ▼      ▼
                  Identity Map       Unit of Work
                       │                  │
                       └────────┬─────────┘
                                ▼
                       Persistence Engine
                                │
                       Persistence Planner
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

---

# 8. Dos APIs, un solo ORM

VoltStack deberá ofrecer dos estilos principales de interacción:

```text
Model API
```

y:

```text
Repository / EntityManager API
```

sin crear dos ORM diferentes.

---

# 9. Model API

La experiencia podrá ser:

```php
$user = User::find(10);

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

Esta API busca productividad y familiaridad para desarrolladores de Laravel.

---

# 10. Repository API

También deberá permitirse:

```php
$user = $entityManager
    ->repository(User::class)
    ->find(10);

$user->rename('Ana');

$entityManager->flush();
```

Esto favorece:

- Domain Models desacoplados;
- DDD;
- testing;
- repositories especializados;
- separación de persistencia.

---

# 11. Una sola infraestructura

Ambas APIs deberán converger:

```text
Model API ──────────┐
                    │
                    ▼
              Entity Manager
                    │
Repository API ─────┘
                    │
                    ▼
                 ORM Core
```

Nunca:

```text
ActiveRecordORM
+
DataMapperORM
```

como motores separados.

---

# 12. Active Record como fachada

El estilo:

```php
$user->save();
```

será una **convenience API**.

Internamente:

```text
Model::save()
    ↓
ModelPersistenceBridge
    ↓
EntityManager
    ↓
UnitOfWork
    ↓
Persistence Engine
```

---

# 13. Regla de Active Record

> **Active Record será una API sobre el ORM; no un segundo Persistence Engine.**

---

# 14. Entidad

Una entidad representa identidad de dominio persistente.

Ejemplo:

```php
#[Entity]
#[Table('users')]
final class User
{
    #[Id]
    #[Column(type: 'uuid')]
    private UserId $id;

    #[Column]
    private string $name;
}
```

---

# 15. Entity ≠ Row

Una fila:

```text
users
--------------------------------
id | name | email | created_at
```

es una representación relacional.

Una entidad:

```text
User
```

es un objeto con:

```text
identity
state
behavior
relationships
domain semantics
```

---

# 16. Entity identity

VoltStack distinguirá:

```text
EntityIdentity
≠
ObjectIdentity
≠
DatabasePrimaryKeyRepresentation
```

Por ejemplo:

```php
UserId
```

puede mapearse a:

```text
UUID
CHAR(36)
BINARY(16)
native UUID
```

según plataforma.

---

# 17. Entity key

Se propone:

```php
final readonly class EntityKey
{
    public function __construct(
        public EntityType $type,
        public EntityIdentifier $identifier,
    ) {}
}
```

---

# 18. Composite identifiers

El ORM deberá soportar cuando sea necesario:

```text
EntityIdentifier
    ├── SimpleIdentifier
    └── CompositeIdentifier
```

aunque se recomienda mantener IDs simples para la mayoría de aplicaciones.

---

# 19. Entity Metadata

La infraestructura central será:

```text
EntityMetadata
```

que describe cómo una entidad participa en persistencia.

---

# 20. Metadata conceptual

```text
EntityMetadata
├── EntityType
├── TableMapping
├── IdentifierMapping
├── FieldMappings
├── RelationshipMappings
├── EmbeddedMappings
├── LifecycleMetadata
├── LockingMetadata
├── InheritanceMetadata
└── ExtensionMetadata
```

---

# 21. Metadata ≠ Reflection

Reflection puede ser una fuente.

No será la representación final.

```text
PHP Reflection
      ↓
Metadata Loader
      ↓
Metadata Normalizer
      ↓
Metadata Validator
      ↓
Compiled Entity Metadata
```

---

# 22. Metadata compilation

En producción:

```text
Attributes
   ↓
Reflection
   ↓
Metadata Compiler
   ↓
Compiled Metadata
   ↓
Metadata Cache
```

permitiendo evitar reflection repetitiva por request.

---

# 23. Metadata sources

La arquitectura podrá soportar:

```text
PHP Attributes
Programmatic Mapping
Compiled Mapping
Extension Mapping
```

El formato recomendado será PHP Attributes.

---

# 24. Attributes

Ejemplo:

```php
#[Entity]
#[Table('users')]
final class User
{
    #[Id]
    #[Column(type: 'uuid')]
    private UserId $id;

    #[Column(length: 150)]
    private string $name;

    #[Column(unique: true)]
    private string $email;
}
```

---

# 25. Attributes ≠ Metadata runtime

Los attributes serán entrada declarativa.

El runtime utilizará:

```text
CompiledEntityMetadata
```

---

# 26. Entity Manager

`EntityManager` será el principal coordinador del ORM.

Contrato conceptual:

```php
interface EntityManager
{
    public function find(
        string $entity,
        mixed $id,
    ): ?object;

    public function persist(object $entity): void;

    public function remove(object $entity): void;

    public function flush(): void;

    public function clear(): void;

    public function repository(string $entity): Repository;
}
```

---

# 27. EntityManager ≠ God Object

Aunque sea API de coordinación, delegará internamente:

```text
EntityManager
├── MetadataRegistry
├── IdentityMap
├── UnitOfWork
├── EntityLoader
├── EntityPersister
├── RepositoryResolver
└── HydrationEngine
```

---

# 28. persist()

`persist()` no significa:

```text
execute INSERT now
```

Significa:

> registrar una entidad dentro del contexto de persistencia.

---

# 29. remove()

Igualmente:

```php
$entityManager->remove($user);
```

no ejecutará necesariamente `DELETE` inmediatamente.

---

# 30. flush()

`flush()` sincroniza cambios pendientes del ORM con la base de datos.

Conceptualmente:

```text
Tracked Entities
      ↓
UnitOfWork
      ↓
Change Detection
      ↓
ChangeSets
      ↓
Persistence Planner
      ↓
Persistence Plan
      ↓
Query Engine
      ↓
Execution
      ↓
State Synchronization
```

---

# 31. flush() ≠ transaction commit

Regla crítica:

```text
flush()
≠
commit()
```

Puede existir:

```php
$transaction->run(function () use ($entityManager) {
    $entityManager->flush();

    // more work

    $entityManager->flush();
});
```

con un único commit posterior.

---

# 32. Unit of Work

`UnitOfWork` rastreará el estado de entidades dentro de un scope.

Estados conceptuales:

```text
NEW
MANAGED
DIRTY
REMOVED
DETACHED
```

podrán refinarse en documentos posteriores.

---

# 33. UnitOfWork ≠ Transaction

`UnitOfWork` representa:

```text
in-memory persistence changes
```

mientras Transaction representa:

```text
database atomicity/isolation boundary
```

---

# 34. Identity Map

El Identity Map garantiza dentro del scope:

```text
Same Entity Type
+
Same Entity Identifier
        ↓
Same Managed Object Instance
```

formalmente:

```text
find(User, 10) === find(User, 10)
```

dentro del mismo EntityManager scope.

---

# 35. Identity Map ≠ Cache

Identity Map:

```text
scope-local identity consistency
```

Cache:

```text
cross-operation reusable data optimization
```

Son conceptos distintos.

---

# 36. Ejemplo

```php
$a = $entityManager->find(User::class, 10);
$b = $entityManager->find(User::class, 10);

assert($a === $b);
```

---

# 37. Identity Map lifecycle

En un request tradicional:

```text
Request Start
     ↓
IdentityMap created
     ↓
ORM operations
     ↓
Request End
     ↓
IdentityMap cleared
```

---

# 38. FrankenPHP

Con workers persistentes:

```text
Worker
 ├── Request A
 │    └── IdentityMap A
 │
 ├── RESET
 │
 └── Request B
      └── IdentityMap B
```

Nunca:

```text
Worker
└── Global IdentityMap
     ├── Request A entities
     └── Request B entities
```

---

# 39. Entity state

El ORM deberá distinguir:

```text
Entity Domain State
```

de:

```text
ORM Persistence State
```

La entidad no necesita conocer si está:

```text
MANAGED
DIRTY
REMOVED
```

---

# 40. External state tracking

Preferencia arquitectónica:

```text
Entity
   │
   ▼
UnitOfWork
   │
   └── EntityState
```

en lugar de contaminar entidades con flags internos.

---

# 41. Change tracking

Se contemplarán estrategias como:

```text
SNAPSHOT
EXPLICIT
NOTIFY
CUSTOM
```

con `SNAPSHOT` como default razonable.

---

# 42. Snapshot

Al hidratar:

```text
Entity
+
Original State Snapshot
```

Después:

```text
Current State
-
Original Snapshot
=
ChangeSet
```

---

# 43. ChangeSet

Ejemplo:

```text
User#10

name:
    old = "Alice"
    new = "Ana"

email:
    unchanged
```

---

# 44. ChangeSet ≠ SQL update

El ChangeSet expresa diferencia de estado.

Persistence Planner decidirá cómo convertirlo en operaciones.

---

# 45. Persistence Engine

El ORM tendrá:

```text
Persistence Engine
```

responsable de convertir cambios ORM en operaciones del Query Engine.

---

# 46. Persistence architecture

```text
UnitOfWork
    ↓
ChangeSets
    ↓
Persistence Planner
    ↓
Entity Persistence Plan
    ↓
Insert/Update/Delete Operations
    ↓
Query Model
    ↓
Query Engine
```

---

# 47. Persistence Engine ≠ Query Engine

Persistence Engine entiende:

```text
entities
metadata
relationships
change sets
entity dependencies
```

Query Engine entiende:

```text
relations
predicates
expressions
joins
query plans
```

---

# 48. Persistence ordering

Considere:

```text
Order
 └── Customer
```

Si ambos son nuevos:

```text
INSERT Customer
      ↓
obtain identifier
      ↓
INSERT Order
```

El Persistence Planner deberá ordenar según dependencias.

---

# 49. Persistence dependency graph

Conceptualmente:

```text
EntityChangeGraph
        ↓
Dependency Analysis
        ↓
PersistenceOperationGraph
        ↓
Persistence Plan
```

---

# 50. Cycles

Relaciones cíclicas podrán requerir:

```text
deferred FK
nullable transitional FK
post-insert update
platform capability
```

pero la estrategia será explícita.

---

# 51. Querying entities

El ORM deberá ofrecer:

```text
Entity Query System
```

sobre el Query Engine.

---

# 52. Entity Query ≠ SQL Query

Ejemplo:

```php
User::query()
    ->where('active', true)
    ->where('profile.country', 'MX')
    ->get();
```

se transforma:

```text
Entity Query
    ↓
Entity Metadata Resolution
    ↓
Relationship Resolution
    ↓
Query Model
    ↓
Query Engine
```

---

# 53. Query Builder reuse

No deberá existir:

```text
ORM SQL Builder
```

paralelo al Query Builder general.

---

# 54. Entity-aware query layer

Sí podrá existir:

```text
EntityQueryBuilder
```

pero su salida será:

```text
Query Model / Query AST
```

no SQL.

---

# 55. Repository

Repository representa acceso a colecciones de entidades.

Contrato conceptual:

```php
interface Repository
{
    public function find(mixed $id): ?object;

    public function findAll(): iterable;

    public function findBy(array $criteria): iterable;

    public function query(): EntityQueryBuilder;
}
```

---

# 56. Repository specialization

Ejemplo:

```php
final class UserRepository extends EntityRepository
{
    public function activeAdministrators(): iterable
    {
        return $this->query()
            ->where('active', true)
            ->whereRelation('roles', 'name', 'admin')
            ->get();
    }
}
```

---

# 57. Repository ≠ Service

Un Repository no deberá convertirse en:

```text
UserService
BillingService
NotificationService
```

Su responsabilidad sigue siendo acceso/persistencia de entidades.

---

# 58. Repository ≠ Table Gateway

Repository trabaja con:

```text
entities
```

no simplemente rows.

---

# 59. Hydration

El Hydration System transforma resultados relacionales en representaciones ORM.

```text
Database Result
      ↓
Result System
      ↓
Hydration Plan
      ↓
Entity Hydrator
      ↓
Identity Map
      ↓
Entity
```

---

# 60. Hydration ≠ construction only

Hydration puede involucrar:

```text
entity creation
identity resolution
field conversion
value object reconstruction
relationship initialization
snapshot creation
UnitOfWork registration
```

---

# 61. Constructor strategy

VoltStack no deberá asumir universalmente:

```php
new Entity(...all database columns...)
```

Se definirán estrategias explícitas de instanciación.

---

# 62. Domain constructors

El ORM deberá permitir entidades con constructores de dominio:

```php
final class User
{
    public function __construct(
        UserId $id,
        Email $email,
    ) {
        // domain invariants
    }
}
```

sin exigir constructor vacío público.

---

# 63. Hydration factory

Podrá existir:

```text
EntityInstantiator
```

separado del hydrator.

---

# 64. Hydration and Identity Map

Antes de crear una entidad:

```text
Result Row
   ↓
Resolve Entity Key
   ↓
Identity Map lookup
   ├── found → reuse
   └── missing → instantiate/hydrate/register
```

---

# 65. Duplicate joins

Una consulta:

```text
User
JOIN Posts
```

puede devolver múltiples rows del mismo User.

Identity Map evitará múltiples objetos para la misma identidad.

---

# 66. Partial entities

Las partial entities deberán tratarse con extrema precaución.

```text
Partial Entity
≠
Fully Loaded Entity
```

---

# 67. Partial hydration

El estado de carga deberá ser explícito.

Nunca:

```text
missing field
=
null
```

si el campo simplemente no fue cargado.

---

# 68. Relationships

El ORM modelará asociaciones entre entidades.

```text
OneToOne
OneToMany
ManyToOne
ManyToMany
Polymorphic
```

---

# 69. ORM Relationship ≠ Foreign Key

Una relación ORM expresa navegación y semántica de objetos.

Foreign Key expresa integridad referencial de base de datos.

---

# 70. Ejemplo

```php
#[ManyToOne(target: Customer::class)]
private Customer $customer;
```

puede corresponder a:

```text
orders.customer_id
→
customers.id
```

pero son representaciones diferentes.

---

# 71. Relationship metadata

Deberá describir:

```text
source entity
target entity
cardinality
ownership
join mapping
loading strategy
cascade policy
orphan behavior
ordering
inverse mapping
```

---

# 72. Ownership

El ORM deberá distinguir claramente:

```text
Owning Side
Inverse Side
```

cuando sea necesario.

---

# 73. Cascade

Las cascades ORM deberán ser explícitas:

```text
PERSIST
REMOVE
DETACH
REFRESH
MERGE
```

si las operaciones correspondientes son soportadas.

---

# 74. ORM cascade ≠ DB cascade

```text
ORM Cascade Remove
≠
ON DELETE CASCADE
```

---

# 75. Orphan removal

También:

```text
OrphanRemoval
≠
DatabaseCascade
```

---

# 76. Loading strategies

Se contemplarán:

```text
LAZY
EAGER
EXPLICIT
BATCH
```

---

# 77. Default loading

La estrategia deberá evitar defaults que provoquen grafos completos accidentalmente.

Preferencia:

```text
to-one → configurable
to-many → lazy/explicit
```

con optimización basada en contexto.

---

# 78. Lazy loading

Lazy loading deberá estar controlado y observable.

No será:

```text
magic hidden database I/O everywhere
```

---

# 79. Lazy-loading boundary

Una propiedad lazy podrá requerir:

```text
active ORM context
```

o una estrategia proxy/collection claramente definida.

---

# 80. Detached entity

Acceso lazy sobre entidad detached deberá producir comportamiento explícito.

No reconexión mágica.

---

# 81. N+1

El ORM deberá integrarse con:

```text
N+1 Detection System
```

definido posteriormente.

---

# 82. Eager loading

Ejemplo:

```php
User::query()
    ->with('profile', 'roles')
    ->get();
```

deberá traducirse a una estrategia de carga.

No necesariamente a un único JOIN.

---

# 83. Eager loading strategy

El ORM podrá elegir entre:

```text
JOIN
SELECT IN
BATCH
SUBQUERY
```

dependiendo de semántica y capacidades.

---

# 84. Query planner boundary

El ORM puede decidir:

```text
relationship loading strategy
```

pero el Query Planner decide:

```text
physical database query plan
```

Son niveles diferentes.

---

# 85. Type system integration

El ORM reutilizará el Database Type System.

```text
PHP Value
    ↓
ORM Field Mapping
    ↓
Database Logical Type
    ↓
Value Conversion
    ↓
Query Parameter
```

---

# 86. ORM casts

Los casts de developer experience no deberán crear un segundo sistema incompatible de tipos.

---

# 87. Value Objects

Ejemplo:

```php
final readonly class Email
{
    public function __construct(
        public string $value,
    ) {}
}
```

podrá mapearse:

```text
Email
 ↓
VARCHAR
```

mediante:

```text
ValueObjectMapping
```

---

# 88. Embedded objects

Podrá soportarse:

```php
final class Address
{
    private string $street;
    private string $city;
}
```

embebido en:

```text
users.address_street
users.address_city
```

sin identidad ORM independiente.

---

# 89. Entity ≠ Value Object

Entity:

```text
identity-based
```

Value Object:

```text
value-based
```

---

# 90. Lifecycle

Se modelará un Entity Lifecycle explícito.

Eventos potenciales:

```text
prePersist
postPersist
preUpdate
postUpdate
preRemove
postRemove
postLoad
```

---

# 91. Lifecycle callbacks

Deberán utilizarse con moderación.

No deberán convertirse en un mecanismo oculto de business orchestration.

---

# 92. Lifecycle callback ≠ Domain Event

Son conceptos distintos.

---

# 93. ORM events

El ORM podrá producir eventos estructurados hacia el Event System.

---

# 94. Domain events

Las entidades podrán producir domain events independientemente del ORM.

El ORM no deberá hacerlos equivalentes automáticamente.

---

# 95. Event timing

Deberá distinguirse:

```text
before persistence planning
after SQL execution
after flush
after transaction commit
```

---

# 96. PostPersist ≠ committed

Un evento después del INSERT no significa necesariamente:

```text
transaction committed
```

---

# 97. Transaction integration

ORM consumirá:

```text
TransactionManager
```

pero no será su propietario arquitectónico.

---

# 98. Ejemplo

```php
$transactions->run(function () use ($entityManager) {
    $entityManager->persist($order);
    $entityManager->flush();
});
```

---

# 99. Flush failure

Si flush falla:

```text
Database Transaction State
```

y:

```text
UnitOfWork State
```

deberán reconciliarse explícitamente.

---

# 100. No false synchronization

Después de una ejecución con:

```text
UNKNOWN_OUTCOME
```

el ORM no deberá marcar silenciosamente todas las entidades como sincronizadas.

---

# 101. Outcome certainty

Se reutilizará:

```text
CERTAIN_SUCCESS
CERTAIN_FAILURE
PARTIAL
UNKNOWN
```

del Execution System.

---

# 102. Persistence synchronization

Solo:

```text
CERTAIN_SUCCESS
```

permite confirmar ciertos cambios ORM como persistidos sin reconciliación adicional.

---

# 103. Generated identifiers

Para:

```text
auto increment
sequence
identity
database-generated UUID
```

el Persistence Engine deberá recuperar IDs mediante Result System/capabilities.

---

# 104. Generated ID lifecycle

```text
NEW Entity
    ↓
INSERT
    ↓
Generated Identifier
    ↓
Entity Identifier Assignment
    ↓
Identity Map Registration
```

---

# 105. ID assignment failure

No deberá dejar Identity Map inconsistente.

---

# 106. Optimistic locking

El ORM deberá poder utilizar:

```text
version
```

o:

```text
timestamp
```

como mecanismo de optimistic locking.

---

# 107. Ejemplo

```text
UPDATE users
SET name = ?, version = 8
WHERE id = ?
AND version = 7
```

conceptualmente será producido mediante Query Model.

---

# 108. Affected rows

Si:

```text
affectedRows = 0
```

puede significar:

```text
OptimisticLockConflict
```

cuando el plan de persistencia así lo define.

---

# 109. Pessimistic locking

Será integración con Transaction/Query Locking.

No será implementado como lógica especial del Entity object.

---

# 110. Entity state machine

Conceptualmente:

```text
             persist()
  NEW ───────────────────► MANAGED
                             │
                             │ changes
                             ▼
                           DIRTY
                             │
                             │ flush
                             ▼
                          MANAGED
                             │
                             │ remove()
                             ▼
                          REMOVED
                             │
                             │ flush
                             ▼
                          DETACHED
```

Los estados exactos se formalizarán en el documento 121.

---

# 111. Detached entities

Una entidad detached:

```text
exists as PHP object
```

pero no pertenece al contexto ORM actual.

---

# 112. clear()

```php
$entityManager->clear();
```

deberá:

```text
clear IdentityMap
clear UnitOfWork tracking
release ORM-scoped references
```

sin cerrar arbitrariamente conexiones propiedad de otras capas.

---

# 113. EntityManager scope

El EntityManager operativo será:

```text
request-scoped
operation-scoped
job-scoped
```

según runtime.

Nunca process-global mutable.

---

# 114. EntityManagerFactory

Se propone:

```php
interface EntityManagerFactory
{
    public function create(
        DatabaseContext $context,
    ): EntityManager;
}
```

---

# 115. Shared immutable ORM infrastructure

Podrá compartirse:

```text
Compiled Metadata
Metadata Registry
Mapping Definitions
Type Definitions
Hydration Templates
Repository Definitions
Frozen Extension Registries
```

---

# 116. Mutable scoped ORM infrastructure

No deberá compartirse entre requests:

```text
EntityManager
UnitOfWork
IdentityMap
Entity Snapshots
Managed Entities
ChangeSets
Pending Operations
Lazy-loading Sessions
Persistence Execution State
```

---

# 117. Request lifecycle

```text
Request Start
      ↓
DatabaseContext
      ↓
EntityManager
      ↓
IdentityMap + UnitOfWork
      ↓
Application
      ↓
Optional flush
      ↓
clear/reset
      ↓
Request End
```

---

# 118. Job lifecycle

Para jobs:

```text
Job Start
   ↓
ORM Scope
   ↓
Job Processing
   ↓
flush
   ↓
clear/reset
   ↓
Job End
```

---

# 119. Long-running processing

Para millones de entidades:

```php
foreach ($users as $user) {
    // process

    if (++$count % 1000 === 0) {
        $entityManager->flush();
        $entityManager->clear();
    }
}
```

deberá evitar crecimiento ilimitado del Identity Map.

---

# 120. Memory governance

El ORM deberá integrarse con:

```text
Resource Governance System
```

para detectar:

```text
too many managed entities
large snapshots
oversized hydration graphs
unbounded relationship collections
```

---

# 121. Query streaming + ORM

Streaming de resultados ORM requiere especial cuidado.

```text
Database Cursor
      ↓
Row
      ↓
Hydration
      ↓
Managed Entity
```

puede seguir haciendo crecer Identity Map.

---

# 122. Streaming strategy

Se deberán permitir estrategias como:

```text
TRACKED
DETACHED_AFTER_YIELD
READ_ONLY
SCALAR
PROJECTION
```

---

# 123. Read-only entities

El ORM podrá soportar un modo:

```text
READ_ONLY
```

para consultas donde no se necesita change tracking.

---

# 124. Read-only optimization

Puede omitir:

```text
snapshots
dirty checking
UnitOfWork registration
```

cuando semánticamente sea seguro.

---

# 125. Projection

No toda consulta ORM debe devolver entidades.

Podrá devolver:

```text
DTO
Scalar
Tuple
Array
Projection Object
```

---

# 126. Projection ≠ partial entity

Preferir:

```text
UserSummary
```

a una entidad `User` incompleta cuando no se requiere identidad gestionada.

---

# 127. ORM Query Result

Se propone:

```text
EntityResult
ProjectionResult
ScalarResult
TupleResult
```

sobre la infraestructura del `Database Result System`.

---

# 128. Metadata registry

Se propone:

```php
interface EntityMetadataRegistry
{
    public function get(string $entityClass): EntityMetadata;

    public function has(string $entityClass): bool;
}
```

---

# 129. Metadata registry immutability

En producción deberá poder:

```text
compile
freeze
share
```

sin mutación durante requests.

---

# 130. Metadata validation

Antes de runtime deberá validar:

```text
entity identifier
table mapping
field collisions
type compatibility
relationship mapping
ownership
join columns
cascade definitions
value objects
inheritance
locking
extensions
```

---

# 131. Mapping errors fail early

Preferencia:

```text
boot/compile time
```

en lugar de descubrir errores después de tráfico real.

---

# 132. Schema awareness

ORM Metadata podrá compararse contra:

```text
Schema Model
```

para diagnóstico.

Pero:

```text
ORM Mapping
≠
Database Schema
```

---

# 133. Schema validation

Podrá existir:

```text
OrmSchemaValidator
```

que compruebe:

```text
mapping
vs
observed schema
```

sin convertir automáticamente el ORM en Schema System.

---

# 134. ORM-generated migrations

En el futuro:

```text
ORM Metadata
      ↓
Desired Schema Model
      ↓
Schema Diff
      ↓
Migration Candidate
```

podrá facilitar migrations.

Pero:

> **El ORM no deberá ejecutar automáticamente cambios de esquema al detectar diferencias.**

---

# 135. Mapping → desired schema

El mapping podrá proyectarse a:

```text
Schema Model
```

mediante un adaptador explícito:

```text
OrmSchemaProjector
```

---

# 136. Schema remains authoritative operationally

Las migrations continúan siendo el mecanismo gobernado de evolución.

---

# 137. Inheritance

El ORM podrá contemplar:

```text
SINGLE_TABLE
JOINED
TABLE_PER_CLASS
```

solo cuando existan contratos claros.

---

# 138. Inheritance cost

Las estrategias complejas no deberán habilitarse sin explicar sus consecuencias de:

```text
joins
nullability
query complexity
persistence
schema evolution
```

---

# 139. Default inheritance

Preferencia:

```text
no implicit inheritance mapping
```

---

# 140. Polymorphism

Polymorphic relationships podrán soportarse como extensión ORM explícita.

---

# 141. Polymorphic ≠ Foreign Key

Muchos esquemas polimórficos no pueden representar integridad completa mediante una FK tradicional.

El ORM deberá comunicar esta diferencia.

---

# 142. Soft deletes

Soft Delete será capability/extension del ORM, no comportamiento universal oculto.

---

# 143. Global scopes

Deberán tratarse cuidadosamente.

Ejemplo:

```text
SoftDeleteScope
TenantScope
SecurityScope
```

No deberán convertirse en filtros imposibles de auditar.

---

# 144. Query scopes

Toda transformación automática de query deberá ser:

```text
typed
registered
inspectable
disableable where policy permits
deterministic
```

---

# 145. Tenant scopes

Multitenancy seguirá siendo integración opcional.

El ORM core no asumirá:

```text
tenant_id
```

universal.

---

# 146. Multitenancy integration

Cuando el paquete esté instalado:

```text
TenantContext
      ↓
EntityManagerFactory
      ↓
DatabaseContext
      ↓
EntityManager
```

---

# 147. Tenant Identity Map

La identidad efectiva deberá incluir el contexto necesario.

Conceptualmente:

```text
Tenant A + User#10
≠
Tenant B + User#10
```

---

# 148. No tenant leakage

Una entidad gestionada bajo Tenant A jamás podrá reutilizarse automáticamente bajo Tenant B.

---

# 149. Connection routing

El ORM no resolverá directamente:

```text
primary
replica
shard
tenant connection
```

Delegará a Database Context/Connection Routing.

---

# 150. Read/write awareness

Entity Query podrá declarar:

```text
READ
```

y Persistence:

```text
WRITE
```

pero el routing final pertenece a Connection/Distribution System.

---

# 151. Sticky reads

Después de write, read consistency puede requerir primary.

Esto será gestionado por Read/Write Routing, no por hacks ORM.

---

# 152. Cache integration

ORM podrá integrarse posteriormente con:

```text
Entity Cache
Metadata Cache
Query Cache
Result Cache
```

pero:

```text
IdentityMap
≠
EntityCache
```

---

# 153. Cache consistency

Cache no deberá modificar la semántica de UnitOfWork.

---

# 154. Entity cache lookup

Conceptualmente:

```text
IdentityMap
   ↓ miss
Entity Cache
   ↓ miss
Database
```

pero la estrategia exacta será definida en el bloque Cache.

---

# 155. Event integration

ORM deberá producir eventos hacia:

```text
VoltStack Event System
```

mediante una integración opcional/contractual.

---

# 156. Telemetry integration

Se deberán observar:

```text
entity loads
entity hydration
IdentityMap hits
UnitOfWork size
dirty checking
flush duration
persistence operations
relationship loads
N+1
metadata cache
```

---

# 157. Telemetry ≠ ORM semantics

Desactivar Telemetry no deberá cambiar el comportamiento ORM.

---

# 158. Security

ORM deberá preservar las garantías de parameter binding.

Nunca:

```php
User::where("name = '$input'");
```

como mecanismo normal.

---

# 159. Field resolution

Preferir:

```php
User::query()->where('name', $input);
```

que produzca:

```text
EntityFieldReference
+
BoundParameter
```

---

# 160. Raw expressions

Raw Query Escape Hatches seguirán las reglas del documento 53.

El ORM no creará un bypass adicional.

---

# 161. Mass assignment

La ergonomía Model API podrá tener:

```text
fillable
guarded
```

o una solución equivalente.

Pero:

```text
Mass Assignment Protection
≠
Authorization
```

---

# 162. Domain invariants

ORM no deberá saltarse silenciosamente invariantes de dominio al modificar entidades.

---

# 163. Hydration exception

Hydration puede necesitar acceso controlado a estado interno, pero:

```text
Hydration Mechanism
≠
Application Mutation API
```

---

# 164. Reflection writes

Si se utilizan para hydration:

- estarán encapsulados;
- serán compilables/cacheables;
- no serán API pública;
- deberán respetar readonly/value-object strategies definidas.

---

# 165. PHP readonly

El ORM deberá tener una política explícita para:

```php
readonly
```

y propiedades promovidas/inmutables.

No deberá depender de hacks no soportados.

---

# 166. Immutable entities

La arquitectura podrá soportarlas mediante estrategias específicas.

Pero UnitOfWork deberá comprender reemplazos de instancia y cambios de identidad cuidadosamente.

---

# 167. Mutable entities

Serán el modelo inicial más sencillo para managed entities.

---

# 168. Domain purity spectrum

VoltStack deberá permitir:

```text
Rich Domain Entity
       ↕
POPO Entity
       ↕
VoltStack Model
```

sin exigir que todas las aplicaciones adopten DDD.

---

# 169. Base Model

Podrá existir:

```php
abstract class Model
```

para Developer Experience.

---

# 170. Model internals

`Model` podrá ofrecer:

```text
query()
find()
save()
delete()
refresh()
relations
casts
serialization helpers
```

pero delegará persistencia.

---

# 171. Model static APIs

Una llamada:

```php
User::find(10);
```

deberá resolver un EntityManager contextual mediante una fachada/bridge controlado.

No mediante un singleton global mutable.

---

# 172. Model context

Conceptualmente:

```text
Model Static API
      ↓
ModelContextResolver
      ↓
Current Request DatabaseContext
      ↓
EntityManager
```

---

# 173. Static API ≠ static state

Regla:

> **VoltStack puede ofrecer sintaxis estática sin almacenar estado ORM global estático.**

---

# 174. Dependency Injection API

Siempre deberá existir alternativa:

```php
public function __construct(
    private UserRepository $users,
) {}
```

---

# 175. Facade API

Podrá existir:

```php
ORM::manager();
```

pero resolverá contexto scoped desde Container.

---

# 176. ORM configuration

Conceptualmente:

```php
'orm' => [
    'mapping' => [
        'attributes' => true,
    ],

    'change_tracking' => 'snapshot',

    'metadata' => [
        'compile' => true,
        'cache' => true,
    ],

    'lazy_loading' => true,

    'strict' => true,
];
```

---

# 177. Secure/strict defaults

Se recomienda:

```text
strict metadata validation
no silent type fallback
no hidden cross-tenant state
parameterized queries
bounded managed entity warnings
explicit raw operations
explicit partial entities
```

---

# 178. No hidden schema writes

Nunca:

```text
Entity mapping changed
        ↓
ORM automatically ALTER TABLE production
```

---

# 179. No hidden flush

Métodos de lectura no deberán provocar:

```text
flush()
```

silenciosamente.

---

# 180. Auto-flush

Si se ofrece alguna estrategia, deberá ser explícita y configurable.

Default recomendado:

```text
explicit flush semantics
```

para EntityManager API.

Model API podrá ofrecer `save()` como flush/persist convenience sobre la entidad específica según contrato documentado.

---

# 181. save() semantics

Debe definirse cuidadosamente.

Posible contrato:

```text
Model::save()
    ↓
register/update entity
    ↓
flush affected UnitOfWork scope
```

Pero se recomienda evitar que `save()` haga commit de transaction.

---

# 182. save() ≠ commit

Regla innegociable:

```text
save()
≠
transaction commit
```

---

# 183. delete() ≠ commit

Igualmente.

---

# 184. Bulk operations

Bulk:

```text
UPDATE users SET active = false ...
```

no deberá necesariamente hidratar miles de entidades.

---

# 185. Bulk ORM operations

Podrá existir:

```text
EntityBulkUpdate
EntityBulkDelete
```

traducidos directamente al Query Engine.

---

# 186. Bulk operations and UnitOfWork

Problema:

```text
DB state changes
but
managed entities remain stale
```

Por tanto, bulk operations deberán declarar una política:

```text
CLEAR_AFFECTED
REFRESH_AFFECTED
REJECT_IF_MANAGED
ALLOW_STALE_EXPLICITLY
```

---

# 187. No silent stale Identity Map

VoltStack no deberá permitir que bulk mutation deje silenciosamente managed entities consideradas actuales.

---

# 188. Native database features

ORM podrá aprovechar:

```text
RETURNING
UPSERT
ON CONFLICT
generated columns
JSON
window queries
CTEs
```

a través del Query Engine/capability model.

---

# 189. Capability-driven ORM

Nunca:

```php
if ($driver === 'pgsql') {
    ...
}
```

en lógica ORM central.

Preferir:

```php
if ($capabilities->supportsReturning()) {
    ...
}
```

---

# 190. Platform leakage

Detalles de plataforma deberán permanecer bajo:

```text
Query / Compiler / Platform
```

salvo capabilities semánticamente relevantes para ORM.

---

# 191. Error model

El ORM deberá traducir errores de capas inferiores solo cuando pueda añadir semántica ORM real.

---

# 192. Ejemplo

Un:

```text
unique constraint violation
```

puede conservar el Database Error original.

No deberá convertirse automáticamente en:

```text
"email already exists"
```

porque eso pertenece al dominio/aplicación.

---

# 193. Optimistic locking exception

Sí existe semántica ORM suficiente para:

```text
OptimisticLockException
```

---

# 194. Entity not found

Debe distinguirse:

```text
find() → null
```

de:

```text
getReference() / require()
→ EntityNotFoundException
```

según API.

---

# 195. ORM exception hierarchy

```text
DatabaseOrmException
├── EntityException
│   ├── EntityNotFoundException
│   ├── InvalidEntityException
│   └── EntityIdentityException
│
├── MetadataException
│   ├── EntityMetadataNotFoundException
│   ├── InvalidEntityMetadataException
│   ├── DuplicateEntityMappingException
│   └── MappingConflictException
│
├── EntityManagerException
│   ├── EntityManagerClosedException
│   ├── DetachedEntityException
│   └── InvalidEntityStateException
│
├── UnitOfWorkException
│   ├── UnitOfWorkConsistencyException
│   ├── ChangeTrackingException
│   └── PersistenceDependencyCycleException
│
├── HydrationException
│   ├── EntityHydrationException
│   ├── PartialEntityException
│   └── IdentityCollisionException
│
├── RelationshipException
│   ├── InvalidRelationshipException
│   ├── LazyLoadingException
│   └── RelationshipOwnershipException
│
├── PersistenceException
│   ├── EntityPersistenceException
│   ├── OptimisticLockException
│   ├── PersistenceUnknownOutcomeException
│   └── PersistenceConsistencyException
│
└── OrmInvariantException
```

---

# 196. Unknown outcome

`PersistenceUnknownOutcomeException` será especialmente importante.

Ejemplo:

```text
UPDATE sent
connection lost
database outcome unknown
```

El ORM no deberá fingir saber si:

```text
entity persisted
```

---

# 197. EntityManager recovery

Después de ciertos errores graves, EntityManager podrá quedar:

```text
TAINTED
```

o:

```text
CLOSED
```

para impedir continuar con estado inconsistente.

---

# 198. EntityManager lifecycle state

Conceptualmente:

```text
OPEN
TAINTED
CLOSED
```

---

# 199. TAINTED

Un manager tainted podrá permitir:

```text
diagnostics
clear
close
```

pero no necesariamente nuevos flushes.

---

# 200. Testing

El ORM deberá ser diseñado para:

```text
Unit Testing
Integration Testing
Mapping Testing
Persistence Testing
Hydration Testing
Relationship Testing
Concurrency Testing
Runtime Isolation Testing
```

---

# 201. Unit testing domain entities

Las entidades de dominio no deberán necesitar una base de datos para tests ordinarios.

---

# 202. Repository testing

Repositories podrán probarse contra:

```text
SQLite
test database
database fixture
fake/query spy
```

según nivel de prueba.

---

# 203. ORM fake

Un fake ORM no deberá fingir semánticas relacionales complejas que no puede reproducir.

---

# 204. Conformance tests

Drivers/plataformas deberán ejecutar matrices para comprobar comportamiento ORM sobre:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 205. Metadata tests

Deberán verificar:

```text
determinism
serialization
compilation
cache
invalid mappings
relationship cycles
identifier mappings
types
```

---

# 206. Identity Map tests

Deberán probar:

```text
same key → same object
different key → different object
tenant isolation
clear()
detached behavior
partial hydration
```

---

# 207. UnitOfWork tests

Deberán cubrir:

```text
new
managed
dirty
removed
detached
change sets
flush success
flush failure
unknown outcome
```

---

# 208. Persistent runtime tests

Especialmente:

```text
Request A
→ manage User#10

RESET

Request B
→ User#10 must not reuse Request A instance
```

---

# 209. Concurrency

EntityManager no deberá asumirse thread-safe/coroutine-safe si contiene estado mutable.

---

# 210. Concurrency ownership

Un EntityManager pertenece a:

```text
one logical ORM scope
```

y no deberá compartirse concurrentemente salvo contrato explícito.

---

# 211. Async execution

Si VoltStack incorpora ejecución concurrente:

```text
Task A
Task B
```

cada tarea deberá recibir scopes ORM compatibles con sus garantías.

---

# 212. ORM reset contract

Se propone:

```php
interface OrmResettable
{
    public function reset(): void;
}
```

implementado por coordinadores scoped apropiados.

---

# 213. Reset order

Conceptualmente:

```text
stop lazy operations
      ↓
resolve/cancel pending ORM work
      ↓
clear UnitOfWork
      ↓
clear IdentityMap
      ↓
release ORM resources
      ↓
release connection leases downstream
```

---

# 214. No implicit flush on reset

Regla crítica:

> **El final de un request nunca deberá hacer flush automático de cambios pendientes como efecto de cleanup.**

---

# 215. Unflushed changes

Podrán producir diagnóstico en development:

```text
ORM scope ended with unflushed managed changes
```

pero no persistirse silenciosamente.

---

# 216. Metadata cache

El cache de metadata podrá ser:

```text
process-shared
```

porque será inmutable.

---

# 217. Identity Map cache

No será process-shared.

---

# 218. ORM optimization

Áreas de optimización:

```text
compiled metadata
compiled property access
hydration plans
identity lookup
dirty checking
batch persistence
relationship batch loading
query compilation reuse
prepared statements
read-only mode
projections
```

---

# 219. Compiled property access

Reflection repetitiva podrá sustituirse por accessors compilados/cacheados.

---

# 220. Dirty checking optimization

Podrán existir:

```text
field-level snapshots
dirty flags
change notification
immutable value comparisons
```

sin modificar semántica.

---

# 221. Flush complexity

Naive:

```text
O(all managed entities)
```

puede ser costoso.

VoltStack deberá permitir estrategias que reduzcan trabajo cuando se conocen entidades modificadas.

---

# 222. UnitOfWork scale

Se deberá observar:

```text
managed entity count
new count
dirty count
removed count
snapshot memory
relationship change count
```

---

# 223. Batch persistence

Cuando sea posible:

```text
N entity inserts
```

podrán convertirse en operaciones batch.

Pero preservando:

```text
generated IDs
ordering
relationships
events
failure semantics
```

---

# 224. Batch optimization ≠ semantic change

Nunca sacrificar:

```text
correctness
```

por reducir round trips.

---

# 225. Extension architecture

Se propone:

```text
ORM Extension Registry
```

con extensiones para:

```text
metadata
mapping
types
repositories
hydration
persistence
query scopes
lifecycle
```

---

# 226. Extension registry

Deberá ser:

```text
registered during bootstrap
validated
dependency ordered
frozen
```

antes de requests.

---

# 227. No last-wins

Dos extensiones intentando registrar el mismo identificador crítico producirán error salvo composición explícita.

---

# 228. Extension boundaries

Una extensión ORM no deberá:

- generar SQL directamente;
- acceder a PDO directamente;
- mantener current entity manager global;
- alterar metadata congelada;
- saltarse UnitOfWork;
- saltarse parameter binding;
- ignorar tenant isolation;
- ocultar database I/O no declarado.

---

# 229. Package architecture

El ORM pertenecerá conceptualmente a:

```text
VoltStack\Quantum\Database\ORM
```

---

# 230. Namespace propuesto

```text
VoltStack\Quantum\Database\ORM
├── Contract
├── Entity
├── Model
├── Metadata
├── Mapping
├── Attribute
├── Manager
├── Repository
├── Query
├── State
├── Identity
├── UnitOfWork
├── Persistence
├── Hydration
├── Relationship
├── Type
├── Lifecycle
├── Locking
├── Runtime
├── Extension
├── Telemetry
└── Exception
```

---

# 231. Estructura propuesta

```text
src/
└── Quantum/
    └── Database/
        └── ORM/
            ├── Contract/
            │   ├── EntityManager.php
            │   ├── Repository.php
            │   ├── EntityMetadataRegistry.php
            │   └── EntityPersister.php
            │
            ├── Entity/
            │   ├── EntityType.php
            │   ├── EntityKey.php
            │   ├── EntityIdentifier.php
            │   └── EntityDescriptor.php
            │
            ├── Model/
            │   ├── Model.php
            │   ├── ModelContextResolver.php
            │   ├── ModelPersistenceBridge.php
            │   └── ModelQueryBridge.php
            │
            ├── Metadata/
            │   ├── EntityMetadata.php
            │   ├── CompiledEntityMetadata.php
            │   ├── EntityMetadataRegistry.php
            │   ├── MetadataLoader.php
            │   ├── MetadataCompiler.php
            │   └── MetadataValidator.php
            │
            ├── Mapping/
            │   ├── EntityMapping.php
            │   ├── FieldMapping.php
            │   ├── IdentifierMapping.php
            │   ├── RelationshipMapping.php
            │   ├── EmbeddedMapping.php
            │   └── ValueObjectMapping.php
            │
            ├── Attribute/
            │   ├── Entity.php
            │   ├── Table.php
            │   ├── Id.php
            │   ├── Column.php
            │   ├── OneToOne.php
            │   ├── OneToMany.php
            │   ├── ManyToOne.php
            │   └── ManyToMany.php
            │
            ├── Manager/
            │   ├── DefaultEntityManager.php
            │   ├── EntityManagerFactory.php
            │   └── EntityManagerState.php
            │
            ├── Repository/
            │   ├── EntityRepository.php
            │   ├── RepositoryResolver.php
            │   └── RepositoryFactory.php
            │
            ├── Query/
            │   ├── EntityQueryBuilder.php
            │   ├── EntityQuery.php
            │   ├── EntityQueryTranslator.php
            │   └── EntityFieldResolver.php
            │
            ├── Identity/
            │   └── IdentityMap.php
            │
            ├── State/
            │   ├── EntityState.php
            │   ├── EntitySnapshot.php
            │   └── ChangeSet.php
            │
            ├── UnitOfWork/
            │   ├── UnitOfWork.php
            │   ├── ChangeDetector.php
            │   └── EntityChangeGraph.php
            │
            ├── Persistence/
            │   ├── PersistenceEngine.php
            │   ├── PersistencePlanner.php
            │   ├── PersistencePlan.php
            │   ├── InsertPersister.php
            │   ├── UpdatePersister.php
            │   └── DeletePersister.php
            │
            ├── Hydration/
            │   ├── EntityHydrator.php
            │   ├── EntityInstantiator.php
            │   ├── HydrationPlan.php
            │   └── HydrationContext.php
            │
            ├── Relationship/
            │   ├── RelationshipMetadata.php
            │   ├── RelationshipLoader.php
            │   ├── LazyLoader.php
            │   └── BatchRelationshipLoader.php
            │
            ├── Lifecycle/
            │   ├── EntityLifecycle.php
            │   ├── LifecycleCallback.php
            │   └── LifecycleEventDispatcher.php
            │
            ├── Locking/
            │   ├── OptimisticLock.php
            │   └── PessimisticLock.php
            │
            ├── Runtime/
            │   ├── OrmScope.php
            │   ├── OrmContext.php
            │   └── OrmResetter.php
            │
            ├── Extension/
            │   ├── OrmExtension.php
            │   └── OrmExtensionRegistry.php
            │
            ├── Telemetry/
            │   └── OrmTelemetry.php
            │
            └── Exception/
                └── ...
```

---

# 232. Dependency model interno

```text
Model ──────────────┐
                    │
Repository ─────────┤
                    ▼
              EntityManager
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
    Metadata    IdentityMap    UnitOfWork
       │                         │
       │                         ▼
       │                  Persistence
       │                         │
       ├─────────────┐           │
       ▼             ▼           │
    Query         Hydration      │
       │             │           │
       └─────────────┴───────────┘
                     │
                     ▼
                Query Engine
```

---

# 233. Prohibición de dependencias inversas

Nunca:

```text
Query Compiler
    ↓
EntityManager
```

Nunca:

```text
Connection
    ↓
UnitOfWork
```

Nunca:

```text
Schema Model
    ↓
Entity
```

---

# 234. Public API

La API deberá soportar tanto:

```php
$user = User::find($id);
```

como:

```php
$user = $users->find($id);
```

y:

```php
$user = $entityManager->find(User::class, $id);
```

sin cambiar el motor subyacente.

---

# 235. Developer Experience

Objetivo:

```text
Laravel ergonomics
+
Doctrine architectural rigor
+
VoltStack persistent-runtime safety
+
VoltStack Query Engine
```

---

# 236. API progresiva

Nivel 1:

```php
User::find(1);
```

Nivel 2:

```php
User::query()->where(...)->get();
```

Nivel 3:

```php
$userRepository->query();
```

Nivel 4:

```php
$entityManager->persist($entity);
```

Nivel 5:

```php
$queryEngine->query(...);
```

sin obligar al desarrollador común a manejar las capas inferiores.

---

# 237. Escape hatch

Cuando ORM no sea apropiado:

```text
Entity Query
    ↓
Query Builder
    ↓
Raw Expression (explicit)
```

El desarrollador podrá bajar de nivel.

---

# 238. No ORM tax

Usar Query Builder directamente no deberá requerir inicializar:

```text
UnitOfWork
IdentityMap
Entity Metadata
```

si no son necesarios.

---

# 239. Coexistencia

Dentro de la misma aplicación:

```text
ORM
+
Query Builder
+
Raw Queries
```

podrán coexistir.

Pero deberán respetar consistencia del ORM cuando operen sobre datos ya gestionados.

---

# 240. External mutation problem

Si:

```php
$user = User::find(10);

DB::table('users')
    ->where('id', 10)
    ->update(['name' => 'Carlos']);
```

el `$user` administrado puede quedar stale.

---

# 241. Explicit synchronization

La arquitectura deberá ofrecer:

```text
refresh(entity)
clear(entity/type)
invalidate managed state
```

sin fingir sincronización automática imposible.

---

# 242. Architectural contracts

Los contratos principales serán:

```text
EntityManager
EntityMetadataRegistry
Repository
EntityQuery
IdentityMap
UnitOfWork
ChangeDetector
PersistenceEngine
PersistencePlanner
EntityHydrator
EntityInstantiator
RelationshipLoader
OrmContext
OrmResetter
```

---

# 243. Invariantes arquitectónicas

## DB-ORM-ARCH-001
ORM será distinto de Query Builder.

## DB-ORM-ARCH-002
ORM será distinto de SQL Compiler.

## DB-ORM-ARCH-003
ORM será distinto de Execution Engine.

## DB-ORM-ARCH-004
ORM será distinto de Driver.

## DB-ORM-ARCH-005
ORM será distinto de Schema System.

## DB-ORM-ARCH-006
ORM será distinto de Migration System.

## DB-ORM-ARCH-007
ORM será distinto de EntityManager.

## DB-ORM-ARCH-008
ORM será distinto de Repository.

## DB-ORM-ARCH-009
ORM será distinto de Active Record.

## DB-ORM-ARCH-010
ORM será distinto de UnitOfWork.

## DB-ORM-ARCH-011
ORM será distinto de IdentityMap.

## DB-ORM-ARCH-012
ORM será distinto de Hydrator.

## DB-ORM-ARCH-013
Entity será distinta de database row.

## DB-ORM-ARCH-014
Entity será distinta de Query.

## DB-ORM-ARCH-015
Entity nunca generará SQL.

## DB-ORM-ARCH-016
Model API será distinta de Persistence Engine.

## DB-ORM-ARCH-017
Repository será distinto de Query Builder.

## DB-ORM-ARCH-018
UnitOfWork será distinto de Transaction.

## DB-ORM-ARCH-019
IdentityMap será distinto de Cache.

## DB-ORM-ARCH-020
Hydration será distinta de Persistence.

## DB-ORM-ARCH-021
Mapping será distinto de Schema.

## DB-ORM-ARCH-022
ORM Relationship será distinta de Foreign Key.

## DB-ORM-ARCH-023
ORM validation será distinta de database constraint.

## DB-ORM-ARCH-024
Entity lifecycle será distinto de request lifecycle.

## DB-ORM-ARCH-025
flush será distinto de commit.

## DB-ORM-ARCH-026
persist será distinto de INSERT inmediato.

## DB-ORM-ARCH-027
remove será distinto de DELETE inmediato.

## DB-ORM-ARCH-028
ORM dependerá del Query Engine.

## DB-ORM-ARCH-029
Query Engine no dependerá del ORM.

## DB-ORM-ARCH-030
ORM nunca compilará SQL directamente.

## DB-ORM-ARCH-031
Persistence Engine producirá operaciones del Query Engine.

## DB-ORM-ARCH-032
Model API y Repository API compartirán ORM Core.

## DB-ORM-ARCH-033
Active Record será convenience API.

## DB-ORM-ARCH-034
Active Record no tendrá segundo Persistence Engine.

## DB-ORM-ARCH-035
Entity identity será distinta de PHP object identity.

## DB-ORM-ARCH-036
Entity identity será distinta de physical DB representation.

## DB-ORM-ARCH-037
Composite identifiers serán explícitos.

## DB-ORM-ARCH-038
Entity Metadata será first-class.

## DB-ORM-ARCH-039
Reflection será distinta de compiled metadata.

## DB-ORM-ARCH-040
Attributes serán distintos de runtime metadata.

## DB-ORM-ARCH-041
Metadata podrá compilarse.

## DB-ORM-ARCH-042
Compiled metadata será immutable.

## DB-ORM-ARCH-043
Metadata registry podrá compartirse si está frozen.

## DB-ORM-ARCH-044
EntityManager será scoped.

## DB-ORM-ARCH-045
EntityManager no será process-global mutable.

## DB-ORM-ARCH-046
EntityManager delegará responsabilidades internas.

## DB-ORM-ARCH-047
persist registrará intención ORM.

## DB-ORM-ARCH-048
flush sincronizará UnitOfWork.

## DB-ORM-ARCH-049
flush no hará commit implícito.

## DB-ORM-ARCH-050
UnitOfWork rastreará estado ORM.

## DB-ORM-ARCH-051
UnitOfWork no será transaction manager.

## DB-ORM-ARCH-052
IdentityMap garantizará identidad dentro del scope.

## DB-ORM-ARCH-053
IdentityMap no cruzará requests.

## DB-ORM-ARCH-054
IdentityMap no será second-level cache.

## DB-ORM-ARCH-055
ORM persistence state no necesitará almacenarse dentro de entity.

## DB-ORM-ARCH-056
ChangeSet será distinto de SQL UPDATE.

## DB-ORM-ARCH-057
Change tracking será configurable mediante estrategia explícita.

## DB-ORM-ARCH-058
Persistence ordering será dependency-aware.

## DB-ORM-ARCH-059
Persistence dependency cycles serán explícitos.

## DB-ORM-ARCH-060
EntityQuery no producirá SQL directamente.

## DB-ORM-ARCH-061
EntityQuery se traducirá al Query Model.

## DB-ORM-ARCH-062
No existirá un ORM SQL Builder paralelo.

## DB-ORM-ARCH-063
Repository trabajará con entities.

## DB-ORM-ARCH-064
Repository no será application service.

## DB-ORM-ARCH-065
Hydration utilizará Result System.

## DB-ORM-ARCH-066
Hydration consultará IdentityMap.

## DB-ORM-ARCH-067
Hydration no duplicará una managed entity con misma identidad.

## DB-ORM-ARCH-068
Partial entity será distinta de fully loaded entity.

## DB-ORM-ARCH-069
Un campo no cargado no será tratado automáticamente como null.

## DB-ORM-ARCH-070
Projection será preferible a partial entity cuando corresponda.

## DB-ORM-ARCH-071
ORM Relationship será metadata explícita.

## DB-ORM-ARCH-072
ORM Cascade será distinta de database cascade.

## DB-ORM-ARCH-073
Orphan removal será distinto de database cascade.

## DB-ORM-ARCH-074
Lazy loading será observable.

## DB-ORM-ARCH-075
Lazy loading no reabrirá contexto mágico sobre detached entities.

## DB-ORM-ARCH-076
Eager loading no implicará necesariamente JOIN único.

## DB-ORM-ARCH-077
Relationship loading strategy será distinta de physical query planning.

## DB-ORM-ARCH-078
ORM reutilizará Database Type System.

## DB-ORM-ARCH-079
ORM casts no crearán un segundo type system.

## DB-ORM-ARCH-080
Value Object será distinto de Entity.

## DB-ORM-ARCH-081
Embedded object no tendrá identidad ORM implícita.

## DB-ORM-ARCH-082
Lifecycle callback será distinto de Domain Event.

## DB-ORM-ARCH-083
PostPersist será distinto de transaction committed.

## DB-ORM-ARCH-084
ORM consumirá Transaction Manager.

## DB-ORM-ARCH-085
ORM no reemplazará Transaction Manager.

## DB-ORM-ARCH-086
Flush failure deberá reconciliar UnitOfWork.

## DB-ORM-ARCH-087
UNKNOWN_OUTCOME no producirá false synchronization.

## DB-ORM-ARCH-088
Generated IDs se resolverán mediante capacidades/resultados estructurados.

## DB-ORM-ARCH-089
Generated ID assignment mantendrá IdentityMap consistente.

## DB-ORM-ARCH-090
Optimistic locking será first-class.

## DB-ORM-ARCH-091
Pessimistic locking será integración con Query/Transaction.

## DB-ORM-ARCH-092
Detached entity no pertenecerá al UnitOfWork actual.

## DB-ORM-ARCH-093
clear() no cerrará recursos ajenos al ORM.

## DB-ORM-ARCH-094
EntityManagerFactory recibirá contexto explícito.

## DB-ORM-ARCH-095
Managed entities no serán shared immutable infrastructure.

## DB-ORM-ARCH-096
Entity snapshots serán scoped.

## DB-ORM-ARCH-097
Pending persistence operations serán scoped.

## DB-ORM-ARCH-098
Request end limpiará estado ORM.

## DB-ORM-ARCH-099
Request cleanup nunca hará implicit flush.

## DB-ORM-ARCH-100
Long-running jobs podrán clear periódicamente UnitOfWork.

## DB-ORM-ARCH-101
Streaming ORM tendrá política de tracking explícita.

## DB-ORM-ARCH-102
Read-only mode podrá omitir change tracking.

## DB-ORM-ARCH-103
Read-only optimization no cambiará semántica.

## DB-ORM-ARCH-104
ORM podrá devolver projections.

## DB-ORM-ARCH-105
Metadata errors deberán fallar temprano cuando sea posible.

## DB-ORM-ARCH-106
ORM Mapping será distinto de observed schema.

## DB-ORM-ARCH-107
ORM podrá proyectar desired schema mediante adaptador explícito.

## DB-ORM-ARCH-108
ORM nunca alterará schema automáticamente en producción.

## DB-ORM-ARCH-109
Schema evolution seguirá gobernada por Migration System.

## DB-ORM-ARCH-110
Inheritance mapping será explícito.

## DB-ORM-ARCH-111
Polymorphic relationship será distinta de FK.

## DB-ORM-ARCH-112
Soft Delete no será comportamiento universal oculto.

## DB-ORM-ARCH-113
Global query scopes serán inspectables.

## DB-ORM-ARCH-114
Multitenancy no será dependencia obligatoria del ORM core.

## DB-ORM-ARCH-115
Tenant EntityManager state estará aislado.

## DB-ORM-ARCH-116
Tenant A Entity#X será distinta de Tenant B Entity#X.

## DB-ORM-ARCH-117
ORM no resolverá conexiones mediante vendor conditionals.

## DB-ORM-ARCH-118
Read/write routing pertenecerá a Distribution/Connection layers.

## DB-ORM-ARCH-119
IdentityMap será distinta de Entity Cache.

## DB-ORM-ARCH-120
Telemetry no cambiará semántica ORM.

## DB-ORM-ARCH-121
ORM conservará parameter binding.

## DB-ORM-ARCH-122
Raw expressions usarán escape hatch existente.

## DB-ORM-ARCH-123
Mass assignment protection será distinta de Authorization.

## DB-ORM-ARCH-124
Hydration internals no serán application mutation API.

## DB-ORM-ARCH-125
Readonly PHP tendrá estrategia explícita.

## DB-ORM-ARCH-126
Static Model API no implicará static ORM state.

## DB-ORM-ARCH-127
Dependency Injection API siempre estará disponible.

## DB-ORM-ARCH-128
save() será distinto de commit().

## DB-ORM-ARCH-129
delete() será distinto de commit().

## DB-ORM-ARCH-130
Bulk operations podrán evitar hydration.

## DB-ORM-ARCH-131
Bulk mutation deberá reconciliar managed state.

## DB-ORM-ARCH-132
Bulk mutation no dejará stale managed state silenciosamente.

## DB-ORM-ARCH-133
ORM capabilities serán capability-driven.

## DB-ORM-ARCH-134
Vendor conditionals no dominarán ORM core.

## DB-ORM-ARCH-135
Database errors solo se traducirán cuando exista semántica ORM adicional.

## DB-ORM-ARCH-136
EntityManager podrá quedar tainted después de outcome uncertainty.

## DB-ORM-ARCH-137
EntityManager state será explícito.

## DB-ORM-ARCH-138
Entities serán unit-testable sin DB cuando su dominio lo permita.

## DB-ORM-ARCH-139
ORM fake no fingirá semánticas que no implementa.

## DB-ORM-ARCH-140
Persistent-runtime isolation será testeada.

## DB-ORM-ARCH-141
EntityManager mutable no se asumirá concurrency-safe.

## DB-ORM-ARCH-142
ORM reset será explícito.

## DB-ORM-ARCH-143
Compiled metadata podrá ser process-shared.

## DB-ORM-ARCH-144
IdentityMap nunca será process-shared.

## DB-ORM-ARCH-145
ORM optimizations preservarán correctness.

## DB-ORM-ARCH-146
Batch persistence preservará entity semantics.

## DB-ORM-ARCH-147
Extension registry será frozen.

## DB-ORM-ARCH-148
Duplicate critical extension registrations serán error.

## DB-ORM-ARCH-149
Extensions no generarán SQL directamente.

## DB-ORM-ARCH-150
Extensions no accederán directamente al driver.

## DB-ORM-ARCH-151
Extensions no mantendrán global current EntityManager.

## DB-ORM-ARCH-152
Extensions no alterarán frozen metadata.

## DB-ORM-ARCH-153
Extensions no saltarán UnitOfWork para managed persistence.

## DB-ORM-ARCH-154
Extensions no romperán tenant isolation.

## DB-ORM-ARCH-155
Query Builder podrá utilizarse sin ORM overhead innecesario.

## DB-ORM-ARCH-156
ORM, Query Builder y Raw Queries podrán coexistir.

## DB-ORM-ARCH-157
External mutations deberán poder invalidar/refresh managed state.

## DB-ORM-ARCH-158
No se asumirá sincronización automática después de external mutation.

## DB-ORM-ARCH-159
FrankenPHP request boundaries aislarán estado ORM.

## DB-ORM-ARCH-160
La arquitectura ORM mantendrá una sola fuente de verdad para persistencia de entidades: UnitOfWork + Persistence Engine sobre Query Engine.

---

# 244. Anti-patterns

## 244.1 SQL dentro del Model

Incorrecto:

```php
class User
{
    public function save()
    {
        PDO::query(...);
    }
}
```

---

## 244.2 SQL dentro del Repository

Incorrecto como arquitectura ORM principal:

```php
return $pdo->query(
    'SELECT * FROM users'
);
```

---

## 244.3 Segundo Query Builder ORM

Incorrecto:

```text
Database Query Builder
+
ORM SQL Builder
```

---

## 244.4 Global EntityManager

Incorrecto:

```php
static EntityManager $manager;
```

en FrankenPHP.

---

## 244.5 Global IdentityMap

Especialmente peligroso:

```php
static array $entities = [];
```

---

## 244.6 Active Record como segundo ORM

Incorrecto:

```text
Model::save()
→ ActiveRecordPersistenceEngine

EntityManager::flush()
→ DataMapperPersistenceEngine
```

Correcto:

```text
Model API ─────┐
               ▼
          ORM Core
               ▲
Repository ────┘
```

---

## 244.7 Entity = array glorificado

El ORM no deberá limitar el modelo de entidad a:

```php
$model->attributes['name'];
```

aunque Model API pueda ofrecer acceso dinámico opcional.

---

## 244.8 Every query returns entities

Incorrecto.

Para reporting:

```text
Projection / Scalar / DTO
```

puede ser superior.

---

## 244.9 Hidden N+1

Incorrecto:

```php
foreach ($users as $user) {
    echo $user->profile->country;
}
```

generando miles de queries sin diagnostics.

---

## 244.10 Auto-flush at request end

Incorrecto y peligroso:

```text
Request finished
→ flush everything automatically
```

---

## 244.11 save() = commit()

Rompe composición transaccional.

---

## 244.12 IdentityMap = cache global

Produce contaminación entre requests/tenants.

---

## 244.13 Mapping = schema

Cambiar un attribute ORM no significa que la base haya cambiado.

---

## 244.14 Automatic production schema synchronization

Debe evitarse.

---

## 244.15 ORM cascade = database cascade

Puede generar comportamiento incorrecto y doble ejecución.

---

## 244.16 Partial entity with null placeholders

Destruye la diferencia:

```text
not loaded
vs
NULL
```

---

## 244.17 Hidden vendor branches

Incorrecto:

```php
switch ($driver) {
    case 'mysql':
    case 'pgsql':
}
```

disperso por ORM.

---

## 244.18 Continue after unknown outcome

Incorrecto:

```text
flush failed with UNKNOWN_OUTCOME
→ keep using UnitOfWork normally
```

---

# 245. Ejemplo completo — lectura

```php
$user = User::find(42);
```

flujo:

```text
User::find(42)
      ↓
ModelContextResolver
      ↓
EntityManager
      ↓
EntityMetadata(User)
      ↓
EntityKey(User, 42)
      ↓
IdentityMap
      ├── HIT
      │    ↓
      │   User
      │
      └── MISS
           ↓
      EntityLoader
           ↓
      Entity Query
           ↓
      Query Model
           ↓
      Query Engine
           ↓
      Execution Engine
           ↓
      Result
           ↓
      Entity Hydrator
           ↓
      User
           ↓
      IdentityMap registration
           ↓
      UnitOfWork registration
           ↓
      snapshot
           ↓
      return User
```

---

# 246. Ejemplo completo — modificación

```php
$user = User::find(42);

$user->rename('Ana');

$user->save();
```

flujo:

```text
Managed User
     ↓
Domain mutation
     ↓
Current state differs
     ↓
Model::save()
     ↓
ModelPersistenceBridge
     ↓
EntityManager
     ↓
UnitOfWork
     ↓
Change Detection
     ↓
ChangeSet
     ↓
Persistence Planner
     ↓
Update Operation
     ↓
Query Model
     ↓
Query Engine
     ↓
Execution Engine
     ↓
Database
     ↓
CERTAIN_SUCCESS
     ↓
Snapshot refresh
     ↓
Entity remains MANAGED
```

---

# 247. Ejemplo — transacción

```php
$transactions->run(function () use ($entityManager, $order) {
    $entityManager->persist($order);

    $entityManager->flush();

    $this->inventory->reserve($order);

    $entityManager->flush();
});
```

Conceptualmente:

```text
BEGIN
  │
  ├── ORM flush #1
  │
  ├── domain/application operation
  │
  └── ORM flush #2
  │
COMMIT
```

Esto demuestra:

```text
flush()
≠
commit()
```

---

# 248. Ejemplo — Repository

```php
final class UserRepository extends EntityRepository
{
    public function findActiveByEmail(Email $email): ?User
    {
        return $this->query()
            ->where('email', $email)
            ->where('active', true)
            ->first();
    }
}
```

Internamente:

```text
Repository
   ↓
Entity Query
   ↓
Metadata Resolution
   ↓
Query Model
   ↓
Query Engine
   ↓
Hydration
```

---

# 249. Ejemplo — projection

```php
$users = User::query()
    ->select([
        'id',
        'name',
        'email',
    ])
    ->as(UserSummary::class)
    ->get();
```

resultado:

```text
UserSummary[]
```

no:

```text
partially managed User[]
```

---

# 250. Ejemplo — eager loading

```php
$orders = Order::query()
    ->with([
        'customer',
        'items.product',
    ])
    ->get();
```

ORM:

```text
Relationship Graph
       ↓
Loading Planner
       ↓
Possible strategy:

Query 1 → Orders
Query 2 → Customers WHERE id IN (...)
Query 3 → Items WHERE order_id IN (...)
Query 4 → Products WHERE id IN (...)
```

en vez de obligar a:

```text
one gigantic JOIN
```

---

# 251. Ejemplo — UnitOfWork

Estado inicial:

```text
IdentityMap
├── User#10
├── User#11
└── Order#30
```

Cambios:

```text
User#10 → DIRTY
User#11 → MANAGED
Order#30 → REMOVED
Order#31 → NEW
```

Flush:

```text
Change Detection
       ↓
Persistence Graph
       ↓
DELETE/INSERT/UPDATE ordering
       ↓
Query Models
       ↓
Execution
       ↓
State synchronization
```

---

# 252. Ejemplo — error

Suponga:

```text
UPDATE sent
↓
connection lost
↓
outcome unknown
```

Nunca:

```text
mark entity synchronized
```

Correcto:

```text
Persistence Outcome = UNKNOWN
       ↓
UnitOfWork consistency uncertain
       ↓
EntityManager = TAINTED
       ↓
reconciliation / clear / abort scope
```

---

# 253. Arquitectura final

```text
┌───────────────────────────────────────────────────────────┐
│                       APPLICATION                         │
└────────────────────────────┬──────────────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
       Laravel-like API              Repository / DDD API
              │                             │
              └──────────────┬──────────────┘
                             ▼
                       ENTITY MANAGER
                             │
       ┌─────────────────────┼─────────────────────┐
       │                     │                     │
       ▼                     ▼                     ▼
    Metadata             Identity Map          UnitOfWork
       │                     │                     │
       │                     │              ┌──────┴──────┐
       │                     │              ▼             ▼
       │                     │          ChangeSet      Entity State
       │                     │              │
       │                     └──────┬───────┘
       │                            ▼
       │                   Persistence Engine
       │                            │
       │                   Persistence Planner
       │                            │
       ├────────────────────────────┤
       │                            │
       ▼                            ▼
 Entity Query System         Persistence Operations
       │                            │
       └──────────────┬─────────────┘
                      ▼
                  QUERY ENGINE
                      │
              ┌───────┴────────┐
              ▼                ▼
          Optimizer         Planner
              │                │
              └───────┬────────┘
                      ▼
                  Compiler
                      │
                      ▼
              EXECUTION ENGINE
                      │
                      ▼
                CONNECTION
                      │
                      ▼
                   DRIVER
                      │
                      ▼
                  DATABASE
```

Hydration regresa por:

```text
Database
   ↓
Result
   ↓
Hydration Engine
   ↓
Identity Map
   ↓
UnitOfWork
   ↓
Entity
```

---

# 254. Fórmula arquitectónica

```text
VoltStack ORM
=
Entity Model
+
Compiled Metadata
+
Mapping
+
Model API
+
Repository API
+
Entity Manager
+
Entity Query
+
Identity Map
+
Unit of Work
+
Change Tracking
+
Persistence Planning
+
Persistence Engine
+
Hydration
+
Relationship Management
+
Lifecycle
+
Type Integration
+
Transaction Integration
+
Concurrency Control
+
Runtime Isolation
+
Extension Governance
```

---

# 255. Fórmula de persistencia

```text
EntityPersistence
=
EntityState
+
Metadata
+
ChangeSet
+
RelationshipDependencies
+
PersistencePlanning
+
QueryModelGeneration
+
ExecutionOutcome
+
StateReconciliation
```

---

# 256. Fórmula de identidad

Dentro de un ORM scope:

```text
∀ a,b:

EntityType(a) = EntityType(b)
∧
EntityIdentifier(a) = EntityIdentifier(b)

⇒

ManagedInstance(a) === ManagedInstance(b)
```

considerando además el Database/Tenant Context cuando forme parte del namespace efectivo de identidad.

---

# 257. Fórmula de flush

```text
flush(UoW)
=
DetectChanges
→
BuildPersistenceGraph
→
PlanOperations
→
GenerateQueryModels
→
Execute
→
VerifyOutcome
→
SynchronizeManagedState
```

---

# 258. Fórmula de seguridad de runtime

```text
SafeOrmScope
=
ScopedEntityManager
∧
ScopedIdentityMap
∧
ScopedUnitOfWork
∧
ScopedManagedEntities
∧
ImmutableSharedMetadata
∧
DeterministicReset
∧
NoCrossRequestState
```

---

# 259. Fórmula de API dual

```text
LaravelStyleModelAPI
+
RepositoryEntityManagerAPI
        ↓
     Same ORM Core
        ↓
Same UnitOfWork
+
Same IdentityMap
+
Same Persistence Engine
```

---

# 260. Regla maestra

> **El ORM de VoltStack convierte el modelo de objetos de la aplicación en operaciones de persistencia estructuradas, pero nunca convierte directamente entidades en SQL.**

La división final será:

```text
Entity
    ↓
"What domain object exists?"

Metadata
    ↓
"How is it mapped?"

EntityManager
    ↓
"What entities participate in this persistence context?"

IdentityMap
    ↓
"Which object instance represents each persistent identity?"

UnitOfWork
    ↓
"What changed?"

Persistence Planner
    ↓
"In what semantic order must those changes be persisted?"

Query Engine
    ↓
"What relational operations express those changes?"

SQL Compiler
    ↓
"How are those operations represented for this platform?"

Execution Engine
    ↓
"How are they executed safely?"

Driver
    ↓
"How do we communicate with the database?"
```

---

# 261. Resultado arquitectónico

Con esta arquitectura VoltStack podrá combinar:

```text
Laravel
├── approachable Model API
├── expressive querying
├── developer productivity
└── low entry barrier

Doctrine-style architecture
├── EntityManager
├── Repository
├── IdentityMap
├── UnitOfWork
├── explicit metadata
└── Data Mapper capabilities

VoltStack
├── Query AST
├── Semantic Query Engine
├── Optimizer
├── Planner
├── platform-aware Compiler
├── Execution Engine
├── capability model
├── persistent-runtime isolation
├── FrankenPHP-first runtime
└── optional integrations
```

sin heredar la necesidad de mantener motores separados.

La arquitectura resultante será:

```text
Developer-friendly outside
+
strictly layered inside
```

---

# 262. Estado del proyecto documental

Con:

```text
112_DATABASE_ORM_ARCHITECTURE.md
```

queda abierto:

```text
BLOCK 10 — ORM
```

El bloque será:

```text
112_DATABASE_ORM_ARCHITECTURE.md
113_DATABASE_ENTITY_MODEL.md
114_DATABASE_MODEL_API_SYSTEM.md
115_DATABASE_ENTITY_METADATA_SYSTEM.md
116_DATABASE_ENTITY_MAPPING_SYSTEM.md
117_DATABASE_ATTRIBUTE_MAPPING_SYSTEM.md
118_DATABASE_ENTITY_MANAGER_SYSTEM.md
119_DATABASE_REPOSITORY_SYSTEM.md
120_DATABASE_ENTITY_QUERY_SYSTEM.md
121_DATABASE_ENTITY_STATE_SYSTEM.md
122_DATABASE_ENTITY_LIFECYCLE_SYSTEM.md
```

Después se profundizará en:

```text
BLOCK 11
Identity Map
Unit of Work
Change Tracking
Persistence Engine
Flush
```

y posteriormente:

```text
BLOCK 12
Hydration
```

y:

```text
BLOCK 13
Relationships
```

Por tanto, este documento define las fronteras macro sin absorber las especificaciones detalladas que pertenecen a esos bloques.

---

# 263. Siguiente documento

```text
113_DATABASE_ENTITY_MODEL.md
```

Este documento definirá formalmente:

```text
Entity
Entity Type
Entity Identity
Entity Identifier
Entity Key
Entity Descriptor
Entity Class
Entity Instance
Entity Equality
Entity Mutability
Entity Construction
Entity Domain State
Persistent Identity
Transient Identity
Composite Identity
Generated Identity
Detached Identity
Entity References
Entity Proxies
Entity Runtime Boundaries
```

manteniendo como principio:

> **Una entidad es un objeto de dominio identificado por identidad persistente; no es una fila, un registro genérico, un contenedor de atributos ni un objeto SQL.**