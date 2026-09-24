# 122_DATABASE_ENTITY_LIFECYCLE_SYSTEM.md

# VoltStack Quantum Database
## Database Entity Lifecycle System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 122 — Database Entity Lifecycle System  
**Bloque:** 10 — ORM  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Entity Lifecycle System` define cómo VoltStack representa, ordena, ejecuta y gobierna los puntos de extensión asociados al ciclo de vida ORM de una entidad.

El sistema permitirá observar y reaccionar ante fases como:

```text
load
persist
update
remove
refresh
flush
detach
```

sin confundirlas con:

```text
Domain Events
Database Transactions
Entity State
Change Tracking
Persistence Planning
SQL Execution
Framework Events
```

Principio central:

> **El Entity Lifecycle System observa y coordina fases explícitas del ciclo de persistencia de una entidad, pero nunca convierte una fase ORM en evidencia de que la transacción correspondiente ya fue confirmada por la base de datos.**

---

# 2. Problema

Un ORM necesita ejecutar comportamiento alrededor de operaciones como:

```php
$entityManager->persist($user);
$entityManager->flush();
$entityManager->remove($user);
```

Ejemplos legítimos:

```text
set createdAt
set updatedAt
normalize persistent state
validate an invariant
record telemetry
initialize derived values
collect domain events
```

Pero una implementación ingenua puede introducir:

```text
recursive flush
hidden queries
hidden transactions
incorrect event ordering
postPersist before commit confusion
side effects impossible to rollback
persistent worker leaks
```

Por ello lifecycle deberá ser una arquitectura formal.

---

# 3. Lifecycle Callback ≠ Domain Event

Esta separación será fundamental.

Un callback:

```text
prePersist
```

significa:

> El ORM está entrando en una determinada fase de persistencia.

Un evento:

```text
OrderPlaced
```

significa:

> Algo relevante ocurrió dentro del dominio.

Por tanto:

```text
Lifecycle Callback
≠
Domain Event
```

---

# 4. Lifecycle Event ≠ Framework Event

Aunque VoltStack pueda integrar ambos:

```text
ORM Lifecycle Event
```

pertenece al Database ORM.

Mientras:

```text
VoltStack EventSystem Event
```

pertenece al sistema general de eventos.

La integración será mediante adaptadores.

---

# 5. Lifecycle ≠ Entity State

El documento anterior definió:

```text
NEW
MANAGED
REMOVED
DETACHED
UNKNOWN
```

Lifecycle describe:

```text
qué fase está atravesando una operación
```

Entity State describe:

```text
qué relación mantiene la entidad con PersistenceContext
```

---

# 6. Lifecycle ≠ Persistence Operation

Ejemplo:

```text
preUpdate
```

no es:

```text
UPDATE
```

Es un punto lógico anterior a una operación de persistencia.

---

# 7. Lifecycle ≠ SQL hook

Nunca deberá modelarse como:

```text
before SQL
after SQL
```

porque ORM Persistence Engine opera sobre semántica de entidades.

El SQL pertenece posteriormente al Compiler.

---

# 8. Lifecycle ≠ Transaction Lifecycle

Debe mantenerse:

```text
Entity Lifecycle
        ≠
Transaction Lifecycle
```

Los eventos:

```text
transactionStarted
transactionCommitted
transactionRolledBack
```

pertenecen al Transaction System.

---

# 9. Regla crítica

```text
postPersist
≠
transaction committed

postUpdate
≠
transaction committed

postRemove
≠
transaction committed

postFlush
≠
transaction committed
```

---

# 10. Ejemplo

```text
BEGIN

INSERT user
postPersist()

UPDATE profile
postUpdate()

postFlush()

ROLLBACK
```

Todos los callbacks anteriores pudieron ejecutarse aunque:

```text
nothing was committed
```

---

# 11. Side effects externos

Por tanto, esto es peligroso:

```php
#[PostPersist]
public function sendWelcomeEmail(): void
{
    $mailer->send(...);
}
```

porque el correo podría enviarse antes de:

```text
COMMIT
```

---

# 12. Solución

Los efectos que requieran commit deberán utilizar mecanismos como:

```text
AfterCommit Event
Transactional Outbox
Job After Commit
Transaction Synchronization
```

no `postPersist()`.

---

# 13. Posición arquitectónica

```text
Entity
  │
  ▼
EntityManager
  │
  ▼
UnitOfWork
  │
  ├───────────────┐
  ▼               ▼
Change Tracking   Lifecycle System
  │               │
  └──────┬────────┘
         ▼
Persistence Planner
         │
         ▼
Persistence Engine
         │
         ▼
Query Engine
         │
         ▼
Execution Engine
```

Lifecycle nunca sustituye ninguno de estos componentes.

---

# 14. Arquitectura general

```text
Lifecycle Metadata
        │
        ▼
Lifecycle Registry
        │
        ▼
Lifecycle Dispatcher
        │
        ├── Entity Callbacks
        ├── Entity Listeners
        ├── Subscribers
        └── Framework Integration
                │
                ▼
        Lifecycle Invocation
                │
                ▼
        Lifecycle Result
```

---

# 15. Lifecycle phase

Se define:

```php
enum EntityLifecyclePhase
{
    case LOAD;
    case PERSIST;
    case UPDATE;
    case REMOVE;
    case REFRESH;
    case FLUSH;
    case DETACH;
}
```

Pero una fase no será equivalente a un evento concreto.

---

# 16. Lifecycle event type

Se propone:

```php
enum EntityLifecycleEventType
{
    case PRE_PERSIST;
    case POST_PERSIST;

    case PRE_UPDATE;
    case POST_UPDATE;

    case PRE_REMOVE;
    case POST_REMOVE;

    case POST_LOAD;

    case PRE_REFRESH;
    case POST_REFRESH;

    case PRE_FLUSH;
    case POST_FLUSH;

    case PRE_DETACH;
    case POST_DETACH;
}
```

---

# 17. Core lifecycle events

El conjunto inicial recomendado:

```text
prePersist
postPersist

preUpdate
postUpdate

preRemove
postRemove

postLoad

preRefresh
postRefresh

preFlush
postFlush

preDetach
postDetach
```

---

# 18. Eventos mínimos vs extensiones

No todos deberán ser necesariamente expuestos como attributes públicos desde V1.

VoltStack podrá mantener:

```text
small stable public lifecycle
+
richer internal lifecycle
```

---

# 19. PRE_PERSIST

`prePersist` ocurre cuando una entidad `NEW` va a participar en una operación de persistencia.

Conceptualmente:

```text
NEW
 ↓
prePersist
 ↓
ChangeSet preparation
 ↓
Persistence planning
```

---

# 20. prePersist ≠ persist()

Debe distinguirse:

```php
$entityManager->persist($user);
```

de:

```text
prePersist
```

`persist()` puede solamente registrar:

```text
UNTRACKED → NEW
```

El callback puede diferirse hasta flush.

---

# 21. Recomendación

VoltStack deberá ejecutar:

```text
prePersist
```

durante el ciclo de flush cuando realmente se prepare la inserción.

Esto evita ejecutar callbacks para entidades que nunca se sincronizan.

---

# 22. prePersist modifications

`prePersist` podrá modificar campos persistentes.

Ejemplo:

```php
public function prePersist(): void
{
    $this->createdAt ??= new DateTimeImmutable();
}
```

---

# 23. Recalcular ChangeSet

Si `prePersist` modifica campos persistentes:

```text
callback
→ state mutation
→ ChangeSet must observe final values
```

---

# 24. POST_PERSIST

`postPersist` ocurre después de que el Persistence Engine haya obtenido un resultado suficientemente cierto para considerar ejecutada la inserción ORM.

---

# 25. postPersist puede observar generated ID

Ejemplo:

```text
INSERT
 ↓
generated ID = 42
 ↓
assign identity
 ↓
state reconciliation
 ↓
postPersist
```

Por tanto:

```php
$postPersistEvent->entity()->id();
```

podrá disponer del ID generado.

---

# 26. postPersist certainty

`postPersist` no deberá dispararse si:

```text
INSERT outcome = UNKNOWN
```

como si hubiera sido éxito.

---

# 27. PRE_UPDATE

Ocurre cuando:

```text
MANAGED
+
persistent ChangeSet
```

va a convertirse en una operación de actualización.

---

# 28. preUpdate context

Deberá poder acceder a:

```text
Entity
EntityMetadata
ChangeSet
EntityManagerContext
```

de manera controlada.

---

# 29. ChangeSet mutability

Una decisión importante:

```text
preUpdate
```

puede modificar la entidad.

Entonces el ChangeSet deberá ser:

```text
recomputed
```

o modificarse mediante una API controlada.

---

# 30. Recomendación

Preferir:

```text
callback mutates entity
→ ORM recomputes affected ChangeSet
```

sobre permitir editar estructuras internas arbitrariamente.

---

# 31. POST_UPDATE

Ocurre después de un UPDATE ORM con outcome suficientemente cierto.

Pero:

```text
postUpdate
≠
COMMIT
```

---

# 32. PRE_REMOVE

Ocurre cuando una entidad:

```text
REMOVED
```

está a punto de convertirse en operación DELETE.

---

# 33. preRemove restrictions

Modificar persistent fields en `preRemove` normalmente no tendrá utilidad.

Podrá permitirse modificar:

```text
non-persistent runtime state
domain bookkeeping
```

pero no deberá transformar silenciosamente DELETE en UPDATE.

---

# 34. Cancelación de remove

`preRemove` no deberá cancelar eliminación mediante:

```text
return false
```

de forma implícita.

Si VoltStack soporta veto:

```text
LifecycleVeto
```

deberá ser una capacidad formal y restringida.

---

# 35. Recomendación

V1:

```text
callbacks cannot veto core persistence operation
```

Una validación deberá fallar mediante excepción explícita antes de ejecución.

---

# 36. POST_REMOVE

Ocurre después de DELETE confirmado suficientemente.

La entidad puede encontrarse posteriormente:

```text
DETACHED
```

---

# 37. postRemove identity

El objeto puede conservar su identificador lógico.

Pero:

```text
identifier exists
≠
row exists
```

---

# 38. POST_LOAD

Ocurre después de:

```text
Result
→ Hydration
→ IdentityMap reconciliation
→ Entity state initialization
```

cuando la entidad queda disponible para uso ORM.

---

# 39. postLoad ≠ constructor

La entidad puede haber sido creada mediante:

```text
EntityInstantiator
```

sin ejecutar el constructor normal de dominio.

`postLoad` tampoco deberá utilizarse como sustituto universal del constructor.

---

# 40. postLoad y IdentityMap

Si una query devuelve varias filas del mismo entity por JOIN:

```text
User#10
User#10
User#10
```

IdentityMap produce:

```text
same managed instance
```

`postLoad` no deberá ejecutarse tres veces arbitrariamente.

---

# 41. Regla recomendada

```text
postLoad
```

se ejecuta cuando una instancia es materializada/initialized como entidad cargada dentro del contexto, no por cada row física.

---

# 42. Cached entity

Si:

```text
IdentityMap already contains User#10
```

una nueva query que reutiliza la misma instancia no implica necesariamente nuevo `postLoad`.

---

# 43. PRE_REFRESH

Antes de sobrescribir managed state con datos recargados.

Puede observar:

```text
current state
refresh policy
```

pero no debería bloquear silenciosamente refresh.

---

# 44. POST_REFRESH

Después de:

```text
reload
hydrate
snapshot replacement
state reconciliation
```

---

# 45. PRE_FLUSH

`preFlush` es un evento de scope ORM, no necesariamente de una sola entidad.

---

# 46. Diferencia

```text
prePersist(User)
```

es entity-specific.

```text
preFlush(EntityManager)
```

es persistence-context-specific.

---

# 47. Lifecycle scopes

Se propone:

```php
enum LifecycleEventScope
{
    case ENTITY;
    case PERSISTENCE_CONTEXT;
}
```

---

# 48. PRE_FLUSH

Podrá permitir:

- preparación de estado;
- validaciones;
- sincronización interna;
- recolección controlada de nuevas entidades.

Pero es una zona sensible a reentrancy.

---

# 49. POST_FLUSH

Ocurre después de terminar un ciclo de flush ORM.

No implica:

```text
transaction commit
```

---

# 50. PRE_DETACH

Puede ejecutarse antes de separar una entidad de PersistenceContext.

---

# 51. POST_DETACH

Puede ejecutarse después de remover:

```text
IdentityMap membership
tracking
snapshots
UnitOfWork registration
```

---

# 52. clear()

`EntityManager::clear()` podría implicar muchas entidades.

Ejecutar callbacks individuales para millones de entidades sería costoso.

---

# 53. Clear lifecycle policy

Debe distinguirse:

```text
explicit detach(entity)
```

de:

```text
bulk context clear()
```

---

# 54. Recomendación

`clear()` deberá disponer de lifecycle bulk/context events separados si son necesarios.

No ejecutar automáticamente millones de:

```text
preDetach/postDetach
```

salvo policy explícita.

---

# 55. Lifecycle callback

Un callback es comportamiento declarado directamente en la entidad.

Ejemplo:

```php
#[PrePersist]
public function initializeCreatedAt(): void
{
    $this->createdAt ??= new DateTimeImmutable();
}
```

---

# 56. Entity callback restrictions

Un callback deberá ser:

- deterministic cuando sea posible;
- rápido;
- local;
- sin `flush()` recursivo;
- sin control de transacciones;
- sin depender de SQL vendor-specific.

---

# 57. Entity callback dependencies

No se recomienda inyectar servicios arbitrarios dentro de entidades solo para lifecycle.

Para eso existen:

```text
Entity Listeners
```

---

# 58. Entity Listener

Un listener es un servicio externo que observa lifecycle de uno o más entity types.

Ejemplo:

```php
final class UserLifecycleListener
{
    public function prePersist(
        User $user,
        PrePersistEvent $event
    ): void {
        // ...
    }
}
```

---

# 59. Listener advantages

Permite:

```text
dependency injection
cross-cutting behavior
separation from domain entity
testability
```

---

# 60. Listener ≠ Subscriber

Listener suele estar asociado a:

```text
specific entity
specific lifecycle event
```

Subscriber declara interés en varios eventos.

---

# 61. Lifecycle Subscriber

Ejemplo:

```php
final class TimestampSubscriber
    implements EntityLifecycleSubscriber
{
    public static function subscribedEvents(): array
    {
        return [
            EntityLifecycleEventType::PRE_PERSIST,
            EntityLifecycleEventType::PRE_UPDATE,
        ];
    }
}
```

---

# 62. Global subscriber

Puede observar múltiples entity types.

Debe usarse con cuidado para evitar:

```text
hidden global behavior
```

---

# 63. Callback source taxonomy

```text
Entity Callback
Entity Listener
Lifecycle Subscriber
Framework Adapter
Extension Listener
```

---

# 64. Lifecycle descriptor

Metadata compilada deberá representar callbacks como:

```php
final readonly class LifecycleCallbackDescriptor
{
    public function __construct(
        public EntityLifecycleEventType $event,
        public string $method,
        public int $priority,
    ) {}
}
```

---

# 65. Listener descriptor

```php
final readonly class EntityLifecycleListenerDescriptor
{
    public function __construct(
        public EntityType $entityType,
        public EntityLifecycleEventType $event,
        public string $serviceId,
        public string $method,
        public int $priority,
    ) {}
}
```

---

# 66. Compiled lifecycle metadata

Reflection y Attributes solo deberán utilizarse durante:

```text
metadata compilation
```

No para inspeccionar callbacks repetidamente durante cada flush.

---

# 67. Pipeline

```text
PHP Attributes
      ↓
Metadata Loader
      ↓
Lifecycle Metadata Normalizer
      ↓
Lifecycle Validator
      ↓
Compiled Lifecycle Metadata
      ↓
Metadata Cache
```

---

# 68. Attributes

Ejemplo:

```php
use VoltStack\Quantum\Database\ORM\Lifecycle\Attribute\PostLoad;
use VoltStack\Quantum\Database\ORM\Lifecycle\Attribute\PrePersist;
use VoltStack\Quantum\Database\ORM\Lifecycle\Attribute\PreUpdate;

final class User
{
    #[PrePersist]
    public function initializeCreatedAt(): void
    {
    }

    #[PreUpdate]
    public function updateTimestamp(): void
    {
    }

    #[PostLoad]
    public function initializeRuntimeState(): void
    {
    }
}
```

---

# 69. Attribute ≠ runtime handler

El Attribute solo declara metadata.

Nunca deberá contener lógica de dispatch.

---

# 70. Validación de callbacks

En boot/metadata compilation deberán detectarse:

```text
method does not exist
invalid visibility
invalid parameters
invalid return contract
duplicate incompatible callback
unsupported event
static callback where forbidden
```

---

# 71. Signature model

VoltStack podrá permitir:

```php
#[PrePersist]
private function beforePersist(): void
```

y:

```php
#[PrePersist]
private function beforePersist(
    PrePersistEvent $event
): void
```

pero las firmas permitidas deberán ser finitas.

---

# 72. No arbitrary reflection invocation

No aceptar cualquier combinación de parámetros por heurística.

---

# 73. Lifecycle event objects

Se propone:

```text
EntityLifecycleEvent
├── PrePersistEvent
├── PostPersistEvent
├── PreUpdateEvent
├── PostUpdateEvent
├── PreRemoveEvent
├── PostRemoveEvent
├── PostLoadEvent
├── PreRefreshEvent
├── PostRefreshEvent
├── PreDetachEvent
└── PostDetachEvent
```

Y:

```text
PersistenceContextLifecycleEvent
├── PreFlushEvent
└── PostFlushEvent
```

---

# 74. Base event

```php
abstract readonly class EntityLifecycleEvent
{
    public function __construct(
        public object $entity,
        public EntityMetadata $metadata,
        public EntityLifecycleContext $context,
    ) {}
}
```

---

# 75. Lifecycle context

No deberá exponer todo el framework.

Ejemplo:

```php
final readonly class EntityLifecycleContext
{
    public function __construct(
        public EntityManagerId $entityManagerId,
        public PersistenceContextId $persistenceContextId,
        public ?TransactionContextId $transactionContextId,
        public LifecycleInvocationId $invocationId,
    ) {}
}
```

---

# 76. EntityManager access

No todos los callbacks necesitan acceso al `EntityManager`.

Exponerlo universalmente facilita:

```text
recursive persistence
hidden queries
hidden flush
```

---

# 77. Recomendación

Event objects públicos deberán ofrecer capacidades mínimas.

Listeners avanzados podrán recibir una interfaz restringida:

```text
LifecyclePersistenceContext
```

en lugar del EntityManager completo.

---

# 78. LifecyclePersistenceContext

Podría permitir:

```text
metadata()
stateOf()
schedulePersist()
scheduleRemove()
```

según evento.

Pero no necesariamente:

```text
flush()
beginTransaction()
commit()
rawSql()
```

---

# 79. Capability-based lifecycle context

Cada evento podrá exponer capacidades específicas.

Ejemplo:

```text
PrePersistContext
    can inspect metadata
    can inspect entity state
    cannot flush
```

---

# 80. Ordering

El orden de callbacks deberá ser determinista.

---

# 81. Fuentes

Posible orden:

```text
1. Entity callbacks
2. Entity-specific listeners
3. Subscribers
4. Framework integration observers
```

Pero VoltStack no deberá depender de orden implícito accidental.

---

# 82. Priority

Cada listener podrá declarar:

```text
priority
```

con orden determinista.

---

# 83. Tie-breaker

Misma prioridad:

```text
stable registration ID
```

o descriptor ID.

Nunca:

```text
hash map iteration order
```

---

# 84. LifecycleExecutionOrder

Se propone:

```text
priority DESC
then stable listener ID ASC
then method descriptor order
```

---

# 85. Entity callbacks order

Múltiples callbacks del mismo tipo deberán tener orden explícito o estable por metadata compilation.

---

# 86. Inheritance

Si entidades usan herencia:

```text
ParentEntity
    ↓
ChildEntity
```

debe definirse el orden.

---

# 87. Recomendación inheritance

Para `pre*`:

```text
parent → child
```

y para `post*`:

```text
child → parent
```

puede ser intuitivo.

Sin embargo, para evitar complejidad, VoltStack puede usar un único orden declarado/compilado.

---

# 88. Mejor regla

> El orden será determinado por metadata compilada y nunca por comportamiento incidental de Reflection.

---

# 89. Duplicate callbacks

Si una misma función se registra mediante:

```text
attribute
listener config
subscriber
```

no deberá ejecutarse accidentalmente varias veces.

---

# 90. Stable listener identity

Cada handler tendrá:

```text
LifecycleHandlerId
```

---

# 91. Dispatcher

Componente:

```text
EntityLifecycleDispatcher
```

Responsabilidades:

- obtener handlers compilados;
- ordenar;
- crear event context;
- invocar;
- manejar errores;
- detectar reentrancy;
- registrar telemetry.

---

# 92. Dispatcher no decide persistencia

No deberá decidir:

```text
INSERT
UPDATE
DELETE
```

---

# 93. Invocation plan

Para hot path podrá compilarse:

```text
LifecycleInvocationPlan
```

por:

```text
EntityType + LifecycleEventType
```

---

# 94. Lifecycle plan cache

Puede ser process-shared si:

```text
immutable
compiled
tenant-independent or correctly keyed
```

---

# 95. Lifecycle invocation state

Mutable:

```text
current callback stack
invocation IDs
reentrancy guards
temporary context
```

deberá ser operation-scoped.

---

# 96. Reentrancy

Caso:

```text
prePersist(User)
    ↓
listener
    ↓
entityManager->flush()
```

deberá bloquearse.

---

# 97. Recursive flush

Regla:

```text
flush while flush lifecycle active
→ RecursiveFlushException
```

---

# 98. Persist from callback

Más complejo:

```text
prePersist(Order)
    ↓
persist(AuditRecord)
```

Puede ser legítimo en ciertos ORMs.

---

# 99. Política

VoltStack deberá diferenciar:

```text
schedule additional entity
```

de:

```text
execute nested flush
```

---

# 100. Lifecycle mutation window

Se propone permitir nuevas entidades durante ventanas controladas:

```text
preFlush
prePersist
```

si el UnitOfWork puede estabilizar el graph.

---

# 101. Stabilization loop

Conceptualmente:

```text
Collect UoW
   ↓
Lifecycle pre-phase
   ↓
New changes/entities?
   ├── yes → recompute
   └── no
         ↓
Freeze Persistence Plan
```

---

# 102. Infinite lifecycle mutation

Debe existir:

```text
LifecycleStabilizationLimit
```

para impedir:

```text
callback creates entity
callback creates entity
callback creates entity
...
```

---

# 103. Lifecycle stabilization

```text
UoW₀
 ↓ lifecycle
UoW₁
 ↓ lifecycle
UoW₂
 ↓
stable
```

Solo entonces:

```text
PersistencePlan
```

---

# 104. Max iterations

Configurable con default pequeño.

Ejemplo conceptual:

```text
maxLifecycleStabilizationPasses = 8
```

No debe considerarse parte de la semántica pública permanente.

---

# 105. Failure to stabilize

Produce:

```text
LifecycleStabilizationException
```

---

# 106. ChangeSet interaction

Secuencia recomendada para UPDATE:

```text
Initial dirty detection
        ↓
Initial ChangeSet
        ↓
preUpdate
        ↓
Entity mutation?
        ↓
Recompute ChangeSet
        ↓
Validate final ChangeSet
        ↓
Persistence Operation
```

---

# 107. Empty ChangeSet after preUpdate

Si callback revierte todos los cambios:

```text
ChangeSet = ∅
```

el UPDATE podrá eliminarse del PersistencePlan.

---

# 108. New ChangeSet after callback

Si callback modifica otro campo persistente:

```text
name changed
+
updatedAt changed
```

ambos deberán aparecer.

---

# 109. postUpdate mutations

Modificar persistent state en:

```text
postUpdate
```

es peligroso.

La operación UPDATE ya ocurrió.

---

# 110. Regla

Si `postUpdate` modifica un persistent field:

```text
entity becomes DIRTY again
```

No se ejecutará automáticamente un segundo UPDATE dentro del mismo flush salvo política explícita.

---

# 111. Recomendación

```text
post* persistent mutation
→ mark dirty for future flush
```

y emitir diagnostic cuando sea accidental.

---

# 112. postPersist mutation

La misma regla.

```text
postPersist
→ persistent mutation
→ future dirty state
```

---

# 113. postLoad mutation

Puede ocurrir.

Debe definirse si los cambios forman parte del baseline.

---

# 114. Regla recomendada

```text
hydrate
→ postLoad
→ establish final snapshot
```

si `postLoad` está destinado a inicializar estado persistente.

Pero esto puede esconder mutaciones.

---

# 115. Mejor separación

Para máxima claridad:

```text
hydrate persistent fields
→ establish persistent baseline
→ postLoad
```

Entonces cualquier cambio persistente en `postLoad` es una modificación real.

---

# 116. Decisión VoltStack

Usar:

```text
Hydrate
→ IdentityMap registration
→ Snapshot baseline
→ MANAGED
→ postLoad
```

Por tanto:

```text
persistent mutation in postLoad
→ potentially DIRTY
```

---

# 117. Runtime-only initialization

`postLoad` es apropiado para:

```text
derived transient state
memoization reset
non-persistent helpers
```

---

# 118. Failure semantics

Si un:

```text
prePersist
```

lanza excepción antes de cualquier DB effect:

```text
flush fails before persistence operation
```

---

# 119. Callback exception

Deberá conservar:

```text
original exception
handler identity
entity type
event type
invocation ID
```

---

# 120. LifecycleInvocationException

Ejemplo:

```text
LifecycleInvocationException
    caused by DomainInvariantException
```

sin perder la causa.

---

# 121. Pre-event failure

Generalmente:

```text
pre* failure
→ operation not executed
```

para la entidad afectada.

Si el flush ya ejecutó operaciones previas, la situación puede ser parcialmente aplicada dentro de la transacción.

---

# 122. Post-event failure

Más delicado.

```text
DB operation succeeded
↓
postPersist throws
```

La operación ya ocurrió físicamente.

---

# 123. Post-event failure ≠ operation not executed

Nunca reportar:

```text
INSERT failed
```

si en realidad:

```text
INSERT succeeded
postPersist failed
```

---

# 124. Outcome model

Debe preservar:

```text
PersistenceOperationOutcome = SUCCEEDED
LifecycleOutcome = FAILED
TransactionOutcome = PENDING
FlushOutcome = FAILED
```

---

# 125. Transaction rollback after callback failure

Si existe una transacción:

```text
postPersist throws
→ flush aborts
→ transaction may rollback
```

Pero rollback deberá ser responsabilidad de la política transaccional correspondiente.

---

# 126. No hidden transaction management

Lifecycle System no hará:

```text
begin
commit
rollback
```

arbitrariamente.

---

# 127. Non-transactional platform

Si una operación no es transactional:

```text
post* failure
```

puede dejar cambios durables.

Esto deberá aparecer en diagnostics.

---

# 128. UNKNOWN execution outcome

Si Persistence Engine reporta:

```text
UNKNOWN
```

no ejecutar:

```text
postPersist
postUpdate
postRemove
```

como si existiera éxito confirmado.

---

# 129. Lifecycle success condition

Para `post*` de persistencia:

```text
post event allowed
=
OperationOutcomeSemanticallySuccessful
∧
OutcomeCertaintySufficient
```

---

# 130. After-commit lifecycle

No conviene llamar:

```text
postPersistCommitted
```

porque mezcla Entity Lifecycle con Transaction Lifecycle.

---

# 131. Mejor arquitectura

Usar:

```text
Transaction Synchronization
+
Committed Persistence Effects
```

para producir:

```text
AfterCommit domain/framework events
```

---

# 132. TransactionSynchronization

Conceptualmente:

```text
flush
 ↓
record committed-effect candidates
 ↓
transaction commit
 ↓
TransactionSynchronization
 ↓
publish after-commit effects
```

---

# 133. Outbox

Para side effects durables:

```text
Domain Change
+
Outbox Record
    ↓ same transaction
COMMIT
    ↓
Outbox Processor
    ↓
External Event
```

---

# 134. Lifecycle callback no reemplaza Outbox

Regla crítica para:

```text
email
webhook
message broker
external API
```

---

# 135. Domain events

Una entidad puede registrar:

```php
$this->recordEvent(
    new OrderPlaced(...)
);
```

El ORM podrá recolectarlos.

Pero:

```text
recorded
≠
published
```

---

# 136. Domain event publishing

Puede integrarse:

```text
Entity
 ↓
Domain Event Collector
 ↓
UnitOfWork
 ↓
Transaction-aware Event Bridge
 ↓
after commit / outbox
```

---

# 137. EventSystem integration

El ORM podrá publicar lifecycle observations al EventSystem mediante:

```text
DatabaseLifecycleEventBridge
```

---

# 138. Bridge optional

Database ORM no deberá depender obligatoriamente del EventSystem.

---

# 139. No semantic mutation by telemetry

Listeners destinados a:

```text
telemetry
profiling
debugging
```

deberán ser observational.

---

# 140. Listener categories

Puede existir:

```php
enum LifecycleHandlerMode
{
    case MUTATING;
    case OBSERVATIONAL;
}
```

---

# 141. Observational listener

No deberá modificar:

```text
entity
UnitOfWork
ChangeSet
PersistenceContext
```

---

# 142. Mutating listener

Solo será permitido en eventos donde exista una mutation window válida.

---

# 143. Mutation policy matrix

| Event | Persistent mutation |
|---|---|
| prePersist | allowed |
| postPersist | allowed but future dirty state |
| preUpdate | allowed + recompute |
| postUpdate | future dirty state |
| preRemove | discouraged/restricted |
| postRemove | no persistence effect |
| postLoad | treated as new mutation |
| preRefresh | restricted |
| postRefresh | future dirty state |
| preFlush | controlled |
| postFlush | future flush only |

---

# 144. Listener dependency injection

Entity listeners serán servicios normales del container.

Ejemplo:

```php
final class SlugListener
{
    public function __construct(
        private SlugGenerator $generator,
    ) {}
}
```

---

# 145. Listener lifetime

Listeners deberán ser:

```text
stateless
```

o tener lifecycle compatible con persistent runtime.

---

# 146. Singleton listener

Puede ser singleton únicamente si:

```text
no request mutable state
no EntityManager retained
no entity retained
```

---

# 147. Scoped listener

Si requiere request state:

```text
request/operation scope
```

---

# 148. Prohibición

Nunca almacenar:

```php
private ?object $lastEntity;
```

en un singleton lifecycle listener.

---

# 149. Persistent runtime

En FrankenPHP:

```text
Worker
 ├── immutable Lifecycle Metadata
 ├── immutable Invocation Plans
 │
 ├── Request A
 │   └── mutable Lifecycle Invocation Context
 │
 ├── RESET
 │
 └── Request B
```

---

# 150. Shared state permitido

```text
Compiled Lifecycle Metadata
Lifecycle Handler Definitions
Immutable Invocation Plans
Frozen Extension Registry
```

---

# 151. Shared state prohibido

```text
current entity
current EntityManager
current transaction
current callback stack
current ChangeSet
current tenant
pending domain events
```

---

# 152. Async/coroutine runtime

RoadRunner/OpenSwoole adapters deberán mantener:

```text
logical operation scope
```

incluso cuando existan:

```text
coroutines
fibers
concurrent requests
```

---

# 153. Lifecycle dispatcher thread safety

El dispatcher compartido solo podrá contener:

```text
immutable definitions
```

Mutable invocation state deberá estar fuera.

---

# 154. Multitenancy

Lifecycle callbacks no deberán resolver tenant mediante global mutable state.

---

# 155. Tenant context

Si es necesario:

```text
LifecycleContext
→ DatabaseContext
→ logical TenantContext
```

de forma scope-safe.

---

# 156. Cross-tenant operations

Un callback no deberá:

```text
switch tenant
```

silenciosamente durante flush.

---

# 157. Database context mutation

Modificar:

```text
tenant
shard
connection target
```

durante una lifecycle invocation activa deberá rechazarse.

---

# 158. Security

Lifecycle hooks son código privilegiado respecto al ORM.

Pueden observar:

```text
entities
changes
metadata
```

por lo que deben estar sujetos a:

```text
trusted application/package code
extension governance
```

---

# 159. Lifecycle ≠ Authorization

Un callback:

```text
preRemove
```

no deberá ser el mecanismo principal de autorización.

Authorization debe ocurrir antes en la capa correspondiente.

---

# 160. Defense in depth

Un lifecycle listener puede validar invariantes de seguridad/persistencia, pero:

```text
listener check
≠
Authorization System
```

---

# 161. Sensitive fields

Diagnostics de lifecycle no deberán imprimir automáticamente:

```text
password
token
secret
PII
```

---

# 162. Handler source trust

Packages/plugins deberán declarar lifecycle handlers mediante extension contracts registrados.

No mediante arbitrary runtime reflection scanning.

---

# 163. Frozen registry

Después de bootstrap:

```text
LifecycleRegistry
→ FROZEN
```

en producción.

---

# 164. Dynamic registration

Podrá permitirse en:

```text
development
testing
```

pero deberá invalidar caches determinísticamente.

---

# 165. Lifecycle extension ID

Toda extensión deberá poseer:

```text
stable LifecycleExtensionId
```

---

# 166. Handler collision

Dos handlers no se sustituyen mediante:

```text
last wins
```

Las colisiones incompatibles serán errores.

---

# 167. Performance

Lifecycle hot path deberá evitar:

```text
reflection per entity
container lookup per callback when cacheable
rebuilding listener lists
dynamic method scanning
```

---

# 168. Compiled invocation

Ideal:

```text
EntityType
+
EventType
→ LifecycleInvocationPlan
```

---

# 169. Empty lifecycle fast path

Si una entidad no tiene handlers:

```text
dispatch
→ near-zero overhead
```

---

# 170. Bitmask optimization

Compiled metadata podrá incluir:

```text
LifecycleEventMask
```

Ejemplo conceptual:

```text
PRE_PERSIST | PRE_UPDATE | POST_LOAD
```

para comprobar rápidamente si existe lifecycle work.

---

# 171. No premature complexity

La optimización no deberá comprometer claridad ni extensibilidad.

---

# 172. Batch operations

Bulk UPDATE/DELETE puede no hidratar entidades.

Por tanto:

```text
bulk update
≠
N × preUpdate/postUpdate entity callbacks
```

---

# 173. Regla bulk

Entity lifecycle hooks solo se garantizan cuando una operación atraviesa:

```text
Entity Persistence Engine
```

con entidades individuales.

---

# 174. Bulk lifecycle

Si se requiere, deberá existir:

```text
BulkPersistenceLifecycle
```

separado.

---

# 175. Direct Query Builder

Esto:

```php
DB::table('users')->update(...);
```

no ejecutará:

```text
User::preUpdate()
User::postUpdate()
```

---

# 176. Raw SQL

Raw SQL tampoco ejecutará entity lifecycle.

---

# 177. Repository bulk methods

Si:

```php
$repository->deleteWhere(...)
```

usa bulk persistence sin entidades:

```text
entity lifecycle callbacks may not run
```

deberá documentarse.

---

# 178. Soft delete

Si Soft Delete se implementa como extensión ORM:

```text
remove()
→ UPDATE deleted_at
```

la semántica lifecycle deberá decidir si representa:

```text
REMOVE lifecycle
```

aunque físicamente sea UPDATE.

---

# 179. Regla semántica

Lifecycle se basa en:

```text
ORM semantic operation
```

no en SQL físico.

Por tanto un soft delete puede producir:

```text
preRemove/postRemove
```

aunque el compiler genere UPDATE.

---

# 180. Temporal entities

La misma regla aplica a sistemas temporales/versionados.

---

# 181. Cascade persistence

Supongamos:

```text
Order
 ├── OrderItem A
 └── OrderItem B
```

con cascade persist.

Lifecycle deberá respetar el Persistence Graph.

---

# 182. Callback ordering across entities

No deberá prometerse un orden arbitrario basado en:

```text
array insertion order
```

---

# 183. Dependency ordering

Cuando exista dependencia real:

```text
parent INSERT
before child INSERT
```

los `postPersist` correspondientes seguirán la ejecución semántica real.

---

# 184. Lifecycle ordering graph

Conceptualmente:

```text
EntityChangeGraph
       ↓
PersistencePlan
       ↓
Lifecycle execution points
```

---

# 185. pre-events and planning

Algunos pre-events ocurren antes de congelar el plan porque pueden modificar state.

---

# 186. post-events and execution

Los post-events ocurren después del outcome semántico de la operación correspondiente.

---

# 187. Cascade generated during callback

Si un callback agrega una relación con nueva entidad:

```text
prePersist(Order)
→ adds NEW OrderItem
```

UnitOfWork deberá:

```text
detect
register
recompute graph
```

si la mutation window lo permite.

---

# 188. Orphan removal

Lifecycle seguirá la operación semántica resultante.

Si un orphan es programado para removal:

```text
preRemove
...
postRemove
```

según Persistence Plan.

---

# 189. Hydration errors

Si hydration falla:

```text
postLoad
```

no se ejecutará.

---

# 190. Partial hydration

Si se permite partial entity:

```text
postLoad
```

deberá saber mediante context que:

```text
entity may be partial
```

o podrán existir restricciones.

---

# 191. Recomendación

Callbacks que requieren entidad completa deberán poder declarar:

```text
requiresFullEntity = true
```

o ser rechazados para partial hydration.

---

# 192. Lazy loading

Inicializar una relación lazy:

```text
does not trigger postLoad on owner again
```

---

# 193. Loaded related entity

Una entidad relacionada recién materializada sí podrá recibir su propio:

```text
postLoad
```

---

# 194. Refresh and postLoad

`refresh()` no deberá disparar:

```text
postLoad
```

si existe:

```text
postRefresh
```

Esto evita ambigüedad.

---

# 195. Lifecycle causality

Cada invocation podrá registrar:

```text
cause
```

Ejemplos:

```text
QUERY_LOAD
RELATION_LOAD
REFRESH
CASCADE_PERSIST
EXPLICIT_PERSIST
CASCADE_REMOVE
EXPLICIT_REMOVE
FLUSH
```

---

# 196. LifecycleInvocationId

Cada invocation tendrá ID para:

```text
telemetry
debugging
causality
error reporting
```

---

# 197. FlushCycleId

Todos los callbacks de un mismo flush podrán compartir:

```text
FlushCycleId
```

---

# 198. TransactionContextId

Si existe transacción:

```text
TransactionContextId
```

podrá incluirse como correlación.

Pero:

```text
presence of transaction ID
≠
committed
```

---

# 199. Telemetry

Métricas posibles:

```text
orm.lifecycle.invocations
orm.lifecycle.duration
orm.lifecycle.failures
orm.lifecycle.handlers
orm.lifecycle.reentrancy_blocked
orm.lifecycle.stabilization_passes
orm.lifecycle.mutations
```

---

# 200. Tracing

Span conceptual:

```text
orm.lifecycle.pre_update
```

con:

```text
entity.type
handler.count
flush.cycle
duration
outcome
```

---

# 201. Cardinality

No incluir raw entity IDs de alta cardinalidad por defecto en metrics.

---

# 202. Debugging

Debug Toolbar podrá mostrar:

```text
User
  prePersist
    User::initializeCreatedAt      0.03 ms
    TimestampListener             0.07 ms

Order
  postPersist
    DomainEventCollector          0.02 ms
```

---

# 203. Slow lifecycle detection

Callbacks lentos podrán detectarse.

Ejemplo:

```text
Lifecycle handler took 430 ms
```

---

# 204. Hidden query detection

Telemetry podrá detectar:

```text
query executed inside lifecycle callback
```

---

# 205. Lifecycle query policy

No necesariamente prohibir todas las queries.

Pero deberán ser:

```text
observable
bounded
non-recursive
```

---

# 206. Query in preUpdate

Puede provocar:

```text
N+1 lifecycle query
```

al actualizar miles de entidades.

El profiler deberá detectarlo.

---

# 207. Recommended policy

Lifecycle callbacks deberán evitar I/O siempre que sea posible.

---

# 208. Testing architecture

Se requieren:

```text
Unit Tests
Integration Tests
Ordering Tests
Failure Tests
Transaction Tests
Reentrancy Tests
Runtime Isolation Tests
Performance Tests
```

---

# 209. Test: prePersist

Verificar:

```text
persist(entity)
→ no callback yet

flush()
→ prePersist once
```

según política elegida.

---

# 210. Test: postPersist

Verificar:

```text
INSERT success
→ generated ID assigned
→ postPersist sees ID
```

---

# 211. Test: unknown INSERT

```text
INSERT outcome UNKNOWN
→ no postPersist success event
```

---

# 212. Test: preUpdate recomputation

```text
name changed
→ preUpdate changes updatedAt
→ final ChangeSet contains both
```

---

# 213. Test: postUpdate mutation

```text
postUpdate changes persistent field
→ entity remains/becomes DIRTY
→ no hidden recursive UPDATE
```

---

# 214. Test: remove

```text
preRemove
→ DELETE
→ postRemove
```

---

# 215. Test: rollback

```text
postPersist executes
→ transaction rollback
→ no claim that postPersist meant commit
```

---

# 216. Test: after-commit separation

Verificar que:

```text
postPersist
```

y:

```text
transaction afterCommit
```

son eventos distintos.

---

# 217. Test: callback exception

```text
prePersist throws
→ no INSERT for that operation
```

---

# 218. Test: post callback exception

```text
INSERT succeeds
→ postPersist throws
→ outcome reports DB operation success + lifecycle failure
```

---

# 219. Test: recursive flush

```text
prePersist → flush()
→ RecursiveFlushException
```

---

# 220. Test: stabilization

```text
prePersist A
→ creates B
→ B registered
→ graph recomputed
→ stable plan
```

---

# 221. Test: stabilization overflow

```text
callback continually creates entities
→ LifecycleStabilizationException
```

---

# 222. Test: IdentityMap postLoad

Duplicate join rows deberán ejecutar:

```text
postLoad once per materialized managed entity
```

---

# 223. Test: refresh

```text
refresh()
→ preRefresh
→ reload
→ postRefresh
```

sin `postLoad` duplicado.

---

# 224. Test: bulk operation

```text
bulk update 10,000 rows
→ no 10,000 entity lifecycle callbacks
```

si no hubo entidades materializadas.

---

# 225. Test: soft delete

Verificar:

```text
semantic remove
→ remove lifecycle
```

aunque SQL físico sea UPDATE.

---

# 226. Test: persistent runtime

```text
Request A lifecycle stack
→ reset
→ Request B sees empty stack
```

---

# 227. Test: tenant isolation

Callback de Tenant A nunca deberá observar:

```text
Tenant B PersistenceContext
```

---

# 228. Test: deterministic order

Mismos handlers + metadata:

```text
→ same invocation order
```

en todas las ejecuciones.

---

# 229. Error hierarchy

```text
DatabaseOrmException
└── EntityLifecycleException
    ├── InvalidLifecycleCallbackException
    ├── InvalidLifecycleListenerException
    ├── InvalidLifecycleSubscriberException
    ├── LifecycleMetadataException
    ├── LifecycleHandlerCollisionException
    ├── LifecycleInvocationException
    ├── LifecycleHandlerFailureException
    ├── LifecycleMutationException
    ├── LifecycleMutationNotAllowedException
    ├── LifecycleReentrancyException
    ├── RecursiveFlushException
    ├── LifecycleStabilizationException
    ├── LifecycleContextException
    ├── LifecycleOrderingException
    ├── LifecycleExtensionException
    ├── LifecycleRuntimeIsolationException
    └── EntityLifecycleInvariantException
```

---

# 230. Diagnostic example

```text
DB-ORM-LIFECYCLE-REENTRANCY-001

Event:
PRE_UPDATE

Entity:
App\Entity\User

Handler:
App\Database\UserListener::preUpdate

Operation:
EntityManager::flush()

Reason:
A lifecycle handler attempted to start a nested flush while
flush cycle 01J... is already active.

Action:
Schedule the required entity changes in the current UnitOfWork
instead of starting a recursive flush.
```

---

# 231. Post-event failure diagnostic

```text
DB-ORM-LIFECYCLE-POST-002

Event:
POST_PERSIST

Entity:
App\Entity\Order

Persistence operation:
INSERT

Persistence outcome:
SUCCEEDED

Lifecycle handler:
OrderListener::postPersist

Lifecycle outcome:
FAILED

Transaction state:
ACTIVE

Important:
The INSERT succeeded at the database-operation level, but the
transaction has not yet been committed.
```

---

# 232. Arquitectura de directorios

```text
src/Quantum/Database/ORM/Lifecycle/
│
├── Contract/
│   ├── EntityLifecycleDispatcher.php
│   ├── EntityLifecycleListener.php
│   ├── EntityLifecycleSubscriber.php
│   ├── LifecycleHandler.php
│   └── LifecycleExtension.php
│
├── Attribute/
│   ├── PrePersist.php
│   ├── PostPersist.php
│   ├── PreUpdate.php
│   ├── PostUpdate.php
│   ├── PreRemove.php
│   ├── PostRemove.php
│   ├── PostLoad.php
│   ├── PreRefresh.php
│   ├── PostRefresh.php
│   ├── PreDetach.php
│   └── PostDetach.php
│
├── Event/
│   ├── EntityLifecycleEvent.php
│   ├── PrePersistEvent.php
│   ├── PostPersistEvent.php
│   ├── PreUpdateEvent.php
│   ├── PostUpdateEvent.php
│   ├── PreRemoveEvent.php
│   ├── PostRemoveEvent.php
│   ├── PostLoadEvent.php
│   ├── PreRefreshEvent.php
│   ├── PostRefreshEvent.php
│   ├── PreDetachEvent.php
│   ├── PostDetachEvent.php
│   ├── PreFlushEvent.php
│   └── PostFlushEvent.php
│
├── Metadata/
│   ├── LifecycleMetadata.php
│   ├── LifecycleCallbackDescriptor.php
│   ├── LifecycleListenerDescriptor.php
│   ├── LifecycleHandlerDescriptor.php
│   └── LifecycleEventMask.php
│
├── Registry/
│   ├── LifecycleRegistry.php
│   └── LifecycleHandlerRegistry.php
│
├── Dispatch/
│   ├── DefaultEntityLifecycleDispatcher.php
│   ├── LifecycleInvocationPlan.php
│   ├── LifecycleInvocationPlanner.php
│   └── LifecycleHandlerInvoker.php
│
├── Context/
│   ├── EntityLifecycleContext.php
│   ├── LifecyclePersistenceContext.php
│   ├── LifecycleInvocationId.php
│   └── FlushCycleId.php
│
├── Ordering/
│   ├── LifecycleHandlerOrder.php
│   └── LifecycleOrderingResolver.php
│
├── Mutation/
│   ├── LifecycleMutationPolicy.php
│   ├── LifecycleMutationWindow.php
│   └── LifecycleStabilizer.php
│
├── Runtime/
│   ├── LifecycleInvocationStack.php
│   ├── LifecycleReentrancyGuard.php
│   └── LifecycleRuntimeResetter.php
│
├── Integration/
│   ├── LifecycleEventBridge.php
│   ├── TransactionLifecycleBridge.php
│   └── DomainEventCollectorBridge.php
│
├── Telemetry/
│   ├── LifecycleTelemetry.php
│   └── LifecycleProfiler.php
│
├── Extension/
│   ├── LifecycleExtensionRegistry.php
│   └── LifecycleExtensionId.php
│
└── Exception/
    └── ...
```

---

# 233. Invariantes arquitectónicas

## DB-ORM-LIFECYCLE-001
Entity Lifecycle será distinto de Domain Lifecycle.

## DB-ORM-LIFECYCLE-002
Lifecycle Callback será distinto de Domain Event.

## DB-ORM-LIFECYCLE-003
ORM Lifecycle Event será distinto de Framework Event.

## DB-ORM-LIFECYCLE-004
Entity Lifecycle será distinto de Transaction Lifecycle.

## DB-ORM-LIFECYCLE-005
Entity Lifecycle será distinto de Entity State.

## DB-ORM-LIFECYCLE-006
Entity Lifecycle será distinto de SQL execution hooks.

## DB-ORM-LIFECYCLE-007
Lifecycle se expresará en semántica ORM.

## DB-ORM-LIFECYCLE-008
postPersist no significará transaction commit.

## DB-ORM-LIFECYCLE-009
postUpdate no significará transaction commit.

## DB-ORM-LIFECYCLE-010
postRemove no significará transaction commit.

## DB-ORM-LIFECYCLE-011
postFlush no significará transaction commit.

## DB-ORM-LIFECYCLE-012
External side effects que requieran commit no dependerán de postPersist.

## DB-ORM-LIFECYCLE-013
Transactional Outbox será preferible para efectos externos durables.

## DB-ORM-LIFECYCLE-014
persist() no será equivalente a prePersist.

## DB-ORM-LIFECYCLE-015
prePersist podrá diferirse hasta flush.

## DB-ORM-LIFECYCLE-016
prePersist podrá modificar persistent state.

## DB-ORM-LIFECYCLE-017
Cambios de prePersist serán considerados por Persistence Planner.

## DB-ORM-LIFECYCLE-018
postPersist requerirá persistence outcome suficientemente cierto.

## DB-ORM-LIFECYCLE-019
postPersist podrá observar generated identity cuando haya sido establecida.

## DB-ORM-LIFECYCLE-020
UNKNOWN INSERT outcome no producirá postPersist de éxito.

## DB-ORM-LIFECYCLE-021
preUpdate operará sobre una entidad con cambios persistentes.

## DB-ORM-LIFECYCLE-022
Cambios realizados en preUpdate requerirán recomputación controlada del ChangeSet.

## DB-ORM-LIFECYCLE-023
Handlers no editarán internals del ChangeSet arbitrariamente.

## DB-ORM-LIFECYCLE-024
postUpdate no significará commit.

## DB-ORM-LIFECYCLE-025
preRemove no ejecutará DELETE.

## DB-ORM-LIFECYCLE-026
preRemove no cancelará implícitamente mediante return false.

## DB-ORM-LIFECYCLE-027
Veto de persistence requerirá contrato explícito si alguna vez se soporta.

## DB-ORM-LIFECYCLE-028
postRemove requerirá DELETE semánticamente exitoso.

## DB-ORM-LIFECYCLE-029
postRemove no significará commit.

## DB-ORM-LIFECYCLE-030
postLoad ocurrirá por materialización lógica de entidad.

## DB-ORM-LIFECYCLE-031
postLoad no ocurrirá por cada row duplicada de JOIN.

## DB-ORM-LIFECYCLE-032
IdentityMap reuse no implicará nuevo postLoad.

## DB-ORM-LIFECYCLE-033
postLoad no sustituirá constructor de dominio.

## DB-ORM-LIFECYCLE-034
refresh utilizará lifecycle propio.

## DB-ORM-LIFECYCLE-035
refresh no reutilizará postLoad ambiguamente.

## DB-ORM-LIFECYCLE-036
preFlush será context-scoped.

## DB-ORM-LIFECYCLE-037
postFlush será context-scoped.

## DB-ORM-LIFECYCLE-038
preFlush no significará transaction begin.

## DB-ORM-LIFECYCLE-039
postFlush no significará transaction commit.

## DB-ORM-LIFECYCLE-040
detach lifecycle será distinto de remove lifecycle.

## DB-ORM-LIFECYCLE-041
clear() no deberá generar O(N) callbacks obligatoriamente.

## DB-ORM-LIFECYCLE-042
Bulk clear podrá utilizar context lifecycle separado.

## DB-ORM-LIFECYCLE-043
Entity callbacks podrán declararse mediante Attributes.

## DB-ORM-LIFECYCLE-044
Attributes serán metadata declarativa.

## DB-ORM-LIFECYCLE-045
Attributes no implementarán dispatch.

## DB-ORM-LIFECYCLE-046
Reflection no será hot-path lifecycle mechanism.

## DB-ORM-LIFECYCLE-047
Lifecycle metadata será compilable.

## DB-ORM-LIFECYCLE-048
Compiled lifecycle metadata será immutable.

## DB-ORM-LIFECYCLE-049
Invalid callbacks se detectarán preferentemente durante bootstrap.

## DB-ORM-LIFECYCLE-050
Callback signatures serán explícitas.

## DB-ORM-LIFECYCLE-051
VoltStack no inferirá parámetros arbitrariamente por Reflection.

## DB-ORM-LIFECYCLE-052
Entity listeners serán servicios externos.

## DB-ORM-LIFECYCLE-053
Entity listeners podrán usar dependency injection.

## DB-ORM-LIFECYCLE-054
Subscribers declararán eventos explícitamente.

## DB-ORM-LIFECYCLE-055
Global subscribers deberán ser visibles e inspeccionables.

## DB-ORM-LIFECYCLE-056
Global subscribers no crearán hidden behavior no diagnosticable.

## DB-ORM-LIFECYCLE-057
Lifecycle handlers tendrán stable identity.

## DB-ORM-LIFECYCLE-058
Handler ordering será determinista.

## DB-ORM-LIFECYCLE-059
Handler ordering no dependerá de hash iteration.

## DB-ORM-LIFECYCLE-060
Priority ties tendrán stable tie-breaker.

## DB-ORM-LIFECYCLE-061
Inheritance ordering será compilado explícitamente.

## DB-ORM-LIFECYCLE-062
Handler duplicates no producirán accidental double invocation.

## DB-ORM-LIFECYCLE-063
Dispatcher no decidirá INSERT/UPDATE/DELETE.

## DB-ORM-LIFECYCLE-064
Dispatcher no generará SQL.

## DB-ORM-LIFECYCLE-065
Dispatcher no ejecutará queries directamente.

## DB-ORM-LIFECYCLE-066
Lifecycle invocation plans podrán cachearse si son immutable.

## DB-ORM-LIFECYCLE-067
Mutable invocation state será scope-local.

## DB-ORM-LIFECYCLE-068
Recursive flush será rechazado.

## DB-ORM-LIFECYCLE-069
Nested persistence scheduling será distinto de nested flush.

## DB-ORM-LIFECYCLE-070
Additional entities podrán registrarse únicamente en mutation windows permitidas.

## DB-ORM-LIFECYCLE-071
UnitOfWork deberá estabilizarse después de lifecycle mutations.

## DB-ORM-LIFECYCLE-072
Lifecycle stabilization será bounded.

## DB-ORM-LIFECYCLE-073
Infinite lifecycle mutation será error.

## DB-ORM-LIFECYCLE-074
PersistencePlan no se congelará antes de completar pre-lifecycle stabilization.

## DB-ORM-LIFECYCLE-075
preUpdate persistent mutations serán visibles al ChangeSet final.

## DB-ORM-LIFECYCLE-076
Empty final ChangeSet podrá eliminar UPDATE innecesario.

## DB-ORM-LIFECYCLE-077
postUpdate persistent mutation no causará hidden recursive UPDATE.

## DB-ORM-LIFECYCLE-078
postPersist persistent mutation no causará hidden recursive INSERT/UPDATE.

## DB-ORM-LIFECYCLE-079
Post-event persistent mutations podrán quedar pendientes para futuro flush.

## DB-ORM-LIFECYCLE-080
postLoad persistent mutation será considerada una nueva modificación bajo la política definida.

## DB-ORM-LIFECYCLE-081
Snapshot baseline deberá establecerse en un punto determinista respecto a postLoad.

## DB-ORM-LIFECYCLE-082
Lifecycle handler failure preservará original exception.

## DB-ORM-LIFECYCLE-083
Lifecycle diagnostics identificarán handler y event.

## DB-ORM-LIFECYCLE-084
Pre-event failure no será confundido con DB execution failure.

## DB-ORM-LIFECYCLE-085
Post-event failure no será confundido con DB operation failure.

## DB-ORM-LIFECYCLE-086
Persistence outcome y Lifecycle outcome serán dimensiones separadas.

## DB-ORM-LIFECYCLE-087
Transaction outcome será dimensión separada.

## DB-ORM-LIFECYCLE-088
Flush outcome podrá fallar aunque una operación DB individual haya tenido éxito.

## DB-ORM-LIFECYCLE-089
Lifecycle System no administrará transactions arbitrariamente.

## DB-ORM-LIFECYCLE-090
UNKNOWN persistence outcome preservará uncertainty.

## DB-ORM-LIFECYCLE-091
UNKNOWN outcome no producirá post-event de éxito.

## DB-ORM-LIFECYCLE-092
After-commit semantics pertenecerán a Transaction Synchronization.

## DB-ORM-LIFECYCLE-093
Lifecycle callbacks no reemplazarán Transactional Outbox.

## DB-ORM-LIFECYCLE-094
Recorded domain event no significará published domain event.

## DB-ORM-LIFECYCLE-095
Domain event publishing podrá ser transaction-aware.

## DB-ORM-LIFECYCLE-096
EventSystem integration será opcional.

## DB-ORM-LIFECYCLE-097
Database ORM no tendrá dependencia obligatoria de EventSystem.

## DB-ORM-LIFECYCLE-098
Observational handlers no modificarán ORM semantics.

## DB-ORM-LIFECYCLE-099
Mutating handlers solo operarán en mutation windows válidas.

## DB-ORM-LIFECYCLE-100
Lifecycle context aplicará least capability.

## DB-ORM-LIFECYCLE-101
Callbacks no recibirán EntityManager completo sin necesidad.

## DB-ORM-LIFECYCLE-102
Lifecycle context no expondrá raw SQL por defecto.

## DB-ORM-LIFECYCLE-103
Lifecycle context no expondrá commit/rollback por defecto.

## DB-ORM-LIFECYCLE-104
Singleton listeners no retendrán entities.

## DB-ORM-LIFECYCLE-105
Singleton listeners no retendrán EntityManager.

## DB-ORM-LIFECYCLE-106
Singleton listeners no retendrán transaction state.

## DB-ORM-LIFECYCLE-107
Request mutable listener state será scoped.

## DB-ORM-LIFECYCLE-108
Persistent runtime compartirá solo lifecycle definitions immutable.

## DB-ORM-LIFECYCLE-109
Current lifecycle invocation nunca será process-global.

## DB-ORM-LIFECYCLE-110
Current entity nunca será process-global.

## DB-ORM-LIFECYCLE-111
Current tenant nunca será process-global.

## DB-ORM-LIFECYCLE-112
Lifecycle callback stack será resettable.

## DB-ORM-LIFECYCLE-113
Request A lifecycle state no sobrevivirá a Request B.

## DB-ORM-LIFECYCLE-114
FrankenPHP será soportado mediante operation-scoped mutable state.

## DB-ORM-LIFECYCLE-115
RoadRunner/OpenSwoole deberán preservar logical scope isolation.

## DB-ORM-LIFECYCLE-116
Coroutine execution no compartirá invocation stack accidentalmente.

## DB-ORM-LIFECYCLE-117
Lifecycle no cambiará tenant durante flush silenciosamente.

## DB-ORM-LIFECYCLE-118
Lifecycle no cambiará shard durante flush silenciosamente.

## DB-ORM-LIFECYCLE-119
Lifecycle no cambiará connection target silenciosamente.

## DB-ORM-LIFECYCLE-120
Lifecycle no sustituirá Authorization System.

## DB-ORM-LIFECYCLE-121
Lifecycle identifier access no implicará autorización.

## DB-ORM-LIFECYCLE-122
Sensitive fields no aparecerán en diagnostics por defecto.

## DB-ORM-LIFECYCLE-123
Extension handlers deberán registrarse mediante contratos.

## DB-ORM-LIFECYCLE-124
Production LifecycleRegistry podrá congelarse.

## DB-ORM-LIFECYCLE-125
Runtime extension registration invalidará caches explícitamente cuando sea permitida.

## DB-ORM-LIFECYCLE-126
Extension collisions no usarán last-wins.

## DB-ORM-LIFECYCLE-127
Empty lifecycle fast path tendrá overhead mínimo.

## DB-ORM-LIFECYCLE-128
Lifecycle invocation no hará Reflection repetitiva por entidad.

## DB-ORM-LIFECYCLE-129
Lifecycle Event Mask podrá optimizar dispatch sin alterar semántica.

## DB-ORM-LIFECYCLE-130
Bulk SQL no disparará automáticamente entity callbacks.

## DB-ORM-LIFECYCLE-131
Direct Query Builder no disparará entity lifecycle.

## DB-ORM-LIFECYCLE-132
Raw SQL no disparará entity lifecycle.

## DB-ORM-LIFECYCLE-133
Repository bulk operations documentarán su lifecycle behavior.

## DB-ORM-LIFECYCLE-134
Soft delete lifecycle seguirá semántica ORM, no SQL físico.

## DB-ORM-LIFECYCLE-135
Temporal persistence lifecycle seguirá operación semántica.

## DB-ORM-LIFECYCLE-136
Cascade lifecycle respetará Persistence Graph.

## DB-ORM-LIFECYCLE-137
Cross-entity callback order no dependerá de collection insertion order.

## DB-ORM-LIFECYCLE-138
Persistence dependencies podrán influir en operation lifecycle ordering.

## DB-ORM-LIFECYCLE-139
Pre-events mutantes ocurrirán antes del freeze final del PersistencePlan.

## DB-ORM-LIFECYCLE-140
Post-events ocurrirán después del outcome semántico correspondiente.

## DB-ORM-LIFECYCLE-141
Cascade entities creadas en callback serán reanalizadas cuando la policy lo permita.

## DB-ORM-LIFECYCLE-142
Orphan removal usará remove lifecycle semántico.

## DB-ORM-LIFECYCLE-143
Hydration failure impedirá postLoad.

## DB-ORM-LIFECYCLE-144
Partial entity lifecycle será explícitamente gobernado.

## DB-ORM-LIFECYCLE-145
Lazy relation initialization no volverá a disparar postLoad del owner.

## DB-ORM-LIFECYCLE-146
Newly materialized related entities podrán recibir su propio postLoad.

## DB-ORM-LIFECYCLE-147
Lifecycle causality será observable.

## DB-ORM-LIFECYCLE-148
LifecycleInvocationId será operation-scoped.

## DB-ORM-LIFECYCLE-149
FlushCycleId permitirá correlación sin implicar commit.

## DB-ORM-LIFECYCLE-150
TransactionContextId no implicará transaction success.

## DB-ORM-LIFECYCLE-151
Lifecycle telemetry no modificará semantics.

## DB-ORM-LIFECYCLE-152
Metrics evitarán cardinalidad ilimitada.

## DB-ORM-LIFECYCLE-153
Slow lifecycle handlers serán diagnosticables.

## DB-ORM-LIFECYCLE-154
Queries realizadas desde callbacks serán observables.

## DB-ORM-LIFECYCLE-155
Lifecycle N+1 podrá ser detectado por Telemetry.

## DB-ORM-LIFECYCLE-156
Lifecycle handlers deberán evitar I/O innecesario.

## DB-ORM-LIFECYCLE-157
Mismos metadata y handlers producirán mismo invocation order.

## DB-ORM-LIFECYCLE-158
Lifecycle dispatch será determinista bajo inputs estructurales iguales.

## DB-ORM-LIFECYCLE-159
Lifecycle extensions no podrán romper invariantes core.

## DB-ORM-LIFECYCLE-160
Entity Lifecycle nunca afirmará commit cuando únicamente conoce éxito de una operación ORM.

---

# 234. Anti-pattern: email en postPersist

Incorrecto:

```php
#[PostPersist]
public function sendEmail(): void
{
    $mailer->send(...);
}
```

si el correo solo debe enviarse después del commit.

Correcto:

```text
postPersist
    ↓
record transactional effect
    ↓
COMMIT
    ↓
after-commit/outbox
    ↓
send email
```

---

# 235. Anti-pattern: flush recursivo

Incorrecto:

```php
public function preUpdate(
    PreUpdateEvent $event
): void {
    $event->entityManager()->flush();
}
```

---

# 236. Anti-pattern: SQL desde entidad

Incorrecto:

```php
#[PrePersist]
public function beforePersist(): void
{
    PDO::exec(...);
}
```

---

# 237. Anti-pattern: callback como autorización

Incorrecto:

```php
#[PreRemove]
public function authorizeDelete(): void
{
    // primary authorization mechanism
}
```

La autorización pertenece al Authorization System.

---

# 238. Anti-pattern: callback order accidental

Incorrecto:

```text
whichever Reflection returns first
```

---

# 239. Anti-pattern: postPersist = durable

Incorrecto:

```text
postPersist
→ publish irreversible external event
```

sin conocer commit.

---

# 240. Anti-pattern: global current entity

Incorrecto:

```php
LifecycleContext::$currentEntity = $entity;
```

en persistent runtime.

---

# 241. Anti-pattern: listener singleton mutable

Incorrecto:

```php
final class Listener
{
    private array $processedEntities = [];
}
```

si el servicio vive durante todo el worker.

---

# 242. Anti-pattern: entity callbacks en bulk SQL

Incorrecto esperar:

```text
UPDATE users SET active = false
```

→ `preUpdate()` por cada usuario.

---

# 243. Anti-pattern: postUpdate mutation loop

Incorrecto:

```text
postUpdate changes field
→ auto UPDATE
→ postUpdate changes field
→ auto UPDATE
→ ...
```

---

# 244. Flujo completo INSERT

```text
Entity NEW
    │
    ▼
UnitOfWork
    │
    ▼
PRE_PERSIST
    │
    ▼
Re-evaluate Persistent State
    │
    ▼
Change/Persistence Graph
    │
    ▼
Persistence Plan
    │
    ▼
INSERT Query Model
    │
    ▼
Query Engine
    │
    ▼
Execution
    │
    ├── FAILURE
    │
    ├── UNKNOWN
    │
    └── SUCCESS
            │
            ▼
    Generated Identity
            │
            ▼
    Entity State Reconciliation
            │
            ▼
        POST_PERSIST
```

---

# 245. Flujo completo UPDATE

```text
MANAGED Entity
      │
      ▼
Dirty Detection
      │
      ▼
Initial ChangeSet
      │
      ▼
PRE_UPDATE
      │
      ▼
Recompute ChangeSet
      │
      ├── empty → no UPDATE
      │
      ▼
Persistence Operation
      │
      ▼
Execution Outcome
      │
      ▼
State/Snapshot Reconciliation
      │
      ▼
POST_UPDATE
      │
      ▼
Post-event mutation?
      │
      └── yes → DIRTY for future flush
```

---

# 246. Flujo completo REMOVE

```text
MANAGED
   │
remove()
   ▼
REMOVED
   │
flush
   ▼
PRE_REMOVE
   │
   ▼
Persistence Plan
   │
   ▼
DELETE semantic operation
   │
   ▼
Execution Outcome
   │
   ├── UNKNOWN → reconciliation/recovery
   │
   └── SUCCESS
          │
          ▼
      DETACHED
          │
          ▼
      POST_REMOVE
```

---

# 247. Flujo de LOAD

```text
Result
  │
  ▼
Hydration Plan
  │
  ▼
EntityKey
  │
  ▼
IdentityMap
  │
  ├── existing
  │     └── reuse
  │
  └── new
        │
        ▼
   Entity Instantiation
        │
        ▼
   Persistent Hydration
        │
        ▼
   Snapshot
        │
        ▼
   MANAGED
        │
        ▼
   POST_LOAD
```

---

# 248. Flujo de FLUSH

```text
EntityManager::flush()
        │
        ▼
Reentrancy Guard
        │
        ▼
PRE_FLUSH
        │
        ▼
UnitOfWork Collection
        │
        ▼
Dirty Detection
        │
        ▼
Entity PRE Events
        │
        ▼
Lifecycle Stabilization
        │
        ├── changed → repeat bounded analysis
        │
        ▼
Freeze Persistence Plan
        │
        ▼
Execute Operations
        │
        ├── POST_PERSIST
        ├── POST_UPDATE
        └── POST_REMOVE
        │
        ▼
Reconcile State
        │
        ▼
POST_FLUSH
        │
        ▼
Flush Result
```

---

# 249. Relación con Transaction

```text
EntityManager::flush()
        │
        ▼
ORM Lifecycle
        │
        ▼
Persistence Operations
        │
        ▼
post*
        │
        ▼
postFlush
        │
        │
        ▼
Transaction still ACTIVE
        │
        ├── COMMIT
        │     ↓
        │   AfterCommit
        │
        └── ROLLBACK
              ↓
            Rollback Reconciliation
```

---

# 250. Fórmula de lifecycle

```text
EntityLifecycle
=
SemanticPersistencePhase
+
EntityContext
+
CompiledHandlers
+
DeterministicOrdering
+
ControlledMutationWindow
+
OutcomeAwareDispatch
```

---

# 251. Fórmula de pre-event

```text
PreLifecycleEvent
=
BeforeSemanticPersistenceOperation
∧
BeforeIrreversibleStateReconciliation
```

No significa necesariamente:

```text
immediately before SQL statement
```

---

# 252. Fórmula de post-event

```text
PostPersistenceLifecycleEvent
=
SemanticOperationSucceeded
∧
OutcomeCertaintySufficient
∧
ORMStateReconciled
```

pero:

```text
PostPersistenceLifecycleEvent
≠
TransactionCommitted
```

---

# 253. Fórmula de lifecycle stabilization

```text
StableUoW
=
LifecyclePassₙ(UoW)
where
LifecyclePassₙ(UoW) = LifecyclePassₙ₋₁(UoW)
```

bajo equivalencia estructural relevante.

---

# 254. Condición de planning

```text
PersistencePlanAllowed
=
UnitOfWorkCollected
∧
PreLifecycleCompleted
∧
LifecycleMutationsStabilized
∧
ChangeSetsRecomputed
∧
PersistenceGraphValid
```

---

# 255. Fórmula de post-event failure

```text
PostEventFailure
=
PersistenceOperationSucceeded
+
LifecycleHandlerFailed
```

No:

```text
PersistenceOperationFailed
```

---

# 256. Fórmula de after-commit

```text
AfterCommitEffectAllowed
=
TransactionCommitted
∧
RelevantPersistenceEffectRecorded
∧
EffectPolicySatisfied
```

No:

```text
postPersist fired
```

---

# 257. Fórmula de runtime safety

```text
SafeLifecycleRuntime
=
ImmutableCompiledMetadata
+
ImmutableInvocationPlans
+
ScopedInvocationState
+
ScopedReentrancyGuard
+
ScopedLifecycleContext
+
ScopedTransactionCorrelation
+
DeterministicReset
+
NoCrossRequestReferences
```

---

# 258. Master Formula

```text
Database Entity Lifecycle System
=
Lifecycle Event Model
+
Lifecycle Phases
+
Entity Callbacks
+
Entity Listeners
+
Lifecycle Subscribers
+
Compiled Lifecycle Metadata
+
Deterministic Handler Ordering
+
Lifecycle Dispatcher
+
Restricted Lifecycle Context
+
Mutation Windows
+
ChangeSet Recalculation
+
UnitOfWork Stabilization
+
Reentrancy Protection
+
Persistence Outcome Awareness
+
Failure Semantics
+
Transaction Boundary Separation
+
After-Commit Integration
+
Domain Event Separation
+
Persistent Runtime Isolation
+
Telemetry
+
Diagnostics
+
Extension Governance
```

---

# 259. Master Rule

> **En VoltStack, un lifecycle hook describe un momento semántico del trabajo ORM sobre una entidad; nunca debe interpretarse como evidencia de que la operación fue comprometida definitivamente en la base de datos, y cualquier efecto que requiera durabilidad deberá vincularse al resultado real de la transacción.**

---

# 260. Arquitectura final del bloque ORM

Con este documento queda cerrado el bloque arquitectónico inicial del ORM:

```text
112 ORM Architecture
       │
       ▼
113 Entity Model
       │
       ▼
114 Model API
       │
       ▼
115 Entity Metadata
       │
       ▼
116 Entity Mapping
       │
       ▼
117 Attribute Mapping
       │
       ▼
118 Entity Manager
       │
       ▼
119 Repository
       │
       ▼
120 Entity Query
       │
       ▼
121 Entity State
       │
       ▼
122 Entity Lifecycle
```

El bloque define ahora:

```text
What an Entity is
        +
How entities are exposed
        +
How entities are described
        +
How entities are mapped
        +
How entities are coordinated
        +
How entities are queried
        +
How ORM runtime state is represented
        +
How persistence lifecycle is observed
```

---

# 261. Transición al Persistence Core

El siguiente bloque entra en una parte crítica de VoltStack ORM:

```text
Identity Map
Unit of Work
Change Tracking
Snapshots
Persistence Engine
Persistence Planner
INSERT
UPDATE
DELETE
Flush
Batch Persistence
Consistency
```

Arquitectónicamente:

```text
ORM Public Layer
      │
      ▼
EntityManager
      │
      ▼
Entity State
      │
      ▼
IdentityMap
      │
      ▼
UnitOfWork
      │
      ▼
Change Tracking
      │
      ▼
Persistence Engine
      │
      ▼
Query Engine
```

---

# 262. Siguiente documento

```text
123_DATABASE_IDENTITY_MAP_SYSTEM.md
```

Este documento deberá formalizar:

```text
Identity Map architecture
EntityKey → Entity Instance
canonical managed instance
object identity vs entity identity
identity namespace
tenant/shard isolation
simple identifiers
composite identifiers
typed identifier normalization
generated identifiers
registration
lookup
collision detection
identity conflicts
proxy identity
inheritance identity
entity references
hydration integration
UnitOfWork integration
detach/clear
generated ID re-keying
rollback implications
memory management
WeakReference considerations
persistent runtime isolation
streaming behavior
diagnostics
telemetry
concurrency assumptions
```

con la invariante central:

> **Dentro de un mismo PersistenceContext, un `EntityKey` establecido puede corresponder como máximo a una única instancia canónica administrada.**