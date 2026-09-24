# 131_DATABASE_DELETE_PERSISTENCE_SYSTEM.md

# VoltStack Quantum Database
## Database Delete Persistence System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 131 — Database Delete Persistence System  
**Bloque:** 11 — Identity Map, Unit of Work & Persistence  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Delete Persistence System` define la arquitectura responsable de transformar la intención ORM de eliminar una entidad en una operación persistente de eliminación segura, planificada, verificable y reconciliable.

El sistema cubre tanto la eliminación física:

```text
DELETE FROM ...
```

como estrategias semánticas alternativas, por ejemplo:

```text
soft delete
temporal retirement
archival transition
extension-defined removal
```

sin permitir que la capa ORM genere SQL directamente.

Flujo conceptual:

```text
MANAGED Entity
      ↓
remove(entity)
      ↓
Removal Intent
      ↓
UnitOfWork
      ↓
REMOVED / SCHEDULED
      ↓
Persistence Engine
      ↓
DeleteEntityOperation
      ↓
Persistence Planner
      ↓
Relationship / Dependency Analysis
      ↓
DeletePersistenceStep
      ↓
Delete Persistence System
      ↓
Delete Query Model
      ↓
Query Engine
      ↓
Execution Engine
      ↓
Database
      ↓
Delete Execution Outcome
      ↓
Delete Reconciliation
      ↓
IdentityMap / Snapshot / UoW
      ↓
DETACHED
```

Principio central:

> **`remove()` en VoltStack registra intención de eliminación; nunca significa que la fila haya sido eliminada inmediatamente. La transición final de una entidad desde el contexto administrado solo puede producirse después de una ejecución suficientemente cierta y de la reconciliación coherente del `IdentityMap`, snapshot, `UnitOfWork` y estado ORM.**

---

# 2. Responsabilidad

El sistema responde:

> ¿Cómo convertir una intención ORM de eliminación en una operación persistente segura sin confundir intención, ejecución, eliminación física y commit transaccional?

No responde directamente:

```text
¿Cómo detectar una relación?
¿Cómo generar SQL?
¿Cómo abrir la conexión?
¿Cómo hacer commit?
¿Cómo implementar autorización?
¿Cómo ejecutar PDO?
¿Cómo decidir una política de retención?
```

---

# 3. Posición arquitectónica

```text
EntityManager::remove($entity)
             │
             ▼
         UnitOfWork
             │
             ▼
      Removal Scheduling
             │
             ▼
      Persistence Engine
             │
             ▼
    DeleteEntityOperation
             │
             ▼
    Persistence Planner
             │
      ┌──────┴──────┐
      ▼             ▼
 Dependencies    Cascades
      │             │
      └──────┬──────┘
             ▼
 DeletePersistenceStep
             │
             ▼
 Delete Persistence System
             │
             ▼
    Delete Query Model
             │
             ▼
        Query Engine
             │
             ▼
        SQL Compiler
             │
             ▼
      Execution Engine
             │
             ▼
          Database
             │
             ▼
    Execution Outcome
             │
             ▼
       Reconciliation
```

---

# 4. Separaciones fundamentales

```text
remove()
≠ DELETE execution

REMOVED
≠ deleted from database

Delete Persistence
≠ SQL Compiler

Delete Persistence
≠ Execution Engine

Delete Persistence
≠ Transaction Manager

Delete Persistence
≠ Bulk Delete

Delete Persistence
≠ Soft Delete Policy

postRemove
≠ transaction commit

DETACHED
≠ deleted globally

Row missing
≠ successful ORM delete automatically
```

---

# 5. `remove()` como intención

Ejemplo:

```php
$entityManager->remove($user);
```

No deberá producir inmediatamente:

```sql
DELETE FROM users WHERE id = ?
```

En su lugar:

```text
remove(user)
    ↓
validate entity/context
    ↓
schedule removal
    ↓
UnitOfWork
    ↓
Persistence Plan
    ↓
flush()
    ↓
execution
```

---

# 6. Razón

La eliminación puede depender de:

```text
foreign keys
cascade rules
orphan removal
optimistic locking
entity lifecycle
transaction state
tenant context
shard routing
soft-delete policy
other scheduled operations
```

Por ello:

```text
remove()
```

no posee suficiente contexto para ejecutar inmediatamente.

---

# 7. Estado REMOVED

Una entidad programada para eliminación entra conceptualmente en:

```text
REMOVED
```

pero:

```text
REMOVED
≠
row physically absent
```

---

# 8. REMOVED como estado ORM

Representa:

> Esta entidad continúa perteneciendo al `PersistenceContext`, pero existe una intención activa de eliminar su representación persistente.

---

# 9. Estado conceptual

```text
MANAGED
   │
 remove()
   ▼
REMOVED
   │
 flush()
   ▼
Delete Scheduled
   │
 execute
   ▼
Delete Outcome
```

---

# 10. Resultado exitoso

Con certeza suficiente:

```text
REMOVED
   ↓
DELETE succeeds
   ↓
ORM reconciliation
   ↓
DETACHED
```

---

# 11. Resultado fallido

```text
REMOVED
   ↓
DELETE fails certainly
   ↓
failure policy
```

La entidad no deberá declararse eliminada como si la operación hubiera ocurrido.

---

# 12. Resultado desconocido

```text
REMOVED
   ↓
DELETE sent
   ↓
connection lost
   ↓
UNKNOWN
```

No se puede afirmar:

```text
row exists
```

ni:

```text
row does not exist
```

---

# 13. DeleteEntityOperation

```php
final readonly class DeleteEntityOperation
{
    public function __construct(
        public PersistenceOperationId $id,
        public object $entity,
        public EntityKey $entityKey,
        public EntityMetadata $metadata,
        public DeletePersistenceIntent $intent,
        public PersistenceOperationContext $context,
    ) {}
}
```

---

# 14. DeletePersistenceIntent

```php
enum DeletePersistenceIntent
{
    case PHYSICAL_DELETE;
    case SOFT_DELETE;
    case TEMPORAL_RETIREMENT;
    case ARCHIVAL_TRANSITION;
    case EXTENSION_DEFINED;
}
```

---

# 15. Semantic delete

El ORM trabaja inicialmente con:

```text
Remove Entity
```

no necesariamente con:

```text
SQL DELETE
```

---

# 16. Removal strategy

La estrategia efectiva será resuelta por:

```php
interface DeleteStrategyResolver
{
    public function resolve(
        EntityMetadata $metadata,
        DeletePersistenceContext $context,
    ): DeletePersistenceStrategy;
}
```

---

# 17. DeletePersistenceStrategy

```php
enum DeletePersistenceStrategy
{
    case PHYSICAL;
    case SOFT;
    case TEMPORAL;
    case ARCHIVAL;
    case CUSTOM;
}
```

---

# 18. Physical delete

Representa eliminación física de la representación relacional principal.

Conceptualmente:

```text
EntityKey
   ↓
Delete Predicate
   ↓
DELETE Query Model
```

---

# 19. Soft delete

Puede representar:

```text
deleted_at = timestamp
```

o:

```text
status = DELETED
```

mediante una operación física `UPDATE`.

---

# 20. Regla crítica

```text
ORM Remove Semantics
≠
Physical SQL DELETE
```

---

# 21. Soft delete architecture

El ORM puede recibir:

```text
RemoveEntityOperation
```

y resolver:

```text
SoftDeletePersistenceStep
```

que finalmente produzca un:

```text
UpdateQueryModel
```

---

# 22. Soft delete no duplica Update System

Cuando una eliminación semántica requiere `UPDATE`, deberá reutilizar los componentes apropiados del pipeline de actualización.

---

# 23. Soft delete ≠ ordinary update

Sin embargo, semánticamente sigue siendo:

```text
Entity Removal
```

por lo que lifecycle/state/query scopes deberán tratarla como eliminación.

---

# 24. Future dedicated system

La política completa de soft delete será formalizada en:

```text
269_DATABASE_SOFT_DELETE_SYSTEM.md
```

Este documento define únicamente el punto de integración.

---

# 25. Contrato principal

```php
interface DeletePersistenceSystem
{
    public function prepare(
        DeletePersistenceRequest $request,
    ): PreparedDeletePersistence;

    public function reconcile(
        PreparedDeletePersistence $delete,
        DeleteExecutionOutcome $outcome,
    ): DeleteReconciliationResult;
}
```

---

# 26. DeletePersistenceRequest

```php
final readonly class DeletePersistenceRequest
{
    public function __construct(
        public DeleteEntityOperation $operation,
        public EntityMetadata $metadata,
        public PersistencePlanningContext $planningContext,
        public PersistenceExecutionContext $executionContext,
    ) {}
}
```

---

# 27. PreparedDeletePersistence

```php
final readonly class PreparedDeletePersistence
{
    public function __construct(
        public DeletePersistenceId $id,
        public PersistenceOperationId $operationId,
        public EntityKey $entityKey,
        public DeletePersistenceStrategy $strategy,
        public DeletePredicateSet $predicates,
        public DeleteConcurrencyRequirement $concurrency,
        public DeleteExecutionRequirementSet $requirements,
        public DeleteReplaySafety $replaySafety,
        public DeletePersistenceFingerprint $fingerprint,
    ) {}
}
```

---

# 28. Prepared delete ≠ executed delete

```text
PreparedDeletePersistence
```

solo representa una operación lista para traducción/ejecución.

No demuestra cambio en DB.

---

# 29. Identidad requerida

Una eliminación física ordinaria requiere:

```text
Established EntityKey
```

---

# 30. NEW entity removal

Caso:

```php
$user = new User();

$entityManager->persist($user);
$entityManager->remove($user);
$entityManager->flush();
```

Si nunca fue insertada:

```text
NEW
↓
remove()
↓
cancel scheduled INSERT
↓
no database DELETE
```

---

# 31. NEW → remove

Debe poder reducirse a:

```text
NO_PERSISTENCE_OPERATION
```

cuando no existe efecto persistente previo.

---

# 32. Generated identity not required

Una entidad NEW sin ID generada por DB puede eliminarse del UoW sin inventar un identificador.

---

# 33. Insert + delete same flush

Si una entidad fue insertada físicamente dentro del mismo flush antes de ser eliminada por una operación posterior, el planner deberá decidir la secuencia correcta.

---

# 34. Optimization

Si ninguna operación ha alcanzado la DB:

```text
INSERT + DELETE
```

puede colapsarse a:

```text
NO_OP
```

cuando sea semánticamente seguro.

---

# 35. Side effects matter

No colapsar si existen efectos observables requeridos:

```text
lifecycle
audit
outbox
relationship dependencies
database-generated semantics
```

sin verificar políticas.

---

# 36. Detached entity removal

Por defecto:

```text
remove(DETACHED entity)
```

deberá fallar.

---

# 37. Razón

El ORM no posee necesariamente:

```text
current snapshot
current version
canonical managed identity
relationship state
```

para eliminarla correctamente.

---

# 38. Detached removal

Podrá requerir:

```text
find()
merge/re-attach explicit API
reference-based delete API
repository deleteById()
```

según contratos futuros.

---

# 39. Reference deletion

Puede existir una API explícita:

```php
$repository->deleteById($id);
```

pero:

```text
deleteById()
≠
remove(managed entity)
```

---

# 40. Entity lifecycle difference

Un delete directo por ID no posee necesariamente una instancia materializada.

Por tanto no debe prometer:

```text
per-entity lifecycle callbacks
```

sin materializar la entidad.

---

# 41. Identifier preservation

Eliminar una entidad no modifica automáticamente su identifier en memoria.

---

# 42. Después de delete

Puede quedar:

```php
$user->id(); // UserId(42)
```

aunque:

```text
EntityState = DETACHED
```

---

# 43. Razón

La identidad histórica del objeto puede seguir siendo útil para:

```text
domain events
logging
audit
diagnostics
references
```

---

# 44. Deleted entity ≠ anonymous entity

No limpiar:

```text
id = null
```

automáticamente.

---

# 45. Re-persist deleted object

No deberá asumirse automáticamente que:

```php
$entityManager->persist($deletedEntity);
```

significa:

```text
INSERT again
```

---

# 46. Reinsertion policy

Debe ser explícita.

Podría requerir:

```text
new entity instance
new identity
explicit restore
soft-delete restore
```

---

# 47. Delete predicates

El sistema construye predicates semánticos.

```php
final readonly class DeletePredicateSet
{
    public function __construct(
        public array $predicates,
    ) {}
}
```

---

# 48. Identity predicate

Como mínimo:

```text
EntityIdentityPredicate
```

---

# 49. Simple identity

Conceptualmente:

```text
id = 42
```

---

# 50. Composite identity

Conceptualmente:

```text
order_id = 100
AND
line_number = 4
```

---

# 51. Query Model

No se crean strings SQL.

Se crea:

```text
DeleteQueryModel
```

---

# 52. Tenant predicate

En shared-schema multitenancy:

```text
tenant_id = currentTenant
```

puede formar parte del predicate.

---

# 53. Tenant protection

Una eliminación ORM jamás deberá permitir que:

```text
EntityKey tenant A
```

produzca una eliminación en:

```text
tenant B
```

---

# 54. Namespace consistency

Debe cumplirse:

```text
EntityKey.IdentityNamespace
=
PersistenceContext.IdentityNamespace
```

cuando el mapping así lo requiera.

---

# 55. Shard routing

La eliminación debe dirigirse al shard correspondiente a la identidad/contexto.

---

# 56. Shard mismatch

Produce error antes de ejecución cuando sea detectable.

---

# 57. Optimistic locking

Una entidad versionada puede requerir:

```text
identity
+
expected version
```

para eliminación.

---

# 58. Delete optimistic predicate

Conceptualmente:

```sql
DELETE FROM users
WHERE id = ?
  AND version = ?
```

La SQL es ilustrativa.

---

# 59. Semantic model

```text
DeletePredicate
├── Identity = User#42
└── ExpectedVersion = 7
```

---

# 60. Optimistic delete success

```text
ExpectedVersion matches
+
target exists
+
delete succeeds
```

---

# 61. Optimistic delete conflict

Si:

```text
affectedRows = 0
```

bajo un contrato donde se esperaba exactamente una fila:

```text
OptimisticLockException
```

---

# 62. Missing vs version conflict

No siempre puede saberse si:

```text
row missing
```

o:

```text
version mismatch
```

---

# 63. No fabricated diagnosis

VoltStack no afirmará una causa específica si la DB no permite observarla con certeza.

---

# 64. Delete cardinality expectation

Una eliminación de entidad por identidad normalmente espera:

```text
0 or 1 candidate row
```

y semánticamente puede exigir:

```text
exactly 1
```

según política.

---

# 65. DeleteCardinalityExpectation

```php
enum DeleteCardinalityExpectation
{
    case EXACTLY_ONE;
    case ZERO_OR_ONE;
    case EXTENSION_DEFINED;
}
```

---

# 66. Strict managed delete

Para una entidad `MANAGED` que se cree persistente:

```text
EXACTLY_ONE
```

puede ser la política predeterminada.

---

# 67. Already missing row

Si una entidad fue eliminada externamente:

```text
IdentityMap still contains entity
↓
remove()
↓
DELETE affects 0
```

VoltStack deberá aplicar una política explícita.

---

# 68. MissingDeleteTargetPolicy

```php
enum MissingDeleteTargetPolicy
{
    case FAIL;
    case ACCEPT_AS_ALREADY_ABSENT;
    case OPTIMISTIC_CONFLICT;
    case EXTENSION_DEFINED;
}
```

---

# 69. Default

Para managed entity:

```text
FAIL
```

o:

```text
OPTIMISTIC_CONFLICT
```

es más seguro que fingir éxito.

---

# 70. Idempotent delete API

Un API diferente puede definir:

```text
ensure entity absent
```

y aceptar:

```text
already absent
```

como éxito.

---

# 71. Semantic distinction

```text
Delete this managed entity
```

no es idéntico a:

```text
Ensure row X does not exist
```

---

# 72. Delete replay safety

DELETE por identidad puede parecer idempotente:

```text
DELETE #1 → row removed
DELETE #2 → 0 rows
```

pero la semántica ORM puede no serlo.

---

# 73. Side effects

Puede haber:

```text
triggers
audit
cascade deletes
outbox records
lifecycle events
temporal history
```

---

# 74. DeleteReplaySafety

```php
enum DeleteReplaySafety
{
    case IDEMPOTENT;
    case CONDITIONALLY_SAFE;
    case UNSAFE;
    case UNKNOWN;
}
```

---

# 75. Retry ownership

El Delete System declara replay semantics.

No ejecuta retry global arbitrariamente.

---

# 76. UNKNOWN delete

Caso crítico:

```text
DELETE sent
↓
DB executes
↓
network disconnects
↓
client receives no result
```

---

# 77. Estado

```text
DeleteExecutionOutcome = UNKNOWN
```

---

# 78. UNKNOWN ≠ failed

No puede mantenerse la entidad como si se supiera que la fila existe.

---

# 79. UNKNOWN ≠ succeeded

Tampoco puede detacharse como si se supiera que la fila desapareció.

---

# 80. EntityManager taint

Una incertidumbre no reconciliable deberá poder producir:

```text
EntityManager = TAINTED
```

---

# 81. IdentityMap under UNKNOWN

No eliminar la entrada silenciosamente.

---

# 82. Snapshot under UNKNOWN

No destruirlo silenciosamente.

---

# 83. UnitOfWork under UNKNOWN

Debe conservar evidencia suficiente del intento.

---

# 84. Reconciliation required

Opciones posteriores:

```text
transaction rollback
transaction outcome resolution
discard persistence context
explicit reload/reconciliation
safe retry
```

---

# 85. Physical delete preparation

```text
DeleteEntityOperation
      ↓
Validate state
      ↓
Resolve identity
      ↓
Resolve strategy
      ↓
Resolve dependencies
      ↓
Resolve concurrency
      ↓
Build predicates
      ↓
Build execution requirements
      ↓
PreparedDeletePersistence
```

---

# 86. Delete Query Model translation

```php
interface DeletePersistenceQueryTranslator
{
    public function translate(
        PreparedDeletePersistence $delete,
        DeletePersistenceTranslationContext $context,
    ): DeleteQueryModel;
}
```

---

# 87. DeleteQueryModel

Conceptualmente:

```text
DeleteQueryModel
├── target
├── predicates
├── parameters
├── concurrency metadata
├── cardinality expectation
└── execution metadata
```

---

# 88. SQL remains downstream

```text
DeleteQueryModel
↓
Semantic Engine
↓
Optimizer
↓
Planner
↓
Compiler
↓
SQL
```

---

# 89. No raw SQL in ORM

Prohibido:

```php
$sql = "DELETE FROM {$table} WHERE id = ?";
```

---

# 90. Cascade remove

Mapping puede declarar:

```text
cascade REMOVE
```

---

# 91. Cascade remove ≠ database ON DELETE CASCADE

Son mecanismos diferentes.

---

# 92. ORM cascade

```text
remove(parent)
↓
discover mapped cascade
↓
schedule child removals
↓
PersistenceOperationGraph
```

---

# 93. Database cascade

```text
DELETE parent
↓
DB FK ON DELETE CASCADE
↓
DB deletes children
```

---

# 94. No double deletion

Si DB cascade ya resuelve físicamente una dependencia, el ORM no deberá generar deletes redundantes sin razón.

---

# 95. But ORM state matters

Aunque DB elimine hijos automáticamente, el ORM puede tener hijos:

```text
MANAGED
```

en el `IdentityMap`.

---

# 96. Reconciliation

Debe existir política para esos objetos.

Ejemplo:

```text
DB-cascaded child
↓
mark/detach affected managed child
```

cuando el ORM conozca el alcance.

---

# 97. Unknown cascade population

Si DB puede eliminar filas que el ORM no tiene materializadas:

```text
no need to materialize them merely to detach
```

---

# 98. Cascade planning

Debe evitar:

```text
load entire object graph
```

si la semántica puede resolverse mediante metadata/DB constraints.

---

# 99. Cascade cycle

Ejemplo:

```text
A removes B
B removes A
```

No debe producir recursión infinita.

---

# 100. Graph traversal

Utilizar:

```text
visited EntityKey set
```

o mecanismo equivalente.

---

# 101. Cascade scheduling idempotency

La misma entidad descubierta por dos rutas de cascade deberá programarse una sola vez.

---

# 102. Orphan removal

Ejemplo:

```php
$order->removeLine($line);
```

Mapping:

```text
orphanRemoval = true
```

puede producir:

```text
DeleteEntityOperation(Line)
```

---

# 103. Orphan removal ≠ cascade remove

```text
Cascade Remove:
parent removed → child removed
```

```text
Orphan Removal:
relationship ownership lost → child removed
```

---

# 104. Orphan detection

Pertenece a relación/UoW/change tracking.

Delete Persistence consume la operación resultante.

---

# 105. Orphan reattachment

Si durante el mismo UoW:

```text
child removed from parent A
↓
child attached to parent B
```

el planner debe evaluar si realmente continúa siendo orphan.

---

# 106. No premature delete

No generar DELETE en el momento de modificar la colección.

---

# 107. Persistence stabilization

La decisión se toma sobre el estado estabilizado del UoW.

---

# 108. Relationship dependency ordering

Ejemplo:

```text
Parent
└── Child FK NOT NULL
```

Sin DB cascade:

```text
DELETE child
↓
DELETE parent
```

---

# 109. Nullable FK

Otra estrategia:

```text
UPDATE child SET parent_id = NULL
↓
DELETE parent
```

---

# 110. Restrict

Si semántica es:

```text
ON DELETE RESTRICT
```

y la relación sigue existiendo:

```text
delete may be invalid
```

---

# 111. Planner responsibility

`Persistence Planner` determina:

```text
operation ordering
dependency graph
cycle handling
```

Delete System ejecuta su step.

---

# 112. Delete-before-insert

Puede existir una restricción unique donde sea necesario:

```text
DELETE old row
↓
INSERT replacement
```

---

# 113. Insert-before-delete

Otros grafos pueden requerir:

```text
INSERT replacement
↓
UPDATE references
↓
DELETE old entity
```

---

# 114. No universal ordering

No existe:

```text
always INSERT → UPDATE → DELETE
```

como regla absoluta.

---

# 115. Planner graph wins

La topología real depende de constraints y operaciones.

---

# 116. Lifecycle preRemove

`preRemove` ocurre cuando la eliminación semántica está realmente siendo preparada.

---

# 117. preRemove ≠ remove()

Recomendación:

```text
remove()
→ schedule

flush()
→ preRemove
```

---

# 118. Razón

Entre ambos pueden ocurrir:

```text
cancel removal
UoW stabilization
relationship changes
operation collapse
```

---

# 119. preRemove mutation

Debe estar restringida.

---

# 120. Why restricted

Modificar campos justo antes de eliminar físicamente puede no tener ningún efecto persistente y producir semántica confusa.

---

# 121. Explicit policy

El lifecycle context debe indicar qué mutaciones son permitidas.

---

# 122. preRemove veto

No usar:

```php
return false;
```

como veto implícito.

---

# 123. Veto explícito

Usar:

```text
domain validation
policy exception
explicit cancellation API
```

si se diseña.

---

# 124. postRemove

Se ejecuta después de una eliminación semánticamente exitosa y reconciliada.

---

# 125. postRemove ≠ commit

Regla crítica:

```text
postRemove
⇏
transaction committed
```

---

# 126. External effects

No enviar directamente desde `postRemove`:

```text
email
webhook
irreversible API request
broker publication
```

si requieren durabilidad.

---

# 127. After commit

Utilizar:

```text
Transaction Synchronization
Transactional Outbox
After-Commit Job
```

---

# 128. Soft-delete lifecycle

Un soft delete deberá producir semántica:

```text
preRemove
↓
soft-delete physical UPDATE
↓
postRemove
```

no:

```text
preUpdate/postUpdate
```

como evento principal, salvo eventos internos específicos.

---

# 129. Razón

El dominio ORM solicitó:

```text
remove(entity)
```

---

# 130. Event semantics > SQL verb

Regla:

> El evento ORM describe la operación semántica, no necesariamente el verbo SQL físico.

---

# 131. Physical delete success

Con certeza suficiente:

```text
DatabaseOperationOutcome = SUCCEEDED
```

---

# 132. Reconciliation sequence

Recomendación:

```text
physical/semantic delete succeeds
↓
validate affected rows
↓
reconcile cascades
↓
remove snapshot
↓
remove IdentityMap registration
↓
transition entity → DETACHED
↓
complete UoW operation
↓
postRemove
```

---

# 133. Atomic in-memory reconciliation

Los cambios de:

```text
IdentityMap
SnapshotRegistry
EntityStateRegistry
UnitOfWork
```

deberán coordinarse para evitar estado parcial.

---

# 134. Reconciliation transaction

No es una transacción DB.

Es una operación interna coherente sobre estructuras del `PersistenceContext`.

---

# 135. Failure during reconciliation

Caso:

```text
DB delete succeeded
↓
IdentityMap removed
↓
Snapshot removal throws
```

Esto es crítico.

---

# 136. Response

```text
EntityManager → TAINTED
```

si no puede restaurarse de manera segura.

---

# 137. No fake DB rollback

No afirmar que la fila volvió porque falló una estructura en memoria.

---

# 138. IdentityMap removal

Solo tras resultado suficientemente cierto.

---

# 139. Reverse index

Debe eliminarse también:

```text
object → EntityKey
```

---

# 140. Snapshot disposal

El snapshot administrado deja de representar una entidad activa.

---

# 141. Historical diagnostics

Podrá conservarse metadata diagnóstica separada, pero no como snapshot activo.

---

# 142. Entity state

Tras reconciliación:

```text
REMOVED
→
DETACHED
```

---

# 143. Object remains alive

PHP puede seguir teniendo referencias:

```php
$user
```

---

# 144. Detached semantics

Ese objeto ya no forma parte del `PersistenceContext`.

---

# 145. Lazy relationships after delete

Una entidad detached no deberá disparar lazy loading usando implícitamente el antiguo EntityManager.

---

# 146. Proxy after delete

Un proxy eliminado/detached tampoco deberá reconectarse silenciosamente.

---

# 147. Generated ID after delete

El ID puede permanecer.

---

# 148. Snapshot after delete

No.

---

# 149. IdentityMap entry after delete

No.

---

# 150. UnitOfWork scheduled operation after delete

No.

---

# 151. State registry after delete

Puede registrar:

```text
DETACHED
```

si conserva tracking externo, o eliminar registro y derivarlo como detached/untracked según arquitectura.

---

# 152. EntityState consistency

No deberá existir:

```text
DETACHED
+
IdentityMap contains entity
```

como estado estable.

---

# 153. Transaction separation

Un delete puede ser ejecutado dentro de:

```text
Transaction = ACTIVE
```

---

# 154. ORM reconciliation before commit

Puede ocurrir:

```text
DELETE succeeds
↓
ORM marks entity DETACHED
↓
transaction still ACTIVE
```

---

# 155. Later rollback

```text
ROLLBACK
```

puede restaurar la fila en DB.

---

# 156. Important problem

En memoria:

```text
entity = DETACHED
```

pero DB después del rollback:

```text
row exists again
```

---

# 157. Rollback reconciliation

Debe existir una política coordinada con:

```text
Transaction Manager
UnitOfWork
Persistence Context
```

---

# 158. Possible strategies

```text
restore managed state from rollback journal
invalidate/clear PersistenceContext
mark manager tainted
require explicit refresh
```

---

# 159. No naïve reattach

No basta con:

```text
DETACHED → MANAGED
```

porque:

```text
relationships
snapshots
versions
generated values
other operations
```

pueden haber cambiado.

---

# 160. Future transaction architecture

La política definitiva se formalizará en:

```text
164_DATABASE_TRANSACTION_ARCHITECTURE.md
165_DATABASE_TRANSACTION_MANAGER_SYSTEM.md
```

y documentos relacionados.

---

# 161. Rollback journal integration

Persistence Engine podrá producir suficiente información para futura reconciliación:

```text
DeletedEntityRollbackRecord
```

---

# 162. Conceptual record

```php
final readonly class DeletedEntityRollbackRecord
{
    public function __construct(
        public EntityKey $entityKey,
        public EntitySnapshot $snapshotBeforeDelete,
        public EntityState $stateBeforeDelete,
        public PersistenceOperationId $operationId,
    ) {}
}
```

---

# 163. Rollback record ≠ guarantee

Es evidencia para reconciliación.

No significa que siempre pueda restaurarse el objeto de manera segura.

---

# 164. Database cascades and rollback

La restauración de cascaded managed entities puede ser compleja.

Por ello:

```text
clear context after rollback
```

puede ser política más segura en ciertos casos.

---

# 165. Soft delete rollback

También requiere reconciliar:

```text
deletedAt
snapshot
entity state
query visibility
```

---

# 166. Delete execution requirements

```php
final readonly class DeleteExecutionRequirementSet
{
    public function __construct(
        public bool $requiresAffectedRows,
        public bool $requiresSameConnection,
        public bool $requiresTransaction,
        public bool $requiresOptimisticLockValidation,
        public bool $requiresCascadeReconciliation,
    ) {}
}
```

---

# 167. Write intent

Toda eliminación persistente declara:

```text
ConnectionIntent = WRITE
```

---

# 168. Primary routing

La eliminación no deberá dirigirse a una read replica ordinaria.

---

# 169. Connection ownership

Delete Persistence no abre conexiones directamente.

---

# 170. Transaction ownership

Delete Persistence no comienza transacciones arbitrariamente.

---

# 171. DeleteExecutionOutcome

```php
final readonly class DeleteExecutionOutcome
{
    public function __construct(
        public DeleteExecutionStatus $status,
        public PersistenceExecutionCertainty $certainty,
        public ?int $affectedRows,
        public DeleteExecutionDiagnosticCollection $diagnostics,
    ) {}
}
```

---

# 172. Status

```php
enum DeleteExecutionStatus
{
    case SUCCEEDED;
    case FAILED;
    case UNKNOWN;
}
```

---

# 173. Success requirements

No basta:

```text
driver did not throw
```

---

# 174. Semantic success

Puede requerir:

```text
execution succeeded
+
outcome certain
+
cardinality satisfied
+
optimistic policy satisfied
+
required cascade semantics satisfied
```

---

# 175. Delete reconciliation result

```php
final readonly class DeleteReconciliationResult
{
    public function __construct(
        public DeleteReconciliationStatus $status,
        public EntityState $resultingEntityState,
        public bool $identityMapDetached,
        public bool $snapshotRemoved,
        public bool $unitOfWorkReconciled,
    ) {}
}
```

---

# 176. Reconciliation status

```php
enum DeleteReconciliationStatus
{
    case RECONCILED;
    case PARTIALLY_RECONCILED;
    case NOT_RECONCILED;
    case UNKNOWN;
}
```

---

# 177. Partial reconciliation

No deberá ocultarse.

---

# 178. Taint policy

```text
PARTIALLY_RECONCILED
→
EntityManager TAINTED
```

por defecto.

---

# 179. Bulk delete

Ejemplo:

```php
User::query()
    ->where('inactive', true)
    ->delete();
```

no es equivalente a:

```text
remove(entity) × N
```

---

# 180. Bulk delete may not materialize entities

Por tanto no puede garantizar:

```text
preRemove per entity
postRemove per entity
snapshot reconciliation per entity
IdentityMap detachment per entity
```

sin estrategia adicional.

---

# 181. Bulk delete stale state

Si existen entidades afectadas ya administradas:

```text
IdentityMap may become stale
```

---

# 182. BulkDeleteManagedStatePolicy

```php
enum BulkDeleteManagedStatePolicy
{
    case CLEAR_CONTEXT;
    case MARK_POTENTIALLY_STALE;
    case REJECT_IF_MANAGED_ENTITIES_MAY_BE_AFFECTED;
    case NO_AUTOMATIC_RECONCILIATION;
}
```

---

# 183. Default conservative behavior

Debe evitar fingir que conoce exactamente las filas afectadas si no las materializó.

---

# 184. Delete all guard

Una operación sin predicates:

```text
DELETE all rows
```

no deberá surgir accidentalmente de entity removal.

---

# 185. Entity delete always identity-scoped

Para eliminación de una entidad administrada:

```text
identity predicate required
```

---

# 186. Empty predicate protection

Si la traducción produce:

```text
PredicateSet = ∅
```

para physical entity delete:

```text
UnsafeDeletePredicateException
```

---

# 187. Security boundary

Este guard no reemplaza autorización.

---

# 188. SQL injection

Parameters siguen usando:

```text
typed parameter binding
```

---

# 189. No identifier interpolation

No:

```php
"DELETE ... WHERE id = {$id}"
```

---

# 190. Database trigger behavior

DELETE puede activar:

```text
audit trigger
cascade trigger
history trigger
notification trigger
```

---

# 191. Trigger side effects

No deberán asumirse:

```text
idempotent
reversible
observable
```

por defecto.

---

# 192. Retry impact

Por ello replay safety puede ser:

```text
UNKNOWN
```

incluso si el DELETE principal parece idempotente.

---

# 193. Temporal databases

Una eliminación puede convertirse en:

```text
close validity interval
```

en lugar de borrar físicamente.

---

# 194. Archival strategy

Puede convertirse en:

```text
copy/move to archive
+
remove active representation
```

---

# 195. Multi-step removal

Estas estrategias pueden generar múltiples `PersistenceStep`.

---

# 196. Planner integration

No deben ejecutarse como SQL ad hoc desde el entity persister.

---

# 197. Extension architecture

```php
interface DeletePersistenceExtension
{
    public function contribute(
        DeletePersistenceExtensionContext $context,
    ): DeletePersistenceContribution;
}
```

---

# 198. Extension use cases

```text
tenant predicates
audit metadata
soft-delete strategy
temporal strategy
retention rules
custom concurrency predicate
custom delete capability
```

---

# 199. Extension restrictions

No podrán:

```text
remove identity predicate
bypass tenant boundary
silently bypass optimistic locking
commit
rollback
execute arbitrary SQL
mutate process-global state
```

---

# 200. Predicate contribution

Cada predicate tendrá provenance.

---

# 201. Required predicate

Puede marcarse:

```text
NON_REMOVABLE
```

por ejemplo:

```text
tenant discriminator
identity
security invariant
optimistic version
```

según contexto.

---

# 202. Extension conflict

Dos estrategias incompatibles:

```text
PHYSICAL_DELETE
vs
SOFT_DELETE
```

sin resolución explícita deberán producir error.

---

# 203. DeleteStrategyConflictException

No:

```text
last registered wins
```

---

# 204. Persistent runtime

FrankenPHP mantiene workers vivos.

Por ello:

```text
DeletePersistenceRequest
PreparedDeletePersistence
Entity
Snapshot
Removal Intent
Execution Outcome
```

son scoped.

---

# 205. Shared immutable state

Puede compartirse:

```text
compiled metadata
mapping definitions
frozen strategy registry
compiled accessors
capability descriptors
```

---

# 206. Scoped mutable state

Nunca compartir entre requests:

```text
EntityManager
UnitOfWork
IdentityMap
EntityStateRegistry
SnapshotRegistry
tenant
transaction
delete operation
```

---

# 207. No global pending deletes

Prohibido:

```php
static array $pendingDeletes;
```

---

# 208. Worker reset

Al terminar scope:

```text
pending removal state = cleared
```

---

# 209. No implicit flush on reset

El reset nunca deberá ejecutar pendientes automáticamente.

---

# 210. Request termination

Si existen deletes no flushed:

```text
discard according to lifecycle policy
```

No:

```text
auto DELETE
```

---

# 211. OpenSwoole

Cada coroutine requiere contexto aislado.

---

# 212. Concurrent delete scheduling

Un mismo `EntityManager` no admite mutación concurrente sin sincronización explícita.

---

# 213. Duplicate remove()

Caso:

```php
$em->remove($user);
$em->remove($user);
```

debe ser:

```text
idempotent scheduling
```

---

# 214. Duplicate scheduling ≠ duplicate DELETE

Solo una operación de eliminación.

---

# 215. remove() after scheduled remove

No genera un segundo lifecycle preRemove/postRemove.

---

# 216. persist() after remove()

Debe tener política explícita.

Posibilidades:

```text
cancel removal
reject transition
restore managed state
```

---

# 217. Recommended V1

Antes de ejecutar:

```text
persist(REMOVED entity)
```

puede cancelar la eliminación si el UoW aún no ha sido congelado.

Después del freeze:

```text
reject mutation
```

---

# 218. Frozen persistence plan

Una vez congelado:

```text
PersistencePlan
```

no deberá modificarse arbitrariamente.

---

# 219. Remove after plan freeze

Debe ser rechazado o programado para un flush posterior según contexto.

---

# 220. Cancellation token

Internamente puede existir:

```php
final readonly class RemovalRegistration
{
    public function __construct(
        public RemovalRegistrationId $id,
        public EntityKey $entityKey,
        public RemovalRegistrationState $state,
    ) {}
}
```

---

# 221. RemovalRegistrationState

```php
enum RemovalRegistrationState
{
    case SCHEDULED;
    case PLANNED;
    case EXECUTING;
    case EXECUTED;
    case RECONCILED;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 222. State ≠ EntityState

Este estado describe la operación de eliminación.

No reemplaza:

```text
MANAGED
REMOVED
DETACHED
```

---

# 223. Observability

Telemetry propuesta:

```text
orm.persistence.delete.scheduled
orm.persistence.delete.cancelled
orm.persistence.delete.prepared
orm.persistence.delete.executed
orm.persistence.delete.succeeded
orm.persistence.delete.failed
orm.persistence.delete.unknown
orm.persistence.delete.reconciled
orm.persistence.delete.optimistic_conflict
orm.persistence.delete.missing_target
orm.persistence.delete.cascade_operations
orm.persistence.delete.orphan_operations
orm.persistence.delete.soft
orm.persistence.delete.physical
orm.persistence.delete.duration
orm.persistence.delete.reconciliation_duration
orm.persistence.delete.tainted_context
```

---

# 224. Delete strategy metric

Valores de cardinalidad baja:

```text
physical
soft
temporal
archival
```

son aceptables como labels.

---

# 225. Entity IDs

No incluir IDs de entidades en métricas por defecto.

---

# 226. Sensitive tenant IDs

Tampoco como labels de alta cardinalidad.

---

# 227. Diagnostics

Ejemplo:

```text
Delete Persistence
────────────────────────────────

Entity:
  App\Domain\User

Entity Key:
  User#42

State:
  REMOVED

Strategy:
  PHYSICAL

Identity Predicate:
  configured

Tenant Predicate:
  configured

Optimistic Lock:
  enabled
  expected version: 7

Cascade:
  ORM removals: 2
  DB cascades: 1

Orphan Removal:
  0

Execution:
  status: SUCCEEDED
  certainty: CERTAIN
  affected rows: 1

Reconciliation:
  IdentityMap: detached
  Snapshot: removed
  UnitOfWork: reconciled
  EntityState: DETACHED

Transaction:
  ACTIVE / NOT YET COMMITTED
```

---

# 228. Error hierarchy

```text
DatabaseOrmException
└── PersistenceException
    └── DeletePersistenceException
        ├── DeletePersistencePreparationException
        ├── DeletePersistenceValidationException
        ├── InvalidDeleteEntityStateException
        ├── MissingDeleteIdentityException
        ├── DeleteIdentityMismatchException
        ├── DeleteContextMismatchException
        ├── DeleteNamespaceMismatchException
        ├── DeleteShardMismatchException
        ├── UnsafeDeletePredicateException
        ├── DeletePredicateException
        ├── DeleteStrategyException
        ├── DeleteStrategyConflictException
        ├── DetachedEntityDeleteException
        ├── DeleteDependencyException
        ├── DeleteDependencyCycleException
        ├── CascadeDeleteException
        ├── CascadeDeleteCycleException
        ├── OrphanRemovalException
        ├── DeleteOptimisticLockException
        ├── DeleteCardinalityException
        ├── MissingDeleteTargetException
        ├── DeleteExecutionException
        ├── DeleteOutcomeUnknownException
        ├── DeleteReplaySafetyException
        ├── DeleteReconciliationException
        ├── DeleteIdentityMapReconciliationException
        ├── DeleteSnapshotReconciliationException
        ├── DeleteUnitOfWorkReconciliationException
        ├── DeleteLifecycleException
        ├── DeleteExtensionException
        ├── DeleteRuntimeIsolationException
        └── DeletePersistenceInvariantException
```

---

# 229. Estructura propuesta

```text
src/Quantum/Database/ORM/Persistence/Delete/
│
├── Contract/
│   ├── DeletePersistenceSystem.php
│   ├── DeletePersistencePreparer.php
│   ├── DeletePersistenceReconciler.php
│   ├── DeletePersistenceQueryTranslator.php
│   └── DeleteStrategyResolver.php
│
├── Operation/
│   ├── DeleteEntityOperation.php
│   ├── DeletePersistenceId.php
│   ├── DeletePersistenceIntent.php
│   ├── RemovalRegistration.php
│   ├── RemovalRegistrationId.php
│   └── RemovalRegistrationState.php
│
├── Preparation/
│   ├── DefaultDeletePersistenceSystem.php
│   ├── DeletePersistenceRequest.php
│   ├── PreparedDeletePersistence.php
│   ├── DeletePersistenceValidator.php
│   └── DeletePersistenceFingerprint.php
│
├── Strategy/
│   ├── DeletePersistenceStrategy.php
│   ├── DefaultDeleteStrategyResolver.php
│   ├── PhysicalDeleteStrategy.php
│   ├── SoftDeleteStrategyAdapter.php
│   ├── TemporalDeleteStrategyAdapter.php
│   └── ArchivalDeleteStrategyAdapter.php
│
├── Predicate/
│   ├── DeletePredicate.php
│   ├── DeletePredicateSet.php
│   ├── IdentityDeletePredicate.php
│   ├── VersionDeletePredicate.php
│   └── DeletePredicateResolver.php
│
├── Concurrency/
│   ├── DeleteConcurrencyRequirement.php
│   ├── DeleteCardinalityExpectation.php
│   ├── MissingDeleteTargetPolicy.php
│   └── DeleteReplaySafety.php
│
├── Relationship/
│   ├── CascadeDeleteResolver.php
│   ├── CascadeDeletePlan.php
│   ├── OrphanRemovalResolver.php
│   └── DeleteRelationshipDependency.php
│
├── Query/
│   ├── DefaultDeletePersistenceQueryTranslator.php
│   ├── DeletePersistenceTranslationContext.php
│   └── DeleteQueryMetadata.php
│
├── Execution/
│   ├── DeleteExecutionOutcome.php
│   ├── DeleteExecutionStatus.php
│   ├── DeleteExecutionRequirementSet.php
│   └── DeleteFailurePhase.php
│
├── Reconciliation/
│   ├── DefaultDeletePersistenceReconciler.php
│   ├── DeleteReconciliationResult.php
│   ├── DeleteReconciliationStatus.php
│   ├── DeleteIdentityMapReconciler.php
│   ├── DeleteSnapshotReconciler.php
│   ├── DeleteUnitOfWorkReconciler.php
│   └── DeletedEntityRollbackRecord.php
│
├── Extension/
│   ├── DeletePersistenceExtension.php
│   ├── DeletePersistenceExtensionRegistry.php
│   ├── DeletePersistenceExtensionContext.php
│   └── DeletePersistenceContribution.php
│
├── Telemetry/
│   ├── DeletePersistenceTelemetry.php
│   └── DeletePersistenceDiagnostics.php
│
└── Exception/
    └── ...
```

---

# 230. Testing Strategy

El sistema deberá probar al menos:

```text
managed deletion
new entity cancellation
detached entities
composite identities
optimistic locking
missing rows
cascade remove
DB cascade
orphan removal
soft delete integration
unknown outcomes
rollback interaction
IdentityMap reconciliation
snapshot reconciliation
bulk delete separation
tenant isolation
persistent runtime
extensions
```

---

# 231. Test — remove does not execute

```text
remove(entity)
```

no genera query inmediatamente.

---

# 232. Test — state transition

```text
MANAGED
→ remove()
→ REMOVED
```

---

# 233. Test — successful physical delete

```text
REMOVED
→ execute
→ reconcile
→ DETACHED
```

---

# 234. Test — NEW entity

```text
NEW
→ remove()
→ no INSERT
→ no DELETE
```

cuando sea seguro.

---

# 235. Test — assigned ID NEW entity

Tener ID no implica que exista row.

No debe ejecutar DELETE automáticamente.

---

# 236. Test — detached entity

`remove()` falla según política.

---

# 237. Test — duplicate remove

No duplica operación.

---

# 238. Test — identity preserved

Después de delete, el objeto puede conservar su identifier.

---

# 239. Test — IdentityMap cleanup

Forward y reverse indexes desaparecen.

---

# 240. Test — snapshot cleanup

Snapshot activo desaparece.

---

# 241. Test — UnitOfWork cleanup

Operación queda reconciliada.

---

# 242. Test — no premature detach

Antes del éxito:

```text
IdentityMap still contains entity
```

---

# 243. Test — optimistic success

```text
identity + expected version
→ affected rows 1
→ success
```

---

# 244. Test — optimistic conflict

```text
affected rows 0
→ conflict
```

---

# 245. Test — missing target

Aplica política explícita.

---

# 246. Test — unknown outcome

No detach ciego.

---

# 247. Test — timeout after send

Puede producir UNKNOWN.

---

# 248. Test — cancellation before send

No produce DB effect.

---

# 249. Test — retry classification

Triggers pueden convertir replay safety en UNKNOWN/UNSAFE.

---

# 250. Test — cascade ORM

Dependientes se programan una sola vez.

---

# 251. Test — cascade cycle

No produce recursión infinita.

---

# 252. Test — DB cascade

No genera deletes físicos redundantes cuando metadata/capabilities permiten evitarlo.

---

# 253. Test — managed DB-cascaded child

Se aplica política de reconciliación.

---

# 254. Test — orphan removal

Solo elimina orphan estabilizado.

---

# 255. Test — orphan reattached

No elimina si dejó de ser orphan antes del freeze.

---

# 256. Test — FK dependency

Planner ordena correctamente.

---

# 257. Test — soft delete

ORM lifecycle sigue siendo remove.

---

# 258. Test — soft delete physical operation

Puede usar Update Query Model.

---

# 259. Test — postRemove

Solo después de semantic success.

---

# 260. Test — postRemove ≠ commit

Garantizado.

---

# 261. Test — postRemove failure

No reetiqueta DB outcome como failed si la operación ya sucedió.

---

# 262. Test — rollback

No realiza reattach ingenuo.

---

# 263. Test — bulk delete

No dispara callbacks per-entity sin garantía explícita.

---

# 264. Test — empty predicate

Physical entity delete es rechazado.

---

# 265. Test — tenant isolation

Nunca elimina misma ID de tenant distinto.

---

# 266. Test — shard isolation

Nunca enruta a shard incompatible.

---

# 267. Test — extension conflict

Physical vs soft conflict falla determinísticamente.

---

# 268. Test — FrankenPHP

Delete state de Request A no aparece en Request B.

---

# 269. Test — OpenSwoole

Coroutine A no comparte UoW/IdentityMap con B.

---

# 270. Test — reset

No ejecuta deletes pendientes implícitamente.

---

# 271. Anti-patterns

## 271.1 `remove()` ejecuta SQL

Prohibido.

## 271.2 REMOVED significa DB row absent

Incorrecto.

## 271.3 postRemove significa committed

Incorrecto.

## 271.4 Limpiar ID después del delete

Incorrecto por defecto.

## 271.5 Eliminar detached entity como managed

Peligroso.

## 271.6 Hacer cascade cargando todo el grafo

Evitar.

## 271.7 ORM cascade = DB cascade

Incorrecto.

## 271.8 Orphan removal inmediato

Incorrecto.

## 271.9 Reintentar DELETE ciegamente

Prohibido.

## 271.10 Timeout = delete failed

Incorrecto.

## 271.11 `affectedRows = 0` = success universal

Incorrecto.

## 271.12 `affectedRows = 0` = optimistic conflict universal

También incorrecto.

## 271.13 Soft delete como simple UPDATE sin lifecycle remove

Incorrecto.

## 271.14 Detach antes de ejecutar

Prohibido.

## 271.15 Borrar snapshot antes del éxito

Prohibido.

## 271.16 Bulk delete = remove(entity) × N

Incorrecto.

## 271.17 Predicate vacío

Crítico.

## 271.18 Tenant global estático

Prohibido.

## 271.19 Rollback = reattach automático

Incorrecto.

## 271.20 Extensión ejecutando SQL directo

Prohibido.

---

# 272. Architectural Invariants

## DB-ORM-DELETE-001
`remove()` registrará intención y no ejecutará SQL inmediatamente.

## DB-ORM-DELETE-002
REMOVED será distinto de physical deletion.

## DB-ORM-DELETE-003
Delete Persistence será distinto de SQL Compiler.

## DB-ORM-DELETE-004
Delete Persistence será distinto de Execution Engine.

## DB-ORM-DELETE-005
Delete Persistence será distinto de Transaction Manager.

## DB-ORM-DELETE-006
Delete Persistence será distinto de Bulk Delete.

## DB-ORM-DELETE-007
Delete Persistence no utilizará PDO directamente.

## DB-ORM-DELETE-008
Delete Persistence no generará SQL strings.

## DB-ORM-DELETE-009
Delete Persistence no hará commit.

## DB-ORM-DELETE-010
Delete Persistence no hará rollback.

## DB-ORM-DELETE-011
Una entidad MANAGED podrá transicionar a REMOVED.

## DB-ORM-DELETE-012
REMOVED continuará perteneciendo al PersistenceContext hasta reconciliación.

## DB-ORM-DELETE-013
Delete success suficientemente cierto permitirá REMOVED → DETACHED.

## DB-ORM-DELETE-014
UNKNOWN no permitirá afirmar eliminación.

## DB-ORM-DELETE-015
UNKNOWN no permitirá afirmar permanencia.

## DB-ORM-DELETE-016
DeleteEntityOperation representará intención semántica.

## DB-ORM-DELETE-017
DeleteEntityOperation será distinto de execution.

## DB-ORM-DELETE-018
ORM remove será distinto de physical SQL DELETE.

## DB-ORM-DELETE-019
Soft delete podrá implementar ORM removal mediante physical UPDATE.

## DB-ORM-DELETE-020
Soft delete reutilizará infraestructura canónica cuando corresponda.

## DB-ORM-DELETE-021
Soft delete seguirá teniendo lifecycle de removal.

## DB-ORM-DELETE-022
Prepared delete no significará executed delete.

## DB-ORM-DELETE-023
Physical managed delete requerirá identidad establecida.

## DB-ORM-DELETE-024
NEW sin persistencia física podrá cancelar su INSERT sin DELETE.

## DB-ORM-DELETE-025
Assigned identifier no demostrará row existence.

## DB-ORM-DELETE-026
Database-generated fake IDs estarán prohibidos.

## DB-ORM-DELETE-027
INSERT+DELETE podrá colapsarse solo si es semánticamente seguro.

## DB-ORM-DELETE-028
Observable side effects impedirán optimizaciones inseguras.

## DB-ORM-DELETE-029
Detached entity removal no se tratará como managed removal por defecto.

## DB-ORM-DELETE-030
Direct delete-by-id será distinto de managed entity removal.

## DB-ORM-DELETE-031
Direct delete-by-id no prometerá entity lifecycle sin materialización.

## DB-ORM-DELETE-032
Delete no limpiará identifier automáticamente.

## DB-ORM-DELETE-033
Deleted object podrá conservar identidad histórica.

## DB-ORM-DELETE-034
Re-persist deleted entity tendrá política explícita.

## DB-ORM-DELETE-035
Entity delete utilizará predicates tipados.

## DB-ORM-DELETE-036
Identity predicate será obligatorio para physical managed delete.

## DB-ORM-DELETE-037
Composite identity será soportada.

## DB-ORM-DELETE-038
Tenant predicate podrá ser obligatorio.

## DB-ORM-DELETE-039
Tenant predicate obligatorio no será bypassable silenciosamente.

## DB-ORM-DELETE-040
Entity namespace deberá concordar con PersistenceContext.

## DB-ORM-DELETE-041
Shard routing deberá respetar EntityKey/context.

## DB-ORM-DELETE-042
Optimistic delete será first-class.

## DB-ORM-DELETE-043
Expected version podrá formar parte del delete predicate.

## DB-ORM-DELETE-044
Zero affected rows no tendrá interpretación universal.

## DB-ORM-DELETE-045
Missing target y version conflict no serán confundidos sin evidencia.

## DB-ORM-DELETE-046
Delete cardinality expectation será explícita.

## DB-ORM-DELETE-047
Managed delete podrá exigir EXACTLY_ONE.

## DB-ORM-DELETE-048
Already-missing semantics dependerán de policy.

## DB-ORM-DELETE-049
Ensure-absent será distinto de delete-managed-entity.

## DB-ORM-DELETE-050
Delete replay safety será explícita.

## DB-ORM-DELETE-051
Physical DELETE no será asumido universalmente idempotente.

## DB-ORM-DELETE-052
Triggers serán considerados para replay safety.

## DB-ORM-DELETE-053
Retry ownership permanecerá separado.

## DB-ORM-DELETE-054
UNKNOWN será first-class.

## DB-ORM-DELETE-055
UNKNOWN no eliminará IdentityMap entry ciegamente.

## DB-ORM-DELETE-056
UNKNOWN no eliminará snapshot ciegamente.

## DB-ORM-DELETE-057
UNKNOWN preservará evidencia en UoW.

## DB-ORM-DELETE-058
Uncertainty podrá taint EntityManager.

## DB-ORM-DELETE-059
Delete Query Model será distinto de SQL.

## DB-ORM-DELETE-060
ORM no generará vendor-specific DELETE SQL.

## DB-ORM-DELETE-061
Parameters serán tipados y bound downstream.

## DB-ORM-DELETE-062
Cascade remove será distinto de DB cascade.

## DB-ORM-DELETE-063
ORM cascade producirá persistence operations.

## DB-ORM-DELETE-064
DB cascade no producirá redundant deletes sin necesidad.

## DB-ORM-DELETE-065
DB cascade deberá considerar managed child reconciliation.

## DB-ORM-DELETE-066
DB cascade no requerirá materializar todos los hijos.

## DB-ORM-DELETE-067
Cascade graph deberá detectar ciclos.

## DB-ORM-DELETE-068
Cascade scheduling será idempotente.

## DB-ORM-DELETE-069
Orphan removal será distinto de cascade remove.

## DB-ORM-DELETE-070
Orphan removal no ejecutará DELETE inmediatamente.

## DB-ORM-DELETE-071
Orphan status se resolverá sobre UoW estabilizado.

## DB-ORM-DELETE-072
Reattached orphan no será eliminado indebidamente.

## DB-ORM-DELETE-073
Persistence Planner controlará delete ordering.

## DB-ORM-DELETE-074
No existirá universal INSERT→UPDATE→DELETE ordering.

## DB-ORM-DELETE-075
FK constraints participarán en dependency planning.

## DB-ORM-DELETE-076
Delete-before-insert podrá ser válido.

## DB-ORM-DELETE-077
Insert-before-delete podrá ser válido.

## DB-ORM-DELETE-078
preRemove ocurrirá en semantic removal preparation.

## DB-ORM-DELETE-079
preRemove será distinto de remove().

## DB-ORM-DELETE-080
preRemove mutation estará restringida.

## DB-ORM-DELETE-081
preRemove no usará false-return veto implícito.

## DB-ORM-DELETE-082
postRemove ocurrirá después de semantic delete success.

## DB-ORM-DELETE-083
postRemove será distinto de commit.

## DB-ORM-DELETE-084
Irreversible external effects requerirán durability-aware mechanism.

## DB-ORM-DELETE-085
Soft delete utilizará remove lifecycle.

## DB-ORM-DELETE-086
ORM event semantics prevalecerán sobre physical SQL verb.

## DB-ORM-DELETE-087
Successful delete requerirá suficiente outcome certainty.

## DB-ORM-DELETE-088
Affected-row semantics serán validadas.

## DB-ORM-DELETE-089
Reconciliation actualizará IdentityMap.

## DB-ORM-DELETE-090
Reconciliation actualizará SnapshotRegistry.

## DB-ORM-DELETE-091
Reconciliation actualizará UnitOfWork.

## DB-ORM-DELETE-092
Reconciliation actualizará EntityState.

## DB-ORM-DELETE-093
In-memory reconciliation deberá ser coherente.

## DB-ORM-DELETE-094
Partial reconciliation será error crítico.

## DB-ORM-DELETE-095
Partial reconciliation podrá taint manager.

## DB-ORM-DELETE-096
DB success no será negado por reconciliation failure.

## DB-ORM-DELETE-097
IdentityMap removal ocurrirá solo tras suficiente certeza.

## DB-ORM-DELETE-098
Forward identity index será eliminado.

## DB-ORM-DELETE-099
Reverse object index será eliminado.

## DB-ORM-DELETE-100
Active snapshot será eliminado tras reconciliation.

## DB-ORM-DELETE-101
Entity identifier podrá permanecer tras delete.

## DB-ORM-DELETE-102
Entity object podrá seguir existiendo como detached.

## DB-ORM-DELETE-103
Detached entity no hará lazy-load mediante antiguo manager.

## DB-ORM-DELETE-104
Detached proxy no se reconectará silenciosamente.

## DB-ORM-DELETE-105
DETACHED + active IdentityMap entry será estado inválido estable.

## DB-ORM-DELETE-106
Delete success será distinto de transaction commit.

## DB-ORM-DELETE-107
ORM podrá reconciliar antes del commit.

## DB-ORM-DELETE-108
Rollback podrá invalidar la suposición de DB absence.

## DB-ORM-DELETE-109
Rollback no hará naïve reattach.

## DB-ORM-DELETE-110
Rollback reconciliation pertenecerá a arquitectura transaccional coordinada.

## DB-ORM-DELETE-111
Rollback journal podrá conservar pre-delete evidence.

## DB-ORM-DELETE-112
Rollback journal no garantizará reattachment seguro.

## DB-ORM-DELETE-113
DB cascades podrán requerir context clear después de rollback.

## DB-ORM-DELETE-114
Soft-delete rollback también requerirá reconciliation.

## DB-ORM-DELETE-115
Delete operations tendrán WRITE intent.

## DB-ORM-DELETE-116
Delete no se ejecutará sobre read replica ordinaria.

## DB-ORM-DELETE-117
Delete Persistence no abrirá conexión directamente.

## DB-ORM-DELETE-118
Delete execution outcome será estructurado.

## DB-ORM-DELETE-119
Success será distinto de absence de driver exception.

## DB-ORM-DELETE-120
Semantic success podrá requerir cardinality/concurrency validation.

## DB-ORM-DELETE-121
Reconciliation status será explícito.

## DB-ORM-DELETE-122
PARTIALLY_RECONCILED no será ocultado.

## DB-ORM-DELETE-123
Bulk delete será distinto de remove(entity) × N.

## DB-ORM-DELETE-124
Bulk delete no prometerá per-entity callbacks sin materialización.

## DB-ORM-DELETE-125
Bulk delete podrá dejar managed state stale.

## DB-ORM-DELETE-126
Bulk managed-state policy será explícita.

## DB-ORM-DELETE-127
Entity physical delete nunca tendrá predicate vacío.

## DB-ORM-DELETE-128
Empty predicate será fail-fast.

## DB-ORM-DELETE-129
Delete safety guard será distinto de authorization.

## DB-ORM-DELETE-130
Delete values no serán interpolados en SQL.

## DB-ORM-DELETE-131
Trigger effects no serán asumidos reversibles.

## DB-ORM-DELETE-132
Temporal removal podrá generar no-DELETE physical operations.

## DB-ORM-DELETE-133
Archival removal podrá generar múltiples steps.

## DB-ORM-DELETE-134
Multi-step removal será planificado.

## DB-ORM-DELETE-135
Delete extensions serán deterministas.

## DB-ORM-DELETE-136
Extensions no eliminarán identity predicates.

## DB-ORM-DELETE-137
Extensions no bypassarán tenant predicates.

## DB-ORM-DELETE-138
Extensions no bypassarán optimistic predicates silenciosamente.

## DB-ORM-DELETE-139
Extensions no harán commit.

## DB-ORM-DELETE-140
Extensions no harán rollback.

## DB-ORM-DELETE-141
Extensions no ejecutarán arbitrary SQL.

## DB-ORM-DELETE-142
Extension contributions tendrán provenance.

## DB-ORM-DELETE-143
Delete strategy conflicts serán explícitos.

## DB-ORM-DELETE-144
No existirá last-wins silencioso para estrategias incompatibles.

## DB-ORM-DELETE-145
Compiled immutable metadata podrá compartirse entre workers.

## DB-ORM-DELETE-146
Pending delete state será scoped.

## DB-ORM-DELETE-147
EntityManager será scoped.

## DB-ORM-DELETE-148
UnitOfWork será scoped.

## DB-ORM-DELETE-149
IdentityMap será scoped.

## DB-ORM-DELETE-150
SnapshotRegistry será scoped.

## DB-ORM-DELETE-151
Tenant context será scoped.

## DB-ORM-DELETE-152
Transaction context será scoped.

## DB-ORM-DELETE-153
No existirán global pending deletes.

## DB-ORM-DELETE-154
Worker reset limpiará pending removals.

## DB-ORM-DELETE-155
Worker reset no ejecutará implicit flush.

## DB-ORM-DELETE-156
Request termination no ejecutará implicit delete.

## DB-ORM-DELETE-157
Coroutine scopes permanecerán aislados.

## DB-ORM-DELETE-158
Duplicate remove scheduling será idempotente.

## DB-ORM-DELETE-159
Duplicate remove no producirá duplicate physical DELETE.

## DB-ORM-DELETE-160
Una entidad solo será considerada eliminada del `PersistenceContext` cuando la operación semántica de eliminación haya alcanzado un resultado suficientemente cierto y `IdentityMap`, snapshot, `UnitOfWork` y EntityState hayan sido reconciliados coherentemente, sin interpretar dicho resultado como prueba de commit transaccional.

---

# 273. Fórmulas fundamentales

## 273.1 Removal scheduling

```text
remove(E)
=
RegisterRemovalIntent(E)
```

No:

```text
remove(E)
=
ExecuteDelete(E)
```

---

# 274. Managed removal

```text
State(E) = MANAGED
∧
remove(E)
⇒
State(E) = REMOVED
```

antes de ejecución.

---

# 275. NEW cancellation

```text
State(E) = NEW
∧
NoPersistentInsertOccurred(E)
∧
remove(E)
⇒
CancelPendingPersistence(E)
```

---

# 276. Physical delete candidate

```text
PhysicalDeleteCandidate(E)
=
State(E) = REMOVED
∧
IdentityEstablished(E)
∧
DeleteStrategy(E) = PHYSICAL
```

---

# 277. Entity delete predicate

```text
DeletePredicate(E)
=
IdentityPredicate(E)
+
MandatoryContextPredicates(E)
+
ConcurrencyPredicates(E)
```

---

# 278. Optimistic delete

```text
OptimisticDelete(E)
=
EntityIdentity(E)
∧
ExpectedVersion(E)
```

---

# 279. Successful semantic delete

```text
SuccessfulDelete(E)
=
ExecutionSucceeded
∧
OutcomeCertaintySufficient
∧
CardinalityExpectationSatisfied
∧
ConcurrencyPolicySatisfied
∧
RemovalSemanticsSatisfied
```

---

# 280. Reconciliation

```text
DeleteReconciliation(E)
=
IdentityMapDetached(E)
∧
SnapshotRemoved(E)
∧
UnitOfWorkReconciled(E)
∧
State(E) = DETACHED
```

---

# 281. Final transition

```text
State(E) = REMOVED
∧
SuccessfulDelete(E)
∧
DeleteReconciliation(E)
⇒
State(E) = DETACHED
```

---

# 282. Unknown outcome

```text
DeleteOutcome(E) = UNKNOWN
⇒
¬ AssertDatabasePresence(E)
∧
¬ AssertDatabaseAbsence(E)
```

---

# 283. Cascade

```text
CascadeRemoval(P)
=
Closure(
    P,
    RelationshipsWhereCascadeRemoveEnabled
)
```

con detección de ciclos.

---

# 284. Orphan removal

```text
OrphanDelete(C)
=
PreviouslyOwned(C)
∧
NoLongerOwned(C)
∧
OrphanRemovalEnabled(C)
∧
StillOrphanAtUoWStabilization(C)
```

---

# 285. Durability

```text
SuccessfulDelete
⇏
TransactionCommitted
```

y:

```text
DurableDelete
=
SuccessfulDelete
∧
RequiredTransactionOutcome = COMMITTED
```

---

# 286. Replay safety

```text
SafeDeleteReplay
=
DeleteReplaySemanticsKnown
∧
CardinalityPolicyCompatible
∧
OptimisticPolicyCompatible
∧
TriggerEffectsCompatible
∧
TransactionStateCompatible
∧
ExternalSideEffectsSafe
```

---

# 287. Soft delete

```text
SemanticRemove(E)
∧
Strategy(E) = SOFT
⇒
PhysicalOperation(E) may be UPDATE
```

---

# 288. Lifecycle

```text
SemanticRemove
→ preRemove
→ PhysicalPersistenceOperation
→ Reconciliation
→ postRemove
```

independientemente del verbo SQL utilizado.

---

# 289. Arquitectura maestra

```text
                        Entity
                          │
                          ▼
                 EntityManager::remove()
                          │
                          ▼
                     UnitOfWork
                          │
                          ▼
                       REMOVED
                          │
                          ▼
                UoW Stabilization
                          │
              ┌───────────┼────────────┐
              ▼           ▼            ▼
           Cascades     Orphans     Dependencies
              │           │            │
              └───────────┼────────────┘
                          ▼
                 Persistence Engine
                          │
                          ▼
                DeleteEntityOperation
                          │
                          ▼
                Persistence Planner
                          │
                          ▼
               DeletePersistenceStep
                          │
                          ▼
                      preRemove
                          │
                          ▼
                Strategy Resolution
                   ┌──────┼───────┐
                   ▼      ▼       ▼
                Physical Soft   Temporal
                   │      │       │
                   └──────┼───────┘
                          ▼
                 Predicate Resolution
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
         Identity       Tenant      Version
             │            │            │
             └────────────┼────────────┘
                          ▼
                Prepared Delete
                          │
                          ▼
                    Query Model
                          │
                    ┌─────┴─────┐
                    ▼           ▼
              DeleteQuery    UpdateQuery
                    │           │
                    └─────┬─────┘
                          ▼
                     Query Engine
                          │
                          ▼
                     SQL Compiler
                          │
                          ▼
                   Execution Engine
                          │
                          ▼
                       Database
                          │
                          ▼
                  Execution Outcome
                 ┌────────┼─────────┐
                 ▼        ▼         ▼
              SUCCESS   FAILED    UNKNOWN
                 │                  │
                 ▼                  ▼
           Cardinality           Preserve
           Validation           Uncertainty
                 │                  │
                 ▼                  ▼
          Delete Reconciliation    TAINT?
                 │
      ┌──────────┼──────────┐
      ▼          ▼          ▼
 IdentityMap  Snapshot   UnitOfWork
      │          │          │
      └──────────┼──────────┘
                 ▼
              DETACHED
                 │
                 ▼
             postRemove
                 │
                 ▼
       Transaction still separate
```

---

# 290. Master Formula

```text
Database Delete Persistence System
=
Removal Intent Registration
+
REMOVED Entity State
+
Delete Entity Operation
+
Removal Strategy Resolution
+
Physical Delete
+
Soft Delete Integration
+
Temporal Removal Integration
+
Archival Removal Integration
+
Entity Identity Predicates
+
Composite Identifier Support
+
Tenant/Shard Context Protection
+
Optimistic Delete Predicates
+
Delete Cardinality Expectations
+
Missing Target Semantics
+
Replay Safety
+
Cascade Remove
+
Database Cascade Awareness
+
Orphan Removal
+
Relationship Dependency Planning
+
Cycle Detection
+
Persistence Graph Ordering
+
preRemove Lifecycle
+
Delete Query Model Translation
+
Typed Parameter Binding
+
Execution Outcome Certainty
+
Affected Row Validation
+
UNKNOWN Outcome Preservation
+
IdentityMap Reconciliation
+
Snapshot Disposal
+
UnitOfWork Reconciliation
+
Entity State Reconciliation
+
DETACHED Transition
+
postRemove Lifecycle
+
Transaction Boundary Separation
+
Rollback Awareness
+
Bulk Delete Separation
+
Persistent Runtime Isolation
+
Extension Governance
+
Telemetry
+
Diagnostics
+
Failure Modeling
```

---

# 291. Master Rule

> **En VoltStack, `remove()` registra una intención semántica de eliminación dentro del `UnitOfWork`; no ejecuta inmediatamente una operación de base de datos. Una entidad `REMOVED` continúa formando parte del `PersistenceContext` hasta que el plan de persistencia determine sus dependencias, cascadas, relaciones, estrategia de eliminación y condiciones de concurrencia. Solo un resultado de ejecución suficientemente cierto permite reconciliar `IdentityMap`, snapshot, `UnitOfWork` y EntityState para producir la transición a `DETACHED`; incluso entonces, la eliminación no deberá interpretarse como durable hasta conocerse el verdadero resultado de la transacción.**

---

# 292. Estado del Bloque 11

Con este documento quedan definidos:

```text
123_DATABASE_IDENTITY_MAP_SYSTEM.md
124_DATABASE_UNIT_OF_WORK_ARCHITECTURE.md
125_DATABASE_CHANGE_TRACKING_SYSTEM.md
126_DATABASE_ENTITY_SNAPSHOT_SYSTEM.md
127_DATABASE_PERSISTENCE_ENGINE.md
128_DATABASE_PERSISTENCE_PLANNER_SYSTEM.md
129_DATABASE_INSERT_PERSISTENCE_SYSTEM.md
130_DATABASE_UPDATE_PERSISTENCE_SYSTEM.md
131_DATABASE_DELETE_PERSISTENCE_SYSTEM.md
```

El motor ya posee las tres operaciones fundamentales:

```text
                    UnitOfWork
                        │
                        ▼
                Persistence Engine
                        │
                        ▼
                Persistence Planner
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
        INSERT         UPDATE        DELETE
          │             │             │
          └─────────────┼─────────────┘
                        ▼
                  Query Models
                        │
                        ▼
                   Query Engine
                        │
                        ▼
                Execution Engine
                        │
                        ▼
                   Reconciliation
```

El siguiente paso es definir el mecanismo que convierte todo este trabajo acumulado en una **unidad coordinada de sincronización ORM**:

```text
flush()
```

---

# 293. Siguiente documento

```text
132_DATABASE_FLUSH_SYSTEM.md
```

Deberá formalizar:

```text
EntityManager::flush()
UnitOfWork synchronization boundary
flush lifecycle
flush phases
reentrancy protection
change detection
snapshot comparison
lifecycle stabilization
PersistencePlan construction
plan freezing
INSERT/UPDATE/DELETE coordination
dependency ordering
transaction participation
implicit vs explicit transaction policy
execution phases
partial execution
execution outcome aggregation
generated identities
generated values
optimistic conflicts
UNKNOWN outcomes
post-persistence lifecycle
snapshot reconciliation
IdentityMap reconciliation
UnitOfWork cleanup
entities mutated during flush
future dirty state
flush failure
tainted EntityManager
rollback coordination
nested flush rejection
flush cancellation
flush timeout
batch integration
persistent runtime isolation
telemetry
diagnostics
extensions
testing
```

Regla central propuesta:

> **`flush()` será la frontera explícita de sincronización entre el estado administrado del `PersistenceContext` y la base de datos: deberá estabilizar el `UnitOfWork`, construir y congelar un `PersistencePlan`, ejecutar sus operaciones en orden válido y reconciliar sus resultados sin confundir sincronización ORM con commit transaccional.**