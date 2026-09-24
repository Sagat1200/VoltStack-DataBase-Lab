# 121_DATABASE_ENTITY_STATE_SYSTEM.md

# VoltStack Quantum Database
## Database Entity State System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 121 — Database Entity State System  
**Bloque:** 10 — ORM  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Entity State System` define cómo VoltStack representa, consulta y gobierna el **estado de persistencia runtime** de una instancia de entidad dentro de un `PersistenceContext`.

El sistema deberá distinguir estrictamente entre:

```text
Domain State
Identity State
Persistence State
Change Tracking State
Database State
Transaction State
```

Una entidad puede, por ejemplo:

- tener un identificador asignado;
- no existir todavía en la base de datos;
- estar registrada en un `EntityManager`;
- contener cambios locales;
- haber sido programada para eliminación;
- quedar detached;
- o encontrarse en un estado incierto después de un fallo de ejecución.

Principio central:

> **Entity State describe la relación runtime entre una instancia de entidad y su Persistence Context; no describe el estado de negocio de la entidad y no debe almacenarse como una bandera intrusiva dentro del objeto de dominio.**

---

# 2. Problema fundamental

Una instancia PHP:

```php
$user = new User(...);
```

no responde por sí sola preguntas como:

```text
¿Está administrada por el ORM?
¿Es nueva?
¿Fue cargada de la base de datos?
¿Tiene cambios pendientes?
¿Está programada para eliminarse?
¿Fue detached?
¿Su última operación de persistencia tuvo resultado cierto?
```

Estas propiedades pertenecen al ORM.

No al dominio.

Por tanto:

```text
Entity Object
    ≠
ORM Runtime State
```

---

# 3. Entity Domain State ≠ Persistence State

Ejemplo:

```php
$user->activate();
```

Esto modifica:

```text
Domain State
```

posiblemente:

```text
status = ACTIVE
```

Pero el ORM puede continuar considerando inicialmente:

```text
PersistenceState = MANAGED
ChangeTrackingState = NOT_EVALUATED
```

hasta ejecutar dirty checking.

Por tanto:

```text
Domain mutation
≠
immediate persistence-state transition
```

---

# 4. Identity State ≠ Persistence State

Una entidad puede poseer ID:

```php
$user = new User(
    id: UserId::generate()
);
```

y aun así no haber sido persistida.

Por tanto:

```text
IdentifierAssigned
≠
Managed
≠
Persisted
≠
DatabaseRowExists
```

---

# 5. Entity State ≠ ChangeSet

`EntityState` responde:

```text
What is this object's relationship to the persistence context?
```

`ChangeSet` responde:

```text
What persistent properties changed?
```

Por tanto:

```text
MANAGED
```

no significa necesariamente:

```text
DIRTY
```

---

# 6. Entity State ≠ Snapshot

Snapshot representa:

```text
previous known persistent field values
```

Entity State representa:

```text
ORM lifecycle relationship
```

---

# 7. Entity State ≠ Database State

El ORM puede creer:

```text
MANAGED
```

mientras otra transacción modifica la fila.

Por tanto:

```text
ORM State
≠
Current Database Reality
```

---

# 8. Entity State ≠ Transaction State

Una entidad puede estar:

```text
MANAGED
```

tanto:

```text
inside transaction
```

como:

```text
outside transaction
```

TransactionManager mantiene su propia máquina de estados.

---

# 9. Entity State ≠ Entity Lifecycle Callback

Estados runtime no equivalen a eventos:

```text
MANAGED ≠ postLoad
REMOVED ≠ preRemove
```

Los eventos se definirán en:

```text
122_DATABASE_ENTITY_LIFECYCLE_SYSTEM.md
```

---

# 10. Entity State ≠ Row Existence

No deberá existir la inferencia:

```text
has ID
→ row exists
```

ni:

```text
MANAGED
→ database row currently exists
```

---

# 11. Posición arquitectónica

```text
EntityManager
    │
    ├── IdentityMap
    ├── UnitOfWork
    ├── EntityStateRegistry
    ├── Snapshot Registry
    └── Change Tracking
            │
            ▼
        Entity Object
```

El `EntityStateRegistry` será una pieza del `PersistenceContext`.

---

# 12. Persistence Context

Conceptualmente:

```text
PersistenceContext
=
EntityManager Scope
+
IdentityMap
+
EntityStateRegistry
+
UnitOfWork
+
Snapshots
+
Pending Operations
```

No necesariamente será una única clase física.

---

# 13. Estado externo a la entidad

VoltStack deberá preferir:

```text
external ORM state
```

sobre:

```php
class User
{
    private bool $isManaged;
    private bool $isDirty;
    private bool $isRemoved;
}
```

Ese diseño sería incorrecto.

---

# 14. Razones

Mantener el estado fuera de la entidad permite:

- POPOs limpios;
- entidades independientes del ORM;
- testing sin Database;
- múltiples contextos controlados;
- mejor encapsulación;
- soporte para readonly entities;
- evitar serializar internals;
- persistent runtime isolation.

---

# 15. Taxonomía principal

El estado principal deberá mantenerse pequeño y semánticamente fuerte.

Se propone:

```php
enum EntityPersistenceState
{
    case NEW;
    case MANAGED;
    case REMOVED;
    case DETACHED;
    case UNKNOWN;
}
```

`DIRTY`, `CLEAN` y `READ_ONLY` no deberían mezclarse necesariamente dentro del mismo enum.

---

# 16. Por qué no un enum gigante

Un diseño como:

```text
NEW
MANAGED
CLEAN
DIRTY
REMOVED
DETACHED
READ_ONLY
LOCKED
HYDRATING
FLUSHING
FAILED
```

mezclaría dimensiones independientes.

Es preferible:

```text
PersistenceState
ChangeTrackingState
TrackingMode
IdentityState
OperationState
OutcomeCertainty
```

---

# 17. Modelo multidimensional

```text
EntityRuntimeState
├── PersistenceState
├── IdentityState
├── TrackingState
├── TrackingMode
├── SnapshotState
├── RemovalState
├── OutcomeCertainty
└── Runtime Flags
```

---

# 18. Persistence State

Estados canónicos:

```text
NEW
MANAGED
REMOVED
DETACHED
UNKNOWN
```

---

# 19. NEW

`NEW` representa una instancia que el ORM reconoce como candidata a persistencia pero que todavía no posee una persistencia confirmada dentro del contexto.

Ejemplo:

```php
$user = new User(...);

$entityManager->persist($user);
```

Antes de `persist()` la entidad puede incluso estar:

```text
UNTRACKED
```

Desde el punto de vista del PersistenceContext.

---

# 20. UNTRACKED

Conviene distinguir:

```text
UNTRACKED
```

de:

```text
NEW
```

Una instancia creada con:

```php
new User(...)
```

no tiene por qué ser conocida por el ORM.

---

# 21. UNTRACKED no necesariamente será un estado almacenado

Puede representarse como:

```text
absence from EntityStateRegistry
```

Es decir:

```text
registry.contains(entity) = false
→ UNTRACKED
```

---

# 22. NEW registrado

Después de:

```php
$entityManager->persist($user);
```

el contexto puede registrar:

```text
PersistenceState = NEW
```

---

# 23. `persist()` ≠ INSERT

La transición:

```text
UNTRACKED
→ NEW
```

no ejecuta necesariamente SQL.

---

# 24. MANAGED

`MANAGED` significa:

> La instancia pertenece actualmente al PersistenceContext y participa en las garantías de identidad, tracking y persistencia correspondientes a ese contexto.

---

# 25. MANAGED no significa CLEAN

Una entidad managed puede estar:

```text
CLEAN
DIRTY
NOT_EVALUATED
```

---

# 26. MANAGED no significa persisted recently

Una entidad cargada hace minutos sigue siendo:

```text
MANAGED
```

aunque la fila haya cambiado externamente.

---

# 27. REMOVED

`REMOVED` significa:

> La entidad permanece conocida por el PersistenceContext, pero ha sido programada para eliminación durante una futura sincronización.

Ejemplo:

```php
$entityManager->remove($user);
```

---

# 28. `remove()` ≠ DELETE

La transición:

```text
MANAGED
→ REMOVED
```

no implica inmediatamente:

```sql
DELETE
```

---

# 29. REMOVED conserva información

Antes del flush puede necesitarse conservar:

- EntityKey;
- snapshots;
- relationship dependencies;
- original values;
- lifecycle information;
- deletion plan inputs.

---

# 30. DETACHED

`DETACHED` significa:

> La instancia ya no participa en el PersistenceContext que previamente la administraba.

Puede ocurrir mediante:

```php
$entityManager->detach($user);
```

o:

```php
$entityManager->clear();
```

---

# 31. DETACHED ≠ deleted

Una entidad detached puede seguir existiendo perfectamente en la base de datos.

---

# 32. DETACHED ≠ invalid PHP object

El objeto continúa siendo utilizable como objeto de dominio.

Lo que pierde es su vínculo runtime con el ORM context.

---

# 33. UNKNOWN

`UNKNOWN` será un estado excepcional y conservador.

Representa situaciones donde VoltStack no puede afirmar con seguridad la relación de persistencia después de una operación incierta.

Ejemplo:

```text
INSERT sent
connection lost
commit outcome unknown
generated ID uncertain
```

---

# 34. UNKNOWN no se normaliza automáticamente

Nunca:

```text
UNKNOWN
→ assume MANAGED
```

ni:

```text
UNKNOWN
→ assume NEW
```

---

# 35. UNKNOWN requiere reconciliación

Puede requerir:

```text
refresh
explicit lookup
transaction recovery
connection recovery
manual reconciliation
EntityManager reset
```

según la situación.

---

# 36. EntityStateView

La API pública no deberá exponer necesariamente estructuras internas.

Se propone:

```php
final readonly class EntityStateView
{
    public function __construct(
        public EntityPersistenceState $persistence,
        public EntityTrackingState $tracking,
        public EntityTrackingMode $trackingMode,
        public EntityIdentityState $identity,
        public EntitySnapshotState $snapshot,
        public EntityOutcomeCertainty $certainty,
    ) {}
}
```

---

# 37. Estado derivado

Algunas propiedades podrán derivarse:

```text
isManaged
isNew
isRemoved
isDetached
isDirty
isReadOnly
isKnown
```

sin convertirse en estados independientes.

---

# 38. EntityTrackingState

Se propone:

```php
enum EntityTrackingState
{
    case NOT_APPLICABLE;
    case NOT_EVALUATED;
    case CLEAN;
    case DIRTY;
    case UNKNOWN;
}
```

---

# 39. CLEAN

Significa:

```text
no persistent differences detected
```

respecto al baseline conocido por el tracking strategy.

---

# 40. CLEAN ≠ database synchronized forever

Solo significa:

```text
according to current ORM tracking information
```

---

# 41. DIRTY

Significa:

```text
one or more persistent values differ
```

o que el tracking strategy ha marcado cambios explícitos.

---

# 42. DIRTY ≠ UPDATE already scheduled physically

El UnitOfWork/Persistence Planner decidirá posteriormente la operación.

---

# 43. NOT_EVALUATED

Importante para snapshot tracking.

Una entidad puede estar:

```text
MANAGED
+
NOT_EVALUATED
```

hasta dirty checking.

---

# 44. Tracking mode

Separar:

```php
enum EntityTrackingMode
{
    case SNAPSHOT;
    case EXPLICIT;
    case NOTIFY;
    case READ_ONLY;
    case CUSTOM;
}
```

---

# 45. READ_ONLY como modo

`READ_ONLY` encaja mejor como:

```text
TrackingMode
```

que como PersistenceState.

Una entidad puede ser:

```text
MANAGED + READ_ONLY
```

o:

```text
DETACHED + READ_ONLY
```

dependiendo de la estrategia.

---

# 46. Read-only managed entity

Puede participar en:

```text
IdentityMap
relationship resolution
identity consistency
```

pero no en:

```text
dirty checking
automatic persistence
```

---

# 47. Read-only mutation

Si el objeto PHP es mutable:

```php
$user->rename('X');
```

el ORM no deberá asumir que debe persistir el cambio.

---

# 48. EntityIdentityState

Sin duplicar el documento 113, el State System necesita consumir una vista de identidad.

Ejemplo:

```php
enum EntityIdentityState
{
    case UNASSIGNED;
    case ASSIGNED;
    case ESTABLISHED;
    case UNKNOWN;
}
```

---

# 49. UNASSIGNED

No existe aún un identificador persistente utilizable.

Frecuente con:

```text
database-generated identity
```

antes del INSERT.

---

# 50. ASSIGNED

Existe identificador lógico:

```text
UserId(...)
```

pero esto no prueba persistencia.

---

# 51. ESTABLISHED

El ORM tiene evidencia suficiente para considerar la identidad persistente establecida.

---

# 52. UNKNOWN identity

Puede aparecer después de:

```text
uncertain persistence outcome
```

especialmente con IDs generados por la base de datos.

---

# 53. Persistence State × Identity State

Ejemplos válidos:

| Persistence | Identity | Ejemplo |
|---|---|---|
| NEW | UNASSIGNED | auto-increment antes de INSERT |
| NEW | ASSIGNED | UUID generado por aplicación |
| MANAGED | ESTABLISHED | entidad cargada |
| REMOVED | ESTABLISHED | delete programado |
| DETACHED | ESTABLISHED | entidad desconectada |
| UNKNOWN | UNKNOWN | INSERT outcome incierto |

---

# 54. Combinaciones inválidas

Ejemplo:

```text
MANAGED + UNASSIGNED
```

puede ser inválido para una entidad ya persistida, salvo durante una fase interna controlada.

Las combinaciones deberán validarse mediante invariantes.

---

# 55. EntitySnapshotState

Se propone:

```php
enum EntitySnapshotState
{
    case NONE;
    case AVAILABLE;
    case STALE;
    case UNKNOWN;
}
```

---

# 56. Snapshot NONE

Puede ser correcto para:

```text
NEW entity
READ_ONLY entity
EXPLICIT tracking
DETACHED entity
```

según configuración.

---

# 57. Snapshot AVAILABLE

Existe baseline utilizable para change detection.

---

# 58. Snapshot STALE

Puede aparecer tras ciertas operaciones donde el baseline ya no representa el estado reconciliado.

---

# 59. Snapshot UNKNOWN

Se utiliza cuando la certeza del baseline se perdió.

---

# 60. EntityOutcomeCertainty

Se propone:

```php
enum EntityOutcomeCertainty
{
    case CERTAIN;
    case UNCERTAIN;
    case UNKNOWN;
}
```

---

# 61. Certainty ≠ Persistence State

Ejemplo:

```text
PersistenceState = UNKNOWN
OutcomeCertainty = UNKNOWN
```

es común, pero las dimensiones deben mantenerse separadas para diagnostics y recovery.

---

# 62. State machine principal

Modelo conceptual:

```text
                 persist()
   UNTRACKED ───────────────► NEW
      │                       │
      │ load/hydrate          │ successful flush
      │                       ▼
      └──────────────────► MANAGED
                              │
                       remove()│
                              ▼
                           REMOVED
                              │
                cancel remove │
                              ▼
                           MANAGED

MANAGED ── detach()/clear() ──► DETACHED

REMOVED ── detach()/clear() ──► DETACHED

NEW ── detach()/clear() ──────► DETACHED
```

Y cualquier operación incierta relevante puede conducir a:

```text
UNKNOWN
```

---

# 63. State transition ≠ method call

No todo método produce transición.

Ejemplo:

```php
$entityManager->persist($managedUser);
```

puede ser:

```text
MANAGED → MANAGED
```

idempotente.

---

# 64. StateTransition

Se propone:

```php
final readonly class EntityStateTransition
{
    public function __construct(
        public EntityPersistenceState $from,
        public EntityPersistenceState $to,
        public EntityStateTransitionReason $reason,
    ) {}
}
```

---

# 65. Transition reasons

```text
PERSIST
LOAD
HYDRATE
FLUSH_INSERT_SUCCESS
FLUSH_UPDATE_SUCCESS
REMOVE
CANCEL_REMOVE
FLUSH_DELETE_SUCCESS
DETACH
CLEAR
REFRESH
ROLLBACK_RECONCILIATION
EXECUTION_UNKNOWN
CLOSE_CONTEXT
CUSTOM
```

---

# 66. State transition validator

Se introduce:

```text
EntityStateTransitionValidator
```

para impedir transiciones imposibles.

---

# 67. EntityStateRegistry

Componente principal:

```text
EntityStateRegistry
```

Responsabilidades:

- asociar object identity con state record;
- registrar entidades;
- consultar estado;
- actualizar transiciones;
- eliminar tracking;
- limpiar scope;
- generar diagnostics.

---

# 68. Object identity

La registry deberá identificar la **instancia PHP**, no solo EntityKey.

Conceptualmente:

```text
ObjectIdentity
→ EntityStateRecord
```

---

# 69. PHP implementation

Posibles mecanismos:

```text
WeakMap<object, EntityStateRecord>
```

o una estructura equivalente.

`WeakMap` puede ser especialmente útil para evitar retenciones accidentales.

---

# 70. WeakMap no reemplaza IdentityMap

Importante:

```text
WeakMap
```

resuelve:

```text
object → state
```

Mientras:

```text
IdentityMap
```

resuelve:

```text
EntityKey → canonical managed object
```

Son índices diferentes.

---

# 71. EntityStateRecord

```php
final class EntityStateRecord
{
    public EntityPersistenceState $persistenceState;
    public EntityTrackingState $trackingState;
    public EntityTrackingMode $trackingMode;
    public EntityIdentityState $identityState;
    public EntitySnapshotState $snapshotState;
    public EntityOutcomeCertainty $outcomeCertainty;
}
```

La implementación real puede usar estructuras más especializadas.

---

# 72. StateRecord ≠ EntityMetadata

No almacenar metadata duplicada mutable por entidad.

Puede mantener referencias/IDs a metadata compilada immutable.

---

# 73. StateRegistry scope

Cada `EntityManager` tendrá su propio:

```text
EntityStateRegistry
```

o registry lógicamente aislado.

---

# 74. No global registry

Nunca:

```php
static WeakMap $allEntities;
```

a nivel de proceso.

Esto rompería FrankenPHP.

---

# 75. Registration

Registrar una entidad significa:

```text
PersistenceContext now knows this object
```

No necesariamente:

```text
database knows this object
```

---

# 76. Register NEW

```text
persist(entity)
    ↓
validate entity type
    ↓
resolve identity
    ↓
detect conflicting managed identity
    ↓
register state NEW
    ↓
register UoW intent
```

---

# 77. Register MANAGED

Hydration:

```text
database result
    ↓
EntityKey
    ↓
IdentityMap lookup
    ↓
instantiate/hydrate if needed
    ↓
register MANAGED
    ↓
create snapshot if tracking requires
```

---

# 78. Registration order

El orden exacto deberá impedir estados parciales observables.

Por ejemplo, no deberá quedar:

```text
IdentityMap contains entity
but StateRegistry does not
```

fuera de una operación interna controlada.

---

# 79. Atomic logical registration

Aunque no sea una transacción DB, la incorporación de una entidad al PersistenceContext deberá comportarse como una transición lógica coherente.

---

# 80. Entity identity conflict

Supongamos:

```text
IdentityMap:
User#10 → object A
```

y se intenta registrar:

```text
object B
EntityKey = User#10
```

El sistema deberá producir:

```text
EntityIdentityConflictException
```

salvo una operación explícita de merge que se defina en el futuro.

---

# 81. No silent replacement

Nunca:

```text
User#10 → A
```

se sustituirá silenciosamente por:

```text
User#10 → B
```

---

# 82. Persist transient entity

Ejemplo:

```php
$user = new User(...);

$entityManager->persist($user);
```

produce:

```text
UNTRACKED
→ NEW
```

---

# 83. Persist already NEW

```text
NEW
→ NEW
```

normalmente idempotente.

---

# 84. Persist already MANAGED

```text
MANAGED
→ MANAGED
```

sin duplicar UoW registration.

---

# 85. Persist REMOVED

Aquí debe definirse una política clara.

Una opción segura:

```text
REMOVED
→ MANAGED
```

como cancelación explícita de removal.

Pero no conviene que `persist()` esconda esa semántica.

---

# 86. Recomendación

Introducir explícitamente:

```php
$entityManager->cancelRemoval($entity);
```

o equivalente interno.

Así:

```text
persist(REMOVED)
```

puede ser:

```text
no-op / error / explicit policy
```

y no una restauración accidental.

---

# 87. Persist DETACHED

No deberá reattach automáticamente una entidad detached.

---

# 88. Razón

Un detached object puede contener:

```text
stale state
stale relationships
old version token
cross-context identity
```

---

# 89. Reattach

Si se soporta, deberá ser una operación explícita:

```text
merge
attach
reassociate
```

con semántica propia.

No formar parte implícita de `persist()`.

---

# 90. Persist UNKNOWN

Deberá rechazarse normalmente hasta reconciliación.

---

# 91. Remove NEW

Caso importante:

```text
NEW
→ remove()
```

No existe todavía necesariamente una fila.

Por tanto el resultado puede ser:

```text
NEW registration cancelled
→ DETACHED / UNTRACKED
```

sin generar DELETE.

---

# 92. Remove MANAGED

```text
MANAGED
→ REMOVED
```

---

# 93. Remove REMOVED

Idempotente:

```text
REMOVED
→ REMOVED
```

---

# 94. Remove DETACHED

Deberá fallar o requerir una operación explícita basada en identidad.

No debe asumirse que el objeto detached está actualizado.

---

# 95. Remove UNKNOWN

Deberá bloquearse hasta reconciliación.

---

# 96. Delete by identity

Una API:

```php
$repository->deleteById($id);
```

no necesita crear artificialmente una entidad managed.

Eso pertenece a una operación de persistencia/query específica.

---

# 97. Flush NEW success

Cuando un INSERT tiene resultado cierto:

```text
NEW
→ MANAGED
```

y:

```text
TrackingState → CLEAN
Snapshot → AVAILABLE
Identity → ESTABLISHED
Outcome → CERTAIN
```

según strategy.

---

# 98. Generated identifier

Para DB-generated IDs:

```text
NEW + UNASSIGNED
    ↓
INSERT succeeds
    ↓
generated identifier obtained
    ↓
identity assigned
    ↓
IdentityMap registration
    ↓
MANAGED + ESTABLISHED
```

---

# 99. Generated ID assignment ordering

Debe evitarse:

```text
state MANAGED
before EntityKey can be established
```

si IdentityMap requiere el ID.

---

# 100. Temporary UoW identity

Mientras una entidad NEW no tiene ID puede usar:

```text
TemporaryEntityToken
```

para construir dependency graphs.

---

# 101. Temporary token ≠ EntityKey

Nunca deberá exponerse como:

```text
persistent identity
```

---

# 102. INSERT unknown outcome

Caso crítico:

```text
INSERT sent
connection drops
```

Si no sabemos si ocurrió:

```text
NEW
→ UNKNOWN
```

o una representación equivalente.

Nunca:

```text
NEW
→ MANAGED
```

por optimismo.

---

# 103. Assigned ID and unknown insert

Aunque la entidad ya tenga UUID:

```text
Identifier = ASSIGNED
```

el resultado puede seguir siendo:

```text
PersistenceState = UNKNOWN
```

porque no sabemos si la fila existe.

---

# 104. Flush MANAGED update success

Si un UPDATE tiene resultado cierto:

```text
MANAGED
→ MANAGED
```

pero:

```text
DIRTY
→ CLEAN
```

y snapshot se actualiza.

---

# 105. UPDATE failure before effect

Si existe certeza de que no se ejecutó:

```text
MANAGED + DIRTY
→ MANAGED + DIRTY
```

---

# 106. UPDATE unknown outcome

Si pudo ejecutarse:

```text
MANAGED
→ UNKNOWN
```

o:

```text
MANAGED + outcome UNKNOWN
```

según el modelo final.

Recomendación conservadora:

```text
PersistenceState = UNKNOWN
```

cuando la confianza del contexto completo se haya perdido.

---

# 107. Flush REMOVED success

Después de DELETE confirmado:

```text
REMOVED
→ DETACHED
```

y se elimina de:

```text
IdentityMap
UnitOfWork managed set
Snapshot Registry
```

---

# 108. Deleted entity object

El objeto PHP sigue existiendo:

```php
$user->name();
```

pero ya no es managed.

---

# 109. Delete unknown outcome

```text
REMOVED
→ UNKNOWN
```

si no se conoce si la fila fue eliminada.

---

# 110. Flush order

State transitions finales deberán ocurrir solamente después de que el Persistence Engine determine un outcome suficientemente cierto.

---

# 111. State before physical execution

Durante flush puede existir estado interno:

```text
FLUSHING
INSERTING
UPDATING
DELETING
```

pero no necesariamente deben ser `EntityPersistenceState`.

---

# 112. Operation phase

Se propone una dimensión interna:

```php
enum EntityPersistenceOperationPhase
{
    case IDLE;
    case INSERTING;
    case UPDATING;
    case DELETING;
    case RECONCILING;
}
```

---

# 113. OperationPhase es transitorio

No deberá convertirse en un estado de dominio ni persistirse fuera del scope.

---

# 114. Reentrancy

Mientras:

```text
OperationPhase != IDLE
```

ciertas operaciones deberán rechazarse para evitar:

```text
recursive flush
state corruption
duplicate scheduling
```

---

# 115. Recursive flush

Un lifecycle callback no deberá poder provocar:

```text
flush()
→ callback
→ flush()
→ callback
```

sin una política explícita.

---

# 116. Dirty checking

El Entity State System consume resultados del futuro:

```text
125_DATABASE_CHANGE_TRACKING_SYSTEM.md
```

---

# 117. Dirty checking pipeline

```text
MANAGED
    ↓
Tracking Strategy
    ↓
Compare/Notification
    ↓
ChangeSet
    ↓
TrackingState
    ├── CLEAN
    └── DIRTY
```

---

# 118. Dirty state derived

Para `SNAPSHOT` tracking, `DIRTY` puede ser un estado derivado del ChangeSet.

No necesita mantenerse como una bandera manual siempre.

---

# 119. Explicit tracking

Con:

```text
EXPLICIT
```

el usuario o framework puede marcar:

```text
entity scheduled for dirty checking
```

sin que eso signifique que efectivamente exista un ChangeSet.

---

# 120. Notify tracking

Con:

```text
NOTIFY
```

la entidad/instrumentation comunica modificaciones.

Esto no debe obligar a las entidades estándar a implementar interfaces ORM.

---

# 121. Custom tracking

Extensiones podrán definir estrategias, pero deberán respetar:

```text
state invariants
snapshot contracts
UoW contracts
```

---

# 122. Refresh

```php
$entityManager->refresh($user);
```

deberá tener semántica explícita.

---

# 123. Refresh MANAGED CLEAN

Pipeline:

```text
load current DB representation
    ↓
hydrate into managed instance
    ↓
replace snapshot
    ↓
TrackingState = CLEAN
```

---

# 124. Refresh MANAGED DIRTY

Por defecto deberá ser explícitamente destructivo respecto a cambios locales.

Se recomienda exigir:

```text
force/explicit refresh policy
```

o documentar claramente que refresh descarta cambios locales.

---

# 125. Refresh DETACHED

No deberá reattach automáticamente.

---

# 126. Refresh REMOVED

Normalmente deberá rechazarse.

---

# 127. Refresh UNKNOWN

Puede formar parte de reconciliación, pero no debe asumir que una simple SELECT siempre resuelve todos los outcomes.

---

# 128. Clear

```php
$entityManager->clear();
```

deberá:

```text
detach managed entities
clear IdentityMap
clear StateRegistry
clear snapshots
clear UoW pending state
```

de manera coherente.

---

# 129. Clear ≠ close connection

No deberá cerrar conexiones no relacionadas solo por limpiar ORM state.

---

# 130. Clear ≠ flush

Nunca:

```text
clear()
→ implicit flush()
```

---

# 131. Clear with pending changes

Deberá existir policy.

Recomendación:

```text
explicitly discard pending ORM state
```

con diagnostics si hay cambios no flushed.

---

# 132. EntityManager close

Al cerrar el EntityManager:

```text
all managed entities become effectively detached
```

aunque no sea necesario mantener registros DETACHED después de destruir el contexto.

---

# 133. DETACHED tracking

No es necesario retener eternamente cada objeto detached.

Puede interpretarse:

```text
known previously but no longer managed
```

mientras haya una razón de diagnóstico.

Después del reset completo:

```text
absence from registry
```

es suficiente.

---

# 134. `contains()`

API conceptual:

```php
$entityManager->contains($entity);
```

deberá responder:

```text
true
```

solo cuando la instancia pertenece actualmente a ese PersistenceContext bajo las reglas definidas.

---

# 135. `contains()` and REMOVED

Debe definirse explícitamente.

Recomendación:

```text
REMOVED remains context-known
but contains() = false for "currently managed"
```

y ofrecer:

```text
stateOf()
```

para precisión.

---

# 136. Mejor API

```php
$entityManager->stateOf($entity);
```

podrá devolver:

```text
UNTRACKED
NEW
MANAGED
REMOVED
DETACHED
UNKNOWN
```

como vista pública simplificada.

---

# 137. Public EntityState

Puede existir:

```php
enum EntityState
{
    case UNTRACKED;
    case NEW;
    case MANAGED;
    case REMOVED;
    case DETACHED;
    case UNKNOWN;
}
```

mientras internamente:

```text
UNTRACKED = absence
```

---

# 138. Public vs internal state

Esto permite ergonomía sin obligar a almacenar `UNTRACKED`.

---

# 139. `isDirty()`

Podrá existir:

```php
$entityManager->isDirty($entity);
```

pero debe considerar tracking mode.

---

# 140. Dirty check side effects

Consultar `isDirty()` puede requerir comparación de snapshot.

Debe documentarse si:

```text
state inspection
```

puede activar dirty checking.

Recomendación:

```text
inspectCurrentTrackingState()
```

y:

```text
evaluateDirtyState()
```

como operaciones conceptualmente distintas.

---

# 141. State inspection ≠ state mutation

Las APIs de observación deberán evitar cambiar estado salvo cuando su contrato lo indique.

---

# 142. Snapshot interaction

```text
EntityStateRegistry
```

no debería almacenar necesariamente snapshots completos.

Eso pertenece a:

```text
EntitySnapshotSystem
```

del documento 126.

Solo necesita conocer su disponibilidad/status.

---

# 143. UnitOfWork interaction

```text
UnitOfWork
```

usa EntityState para determinar qué entidades participan en:

```text
insert
update
delete
```

---

# 144. StateRegistry does not plan persistence

No deberá decidir:

```text
SQL order
insert dependency order
transaction boundaries
```

---

# 145. Persistence Engine interaction

```text
State
+
ChangeSet
+
Relationships
+
Metadata
→ Persistence Planner
```

---

# 146. State reconciliation

Después de ejecutar:

```text
PersistencePlan
```

se genera:

```text
PersistenceOutcome
```

que deberá reconciliar EntityState.

---

# 147. EntityStateReconciler

Se propone:

```text
EntityStateReconciler
```

Responsabilidad:

```text
current state
+
operation
+
execution outcome
→ next state
```

---

# 148. Reconciliation table

| Before | Operation | Outcome | After |
|---|---|---|---|
| NEW | INSERT | SUCCESS | MANAGED |
| NEW | INSERT | NOT_EXECUTED | NEW |
| NEW | INSERT | UNKNOWN | UNKNOWN |
| MANAGED | UPDATE | SUCCESS | MANAGED/CLEAN |
| MANAGED | UPDATE | NOT_EXECUTED | MANAGED/DIRTY |
| MANAGED | UPDATE | UNKNOWN | UNKNOWN |
| REMOVED | DELETE | SUCCESS | DETACHED |
| REMOVED | DELETE | NOT_EXECUTED | REMOVED |
| REMOVED | DELETE | UNKNOWN | UNKNOWN |

---

# 149. Affected rows

`0 affected rows` no siempre significa lo mismo.

Puede indicar:

```text
no-op update
optimistic locking conflict
missing row
platform behavior
```

Debe interpretarse mediante persistence semantics.

---

# 150. State System no interpreta SQL row count aisladamente

El Persistence Engine deberá producir un outcome semántico.

---

# 151. Transaction rollback

Caso crítico:

```text
flush succeeds physically
transaction later rolls back
```

El estado ORM debe reconciliarse.

---

# 152. flush() ≠ commit()

Como ya se estableció:

```text
flush
≠
transaction commit
```

Por tanto:

```text
EntityState after flush
```

puede necesitar reconsideración tras rollback.

---

# 153. Transaction-aware state reconciliation

El ORM deberá integrarse con TransactionManager mediante:

```text
transaction synchronization
```

---

# 154. Post-flush state

Supongamos:

```text
NEW
→ INSERT
→ MANAGED
```

dentro de una transacción.

Si después ocurre:

```text
ROLLBACK
```

el ORM no puede fingir que el INSERT sigue persistido.

---

# 155. Rollback strategies

Posibles políticas:

```text
RESTORE_PRE_FLUSH_STATE
MARK_CONTEXT_TAINTED
DETACH_AFFECTED
REQUIRE_REFRESH
```

---

# 156. Recomendación inicial

Para máxima corrección:

```text
rollback affecting flushed ORM changes
→ restore state where deterministically possible
→ otherwise TAINT EntityManager
```

---

# 157. State journal

Para soportar restauración puede existir:

```text
EntityStateJournal
```

durante transaction-bound flush.

---

# 158. State journal contents

Puede registrar:

```text
previous PersistenceState
previous TrackingState
previous IdentityState
previous snapshot reference/version
generated identifier transition
IdentityMap mutations
```

---

# 159. Journal ≠ database transaction log

Es solo información ORM para reconciliación runtime.

---

# 160. Generated ID rollback

Caso:

```text
DB auto-increment ID = 42
INSERT
ROLLBACK
```

El objeto ya recibió:

```text
id = 42
```

No siempre es correcto simplemente volver a:

```text
null
```

especialmente si el dominio expuso el ID.

---

# 161. Generated identity rollback policy

Debe definirse por strategy.

Opciones:

```text
KEEP_ASSIGNED_BUT_NOT_ESTABLISHED
RESET_IF_SAFE
DETACH
TAINT_CONTEXT
CUSTOM
```

---

# 162. Identity mutation constraints

El State System no podrá violar las reglas del documento 113.

---

# 163. External database mutations

Ejemplo:

```php
DB::table('users')
    ->where('id', 10)
    ->update(...);
```

mientras `User#10` está managed.

El ORM state queda potencialmente stale.

---

# 164. External mutation policy

Debe existir una operación explícita:

```text
refresh
clear
invalidate
mark stale
```

---

# 165. No automatic magical synchronization

Entity State no vigilará toda modificación externa.

---

# 166. Bulk update

Ejemplo:

```text
UPDATE many Users
```

sin hydration.

Puede invalidar managed state.

---

# 167. Bulk mutation policies

Como se definió en ORM Architecture:

```text
CLEAR_AFFECTED
REFRESH_AFFECTED
REJECT_IF_MANAGED
ALLOW_STALE_EXPLICITLY
```

---

# 168. Stale state

Podría existir una dimensión:

```php
enum EntityFreshnessState
{
    case CURRENT_AS_KNOWN;
    case STALE;
    case UNKNOWN;
}
```

---

# 169. Freshness ≠ dirty

Una entidad puede ser:

```text
CLEAN
+
STALE
```

si no tiene cambios locales pero la DB fue modificada externamente.

---

# 170. Freshness model

Se recomienda dejarlo como metadata runtime opcional para no sobrecargar el state principal.

---

# 171. Version fields

Un optimistic lock version:

```text
version = 7
```

es parte del persistent state.

No forma parte del EntityPersistenceState.

---

# 172. Optimistic locking conflict

Si UPDATE produce conflicto:

```text
MANAGED + DIRTY
```

deberá permanecer no sincronizado.

Puede marcarse:

```text
CONFLICTED
```

en una dimensión de persistence outcome/concurrency, no convertirlo silenciosamente en CLEAN.

---

# 173. EntityConcurrencyState

Posible extensión futura:

```text
NORMAL
CONFLICTED
UNKNOWN
```

Los detalles pertenecen a:

```text
172_DATABASE_OPTIMISTIC_LOCKING_SYSTEM.md
```

---

# 174. Lazy loading

Una entidad:

```text
MANAGED
```

puede tener relaciones:

```text
UNINITIALIZED
```

---

# 175. Relationship loading state ≠ Entity state

No mezclar:

```text
EntityPersistenceState
```

con:

```text
RelationshipInitializationState
```

---

# 176. Partial entities

Una partial entity puede ser:

```text
MANAGED
```

pero con:

```text
FieldLoadState
```

parcial.

---

# 177. Field load state ≠ Entity persistence state

Esto será gobernado por Hydration System.

---

# 178. Proxy

Un proxy lazy puede representar una entidad:

```text
MANAGED
```

aunque aún no estén inicializados todos sus campos.

---

# 179. Proxy initialization ≠ state transition

```text
proxy uninitialized
→ initialized
```

no significa:

```text
NEW → MANAGED
```

---

# 180. Entity references

Un:

```text
EntityReference<User>
```

no tiene necesariamente `EntityPersistenceState`.

Es una referencia lógica, no una instancia managed.

---

# 181. EntityManager boundary

Una entidad managed por:

```text
EntityManager A
```

no es automáticamente managed por:

```text
EntityManager B
```

---

# 182. Cross-context state

```text
Managed(A)
≠
Managed(B)
```

---

# 183. Same object in two EntityManagers

Debe evitarse por defecto.

Aunque técnicamente PHP permita pasar la misma instancia, puede producir:

```text
conflicting snapshots
conflicting UnitOfWork
conflicting transactions
```

---

# 184. Cross-context registration

Recomendación:

```text
same object registered in another live PersistenceContext
→ CrossPersistenceContextEntityException
```

cuando pueda detectarse.

---

# 185. Context ownership

Podrá utilizarse un:

```text
EntityContextOwnershipRegistry
```

scope-aware o metadata auxiliar.

Debe evitar proceso-global mutable peligroso.

---

# 186. Tenant isolation

Una entidad managed bajo:

```text
Tenant A
```

no podrá migrar implícitamente a:

```text
Tenant B
```

---

# 187. EntityKey context

Como se estableció:

```text
EntityKey
=
IdentityNamespace
+
EntityType
+
CanonicalIdentifier
```

---

# 188. State context binding

`EntityStateRecord` deberá estar vinculado al mismo logical identity namespace que el EntityManager.

---

# 189. Cross-tenant attach

Debe producir:

```text
CrossContextEntityStateException
```

o equivalente.

---

# 190. Sharding

La misma regla aplica a:

```text
Shard A
Shard B
```

---

# 191. Read replicas

Cargar una entidad desde replica no cambia su EntityType.

Pero puede afectar:

```text
freshness
consistency metadata
```

---

# 192. Read-only replica entities

No deberán quedar configuradas de forma que flush accidental escriba de vuelta a una replica.

Write routing sigue siendo downstream.

---

# 193. Serialization

Nunca serializar:

```text
EntityStateRecord
UnitOfWork registration
snapshot pointers
EntityManager reference
IdentityMap state
```

dentro del objeto de dominio.

---

# 194. Queue serialization

Si una entidad se envía a un Job, se recomienda serializar:

```text
EntityReference / EntityKey
```

o datos explícitos.

No una entidad managed con su PersistenceContext.

---

# 195. Deserialization

Una entidad deserializada:

```text
does not become MANAGED automatically
```

---

# 196. Clone

Caso importante:

```php
$copy = clone $managedEntity;
```

---

# 197. Clone state

El clon no deberá heredar automáticamente:

```text
MANAGED
```

porque:

```text
ObjectIdentity changed
```

---

# 198. Clone identity

Si conserva el mismo ID, puede existir:

```text
EntityKey collision
```

al intentar persistirlo.

Debe detectarse.

---

# 199. Recommended clone semantics

```text
clone managed entity
→ UNTRACKED object
```

hasta que una API explícita determine su intención.

---

# 200. PHP `__clone()`

VoltStack no debería exigir que toda entidad implemente un hook ORM `__clone()`.

La detección deberá permanecer externa cuando sea posible.

---

# 201. Reflection

EntityState no dependerá de Reflection en runtime hot path para determinar estado.

Metadata compilada ya existe.

---

# 202. Performance

Operaciones frecuentes:

```text
stateOf(entity)
isManaged(entity)
trackingState(entity)
```

deberán aproximarse a:

```text
O(1)
```

---

# 203. IdentityMap lookup

```text
EntityKey → object
```

también deberá aproximarse a:

```text
O(1)
```

---

# 204. Dirty checking

No necesariamente O(1), porque depende de:

```text
field count
tracking strategy
relationship strategy
```

---

# 205. State memory overhead

El registro por entidad deberá mantenerse compacto.

No duplicar:

```text
full metadata
full mappings
compiled query plans
```

por cada entity instance.

---

# 206. Long-running jobs

Un job que procese:

```text
1,000,000 entities
```

no debe mantener todas managed.

Patrón:

```php
foreach ($stream as $index => $entity) {
    process($entity);

    if ($index % 500 === 0) {
        $entityManager->flush();
        $entityManager->clear();
    }
}
```

---

# 207. Streaming policies

Del ORM Architecture:

```text
TRACKED
DETACHED_AFTER_YIELD
READ_ONLY
SCALAR
PROJECTION
```

---

# 208. DETACHED_AFTER_YIELD

Puede ser útil para evitar crecimiento de:

```text
IdentityMap
StateRegistry
Snapshots
UnitOfWork
```

---

# 209. Persistent runtime

En FrankenPHP:

```text
worker
├── Request A
│   └── EntityManager A
│
├── reset
│
└── Request B
    └── EntityManager B
```

---

# 210. Prohibición crítica

Nunca:

```text
Request A managed entities
→ Request B EntityStateRegistry
```

---

# 211. Runtime reset

Al final de request/job:

```text
discard pending ORM operation state
clear IdentityMap
clear StateRegistry
clear snapshots
clear UnitOfWork
clear transaction-bound journals
clear temporary entity tokens
```

según lifecycle contract.

---

# 212. No implicit flush on reset

Nunca:

```text
request end
→ automatic flush
```

como mecanismo de limpieza.

---

# 213. Pending changes at reset

Deben:

```text
be discarded
+
optionally diagnosed
```

pero no persistidos inesperadamente.

---

# 214. EntityManager taint

El EntityManager podrá entrar en:

```text
TAINTED
```

cuando no pueda garantizar coherencia.

---

# 215. Entity state UNKNOWN vs Manager TAINTED

Son diferentes.

```text
Entity UNKNOWN
```

afecta una entidad.

```text
EntityManager TAINTED
```

significa que el PersistenceContext completo ya no es confiable para ciertas operaciones.

---

# 216. Escalation

Un único UNKNOWN puede requerir:

```text
EntityManager → TAINTED
```

si afecta:

```text
transaction
IdentityMap
generated identity
relationship graph
flush ordering
```

---

# 217. Tainted manager behavior

Deberá rechazar:

```text
flush
persist
remove
possibly managed queries
```

según severidad.

---

# 218. Recovery

La recuperación preferida puede ser:

```text
close/clear context
create new EntityManager
re-read durable state
```

en lugar de intentar reparar arbitrariamente memoria corrupta.

---

# 219. State diagnostics

Se propone:

```php
$state = $entityManager->inspect($user);
```

Resultado:

```text
Entity: App\Entity\User
Object: #0x...
EntityKey: User#42
Persistence: MANAGED
Tracking: DIRTY
TrackingMode: SNAPSHOT
Identity: ESTABLISHED
Snapshot: AVAILABLE
Outcome: CERTAIN
Context: request:8f31...
```

---

# 220. Sensitive diagnostics

Los identificadores podrán ser:

```text
redacted
hashed
truncated
```

según telemetry/security policy.

---

# 221. State history

No se requiere mantener historial completo de todas las transiciones en producción.

Pero puede habilitarse en:

```text
debug
testing
telemetry
```

---

# 222. EntityStateTransitionLog

Opcional:

```text
NEW
→ MANAGED
→ REMOVED
→ DETACHED
```

con razones y timestamps.

---

# 223. Transition log ≠ audit log

Es debugging ORM.

No sustituye Database Audit System.

---

# 224. Events

State transitions podrán emitir eventos internos:

```text
EntityStateChanging
EntityStateChanged
```

pero deberán gobernarse cuidadosamente para no permitir corrupción/reentrancy.

---

# 225. State transition events

Se recomienda que sean principalmente:

```text
observational
```

y no permitan cancelar invariantes fundamentales.

---

# 226. Lifecycle events

Los eventos de dominio/persistencia se definirán en el siguiente documento.

---

# 227. State extension

Podrá existir:

```text
EntityStateExtension
```

para añadir metadata runtime.

Pero no deberá redefinir estados fundamentales arbitrariamente.

---

# 228. Extension metadata

Ejemplos:

```text
temporal version context
soft-delete marker
custom synchronization marker
domain-specific persistence hint
```

---

# 229. Extension state namespace

Cada extensión deberá usar:

```text
stable extension ID
```

para evitar colisiones.

---

# 230. State machine extension restriction

No permitir:

```text
plugin defines NEW → RANDOM_STATE → MANAGED
```

rompiendo UnitOfWork.

Las extensiones podrán añadir dimensiones, no alterar invariantes core sin un contrato formal.

---

# 231. Error hierarchy

```text
DatabaseOrmException
└── EntityStateException
    ├── UnknownEntityStateException
    ├── InvalidEntityStateTransitionException
    ├── EntityNotManagedException
    ├── EntityAlreadyManagedException
    ├── EntityRemovedException
    ├── EntityDetachedException
    ├── EntityStateUnknownException
    ├── EntityIdentityStateException
    ├── EntityIdentityConflictException
    ├── EntityTrackingStateException
    ├── EntitySnapshotStateException
    ├── EntityReadOnlyStateException
    ├── CrossPersistenceContextEntityException
    ├── CrossContextEntityStateException
    ├── EntityStateReconciliationException
    ├── EntityStateRollbackException
    ├── EntityManagerTaintedException
    ├── EntityStateRegistryException
    └── EntityStateInvariantException
```

---

# 232. Invalid transition diagnostic

```text
DB-ENTITY-STATE-TRANSITION-001

Entity:
App\Entity\User

Current state:
DETACHED

Requested operation:
remove()

Reason:
A detached entity cannot be scheduled for removal through
this PersistenceContext.

Suggested actions:
- reload the entity
- delete explicitly by identifier
- use an explicit reassociation mechanism
```

---

# 233. Identity conflict diagnostic

```text
DB-ENTITY-STATE-IDENTITY-002

Entity:
App\Entity\User

EntityKey:
User#42

Managed object:
object#812

Attempted object:
object#991

Reason:
Two different object instances cannot simultaneously represent
the same EntityKey inside one PersistenceContext.
```

---

# 234. Unknown outcome diagnostic

```text
DB-ENTITY-STATE-OUTCOME-003

Entity:
App\Entity\Order

Previous state:
NEW

Operation:
INSERT

Outcome:
UNKNOWN

Connection:
Lost after statement dispatch.

Action:
Entity state cannot be safely promoted to MANAGED.
PersistenceContext has been marked TAINTED.
```

---

# 235. Arquitectura de clases propuesta

```text
src/Quantum/Database/ORM/State/
│
├── Contract/
│   ├── EntityStateRegistry.php
│   ├── EntityStateInspector.php
│   ├── EntityStateReconciler.php
│   ├── EntityStateTransitionValidator.php
│   └── EntityStateExtension.php
│
├── State/
│   ├── EntityState.php
│   ├── EntityPersistenceState.php
│   ├── EntityTrackingState.php
│   ├── EntityTrackingMode.php
│   ├── EntityIdentityState.php
│   ├── EntitySnapshotState.php
│   ├── EntityFreshnessState.php
│   ├── EntityOutcomeCertainty.php
│   └── EntityPersistenceOperationPhase.php
│
├── Record/
│   ├── EntityStateRecord.php
│   └── EntityStateView.php
│
├── Registry/
│   └── DefaultEntityStateRegistry.php
│
├── Transition/
│   ├── EntityStateTransition.php
│   ├── EntityStateTransitionReason.php
│   ├── DefaultEntityStateTransitionValidator.php
│   └── EntityStateTransitionTable.php
│
├── Reconciliation/
│   ├── DefaultEntityStateReconciler.php
│   ├── EntityPersistenceOutcome.php
│   └── EntityReconciliationResult.php
│
├── Operation/
│   ├── EntityOperationState.php
│   └── EntityOperationGuard.php
│
├── Transaction/
│   ├── EntityStateJournal.php
│   ├── EntityStateJournalEntry.php
│   └── EntityTransactionStateSynchronizer.php
│
├── Runtime/
│   ├── EntityStateResetter.php
│   └── EntityContextOwnership.php
│
├── Extension/
│   ├── EntityStateExtensionRegistry.php
│   └── EntityStateExtensionData.php
│
├── Telemetry/
│   ├── EntityStateTelemetry.php
│   └── EntityStateTransitionLog.php
│
└── Exception/
    └── ...
```

---

# 236. Relación con IdentityMap

```text
EntityStateRegistry
object → state

IdentityMap
EntityKey → object
```

Invariante:

```text
MANAGED entity with established identity
→ IdentityMap must resolve to same object
```

---

# 237. Relación con UnitOfWork

```text
EntityState
      ↓
UnitOfWork
      ↓
Change Tracking
      ↓
Persistence Planning
```

El State System no reemplaza UnitOfWork.

---

# 238. Relación con Snapshot System

```text
EntityStateRecord
    ↓
snapshot status/reference
    ↓
EntitySnapshotSystem
```

No duplicar snapshots.

---

# 239. Relación con Change Tracking

```text
MANAGED
+
TrackingMode
+
Snapshot/Notifications
    ↓
Change Tracking
    ↓
CLEAN / DIRTY
```

---

# 240. Relación con Persistence Engine

```text
NEW      → INSERT candidate
MANAGED  → UPDATE candidate if ChangeSet exists
REMOVED  → DELETE candidate
DETACHED → no automatic persistence
UNKNOWN  → block/reconcile
```

---

# 241. Relación con TransactionManager

```text
Persistence outcome
    ↓
Entity State reconciliation
    ↓
Transaction final outcome
    ↓
final ORM reconciliation
```

---

# 242. Relación con EntityManager

EntityManager será la API coordinadora:

```php
$entityManager->persist($entity);
$entityManager->remove($entity);
$entityManager->detach($entity);
$entityManager->refresh($entity);
$entityManager->clear();

$entityManager->stateOf($entity);
$entityManager->contains($entity);
```

pero delegará a los componentes correspondientes.

---

# 243. State transition matrix

| Current | persist | remove | detach | refresh | flush |
|---|---|---|---|---|---|
| UNTRACKED | NEW | error | no-op/error | error | n/a |
| NEW | NEW | cancel NEW | DETACHED | error | INSERT |
| MANAGED | MANAGED | REMOVED | DETACHED | refresh | detect/update |
| REMOVED | explicit policy | REMOVED | DETACHED | error | DELETE |
| DETACHED | error/explicit attach | error | DETACHED | error | n/a |
| UNKNOWN | error | error | allowed cleanup | reconcile only | error |

La tabla final podrá ajustarse con las APIs definitivas, pero las transiciones deberán permanecer explícitas.

---

# 244. Invariantes arquitectónicas

## DB-ENTITY-STATE-001
Entity State será estado ORM runtime.

## DB-ENTITY-STATE-002
Entity State no será Domain State.

## DB-ENTITY-STATE-003
Entity State no será Entity Identity.

## DB-ENTITY-STATE-004
Entity State no será ChangeSet.

## DB-ENTITY-STATE-005
Entity State no será Snapshot.

## DB-ENTITY-STATE-006
Entity State no será Database State.

## DB-ENTITY-STATE-007
Entity State no será Transaction State.

## DB-ENTITY-STATE-008
Entity State no será Lifecycle Event.

## DB-ENTITY-STATE-009
Entity State permanecerá fuera del objeto de dominio por defecto.

## DB-ENTITY-STATE-010
Las entidades no requerirán flags ORM internos.

## DB-ENTITY-STATE-011
UNTRACKED podrá representarse como ausencia del registry.

## DB-ENTITY-STATE-012
NEW significará registrado para persistencia pero no confirmado como persistido.

## DB-ENTITY-STATE-013
MANAGED significará pertenencia al PersistenceContext.

## DB-ENTITY-STATE-014
MANAGED no significará CLEAN.

## DB-ENTITY-STATE-015
MANAGED no garantizará current database equality.

## DB-ENTITY-STATE-016
REMOVED significará scheduled for removal.

## DB-ENTITY-STATE-017
remove() no ejecutará DELETE necesariamente.

## DB-ENTITY-STATE-018
DETACHED no significará deleted.

## DB-ENTITY-STATE-019
DETACHED object seguirá siendo un objeto de dominio válido.

## DB-ENTITY-STATE-020
UNKNOWN será first-class.

## DB-ENTITY-STATE-021
UNKNOWN nunca se convertirá implícitamente en éxito.

## DB-ENTITY-STATE-022
UNKNOWN requerirá reconciliación o reset.

## DB-ENTITY-STATE-023
Persistence State será distinto de Tracking State.

## DB-ENTITY-STATE-024
CLEAN/DIRTY no deberán mezclarse obligatoriamente con Persistence State.

## DB-ENTITY-STATE-025
TrackingMode será dimensión independiente.

## DB-ENTITY-STATE-026
READ_ONLY será preferentemente TrackingMode.

## DB-ENTITY-STATE-027
Read-only no implicará PHP immutability.

## DB-ENTITY-STATE-028
Identity State será dimensión independiente.

## DB-ENTITY-STATE-029
IdentifierAssigned no implicará persistence.

## DB-ENTITY-STATE-030
Established identity requerirá evidencia suficiente.

## DB-ENTITY-STATE-031
Generated identifier podrá estar unavailable para NEW entity.

## DB-ENTITY-STATE-032
Snapshot State será dimensión independiente.

## DB-ENTITY-STATE-033
Outcome Certainty será dimensión independiente.

## DB-ENTITY-STATE-034
State transitions serán explícitas.

## DB-ENTITY-STATE-035
State transitions serán validadas.

## DB-ENTITY-STATE-036
State transition reason será observable.

## DB-ENTITY-STATE-037
EntityStateRegistry indexará por object identity.

## DB-ENTITY-STATE-038
IdentityMap indexará por EntityKey.

## DB-ENTITY-STATE-039
EntityStateRegistry no reemplazará IdentityMap.

## DB-ENTITY-STATE-040
IdentityMap no reemplazará EntityStateRegistry.

## DB-ENTITY-STATE-041
WeakMap podrá usarse para state storage.

## DB-ENTITY-STATE-042
WeakMap no será identidad persistente.

## DB-ENTITY-STATE-043
State registry será scope-local.

## DB-ENTITY-STATE-044
No existirá process-global mutable entity registry.

## DB-ENTITY-STATE-045
Registration no implicará database write.

## DB-ENTITY-STATE-046
persist(UNTRACKED) podrá producir NEW.

## DB-ENTITY-STATE-047
persist() no implicará INSERT.

## DB-ENTITY-STATE-048
persist(NEW) será idempotente.

## DB-ENTITY-STATE-049
persist(MANAGED) no duplicará registration.

## DB-ENTITY-STATE-050
persist(DETACHED) no reattachará implícitamente.

## DB-ENTITY-STATE-051
persist(UNKNOWN) será rechazado por defecto.

## DB-ENTITY-STATE-052
Reattachment será explícito.

## DB-ENTITY-STATE-053
Dos objetos con mismo EntityKey no podrán ser managed simultáneamente en un mismo contexto.

## DB-ENTITY-STATE-054
IdentityMap entries no serán reemplazadas silenciosamente.

## DB-ENTITY-STATE-055
remove(MANAGED) producirá REMOVED.

## DB-ENTITY-STATE-056
remove(REMOVED) será idempotente.

## DB-ENTITY-STATE-057
remove(NEW) no requerirá DELETE si nunca hubo persistencia.

## DB-ENTITY-STATE-058
remove(DETACHED) no será implícitamente permitido.

## DB-ENTITY-STATE-059
remove(UNKNOWN) requerirá reconciliación.

## DB-ENTITY-STATE-060
Successful INSERT reconciliará NEW hacia MANAGED.

## DB-ENTITY-STATE-061
Successful UPDATE mantendrá MANAGED.

## DB-ENTITY-STATE-062
Successful UPDATE reconciliará tracking baseline.

## DB-ENTITY-STATE-063
Successful DELETE producirá DETACHED.

## DB-ENTITY-STATE-064
Physical execution failure sin efectos preservará estado previo cuando sea demostrable.

## DB-ENTITY-STATE-065
Unknown INSERT outcome no producirá MANAGED.

## DB-ENTITY-STATE-066
Unknown UPDATE outcome no producirá CLEAN.

## DB-ENTITY-STATE-067
Unknown DELETE outcome no producirá DETACHED como si fuera éxito.

## DB-ENTITY-STATE-068
Generated identifier se asignará mediante mecanismo controlado.

## DB-ENTITY-STATE-069
TemporaryEntityToken no será EntityKey.

## DB-ENTITY-STATE-070
TemporaryEntityToken no será serializado como persistent identifier.

## DB-ENTITY-STATE-071
EntityState final se actualizará después de outcome semántico suficiente.

## DB-ENTITY-STATE-072
Statement dispatch no implicará persistence success.

## DB-ENTITY-STATE-073
Operation phase será distinto de persistence state.

## DB-ENTITY-STATE-074
Recursive flush estará gobernado.

## DB-ENTITY-STATE-075
Dirty checking no será responsabilidad exclusiva del StateRegistry.

## DB-ENTITY-STATE-076
DIRTY podrá ser estado derivado.

## DB-ENTITY-STATE-077
Tracking strategy determinará cómo detectar cambios.

## DB-ENTITY-STATE-078
Custom tracking respetará invariantes core.

## DB-ENTITY-STATE-079
refresh() será explícito.

## DB-ENTITY-STATE-080
Refresh de dirty entity no descartará cambios silenciosamente.

## DB-ENTITY-STATE-081
Refresh no reattachará detached entity automáticamente.

## DB-ENTITY-STATE-082
clear() no ejecutará flush implícito.

## DB-ENTITY-STATE-083
clear() limpiará IdentityMap.

## DB-ENTITY-STATE-084
clear() limpiará StateRegistry.

## DB-ENTITY-STATE-085
clear() limpiará UnitOfWork state.

## DB-ENTITY-STATE-086
clear() limpiará snapshots scope-local.

## DB-ENTITY-STATE-087
clear() no cerrará conexiones arbitrariamente.

## DB-ENTITY-STATE-088
Pending changes podrán descartarse solamente bajo policy explícita.

## DB-ENTITY-STATE-089
Detached entities no necesitarán retención permanente en registry.

## DB-ENTITY-STATE-090
contains() tendrá semántica definida.

## DB-ENTITY-STATE-091
stateOf() será más preciso que contains().

## DB-ENTITY-STATE-092
State inspection no cambiará estado silenciosamente.

## DB-ENTITY-STATE-093
Dirty evaluation y dirty inspection serán distinguibles.

## DB-ENTITY-STATE-094
StateRegistry no duplicará snapshots completos.

## DB-ENTITY-STATE-095
StateRegistry no planificará SQL.

## DB-ENTITY-STATE-096
StateRegistry no decidirá transaction boundaries.

## DB-ENTITY-STATE-097
Persistence outcome será semántico.

## DB-ENTITY-STATE-098
Row count aislado no determinará EntityState.

## DB-ENTITY-STATE-099
flush() no será commit().

## DB-ENTITY-STATE-100
Transaction rollback podrá requerir state reconciliation.

## DB-ENTITY-STATE-101
ORM deberá conocer transaction final outcome cuando haya flush transaction-bound.

## DB-ENTITY-STATE-102
State journal podrá soportar rollback reconciliation.

## DB-ENTITY-STATE-103
State journal no será DB transaction log.

## DB-ENTITY-STATE-104
Generated ID rollback seguirá identity strategy.

## DB-ENTITY-STATE-105
Rollback no violará identifier immutability silenciosamente.

## DB-ENTITY-STATE-106
External DB mutation podrá volver stale el managed state.

## DB-ENTITY-STATE-107
ORM no detectará mágicamente todas las external mutations.

## DB-ENTITY-STATE-108
Bulk mutations tendrán managed-state policy explícita.

## DB-ENTITY-STATE-109
Freshness será distinto de dirty state.

## DB-ENTITY-STATE-110
Optimistic locking conflict no producirá CLEAN.

## DB-ENTITY-STATE-111
Concurrency conflict será representable.

## DB-ENTITY-STATE-112
Relationship loading state será distinto de EntityPersistenceState.

## DB-ENTITY-STATE-113
Partial field loading será distinto de EntityPersistenceState.

## DB-ENTITY-STATE-114
Proxy initialization no será persistence transition.

## DB-ENTITY-STATE-115
EntityReference no será managed entity.

## DB-ENTITY-STATE-116
Managed state será relativo a un PersistenceContext.

## DB-ENTITY-STATE-117
Managed(A) no implicará Managed(B).

## DB-ENTITY-STATE-118
Same object multi-context management será rechazado o estrictamente gobernado.

## DB-ENTITY-STATE-119
Cross-context state transfer será explícito.

## DB-ENTITY-STATE-120
Tenant identity namespace formará parte de context correctness cuando aplique.

## DB-ENTITY-STATE-121
Cross-tenant implicit attach será rechazado.

## DB-ENTITY-STATE-122
Shard context será respetado.

## DB-ENTITY-STATE-123
Replica origin no redefinirá EntityType.

## DB-ENTITY-STATE-124
Replica reads no autorizarán writes hacia replica.

## DB-ENTITY-STATE-125
ORM state no será serializado dentro de entidades.

## DB-ENTITY-STATE-126
EntityManager references no serán serializadas dentro de entidades.

## DB-ENTITY-STATE-127
Queue serialization no conservará managed state.

## DB-ENTITY-STATE-128
Deserialization no producirá MANAGED automáticamente.

## DB-ENTITY-STATE-129
Cloning no heredará managed state.

## DB-ENTITY-STATE-130
Clone con mismo ID podrá producir identity conflict al registrarse.

## DB-ENTITY-STATE-131
VoltStack no exigirá __clone ORM hook a todas las entidades.

## DB-ENTITY-STATE-132
State lookup hot path deberá aproximarse a O(1).

## DB-ENTITY-STATE-133
Per-entity state storage deberá ser compacto.

## DB-ENTITY-STATE-134
Metadata compilada no será duplicada por entity state record.

## DB-ENTITY-STATE-135
Long-running processing deberá poder clear/detach entidades.

## DB-ENTITY-STATE-136
Streaming podrá usar DETACHED_AFTER_YIELD.

## DB-ENTITY-STATE-137
Request A state no sobrevivirá a Request B.

## DB-ENTITY-STATE-138
Persistent worker reset será determinista.

## DB-ENTITY-STATE-139
Runtime reset no ejecutará implicit flush.

## DB-ENTITY-STATE-140
Pending changes al reset no serán persistidos accidentalmente.

## DB-ENTITY-STATE-141
EntityManager TAINTED será distinto de Entity UNKNOWN.

## DB-ENTITY-STATE-142
Unknown entity outcome podrá taint EntityManager.

## DB-ENTITY-STATE-143
Tainted EntityManager restringirá operaciones peligrosas.

## DB-ENTITY-STATE-144
Recovery podrá requerir un nuevo PersistenceContext.

## DB-ENTITY-STATE-145
State diagnostics serán estructurados.

## DB-ENTITY-STATE-146
Sensitive identifiers podrán redactarse.

## DB-ENTITY-STATE-147
Transition history será opcional.

## DB-ENTITY-STATE-148
Transition history no será audit log.

## DB-ENTITY-STATE-149
State events no podrán romper invariantes core.

## DB-ENTITY-STATE-150
Extensions no redefinirán arbitrariamente la state machine.

## DB-ENTITY-STATE-151
Extension state utilizará namespaces estables.

## DB-ENTITY-STATE-152
Same logical state input producirá reconciliación determinista.

## DB-ENTITY-STATE-153
Persistence State nunca se inferirá exclusivamente por presencia de ID.

## DB-ENTITY-STATE-154
Persistence State nunca se inferirá exclusivamente por igualdad de fields.

## DB-ENTITY-STATE-155
Database row existence nunca se asumirá por MANAGED sin considerar la naturaleza del contexto.

## DB-ENTITY-STATE-156
Entity state transitions serán operation-scoped.

## DB-ENTITY-STATE-157
State registry mutable nunca será process-shared.

## DB-ENTITY-STATE-158
Shared ORM state será únicamente immutable/configuracional.

## DB-ENTITY-STATE-159
Entity State será consumido por UnitOfWork, no lo sustituirá.

## DB-ENTITY-STATE-160
Toda reconciliación de Entity State deberá preservar explícitamente la certeza real del resultado de persistencia.

---

# 245. Anti-pattern: ORM flags en la entidad

Incorrecto:

```php
class User
{
    public bool $ormManaged = false;
    public bool $ormDirty = false;
}
```

---

# 246. Anti-pattern: ID significa persisted

Incorrecto:

```php
if ($entity->id !== null) {
    return EntityState::MANAGED;
}
```

---

# 247. Anti-pattern: dirty dentro de persistence state

Incorrecto:

```text
NEW
CLEAN
DIRTY
REMOVED
```

como una única dimensión.

---

# 248. Anti-pattern: failure = NEW

Incorrecto:

```text
INSERT connection failure
→ NEW
```

si el INSERT pudo haberse aplicado.

Correcto:

```text
→ UNKNOWN
```

---

# 249. Anti-pattern: flush = commit

Incorrecto:

```text
flush succeeded
→ entity permanently synchronized
```

sin considerar transaction rollback.

---

# 250. Anti-pattern: detached auto-reattach

Incorrecto:

```php
$entityManager->persist($oldDetachedUser);
```

y asumir automáticamente:

```text
attach + overwrite current DB state
```

---

# 251. Anti-pattern: global WeakMap

Incorrecto en FrankenPHP:

```php
final class EntityState
{
    public static WeakMap $states;
}
```

---

# 252. Anti-pattern: query refresh sobrescribe dirty

Incorrecto:

```text
dirty User#10
+
normal query returns User#10
→ overwrite fields
→ CLEAN
```

---

# 253. Anti-pattern: clone remains managed

Incorrecto:

```text
clone(managed User#10)
→ second managed User#10
```

---

# 254. Anti-pattern: serialization of ORM internals

Incorrecto:

```text
serialize(
    entity +
    EntityManager +
    StateRecord +
    Snapshot
)
```

---

# 255. API objetivo

```php
$user = new User(
    UserId::generate(),
    'Ana'
);

$entityManager->stateOf($user);
// UNTRACKED

$entityManager->persist($user);

$entityManager->stateOf($user);
// NEW

$entityManager->flush();

$entityManager->stateOf($user);
// MANAGED
```

---

# 256. Dirty state

```php
$user->rename('Ana María');

$entityManager
    ->evaluateDirtyState($user);

// DIRTY
```

---

# 257. Removal

```php
$entityManager->remove($user);

$entityManager->stateOf($user);
// REMOVED

$entityManager->flush();

$entityManager->stateOf($user);
// DETACHED
```

---

# 258. IdentityMap consistency

```text
EntityKey(User#42)
        ↓
IdentityMap
        ↓
$user
        ↓
StateRegistry
        ↓
MANAGED
```

---

# 259. Fórmula de Entity Runtime State

```text
EntityRuntimeState
=
PersistenceState
+
IdentityState
+
TrackingState
+
TrackingMode
+
SnapshotState
+
OutcomeCertainty
+
OptionalFreshnessState
```

---

# 260. Fórmula de Managed

```text
Managed(entity, context)
=
Registered(entity, context)
∧
PersistenceState(entity) = MANAGED
∧
ContextOwnership(entity) = context
∧
IdentityMapConsistent(entity)
```

cuando la entidad posee identidad establecida.

---

# 261. Fórmula de Dirty

```text
Dirty(entity)
=
PersistentChangeDetected(
    CurrentDomainPersistentState,
    TrackingBaseline
)
```

según `TrackingStrategy`.

---

# 262. Fórmula de persist()

```text
persist(entity)
≠
INSERT(entity)
```

sino:

```text
persist(entity)
=
ValidateEntity
+
ResolveIdentityState
+
ValidateContextOwnership
+
RegisterAsNEW
+
ScheduleForUnitOfWork
```

cuando la entidad es transient/untracked.

---

# 263. Fórmula de remove()

```text
remove(entity)
≠
DELETE(entity)
```

sino:

```text
remove(entity)
=
ValidateManagedState
+
TransitionToREMOVED
+
ScheduleRemoval
```

---

# 264. Fórmula de successful INSERT

```text
SuccessfulInsert(entity)
=
NEW
+
CertainExecutionOutcome
+
EstablishedIdentity
→
MANAGED
+
CLEAN
+
SnapshotBaseline
```

---

# 265. Fórmula de unknown outcome

```text
UnknownPersistenceOutcome
⇒
NeverAssumeSuccess
∧
NeverAssumeFailure
∧
PreserveUncertainty
∧
RequireReconciliation
```

---

# 266. Fórmula de IdentityMap consistency

```text
EntityState = MANAGED
∧
Identity = ESTABLISHED

⇒

IdentityMap[
    IdentityNamespace,
    EntityType,
    CanonicalIdentifier
]
=
SameObjectInstance
```

---

# 267. Fórmula de rollback safety

```text
SafeOrmRollbackReconciliation
=
KnownTransactionOutcome
+
PreFlushStateJournal
+
KnownIdentityTransitions
+
KnownSnapshotTransitions
+
DeterministicRestoration
```

Si no puede cumplirse:

```text
EntityManager → TAINTED
```

---

# 268. Fórmula de runtime isolation

```text
SafeEntityStateRuntime
=
ScopedEntityStateRegistry
+
ScopedIdentityMap
+
ScopedUnitOfWork
+
ScopedSnapshots
+
ScopedStateJournal
+
ImmutableSharedMetadata
+
DeterministicReset
+
NoCrossRequestEntityState
```

---

# 269. Master Formula

```text
Database Entity State System
=
External ORM State
+
Persistence State Machine
+
Identity State
+
Tracking State
+
Tracking Mode
+
Snapshot State
+
Outcome Certainty
+
Entity State Registry
+
Context Ownership
+
IdentityMap Consistency
+
Explicit State Transitions
+
Transition Validation
+
Persistence Outcome Reconciliation
+
Transaction Rollback Reconciliation
+
Unknown Outcome Preservation
+
Runtime Isolation
+
Diagnostics
+
Telemetry
+
Extension Governance
```

---

# 270. Master Rule

> **VoltStack nunca deduce el estado de persistencia de una entidad únicamente a partir de su identificador, sus atributos o la existencia del objeto PHP. El estado pertenece al PersistenceContext y toda transición debe derivarse de operaciones ORM explícitas y resultados de persistencia cuya certeza sea conocida.**

---

# 271. Resultado arquitectónico

Con este sistema queda formalizada la separación:

```text
Entity
│
├── Domain State
│
├── Logical Identity
│
└── Behavior
        │
        │ observed by
        ▼
PersistenceContext
│
├── EntityStateRegistry
├── IdentityMap
├── Snapshot System
├── Change Tracking
└── UnitOfWork
        │
        ▼
Persistence Engine
        │
        ▼
Execution Outcome
        │
        ▼
EntityStateReconciler
```

Esto evita convertir las entidades de VoltStack en objetos dependientes del ORM.

Una clase:

```php
final class User
{
    public function __construct(
        private UserId $id,
        private string $name,
    ) {}

    public function rename(string $name): void
    {
        $this->name = $name;
    }
}
```

puede continuar siendo una entidad de dominio completamente válida sin contener:

```text
$isManaged
$isDirty
$isPersisted
$isRemoved
$entityManager
$unitOfWork
```

Toda esa información pertenece a:

```text
VoltStack ORM PersistenceContext
```

---

# 272. Estado del bloque ORM

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
        ↓
120_DATABASE_ENTITY_QUERY_SYSTEM.md
        ↓
121_DATABASE_ENTITY_STATE_SYSTEM.md
```

Con esto ya quedan establecidos:

```text
Entity semantics
Model API
Metadata
Mapping
Attributes
EntityManager
Repository
Entity Query
Persistence State Machine
```

Falta cerrar el bloque base del ORM con el sistema formal de lifecycle.

---

# 273. Siguiente documento

```text
122_DATABASE_ENTITY_LIFECYCLE_SYSTEM.md
```

Deberá definir:

```text
Entity Lifecycle
Persistence Lifecycle
Lifecycle Phase
Lifecycle Callback
Lifecycle Listener
Lifecycle Subscriber
Lifecycle Event
prePersist
postPersist
preUpdate
postUpdate
preRemove
postRemove
postLoad
preFlush
postFlush
postRefresh
state transition interaction
UnitOfWork interaction
transaction boundaries
event ordering
callback ordering
reentrancy
failure semantics
async restrictions
domain event separation
persistent runtime isolation
```

manteniendo especialmente:

```text
Lifecycle Callback
≠
Domain Event

postPersist
≠
Transaction Commit

postUpdate
≠
Transaction Commit

postRemove
≠
Transaction Commit

postFlush
≠
Transaction Commit
```

La regla central será:

> **El Entity Lifecycle System observa y coordina fases bien definidas del ciclo de persistencia de una entidad, pero nunca debe hacer pasar una operación ORM ejecutada por una operación definitivamente comprometida en la base de datos cuando la transacción aún no ha concluido.**