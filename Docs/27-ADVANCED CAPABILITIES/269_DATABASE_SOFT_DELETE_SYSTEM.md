# 269_DATABASE_SOFT_DELETE_SYSTEM.md

# VoltStack Quantum Database
## Soft Delete System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 269 — Soft Delete System  
**Bloque:** 27 — Advanced Database Capabilities  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `268_DATABASE_HISTORY_AND_VERSIONING_SYSTEM.md`  
**Siguiente documento:** `270_DATABASE_DATA_RETENTION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura del **Soft Delete System** de VoltStack Database.

El sistema permitirá retirar lógicamente una entidad del conjunto de datos activo sin eliminar inmediatamente su representación física de la base de datos.

El objetivo no es simplemente implementar:

```sql
WHERE deleted_at IS NULL
```

sino establecer una semántica consistente para:

```text
Deletion State
Visibility
Query Filtering
ORM State
Relationships
Restore
Force Delete
Uniqueness
History
Retention
Archival
Cache
Authorization
Multitenancy
Sharding
Transactions
Telemetry
Persistent Runtimes
```

La regla central será:

> **Soft Delete es una transición explícita del ciclo de vida y visibilidad de una entidad; no es una eliminación física aplazada ni un filtro SQL agregado informalmente.**

---

# 2. Problema

Supongamos:

```text
Customer #42
```

con:

```text
deleted_at = NULL
```

La aplicación ejecuta:

```php
$customer->delete();
```

VoltStack puede cambiar el estado lógico a:

```text
deleted_at = 2026-09-19T19:00:00Z
```

La fila sigue existiendo físicamente.

Sin embargo, ahora aparecen preguntas arquitectónicas:

```text
¿Customer::find(42) debe encontrarlo?

¿Customer::query()->get() debe incluirlo?

¿Puede restaurarse?

¿Sus relaciones siguen existiendo?

¿Debe desaparecer de caches?

¿Puede reutilizarse su email único?

¿Debe seguir siendo accesible por administradores?

¿Debe producir una versión histórica?

¿Puede ser eliminado definitivamente?

¿Cuándo?

¿Quién puede hacerlo?

¿Qué ocurre si una transacción falla?

¿Qué ocurre en replicas?

¿Qué ocurre con foreign keys?

¿Qué ocurre en workers persistentes?
```

Estas preguntas pertenecen al Soft Delete System.

---

# 3. Distinciones fundamentales

VoltStack distinguirá estrictamente:

```text
Soft Delete
≠ Hard Delete
≠ Archive
≠ Retention
≠ History
≠ Temporal Expiration
≠ Deactivation
≠ Authorization
≠ Hidden Record
```

---

# 4. Soft Delete ≠ Hard Delete

Soft delete:

```text
ACTIVE
   ↓
SOFT_DELETED
```

La representación física permanece.

Hard delete:

```text
ACTIVE / SOFT_DELETED
          ↓
PHYSICALLY_REMOVED
```

La representación física desaparece del storage activo.

---

# 5. Soft Delete ≠ Archive

Una entidad soft-deleted puede permanecer:

```text
same table
same database
same shard
```

Archive puede moverla a:

```text
cold storage
archive database
archive table
object storage
external archive provider
```

---

# 6. Soft Delete ≠ Retention

Soft delete responde:

```text
¿Debe esta entidad considerarse eliminada lógicamente?
```

Retention responde:

```text
¿Cuánto tiempo debe conservarse?
```

---

# 7. Soft Delete ≠ History

History responde:

```text
¿Cómo evolucionó esta entidad?
```

Soft Delete responde:

```text
¿Está actualmente dentro del conjunto lógico activo?
```

---

# 8. Soft Delete ≠ Temporal Expiration

Una entidad puede dejar de ser válida por:

```text
valid_to
```

sin estar eliminada.

Igualmente puede estar soft-deleted y conservar información temporal.

---

# 9. Soft Delete ≠ Deactivation

Ejemplo:

```text
User.status = DISABLED
```

significa que el usuario continúa siendo una entidad activa del dominio pero tiene determinada capacidad deshabilitada.

No equivale a:

```text
User = SOFT_DELETED
```

---

# 10. Soft Delete ≠ Authorization

Que una entidad esté soft-deleted no sustituye controles como:

```text
canRead()
canRestore()
canForceDelete()
```

---

# 11. Modelo de estado

VoltStack definirá conceptualmente:

```text
ACTIVE
   │
   │ softDelete()
   ▼
SOFT_DELETED
   │
   ├──────── restore() ────────► ACTIVE
   │
   ├──────── archive() ────────► ARCHIVED
   │
   └──────── forceDelete() ────► PHYSICALLY_REMOVED
```

---

# 12. Deleted Entity State

El estado lógico podrá representarse mediante:

```php
enum DeletionState
{
    case ACTIVE;
    case SOFT_DELETED;
}
```

Estados operacionales adicionales podrán existir fuera de este enum:

```text
ARCHIVED
PURGED
UNKNOWN
```

según subsistemas posteriores.

---

# 13. Marker model

La estrategia más común utilizará:

```text
deleted_at
```

pero VoltStack no acoplará la arquitectura a ese nombre.

---

# 14. SoftDeleteMarker

Conceptualmente:

```php
interface SoftDeleteMarker
{
    public function isDeleted(array $persistentState): bool;

    public function deletionMutation(
        Instant $instant
    ): array;

    public function restoreMutation(): array;
}
```

---

# 15. Marker strategies

VoltStack podrá soportar:

```text
TIMESTAMP
BOOLEAN
STATUS
CUSTOM
```

---

# 16. Timestamp marker

Ejemplo:

```text
deleted_at = NULL
```

significa:

```text
ACTIVE
```

y:

```text
deleted_at = Instant
```

significa:

```text
SOFT_DELETED
```

---

# 17. Boolean marker

Ejemplo:

```text
is_deleted = false
is_deleted = true
```

---

# 18. Status marker

Ejemplo:

```text
lifecycle_state = ACTIVE
lifecycle_state = DELETED
```

---

# 19. Default strategy

VoltStack podrá adoptar como default:

```text
TIMESTAMP
```

porque conserva adicionalmente:

```text
Deletion Instant
```

sin requerir una segunda columna.

---

# 20. Column name

El nombre default podrá ser:

```text
deleted_at
```

pero será metadata configurable.

---

# 21. SoftDeleteMetadata

Conceptualmente:

```php
final readonly class SoftDeleteMetadata
{
    public function __construct(
        public FieldId $markerField,
        public SoftDeleteMarkerType $markerType,
        public SoftDeleteVisibilityPolicy $visibility,
        public SoftDeleteCascadePolicy $cascade,
    ) {}
}
```

---

# 22. Declaración mediante attribute

Ejemplo:

```php
#[SoftDelete]
final class Customer
{
}
```

o:

```php
#[SoftDelete(
    field: 'deletedAt'
)]
final class Customer
{
}
```

---

# 23. Field mapping

```php
#[Column(name: 'deleted_at', nullable: true)]
private ?Instant $deletedAt = null;
```

---

# 24. Metadata compilation

La declaración será procesada por:

```text
DATABASE_METADATA_COMPILATION_SYSTEM
```

produciendo metadata immutable.

---

# 25. Attribute ≠ behavior

El attribute:

```php
#[SoftDelete]
```

no ejecuta queries.

Sólo declara semántica.

---

# 26. Query visibility

Toda consulta sobre una entidad soft-deletable tendrá un:

```text
SoftDeleteVisibility
```

---

# 27. Visibility modes

```php
enum SoftDeleteVisibility
{
    case ACTIVE_ONLY;
    case INCLUDE_DELETED;
    case DELETED_ONLY;
}
```

---

# 28. Default visibility

El default será:

```text
ACTIVE_ONLY
```

---

# 29. ACTIVE_ONLY

Semánticamente:

```text
entity deletion state = ACTIVE
```

No simplemente:

```sql
deleted_at IS NULL
```

aunque el compiler pueda producir esa representación.

---

# 30. INCLUDE_DELETED

Incluye:

```text
ACTIVE
+
SOFT_DELETED
```

---

# 31. DELETED_ONLY

Incluye exclusivamente:

```text
SOFT_DELETED
```

---

# 32. API propuesta

```php
Customer::query()->get();
```

equivale conceptualmente a:

```text
ACTIVE_ONLY
```

---

# 33. Include deleted

```php
Customer::query()
    ->withDeleted()
    ->get();
```

---

# 34. Deleted only

```php
Customer::query()
    ->onlyDeleted()
    ->get();
```

---

# 35. Naming compatibility

VoltStack podrá ofrecer aliases ergonómicos como:

```php
withTrashed()
onlyTrashed()
```

si se desea compatibilidad mental con Laravel.

Sin embargo, la terminología arquitectónica será:

```text
deleted
soft-deleted
```

---

# 36. Query semantics

La llamada:

```php
Customer::query()
```

no agregará SQL directamente.

Generará un:

```text
EntityQuery
```

con:

```text
SoftDeleteVisibility::ACTIVE_ONLY
```

---

# 37. Correct pipeline

```text
Model / Repository API
        ↓
Entity Query
        ↓
Query Context
        ↓
Soft Delete Visibility
        ↓
Semantic Query Resolution
        ↓
Predicate AST
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

# 38. Incorrect pipeline

No:

```php
$sql .= ' WHERE deleted_at IS NULL';
```

desde el Model API.

---

# 39. Semantic predicate

Conceptualmente:

```text
EntityIsActive(Customer)
```

puede convertirse en:

```text
deleted_at IS NULL
```

según metadata.

---

# 40. Why semantic predicate

Permite soportar:

```text
timestamp
boolean
status
custom marker
```

sin contaminar Query Builder.

---

# 41. SoftDeletePredicateResolver

Responsabilidad:

```text
SoftDeleteMetadata
+
Visibility
↓
Predicate AST
```

---

# 42. Query Builder independence

El Query Builder genérico no necesita saber qué significa soft delete.

La integración ocurre en:

```text
Entity Query Layer
+
Semantic Query Layer
```

---

# 43. Raw Query Builder

Una query de bajo nivel:

```php
DB::table('customers')->get();
```

no necesariamente aplicará soft-delete automáticamente.

---

# 44. Important distinction

```text
ORM Entity Query
≠
Raw Table Query
```

---

# 45. No hidden domain semantics in raw SQL layer

El Query Builder de tablas no deberá inferir que:

```text
customers.deleted_at
```

es soft-delete marker sólo por el nombre.

---

# 46. Explicit raw opt-in

Una API avanzada podría permitir:

```php
DB::table('customers')
    ->applyEntityPolicy(Customer::class)
```

pero deberá ser explícita.

---

# 47. Find semantics

```php
Customer::find(42);
```

por default:

```text
ACTIVE_ONLY
```

---

# 48. Deleted entity find

Si #42 está soft-deleted:

```php
Customer::find(42);
```

retorna:

```text
null
```

o equivalente de not-found API.

---

# 49. Find including deleted

```php
Customer::query()
    ->withDeleted()
    ->find(42);
```

podrá recuperarlo.

---

# 50. Repository semantics

```php
$repository->find($id);
```

seguirá:

```text
ACTIVE_ONLY
```

por default.

---

# 51. Repository explicit history

```php
$repository
    ->query()
    ->withDeleted()
    ->find($id);
```

---

# 52. IdentityMap challenge

Supongamos:

```text
Customer #42
```

está cargado como ACTIVE.

Posteriormente se ejecuta:

```php
$customer->delete();
```

El mismo objeto no puede continuar siendo conceptualmente ACTIVE.

---

# 53. IdentityMap update

Después de soft delete confirmado dentro del contexto ORM:

```text
EntityState
```

deberá reflejar la transición.

---

# 54. Entity state extension

Podrá existir:

```text
MANAGED_ACTIVE
MANAGED_SOFT_DELETED
```

como dimensión de lifecycle.

Preferentemente:

```text
PersistenceState
+
DeletionState
```

se mantendrán separados.

---

# 55. Orthogonal state model

Ejemplo:

```text
PersistenceState = MANAGED
DeletionState    = SOFT_DELETED
```

---

# 56. Why orthogonal

Evita una explosión de estados como:

```text
NEW_ACTIVE
NEW_DELETED
MANAGED_ACTIVE
MANAGED_DELETED
DIRTY_ACTIVE
DIRTY_DELETED
...
```

---

# 57. Soft delete operation

```php
$entityManager->softDelete($customer);
```

conceptualmente:

```text
resolve metadata
↓
validate lifecycle
↓
create deletion mutation
↓
register ChangeSet
↓
Persistence Planner
↓
UPDATE Query Model
```

---

# 58. Model API

Podrá ofrecer:

```php
$customer->delete();
```

como convenience API.

---

# 59. Single persistence engine

```text
$customer->delete()
```

y:

```text
$entityManager->softDelete($customer)
```

convergerán en el mismo:

```text
SoftDeleteCoordinator
→ UnitOfWork
→ Persistence Engine
```

---

# 60. Delete ≠ immediate UPDATE

La llamada:

```php
$customer->delete();
```

no necesariamente ejecutará SQL inmediatamente.

Dependiendo del API podrá:

```text
schedule soft delete
```

y sincronizar durante:

```text
flush()
```

---

# 61. Model API ergonomics

Si VoltStack ofrece una API Laravel-like que históricamente espera operación inmediata, ésta podrá hacer internamente:

```text
schedule
+
flush
```

pero no:

```text
commit caller transaction
```

---

# 62. Flush ≠ commit

Regla permanente:

```text
softDelete()
≠ commit()
```

---

# 63. Transaction semantics

Soft delete será una mutation normal y participará en:

```text
Transaction Manager
UnitOfWork
Optimistic Locking
Persistence Planner
Events
Cache Invalidation
History
```

---

# 64. Rollback

Si:

```text
soft delete
↓
flush
↓
ROLLBACK
```

la base de datos continuará con el estado anterior.

---

# 65. Object graph after rollback

Como en el modelo general:

```text
Database Rollback
≠
Automatic Object Graph Rewind
```

---

# 66. ORM reconciliation

El EntityManager deberá:

```text
reconcile
refresh
detach
taint
```

según la política de rollback existente.

---

# 67. UNKNOWN commit

Si:

```text
soft delete sent
COMMIT sent
connection lost
```

entonces:

```text
DeletionOutcome = UNKNOWN
```

---

# 68. UNKNOWN ≠ deleted

---

# 69. UNKNOWN ≠ active

VoltStack no inventará certeza.

---

# 70. Optimistic locking

Soft delete deberá respetar:

```text
DATABASE_OPTIMISTIC_LOCKING_SYSTEM
```

---

# 71. Example

Estado:

```text
id = 42
lock_version = 7
deleted_at = NULL
```

Mutation:

```text
set deleted_at = T
set lock_version = 8
where id = 42
and lock_version = 7
```

conceptualmente.

---

# 72. Affected rows zero

Puede significar:

```text
concurrent modification
already deleted
row physically missing
```

El sistema deberá interpretar según metadata y plan.

---

# 73. Idempotency

Llamar soft delete sobre una entidad ya soft-deleted podrá seguir una política.

---

# 74. SoftDeleteRepeatedPolicy

```text
NO_OP
ERROR
REFRESH_TIMESTAMP
CUSTOM
```

---

# 75. Default repeated behavior

Recomendado:

```text
NO_OP
```

cuando el estado soft-deleted esté demostrado.

---

# 76. Refresh timestamp danger

Actualizar:

```text
deleted_at
```

cada vez que se llame `delete()` altera:

```text
retention clock
history
audit
```

Por ello no será default.

---

# 77. Restore

Una entidad soft-deleted podrá regresar a:

```text
ACTIVE
```

si la política lo permite.

---

# 78. API

```php
$customer->restore();
```

o:

```php
$entityManager->restore($customer);
```

---

# 79. Restore mutation

Timestamp strategy:

```text
deleted_at = NULL
```

---

# 80. Restore ≠ History Restore

Muy importante:

```text
SoftDeleteRestore
≠
HistoricalVersionRestore
```

---

# 81. Soft delete restore

Cambia:

```text
SOFT_DELETED → ACTIVE
```

---

# 82. History restore

Reconstruye un estado histórico anterior y crea una nueva versión.

---

# 83. Both may interact

Restaurar una entidad soft-deleted puede generar:

```text
History Version
```

si está habilitado el History System.

---

# 84. Restore validation

Antes de restaurar:

```text
entity exists
entity is soft-deleted
authorization permits restore
uniqueness remains valid
relationships remain valid
tenant context matches
shard ownership matches
```

---

# 85. Unique constraint problem

Supongamos:

```text
Customer #42
email = a@example.com
deleted_at = T1
```

Después se crea:

```text
Customer #84
email = a@example.com
deleted_at = NULL
```

Ahora:

```php
Customer::withDeleted()->find(42)->restore();
```

puede violar uniqueness.

---

# 86. Restore ≠ guaranteed success

Soft delete no garantiza que una entidad pueda restaurarse indefinidamente.

---

# 87. Unique semantics

VoltStack distinguirá:

```text
PHYSICAL_UNIQUE
ACTIVE_ONLY_UNIQUE
DOMAIN_UNIQUE
```

---

# 88. Physical unique

DB:

```text
UNIQUE(email)
```

impide reutilizar el email aunque la entidad esté soft-deleted.

---

# 89. Active-only unique

Semántica:

```text
email unique among ACTIVE rows
```

---

# 90. Platform challenge

No todos los DBMS representan partial/filtered unique indexes igual.

---

# 91. Capability-driven uniqueness

La solución deberá usar:

```text
DATABASE_PLATFORM_CAPABILITY_SYSTEM
```

y no:

```php
if ($driver === 'postgres') ...
```

---

# 92. Active uniqueness implementation

Podría usar según plataforma:

```text
partial unique index
generated column
composite strategy
application constraint
other safe strategy
```

---

# 93. Semantic equivalence first

No basta con que la plataforma acepte SQL.

Debe preservar:

```text
unique among logically active entities
```

---

# 94. Restore uniqueness check

El planner deberá detectar conflictos cuando sea posible.

Pero:

```text
pre-check
≠
atomic uniqueness guarantee
```

---

# 95. Database constraint remains authority

Cuando exista una restricción física apropiada, será la garantía final frente a carreras concurrentes.

---

# 96. Relationships

Soft Delete deberá integrarse con:

```text
DATABASE_RELATIONSHIP_ARCHITECTURE
```

---

# 97. Relationship visibility

Supongamos:

```text
Order → Customer
```

y Customer está soft-deleted.

¿Qué retorna?

```php
$order->customer;
```

---

# 98. Relationship policy

Podrá ser:

```text
ACTIVE_ONLY
INCLUDE_DELETED
INHERIT_QUERY
CUSTOM
```

---

# 99. Default relationship visibility

Por default:

```text
ACTIVE_ONLY
```

para relaciones ordinarias.

---

# 100. Important consequence

Una FK puede existir físicamente mientras:

```text
$order->customer
```

retorna:

```text
null / unavailable
```

por visibility policy.

---

# 101. Physical relationship ≠ visible relationship

---

# 102. Required relationship problem

Si metadata declara:

```text
Order.customer required
```

pero Customer está soft-deleted, la FK sigue existiendo.

Desde perspectiva física:

```text
valid
```

Desde perspectiva visible:

```text
target unavailable
```

---

# 103. Required-visible semantics

VoltStack deberá distinguir:

```text
physical required
logical required
visible required
```

cuando sea necesario.

---

# 104. Eager loading

```php
Order::query()
    ->with('customer')
    ->get();
```

deberá aplicar la visibility policy de Customer.

---

# 105. Lazy loading

También deberá aplicar la misma semántica.

---

# 106. No eager/lazy divergence

```text
EagerLoad(Customer)
```

y:

```text
LazyLoad(Customer)
```

no deberán devolver conjuntos semánticamente distintos sólo por estrategia de loading.

---

# 107. Explicit relation override

Puede permitirse:

```php
$order->customer()
    ->withDeleted()
    ->first();
```

si autorización y metadata lo permiten.

---

# 108. Cascade soft delete

Cuando se elimina lógicamente un parent:

```text
Customer
  ↓
Orders
```

¿deben soft-deletarse Orders?

No necesariamente.

---

# 109. Default cascade

Será:

```text
NONE
```

salvo declaración explícita.

---

# 110. SoftDeleteCascadePolicy

```php
enum SoftDeleteCascadePolicy
{
    case NONE;
    case OWNED;
    case EXPLICIT;
    case CUSTOM;
}
```

---

# 111. Why no automatic cascade

Porque:

```text
parent invisible
```

no significa necesariamente:

```text
child deleted
```

---

# 112. Cascade graph

Si se habilita cascade:

```text
Root
 ↓
Owned A
 ↓
Owned B
```

deberá utilizar un plan explícito.

---

# 113. SoftDeletePlan

```php
final readonly class SoftDeletePlan
{
    public function __construct(
        public EntityIdentity $root,
        public array $mutations,
        public array $relationshipActions,
        public array $historyActions,
        public array $cacheActions,
    ) {}
}
```

---

# 114. Cascade cycle detection

El planner deberá detectar ciclos.

---

# 115. Cascade ≠ recursive method calls

No:

```php
foreach ($children as $child) {
    $child->delete();
}
```

como arquitectura base.

---

# 116. Graph planner

Se utilizará:

```text
Relationship Metadata
+
Ownership
+
Cascade Policy
↓
Soft Delete Graph Plan
```

---

# 117. Restore cascade

Restaurar parent no implica automáticamente restaurar children.

---

# 118. Why restore cascade is harder

Un child pudo ser eliminado:

```text
before parent
because of parent
after parent
independently
```

---

# 119. Deletion provenance

Para restore cascade seguro puede requerirse:

```text
DeletionOperationId
```

---

# 120. Example

```text
DeleteOperation OP-123

Customer #42
Order #1
Order #2
```

Si todos fueron eliminados por OP-123, una restore policy puede reconocer el grupo.

---

# 121. Timestamp alone insufficient

Que varias entidades tengan timestamps cercanos no demuestra que pertenezcan a la misma operación lógica.

---

# 122. Cascade restore policy

Podrá ser:

```text
NONE
SAME_DELETE_OPERATION
EXPLICIT
CUSTOM
```

---

# 123. Many-to-many

Soft delete del target no necesariamente elimina membership rows.

---

# 124. Example

```text
User
Role
user_roles
```

Soft-delete Role puede mantener:

```text
user_roles
```

físicamente.

---

# 125. Membership visibility

La relación visible filtrará targets soft-deleted según policy.

---

# 126. Soft-deletable association entity

Si la asociación misma tiene dominio:

```text
Membership
```

puede implementar soft delete independientemente.

---

# 127. Polymorphic relations

El Soft Delete System deberá resolver metadata después de conocer:

```text
target EntityType
```

---

# 128. No generic deleted_at assumption

Targets polimórficos pueden usar distintas estrategias de marker.

---

# 129. Query joins

Un JOIN hacia una entidad soft-deletable deberá preservar la semántica de join.

---

# 130. LEFT JOIN danger

Agregar:

```sql
WHERE child.deleted_at IS NULL
```

después de un LEFT JOIN puede cambiar accidentalmente cardinalidad/semántica.

---

# 131. Correct semantic planning

El planner deberá decidir si el predicate pertenece a:

```text
JOIN condition
subquery
relationship loader
root predicate
```

según la operación.

---

# 132. Soft-delete filtering ≠ textual predicate injection

Esta es una razón fundamental para integrarlo al:

```text
Semantic Query Graph
```

---

# 133. Aggregations

```php
Customer::query()->count();
```

por default contará:

```text
ACTIVE_ONLY
```

---

# 134. Including deleted

```php
Customer::query()
    ->withDeleted()
    ->count();
```

contará ambos estados.

---

# 135. Grouping

Soft-delete filtering deberá aplicarse antes de la cardinalidad lógica correspondiente.

---

# 136. Pagination

```text
Pagination
```

operará sobre el conjunto visible.

---

# 137. Total count

Por default:

```text
total = active records
```

---

# 138. Cursor pagination

El cursor deberá quedar vinculado al:

```text
SoftDeleteVisibility
```

dentro del query fingerprint.

---

# 139. Why cursor binding matters

Un cursor generado bajo:

```text
ACTIVE_ONLY
```

no debe reutilizarse silenciosamente bajo:

```text
INCLUDE_DELETED
```

---

# 140. Chunk processing

Chunk traversal también deberá preservar la visibility policy.

---

# 141. Mutation during chunk

Si el callback soft-delete rows mientras procesa:

```text
ACTIVE_ONLY
```

la población se reduce.

Esto interactúa con:

```text
OFFSET
KEYSET
```

como se definió en `201_DATABASE_CHUNK_PROCESSING_SYSTEM.md`.

---

# 142. Offset risk

Ejemplo:

```text
fetch offset 0
soft-delete first 100
fetch offset 100
```

puede saltar registros.

---

# 143. Keyset preferred

Para procesamiento que soft-delete entidades durante traversal:

```text
KEYSET
```

será generalmente más seguro cuando exista ordering estable.

---

# 144. Lazy Collection

Lazy traversal heredará:

```text
SoftDeleteVisibility
```

del query inicial.

---

# 145. Visibility immutable during traversal

Cambiar globalmente la visibility a mitad de una Lazy Collection estará prohibido.

---

# 146. Bulk soft delete

Debe distinguirse:

```text
entity-aware soft delete
```

de:

```text
bulk soft delete
```

---

# 147. Entity-aware

```text
load entities
→ lifecycle
→ UoW
→ events
→ history
```

---

# 148. Bulk

```text
predicate
→ set deletion marker
```

sin necesariamente hidratar cada entidad.

---

# 149. Bulk semantic difference

Bulk soft delete puede no ejecutar:

```text
per-entity callbacks
per-entity lifecycle methods
full object graph logic
```

---

# 150. Explicit API

Ejemplo:

```php
Customer::query()
    ->where('inactive', true)
    ->softDeleteBulk();
```

deberá documentar su semántica.

---

# 151. No fake entity lifecycle

Bulk mutation no fingirá haber hidratado entidades.

---

# 152. Bulk history

Si History requiere versión por entidad, el planner deberá decidir:

```text
supported
special bulk strategy
fallback
reject
```

---

# 153. UNKNOWN ≠ supported

Si no puede preservar las garantías de History:

```text
bulk soft delete
```

deberá rechazarse o degradarse explícitamente según policy.

---

# 154. Hard delete

API conceptual:

```php
$customer->forceDelete();
```

---

# 155. Hard delete semantics

```text
Physical DELETE
```

sobre la representación persistente.

---

# 156. forceDelete authorization

Será distinta de:

```text
softDelete authorization
```

---

# 157. Force delete default

Deberá ser una operación de mayor privilegio.

---

# 158. Force delete active entity

Podrá configurarse:

```text
ALLOW
REQUIRE_SOFT_DELETED
FORBID
CUSTOM
```

---

# 159. Safer default

Para modelos soft-deletable críticos:

```text
REQUIRE_SOFT_DELETED
```

puede ser el default recomendado.

---

# 160. Hard delete relationships

Debe respetar:

```text
foreign keys
cascade policies
retention
legal hold
history
archive requirements
```

---

# 161. Database cascade

```text
ON DELETE CASCADE
```

es una operación física.

No equivale a:

```text
SoftDeleteCascadePolicy
```

---

# 162. Critical distinction

```text
DB ON DELETE CASCADE
≠
ORM Soft Delete Cascade
```

---

# 163. History integration

Si la entidad está versionada:

```text
ACTIVE
→ SOFT_DELETED
```

podrá producir nueva versión.

---

# 164. History version reason

Por ejemplo:

```text
SOFT_DELETE
```

---

# 165. Restore history reason

```text
SOFT_DELETE_RESTORE
```

---

# 166. Force delete history policy

Puede ser:

```text
RETAIN_HISTORY
PURGE_HISTORY
ARCHIVE_HISTORY
LEGAL_POLICY
CUSTOM
```

---

# 167. History survival

Por default, no se deberá asumir:

```text
hard delete entity
→ delete all history
```

---

# 168. History + retention

El siguiente sistema:

```text
270_DATABASE_DATA_RETENTION_SYSTEM.md
```

decidirá cuándo la información puede o debe eliminarse definitivamente.

---

# 169. Retention clock

Soft delete puede iniciar:

```text
RetentionPeriod
```

Ejemplo:

```text
soft deleted at T0
purge eligible after T0 + 90 days
```

---

# 170. But soft delete ≠ retention

El Soft Delete System sólo proporciona:

```text
DeletionInstant
```

y estado.

Retention determina la política temporal.

---

# 171. Legal hold

Si existe:

```text
LegalHold
```

force delete/purge puede estar prohibido aunque haya vencido retention.

---

# 172. Archival integration

Antes del purge puede requerirse:

```text
archive
```

según policy.

---

# 173. Cache integration

Soft delete cambia la visibilidad de la entidad.

Por tanto puede invalidar:

```text
Entity Cache
Result Cache
Query Cache
Relationship Cache
Pagination Cache
```

---

# 174. Entity cache

Una cache entry:

```text
Customer #42 ACTIVE
```

no puede continuar sirviéndose como activa después del commit del soft delete.

---

# 175. Cache publication

No invalidar como confirmado antes del transaction outcome adecuado si ello rompe consistencia.

---

# 176. AfterCommit invalidation

Preferido:

```text
soft delete
↓
commit
↓
semantic invalidation
```

---

# 177. Rollback

Rollback cancela invalidation asociada a la mutation cuando todavía no fue publicada.

---

# 178. UNKNOWN commit

Aplicar:

```text
conservative invalidation
```

cuando sea necesario.

---

# 179. Cache keys

Queries deberán incorporar semánticamente:

```text
SoftDeleteVisibility
```

---

# 180. Critical cache invariant

```text
ACTIVE_ONLY result cache
≠
INCLUDE_DELETED result cache
```

---

# 181. Query compilation cache

El compiled query puede variar por visibility shape.

---

# 182. Metadata cache

Puede cachear:

```text
SoftDeleteMetadata
```

como metadata immutable/generation-aware.

---

# 183. Security

Soft-deleted records suelen contener datos todavía sensibles.

Por ello:

```text
not visible in normal query
```

no significa:

```text
not sensitive
```

---

# 184. Permissions

Conceptualmente:

```text
entity.read
entity.read_deleted
entity.soft_delete
entity.restore
entity.force_delete
```

---

# 185. Authorization before visibility override

Una llamada:

```php
->withDeleted()
```

no debe otorgar autorización.

---

# 186. Query capability ≠ permission

```text
withDeleted()
```

expresa query semantics.

Authorization decide si el caller puede utilizarla.

---

# 187. Internal framework access

Algunos subsistemas internos podrán requerir acceso a deleted records.

Ejemplos:

```text
retention worker
archive worker
administration
integrity checker
```

pero deberán utilizar contextos privilegiados explícitos.

---

# 188. No global disable

Evitar APIs como:

```php
SoftDeletes::disableGlobally();
```

en workers persistentes.

---

# 189. Scoped override

Usar:

```text
SoftDeleteVisibilityContext
```

scope-local.

---

# 190. Security against bypass

Raw SQL seguirá siendo una escape hatch de bajo nivel.

Por tanto:

```text
ORM visibility policy
```

no deberá presentarse como security boundary absoluta frente a código con acceso raw DB.

---

# 191. Database permissions

Para aislamiento fuerte pueden requerirse:

```text
views
RLS
separate credentials
database permissions
```

como capas adicionales.

---

# 192. Query Audit

Operaciones sensibles podrán auditar:

```text
read deleted
restore
force delete
bulk delete
```

---

# 193. Sensitive Data Protection

Soft-deleted rows seguirán bajo:

```text
232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM
```

---

# 194. Multitenancy

Soft Delete nunca elimina:

```text
TenantContext
```

de la query.

---

# 195. Correct query semantics

Conceptualmente:

```text
tenant_id = currentTenant
AND
deletion_state = ACTIVE
```

---

# 196. Order of concerns

No deberá existir:

```text
withDeleted()
→ accidentally remove tenant filter
```

---

# 197. Visibility dimensions

Conceptualmente:

```text
QueryVisibility =
    TenantVisibility
  × DeletionVisibility
  × AuthorizationVisibility
  × TemporalVisibility
```

---

# 198. Orthogonal dimensions

Cada dimensión deberá mantenerse independiente.

---

# 199. Cross-tenant deleted query

Prohibida por default.

---

# 200. Tenant admin

Un tenant admin podrá quizá:

```text
include deleted within own tenant
```

sin acceso a otros tenants.

---

# 201. System admin

Cross-tenant administration será un contexto explícito y privilegiado.

---

# 202. Sharding

Soft delete deberá ejecutarse sobre el shard propietario.

---

# 203. Routing

```text
EntityIdentity
+
TenantContext
+
ShardKey
↓
Shard Routing
↓
Soft Delete Mutation
```

---

# 204. No broadcast delete

Una soft delete ordinaria no deberá:

```text
broadcast UPDATE to all shards
```

si el owner puede resolverse.

---

# 205. Unknown shard

Si no puede resolverse:

```text
UNKNOWN
```

no equivale a:

```text
ALL_SHARDS
```

---

# 206. Replica reads

Una replica puede seguir mostrando una entidad como ACTIVE después de que writer confirmó soft delete.

---

# 207. Read-your-writes

Después de soft delete:

```text
sticky writer
```

o minimum replication position podrá ser necesario.

---

# 208. Restore and replicas

Mismo problema inverso:

```text
replica may still show deleted
```

después de restore.

---

# 209. Search indexes

Si existe full-text/external search integration, soft delete deberá producir:

```text
remove from active search visibility
```

o equivalente.

---

# 210. Search index ≠ source of truth

Database sigue siendo autoridad.

---

# 211. External indexes

Sincronización preferentemente mediante:

```text
afterCommit event
outbox
```

---

# 212. Failure of external de-indexing

No convierte automáticamente DB rollback después de commit.

Se tratará como fallo de proyección/integración.

---

# 213. Event model

Eventos conceptuales:

```text
SoftDeletePlanned
SoftDeleteScheduled
SoftDeletePersisted
SoftDeleteCommitted
SoftDeleteRolledBack
SoftDeleteOutcomeUnknown

SoftDeleteRestorePlanned
SoftDeleteRestored
SoftDeleteRestoreCommitted

ForceDeletePlanned
ForceDeleteCommitted
```

---

# 214. Persisted ≠ committed

---

# 215. Committed event

Integraciones externas deberán preferir:

```text
SoftDeleteCommitted
```

cuando necesiten certeza transaccional.

---

# 216. Event listener failure

Un listener después de commit:

```text
cannot undo database commit
```

---

# 217. Outbox

Para efectos externos críticos:

```text
search de-index
external storage cleanup
external notifications
```

se recomienda Outbox cuando aplique.

---

# 218. Telemetry

Métricas posibles:

```text
database_soft_delete_total
database_soft_delete_restore_total
database_force_delete_total
database_soft_delete_conflict_total
database_soft_delete_unknown_outcome_total
database_deleted_query_total
database_soft_delete_cascade_total
```

---

# 219. Bounded dimensions

Permitido:

```text
operation
result
marker_type
cascade_policy
```

Evitar:

```text
entity_id
tenant_id
email
deleted_at
```

como labels.

---

# 220. Tracing

Spans:

```text
database.soft_delete.plan
database.soft_delete.persist
database.soft_delete.restore
database.soft_delete.force
```

---

# 221. Query profiler

Podrá mostrar:

```text
Soft Delete Visibility: ACTIVE_ONLY
Predicate injected: semantic
Marker: timestamp
```

---

# 222. Debug information

Ejemplo:

```text
Entity:
  Customer

Soft Delete:
  enabled

Marker:
  deletedAt

Physical Column:
  deleted_at

Marker Type:
  TIMESTAMP

Visibility:
  ACTIVE_ONLY

Tenant:
  scoped

Authorization:
  applied

Predicate:
  semantic soft-delete predicate

Cache:
  visibility-aware

History:
  enabled
```

---

# 223. Developer Debug Toolbar

Podrá mostrar:

```text
Soft Delete Filters: 12
withDeleted Queries: 2
onlyDeleted Queries: 1
Soft Deletes: 3
Restores: 1
Force Deletes: 0
```

sin exponer información sensible.

---

# 224. Performance

El predicate:

```text
deleted_at IS NULL
```

puede afectar:

```text
index selection
cardinality estimates
unique constraints
join planning
```

---

# 225. Index design

Dependiendo del workload podrá ser útil:

```text
(deleted_at)
(tenant_id, deleted_at)
(deleted_at, created_at)
```

pero VoltStack no impondrá índices universales.

---

# 226. Metadata recommendations

Schema diagnostics podrá sugerir índices basándose en:

```text
query patterns
tenant strategy
ordering
platform capability
```

---

# 227. Soft delete density

Si 95% de una tabla está soft-deleted:

```text
active queries
```

pueden degradarse significativamente.

---

# 228. Retention and archive importance

Por ello soft delete no debe convertirse en:

```text
permanent garbage accumulation strategy
```

---

# 229. Resource governance

Bulk operations deberán respetar:

```text
max affected rows
timeout
transaction budget
memory budget
lock budget
```

---

# 230. Massive soft delete

No debería realizar:

```text
single UPDATE millions of rows
```

sin policy explícita.

---

# 231. Large dataset strategy

Podrá utilizar:

```text
Chunk Processing
Bulk Update
Large Dataset Processing
```

según garantías requeridas.

---

# 232. Concurrency

Dos procesos pueden intentar:

```text
soft delete
restore
```

simultáneamente.

---

# 233. Example race

```text
T1: restore Customer #42
T2: forceDelete Customer #42
```

---

# 234. Concurrency control

Podrá usar:

```text
Optimistic Locking
Pessimistic Locking
Transaction Isolation
```

según operación.

---

# 235. No last-write-wins assumption

Operaciones lifecycle sensibles no deberán aceptar siempre:

```text
last write wins
```

sin política.

---

# 236. Restore conflict

Debe existir error explícito:

```text
SoftDeleteRestoreConflictException
```

cuando corresponda.

---

# 237. Force delete conflict

Igualmente:

```text
ForceDeleteConflictException
```

---

# 238. Failure model

Estados operacionales:

```text
PLANNED
SCHEDULED
EXECUTED
COMMITTED
ROLLED_BACK
FAILED
UNKNOWN
```

---

# 239. EXECUTED ≠ COMMITTED

---

# 240. FAILED ≠ ROLLED_BACK

---

# 241. Retry

Soft delete puede ser retryable si:

```text
transaction retry policy
operation idempotency
outcome certainty
```

lo permiten.

---

# 242. UNKNOWN commit retry

No repetir ciegamente:

```text
softDelete()
```

si la primera transaction pudo confirmar y la semántica secundaria no es idempotente.

---

# 243. Marker mutation may be idempotent

```text
SET deleted_at = fixed T
```

puede ser físicamente idempotente.

Pero:

```text
history events
audit
external effects
cascade
```

pueden no serlo.

---

# 244. Operation identity

Una:

```text
DeletionOperationId
```

podrá correlacionar:

```text
root mutation
cascade mutations
history
audit
outbox
telemetry
```

sin usarse como telemetry label de alta cardinalidad.

---

# 245. Persistent runtimes

FrankenPHP será el runtime predeterminado.

Soft Delete deberá ser seguro en workers persistentes.

---

# 246. Forbidden state

No:

```php
static bool $includeDeleted = false;
```

---

# 247. Why dangerous

Request A:

```text
withDeleted
```

podría contaminar Request B.

---

# 248. Scope-local visibility

Utilizar:

```text
QueryContext
```

o:

```text
SoftDeleteVisibilityContext
```

por operación.

---

# 249. Request reset

Al finalizar request:

```text
visibility override
deletion operation context
temporary lifecycle state
restore context
```

deberá liberarse.

---

# 250. FrankenPHP

El mismo worker podrá procesar:

```text
Request A
Request B
Request C
```

sin heredar `withDeleted()`.

---

# 251. RoadRunner

Misma regla.

---

# 252. OpenSwoole

Además:

```text
Coroutine A
```

no compartirá visibility mutable con:

```text
Coroutine B
```

---

# 253. Serialization

Una entidad soft-deleted podrá serializarse sólo si:

```text
authorization
visibility policy
serializer policy
```

lo permiten.

---

# 254. No accidental API exposure

Un admin endpoint que carga:

```text
withDeleted()
```

no deberá provocar que el serializer global exponga deleted entities en otro request.

---

# 255. Queue jobs

Si un job recibe:

```text
CustomerId(42)
```

y al ejecutarse la entidad ya está soft-deleted:

```php
Customer::find(42)
```

puede devolver null.

---

# 256. Job policy

El job deberá declarar si necesita:

```text
ACTIVE_ONLY
INCLUDE_DELETED
```

---

# 257. Do not serialize live entity assumptions

Preferible:

```text
EntityIdentity
+
required visibility intent
```

sobre depender de un objeto ORM serializado.

---

# 258. CLI

Herramientas administrativas podrán ofrecer:

```text
db:soft-delete
db:restore-deleted
db:list-deleted
db:purge-deleted
```

pero:

```text
purge
```

deberá integrarse con Retention y Authorization.

---

# 259. Migration behavior

Agregar soft delete a una tabla existente requiere definir:

```text
marker column
default active semantics
indexes
unique constraints
existing rows
rollback strategy
```

---

# 260. Example migration

Conceptualmente:

```php
$table->softDeletes();
```

puede generar una:

```text
SoftDeleteSchemaDefinition
```

no SQL directamente.

---

# 261. Schema Builder

Podrá ofrecer:

```php
$table->softDeletes();
$table->softDeletes('removed_at');
```

---

# 262. Schema definition

La definición deberá mapearse a:

```text
nullable temporal field
```

o estrategia configurada.

---

# 263. Migration safety

Cambiar uniqueness para introducir active-only uniqueness puede ser:

```text
LOCKING
REBUILDING
PLATFORM_SPECIFIC
```

y deberá pasar por Migration Safety System.

---

# 264. Zero-downtime introduction

Para tablas grandes puede requerirse:

```text
expand
backfill
deploy compatible application
build index
switch semantics
contract
```

---

# 265. Removing soft delete

Eliminar el feature de una entidad exige decidir qué hacer con:

```text
currently deleted rows
```

---

# 266. Migration choices

```text
restore all
hard delete all
archive all
abort
custom
```

---

# 267. No implicit decision

VoltStack no deberá elegir automáticamente una opción destructiva.

---

# 268. Extension model

Soft Delete deberá ser extensible mediante:

```text
SoftDeleteMarker
SoftDeletePolicy
SoftDeleteCascadePolicy
SoftDeleteVisibilityPolicy
SoftDeletePlannerExtension
```

---

# 269. Custom marker

Un dominio puede representar:

```text
state = RETIRED
```

como deleted semantics.

---

# 270. Custom marker safety

Debe declarar:

```text
active predicate
deleted predicate
delete mutation
restore mutation
type
```

---

# 271. Custom SQL forbidden

El extension point preferirá:

```text
Predicate AST
Mutation Model
```

sobre strings SQL.

---

# 272. Platform independence

Soft Delete System no tendrá:

```php
if ($platform === 'mysql') { ... }
```

en su dominio principal.

---

# 273. Platform responsibilities

Platform/Dialect/Compiler resolverán:

```text
physical indexes
filtered uniqueness
generated columns
temporal declaration
SQL syntax
```

---

# 274. MySQL/MariaDB distinction

VoltStack continuará tratando:

```text
MySQL
MariaDB
```

como plataformas distintas.

---

# 275. SQLite

La semántica deberá adaptarse a sus capabilities sin fingir soporte inexistente.

---

# 276. PostgreSQL

Podrá aprovechar capacidades nativas cuando preserven la semántica.

---

# 277. No vendor leakage

La API pública seguirá siendo:

```php
->withDeleted()
->onlyDeleted()
->restore()
->forceDelete()
```

independientemente de plataforma.

---

# 278. Proposed components

```text
SoftDeleteManager
SoftDeletePlanner
SoftDeleteMetadata
SoftDeleteMarker
SoftDeletePredicateResolver
SoftDeleteVisibility
SoftDeleteContext
SoftDeleteCascadePlanner
SoftDeleteRestorePlanner
ForceDeletePlanner
DeletionOperationId
```

---

# 279. SoftDeleteManager

Coordina la API de alto nivel.

No será:

```text
SQL executor
transaction manager
authorization engine
history engine
```

---

# 280. SoftDeletePlanner

Recibe:

```text
Entity
Metadata
Current State
Operation
Context
```

y produce:

```text
SoftDeletePlan
```

---

# 281. RestorePlanner

Valida:

```text
deletion state
uniqueness
authorization context
relationships
history policy
retention restrictions
```

y produce:

```text
SoftDeleteRestorePlan
```

---

# 282. ForceDeletePlanner

Valida:

```text
force-delete permission
retention
legal hold
history policy
FK constraints
cascade
archive requirements
```

---

# 283. Query integration architecture

```text
Application Query
       │
       ▼
Entity Query
       │
       ▼
Query Context
       │
       ├── Tenant Context
       ├── Authorization Context
       ├── Temporal Context
       └── Soft Delete Visibility
                │
                ▼
        Semantic Analyzer
                │
                ▼
      SoftDeletePredicateResolver
                │
                ▼
           Predicate AST
                │
                ▼
             Planner
                │
                ▼
            Compiler
                │
                ▼
             Database
```

---

# 284. Mutation architecture

```text
$entity->delete()
       │
       ▼
Model Context Resolver
       │
       ▼
SoftDeleteManager
       │
       ▼
SoftDeletePlanner
       │
       ├── Metadata
       ├── Relationships
       ├── Security
       ├── History
       └── Cache Semantics
       │
       ▼
UnitOfWork / ChangeSet
       │
       ▼
Persistence Planner
       │
       ▼
Query Model
       │
       ▼
Transaction
       │
       ▼
Database
```

---

# 285. Restore architecture

```text
restore(entity)
      │
      ▼
SoftDeleteRestorePlanner
      │
      ├── Is deleted?
      ├── Authorized?
      ├── Unique constraints valid?
      ├── Relationships valid?
      ├── Retention allows?
      └── History policy
      │
      ▼
Restore ChangeSet
      │
      ▼
Persistence Planner
      │
      ▼
Transaction
      │
      ▼
ACTIVE
```

---

# 286. Force delete architecture

```text
forceDelete(entity)
       │
       ▼
ForceDeletePlanner
       │
       ├── Authorization
       ├── Legal Hold
       ├── Retention
       ├── Archive Policy
       ├── History Policy
       ├── Relationships
       └── FK Constraints
       │
       ▼
Physical Delete Plan
       │
       ▼
Persistence Engine
       │
       ▼
Transaction
       │
       ▼
PHYSICALLY_REMOVED
```

---

# 287. Directory proposal

```text
src/Quantum/Database/SoftDelete/
│
├── Contract/
│   ├── SoftDeleteManager.php
│   ├── SoftDeletePlanner.php
│   ├── SoftDeleteMarker.php
│   ├── SoftDeletePredicateResolver.php
│   ├── SoftDeleteRestorePlanner.php
│   └── ForceDeletePlanner.php
│
├── Metadata/
│   ├── SoftDelete.php
│   ├── SoftDeleteMetadata.php
│   ├── CompiledSoftDeleteMetadata.php
│   ├── SoftDeleteMarkerType.php
│   └── SoftDeleteMetadataCompiler.php
│
├── State/
│   ├── DeletionState.php
│   ├── DeletionInstant.php
│   ├── DeletionOperationId.php
│   └── SoftDeleteOutcome.php
│
├── Marker/
│   ├── TimestampSoftDeleteMarker.php
│   ├── BooleanSoftDeleteMarker.php
│   ├── StatusSoftDeleteMarker.php
│   └── CustomSoftDeleteMarker.php
│
├── Query/
│   ├── SoftDeleteVisibility.php
│   ├── SoftDeleteVisibilityContext.php
│   ├── DefaultSoftDeletePredicateResolver.php
│   └── SoftDeleteSemanticPredicate.php
│
├── Planning/
│   ├── DefaultSoftDeletePlanner.php
│   ├── SoftDeletePlan.php
│   ├── DefaultSoftDeleteRestorePlanner.php
│   ├── SoftDeleteRestorePlan.php
│   ├── DefaultForceDeletePlanner.php
│   └── ForceDeletePlan.php
│
├── Cascade/
│   ├── SoftDeleteCascadePolicy.php
│   ├── SoftDeleteCascadePlanner.php
│   ├── RestoreCascadePolicy.php
│   └── SoftDeleteGraphPlan.php
│
├── Constraint/
│   ├── SoftDeleteUniquePolicy.php
│   ├── ActiveUniqueConstraint.php
│   └── SoftDeleteConstraintValidator.php
│
├── Integration/
│   ├── SoftDeleteHistoryIntegration.php
│   ├── SoftDeleteCacheIntegration.php
│   ├── SoftDeleteEventIntegration.php
│   └── SoftDeleteTelemetryIntegration.php
│
├── Runtime/
│   ├── SoftDeleteContext.php
│   └── SoftDeleteContextResolver.php
│
├── Diagnostics/
│   ├── SoftDeleteDiagnostics.php
│   └── SoftDeleteExplain.php
│
└── Exception/
    ├── SoftDeleteException.php
    ├── EntityAlreadyDeletedException.php
    ├── EntityNotDeletedException.php
    ├── SoftDeleteRestoreConflictException.php
    ├── SoftDeleteUniqueConstraintException.php
    ├── SoftDeleteCascadeException.php
    ├── ForceDeleteException.php
    ├── ForceDeleteConflictException.php
    ├── SoftDeleteSecurityException.php
    └── SoftDeleteOutcomeUnknownException.php
```

---

# 288. Exception hierarchy

```text
DatabaseException
└── SoftDeleteException
    ├── EntityAlreadyDeletedException
    ├── EntityNotDeletedException
    ├── SoftDeleteRestoreConflictException
    ├── SoftDeleteUniqueConstraintException
    ├── SoftDeleteCascadeException
    ├── SoftDeleteVisibilityException
    ├── ForceDeleteException
    ├── ForceDeleteConflictException
    ├── SoftDeleteSecurityException
    └── SoftDeleteOutcomeUnknownException
```

---

# 289. Diagnostics example

```text
VoltStack Soft Delete Diagnostics
────────────────────────────────────

Entity:
  Customer

Soft Delete:
  ENABLED

Marker:
  deletedAt

Column:
  deleted_at

Marker Type:
  TIMESTAMP

Current State:
  SOFT_DELETED

Deleted At:
  [available internally]

Visibility:
  DELETED_ONLY

Tenant:
  scoped

Shard:
  resolved

History:
  enabled

Retention:
  policy attached

Legal Hold:
  none

Restore:
  eligible

Force Delete:
  requires retention evaluation
```

---

# 290. Explain query example

```text
Entity Query
────────────────────────────────────

Entity:
  Customer

Base Predicate:
  country = ?

Tenant Scope:
  tenant_id = ?

Soft Delete Visibility:
  ACTIVE_ONLY

Soft Delete Semantic Predicate:
  EntityIsActive(Customer)

Physical Predicate:
  resolved by compiler

Authorization:
  applied

Ordering:
  created_at DESC

Cache Identity:
  includes deletion visibility
```

---

# 291. Explain restore example

```text
Soft Delete Restore Plan
────────────────────────────────────

Entity:
  Customer

Current Deletion State:
  SOFT_DELETED

Restore:
  requested

Authorization:
  ✓

Tenant:
  ✓

Shard:
  ✓

Unique Constraints:
  ✓

Relationships:
  ✓

Legal Hold:
  not applicable

History:
  create restore version

Cache:
  invalidate after commit

Result:
  RESTORABLE
```

---

# 292. Testing strategy

## Metadata tests

Probar:

```text
default marker
custom marker
timestamp marker
boolean marker
status marker
invalid metadata
```

---

# 293. Query tests

Probar:

```text
ACTIVE_ONLY
INCLUDE_DELETED
DELETED_ONLY
find()
count()
exists()
aggregates
joins
subqueries
CTEs
pagination
cursor pagination
```

---

# 294. Mutation tests

```text
soft delete
repeated delete
restore
repeated restore
force delete
```

---

# 295. Transaction tests

```text
commit
rollback
deadlock
retry
unknown commit
savepoint
nested transaction
```

---

# 296. Concurrency tests

```text
delete vs delete
delete vs update
delete vs restore
restore vs force delete
restore vs unique insert
```

---

# 297. Relationship tests

```text
one-to-one
one-to-many
many-to-one
many-to-many
polymorphic
eager loading
lazy loading
batch loading
cascade
restore cascade
```

---

# 298. Unique tests

```text
physical unique
active-only unique
restore conflict
concurrent restore
platform-specific capability
```

---

# 299. History tests

```text
delete creates version
restore creates version
force delete retains history
rollback does not publish history
unknown commit remains unknown
```

---

# 300. Cache tests

```text
active result cached
soft delete committed
old active result invalidated

rollback
cache remains consistent

unknown commit
conservative invalidation
```

---

# 301. Multitenancy tests

Demostrar:

```text
Tenant A withDeleted()
```

no puede ver:

```text
Tenant B deleted entities
```

---

# 302. Persistent runtime tests

Sobre el mismo worker:

```text
Request A → withDeleted()
Request B → normal query
```

Request B deberá volver a:

```text
ACTIVE_ONLY
```

---

# 303. OpenSwoole concurrency test

```text
Coroutine A → INCLUDE_DELETED
Coroutine B → ACTIVE_ONLY
```

sin contaminación.

---

# 304. Property test

Para una entidad inicialmente activa:

```text
restore(softDelete(E))
≈
E
```

para los campos no modificados por lifecycle/history metadata.

---

# 305. Visibility property

```text
E ∈ Query(ACTIVE_ONLY)
⇔
DeletionState(E) = ACTIVE
```

bajo un snapshot lógico consistente.

---

# 306. Deleted-only property

```text
E ∈ Query(DELETED_ONLY)
⇔
DeletionState(E) = SOFT_DELETED
```

---

# 307. Include property

```text
Query(INCLUDE_DELETED)
=
Query(ACTIVE_ONLY)
∪
Query(DELETED_ONLY)
```

cuando no existen otros filtros diferenciadores.

---

# 308. Invariantes arquitectónicos

## DB-SOFTDELETE-001

Soft Delete será distinto de Hard Delete.

## DB-SOFTDELETE-002

Soft Delete será distinto de Archive.

## DB-SOFTDELETE-003

Soft Delete será distinto de Retention.

## DB-SOFTDELETE-004

Soft Delete será distinto de History.

## DB-SOFTDELETE-005

Soft Delete será distinto de Temporal Expiration.

## DB-SOFTDELETE-006

Soft Delete será distinto de Deactivation.

## DB-SOFTDELETE-007

Soft Delete no sustituirá Authorization.

## DB-SOFTDELETE-008

El estado default será ACTIVE.

## DB-SOFTDELETE-009

La visibilidad default será ACTIVE_ONLY.

## DB-SOFTDELETE-010

El marker será metadata-driven.

## DB-SOFTDELETE-011

`deleted_at` será convención, no dependencia arquitectónica.

## DB-SOFTDELETE-012

Timestamp será una estrategia, no la única estrategia.

## DB-SOFTDELETE-013

Boolean marker será soportable.

## DB-SOFTDELETE-014

Status marker será soportable.

## DB-SOFTDELETE-015

Custom marker será extensible.

## DB-SOFTDELETE-016

Soft Delete Metadata será compilable.

## DB-SOFTDELETE-017

Compiled metadata será immutable.

## DB-SOFTDELETE-018

Attribute no ejecutará queries.

## DB-SOFTDELETE-019

Visibility será parte del Query Context.

## DB-SOFTDELETE-020

Soft-delete filtering será semántico.

## DB-SOFTDELETE-021

Model API no concatenará SQL.

## DB-SOFTDELETE-022

Repository API no concatenará SQL.

## DB-SOFTDELETE-023

SoftDeletePredicateResolver producirá AST.

## DB-SOFTDELETE-024

Compiler será responsable de SQL físico.

## DB-SOFTDELETE-025

Raw Table Query no inferirá soft-delete por nombre de columna.

## DB-SOFTDELETE-026

ORM Entity Query aplicará visibility por default.

## DB-SOFTDELETE-027

find() respetará ACTIVE_ONLY.

## DB-SOFTDELETE-028

withDeleted() será explícito.

## DB-SOFTDELETE-029

onlyDeleted() será explícito.

## DB-SOFTDELETE-030

withDeleted() no otorgará autorización.

## DB-SOFTDELETE-031

DeletionState será ortogonal a PersistenceState.

## DB-SOFTDELETE-032

Soft delete no requerirá explosión combinatoria de EntityState.

## DB-SOFTDELETE-033

Model API y EntityManager usarán mismo engine.

## DB-SOFTDELETE-034

Soft delete no implicará immediate SQL necesariamente.

## DB-SOFTDELETE-035

Soft delete no implicará commit.

## DB-SOFTDELETE-036

Flush será distinto de commit.

## DB-SOFTDELETE-037

Rollback no implicará automatic object graph rewind.

## DB-SOFTDELETE-038

UNKNOWN transaction outcome permanecerá UNKNOWN.

## DB-SOFTDELETE-039

UNKNOWN no será tratado como ACTIVE.

## DB-SOFTDELETE-040

UNKNOWN no será tratado como DELETED.

## DB-SOFTDELETE-041

Soft delete respetará optimistic locking.

## DB-SOFTDELETE-042

Repeated delete tendrá policy explícita.

## DB-SOFTDELETE-043

Repeated delete no refrescará deletion timestamp por default.

## DB-SOFTDELETE-044

Restore será explícito.

## DB-SOFTDELETE-045

Soft-delete restore será distinto de historical restore.

## DB-SOFTDELETE-046

Restore no tendrá éxito garantizado.

## DB-SOFTDELETE-047

Restore validará uniqueness.

## DB-SOFTDELETE-048

Pre-check de uniqueness no sustituirá constraint atómico.

## DB-SOFTDELETE-049

Physical uniqueness será distinta de active-only uniqueness.

## DB-SOFTDELETE-050

Active-only uniqueness será capability-driven.

## DB-SOFTDELETE-051

Vendor-specific uniqueness no contaminará public API.

## DB-SOFTDELETE-052

Relationship visibility será explícita.

## DB-SOFTDELETE-053

Physical FK será distinta de visible relationship.

## DB-SOFTDELETE-054

Eager y Lazy Loading preservarán la misma deletion semantics.

## DB-SOFTDELETE-055

Relationship override será explícito.

## DB-SOFTDELETE-056

Cascade soft delete estará deshabilitado por default.

## DB-SOFTDELETE-057

Parent deletion no implicará child deletion automáticamente.

## DB-SOFTDELETE-058

Cascade utilizará graph planning.

## DB-SOFTDELETE-059

Cascade no se implementará como recursive arbitrary calls.

## DB-SOFTDELETE-060

Restore cascade será independiente de delete cascade.

## DB-SOFTDELETE-061

Restore cascade requerirá provenance cuando sea necesario.

## DB-SOFTDELETE-062

Timestamp proximity no demostrará cascade provenance.

## DB-SOFTDELETE-063

Many-to-many membership no será eliminado automáticamente.

## DB-SOFTDELETE-064

Association entity podrá tener soft delete independiente.

## DB-SOFTDELETE-065

Polymorphic target resolverá su propia metadata.

## DB-SOFTDELETE-066

LEFT JOIN semantics no serán rotas por predicate injection textual.

## DB-SOFTDELETE-067

Soft-delete predicates participarán en Semantic Query Graph.

## DB-SOFTDELETE-068

Aggregates operarán sobre visible dataset.

## DB-SOFTDELETE-069

Pagination operará sobre visible dataset.

## DB-SOFTDELETE-070

Pagination total respetará visibility.

## DB-SOFTDELETE-071

Cursor fingerprint incluirá visibility.

## DB-SOFTDELETE-072

Cursor ACTIVE_ONLY no será reutilizable como INCLUDE_DELETED sin validación.

## DB-SOFTDELETE-073

Chunk Processing preservará visibility.

## DB-SOFTDELETE-074

Lazy Collection preservará visibility.

## DB-SOFTDELETE-075

Traversal visibility no mutará globalmente.

## DB-SOFTDELETE-076

Bulk soft delete será distinto de entity-aware soft delete.

## DB-SOFTDELETE-077

Bulk soft delete no fingirá entity lifecycle.

## DB-SOFTDELETE-078

Bulk history requerirá soporte explícito.

## DB-SOFTDELETE-079

UNKNOWN bulk capability no será tratado como supported.

## DB-SOFTDELETE-080

Force delete será distinto de soft delete.

## DB-SOFTDELETE-081

Force delete requerirá mayor control de seguridad.

## DB-SOFTDELETE-082

Force delete policy será configurable.

## DB-SOFTDELETE-083

DB ON DELETE CASCADE será distinto de SoftDeleteCascade.

## DB-SOFTDELETE-084

Soft delete podrá generar historical version.

## DB-SOFTDELETE-085

Restore podrá generar historical version.

## DB-SOFTDELETE-086

Hard delete no eliminará history implícitamente.

## DB-SOFTDELETE-087

Retention decidirá eventual purge.

## DB-SOFTDELETE-088

Soft delete podrá iniciar retention clock.

## DB-SOFTDELETE-089

Soft delete no implementará retention por sí mismo.

## DB-SOFTDELETE-090

Legal hold podrá impedir force delete.

## DB-SOFTDELETE-091

Archive podrá preceder purge.

## DB-SOFTDELETE-092

Soft delete invalidará semantic caches correspondientes.

## DB-SOFTDELETE-093

Uncommitted deletion no será publicada como committed.

## DB-SOFTDELETE-094

Rollback cancelará pending cache publication.

## DB-SOFTDELETE-095

UNKNOWN commit permitirá conservative invalidation.

## DB-SOFTDELETE-096

Cache keys distinguirán visibility.

## DB-SOFTDELETE-097

ACTIVE_ONLY cache será distinta de INCLUDE_DELETED cache.

## DB-SOFTDELETE-098

Soft-deleted data seguirá considerándose sensitive data.

## DB-SOFTDELETE-099

read_deleted será conceptualmente distinto de ordinary read.

## DB-SOFTDELETE-100

restore permission será distinta de delete permission.

## DB-SOFTDELETE-101

force_delete permission será distinta de soft_delete permission.

## DB-SOFTDELETE-102

withDeleted() será query semantics, no permission.

## DB-SOFTDELETE-103

No existirá global mutable soft-delete disable.

## DB-SOFTDELETE-104

Raw DB access será una lower-level escape hatch.

## DB-SOFTDELETE-105

Soft-delete ORM filtering no se presentará como database security boundary absoluta.

## DB-SOFTDELETE-106

Sensitive deleted operations serán auditables.

## DB-SOFTDELETE-107

Tenant filtering será independiente de deletion filtering.

## DB-SOFTDELETE-108

withDeleted() no eliminará tenant scope.

## DB-SOFTDELETE-109

Cross-tenant deleted query estará prohibida por default.

## DB-SOFTDELETE-110

Deletion visibility y tenant visibility serán dimensiones ortogonales.

## DB-SOFTDELETE-111

Authorization visibility será otra dimensión independiente.

## DB-SOFTDELETE-112

Temporal visibility será independiente de deletion visibility.

## DB-SOFTDELETE-113

Soft delete se ejecutará en shard propietario.

## DB-SOFTDELETE-114

Unknown shard no significará broadcast.

## DB-SOFTDELETE-115

Ordinary soft delete no hará broadcast multi-shard.

## DB-SOFTDELETE-116

Replica puede estar stale respecto al deletion state.

## DB-SOFTDELETE-117

Read-your-writes aplicará después de soft delete.

## DB-SOFTDELETE-118

Read-your-writes aplicará después de restore.

## DB-SOFTDELETE-119

External search index no será source of truth.

## DB-SOFTDELETE-120

External de-indexing deberá ocurrir post-commit/outbox cuando requiera certeza.

## DB-SOFTDELETE-121

External side-effect failure no revertirá mágicamente committed DB state.

## DB-SOFTDELETE-122

Persisted será distinto de committed.

## DB-SOFTDELETE-123

AfterCommit listener failure no deshará commit.

## DB-SOFTDELETE-124

Telemetry tendrá dimensiones bounded.

## DB-SOFTDELETE-125

Entity IDs no serán telemetry labels por default.

## DB-SOFTDELETE-126

Soft-delete predicates serán observables en diagnostics.

## DB-SOFTDELETE-127

Soft delete density será diagnosticable.

## DB-SOFTDELETE-128

Soft delete no será estrategia de garbage retention permanente.

## DB-SOFTDELETE-129

Mass deletion estará resource-governed.

## DB-SOFTDELETE-130

Mass deletion no asumirá una sola transaction ilimitada.

## DB-SOFTDELETE-131

Concurrent lifecycle operations serán controlables.

## DB-SOFTDELETE-132

Last-write-wins no será default universal.

## DB-SOFTDELETE-133

Restore conflict será explícito.

## DB-SOFTDELETE-134

Force delete conflict será explícito.

## DB-SOFTDELETE-135

EXECUTED será distinto de COMMITTED.

## DB-SOFTDELETE-136

FAILED será distinto de ROLLED_BACK.

## DB-SOFTDELETE-137

Retry dependerá de outcome certainty.

## DB-SOFTDELETE-138

UNKNOWN commit no se reintentará ciegamente.

## DB-SOFTDELETE-139

Marker idempotency no implicará lifecycle idempotency total.

## DB-SOFTDELETE-140

DeletionOperationId podrá correlacionar cascades.

## DB-SOFTDELETE-141

DeletionOperationId no será high-cardinality telemetry label.

## DB-SOFTDELETE-142

SoftDeleteVisibility será scope-local.

## DB-SOFTDELETE-143

No existirá static `$includeDeleted`.

## DB-SOFTDELETE-144

FrankenPHP worker no filtrará deletion visibility.

## DB-SOFTDELETE-145

RoadRunner worker no filtrará deletion visibility.

## DB-SOFTDELETE-146

OpenSwoole coroutine no compartirá deletion visibility mutable.

## DB-SOFTDELETE-147

Runtime reset limpiará temporary soft-delete state.

## DB-SOFTDELETE-148

Serialization no expondrá deleted entities accidentalmente.

## DB-SOFTDELETE-149

Queue jobs declararán visibility cuando necesiten deleted entities.

## DB-SOFTDELETE-150

Jobs no dependerán de global deletion visibility.

## DB-SOFTDELETE-151

Schema Builder podrá expresar soft-delete columns.

## DB-SOFTDELETE-152

Schema Builder no generará SQL directamente.

## DB-SOFTDELETE-153

Soft-delete schema changes pasarán por Migration System.

## DB-SOFTDELETE-154

Active-only uniqueness migration pasará por Migration Safety.

## DB-SOFTDELETE-155

Zero-downtime soft-delete adoption será soportable.

## DB-SOFTDELETE-156

Removing soft delete requerirá decisión sobre deleted rows.

## DB-SOFTDELETE-157

No se realizará destructive migration decision implícita.

## DB-SOFTDELETE-158

Custom marker producirá semantic predicates.

## DB-SOFTDELETE-159

Custom marker no requerirá raw SQL.

## DB-SOFTDELETE-160

Platform differences se resolverán mediante capabilities.

## DB-SOFTDELETE-161

MySQL y MariaDB permanecerán plataformas separadas.

## DB-SOFTDELETE-162

Public API no expondrá vendor-specific syntax.

## DB-SOFTDELETE-163

SoftDeleteManager no será TransactionManager.

## DB-SOFTDELETE-164

SoftDeleteManager no será AuthorizationEngine.

## DB-SOFTDELETE-165

SoftDeleteManager no será HistoryEngine.

## DB-SOFTDELETE-166

SoftDeletePlanner no ejecutará SQL.

## DB-SOFTDELETE-167

RestorePlanner no ejecutará SQL.

## DB-SOFTDELETE-168

ForceDeletePlanner no ejecutará SQL.

## DB-SOFTDELETE-169

Compiler no decidirá lifecycle semantics.

## DB-SOFTDELETE-170

Executor no decidirá si una entidad está soft-deletable.

## DB-SOFTDELETE-171

Connection no conocerá SoftDeleteMetadata.

## DB-SOFTDELETE-172

Driver no conocerá SoftDeleteVisibility.

## DB-SOFTDELETE-173

ORM definirá lifecycle semantics.

## DB-SOFTDELETE-174

Query Engine representará semantic predicates.

## DB-SOFTDELETE-175

Compiler representará esos predicates en SQL.

## DB-SOFTDELETE-176

Executor ejecutará sin interpretar soft-delete domain semantics.

## DB-SOFTDELETE-177

Restore será una persistence mutation normal.

## DB-SOFTDELETE-178

Force delete será una physical persistence mutation explícita.

## DB-SOFTDELETE-179

Soft delete no sustituirá archival.

## DB-SOFTDELETE-180

Soft delete no sustituirá data lifecycle governance.

---

# 309. Modelo formal

Sea:

```text
E
```

una entidad soft-deletable.

Definimos:

```text
D(E) ∈ {ACTIVE, SOFT_DELETED}
```

---

# 310. Timestamp marker

Para marker temporal:

```text
D(E) = ACTIVE
⇔
deletedAt(E) = NULL
```

y:

```text
D(E) = SOFT_DELETED
⇔
deletedAt(E) ≠ NULL
```

---

# 311. Active query

Para query lógica:

```text
Q
```

la query default será:

```text
Qactive =
Q ∩ { E | D(E) = ACTIVE }
```

---

# 312. Deleted query

```text
Qdeleted =
Q ∩ { E | D(E) = SOFT_DELETED }
```

---

# 313. Include deleted

```text
Qall =
Qactive ∪ Qdeleted
```

bajo el mismo resto de scopes.

---

# 314. Tenant composition

Para tenant:

```text
T
```

y deletion visibility:

```text
V
```

el conjunto observable será:

```text
Visible(Q,T,V,A)
=
Q
∩ TenantScope(T)
∩ DeletionScope(V)
∩ AuthorizationScope(A)
```

---

# 315. Soft delete transition

```text
SoftDelete(E, t)
:
D(E)=ACTIVE
→
D(E)=SOFT_DELETED
```

con:

```text
deletedAt(E)=t
```

para timestamp strategy.

---

# 316. Restore transition

```text
Restore(E)
:
D(E)=SOFT_DELETED
→
D(E)=ACTIVE
```

si:

```text
Authorization
∧ Constraints
∧ LifecyclePolicy
```

son válidos.

---

# 317. Force delete

```text
ForceDelete(E)
:
ExistsPhysical(E)
→
¬ExistsPhysical(E)
```

sujeto a:

```text
Authorization
∧ RetentionPolicy
∧ LegalPolicy
∧ ReferentialIntegrity
∧ HistoryPolicy
```

---

# 318. Arquitectura conceptual final

```text
                         Entity
                           │
                           ▼
                    Lifecycle State
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
           ACTIVE                 SOFT_DELETED
              │                         │
              │                         ├──── restore ────┐
              │                         │                 │
              │                         └─ force delete   │
              │                                           │
              └──────── soft delete ──────────────────────┘

Query
 │
 ▼
Entity Query Context
 │
 ├── Tenant
 ├── Authorization
 ├── Temporal View
 └── SoftDeleteVisibility
          │
          ▼
 Semantic Query Model
          │
          ▼
      Predicate AST
          │
          ▼
       Optimizer
          │
          ▼
        Planner
          │
          ▼
       Compiler
          │
          ▼
        Executor
          │
          ▼
       Database
```

---

# 319. Reglas definitivas

> **Soft Delete es una política de ciclo de vida y visibilidad, no una técnica de concatenación SQL.**

Por tanto:

```text
Soft Delete
≠ Hard Delete
```

```text
Soft Delete
≠ Archive
```

```text
Soft Delete
≠ Retention
```

```text
Soft Delete
≠ History
```

```text
Soft Delete
≠ Authorization
```

```text
Soft Delete Restore
≠ Historical Restore
```

```text
DB Cascade
≠ Soft Delete Cascade
```

y:

```text
withDeleted()
≠ bypassSecurity()
```

---

# 320. Resultado arquitectónico

VoltStack podrá ofrecer una experiencia simple:

```php
$customer->delete();

Customer::find($id);

Customer::query()
    ->withDeleted()
    ->find($id);

Customer::query()
    ->onlyDeleted()
    ->paginate();

$customer->restore();

$customer->forceDelete();
```

mientras internamente mantiene separadas:

```text
Deletion State
Visibility
Query Semantics
Persistence
Transactions
Optimistic Locking
Relationships
Cascade
History
Retention
Archive
Cache
Authorization
Multitenancy
Sharding
Telemetry
Resource Governance
Persistent Runtime Isolation
```

---

# 321. Decisión arquitectónica final

VoltStack adoptará un **Soft Delete System semántico, metadata-driven y ORM-aware**, integrado con el Query Engine sin introducir lógica SQL específica en Model, Repository o EntityManager.

La arquitectura seguirá:

```text
Entity Metadata
      ↓
Soft Delete Metadata
      ↓
Entity Query Context
      ↓
Deletion Visibility
      ↓
Semantic Predicate
      ↓
Query AST
      ↓
Planner
      ↓
Compiler
```

para lectura, y:

```text
delete()/restore()/forceDelete()
             ↓
      Lifecycle Planner
             ↓
         UnitOfWork
             ↓
         ChangeSet
             ↓
     Persistence Planner
             ↓
        Transaction
             ↓
          Database
```

para mutaciones.

El principio rector será:

> **Una entidad soft-deleted continúa existiendo físicamente, pero cambia explícitamente su estado lógico y su visibilidad. VoltStack debe preservar esa distinción en todo el sistema, desde el Query Engine hasta ORM, relaciones, seguridad, historial, cache, multitenancy y runtime persistente.**

---

# 322. Bloque 27 — progreso

```text
BLOCK 27 — ADVANCED DATABASE CAPABILITIES

✓ 267_DATABASE_TEMPORAL_DATA_SYSTEM.md
✓ 268_DATABASE_HISTORY_AND_VERSIONING_SYSTEM.md
✓ 269_DATABASE_SOFT_DELETE_SYSTEM.md
□ 270_DATABASE_DATA_RETENTION_SYSTEM.md
□ 271_DATABASE_DATA_ARCHIVAL_SYSTEM.md
□ 272_DATABASE_FULL_TEXT_SEARCH_SYSTEM.md
□ 273_DATABASE_JSON_QUERY_SYSTEM.md
□ 274_DATABASE_GEOGRAPHIC_DATA_EXTENSION_SYSTEM.md
□ 275_DATABASE_DATABASE_FEATURE_CAPABILITY_SYSTEM.md
```

---

# 323. Siguiente documento

```text
270_DATABASE_DATA_RETENTION_SYSTEM.md
```

El siguiente documento definirá:

```text
Retention Policies
Retention Rules
Retention Windows
Retention Clock
Retention Anchors
Expiration
Purge Eligibility
Legal Hold
Minimum Retention
Maximum Retention
Soft Delete Retention
History Retention
Tenant Retention
Data Classification
Retention Planner
Retention Enforcement
Retention Jobs
Safe Purging
Batch Purging
Distributed Retention
Retention + Archive
Retention + Backup
Retention + Security
Retention + Audit
Retention + Telemetry
Retention + Persistent Runtimes
```

estableciendo como principio:

> **Data Retention determinará cuánto tiempo debe conservarse la información y cuándo puede o debe eliminarse; no será equivalente a soft delete, archive, backup, TTL de cache ni simple borrado periódico.**