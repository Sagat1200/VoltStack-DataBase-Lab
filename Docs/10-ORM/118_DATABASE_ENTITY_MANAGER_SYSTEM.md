# 118_DATABASE_ENTITY_MANAGER_SYSTEM.md

# VoltStack Quantum Database
## Database Entity Manager System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 118 — Database Entity Manager System  
**Bloque:** 10 — ORM  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Entity Manager System` define el componente de coordinación runtime del ORM de VoltStack responsable de administrar el **Persistence Context** de las entidades durante una operación, request, job o unidad lógica de trabajo.

Su función principal es ofrecer una API coherente para operaciones como:

```php
$entityManager->find(User::class, $id);

$entityManager->persist($user);

$entityManager->remove($user);

$entityManager->flush();

$entityManager->refresh($user);

$entityManager->clear();
```

sin convertir al `EntityManager` en:

- Query Builder;
- SQL Compiler;
- UnitOfWork;
- IdentityMap;
- Hydrator;
- Repository;
- TransactionManager;
- Connection;
- Driver;
- Service Locator.

Principio central:

> **EntityManager coordina el contexto de persistencia de las entidades; delega identidad, tracking, consultas, hidratación, persistencia y transacciones a subsistemas especializados.**

---

# 2. EntityManager ≠ ORM completo

Debe mantenerse:

```text
EntityManager
≠
ORM
```

El ORM completo contiene:

```text
ORM
├── Entity Model
├── Metadata
├── Mapping
├── EntityManager
├── Repository
├── Entity Query
├── IdentityMap
├── UnitOfWork
├── Change Tracking
├── Persistence Engine
├── Hydration
├── Relationships
├── Lifecycle
├── Types
└── Transaction Integration
```

`EntityManager` solamente coordina estas capacidades.

---

# 3. EntityManager ≠ UnitOfWork

Distinción fundamental:

```text
EntityManager
    ↓ coordinates
UnitOfWork
```

No:

```text
EntityManager
=
UnitOfWork
```

`UnitOfWork` mantiene información como:

```text
managed entities
new entities
removed entities
snapshots
change sets
pending persistence operations
```

El `EntityManager` expone operaciones de alto nivel sobre dicho sistema.

---

# 4. EntityManager ≠ IdentityMap

```text
EntityManager
    ↓ uses
IdentityMap
```

`IdentityMap` mantiene:

```text
EntityKey
    ↓
Entity Instance
```

y garantiza dentro del scope:

```text
same EntityKey
→
same managed object instance
```

---

# 5. EntityManager ≠ Repository

El Repository proporciona una API orientada a colección/consulta de entidades:

```php
$users = $entityManager
    ->repository(User::class)
    ->findBy(['active' => true]);
```

El `EntityManager` resuelve repositories.

No implementa internamente todos los métodos de consulta.

---

# 6. EntityManager ≠ Query Builder

Incorrecto:

```php
$entityManager
    ->select('*')
    ->from('users')
    ->where('id', 10);
```

Eso pertenece al Query System.

El EntityManager trabaja conceptualmente con:

```text
EntityType
EntityIdentifier
EntityQuery
Repository
EntityReference
```

---

# 7. EntityManager ≠ SQL Compiler

Nunca deberá existir:

```php
$entityManager->compileSql($entity);
```

La ruta correcta es:

```text
EntityManager
    ↓
Persistence Engine
    ↓
Query Model / AST
    ↓
Query Planner
    ↓
SQL Compiler
```

---

# 8. EntityManager ≠ Connection

El manager no deberá exponer como abstracción principal:

```php
$entityManager->pdo();
```

ni:

```php
$entityManager->mysqli();
```

Si una operación avanzada requiere acceso inferior deberá realizarse mediante APIs explícitas del Database System.

---

# 9. EntityManager ≠ TransactionManager

Debe mantenerse:

```text
flush()
≠
commit()
```

y:

```text
EntityManager
≠
TransactionManager
```

---

# 10. EntityManager ≠ Service Locator

Incorrecto:

```php
$entityManager->get(Cache::class);
$entityManager->get(Logger::class);
$entityManager->get(Mailer::class);
```

El EntityManager deberá exponer únicamente capacidades relacionadas con su dominio.

---

# 11. Posición arquitectónica

```text
Application
    │
    ▼
EntityManager
    │
    ├──────────────► MetadataRegistry
    │
    ├──────────────► RepositoryResolver
    │
    ├──────────────► IdentityMap
    │
    ├──────────────► UnitOfWork
    │
    ├──────────────► EntityLoader
    │
    ├──────────────► EntityRefresher
    │
    ├──────────────► PersistenceCoordinator
    │
    └──────────────► Lifecycle Coordinator
                            │
                            ▼
                         ORM Core
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

# 12. Dependencia general

La dirección deberá permanecer:

```text
Application
    ↓
EntityManager
    ↓
ORM Services
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
Driver
→
EntityManager
```

---

# 13. Responsabilidades

El EntityManager podrá coordinar:

```text
find entity
get reference
persist entity
remove entity
flush
refresh
detach
clear
contains
repository resolution
persistence-context inspection
entity state delegation
lifecycle coordination
```

---

# 14. No responsabilidades

No deberá:

```text
generate SQL
parse SQL
compile SQL
open raw sockets
implement driver protocols
diff schemas
run migrations
perform authorization
manage HTTP sessions
resolve tenants directly
maintain process-global entity instances
```

---

# 15. Persistence Context

El concepto central será:

```text
PersistenceContext
```

que representa el contexto lógico dentro del cual las entidades son:

```text
identified
managed
tracked
loaded
scheduled
synchronized
```

---

# 16. EntityManager y PersistenceContext

Conceptualmente:

```text
EntityManager
    ↓ coordinates
PersistenceContext
    ├── IdentityMap
    ├── UnitOfWork
    ├── EntityState access
    ├── Deferred Relationship State
    └── Persistence bookkeeping
```

---

# 17. PersistenceContext ≠ Transaction

Una transacción puede existir dentro de un PersistenceContext.

Pero:

```text
PersistenceContext
≠
Transaction
```

Un EntityManager puede permanecer abierto después de:

```text
commit
```

si el resultado es conocido y la política lo permite.

---

# 18. PersistenceContext ≠ Connection

El contexto tampoco deberá estar definido por una instancia física de conexión.

Debe depender de un:

```text
DatabaseContext
```

lógico.

---

# 19. DatabaseContext

El manager deberá crearse para un contexto determinado:

```text
DatabaseContext
├── logical target
├── tenant/shard context when applicable
├── routing policy
├── consistency context
└── runtime scope
```

---

# 20. EntityManagerFactory

La creación del manager pertenecerá a:

```text
EntityManagerFactory
```

---

# 21. Factory contract

Conceptualmente:

```php
interface EntityManagerFactory
{
    public function create(
        DatabaseContext $databaseContext,
    ): EntityManager;
}
```

---

# 22. Factory responsibilities

La factory deberá ensamblar:

```text
EntityManager
├── MetadataRegistry
├── IdentityMap
├── UnitOfWork
├── RepositoryResolver
├── EntityLoader
├── EntityRefresher
├── PersistenceCoordinator
├── EntityStateInspector
└── Lifecycle dependencies
```

---

# 23. Factory ≠ manager

La factory administra construcción.

El manager administra el scope runtime.

---

# 24. EntityManager contract

Se propone conceptualmente:

```php
interface EntityManager
{
    public function find(
        string|EntityType $entity,
        mixed $identifier,
    ): ?object;

    public function getReference(
        string|EntityType $entity,
        mixed $identifier,
    ): EntityReference;

    public function persist(object $entity): void;

    public function remove(object $entity): void;

    public function refresh(object $entity): void;

    public function detach(object $entity): void;

    public function clear(?string $entityType = null): void;

    public function contains(object $entity): bool;

    public function flush(): FlushResult;

    public function repository(
        string|EntityType $entity,
    ): EntityRepository;

    public function state(object $entity): EntityState;

    public function isOpen(): bool;

    public function close(): void;
}
```

La API final podrá dividirse en interfaces menores.

---

# 25. Interface Segregation

Para evitar un contrato monolítico se recomienda:

```text
EntityFinder
EntityReferenceProvider
EntityPersistenceManager
EntityStateManager
EntityRepositoryProvider
EntityManagerLifecycle
```

y:

```text
EntityManager
extends
    EntityFinder,
    EntityReferenceProvider,
    EntityPersistenceManager,
    EntityStateManager,
    EntityRepositoryProvider,
    EntityManagerLifecycle
```

---

# 26. EntityManagerInterface vs implementación

Se propone:

```text
EntityManager
```

como contrato público y:

```text
DefaultEntityManager
```

como implementación oficial.

---

# 27. Lifecycle del manager

Estados principales:

```text
OPEN
TAINTED
CLOSED
```

---

# 28. OPEN

```text
OPEN
```

significa que el manager puede aceptar operaciones normales.

---

# 29. TAINTED

```text
TAINTED
```

indica que ocurrió una situación donde el estado sincronizado entre memoria y base de datos ya no puede garantizarse.

Ejemplos:

```text
unknown persistence outcome
connection lost after sending write
transaction outcome unknown
partial flush with uncertain result
```

---

# 30. CLOSED

```text
CLOSED
```

significa que el manager ya no puede utilizarse para operaciones ORM normales.

---

# 31. State machine

```text
          ┌──────────────┐
          │     OPEN     │
          └──────┬───────┘
                 │
       uncertain outcome
                 │
                 ▼
          ┌──────────────┐
          │   TAINTED    │
          └──────┬───────┘
                 │
               close
                 ▼
          ┌──────────────┐
          │    CLOSED    │
          └──────────────┘
```

También:

```text
OPEN
 ↓ close()
CLOSED
```

---

# 32. TAINTED ≠ CLOSED

Un manager `TAINTED` puede permitir:

```text
diagnostics
clear
close
recovery inspection
```

pero no necesariamente nuevas escrituras.

---

# 33. Taint policy

La política deberá definir qué operaciones continúan permitidas.

Ejemplo:

| Operación | OPEN | TAINTED | CLOSED |
|---|---:|---:|---:|
| `find()` | Sí | Política | No |
| `persist()` | Sí | No | No |
| `remove()` | Sí | No | No |
| `flush()` | Sí | No | No |
| `clear()` | Sí | Sí | Política |
| `close()` | Sí | Sí | Idempotente |
| diagnostics | Sí | Sí | Sí |

---

# 34. `find()`

Ejemplo:

```php
$user = $entityManager->find(
    User::class,
    new UserId('01...')
);
```

Pipeline:

```text
find(EntityType, Identifier)
        ↓
Normalize Identifier
        ↓
Build EntityKey
        ↓
IdentityMap Lookup
        │
     hit│
        ├────────────► existing instance
        │
    miss│
        ▼
EntityLoader
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
UnitOfWork registration
        ↓
Entity
```

---

# 35. IdentityMap first

Antes de consultar la base:

```text
find()
    ↓
IdentityMap
```

deberá comprobarse.

---

# 36. Same identity guarantee

Dentro del mismo scope:

```php
$a = $em->find(User::class, 10);
$b = $em->find(User::class, 10);
```

deberá cumplir normalmente:

```php
$a === $b;
```

si ambos representan el mismo:

```text
EntityKey
```

---

# 37. EntityKey context

Recordando el documento 113:

```text
EntityKey
=
IdentityNamespace
+
EntityType
+
CanonicalIdentifier
```

Por tanto:

```text
Tenant A / User#10
≠
Tenant B / User#10
```

---

# 38. `find()` ≠ Repository query

`find()` es una operación especializada por identidad.

No deberá implementarse como:

```php
repository(User::class)
    ->where('id', $id)
    ->first();
```

si eso pierde optimizaciones o semántica de IdentityMap.

---

# 39. Missing entity

Si la entidad no existe:

```php
$em->find(...)
```

devolverá normalmente:

```text
null
```

---

# 40. `findOrFail()`

Podrá existir en la capa Model/Repository/DX:

```php
$em->findOrFail(User::class, $id);
```

pero no es indispensable en el contrato mínimo.

---

# 41. Negative lookup cache

El EntityManager no deberá mantener indefinidamente:

```text
EntityKey → NOT_FOUND
```

porque otra transacción podría insertar la entidad.

Si existe negative caching deberá ser explícito y contextual.

---

# 42. `getReference()`

Ejemplo:

```php
$author = $entityManager->getReference(
    User::class,
    $authorId,
);
```

---

# 43. Reference semantics

`getReference()` deberá producir una referencia lógica:

```text
EntityReference
├── EntityType
├── EntityIdentifier
└── IdentityNamespace
```

No implica que la entidad exista.

---

# 44. Reference ≠ find

```text
getReference()
```

no deberá realizar automáticamente:

```text
SELECT
```

---

# 45. Reference resolution

La resolución posterior podrá pasar por:

```text
EntityReferenceResolver
```

o por mecanismos de lazy loading.

---

# 46. Reference and IdentityMap

Si la entidad ya está en IdentityMap:

```text
getReference(User#10)
```

podrá resolverse inmediatamente al objeto existente.

---

# 47. Reference existence

La existencia de:

```text
EntityReference(User#10)
```

no demuestra:

```text
row User#10 exists
```

---

# 48. `persist()`

Ejemplo:

```php
$entityManager->persist($user);
```

Regla crítica:

> `persist()` no significa `INSERT`.

---

# 49. Persist pipeline

```text
persist(entity)
      ↓
Entity Type Resolution
      ↓
Metadata Lookup
      ↓
Identity/State Validation
      ↓
UnitOfWork Registration
      ↓
Cascade Registration
      ↓
return
```

No deberá ocurrir inmediatamente:

```text
SQL INSERT
```

---

# 50. Persist semantics

Dependiendo del estado:

```text
new entity
    ↓
register for insertion

managed entity
    ↓
already managed / no-op

detached entity
    ↓
reject or explicit attach policy

removed entity
    ↓
state-specific rule
```

Los detalles pertenecen al documento 121.

---

# 51. Persist idempotence

Llamar:

```php
$em->persist($user);
$em->persist($user);
```

no deberá crear dos inserts del mismo objeto dentro del mismo contexto.

---

# 52. Persist cascade

Si metadata declara:

```text
cascade PERSIST
```

el manager podrá delegar traversal a:

```text
PersistenceGraph / UnitOfWork
```

---

# 53. Cascade ≠ recursive manager calls

Debe evitarse una implementación ingenua:

```php
foreach ($relations as $relation) {
    $this->persist($relation);
}
```

sin:

```text
visited set
cycle detection
relationship metadata
cascade policy
```

---

# 54. `remove()`

Ejemplo:

```php
$entityManager->remove($user);
```

Regla:

> `remove()` programa una eliminación ORM; no ejecuta necesariamente un `DELETE` inmediato.

---

# 55. Remove pipeline

```text
remove(entity)
      ↓
Resolve Entity State
      ↓
Validate Managed Identity
      ↓
UnitOfWork Schedule Removal
      ↓
Cascade Remove Analysis
      ↓
return
```

---

# 56. Remove ≠ delete now

Incorrecto:

```php
public function remove(object $entity): void
{
    $this->connection->execute(
        'DELETE ...'
    );
}
```

---

# 57. Remove new entity

La política para:

```text
NEW + remove()
```

deberá definirse explícitamente.

Podría significar:

```text
cancel pending insertion
```

sin emitir DELETE.

---

# 58. Remove detached entity

Por defecto deberá rechazarse o requerir una operación explícita.

No deberá adivinarse qué instancia administrada representa.

---

# 59. `flush()`

`flush()` es una de las operaciones más importantes.

Semántica:

```text
Synchronize known UnitOfWork changes
with the database
```

---

# 60. Flush pipeline

```text
EntityManager::flush()
        ↓
Validate Manager State
        ↓
UnitOfWork
        ↓
Change Detection
        ↓
ChangeSets
        ↓
Entity Change Graph
        ↓
Persistence Planner
        ↓
Persistence Plan
        ↓
Query Models
        ↓
Query Engine
        ↓
Execution
        ↓
Outcome Verification
        ↓
UnitOfWork Reconciliation
        ↓
FlushResult
```

---

# 61. flush() ≠ commit()

Esta regla será absoluta:

```text
flush()
≠
commit()
```

Ejemplo:

```php
$transaction->begin();

$em->persist($user);
$em->flush();

$em->persist($profile);
$em->flush();

$transaction->commit();
```

es válido conceptualmente.

---

# 62. Flush transaction integration

El Persistence Engine podrá requerir una transacción.

Pero la política deberá ser explícita.

Posibilidades:

```text
EXISTING_REQUIRED
AUTO_LOCAL_TRANSACTION
NO_AUTOMATIC_TRANSACTION
CALLER_CONTROLLED
```

---

# 63. Recommended default

Para operaciones de escritura multi-statement:

```text
AUTO_LOCAL_TRANSACTION
```

puede ser un default razonable cuando no existe transacción exterior.

Pero:

```text
auto transaction
≠
EntityManager owns TransactionManager
```

---

# 64. FlushResult

Se propone:

```php
final readonly class FlushResult
{
    public function __construct(
        public FlushStatus $status,
        public PersistenceOutcomeCertainty $certainty,
        public int $inserted,
        public int $updated,
        public int $deleted,
        public FlushId $flushId,
    ) {}
}
```

---

# 65. Flush status

Podría incluir:

```text
SUCCEEDED
FAILED
PARTIALLY_APPLIED
CANCELLED
UNKNOWN
```

---

# 66. Outcome certainty

Separar:

```text
status
```

de:

```text
certainty
```

porque una conexión perdida puede impedir saber si un write fue aplicado.

---

# 67. Unknown flush

Si:

```text
INSERT sent
connection lost
commit acknowledgment missing
```

el resultado puede ser:

```text
UNKNOWN
```

---

# 68. Unknown outcome behavior

El EntityManager no deberá:

```text
mark entity synchronized
```

como si conociera el resultado.

---

# 69. Taint after unknown

Normalmente:

```text
UNKNOWN persistence outcome
    ↓
EntityManager → TAINTED
```

---

# 70. Blind retry forbidden

El manager no deberá ejecutar automáticamente:

```text
flush again
```

después de resultado desconocido.

---

# 71. Why

Porque podría producir:

```text
duplicate insert
duplicate external effect
unexpected version conflict
duplicate relation rows
```

---

# 72. Flush scope

Por defecto:

```php
$em->flush();
```

sincroniza el UnitOfWork completo del manager.

---

# 73. Partial flush

Una API:

```php
$em->flush($entity);
```

deberá evitarse inicialmente.

Razón:

```text
entity dependencies
relationship changes
cascade operations
referential ordering
```

pueden convertir un “partial flush” en semántica peligrosa.

---

# 74. Recommendation

V1:

```text
flush entire PersistenceContext
```

como regla principal.

---

# 75. Future scoped flush

Si posteriormente se implementa:

```text
flushScope(...)
```

deberá utilizar un subgraph explícito del Persistence Graph.

No una simple lista arbitraria de objetos.

---

# 76. Flush reentrancy

No deberá permitirse:

```text
flush()
    ↓ lifecycle callback
        ↓
      flush()
```

sin una política explícita.

---

# 77. Default reentrancy policy

Recomendación:

```text
nested flush
→
reject
```

---

# 78. Reason

Nested flush puede corromper:

```text
ChangeSets
PersistencePlan
operation ordering
IdentityMap reconciliation
event ordering
```

---

# 79. Flush guard

Se propone estado interno:

```text
IDLE
PREPARING
EXECUTING
RECONCILING
```

---

# 80. FlushSession

Cada flush deberá tener:

```text
FlushSession
```

operation-scoped.

---

# 81. FlushSession ≠ EntityManager

El manager puede sobrevivir múltiples flush.

Cada flush tendrá su propia sesión.

---

# 82. Flush identity

```text
FlushId
```

deberá ser distinto de:

```text
TransactionId
QueryExecutionId
RequestId
```

---

# 83. `refresh()`

Ejemplo:

```php
$entityManager->refresh($user);
```

Semántica:

```text
reload authoritative persistent state
into an existing managed entity
```

---

# 84. Refresh pipeline

```text
refresh(entity)
      ↓
Validate Managed
      ↓
Resolve EntityKey
      ↓
Load Fresh State
      ↓
Hydration/State Application
      ↓
Snapshot Replacement
      ↓
UnitOfWork Reconciliation
```

---

# 85. Refresh ≠ find

`find()` puede devolver el objeto ya administrado sin consultar DB.

`refresh()` solicita explícitamente nueva lectura.

---

# 86. Refresh dirty entity

Si la entidad tiene cambios locales, la política deberá ser explícita.

Opciones:

```text
DISCARD_LOCAL_CHANGES
REJECT_IF_DIRTY
MERGE_EXPLICITLY
```

---

# 87. Recommended default

Recomendación:

```text
REJECT_IF_DIRTY
```

o exigir explícitamente:

```php
$em->refresh(
    $user,
    RefreshMode::DISCARD_LOCAL_CHANGES
);
```

para evitar pérdida silenciosa.

---

# 88. Refresh missing entity

Si la fila desapareció:

```text
refresh(managed entity)
```

deberá producir un resultado/exception semántico claro.

No convertir silenciosamente el objeto en `null`.

---

# 89. `detach()`

Ejemplo:

```php
$entityManager->detach($user);
```

significa:

```text
stop managing this entity instance
```

---

# 90. Detach effects

Deberá retirar apropiadamente:

```text
IdentityMap registration
UnitOfWork tracking
snapshots
pending changes
```

según reglas del Entity State System.

---

# 91. Detach ≠ delete

```text
detach(entity)
≠
remove(entity)
```

La base de datos no cambia.

---

# 92. Detach dirty entity

Si existen cambios no flushed:

```text
detach()
```

podría descartarlos del PersistenceContext.

Esto deberá ser visible.

---

# 93. Detach cascade

Podrá existir:

```text
cascade DETACH
```

si el Relationship System lo soporta.

---

# 94. `clear()`

Ejemplo:

```php
$entityManager->clear();
```

significa:

```text
detach/reset all managed entity state
for this PersistenceContext
```

---

# 95. Clear effects

Conceptualmente:

```text
IdentityMap.clear()
UnitOfWork.clear()
Snapshots.clear()
Relationship runtime state.clear()
```

---

# 96. Clear ≠ close

Después:

```php
$em->clear();
```

el manager podrá seguir:

```text
OPEN
```

---

# 97. Clear ≠ connection close

No deberá implicar:

```text
close DB connection
```

---

# 98. Clear ≠ transaction rollback

No deberá implicar:

```text
rollback current transaction
```

---

# 99. Clear with pending changes

Si existen cambios no flushed, `clear()` puede descartarlos.

La API deberá permitir diagnostics o políticas como:

```text
ALLOW_DISCARD
WARN
REJECT_IF_DIRTY
```

---

# 100. Recommended explicitness

Podría existir:

```php
$em->clear(
    ClearMode::REJECT_IF_DIRTY
);
```

para contextos estrictos.

---

# 101. Entity-type clear

Una futura API:

```php
$em->clear(User::class);
```

deberá analizar dependencias.

No deberá dejar referencias internas inconsistentes.

---

# 102. `contains()`

Ejemplo:

```php
$entityManager->contains($user);
```

responde:

```text
¿esta instancia pertenece actualmente
al PersistenceContext de este manager?
```

---

# 103. contains() ≠ row exists

```text
contains($user) == true
```

no significa:

```text
database row definitely exists right now
```

---

# 104. contains() ≠ same identifier exists

Dos objetos pueden poseer mismo ID.

Solo uno puede ser la instancia administrada.

---

# 105. `state()`

Ejemplo:

```php
$state = $entityManager->state($user);
```

delegará a:

```text
EntityStateInspector
```

---

# 106. State details deferred

El state machine completo se define en:

```text
121_DATABASE_ENTITY_STATE_SYSTEM.md
```

Conceptualmente incluirá:

```text
NEW
MANAGED
DIRTY
REMOVED
DETACHED
```

con mayor precisión posteriormente.

---

# 107. EntityManager must not store state on entity

Evitar:

```php
$user->__voltstack_state = 'managed';
```

El estado ORM deberá mantenerse externamente.

---

# 108. Repository resolution

```php
$repository = $entityManager->repository(
    User::class
);
```

---

# 109. RepositoryResolver

El manager delegará:

```text
EntityType
    ↓
RepositoryResolver
    ↓
EntityRepository
```

---

# 110. Repository lifecycle

Los repositories deberán estar asociados al:

```text
EntityManager scope
```

o utilizar un contexto equivalente seguro.

---

# 111. Repository ≠ process-global manager holder

Incorrecto:

```php
final class UserRepository
{
    private static EntityManager $manager;
}
```

---

# 112. Repository cache

El manager puede cachear:

```text
EntityType → Repository instance
```

dentro de su scope.

---

# 113. Custom repositories

Metadata podrá indicar:

```text
User
→
UserRepository
```

---

# 114. Repository constructor

Idealmente:

```php
final class UserRepository extends EntityRepository
{
    public function __construct(
        EntityManager $manager,
        EntityMetadata $metadata,
    ) {}
}
```

o contratos más pequeños.

---

# 115. Repository isolation

Repository de manager A no deberá reutilizar manager B.

Especialmente:

```text
Tenant A
≠
Tenant B
```

---

# 116. Entity Loader

El EntityManager deberá delegar carga a:

```text
EntityLoader
```

---

# 117. EntityLoader responsibilities

```text
identity-based loading
entity query creation
execution delegation
hydration coordination
identity-map integration
```

---

# 118. EntityLoader ≠ Query Engine

El loader conoce entidades.

El Query Engine conoce Query Model.

---

# 119. Load pipeline

```text
EntityKey
   ↓
EntityLoader
   ↓
EntityQueryTranslator
   ↓
Query Model
   ↓
Query Engine
   ↓
Result
   ↓
Hydration
```

---

# 120. EntityRefresher

`refresh()` deberá delegarse a:

```text
EntityRefresher
```

para evitar llenar el manager de lógica especializada.

---

# 121. PersistenceCoordinator

`flush()` deberá delegar a:

```text
PersistenceCoordinator
```

---

# 122. PersistenceCoordinator pipeline

```text
UnitOfWork
    ↓
ChangeTracking
    ↓
ChangeSet collection
    ↓
Persistence Planner
    ↓
Persistence Engine
    ↓
Execution
    ↓
State Reconciliation
```

---

# 123. Coordinator ≠ Persistence Engine

El coordinator administra el workflow.

El Persistence Engine transforma cambios en operaciones persistentes.

---

# 124. Manager internals

Una implementación razonable:

```php
final class DefaultEntityManager implements EntityManager
{
    public function __construct(
        private readonly DatabaseContext $databaseContext,
        private readonly EntityMetadataRegistry $metadata,
        private readonly IdentityMap $identityMap,
        private readonly UnitOfWork $unitOfWork,
        private readonly RepositoryResolver $repositories,
        private readonly EntityLoader $loader,
        private readonly EntityRefresher $refresher,
        private readonly PersistenceCoordinator $persistence,
        private readonly EntityStateInspector $states,
    ) {}
}
```

---

# 125. Constructor size concern

Si el número de dependencias aumenta demasiado, no deberá resolverse con Service Locator.

Podrá utilizarse:

```text
EntityManagerRuntime
```

como composición interna tipada.

---

# 126. EntityManagerRuntime

Ejemplo:

```php
final readonly class EntityManagerRuntime
{
    public function __construct(
        public IdentityMap $identityMap,
        public UnitOfWork $unitOfWork,
        public EntityLoader $loader,
        public EntityRefresher $refresher,
        public PersistenceCoordinator $persistence,
    ) {}
}
```

---

# 127. Runtime object caution

`EntityManagerRuntime` no deberá convertirse en:

```text
bag of every framework service
```

---

# 128. Metadata access

El manager podrá exponer:

```php
$em->metadata(User::class);
```

si se considera útil.

Pero deberá devolver:

```text
immutable EntityMetadata
```

---

# 129. Metadata mutation

No:

```php
$em->metadata(User::class)->table = 'other';
```

---

# 130. Model API integration

El documento 114 definió la API estilo Laravel.

Ejemplo:

```php
$user = User::find(10);

$user->name = 'Alice';

$user->save();
```

deberá converger en:

```text
Model API
    ↓
ModelContextResolver
    ↓
EntityManager
    ↓
same UnitOfWork
    ↓
same Persistence Engine
```

---

# 131. `Model::save()`

Conceptualmente:

```text
Model::save()
    ↓
EntityManager.persist(model)
    ↓
EntityManager.flush()
```

según política de Model API.

---

# 132. No second manager

Model API no deberá crear:

```text
ActiveRecordEntityManager
```

separado.

---

# 133. Repository API integration

También:

```php
$users->save($user);
```

si existe como conveniencia, deberá delegar al mismo manager.

---

# 134. Entity Query integration

El EntityManager podrá proporcionar:

```php
$em->query(User::class);
```

si la API pública lo decide.

Pero:

```text
EntityManager
```

solo crea/resuelve:

```text
EntityQueryBuilder
```

No implementa Query Builder.

---

# 135. Example

```php
$users = $em
    ->query(User::class)
    ->where('active', true)
    ->orderBy('name')
    ->get();
```

internamente:

```text
Entity Query
    ↓
Entity Mapping
    ↓
Query AST
    ↓
Query Engine
```

---

# 136. Query results and IdentityMap

Una Entity Query que retorna entidades deberá respetar:

```text
IdentityMap
```

---

# 137. Existing managed entity

Si una query retorna una fila para una entidad ya administrada:

```text
Result Row
    ↓
EntityKey
    ↓
IdentityMap hit
    ↓
reuse managed instance
```

---

# 138. Overwrite protection

Una query normal no deberá sobrescribir silenciosamente cambios locales dirty.

---

# 139. Query hydration modes

La política podrá distinguir:

```text
MANAGED
READ_ONLY
DETACHED
PROJECTION
SCALAR
```

---

# 140. Managed mode

```text
MANAGED
```

registra entidades en:

```text
IdentityMap
UnitOfWork
```

---

# 141. Read-only mode

```text
READ_ONLY
```

puede evitar:

```text
snapshots
change tracking
```

según contrato.

---

# 142. Detached mode

Podrá hidratar objetos sin registrarlos permanentemente.

---

# 143. Projection mode

Retorna DTO/projection.

No entra al PersistenceContext.

---

# 144. EntityManager and hydration

El manager no hidrata propiedades directamente.

Incorrecto:

```php
$entity->name = $row['name'];
```

dentro del EntityManager.

---

# 145. Hydration delegation

```text
EntityManager
    ↓
EntityLoader
    ↓
Hydration System
```

---

# 146. EntityManager and relationships

El manager tampoco deberá implementar directamente:

```text
lazy collection
eager loading
relationship synchronization
```

---

# 147. Relationship integration

```text
EntityManager
    ↓
Relationship Runtime
    ↓
Entity Loader / Entity Query
```

---

# 148. Lazy loading

Una lazy reference podrá necesitar acceso a un scope ORM.

Pero las entidades no deberán almacenar arbitrariamente el manager.

---

# 149. Avoid manager injection into entities

Incorrecto:

```php
class User
{
    private EntityManager $entityManager;
}
```

como requisito ORM general.

---

# 150. Lazy loading context

Preferir:

```text
LazyReference
→ scoped loader handle
```

o una estrategia equivalente controlada.

---

# 151. Detached lazy relation

Si una entidad está detached y se intenta lazy loading:

```text
do not magically reconnect
```

---

# 152. Possible result

```text
DetachedEntityLazyLoadException
```

o política explícita.

---

# 153. Transaction integration

El EntityManager deberá consumir:

```text
TransactionManager
```

mediante una capa de coordinación cuando sea necesario.

---

# 154. Explicit transaction

Ejemplo:

```php
$transactions->transactional(
    function () use ($em) {
        $em->persist($user);
        $em->persist($profile);
        $em->flush();
    }
);
```

---

# 155. Transaction helper

Podría ofrecerse una comodidad:

```php
$em->transactional(function (EntityManager $em) {
    // ...
});
```

pero internamente deberá delegar al Transaction System.

---

# 156. Transactional helper ≠ transaction ownership

`EntityManager::transactional()` sería una fachada.

No una segunda implementación de transacciones.

---

# 157. Closure retry warning

Si TransactionManager soporta retries:

```php
transactional(fn () => ...)
```

el callback podría ejecutarse más de una vez.

Esto deberá ser explícito.

---

# 158. ORM transaction retry

No deberá reintentar callbacks con:

```text
external side effects
email
HTTP calls
payment calls
```

sin garantías de replay.

---

# 159. Transaction rollback

Si ocurre rollback conocido:

```text
database state
```

vuelve al boundary correspondiente.

Pero el estado en memoria requiere reconciliación.

---

# 160. Rollback ≠ automatic entity rewind

Muy importante:

```text
DB transaction rollback
≠
automatically restore every PHP property
```

---

# 161. Rollback strategy

Después de rollback podrán aplicarse políticas:

```text
KEEP_MANAGED_BUT_UNSYNCHRONIZED
CLEAR
MARK_FOR_REFRESH
TAINT
CLOSE
```

dependiendo del caso.

---

# 162. Conservative default

Para fallos complejos de flush:

```text
TAINT/CLEAR
```

puede ser más seguro que intentar reconstruir mágicamente el estado.

---

# 163. EntityManager and events

El manager podrá emitir eventos de coordinación:

```text
EntityManagerCreated
EntityManagerCleared
EntityManagerTainted
EntityManagerClosed
FlushStarted
FlushCompleted
FlushFailed
```

---

# 164. Events ≠ lifecycle callbacks

Distinguir:

```text
EntityManager events
```

de:

```text
Entity lifecycle callbacks
```

---

# 165. postPersist timing

`postPersist` después del INSERT no implica:

```text
transaction committed
```

---

# 166. Commit events

Los eventos que necesiten semántica:

```text
after commit
```

deberán integrarse con Transaction System.

---

# 167. Error model

Se propone:

```text
DatabaseOrmException
└── EntityManagerException
    ├── EntityManagerClosedException
    ├── EntityManagerTaintedException
    ├── EntityManagerScopeException
    ├── EntityManagerContextException
    ├── EntityManagerReentrancyException
    ├── UnknownEntityTypeException
    ├── InvalidEntityException
    ├── UnmanagedEntityException
    ├── DetachedEntityException
    ├── EntityNotFoundException
    ├── EntityIdentityConflictException
    ├── EntityRefreshException
    ├── EntityDetachException
    ├── EntityClearException
    ├── FlushException
    ├── FlushReentrancyException
    ├── FlushOutcomeUnknownException
    ├── PersistenceContextException
    ├── RepositoryResolutionException
    └── EntityManagerInvariantException
```

---

# 168. Error translation

El EntityManager solo deberá traducir errores cuando pueda añadir semántica ORM real.

Ejemplo:

```text
0 affected rows
+
optimistic version predicate
→
OptimisticLockException
```

No deberá ocultar indiscriminadamente errores de DB.

---

# 169. Exception ≠ outcome

Una excepción no siempre significa:

```text
nothing happened
```

Especialmente en writes.

---

# 170. Error result metadata

Errores de persistencia deberán poder transportar:

```text
FlushId
operation
outcome certainty
transaction state
manager state
recovery guidance
```

---

# 171. EntityManager scope

El manager deberá ser:

```text
request-scoped
operation-scoped
job-scoped
```

según runtime.

Nunca:

```text
process-global mutable singleton
```

---

# 172. FrankenPHP

Modelo recomendado:

```text
FrankenPHP Worker
│
├── shared immutable MetadataRegistry
├── shared immutable Mapping Registry
├── shared immutable Type Registry
│
├── Request A
│   └── EntityManager A
│       ├── IdentityMap A
│       └── UnitOfWork A
│
└── Request B
    └── EntityManager B
        ├── IdentityMap B
        └── UnitOfWork B
```

---

# 173. Forbidden persistent state

Nunca:

```text
Worker
└── Global EntityManager
    ├── User from Request A
    ├── Order from Request B
    └── UnitOfWork from previous request
```

---

# 174. RoadRunner

La misma regla deberá aplicarse.

---

# 175. OpenSwoole

Con concurrencia/coroutines:

```text
EntityManager A
```

no deberá compartirse simultáneamente con:

```text
Coroutine B
```

---

# 176. Thread/coroutine safety

El manager será:

```text
scope-confined
```

no thread-safe por diseño.

---

# 177. Scope ownership

Podrá existir:

```text
EntityManagerScopeId
```

para diagnostics.

---

# 178. Cross-scope entity use

Una entidad administrada por manager A y entregada a manager B deberá tratarse como:

```text
detached/foreign context
```

según Entity State System.

---

# 179. Cross-manager identity

Manager B no deberá asumir que:

```text
object managed by A
```

está administrado por B.

---

# 180. Persistent runtime cleanup

Al terminar request/job:

```text
EntityManagerScope
    ↓
cleanup
```

deberá eliminar:

```text
IdentityMap
UnitOfWork
snapshots
repository instances
lazy loader sessions
pending operations
temporary query state
```

---

# 181. Cleanup ≠ flush

Regla crítica:

> Finalizar un request nunca deberá ejecutar `flush()` implícitamente.

---

# 182. Why

El código:

```php
$em->persist($user);
```

sin:

```php
$em->flush();
```

no deberá guardar datos accidentalmente al destruir el manager.

---

# 183. Destructor

`__destruct()` no deberá realizar persistencia.

---

# 184. Destructor exceptions

Los destructores tampoco son lugar seguro para:

```text
commit
rollback complex recovery
flush
```

---

# 185. Scope reset

El runtime deberá preferir lifecycle explícito:

```text
create
use
cleanup
release
```

---

# 186. Long-running jobs

Para procesamiento grande:

```php
foreach ($users as $user) {
    // ...
}
```

mantener miles/millones de entidades managed puede consumir demasiada memoria.

---

# 187. Batch processing

Patrón:

```php
foreach ($batch as $entity) {
    // mutate

    if (++$count % 500 === 0) {
        $em->flush();
        $em->clear();
    }
}
```

---

# 188. Exact batch size

El tamaño deberá ser configurable/medible.

No se fijará arquitectónicamente en `500`.

---

# 189. Memory telemetry

El manager podrá exponer métricas:

```text
managed_entity_count
identity_map_size
unit_of_work_size
snapshot_memory
pending_insert_count
pending_update_count
pending_delete_count
```

---

# 190. Memory budget

Una futura política podrá advertir:

```text
EntityManager managed entities > configured threshold
```

---

# 191. Streaming

Una query streaming no deberá registrar indefinidamente todas las entidades.

---

# 192. Streaming policies

Se soportarán estrategias como:

```text
TRACKED
DETACHED_AFTER_YIELD
READ_ONLY
SCALAR
PROJECTION
```

---

# 193. DETACHED_AFTER_YIELD

Conceptualmente:

```text
hydrate
→ yield entity
→ detach after safe boundary
```

---

# 194. Tracking warning

```text
TRACKED + millions of rows
```

puede producir crecimiento de memoria.

DX/Telemetry deberá advertirlo.

---

# 195. Multitenancy

El core ORM no deberá hardcodear:

```text
tenant_id
```

---

# 196. Tenant manager creation

Integración:

```text
TenantContext
    ↓
DatabaseContextResolver
    ↓
EntityManagerFactory
    ↓
EntityManager
```

---

# 197. Tenant isolation

Nunca:

```text
EntityManager Tenant A
```

deberá administrar una entidad cuyo:

```text
EntityKey namespace
```

pertenezca a Tenant B.

---

# 198. Tenant switching

Cambiar tenant dentro de un manager existente deberá prohibirse.

Incorrecto:

```php
$em->setTenant($tenantB);
```

---

# 199. Correct pattern

```text
Tenant A
→ EntityManager A

Tenant B
→ EntityManager B
```

---

# 200. Sharding

El manager tampoco decidirá directamente:

```text
shard server
```

---

# 201. Shard context

El `DatabaseContext` podrá contener:

```text
logical shard context
```

resuelto por sistemas inferiores.

---

# 202. Cross-shard UoW

Un UnitOfWork distribuido sobre múltiples shards deberá considerarse una capacidad explícita avanzada.

No deberá aparecer accidentalmente.

---

# 203. Read/write routing

EntityManager declara intención:

```text
find/query → READ
flush → WRITE
```

---

# 204. Physical routing

Después:

```text
Database Routing
→ primary / replica
```

---

# 205. Sticky reads

Después de write:

```text
read-your-writes
```

podrá ser administrado por:

```text
Read/Write Routing System
```

no por vendor logic dentro del EntityManager.

---

# 206. EntityManager and Cache

IdentityMap es cache de identidad local al scope.

Pero:

```text
IdentityMap
≠
Second Level Entity Cache
```

---

# 207. Entity cache

Un futuro:

```text
EntityCache
```

será integración separada.

---

# 208. Cache lookup

El loader podrá seguir:

```text
IdentityMap
    ↓ miss
EntityCache
    ↓ miss
Database
```

si el cache está habilitado.

---

# 209. Cache semantics

EntityManager no deberá esconder inconsistencia de cache.

---

# 210. Security

EntityManager no es Authorization System.

Ejemplo:

```php
$user = $em->find(User::class, 10);
```

no significa:

```text
current actor may view User#10
```

---

# 211. ID possession ≠ permission

El hecho de poseer:

```text
UserId(10)
```

no autoriza acceso.

---

# 212. Authorization integration

Debe ocurrir en:

```text
Application / Authorization layer
```

o mediante integración explícita.

---

# 213. Mass assignment

`EntityManager::persist()` no realiza mass assignment.

Eso pertenece a:

```text
Model API
DTO mapping
input mapping
```

---

# 214. Sensitive diagnostics

`EntityManager` deberá poder redactar:

```text
identifiers
field values
query parameters
```

según política.

---

# 215. Telemetry

Métricas propuestas:

```text
orm.entity_manager.created
orm.entity_manager.closed
orm.entity_manager.tainted

orm.entity_manager.managed_entities
orm.entity_manager.identity_map.size
orm.entity_manager.unit_of_work.size

orm.entity_manager.find.count
orm.entity_manager.find.identity_map_hit
orm.entity_manager.find.database_load

orm.entity_manager.persist.count
orm.entity_manager.remove.count
orm.entity_manager.refresh.count
orm.entity_manager.detach.count
orm.entity_manager.clear.count

orm.entity_manager.flush.count
orm.entity_manager.flush.duration
orm.entity_manager.flush.inserted
orm.entity_manager.flush.updated
orm.entity_manager.flush.deleted
orm.entity_manager.flush.failed
orm.entity_manager.flush.unknown
```

---

# 216. Tracing

Un flush podrá verse como:

```text
ORM Flush
├── Detect Changes
├── Build ChangeSets
├── Build Persistence Graph
├── Plan Persistence
├── Execute Inserts
├── Execute Updates
├── Execute Relationships
├── Execute Deletes
└── Reconcile State
```

---

# 217. Find tracing

```text
EntityManager.find
├── Normalize Identifier
├── IdentityMap Lookup
├── Entity Query
├── Execute
└── Hydrate
```

---

# 218. Telemetry ≠ semantics

Deshabilitar telemetry no deberá cambiar:

```text
entity identity
flush order
transaction behavior
state transitions
```

---

# 219. Debugging

El profiler podrá mostrar:

```text
EntityManager Scope
├── State: OPEN
├── Managed: 32
├── New: 2
├── Dirty: 4
├── Removed: 1
├── IdentityMap hits: 18
├── Flushes: 2
└── Memory: ...
```

---

# 220. Debug Toolbar

Integración futura con:

```text
VoltStack Telemetry / Debug Toolbar
```

sin que el EntityManager dependa directamente de UI.

---

# 221. Testing

El sistema requiere:

```text
unit tests
integration tests
persistence tests
runtime isolation tests
transaction tests
failure tests
concurrency tests
```

---

# 222. Find tests

Probar:

```text
IdentityMap hit
IdentityMap miss
entity found
entity missing
composite identifier
typed identifier
wrong identifier type
different tenant namespace
```

---

# 223. Identity tests

```text
find same EntityKey twice
→ same instance
```

---

# 224. Persist tests

```text
new entity
managed entity
persist twice
detached entity
cascade persist
cyclic graph
generated identifier
assigned identifier
```

---

# 225. Remove tests

```text
managed entity
new entity
detached entity
cascade remove
remove twice
```

---

# 226. Flush tests

```text
empty flush
insert
update
delete
mixed changes
dependency ordering
generated IDs
relationship updates
multiple flushes
nested flush rejection
```

---

# 227. Failure tests

```text
statement failure
constraint violation
deadlock
connection loss
timeout
cancellation
transaction rollback
unknown commit outcome
partial execution
```

---

# 228. Taint tests

Comprobar:

```text
unknown outcome
→ manager TAINTED
```

y que:

```text
persist/remove/flush
```

se rechacen después.

---

# 229. Clear tests

```text
clear empty
clear managed entities
clear dirty context
clear after flush
clear after rollback
clear does not close connection
clear does not commit
```

---

# 230. Refresh tests

```text
managed clean
managed dirty
row changed externally
row deleted externally
optimistic version changed
```

---

# 231. Detach tests

```text
managed
dirty
new
removed
relationship graph
lazy references
```

---

# 232. Repository tests

```text
default repository
custom repository
repository caching
repository isolation between managers
tenant isolation
```

---

# 233. Persistent runtime tests

Simular:

```text
Request A
Request B
Request C
```

sobre el mismo worker.

Comprobar:

```text
no entity leakage
no IdentityMap leakage
no UnitOfWork leakage
no repository leakage
no tenant leakage
no pending flush leakage
```

---

# 234. Coroutine tests

Para OpenSwoole:

```text
Coroutine A
EntityManager A

Coroutine B
EntityManager B
```

deberán operar sin compartir mutable state.

---

# 235. Invariantes arquitectónicas

## DB-ENTITY-MANAGER-001
EntityManager será un coordinador ORM.

## DB-ENTITY-MANAGER-002
EntityManager no será el ORM completo.

## DB-ENTITY-MANAGER-003
EntityManager no será UnitOfWork.

## DB-ENTITY-MANAGER-004
EntityManager no será IdentityMap.

## DB-ENTITY-MANAGER-005
EntityManager no será Repository.

## DB-ENTITY-MANAGER-006
EntityManager no será Query Builder.

## DB-ENTITY-MANAGER-007
EntityManager no será SQL Compiler.

## DB-ENTITY-MANAGER-008
EntityManager no será Execution Engine.

## DB-ENTITY-MANAGER-009
EntityManager no será Connection.

## DB-ENTITY-MANAGER-010
EntityManager no será Driver.

## DB-ENTITY-MANAGER-011
EntityManager no será TransactionManager.

## DB-ENTITY-MANAGER-012
EntityManager no será Service Locator universal.

## DB-ENTITY-MANAGER-013
EntityManager coordinará un PersistenceContext.

## DB-ENTITY-MANAGER-014
PersistenceContext será distinto de Transaction.

## DB-ENTITY-MANAGER-015
PersistenceContext será distinto de Connection.

## DB-ENTITY-MANAGER-016
EntityManager será creado por EntityManagerFactory.

## DB-ENTITY-MANAGER-017
EntityManager tendrá DatabaseContext explícito.

## DB-ENTITY-MANAGER-018
DatabaseContext será lógico, no una conexión física.

## DB-ENTITY-MANAGER-019
Manager state será explícito.

## DB-ENTITY-MANAGER-020
OPEN permitirá operaciones normales.

## DB-ENTITY-MANAGER-021
TAINTED representará pérdida de confianza en sincronización.

## DB-ENTITY-MANAGER-022
CLOSED rechazará operaciones ORM normales.

## DB-ENTITY-MANAGER-023
TAINTED será distinto de CLOSED.

## DB-ENTITY-MANAGER-024
Unknown write outcome podrá taint el manager.

## DB-ENTITY-MANAGER-025
Unknown outcome nunca será tratado como success.

## DB-ENTITY-MANAGER-026
Unknown outcome nunca será tratado automáticamente como failure-before-effect.

## DB-ENTITY-MANAGER-027
Blind retry después de unknown outcome estará prohibido.

## DB-ENTITY-MANAGER-028
`find()` consultará IdentityMap antes de DB.

## DB-ENTITY-MANAGER-029
`find()` normalizará identificadores.

## DB-ENTITY-MANAGER-030
`find()` utilizará EntityKey contextual.

## DB-ENTITY-MANAGER-031
Mismo EntityKey dentro del scope resolverá a la misma instancia administrada.

## DB-ENTITY-MANAGER-032
Mismo ID en diferentes EntityTypes no será misma identidad.

## DB-ENTITY-MANAGER-033
Mismo EntityType/ID en diferentes identity namespaces no será misma identidad.

## DB-ENTITY-MANAGER-034
`find()` missing podrá retornar null.

## DB-ENTITY-MANAGER-035
`getReference()` no implicará query.

## DB-ENTITY-MANAGER-036
EntityReference no probará existencia.

## DB-ENTITY-MANAGER-037
EntityReference no almacenará una conexión viva.

## DB-ENTITY-MANAGER-038
EntityReference no almacenará obligatoriamente EntityManager.

## DB-ENTITY-MANAGER-039
`persist()` no ejecutará INSERT inmediatamente.

## DB-ENTITY-MANAGER-040
`persist()` registrará intención en UnitOfWork.

## DB-ENTITY-MANAGER-041
Persistir dos veces la misma instancia no generará dos inserciones.

## DB-ENTITY-MANAGER-042
Cascade persist utilizará metadata.

## DB-ENTITY-MANAGER-043
Cascade traversal tendrá cycle detection.

## DB-ENTITY-MANAGER-044
`remove()` no ejecutará DELETE inmediatamente.

## DB-ENTITY-MANAGER-045
`remove()` programará eliminación en UnitOfWork.

## DB-ENTITY-MANAGER-046
Remove detached no será aceptado silenciosamente.

## DB-ENTITY-MANAGER-047
`flush()` sincronizará UnitOfWork.

## DB-ENTITY-MANAGER-048
`flush()` no será `commit()`.

## DB-ENTITY-MANAGER-049
`flush()` no será `beginTransaction()`.

## DB-ENTITY-MANAGER-050
`flush()` no implicará fin del EntityManager.

## DB-ENTITY-MANAGER-051
Múltiples flush podrán ocurrir dentro de una transacción.

## DB-ENTITY-MANAGER-052
Una transacción podrá contener múltiples flush.

## DB-ENTITY-MANAGER-053
Flush utilizará Persistence Planner.

## DB-ENTITY-MANAGER-054
Flush utilizará Persistence Engine.

## DB-ENTITY-MANAGER-055
EntityManager no generará SQL durante flush.

## DB-ENTITY-MANAGER-056
Persistence Engine generará Query Models, no SQL directo.

## DB-ENTITY-MANAGER-057
SQL Compiler permanecerá downstream.

## DB-ENTITY-MANAGER-058
Flush tendrá identidad propia.

## DB-ENTITY-MANAGER-059
FlushId será distinto de TransactionId.

## DB-ENTITY-MANAGER-060
Flush outcome tendrá certeza explícita.

## DB-ENTITY-MANAGER-061
Entity state solo se reconciliará cuando el resultado lo permita.

## DB-ENTITY-MANAGER-062
Generated IDs no se inventarán después de unknown outcome.

## DB-ENTITY-MANAGER-063
Nested flush estará prohibido por defecto.

## DB-ENTITY-MANAGER-064
Flush reentrancy tendrá guard explícito.

## DB-ENTITY-MANAGER-065
Partial flush no será API base de V1.

## DB-ENTITY-MANAGER-066
Full UnitOfWork flush será semántica base.

## DB-ENTITY-MANAGER-067
`refresh()` forzará una lectura autoritativa.

## DB-ENTITY-MANAGER-068
`refresh()` será distinto de `find()`.

## DB-ENTITY-MANAGER-069
Refresh dirty no descartará cambios silenciosamente.

## DB-ENTITY-MANAGER-070
Refresh missing tendrá semántica explícita.

## DB-ENTITY-MANAGER-071
`detach()` no eliminará la fila.

## DB-ENTITY-MANAGER-072
Detach retirará la instancia del PersistenceContext.

## DB-ENTITY-MANAGER-073
Detach dirty podrá descartar tracking y deberá ser visible.

## DB-ENTITY-MANAGER-074
`clear()` no será `close()`.

## DB-ENTITY-MANAGER-075
`clear()` no cerrará conexiones por definición.

## DB-ENTITY-MANAGER-076
`clear()` no hará commit.

## DB-ENTITY-MANAGER-077
`clear()` no hará rollback.

## DB-ENTITY-MANAGER-078
Clear con cambios pendientes tendrá política explícita.

## DB-ENTITY-MANAGER-079
`contains()` preguntará pertenencia al PersistenceContext.

## DB-ENTITY-MANAGER-080
`contains()` no probará existencia actual en DB.

## DB-ENTITY-MANAGER-081
Entity state se almacenará fuera de la entidad por defecto.

## DB-ENTITY-MANAGER-082
EntityManager delegará state inspection.

## DB-ENTITY-MANAGER-083
Repository resolution será delegada.

## DB-ENTITY-MANAGER-084
Repository instance será scope-safe.

## DB-ENTITY-MANAGER-085
Repository no tendrá process-global EntityManager.

## DB-ENTITY-MANAGER-086
Custom repository utilizará el mismo ORM core.

## DB-ENTITY-MANAGER-087
EntityManager delegará entity loading.

## DB-ENTITY-MANAGER-088
EntityLoader será distinto de Query Engine.

## DB-ENTITY-MANAGER-089
EntityManager delegará hydration.

## DB-ENTITY-MANAGER-090
EntityManager no asignará campos de row manualmente.

## DB-ENTITY-MANAGER-091
Entity queries respetarán IdentityMap en managed mode.

## DB-ENTITY-MANAGER-092
Query hydration no sobrescribirá dirty state silenciosamente.

## DB-ENTITY-MANAGER-093
Projection results no entrarán al PersistenceContext.

## DB-ENTITY-MANAGER-094
Scalar results no entrarán al PersistenceContext.

## DB-ENTITY-MANAGER-095
Read-only mode tendrá semántica explícita.

## DB-ENTITY-MANAGER-096
EntityManager no implementará relationship loading directamente.

## DB-ENTITY-MANAGER-097
Detached entities no se reconectarán mágicamente para lazy loading.

## DB-ENTITY-MANAGER-098
Entity classes no requerirán almacenar EntityManager.

## DB-ENTITY-MANAGER-099
EntityManager consumirá Transaction System.

## DB-ENTITY-MANAGER-100
EntityManager no duplicará Transaction System.

## DB-ENTITY-MANAGER-101
Transaction rollback no restaurará mágicamente todas las propiedades PHP.

## DB-ENTITY-MANAGER-102
Rollback requerirá reconciliación del PersistenceContext.

## DB-ENTITY-MANAGER-103
After-commit semantics pertenecerán a Transaction integration.

## DB-ENTITY-MANAGER-104
postPersist no significará transaction committed.

## DB-ENTITY-MANAGER-105
Exceptions no implicarán necesariamente zero database effect.

## DB-ENTITY-MANAGER-106
Persistence failures preservarán outcome certainty.

## DB-ENTITY-MANAGER-107
EntityManager será scope-confined.

## DB-ENTITY-MANAGER-108
EntityManager no será process-global singleton.

## DB-ENTITY-MANAGER-109
IdentityMap será scope-local.

## DB-ENTITY-MANAGER-110
UnitOfWork será scope-local.

## DB-ENTITY-MANAGER-111
Managed entities serán scope-local.

## DB-ENTITY-MANAGER-112
Repository instances con manager mutable serán scope-local.

## DB-ENTITY-MANAGER-113
FrankenPHP requests no compartirán EntityManager mutable.

## DB-ENTITY-MANAGER-114
RoadRunner requests no compartirán EntityManager mutable.

## DB-ENTITY-MANAGER-115
OpenSwoole coroutines no compartirán EntityManager mutable.

## DB-ENTITY-MANAGER-116
EntityManager no será asumido thread-safe.

## DB-ENTITY-MANAGER-117
Cross-manager entity use tendrá semántica explícita.

## DB-ENTITY-MANAGER-118
Request cleanup eliminará estado ORM mutable.

## DB-ENTITY-MANAGER-119
Request cleanup no ejecutará flush implícito.

## DB-ENTITY-MANAGER-120
EntityManager destructor no ejecutará persistencia.

## DB-ENTITY-MANAGER-121
EntityManager destructor no ejecutará commit.

## DB-ENTITY-MANAGER-122
Long-running jobs podrán usar flush/clear periódicos.

## DB-ENTITY-MANAGER-123
Streaming tracked tendrá riesgo de crecimiento de memoria observable.

## DB-ENTITY-MANAGER-124
Streaming podrá utilizar detached/read-only/projection modes.

## DB-ENTITY-MANAGER-125
Core EntityManager no hardcodeará `tenant_id`.

## DB-ENTITY-MANAGER-126
Tenant context entrará mediante DatabaseContext.

## DB-ENTITY-MANAGER-127
Un manager no cambiará de tenant durante su lifecycle.

## DB-ENTITY-MANAGER-128
Tenant A y Tenant B utilizarán PersistenceContexts separados.

## DB-ENTITY-MANAGER-129
EntityKey respetará tenant/shard identity namespace cuando corresponda.

## DB-ENTITY-MANAGER-130
EntityManager no elegirá directamente primary/replica.

## DB-ENTITY-MANAGER-131
Read/write routing pertenecerá a sistemas inferiores.

## DB-ENTITY-MANAGER-132
EntityManager declarará intención READ/WRITE.

## DB-ENTITY-MANAGER-133
Sticky reads pertenecerán al routing system.

## DB-ENTITY-MANAGER-134
IdentityMap será distinto de EntityCache.

## DB-ENTITY-MANAGER-135
Second-level cache será integración separada.

## DB-ENTITY-MANAGER-136
EntityManager no realizará autorización implícita.

## DB-ENTITY-MANAGER-137
Poseer EntityId no demostrará autorización.

## DB-ENTITY-MANAGER-138
EntityManager no realizará mass assignment.

## DB-ENTITY-MANAGER-139
Diagnostics podrán redactar identificadores sensibles.

## DB-ENTITY-MANAGER-140
Telemetry no cambiará semántica ORM.

## DB-ENTITY-MANAGER-141
EntityManager deberá ser observable sin filtrar secrets.

## DB-ENTITY-MANAGER-142
Manager state deberá ser visible en diagnostics.

## DB-ENTITY-MANAGER-143
UnitOfWork size deberá poder observarse.

## DB-ENTITY-MANAGER-144
IdentityMap size deberá poder observarse.

## DB-ENTITY-MANAGER-145
Unknown outcome deberá aparecer en telemetry.

## DB-ENTITY-MANAGER-146
Model API utilizará el mismo EntityManager.

## DB-ENTITY-MANAGER-147
Model API no tendrá un persistence engine alternativo.

## DB-ENTITY-MANAGER-148
Repository API utilizará el mismo EntityManager core.

## DB-ENTITY-MANAGER-149
Entity Query utilizará el mismo IdentityMap cuando retorne managed entities.

## DB-ENTITY-MANAGER-150
EntityManager permanecerá una capa de coordinación y no absorberá responsabilidades de subsistemas especializados.

---

# 236. Anti-pattern: God EntityManager

Incorrecto:

```php
final class EntityManager
{
    public function compileSql() {}
    public function connect() {}
    public function hydrate() {}
    public function migrate() {}
    public function createTable() {}
    public function authorize() {}
    public function cache() {}
    public function log() {}
}
```

Correcto:

```text
EntityManager
    ↓ delegates
specialized ORM services
```

---

# 237. Anti-pattern: immediate persistence

Incorrecto:

```php
public function persist(object $entity): void
{
    $this->connection->insert(...);
}
```

Correcto:

```text
persist
→ UnitOfWork

flush
→ Persistence Engine
```

---

# 238. Anti-pattern: static manager

Incorrecto:

```php
final class EntityManager
{
    public static IdentityMap $identityMap;
}
```

Esto sería especialmente peligroso con FrankenPHP.

---

# 239. Anti-pattern: entity holds manager

Incorrecto como requisito universal:

```php
class User
{
    public EntityManager $em;
}
```

El domain model no deberá depender del persistence manager.

---

# 240. Anti-pattern: implicit flush

Incorrecto:

```php
public function __destruct()
{
    $this->flush();
}
```

---

# 241. Anti-pattern: clear closes everything

Incorrecto:

```php
public function clear()
{
    $this->connection->close();
    $this->transaction->rollback();
}
```

---

# 242. Anti-pattern: manager-specific SQL

Incorrecto:

```php
if ($this->driver === 'pgsql') {
    // ...
}
```

EntityManager deberá ser platform-agnostic.

---

# 243. Anti-pattern: tenant switching

Incorrecto:

```php
$em->setTenant($tenantA);

$userA = $em->find(User::class, 1);

$em->setTenant($tenantB);

$userB = $em->find(User::class, 1);
```

Esto podría destruir las garantías de IdentityMap.

---

# 244. Arquitectura de clases propuesta

```text
src/Quantum/Database/ORM/Manager/
│
├── Contract/
│   ├── EntityManager.php
│   ├── EntityFinder.php
│   ├── EntityReferenceProvider.php
│   ├── EntityPersistenceManager.php
│   ├── EntityStateManager.php
│   ├── EntityRepositoryProvider.php
│   └── EntityManagerLifecycle.php
│
├── DefaultEntityManager.php
├── EntityManagerFactory.php
├── DefaultEntityManagerFactory.php
│
├── Context/
│   ├── PersistenceContext.php
│   ├── PersistenceContextId.php
│   ├── EntityManagerScopeId.php
│   └── EntityManagerRuntime.php
│
├── State/
│   ├── EntityManagerState.php
│   ├── EntityManagerStateMachine.php
│   └── EntityManagerTaintReason.php
│
├── Find/
│   ├── EntityLoader.php
│   ├── EntityLoadRequest.php
│   └── EntityLoadResult.php
│
├── Reference/
│   └── EntityReferenceProvider.php
│
├── Refresh/
│   ├── EntityRefresher.php
│   ├── RefreshMode.php
│   └── RefreshResult.php
│
├── Flush/
│   ├── PersistenceCoordinator.php
│   ├── FlushSession.php
│   ├── FlushId.php
│   ├── FlushState.php
│   ├── FlushResult.php
│   └── FlushStatus.php
│
├── Clear/
│   ├── ClearMode.php
│   └── ClearResult.php
│
├── Repository/
│   └── RepositoryResolver.php
│
├── Runtime/
│   ├── EntityManagerScope.php
│   ├── EntityManagerScopeFactory.php
│   └── EntityManagerScopeCleaner.php
│
├── Telemetry/
│   └── EntityManagerTelemetry.php
│
└── Exception/
    ├── EntityManagerException.php
    ├── EntityManagerClosedException.php
    ├── EntityManagerTaintedException.php
    ├── EntityManagerScopeException.php
    ├── EntityManagerReentrancyException.php
    ├── FlushException.php
    ├── FlushReentrancyException.php
    ├── FlushOutcomeUnknownException.php
    └── EntityManagerInvariantException.php
```

---

# 245. Dependencias internas recomendadas

```text
DefaultEntityManager
│
├── DatabaseContext
├── EntityMetadataRegistry
├── IdentityMap
├── UnitOfWork
├── EntityLoader
├── EntityRefresher
├── RepositoryResolver
├── PersistenceCoordinator
└── EntityStateInspector
```

---

# 246. Dependencias que NO deberá recibir

Evitar inyección directa de:

```text
PDO
MySQLConnection
PostgreSQLConnection
SQLCompiler
SchemaManager
MigrationManager
HTTP Request
Current User
```

---

# 247. Public API objetivo

Data Mapper:

```php
$em = $database->entityManager();

$user = $em->find(User::class, $id);

$user->rename('Alice');

$em->flush();
```

---

# 248. Persist new entity

```php
$user = new User(
    UserId::new(),
    'Alice',
);

$em->persist($user);

$em->flush();
```

---

# 249. Remove

```php
$user = $em->find(User::class, $id);

$em->remove($user);

$em->flush();
```

---

# 250. Reference

```php
$author = $em->getReference(
    User::class,
    $authorId,
);

$post = new Post(
    PostId::new(),
    $author,
    'VoltStack ORM',
);

$em->persist($post);

$em->flush();
```

La creación de la referencia no requiere cargar `User`.

---

# 251. Repository

```php
$repository = $em->repository(User::class);

$users = $repository->findActive();
```

---

# 252. Explicit transaction

```php
$transactions->transactional(
    function () use ($em, $order, $payment) {
        $em->persist($order);
        $em->persist($payment);

        $em->flush();
    }
);
```

---

# 253. Multiple flush

```php
$transactions->transactional(
    function () use ($em) {
        $em->persist($account);

        $em->flush();

        $em->persist(
            AuditEntry::forAccount($account)
        );

        $em->flush();
    }
);
```

Ambos flush pertenecen a la misma transacción si el TransactionContext lo establece.

---

# 254. Runtime example

```text
HTTP Request
    ↓
DatabaseContext
    ↓
EntityManagerFactory
    ↓
EntityManager
    ↓
Application
    ↓
flush()
    ↓
Response
    ↓
EntityManagerScopeCleaner
    ↓
clear mutable ORM state
```

---

# 255. Fórmula de EntityManager

```text
EntityManager
=
PersistenceContext Coordinator
+
Entity Lookup API
+
Persistence Registration API
+
Repository Resolution
+
Flush Coordination
+
Lifecycle Management
```

---

# 256. Fórmula de find

```text
find(EntityType, Id)
=
Normalize(Id)
→ EntityKey
→ IdentityMap
→ EntityLoader?
→ Hydration?
→ Managed Entity
```

---

# 257. Fórmula de persist

```text
persist(Entity)
=
ResolveType
+
ValidateIdentity
+
DeterminePersistenceState
+
RegisterInUnitOfWork
+
ApplyCascadeRules
```

No:

```text
persist(Entity)
=
INSERT
```

---

# 258. Fórmula de flush

```text
flush(EntityManager)
=
DetectChanges
→ BuildChangeSets
→ BuildPersistenceGraph
→ PlanPersistence
→ GenerateQueryModels
→ Execute
→ VerifyOutcome
→ ReconcileManagedState
```

---

# 259. Fórmula de identidad

```text
ManagedIdentityGuarantee
=
EntityKey
+
IdentityMap
+
PersistenceContextIsolation
```

---

# 260. Fórmula de seguridad runtime

```text
SafeEntityManagerScope
=
ScopedEntityManager
∧
ScopedIdentityMap
∧
ScopedUnitOfWork
∧
ScopedManagedEntities
∧
ScopedRepositories
∧
ImmutableSharedMetadata
∧
NoImplicitFlushOnCleanup
∧
DeterministicReset
```

---

# 261. Fórmula de resultado incierto

```text
UnknownPersistenceOutcome
⇒
DoNotMarkSynchronized
∧
DoNotBlindRetry
∧
PreserveDiagnostics
∧
TaintOrClosePersistenceContext
```

---

# 262. Master Formula

```text
Database Entity Manager System
=
Scoped Persistence Context
+
EntityManager Factory
+
Entity Lookup Coordination
+
Entity Reference Coordination
+
IdentityMap Integration
+
UnitOfWork Integration
+
Repository Resolution
+
Entity Query Integration
+
Entity Loading
+
Refresh Coordination
+
Persist Registration
+
Remove Registration
+
Flush Coordination
+
Persistence Planning Integration
+
Persistence Outcome Reconciliation
+
Transaction Integration
+
Entity State Inspection
+
Detach/Clear Semantics
+
Lifecycle Coordination
+
Failure Certainty
+
Tainted State Management
+
Persistent Runtime Isolation
+
Multitenancy Context Isolation
+
Read/Write Intent
+
Memory Governance
+
Telemetry
+
Diagnostics
```

---

# 263. Master Rule

> **En VoltStack, EntityManager administra el contexto en el que las entidades son conocidas, identificadas, consultadas, rastreadas y sincronizadas; coordina esos procesos mediante IdentityMap, UnitOfWork, Repository, Hydration y Persistence Engine, pero nunca sustituye a esos subsistemas ni genera SQL directamente.**

---

# 264. Resultado arquitectónico

Después de los documentos `112–118`, la arquitectura ORM queda:

```text
                         Application
                              │
               ┌──────────────┴──────────────┐
               ▼                             ▼
          Model API                    Repository API
               │                             │
               └──────────────┬──────────────┘
                              ▼
                        EntityManager
                              │
       ┌──────────────────────┼──────────────────────┐
       ▼                      ▼                      ▼
 IdentityMap              UnitOfWork           EntityLoader
       │                      │                      │
       │                      ▼                      ▼
       │              Persistence Engine       Entity Query
       │                      │                      │
       └──────────────────────┼──────────────────────┘
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
```

y queda preservada la regla:

```text
Entity
    ↓
EntityManager
    ↓
UnitOfWork
    ↓
Persistence Engine
    ↓
Query Model
    ↓
Query Engine
    ↓
Compiler
    ↓
Execution
```

Nunca:

```text
Entity
→
EntityManager
→
SQL
```

---

# 265. Bloque ORM hasta este punto

```text
112_DATABASE_ORM_ARCHITECTURE
        ↓
113_DATABASE_ENTITY_MODEL
        ↓
114_DATABASE_MODEL_API_SYSTEM
        ↓
115_DATABASE_ENTITY_METADATA_SYSTEM
        ↓
116_DATABASE_ENTITY_MAPPING_SYSTEM
        ↓
117_DATABASE_ATTRIBUTE_MAPPING_SYSTEM
        ↓
118_DATABASE_ENTITY_MANAGER_SYSTEM
```

Con esto ya están definidas las bases de:

```text
Entity
Mapping
Metadata
Developer API
Runtime Persistence Context
```

El siguiente paso es definir formalmente la capa que abstrae las colecciones y consultas orientadas a entidades.

---

# 266. Siguiente documento

```text
119_DATABASE_REPOSITORY_SYSTEM.md
```

Este documento deberá definir:

```text
EntityRepository
RepositoryRegistry
RepositoryResolver
RepositoryFactory
Default Repository
Custom Repository
Entity-aware query delegation
Repository scope
Repository caching
Repository contracts
```

y preservar especialmente:

```text
Repository
≠
EntityManager
≠
Query Builder
≠
Business Service
≠
Persistence Engine
```

La arquitectura continuará:

```text
Application
    ↓
Repository
    ↓
Entity Query
    ↓
EntityManager / Persistence Context
    ↓
Query Engine
```

manteniendo una regla fundamental:

> **Un Repository representa una interfaz orientada a una colección lógica de entidades y sus consultas; no debe convertirse en un contenedor de lógica de negocio ni en un segundo motor de persistencia.**