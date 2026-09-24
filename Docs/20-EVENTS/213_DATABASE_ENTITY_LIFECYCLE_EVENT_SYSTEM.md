# 213_DATABASE_ENTITY_LIFECYCLE_EVENT_SYSTEM.md

# VoltStack Quantum Database
## Database Entity Lifecycle Event System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 213 — Database Entity Lifecycle Event System  
**Bloque:** 20 — Events  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `212_DATABASE_TRANSACTION_EVENT_PIPELINE.md`  
**Siguiente documento:** `214_DATABASE_PERSISTENCE_EVENT_SYSTEM.md`

---

# 1. Propósito

`Database Entity Lifecycle Event System` define cómo VoltStack representará y distribuirá eventos relacionados con el ciclo de vida ORM de una entidad.

El sistema cubrirá:

```text
Entity Construction
↓
Hydration
↓
Identity Resolution
↓
Managed Registration
↓
Lifecycle State
↓
Persist Scheduling
↓
Change Detection
↓
Update Scheduling
↓
Remove Scheduling
↓
Flush
↓
Post-Persistence State
↓
Detach / Clear
```

sin confundir:

```text
Entity State
```

con:

```text
Database Durability
```

La regla central será:

> **Un Entity Lifecycle Event describe un cambio observable en el estado de una entidad dentro del ORM; no prueba por sí mismo que una fila haya sido insertada, actualizada, eliminada o committed de forma durable en la base de datos.**

---

# 2. Relación con la arquitectura ORM existente

Este documento especializa:

```text
112_DATABASE_ORM_ARCHITECTURE.md
113_DATABASE_ENTITY_MODEL.md
114_DATABASE_MODEL_API_SYSTEM.md
115_DATABASE_ENTITY_METADATA_SYSTEM.md
116_DATABASE_ENTITY_MAPPING_SYSTEM.md
118_DATABASE_ENTITY_MANAGER_SYSTEM.md
121_DATABASE_ENTITY_STATE_SYSTEM.md
122_DATABASE_ENTITY_LIFECYCLE_SYSTEM.md
123_DATABASE_IDENTITY_MAP_SYSTEM.md
124_DATABASE_UNIT_OF_WORK_ARCHITECTURE.md
125_DATABASE_CHANGE_TRACKING_SYSTEM.md
126_DATABASE_ENTITY_SNAPSHOT_SYSTEM.md
127_DATABASE_PERSISTENCE_ENGINE.md
132_DATABASE_FLUSH_SYSTEM.md
135_DATABASE_HYDRATION_ARCHITECTURE.md
136_DATABASE_ENTITY_HYDRATOR_SYSTEM.md
209_DATABASE_EVENT_ARCHITECTURE.md
212_DATABASE_TRANSACTION_EVENT_PIPELINE.md
```

---

# 3. Objetivos

El sistema deberá permitir observar:

```text
entity hydration
managed registration
entity state transitions
persist scheduling
remove scheduling
change detection
pre-persistence lifecycle
post-persistence lifecycle
detach
clear
refresh
merge/reattach if supported
partial entity lifecycle
relationship lifecycle correlation
transaction-aware entity events
after-commit entity notifications
```

sin romper:

```text
IdentityMap
UnitOfWork
EntityManager
Persistence Engine
Transaction Manager
```

---

# 4. Distinciones fundamentales

VoltStack deberá mantener:

```text
Entity Lifecycle Event
≠
Domain Event
≠
Persistence Event
≠
Query Event
≠
Transaction Event
≠
Entity State
≠
Database Row State
≠
Database Commit
```

---

# 5. Entity Lifecycle Event ≠ Domain Event

Ejemplo de lifecycle:

```text
EntityPostLoad
```

Ejemplo de dominio:

```text
CustomerActivated
```

El primero describe el ORM.

El segundo describe negocio.

---

# 6. Entity Lifecycle Event ≠ Persistence Event

```text
EntityPrePersist
```

describe el lifecycle de una entidad.

```text
PersistenceInsertExecuted
```

describe el pipeline de persistencia.

Pueden correlacionarse, pero no son lo mismo.

---

# 7. Entity Lifecycle Event ≠ Query Event

Una entidad puede convertirse en `MANAGED` sin que se ejecute una query nueva, por ejemplo si ya está en IdentityMap.

---

# 8. Entity Lifecycle Event ≠ Transaction Event

Un evento:

```text
EntityPostPersist
```

no equivale a:

```text
TransactionCommitted
```

---

# 9. Entity state ≠ Database state

Ejemplo:

```text
Entity State = MANAGED
```

no implica:

```text
row exists durably in database
```

si aún no hubo flush/commit.

---

# 10. EntityPersisted ≠ EntityInserted

La palabra `persisted` puede ser ambigua.

VoltStack deberá evitar nombres que mezclen:

```text
registered for persistence
```

con:

```text
INSERT executed
```

---

# 11. Naming rule

Preferir:

```text
EntityPersistScheduled
EntityInsertPlanned
EntityInsertExecuted
TransactionCommitted
```

a un genérico:

```text
EntityPersisted
```

sin semántica precisa.

---

# 12. Arquitectura general

```text
Entity
  │
  ▼
EntityManager
  │
  ▼
UnitOfWork
  │
  ├── lifecycle state changes
  ├── persist/remove scheduling
  └── change tracking
  │
  ▼
Persistence Engine
  │
  ▼
Flush
  │
  ▼
Query Engine
  │
  ▼
Transaction
```

Los Entity Lifecycle Events observarán principalmente:

```text
EntityManager
UnitOfWork
Hydration
EntityState
```

---

# 13. EntityLifecycleEvent contract

```php
interface EntityLifecycleEvent extends DatabaseEvent
{
    public function entityType(): EntityType;

    public function entityIdentity(): EntityIdentityReference;

    public function lifecycleContext(): EntityLifecycleEventContext;
}
```

---

# 14. EntityIdentityReference

No deberá transportar la entidad completa.

```php
final readonly class EntityIdentityReference
{
    public function __construct(
        public EntityType $type,
        public ?CanonicalEntityIdentifier $identifier,
        public EntityRuntimeId $runtimeId,
    ) {}
}
```

---

# 15. EntityRuntimeId

Será útil para entidades NEW sin identificador de DB todavía.

---

# 16. Runtime ID ≠ Database ID

```text
EntityRuntimeId
≠
CanonicalEntityIdentifier
```

---

# 17. Event payload should not contain live entity by default

Aunque listeners in-process puedan necesitar acceso controlado, el evento base deberá preferir:

```text
identity
state
metadata summary
change summary
```

sobre:

```text
entire mutable entity object
```

---

# 18. Why avoid live entity payloads

Porque introduce:

```text
hidden mutation
serialization problems
async incompatibility
memory retention
reentrancy risk
lifecycle corruption
```

---

# 19. Entity reference access

Podrá existir un contrato especializado in-process:

```php
interface LocalEntityLifecycleEvent extends EntityLifecycleEvent
{
    public function entityReference(): EntityReference;
}
```

pero:

```text
LocalEntityLifecycleEvent
≠
ExternalizableEntityLifecycleEvent
```

---

# 20. Lifecycle state model

Estados conceptuales:

```text
NEW
MANAGED
DIRTY
REMOVED
DETACHED
```

podrán complementarse con:

```text
RESERVED
PARTIAL
STALE
UNKNOWN
```

según subsistema.

---

# 21. State transition events

Familia:

```text
EntityStateTransitionStarted
EntityStateChanged
EntityStateTransitionFailed
```

aunque no todos deberán exponerse públicamente.

---

# 22. State transition model

```php
final readonly class EntityStateTransition
{
    public function __construct(
        public EntityState $from,
        public EntityState $to,
        public EntityStateTransitionReason $reason,
    ) {}
}
```

---

# 23. Transition reason

```php
enum EntityStateTransitionReason
{
    case HYDRATION;
    case PERSIST_SCHEDULED;
    case CHANGE_DETECTED;
    case REMOVE_SCHEDULED;
    case FLUSH_COMPLETED;
    case DETACH;
    case CLEAR;
    case REFRESH;
    case ROLLBACK_RECONCILIATION;
    case OTHER;
}
```

---

# 24. Hydration lifecycle

Conceptualmente:

```text
Row
↓
Hydration Plan
↓
Entity instance / reservation
↓
IdentityMap
↓
Field assignment
↓
Snapshot
↓
Managed registration
↓
PostLoad
```

---

# 25. Hydration events

Podrán existir:

```text
EntityHydrationStarted
EntityHydrated
EntityHydrationFailed
EntityPostLoad
```

---

# 26. EntityHydrated ≠ EntityLoadedFromDatabase

Puede existir hydration desde:

```text
result cache
second-level entity cache
database result
custom source
```

Por tanto:

```text
Hydrated
≠
DB Query Executed
```

---

# 27. EntityHydrated ≠ Managed

Hydration puede construir:

```text
DTO
detached entity
read-only representation
```

según modo.

Para managed entities, deberá existir un boundary claro.

---

# 28. Managed registration event

```text
EntityManaged
```

podrá representar:

> la entidad ha sido incorporada a IdentityMap/UoW bajo un EntityManager scope.

---

# 29. EntityManaged ≠ EntityInserted

Una entidad hidratada existente puede ser `MANAGED`.

Una entidad NEW recién persist-scheduled también puede convertirse en managed según política.

---

# 30. IdentityMap invariant

Para:

```text
EntityType + Identifier + Effective Context
```

debe existir una única instancia managed por scope.

---

# 31. EntityManaged event and IdentityMap

El evento deberá emitirse después de que IdentityMap haya establecido la identidad canónica.

---

# 32. Reservation lifecycle

Durante hydration con ciclos podrá existir:

```text
RESERVED
↓
ACTIVE
```

No será necesario exponer cada reserva como evento público.

---

# 33. Internal lifecycle events

Eventos de bajo nivel podrán ser:

```text
INTERNAL
```

para diagnostics/testing.

---

# 34. Persist scheduling

`EntityManager::persist($entity)` deberá significar:

```text
register/schedule
```

no:

```text
INSERT now
```

---

# 35. Event naming

Por tanto:

```text
EntityPersistScheduled
```

será preferible a:

```text
EntityPersisted
```

para ese boundary.

---

# 36. Persist scheduling event

```php
final readonly class EntityPersistScheduled implements EntityLifecycleEvent
{
    public function __construct(
        public EntityLifecycleEventContext $context,
        public EntityState $previousState,
        public EntityState $currentState,
    ) {}
}
```

---

# 37. Persist scheduled ≠ Flush

```text
persist()
↓
EntityPersistScheduled

later...

flush()
```

---

# 38. Persist scheduled ≠ INSERT planned

Puede faltar:

```text
change analysis
dependency planning
generated ID strategy
relationship planning
```

---

# 39. Persist scheduled ≠ INSERT executed

Nunca deberán confundirse.

---

# 40. Remove scheduling

`EntityManager::remove()` deberá producir conceptualmente:

```text
EntityRemoveScheduled
```

---

# 41. Remove scheduled ≠ DELETE executed

```text
remove()
≠
DELETE
```

---

# 42. Remove scheduled ≠ Entity detached

Una entidad marcada REMOVED puede permanecer managed hasta flush/completion según política.

---

# 43. Dirty detection

Cuando Change Tracking detecta modificaciones:

```text
MANAGED
↓
DIRTY
```

podrá emitirse:

```text
EntityDirtyDetected
```

---

# 44. Dirty detected ≠ UPDATE required

Algunas diferencias pueden:

```text
normalize away
not be mapped
be ignored by policy
be version-only
```

Por tanto, el Persistence Planner decide.

---

# 45. ChangeSet event

Un evento podrá incluir:

```text
field count
changed field names under policy
change-set fingerprint
```

pero no necesariamente valores completos.

---

# 46. Sensitive changes

Cambios de:

```text
password_hash
token
secret
PII
```

no deberán aparecer con valores raw.

---

# 47. EntityChangeSummary

```php
final readonly class EntityChangeSummary
{
    public function __construct(
        public int $changedFieldCount,
        public array $fieldNames,
        public ChangeSetFingerprint $fingerprint,
    ) {}
}
```

---

# 48. Pre lifecycle events

Podrán existir:

```text
EntityPrePersist
EntityPreUpdate
EntityPreRemove
```

---

# 49. Pre lifecycle events are dangerous extension points

Deben definirse cuidadosamente porque pueden permitir mutación del entity.

---

# 50. Pre lifecycle event ≠ generic Database Event

Si `prePersist` permite mutación deliberada, deberá tratarse como:

```text
ORM Lifecycle Hook
```

especializado.

---

# 51. Observer vs Hook split

Se recomienda separar:

```text
EntityPersistSchedulingObserved
```

de:

```text
EntityPrePersistHook
```

---

# 52. Why split

Para evitar que:

```text
listener order
```

cambie silenciosamente:

```text
entity data
change sets
persistence plan
```

---

# 53. Lifecycle hook contract

```php
interface EntityLifecycleHook
{
    public function invoke(
        EntityLifecycleHookContext $context
    ): void;
}
```

---

# 54. Hook mutability

Los hooks podrán tener permisos explícitos:

```php
enum EntityLifecycleHookCapability
{
    case READ_ONLY;
    case MUTATE_ENTITY;
    case VETO;
}
```

---

# 55. Generic listeners remain observational

La arquitectura de eventos general de 209 permanece:

```text
observe by default
```

---

# 56. PrePersist hook timing

Debe ejecutarse antes de que el change-set/persistence plan quede definitivamente congelado.

---

# 57. Hook-induced mutation

Si un `prePersist` modifica una entidad, el sistema deberá:

```text
recompute relevant change-set
```

de forma explícita.

---

# 58. PreUpdate hook timing

Similarmente, si modifica campos:

```text
ChangeSet
↓
PreUpdate Hook
↓
Recompute/validate ChangeSet
↓
Persistence Plan
```

según contrato.

---

# 59. Infinite mutation prevention

No deberá existir:

```text
recompute
↓
hook changes
↓
recompute
↓
hook changes
↓
...
```

sin límites.

---

# 60. Hook iteration policy

```php
final readonly class LifecycleHookPolicy
{
    public function __construct(
        public int $maxMutationRounds,
    ) {}
}
```

---

# 61. Veto semantics

Un hook con capacidad VETO podrá impedir:

```text
persist
update
remove
```

antes de que ocurra el statement.

---

# 62. Veto ≠ listener exception

Debe modelarse explícitamente:

```text
LifecycleDecision::ALLOW
LifecycleDecision::DENY
```

---

# 63. Post lifecycle events

Podrán existir:

```text
EntityPostLoad
EntityPostPersist
EntityPostUpdate
EntityPostRemove
```

pero sus significados deben ser precisos.

---

# 64. EntityPostPersist

Deberá significar uno de dos posibles boundaries:

### Opción A

Después de que INSERT fue ejecutado con éxito.

### Opción B

Después de que la transacción fue committed.

No deberán mezclarse.

---

# 65. Recomendación VoltStack

Usar nombres distintos:

```text
EntityInsertExecuted
```

para statement/persistence success.

Y:

```text
EntityInsertCommitted
```

o evento after-commit especializado si realmente se necesita commit durability.

---

# 66. Avoid ambiguous `postPersist`

Si se ofrece por compatibilidad/ergonomía, deberá documentarse su boundary exacto.

---

# 67. Proposed semantics

Para VoltStack:

```text
EntityPostPersist
=
persistence operation for entity completed within current flush
```

No implica commit.

---

# 68. EntityPostUpdate

Misma regla:

```text
UPDATE execution success
≠
transaction commit
```

---

# 69. EntityPostRemove

Misma regla:

```text
DELETE execution success
≠
transaction commit
```

---

# 70. Commit-aware entity events

Si se requieren:

```text
EntityCreatedCommitted
EntityUpdatedCommitted
EntityDeletedCommitted
```

deberán ser deferred `AFTER_COMMIT`.

---

# 71. Commit-aware event source

Deberán registrarse en:

```text
TransactionEventPipeline
```

durante flush/persistence.

---

# 72. Rollback behavior

Si transaction hace rollback:

```text
EntityCreatedCommitted
```

no debe emitirse.

---

# 73. UNKNOWN transaction outcome

Tampoco deberá emitirse `...Committed`.

---

# 74. AfterRollback entity events

Opcionalmente podrán existir:

```text
EntityInsertRolledBack
EntityUpdateRolledBack
EntityDeleteRolledBack
```

para diagnostics/integration.

---

# 75. Rollback events ≠ automatic PHP rewind

Aunque se emita:

```text
EntityUpdateRolledBack
```

la entidad PHP puede seguir conteniendo valores modificados.

---

# 76. Entity state after rollback

Debe ser resuelto por:

```text
Persistence Consistency System
EntityManager reconciliation policy
```

no por el evento.

---

# 77. Snapshot restoration

VoltStack no deberá fingir:

```text
rollback
↓
entity automatically restored to old values
```

salvo que una política concreta lo implemente.

---

# 78. Detach events

Podrán existir:

```text
EntityDetachStarted
EntityDetached
```

o simplemente:

```text
EntityDetached
```

---

# 79. Detached semantics

Significa:

```text
Entity no longer tracked by this EntityManager
```

No:

```text
entity deleted from DB
```

---

# 80. Clear events

`EntityManager::clear()` podrá producir:

```text
EntityManagerCleared
```

en lugar de millones de:

```text
EntityDetached
```

por default.

---

# 81. Why aggregate clear event

Para evitar event storms.

---

# 82. Optional detailed clear

En debug/testing podría emitirse:

```text
per-entity detach events
```

bajo una policy explícita.

---

# 83. EntityManagerCleared payload

```php
final readonly class EntityManagerClearSummary
{
    public function __construct(
        public ?EntityType $type,
        public int $entityCount,
        public ClearReason $reason,
    ) {}
}
```

---

# 84. Clear reason

```php
enum ClearReason
{
    case EXPLICIT;
    case MEMORY_GOVERNANCE;
    case ERROR_RECOVERY;
    case TRANSACTION_UNKNOWN;
    case REQUEST_END;
    case OTHER;
}
```

---

# 85. Request-end clear

En persistent runtimes, el EntityManager scope puede cerrarse al terminar request/operation.

---

# 86. Scope end ≠ EntityRemoved

Las entidades dejan de estar managed, no son borradas.

---

# 87. Refresh lifecycle

`refresh($entity)` puede:

```text
query database
↓
hydrate current DB state
↓
replace managed field state
↓
snapshot reset
```

---

# 88. Refresh events

Podrán existir:

```text
EntityRefreshStarted
EntityRefreshed
EntityRefreshFailed
```

---

# 89. Refresh ≠ merge

No deberán confundirse.

---

# 90. Merge/reattach

Si VoltStack soporta reattachment:

```text
Detached Entity
↓
Merge
↓
Managed canonical instance
```

deberá existir semántica explícita.

---

# 91. Merge identity rule

No debe crear:

```text
second managed instance with same identity
```

---

# 92. Merge events

Podrían ser:

```text
EntityMergeStarted
EntityMerged
EntityMergeConflict
```

si la feature existe.

---

# 93. Partial entities

Una entidad parcial tiene:

```text
LoadedFieldMask
```

y no debe tratarse como completa.

---

# 94. Partial hydration event

`EntityHydrated` podrá incluir:

```text
FULL
PARTIAL
```

---

# 95. Partial ≠ complete

Esto deberá ser visible en lifecycle context.

---

# 96. Partial entity update danger

Modificar campos no cargados puede tener riesgos.

El persistence system deberá mantener reglas explícitas.

Los eventos solo lo reflejan.

---

# 97. EntityLifecycleEventContext

```php
final readonly class EntityLifecycleEventContext
{
    public function __construct(
        public EntityManagerScopeId $entityManager,
        public EntityRuntimeId $runtimeId,
        public EntityType $entityType,
        public ?CanonicalEntityIdentifier $identifier,
        public EntityState $state,
        public ?LoadedFieldMaskSummary $loadedFields,
        public ?TransactionId $transactionId,
        public ?PersistenceOperationId $persistenceOperationId,
        public PersistenceDomain $domain,
        public ?TenantContextReference $tenant,
        public ?ShardId $shard,
        public EntityLifecycleMetadata $metadata,
    ) {}
}
```

---

# 98. EntityManagerScopeId

Identifica el scope ORM actual.

No deberá confundirse con:

```text
RequestId
TransactionId
```

---

# 99. Same entity across scopes

La misma fila puede producir diferentes instancias PHP en scopes distintos.

---

# 100. Entity runtime identity is scope-sensitive

Por tanto:

```text
EntityRuntimeId
```

solo tiene sentido en el runtime scope apropiado.

---

# 101. Lifecycle event correlation

```text
EntityManagerScopeId
↓
EntityRuntimeId
↓
EntityType + Identifier
↓
PersistenceOperationId?
↓
TransactionId?
↓
QueryId?
```

---

# 102. Generated identifiers

Una NEW entity puede comenzar con:

```text
identifier = null
```

---

# 103. Identifier assignment event

Cuando un generated ID queda disponible:

```text
EntityIdentifierAssigned
```

podrá emitirse.

---

# 104. Identifier assigned ≠ committed

Ejemplo:

```text
INSERT succeeds
↓
generated ID = 100
↓
transaction rolls back
```

La entidad PHP puede seguir teniendo:

```text
id = 100
```

aunque la fila no exista committed.

---

# 105. Generated ID rollback problem

Debe quedar bajo Persistence Consistency policy.

El evento no resolverá automáticamente el ID.

---

# 106. Identifier mutation

Para entidades managed existentes, cambiar ID normalmente deberá estar prohibido o fuertemente restringido.

---

# 107. EntityIdentifierChanged event

No será un evento normal.

Una mutación ilegal deberá ser error, no lifecycle esperado.

---

# 108. Version fields

Optimistic locking puede asignar:

```text
version
```

durante update.

---

# 109. Version updated ≠ committed

Misma regla.

---

# 110. Optimistic conflict

Podrá producir:

```text
EntityOptimisticLockConflict
```

como lifecycle/integration event especializado.

---

# 111. Conflict ≠ failure of whole transaction automatically

La política superior decide.

---

# 112. Relationship lifecycle

Cambios en relaciones podrán causar:

```text
collection membership changes
foreign-key changes
association entity changes
```

---

# 113. Relationship Event ≠ Entity Lifecycle Event

Aunque se relacionen.

Podrán existir eventos especializados de relationship si se justifican.

---

# 114. Collection mutation events

No deberán generar automáticamente millones de eventos de alta cardinalidad.

---

# 115. Relationship persistence

La autoridad está en:

```text
Relationship Persistence System
```

---

# 116. Orphan removal

Si una relación provoca remove scheduling:

```text
EntityRemoveScheduled
```

deberá indicar reason:

```text
ORPHAN_REMOVAL
```

---

# 117. Cascade persist

Similarmente:

```text
EntityPersistScheduled
```

puede indicar:

```text
CASCADE
```

---

# 118. Scheduling reason

```php
enum EntitySchedulingReason
{
    case EXPLICIT;
    case CASCADE;
    case ORPHAN_REMOVAL;
    case RELATIONSHIP_CHANGE;
    case INTERNAL;
}
```

---

# 119. Event storms

Cascade graphs enormes pueden generar miles de entity events.

---

# 120. Volume governance

Los lifecycle events deberán clasificarse como:

```text
MEDIUM
HIGH
VERY_HIGH
```

según tipo.

---

# 121. Per-entity events

Seguirán siendo útiles para hooks precisos.

Pero externalización/telemetry deberá poder agregarlos.

---

# 122. Batch summary events

Podrán existir:

```text
EntityLifecycleBatchSummary
```

para:

```text
inserted count
updated count
removed count
```

sin reemplazar eventos funcionales necesarios.

---

# 123. Factory-created entities

Una entidad creada por factory:

```text
NEW
```

no deberá producir lifecycle DB events hasta interactuar con EntityManager según policy.

---

# 124. Constructor ≠ ORM lifecycle

Crear:

```php
new User(...)
```

no significa:

```text
EntityPersistScheduled
```

---

# 125. Model API

`$model->save()` será convenience API.

Internamente deberá converger:

```text
Model API
↓
EntityManager/UoW
↓
Lifecycle Events
```

---

# 126. Active Record facade ≠ second lifecycle engine

No habrá eventos distintos/incompatibles para Model API.

---

# 127. Repository API

También convergerá al mismo lifecycle.

---

# 128. Hydration baseline rule

Durante hydration:

```text
field assignment
```

no deberá marcar entidad DIRTY.

---

# 129. EntityHydrated event timing

Deberá emitirse después de establecer:

```text
identity
loaded fields
canonical converted values
```

pero antes/después de baseline snapshot según evento específico.

---

# 130. PostLoad timing

Recomendación:

```text
hydrate fields
↓
establish baseline snapshot
↓
register managed state
↓
EntityPostLoad
```

---

# 131. PostLoad mutation

Si `postLoad` hook modifica mapped fields:

```text
entity becomes DIRTY
```

después del baseline.

---

# 132. This is intentional

No deberá esconderse como parte de hydration.

---

# 133. PostLoad observer vs hook

De nuevo:

```text
observer
≠
mutating hook
```

---

# 134. Lazy loading

Cargar una relación lazy no deberá disparar:

```text
EntityPostLoad
```

para la entidad raíz de nuevo.

---

# 135. Newly hydrated related entity

Sí podrá disparar su propio lifecycle.

---

# 136. Second-level entity cache

Hydration desde Entity Cache puede producir:

```text
EntityHydrated
```

con source:

```text
ENTITY_CACHE
```

---

# 137. HydrationSource

```php
enum EntityHydrationSource
{
    case DATABASE_RESULT;
    case ENTITY_CACHE;
    case RESULT_CACHE;
    case CUSTOM;
}
```

---

# 138. Hydration source ≠ entity state

Una entidad sigue pudiendo quedar MANAGED independientemente de su origen.

---

# 139. Refresh source

Podrá utilizar el mismo source model.

---

# 140. Lifecycle origin

```php
enum EntityLifecycleOrigin
{
    case ORM;
    case MODEL_API;
    case REPOSITORY;
    case HYDRATION;
    case RELATIONSHIP;
    case CASCADE;
    case FACTORY;
    case IMPORT;
    case INTERNAL;
}
```

---

# 141. Import with ROW_BULK

No deberá producir entity lifecycle events si no se construyen entidades.

---

# 142. Import with ORM_ENTITY

Sí puede producirlos.

---

# 143. Bulk Update/Delete

Set-based bulk operations normalmente bypass entity lifecycle.

---

# 144. Important invariant

```text
Bulk Update
≠
millions of EntityPreUpdate/PostUpdate
```

---

# 145. Why

Porque:

```text
entities were not loaded
UoW did not manage per-entity changes
```

---

# 146. Bulk lifecycle diagnostics

El Bulk system deberá emitir sus propios eventos.

No fingirá entity events.

---

# 147. Entity lifecycle and transaction commit

Una entidad puede atravesar:

```text
NEW
↓
MANAGED
↓
insert executed
↓
transaction committed
```

---

# 148. Distinct event layers

```text
EntityPersistScheduled
↓
EntityPrePersist Hook
↓
PersistenceInsertPlanned
↓
PersistenceInsertExecuted
↓
EntityPostPersist
↓
TransactionCommitted
↓
EntityInsertCommitted (optional)
```

---

# 149. Update flow

```text
MANAGED
↓
property mutation
↓
EntityDirtyDetected
↓
EntityPreUpdate
↓
PersistenceUpdatePlanned
↓
PersistenceUpdateExecuted
↓
EntityPostUpdate
↓
TransactionCommitted
↓
EntityUpdateCommitted
```

---

# 150. Remove flow

```text
MANAGED
↓
remove()
↓
EntityRemoveScheduled
↓
EntityPreRemove
↓
PersistenceDeletePlanned
↓
PersistenceDeleteExecuted
↓
EntityPostRemove
↓
TransactionCommitted
↓
EntityDeleteCommitted
```

---

# 151. PostRemove state

Después de delete execution, la entidad puede pasar a:

```text
DETACHED
```

o permanecer en estado específico hasta completion según policy.

Esto deberá ser explícito.

---

# 152. Transaction rollback after delete execution

La fila puede volver a existir en DB tras rollback.

La entidad PHP puede seguir DETACHED/REMOVED.

Por tanto requiere reconciliation.

---

# 153. Lifecycle event after rollback

No deberá fingir:

```text
EntityRestored
```

si el ORM no ha reconciliado realmente el objeto.

---

# 154. EntityManager taint

Un UNKNOWN persistence/transaction outcome puede llevar a:

```text
EntityManager = TAINTED
```

---

# 155. EntityManagerTainted event

Podrá existir como evento ORM de alto nivel.

---

# 156. Reason

```php
enum EntityManagerTaintReason
{
    case UNKNOWN_TRANSACTION_OUTCOME;
    case PERSISTENCE_UNCERTAINTY;
    case INTERNAL_INCONSISTENCY;
    case OTHER;
}
```

---

# 157. EntityManager taint ≠ Entity dirty

Son conceptos distintos.

---

# 158. clear after taint

La policy podrá requerir:

```text
clear / close EntityManager
```

---

# 159. EntityManagerClosed event

Podrá indicar cierre del scope ORM.

---

# 160. Close ≠ clear

```text
clear()
=
forget managed entities but manager may remain usable

close()
=
manager no longer usable
```

---

# 161. EntityManager lifecycle events

Podrán existir:

```text
EntityManagerOpened
EntityManagerCleared
EntityManagerTainted
EntityManagerClosed
```

aunque este documento se centra en entidades.

---

# 162. Event ordering

Para hydration:

```text
EntityHydrationStarted
↓
EntityHydrated
↓
EntityManaged
↓
EntityPostLoad
```

según boundary final elegido.

---

# 163. Event ordering for persist

```text
EntityPersistScheduled
↓
EntityPrePersist
↓
Persistence...
↓
EntityPostPersist
```

---

# 164. Event ordering for update

```text
EntityDirtyDetected
↓
EntityPreUpdate
↓
Persistence...
↓
EntityPostUpdate
```

---

# 165. Event ordering for remove

```text
EntityRemoveScheduled
↓
EntityPreRemove
↓
Persistence...
↓
EntityPostRemove
```

---

# 166. Listener failure semantics

Un observer failure después de:

```text
EntityPostPersist
```

no deberá convertir:

```text
INSERT success
```

en:

```text
INSERT failure
```

---

# 167. Hook failure semantics

Un mutating/veto hook ejecutado antes del persistence boundary sí puede impedir continuar.

---

# 168. Observer ≠ hook failure

Debe registrarse separadamente.

---

# 169. LifecycleHookException

```text
EntityLifecycleHookException
```

deberá ser distinta de:

```text
EntityLifecycleListenerException
```

---

# 170. Post hook failure

Si un post-persist hook falla dentro de una transacción todavía activa, la policy puede provocar rollback.

Pero eso no significa que el INSERT statement no se ejecutó.

---

# 171. Precise diagnostics

Debe poder representarse:

```text
INSERT executed
↓
post-persist hook failed
↓
transaction rolled back
```

---

# 172. Externalization

La mayoría de Entity Lifecycle Events serán:

```text
IN_PROCESS
```

---

# 173. Why

Porque pueden ser:

```text
high volume
scope-dependent
mutable-runtime-sensitive
ORM-internal
```

---

# 174. Commit-aware external events

Si una aplicación quiere enviar:

```text
UserCreated
```

deberá preferir:

```text
Domain Event
+
Outbox
```

no externalizar ciegamente:

```text
EntityPostPersist
```

---

# 175. Entity lifecycle ≠ integration event

Regla clave.

---

# 176. Event payload security

No incluir por default:

```text
full entity
all field values
passwords
tokens
PII
relationships
lazy proxies
```

---

# 177. Field metadata exposure

Podrá incluir:

```text
changed field names
```

bajo policy segura.

---

# 178. EntityType security

Incluso el nombre de entidad puede ser sensible en ciertos entornos.

Podrá usarse:

```text
stable EntityTypeId
```

en lugar de FQCN.

---

# 179. Stable EntityTypeId

No persistir externamente FQCN como identidad principal.

---

# 180. Event cardinality

No usar:

```text
entity identifier
tenant ID
runtime ID
```

como metric labels automáticos.

---

# 181. Telemetry bridge

```text
Entity Lifecycle Event
↓
ORM Telemetry Bridge
↓
metrics/spans/logs
```

---

# 182. ORM telemetry document

Se profundizará en:

```text
220_DATABASE_ORM_TELEMETRY_SYSTEM.md
```

---

# 183. Lifecycle metrics

Podrán derivarse:

```text
entities hydrated
entities managed
entities dirty
entities inserted
entities updated
entities removed
entity manager clears
```

sin exponer entidades individuales.

---

# 184. Hydration telemetry

No deberá emitir un span por entidad por default en grandes datasets.

---

# 185. Sampling/aggregation

Podrá agregarse por:

```text
EntityTypeId
operation
origin
```

si cardinalidad es segura.

---

# 186. Event storm protection

Procesar 10M entidades no debe implicar 10M external messages.

---

# 187. Debug mode

Per-entity event recording podrá habilitarse para pruebas y desarrollo.

---

# 188. Persistent runtime safety

Crítico para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 189. Scope-local state

Debe permanecer local:

```text
EntityManagerScopeId
IdentityMap
UnitOfWork
EntityRuntimeIds
lifecycle hook stack
dispatch recursion
temporary listeners
transaction correlation
tenant context
```

---

# 190. No static current entity

Prohibido:

```php
EntityLifecycle::$currentEntity;
```

---

# 191. No static IdentityMap via event system

El Event System no deberá convertirse en acceso global al ORM state.

---

# 192. Request cleanup

Al terminar:

```text
IdentityMap cleared/manager closed
UnitOfWork cleared
hooks scope released
event references released
```

---

# 193. Memory retention risk

Listeners que guarden referencias a entidades pueden impedir garbage collection.

---

# 194. Listener retention policy

Deberá documentarse:

> **Los listeners no deberán retener referencias a managed entities más allá del scope salvo que entiendan las consecuencias.**

---

# 195. Weak references

En diagnostics internos podrían usarse `WeakReference` cuando sea apropiado.

---

# 196. Async listener rule

Nunca enviar live entities a listeners async.

---

# 197. Async-safe representation

Usar:

```text
EntityTypeId
CanonicalEntityIdentifier
event kind
committed outcome if known
bounded metadata
```

---

# 198. NEW entity without ID

No es adecuada para externalización durable antes de identifier assignment/commit.

---

# 199. Lifecycle hook registry

```php
interface EntityLifecycleHookRegistry
{
    public function hooksFor(
        EntityType $type,
        EntityLifecycleHookPoint $point
    ): iterable;
}
```

---

# 200. Hook points

```php
enum EntityLifecycleHookPoint
{
    case POST_LOAD;
    case PRE_PERSIST;
    case POST_PERSIST;
    case PRE_UPDATE;
    case POST_UPDATE;
    case PRE_REMOVE;
    case POST_REMOVE;
}
```

---

# 201. Hook registry compilation

Podrá compilarse desde:

```text
attributes
metadata
configuration
packages
```

---

# 202. Hook ordering

Deberá ser determinista.

---

# 203. Entity callback methods

VoltStack podrá soportar:

```php
#[PrePersist]
public function initializeCreatedAt(): void
{
    // ...
}
```

si se desea DX similar a ORM maduros.

---

# 204. Callback method validation

Bootstrap deberá validar:

```text
method visibility
signature
supported lifecycle point
return semantics
```

---

# 205. Static callbacks

No se recomendarán si introducen hidden global state.

---

# 206. Constructor hooks

No deberán confundirse con lifecycle events.

---

# 207. Domain invariants

Idealmente deben vivir en:

```text
entity/domain methods
```

y no depender únicamente de ORM hooks.

---

# 208. Lifecycle hooks use cases

Válidos:

```text
timestamps
derived persistence fields
technical metadata
audit timestamps
normalization
```

con cuidado.

---

# 209. Dangerous use cases

No recomendado:

```text
send emails
remote API calls
billing charges
message publishing
```

dentro de pre/post persistence hooks.

---

# 210. Why

Porque:

```text
transaction may rollback
retry may replay hooks
listener may execute multiple times
```

---

# 211. External side effects

Deben preferirse:

```text
Domain Event
↓
Outbox
↓
After Commit delivery
```

---

# 212. Retry interaction

Un transaction retry puede repetir:

```text
prePersist
preUpdate
persistence
postPersist
postUpdate
```

según cómo se reconstruya el attempt.

---

# 213. Hook replayability

Hooks deberán clasificarse conceptualmente:

```text
PURE
IDEMPOTENT
SIDE_EFFECTING
UNKNOWN
```

---

# 214. Retry safety diagnostics

VoltStack podrá advertir:

```text
Side-effecting lifecycle hook inside retryable transaction.
```

---

# 215. Entity factory integration

Factory callbacks:

```text
afterMaking
afterCreating
```

no deberán confundirse con ORM lifecycle hooks.

---

# 216. Factory callback ≠ Entity lifecycle event

Aunque puedan coincidir temporalmente.

---

# 217. Seeder integration

Seeder puede crear entidades y provocar lifecycle events si usa ORM.

---

# 218. Fixture integration

Igual.

---

# 219. Test data generator

Por sí solo no produce lifecycle events.

---

# 220. Lazy Collection integration

Lazy hydration de entidades podrá producir lifecycle events por cada nueva instancia hydrated.

---

# 221. Chunk processing

Entity events pueden ser high-volume.

El telemetry bridge deberá agregar.

---

# 222. Large Dataset processing

Para millones de filas se recomendará:

```text
PROJECTION_ONLY
```

cuando lifecycle no sea necesario.

---

# 223. Entity lifecycle cost

Debe incluirse en explain/diagnostics cuando se usa entity processing masivo.

---

# 224. Diagnostics API

Conceptualmente:

```php
DB::orm()->events()->explain(User::class);
```

---

# 225. Explain output

```text
ENTITY LIFECYCLE EVENT PLAN

Entity:
  user

Hooks:
  POST_LOAD:
    NormalizeUserHook

  PRE_PERSIST:
    TimestampHook

  PRE_UPDATE:
    TimestampHook

Observers:
  EntityDirtyDetected:
    ORMProfilerListener

Commit-aware:
  EntityCreatedCommitted:
    enabled

Transaction:
  after-commit integration enabled

Bulk Operations:
  bypass per-entity lifecycle

Externalization:
  disabled

Security:
  field values: REDACTED

Retry Warning:
  no side-effecting hooks detected
```

---

# 226. Lifecycle validation

En development/testing podrá verificarse que:

```text
NEW
→ MANAGED
→ DIRTY
→ MANAGED
→ REMOVED
→ DETACHED
```

solo ocurra mediante transiciones válidas.

---

# 227. Invalid transitions

Ejemplos:

```text
DETACHED → DIRTY without reattach
REMOVED → MANAGED without reconciliation
UNKNOWN → MANAGED silently
```

---

# 228. EntityLifecycleValidator

```php
final class EntityLifecycleValidator
{
    public function observe(
        EntityLifecycleEvent $event
    ): void {
        // development/testing checks
    }
}
```

---

# 229. Lifecycle state authority

La autoridad seguirá siendo:

```text
EntityStateSystem
UnitOfWork
EntityManager
```

no el validator.

---

# 230. Testing support

Debe permitir:

```php
DatabaseEvents::assertDispatched(
    EntityPersistScheduled::class
);
```

---

# 231. Hydration sequence test

```text
EntityHydrationStarted
EntityHydrated
EntityManaged
EntityPostLoad
```

---

# 232. Persist test

```text
EntityPersistScheduled
EntityPrePersist
PersistenceInsertExecuted
EntityPostPersist
```

sin asumir commit.

---

# 233. Commit-aware test

```text
EntityPostPersist
↓
assert EntityCreatedCommitted NOT dispatched
↓
TransactionCommitted
↓
assert EntityCreatedCommitted dispatched
```

---

# 234. Rollback test

```text
EntityPostPersist
↓
TransactionRolledBack
↓
EntityCreatedCommitted MUST NOT dispatch
```

---

# 235. UNKNOWN test

```text
EntityPostPersist
↓
TransactionOutcomeUnknown
↓
EntityCreatedCommitted MUST NOT dispatch
```

---

# 236. Bulk test

```text
BulkUpdate
```

no deberá producir miles de:

```text
EntityPreUpdate
EntityPostUpdate
```

---

# 237. EntityManager clear test

Por default:

```text
EntityManagerCleared
```

en lugar de per-entity detach storm.

---

# 238. Persistent runtime test

Request A y Request B no deberán compartir:

```text
EntityRuntimeId
hook state
IdentityMap
lifecycle context
```

---

# 239. Error hierarchy

```text
EntityLifecycleEventException
├── EntityLifecycleEventDispatchException
├── EntityLifecycleStateException
├── EntityLifecycleHookException
├── EntityLifecycleListenerException
├── EntityLifecycleVetoException
├── EntityLifecycleMutationException
├── EntityLifecycleCorrelationException
├── EntityLifecycleSecurityException
├── EntityLifecycleRetrySafetyException
└── EntityLifecycleCompatibilityException
```

---

# 240. Lifecycle hook failure

Debe distinguirse entre:

```text
PRE hook failed before persistence
POST hook failed after statement
observer failed
```

---

# 241. Error phase

```php
enum EntityLifecycleFailurePhase
{
    case HYDRATION;
    case MANAGED_REGISTRATION;
    case PRE_PERSIST;
    case POST_PERSIST;
    case PRE_UPDATE;
    case POST_UPDATE;
    case PRE_REMOVE;
    case POST_REMOVE;
    case DETACH;
    case REFRESH;
    case COMMIT_AWARE_DELIVERY;
}
```

---

# 242. Directory structure

```text
src/Quantum/Database/Event/Entity/
│
├── Contract/
│   ├── EntityLifecycleEvent.php
│   ├── LocalEntityLifecycleEvent.php
│   ├── EntityLifecycleHook.php
│   └── EntityLifecycleHookRegistry.php
│
├── Event/
│   ├── EntityHydrationStarted.php
│   ├── EntityHydrated.php
│   ├── EntityHydrationFailed.php
│   ├── EntityManaged.php
│   ├── EntityPostLoad.php
│   ├── EntityPersistScheduled.php
│   ├── EntityDirtyDetected.php
│   ├── EntityRemoveScheduled.php
│   ├── EntityIdentifierAssigned.php
│   ├── EntityPrePersist.php
│   ├── EntityPostPersist.php
│   ├── EntityPreUpdate.php
│   ├── EntityPostUpdate.php
│   ├── EntityPreRemove.php
│   ├── EntityPostRemove.php
│   ├── EntityDetached.php
│   ├── EntityRefreshStarted.php
│   ├── EntityRefreshed.php
│   ├── EntityRefreshFailed.php
│   ├── EntityCreatedCommitted.php
│   ├── EntityUpdatedCommitted.php
│   ├── EntityDeletedCommitted.php
│   ├── EntityManagerCleared.php
│   ├── EntityManagerTainted.php
│   └── EntityManagerClosed.php
│
├── Model/
│   ├── EntityRuntimeId.php
│   ├── EntityIdentityReference.php
│   ├── EntityLifecycleEventContext.php
│   ├── EntityLifecycleMetadata.php
│   ├── EntityStateTransition.php
│   ├── EntityStateTransitionReason.php
│   ├── EntitySchedulingReason.php
│   ├── EntityHydrationSource.php
│   ├── EntityLifecycleOrigin.php
│   ├── EntityLifecycleHookPoint.php
│   ├── EntityLifecycleHookCapability.php
│   └── EntityLifecycleFailurePhase.php
│
├── Summary/
│   ├── EntityChangeSummary.php
│   ├── LoadedFieldMaskSummary.php
│   └── EntityManagerClearSummary.php
│
├── Hook/
│   ├── CompiledEntityLifecycleHookRegistry.php
│   ├── EntityLifecycleHookInvoker.php
│   ├── EntityLifecycleHookContext.php
│   ├── LifecycleDecision.php
│   └── LifecycleHookPolicy.php
│
├── Transaction/
│   ├── CommitAwareEntityEventCoordinator.php
│   ├── EntityCommitEventRegistration.php
│   └── EntityRollbackEventRegistration.php
│
├── Security/
│   ├── EntityLifecycleEventSanitizer.php
│   ├── EntityLifecycleExposurePolicy.php
│   └── EntityFieldExposurePolicy.php
│
├── Diagnostics/
│   ├── EntityLifecycleInspector.php
│   ├── EntityLifecycleValidator.php
│   └── EntityLifecycleExplain.php
│
├── Testing/
│   ├── EntityLifecycleEventRecorder.php
│   ├── EntityLifecycleEventAssertions.php
│   └── FakeEntityLifecycleHook.php
│
└── Exception/
    ├── EntityLifecycleEventException.php
    ├── EntityLifecycleEventDispatchException.php
    ├── EntityLifecycleStateException.php
    ├── EntityLifecycleHookException.php
    ├── EntityLifecycleListenerException.php
    ├── EntityLifecycleVetoException.php
    ├── EntityLifecycleMutationException.php
    ├── EntityLifecycleCorrelationException.php
    ├── EntityLifecycleSecurityException.php
    ├── EntityLifecycleRetrySafetyException.php
    └── EntityLifecycleCompatibilityException.php
```

---

# 243. Integración con Hydration

```text
Execution Result
↓
Hydration Plan
↓
EntityHydrationStarted
↓
IdentityMap Reservation
↓
Field Assignment
↓
Snapshot
↓
Managed Registration
↓
EntityHydrated
↓
EntityManaged
↓
EntityPostLoad
```

---

# 244. Integración con UnitOfWork

```text
Entity Mutation
↓
Change Tracking
↓
EntityDirtyDetected
↓
ChangeSet
↓
Persistence Planner
```

---

# 245. Integración con Persist

```text
persist(entity)
↓
EntityPersistScheduled
↓
UnitOfWork
↓
Flush
↓
PrePersist
↓
Persistence Engine
↓
PostPersist
```

---

# 246. Integración con Remove

```text
remove(entity)
↓
EntityRemoveScheduled
↓
UnitOfWork
↓
Flush
↓
PreRemove
↓
DELETE
↓
PostRemove
```

---

# 247. Integración transaction-aware

```text
EntityPostPersist
↓
queue EntityCreatedCommitted
↓
TransactionCommitStarted
↓
COMMIT
├── COMMITTED
│      ↓
│   EntityCreatedCommitted
│
├── ROLLED_BACK
│      ↓
│   discard
│
└── UNKNOWN
       ↓
    discard committed notification
```

---

# 248. Architectural invariants

## DB-ELIFE-001
Entity Lifecycle Event será distinto de Domain Event.

## DB-ELIFE-002
Entity Lifecycle Event será distinto de Persistence Event.

## DB-ELIFE-003
Entity Lifecycle Event será distinto de Query Event.

## DB-ELIFE-004
Entity Lifecycle Event será distinto de Transaction Event.

## DB-ELIFE-005
Entity Lifecycle Event será distinto de Entity State.

## DB-ELIFE-006
Entity State será distinto de Database Row State.

## DB-ELIFE-007
Entity State será distinto de Database Commit State.

## DB-ELIFE-008
EntityRuntimeId será distinto de Database Identifier.

## DB-ELIFE-009
Entity event payload no contendrá live entity por default.

## DB-ELIFE-010
Async lifecycle events nunca transportarán managed entity objects.

## DB-ELIFE-011
EntityHydrated será distinto de DB Query Executed.

## DB-ELIFE-012
EntityHydrated será distinto de EntityManaged.

## DB-ELIFE-013
EntityManaged será distinto de EntityInserted.

## DB-ELIFE-014
IdentityMap será authority de canonical managed identity.

## DB-ELIFE-015
persist() significará scheduling/registering, no INSERT inmediato.

## DB-ELIFE-016
EntityPersistScheduled será distinto de Flush.

## DB-ELIFE-017
EntityPersistScheduled será distinto de InsertPlanned.

## DB-ELIFE-018
EntityPersistScheduled será distinto de InsertExecuted.

## DB-ELIFE-019
remove() significará scheduling, no DELETE inmediato.

## DB-ELIFE-020
EntityRemoveScheduled será distinto de DeleteExecuted.

## DB-ELIFE-021
DirtyDetected será distinto de UpdateRequired.

## DB-ELIFE-022
ChangeSet será distinto de SQL.

## DB-ELIFE-023
Sensitive change values serán redacted.

## DB-ELIFE-024
Generic observers serán distintos de mutating lifecycle hooks.

## DB-ELIFE-025
Pre lifecycle mutation utilizará hook contract explícito.

## DB-ELIFE-026
Hook capability será explícita.

## DB-ELIFE-027
Veto será distinto de listener exception.

## DB-ELIFE-028
Hook ordering será determinista.

## DB-ELIFE-029
Hook mutation rounds serán bounded.

## DB-ELIFE-030
PostPersist será distinto de TransactionCommitted.

## DB-ELIFE-031
PostUpdate será distinto de TransactionCommitted.

## DB-ELIFE-032
PostRemove será distinto de TransactionCommitted.

## DB-ELIFE-033
Committed entity events serán after-commit only.

## DB-ELIFE-034
Rollback impedirá committed entity notifications.

## DB-ELIFE-035
UNKNOWN impedirá committed entity notifications.

## DB-ELIFE-036
Database rollback será distinto de object graph rewind.

## DB-ELIFE-037
Rollback event no implicará entity restoration.

## DB-ELIFE-038
Detach será distinto de Delete.

## DB-ELIFE-039
EntityManager clear no implicará entity delete.

## DB-ELIFE-040
EntityManager clear no emitirá per-entity storm por default.

## DB-ELIFE-041
Refresh será distinto de merge.

## DB-ELIFE-042
Merge no deberá crear duplicate managed identity.

## DB-ELIFE-043
Partial entity será distinta de complete entity.

## DB-ELIFE-044
LoadedFieldMask será preservado en context.

## DB-ELIFE-045
EntityManagerScopeId será distinto de RequestId.

## DB-ELIFE-046
EntityManagerScopeId será distinto de TransactionId.

## DB-ELIFE-047
Generated identifier assignment será distinto de commit.

## DB-ELIFE-048
Generated ID rollback semantics serán handled por consistency policy.

## DB-ELIFE-049
Managed identifier mutation será restringida.

## DB-ELIFE-050
Optimistic version update será distinto de commit.

## DB-ELIFE-051
Relationship lifecycle será distinto de entity lifecycle.

## DB-ELIFE-052
Cascade persist deberá indicar scheduling reason.

## DB-ELIFE-053
Orphan removal deberá indicar scheduling reason.

## DB-ELIFE-054
Large cascade graphs estarán sujetos a volume governance.

## DB-ELIFE-055
Constructor invocation será distinto de ORM lifecycle.

## DB-ELIFE-056
Model API y Repository usarán el mismo lifecycle engine.

## DB-ELIFE-057
Active Record facade no creará segundo lifecycle system.

## DB-ELIFE-058
Hydration assignment no marcará DIRTY.

## DB-ELIFE-059
Baseline snapshot se establecerá antes de PostLoad según policy.

## DB-ELIFE-060
PostLoad mutation podrá volver DIRTY a la entidad.

## DB-ELIFE-061
Lazy relationship load no redisparará PostLoad de root entity.

## DB-ELIFE-062
New related entity hydration sí tendrá su lifecycle.

## DB-ELIFE-063
Entity cache hydration será distinguible por source.

## DB-ELIFE-064
Hydration source será distinto de managed state.

## DB-ELIFE-065
ROW_BULK import no fingirá entity lifecycle.

## DB-ELIFE-066
ORM_ENTITY import sí podrá generar lifecycle events.

## DB-ELIFE-067
Bulk Update no generará per-entity lifecycle por default.

## DB-ELIFE-068
Bulk Delete no generará per-entity lifecycle por default.

## DB-ELIFE-069
Set-based mutation será distinta de ORM entity processing.

## DB-ELIFE-070
EntityPostPersist podrá preceder transaction rollback.

## DB-ELIFE-071
EntityPostUpdate podrá preceder transaction rollback.

## DB-ELIFE-072
EntityPostRemove podrá preceder transaction rollback.

## DB-ELIFE-073
EntityManagerTainted será distinto de EntityDirty.

## DB-ELIFE-074
UNKNOWN persistence outcome podrá taint EntityManager.

## DB-ELIFE-075
EntityManager clear será distinto de close.

## DB-ELIFE-076
Lifecycle event ordering será definido.

## DB-ELIFE-077
Observer failure no reescribirá persistence outcome.

## DB-ELIFE-078
Pre-hook failure podrá impedir persistence antes del boundary.

## DB-ELIFE-079
Post-hook failure podrá provocar rollback si transaction sigue activa.

## DB-ELIFE-080
Post-hook failure no significará que statement no ejecutó.

## DB-ELIFE-081
Lifecycle hooks serán in-process por default.

## DB-ELIFE-082
Entity lifecycle no será integration-event system.

## DB-ELIFE-083
Domain event + outbox será preferible para external side effects.

## DB-ELIFE-084
Entity payload no expondrá secrets.

## DB-ELIFE-085
EntityType external identity no dependerá de FQCN.

## DB-ELIFE-086
Entity IDs no serán metric labels automáticos.

## DB-ELIFE-087
Telemetry podrá agregar lifecycle events.

## DB-ELIFE-088
Per-entity telemetry no será default para large datasets.

## DB-ELIFE-089
Event storm protection será soportada.

## DB-ELIFE-090
Entity lifecycle mutable state será scope-local.

## DB-ELIFE-091
No existirá static current entity.

## DB-ELIFE-092
Event System no será acceso global al IdentityMap.

## DB-ELIFE-093
Listeners no deberán retener managed entities más allá del scope por default.

## DB-ELIFE-094
Async listeners usarán stable entity references.

## DB-ELIFE-095
NEW entity sin ID no será externalizable como committed entity.

## DB-ELIFE-096
Hook registry será compilable.

## DB-ELIFE-097
Hook callback signatures serán validadas.

## DB-ELIFE-098
Domain invariants no deberán depender solo de ORM hooks.

## DB-ELIFE-099
Lifecycle hooks no deberán realizar external side effects por default.

## DB-ELIFE-100
Retry podrá reejecutar hooks.

## DB-ELIFE-101
Side-effecting hooks en retryable transaction serán diagnosticables.

## DB-ELIFE-102
Factory callbacks serán distintos de lifecycle hooks.

## DB-ELIFE-103
Seeder lifecycle dependerá de si usa ORM.

## DB-ELIFE-104
Fixture lifecycle dependerá de si usa ORM.

## DB-ELIFE-105
Test Data Generator por sí solo no producirá lifecycle events.

## DB-ELIFE-106
Lazy Collection puede generar lifecycle events por hydration.

## DB-ELIFE-107
Chunk entity processing puede ser high-volume.

## DB-ELIFE-108
LargeData PROJECTION_ONLY evitará entity lifecycle.

## DB-ELIFE-109
Lifecycle overhead será diagnosticable.

## DB-ELIFE-110
Lifecycle validator no será state authority.

## DB-ELIFE-111
Invalid transitions serán detectables en dev/testing.

## DB-ELIFE-112
Commit-aware events no se emitirán antes de commit.

## DB-ELIFE-113
Commit-aware events no se emitirán tras rollback.

## DB-ELIFE-114
Commit-aware events no se emitirán tras UNKNOWN.

## DB-ELIFE-115
Bulk mutation bypass de lifecycle será explícito.

## DB-ELIFE-116
Persistence Events serán authority para physical persistence stages.

## DB-ELIFE-117
Transaction Events serán authority para transaction outcome events.

## DB-ELIFE-118
Query Events serán authority para query execution observation.

## DB-ELIFE-119
Entity State System será authority para ORM state.

## DB-ELIFE-120
IdentityMap será authority para managed identity.

## DB-ELIFE-121
UnitOfWork será authority para pending ORM work.

## DB-ELIFE-122
EntityManager será coordinator, no event bus.

## DB-ELIFE-123
Lifecycle Events no reemplazarán UnitOfWork.

## DB-ELIFE-124
Lifecycle Events no reemplazarán Change Tracking.

## DB-ELIFE-125
Lifecycle Events no reemplazarán Hydrator.

## DB-ELIFE-126
Lifecycle Events no reemplazarán Persistence Engine.

## DB-ELIFE-127
Lifecycle Events no reemplazarán Transaction Manager.

## DB-ELIFE-128
Lifecycle Events no reemplazarán Domain Events.

## DB-ELIFE-129
Lifecycle Events no garantizarán durability.

## DB-ELIFE-130
Event delivery failure no cambiará entity/database reality.

## DB-ELIFE-131
Entity persistence reality será distinta de event delivery reality.

## DB-ELIFE-132
Persistent runtime cleanup será obligatorio.

## DB-ELIFE-133
FrankenPHP no heredará lifecycle state entre requests.

## DB-ELIFE-134
RoadRunner no heredará lifecycle state entre jobs.

## DB-ELIFE-135
OpenSwoole lifecycle state será coroutine-safe.

## DB-ELIFE-136
Entity event contexts no sobrevivirán accidentalmente al scope.

## DB-ELIFE-137
EntityManager close limpiará event-scoped references.

## DB-ELIFE-138
Hook invocation stack será scope-local.

## DB-ELIFE-139
Lifecycle recursion será gobernada.

## DB-ELIFE-140
Hook-induced mutation loops serán bounded.

## DB-ELIFE-141
EntityPostLoad observer será distinto de PostLoad hook.

## DB-ELIFE-142
Observer registration será distinta de hook registration.

## DB-ELIFE-143
Commit-aware entity events dependerán del Transaction Event Pipeline.

## DB-ELIFE-144
Commit-aware delivery failure no deshará commit.

## DB-ELIFE-145
After-commit entity event será distinto de durable domain event.

## DB-ELIFE-146
Unknown ORM state permanecerá UNKNOWN cuando no pueda reconciliarse.

## DB-ELIFE-147
No se inventará MANAGED/DETACHED state después de uncertainty.

## DB-ELIFE-148
Security tendrá prioridad sobre lifecycle payload richness.

## DB-ELIFE-149
Correctness tendrá prioridad sobre hook convenience.

## DB-ELIFE-150
Lifecycle Event System preservará fronteras del ORM.

---

# 249. Modelo formal

Sea una entidad:

```text
e
```

con estado ORM:

```text
S(e,t)
```

Un lifecycle event:

```text
E_life(e)
```

describe:

```text
S(e,t1) → S(e,t2)
```

o un boundary asociado.

No implica:

```text
DBCommitted(e)
```

---

# 250. Persist formalization

Sea:

```text
persist(e)
```

Entonces:

```text
persist(e)
⇒
Scheduled(e)
```

pero:

```text
Scheduled(e) ↛ InsertExecuted(e)
```

y:

```text
InsertExecuted(e) ↛ TransactionCommitted
```

---

# 251. Remove formalization

```text
remove(e)
⇒
RemoveScheduled(e)
```

pero:

```text
RemoveScheduled(e) ↛ DeleteExecuted(e)
```

y:

```text
DeleteExecuted(e) ↛ TransactionCommitted
```

---

# 252. Commit-aware event formalization

Sea:

```text
C(e)
```

un evento de entidad que debe representar durabilidad.

Entonces:

```text
Emit(C(e))
```

solo si:

```text
PersistenceExecuted(e)
∧
TransactionOutcome = COMMITTED
```

---

# 253. Rollback relation

Si:

```text
PersistenceExecuted(e)
```

pero:

```text
TransactionOutcome = ROLLED_BACK
```

entonces:

```text
Emit(EntityCommittedEvent(e)) = false
```

---

# 254. UNKNOWN relation

Si:

```text
TransactionOutcome = UNKNOWN
```

también:

```text
Emit(EntityCommittedEvent(e)) = false
```

por default.

---

# 255. Identity relation

Para entities managed:

```text
EntityType(e1) = EntityType(e2)
∧
Identifier(e1) = Identifier(e2)
∧
Context(e1) = Context(e2)
```

implica:

```text
e1 === e2
```

dentro del mismo IdentityMap scope.

Los lifecycle events deberán respetar esta identidad.

---

# 256. Arquitectura final

```text
                      Entity Manager
                           │
                           ▼
                       UnitOfWork
                           │
       ┌───────────────────┼───────────────────┐
       ▼                   ▼                   ▼
   Hydration            Scheduling         Change Tracking
       │                   │                   │
       ▼                   ▼                   ▼
 EntityHydrated   Persist/Remove Events   DirtyDetected
       │                   │                   │
       └───────────────────┼───────────────────┘
                           ▼
                    Lifecycle Hooks
                           │
                           ▼
                   Persistence Engine
                           │
                           ▼
                  Persistence Events
                           │
                           ▼
                      Query Engine
                           │
                           ▼
                  Transaction Pipeline
                           │
                    ┌──────┴──────┐
                    ▼             ▼
                 COMMIT         ROLLBACK
                    │
                    ▼
         Commit-aware Entity Events
```

---

# 257. Regla maestra final

VoltStack deberá preservar siempre:

```text
Entity Lifecycle Event
≠
Domain Event
```

```text
Entity Lifecycle Event
≠
Persistence Event
```

```text
Entity Lifecycle Event
≠
Query Event
```

```text
Entity Lifecycle Event
≠
Transaction Event
```

```text
Entity State
≠
Database State
```

```text
persist()
≠
INSERT
```

```text
remove()
≠
DELETE
```

```text
Dirty
≠
UPDATE executed
```

```text
PostPersist
≠
Committed
```

```text
PostUpdate
≠
Committed
```

```text
PostRemove
≠
Committed
```

```text
Generated ID Assigned
≠
Committed
```

```text
Rollback
≠
Object Graph Rewind
```

```text
Detached
≠
Deleted
```

```text
EntityManager Clear
≠
Database Delete
```

```text
Bulk Mutation
≠
Per-Entity Lifecycle
```

```text
ORM Lifecycle Hook
≠
Generic Observer
```

```text
Entity Committed Event
≠
Durable Domain Event
```

y especialmente:

```text
ORM Knowledge
≠
Database Durability
```

---

# 258. Resultado arquitectónico

Con `Database Entity Lifecycle Event System`, VoltStack podrá soportar un ORM con lifecycle rico y extensible:

```text
hydration hooks
post-load processing
pre-persist technical logic
pre-update normalization
remove hooks
state observation
commit-aware entity notifications
debugging
testing
telemetry
```

sin introducir la simplificación peligrosa:

```text
Entity event fired
=
database change committed
```

La arquitectura final mantendrá:

```text
EntityManager
=
coordination authority

UnitOfWork
=
pending state authority

IdentityMap
=
managed identity authority

Persistence Engine
=
database synchronization authority

Transaction Manager
=
commit/rollback authority

Lifecycle Event System
=
typed observation + controlled ORM hooks
```

---

# 259. Bloque 20 — Estado

```text
BLOCK 20 — EVENTS

✓ 209_DATABASE_EVENT_ARCHITECTURE.md
✓ 210_DATABASE_QUERY_EVENT_SYSTEM.md
✓ 211_DATABASE_CONNECTION_EVENT_SYSTEM.md
✓ 212_DATABASE_TRANSACTION_EVENT_PIPELINE.md
✓ 213_DATABASE_ENTITY_LIFECYCLE_EVENT_SYSTEM.md
○ 214_DATABASE_PERSISTENCE_EVENT_SYSTEM.md
○ 215_DATABASE_EVENT_EXTENSION_SYSTEM.md
```

---

# 260. Siguiente documento

```text
214_DATABASE_PERSISTENCE_EVENT_SYSTEM.md
```

El siguiente documento definirá eventos alrededor del pipeline de persistencia:

```text
UnitOfWork
↓
ChangeSets
↓
Persistence Planning
↓
Dependency Ordering
↓
Insert / Update / Delete Operations
↓
Flush
↓
Query Execution
↓
Transaction Coordination
↓
Consistency Reconciliation
```

incluyendo:

```text
PersistenceFlushStarted
PersistencePlanCreated
PersistenceOperationScheduled
PersistenceInsertStarted
PersistenceInsertExecuted
PersistenceUpdateStarted
PersistenceUpdateExecuted
PersistenceDeleteStarted
PersistenceDeleteExecuted
PersistenceFlushCompleted
PersistenceFlushFailed
PersistenceOutcomeUnknown
```

bajo una distinción central:

> **Persistence Event ≠ Entity Lifecycle Event ≠ Query Event ≠ Transaction Event; un Persistence Event describe el intento de sincronización del estado ORM con la base, pero no debe afirmar commit durability hasta que el Transaction System pueda demostrarlo.**