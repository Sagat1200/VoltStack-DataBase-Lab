# 130_DATABASE_UPDATE_PERSISTENCE_SYSTEM.md

# VoltStack Quantum Database
## Database Update Persistence System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 130 — Database Update Persistence System  
**Bloque:** 11 — Identity Map, Unit of Work & Persistence  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Update Persistence System` define la arquitectura responsable de transformar cambios persistentes detectados sobre una entidad `MANAGED` en una operación `UPDATE` semánticamente válida, ejecutarla mediante el pipeline canónico de VoltStack y reconciliar posteriormente el estado de la entidad, su versión, valores generados, snapshot y `UnitOfWork`.

El sistema resuelve:

```text
MANAGED Entity
      ↓
Current Persistent State
      ↓
Snapshot / Tracking State
      ↓
ChangeSet
      ↓
UpdateEntityOperation
      ↓
Persistence Planner
      ↓
UpdatePersistenceStep
      ↓
Update Preparation
      ↓
Update Query Model
      ↓
Query Engine
      ↓
Execution Engine
      ↓
Database
      ↓
Execution Outcome
      ↓
Update Reconciliation
      ↓
Snapshot
      ↓
UnitOfWork
      ↓
MANAGED / CLEAN
```

Principio central:

> **Un `UPDATE` ORM en VoltStack no significa volver a guardar ciegamente una entidad completa; significa sincronizar un `ChangeSet` persistente válido contra un baseline conocido, preservando identidad, concurrencia, versionado, resultado de ejecución y coherencia del `PersistenceContext`.**

---

# 2. Responsabilidad

El sistema responde:

> ¿Cómo convertir un conjunto de cambios persistentes de una entidad administrada en una actualización segura, mínima, verificable y reconciliable?

No responde:

```text
¿Cómo detectar inicialmente todos los cambios?
¿Cómo generar SQL?
¿Cómo abrir una conexión?
¿Cómo hacer commit?
¿Cómo resolver autorización?
¿Cómo ejecutar PDO?
```

Estas responsabilidades pertenecen a otros sistemas.

---

# 3. Posición arquitectónica

```text
EntityManager::flush()
        ↓
UnitOfWork
        ↓
Change Tracking
        ↓
Entity Snapshot System
        ↓
ChangeSet
        ↓
Persistence Engine
        ↓
UpdateEntityOperation
        ↓
Persistence Planner
        ↓
UpdatePersistenceStep
        ↓
Update Persistence System
        ↓
Update Query Model
        ↓
Semantic Query Engine
        ↓
Optimizer
        ↓
Query Planner
        ↓
SQL Compiler
        ↓
Execution Engine
        ↓
Database
        ↓
Execution Outcome
        ↓
Update Reconciliation
        ↓
Snapshot / UoW
```

---

# 4. Separaciones fundamentales

```text
Update Persistence
≠ Change Tracking

Update Persistence
≠ Snapshot System

Update Persistence
≠ Query Builder

Update Persistence
≠ SQL Compiler

Update Persistence
≠ Execution Engine

Update Persistence
≠ Transaction Manager

Update Persistence
≠ Bulk Update

Update Persistence
≠ EntityManager::flush()
```

---

# 5. Update ≠ save entire object

Una entidad:

```php
$user->changeName('Alice');
```

puede producir:

```text
ChangeSet
└── name
    ├── old: Bob
    └── new: Alice
```

La intención lógica es:

```text
update persistent property "name"
```

No:

```text
rewrite every mapped column
```

---

# 6. ChangeSet como entrada semántica

El sistema recibe un `ChangeSet` previamente calculado.

```php
final readonly class EntityChangeSet
{
    public function __construct(
        public EntityKey $entityKey,
        public EntityType $entityType,
        public array $changes,
        public SnapshotVersion $baselineVersion,
    ) {}
}
```

---

# 7. ChangeSet ≠ SQL SET

Un cambio:

```text
email:
    old = EmailAddress("old@example.com")
    new = EmailAddress("new@example.com")
```

no es todavía:

```sql
SET email = ?
```

Debe pasar por:

```text
Mapping
↓
Type Conversion
↓
Update Semantics
↓
Query Model
↓
Compiler
```

---

# 8. Condición fundamental

Una actualización ordinaria requiere:

```text
EntityState = MANAGED
```

más:

```text
PersistentChangeSet ≠ ∅
```

salvo operaciones especiales explícitas.

---

# 9. DIRTY como condición ortogonal

VoltStack mantiene la separación:

```text
MANAGED
```

es estado de relación con el `PersistenceContext`.

Mientras:

```text
DIRTY
```

describe que existen cambios persistentes pendientes.

Por tanto:

```text
MANAGED + DIRTY
```

es válido.

---

# 10. MANAGED ≠ dirty

Una entidad administrada sin cambios:

```text
MANAGED + CLEAN
```

no requiere `UPDATE`.

---

# 11. No-op update

Si:

```text
PersistentChangeSet = ∅
```

la operación deberá normalmente convertirse en:

```text
NO_OP
```

---

# 12. NO_OP ≠ UPDATE

No debe generarse:

```sql
UPDATE users SET ...
```

solo porque se llamó:

```php
$entityManager->flush();
```

---

# 13. UpdateEntityOperation

El `Persistence Engine` produce una operación tipada:

```php
final readonly class UpdateEntityOperation
{
    public function __construct(
        public PersistenceOperationId $id,
        public object $entity,
        public EntityKey $entityKey,
        public EntityMetadata $metadata,
        public EntityChangeSet $changeSet,
        public PersistenceOperationContext $context,
    ) {}
}
```

---

# 14. Update operation ≠ execution

La existencia de:

```text
UpdateEntityOperation
```

solo representa intención persistente planificada.

---

# 15. Contrato principal

```php
interface UpdatePersistenceSystem
{
    public function prepare(
        UpdatePersistenceRequest $request,
    ): PreparedUpdatePersistence;

    public function reconcile(
        PreparedUpdatePersistence $update,
        UpdateExecutionOutcome $outcome,
    ): UpdateReconciliationResult;
}
```

---

# 16. UpdatePersistenceRequest

```php
final readonly class UpdatePersistenceRequest
{
    public function __construct(
        public UpdateEntityOperation $operation,
        public EntityMetadata $metadata,
        public PersistencePlanningContext $planningContext,
        public PersistenceExecutionContext $executionContext,
    ) {}
}
```

---

# 17. PreparedUpdatePersistence

```php
final readonly class PreparedUpdatePersistence
{
    public function __construct(
        public UpdatePersistenceId $id,
        public PersistenceOperationId $operationId,
        public EntityKey $entityKey,
        public UpdateValueSet $values,
        public UpdatePredicateSet $predicates,
        public GeneratedValueRequirementSet $generatedValues,
        public UpdateExecutionRequirementSet $requirements,
        public UpdatePersistenceFingerprint $fingerprint,
    ) {}
}
```

---

# 18. Prepared update ≠ executed update

```text
PreparedUpdatePersistence
```

significa:

```text
validated update intent
```

No demuestra modificación en DB.

---

# 19. Updatable property model

No toda propiedad persistente puede actualizarse.

```php
enum PropertyUpdatability
{
    case UPDATABLE;
    case INSERT_ONLY;
    case IMMUTABLE;
    case DATABASE_GENERATED;
    case COMPUTED;
    case READ_ONLY;
    case RELATIONSHIP_DERIVED;
    case NEVER;
}
```

---

# 20. Persistent ≠ updatable

```text
PersistentProperty
⇏
UpdatableProperty
```

---

# 21. Identifier immutability

Por defecto:

```text
Established Entity Identifier
=
IMMUTABLE
```

---

# 22. Identifier mutation

Si:

```php
$user->id = new UserId(20);
```

sobre una entidad ya administrada con ID 10:

```text
EntityIdentityMutationException
```

---

# 23. Identifier mutation ≠ ordinary UPDATE

No deberá convertirse silenciosamente en:

```sql
UPDATE users SET id = 20 WHERE id = 10
```

---

# 24. IdentityMap reason

Cambiar identidad destruiría:

```text
EntityKey → canonical object
```

y podría invalidar:

```text
relationships
snapshots
foreign keys
repositories
references
proxies
```

---

# 25. UpdateValue

```php
final readonly class UpdateValue
{
    public function __construct(
        public PersistentPropertyId $property,
        public UpdateValueKind $kind,
        public mixed $oldValue,
        public mixed $newValue,
        public DatabaseType $type,
    ) {}
}
```

---

# 26. UpdateValueKind

```php
enum UpdateValueKind
{
    case VALUE_CHANGE;
    case SET_NULL;
    case RELATIONSHIP_IDENTIFIER_CHANGE;
    case DATABASE_EXPRESSION;
    case GENERATED_TRANSITION;
}
```

---

# 27. Old value matters

Para algunas políticas:

```text
old value
```

es necesario para:

```text
optimistic locking
diagnostics
audit
conditional update
conflict detection
```

---

# 28. New value matters

Es el candidato a estado persistente posterior.

---

# 29. Dirty field projection

VoltStack preferirá:

```text
UPDATE changed persistent fields
```

en lugar de:

```text
UPDATE every mapped field
```

cuando sea semánticamente seguro.

---

# 30. Partial UPDATE

Ejemplo:

```text
Changed:
  name
  email
```

produce conceptualmente:

```text
UpdateValueSet
├── name
└── email
```

---

# 31. Full-row update

Podrá existir como estrategia explícita:

```php
enum UpdateProjectionStrategy
{
    case DIRTY_FIELDS;
    case ALL_UPDATABLE_FIELDS;
    case EXPLICIT_FIELDS;
    case EXTENSION_DEFINED;
}
```

---

# 32. Default

Para ORM convencional:

```text
DIRTY_FIELDS
```

deberá ser la estrategia preferida.

---

# 33. Full-row danger

Actualizar todas las columnas puede:

```text
overwrite concurrent changes
rewrite stale values
increase write amplification
trigger unnecessary indexes
produce unnecessary DB work
```

---

# 34. Dirty-field danger

También requiere snapshot/change tracking fiable.

Por tanto:

```text
MinimalUpdate
requires
ReliableChangeSet
```

---

# 35. Change tracking strategies

El sistema debe poder consumir cambios provenientes de:

```text
snapshot comparison
notification tracking
explicit dirty marking
custom tracker
```

sin depender de cómo fueron detectados.

---

# 36. Update Persistence does not detect dirty state

Eso corresponde a:

```text
125_DATABASE_CHANGE_TRACKING_SYSTEM.md
```

---

# 37. Snapshot baseline

El `ChangeSet` debe estar asociado al snapshot sobre el cual fue calculado.

```text
Snapshot S0
↓
Entity mutation
↓
Current State S1
↓
ChangeSet Δ(S0,S1)
```

---

# 38. Baseline identity

La actualización debe preservar:

```text
EntityKey(S0) = EntityKey(S1)
```

---

# 39. Stale ChangeSet

Si el snapshot cambió después de calcular el `ChangeSet`:

```text
StaleChangeSetException
```

o recomputación antes de ejecutar.

---

# 40. ChangeSet fingerprint

Puede usarse:

```php
final readonly class ChangeSetFingerprint
{
    public function __construct(
        public string $value,
    ) {}
}
```

para detectar inconsistencias internas.

---

# 41. preUpdate lifecycle

Antes de sellar la actualización:

```text
preUpdate
```

puede modificar propiedades persistentes.

---

# 42. preUpdate mutation

Después:

```text
recompute ChangeSet
```

---

# 43. Example

Inicialmente:

```text
name: Bob → Alice
```

`preUpdate` modifica:

```text
updatedAt: old → now
```

El `ChangeSet` final será:

```text
name
updatedAt
```

---

# 44. Lifecycle stabilization

```text
ChangeSet
↓
preUpdate
↓
mutation
↓
recompute
↓
new changes?
├── yes → stabilization pass
└── no  → freeze
```

---

# 45. Bounded stabilization

Debe existir límite.

```text
MaxLifecycleStabilizationPasses
```

---

# 46. Infinite mutation

Produce:

```text
LifecycleStabilizationException
```

---

# 47. postUpdate

Se ejecuta después de una actualización semánticamente exitosa y reconciliada.

---

# 48. postUpdate ≠ commit

```text
postUpdate
⇏
transaction committed
```

---

# 49. postUpdate mutation

Una mutación persistente en `postUpdate`:

```text
becomes dirty for future flush
```

---

# 50. No recursive UPDATE

No deberá generarse automáticamente otra actualización dentro del mismo `postUpdate`.

---

# 51. Relationship updates

Supóngase:

```php
$order->setCustomer($otherCustomer);
```

El cambio semántico puede convertirse en:

```text
customer_id:
    old: 10
    new: 20
```

---

# 52. Relationship object ≠ FK value

El sistema resuelve:

```text
Entity Reference
↓
EntityKey / Identifier
↓
Database Relationship Value
```

---

# 53. Related entity identity

Si la nueva entidad relacionada todavía no posee identidad requerida:

```text
dependency
```

debe haber sido resuelta por `Persistence Planner`.

---

# 54. Relationship NULL

```php
$order->setCustomer(null);
```

puede producir:

```text
SET relationship FK = NULL
```

solo si mapping/schema semantics lo permiten.

---

# 55. Nullability validation

Si relación no nullable:

```text
UpdateRelationshipNullabilityException
```

antes de ejecución cuando sea detectable.

---

# 56. Owning side

Solo el lado que controla persistencia de la relación deberá generar el cambio físico correspondiente.

---

# 57. Inverse side ≠ physical update

Modificar únicamente una colección inverse no deberá producir arbitrariamente:

```text
UPDATE child SET ...
```

sin la semántica de relación apropiada.

---

# 58. Relationship collection updates

Cambios many-to-many pueden producir operaciones distintas:

```text
junction insert
junction delete
```

y no necesariamente un `UPDATE` de la entidad principal.

---

# 59. Persistence graph

Estas operaciones se coordinan mediante:

```text
PersistenceOperationGraph
```

---

# 60. Update predicates

Una actualización necesita identificar su target.

```php
final readonly class UpdatePredicateSet
{
    public function __construct(
        public array $predicates,
    ) {}
}
```

---

# 61. Identity predicate

Como mínimo:

```text
Entity Identifier Predicate
```

---

# 62. Composite identity predicate

Ejemplo:

```text
order_id = ?
AND
line_number = ?
```

representado semánticamente, no como SQL raw.

---

# 63. Identity namespace

Tenant/shard/database context no necesariamente se convierte en columna.

Depende de la estrategia.

---

# 64. Tenant discriminator

En shared-schema:

```text
tenant_id = ?
```

puede añadirse como predicate mediante integración.

---

# 65. Security context ≠ identity predicate

No mezclar autorización con identidad.

---

# 66. Optimistic locking

El sistema debe integrarse profundamente con:

```text
172_DATABASE_OPTIMISTIC_LOCKING_SYSTEM.md
```

---

# 67. Versioned entity

Ejemplo:

```text
User
├── id = 10
├── name = Alice
└── version = 7
```

---

# 68. Optimistic update

Semánticamente:

```text
target identity = User#10
expected version = 7
new version = 8
```

---

# 69. Query intent

Conceptualmente:

```sql
UPDATE users
SET name = ?, version = ?
WHERE id = ?
  AND version = ?
```

Pero esta SQL es solo ilustrativa.

El ORM genera un `UpdateQueryModel`.

---

# 70. Version predicate

```text
ExpectedVersion
```

forma parte de los predicates de concurrencia.

---

# 71. Version transition

```text
Vn → Vn+1
```

debe ser controlada.

---

# 72. VersionTransition

```php
final readonly class VersionTransition
{
    public function __construct(
        public PersistentPropertyId $property,
        public mixed $expected,
        public mixed $next,
        public VersionTransitionStrategy $strategy,
    ) {}
}
```

---

# 73. Version strategies

```php
enum VersionTransitionStrategy
{
    case APPLICATION_INCREMENT;
    case DATABASE_INCREMENT;
    case DATABASE_GENERATED;
    case TIMESTAMP;
    case CUSTOM;
}
```

---

# 74. Numeric version

Ejemplo:

```text
7 → 8
```

---

# 75. Timestamp version

Puede utilizar:

```text
database timestamp
```

si la semántica está correctamente soportada.

---

# 76. Version overflow

Debe manejarse explícitamente.

No:

```text
integer wraps silently
```

---

# 77. Zero affected rows

En optimistic locking:

```text
affectedRows = 0
```

puede significar:

```text
entity missing
OR
version mismatch
```

---

# 78. OptimisticLockException

Cuando el contrato de versionado determina que el update no encontró el baseline esperado:

```text
OptimisticLockException
```

---

# 79. Missing vs conflict

Algunas plataformas no permiten distinguir sin consulta adicional.

VoltStack no deberá inventar la causa.

Puede clasificar:

```text
OPTIMISTIC_CONFLICT
```

sin afirmar:

```text
row deleted
```

o:

```text
version changed
```

---

# 80. Follow-up diagnostic query

Puede existir bajo modo diagnóstico, pero:

```text
diagnostic observation
≠
original transaction state
```

debido a concurrencia.

---

# 81. Non-versioned update

Para entidad no versionada:

```text
affectedRows = 0
```

requiere interpretar semántica del driver.

---

# 82. Changed rows vs matched rows

Algunos DB/drivers pueden distinguir o reportar:

```text
matched rows
```

versus:

```text
changed rows
```

---

# 83. Capability-aware affected rows

Nunca asumir:

```text
affectedRows = 0
⇒ row absent
```

universalmente.

---

# 84. UpdateAffectedRowSemantics

```php
enum UpdateAffectedRowSemantics
{
    case MATCHED_ROWS;
    case CHANGED_ROWS;
    case UNKNOWN;
}
```

---

# 85. No-op value assignments

Si DB considera:

```text
SET name = 'Alice'
```

sobre un valor ya `Alice` como `0 changed rows`, esto no debe producir falso optimistic-lock conflict si el contrato no usa version predicate.

---

# 86. Versioned update resolves ambiguity

Cuando versión cambia obligatoriamente:

```text
version = 7 → 8
```

un update exitoso debería representar una modificación física observable bajo las capabilities adecuadas.

---

# 87. Update Query Model

```text
PreparedUpdatePersistence
        ↓
UpdatePersistenceQueryTranslator
        ↓
UpdateQueryModel
```

---

# 88. Translator contract

```php
interface UpdatePersistenceQueryTranslator
{
    public function translate(
        PreparedUpdatePersistence $update,
        UpdatePersistenceTranslationContext $context,
    ): UpdateQueryModel;
}
```

---

# 89. Query Model structure

Conceptualmente:

```text
UpdateQueryModel
├── target
├── assignments
├── predicates
├── parameters
├── generated-value requirements
├── concurrency metadata
└── execution metadata
```

---

# 90. SQL generation remains downstream

El translator no genera:

```text
"UPDATE users SET ..."
```

---

# 91. Type conversion

Cada nuevo valor pasa por:

```text
Domain/PHP Value
↓
ORM Mapping
↓
Database Type
↓
Parameter Value
```

---

# 92. Old values

Los valores antiguos utilizados como predicates también deben convertirse mediante el mismo Type System.

---

# 93. NULL transition

```text
old = "Alice"
new = null
```

es un cambio explícito.

---

# 94. NULL → value

Igualmente:

```text
null → "Alice"
```

---

# 95. NaN and special values

Comparación de cambios debe respetar semántica del tipo.

No usar universalmente:

```php
$old !== $new
```

para todos los tipos.

---

# 96. Type-aware equality

El `Change Tracking System` debe usar comparadores tipados cuando sea necesario.

Update Persistence consume ese resultado.

---

# 97. Database-generated update values

Un `UPDATE` puede generar:

```text
updated_at
version
computed columns
audit token
```

---

# 98. GeneratedValueRequirementSet

El update declara cuáles valores necesita recuperar.

---

# 99. UPDATE RETURNING

Si la plataforma lo soporta:

```text
UPDATE ... RETURNING
```

puede satisfacer esos requisitos.

Pero el compiler genera la sintaxis.

---

# 100. No hardcoded PostgreSQL

No:

```php
if ($driver === 'pgsql') {
    // RETURNING
}
```

en ORM.

---

# 101. Generated value strategy

```php
enum UpdateGeneratedValueStrategy
{
    case UPDATE_RETURNING;
    case FOLLOW_UP_QUERY;
    case APPLICATION_COMPUTED;
    case NO_RETRIEVAL_REQUIRED;
    case EXTENSION_DEFINED;
}
```

---

# 102. Follow-up query safety

Debe considerar:

```text
same connection
transaction context
identity
locking
concurrent writers
```

---

# 103. Generated timestamp

Si `updated_at` es DB-generated y requerido para snapshot:

```text
must retrieve
```

---

# 104. Optional generated property

Si no es requerida inmediatamente:

```text
snapshot may mark property unknown/unloaded
```

según contrato.

---

# 105. Snapshot reconciliation

Tras update exitoso:

```text
Old Snapshot
+
Successful ChangeSet
+
Generated Values
=
New Snapshot
```

---

# 106. Snapshot replacement

Conceptualmente:

```text
S0
↓
Δ
↓
DB success
↓
generated values
↓
S1
```

---

# 107. Snapshot must reflect DB-known state

No simplemente:

```text
clone current entity
```

si existen:

```text
database-generated values
unloaded fields
computed columns
postUpdate mutations
```

---

# 108. postUpdate complication

Supóngase:

```text
DB persisted name = Alice
```

y `postUpdate` hace:

```php
$user->changeName('Carol');
```

El snapshot debe conservar:

```text
name = Alice
```

mientras current entity contiene:

```text
name = Carol
```

Por tanto:

```text
entity remains DIRTY
```

---

# 109. Critical snapshot rule

```text
SnapshotAfterUpdate
=
PersistedState
```

No:

```text
CurrentObjectStateAfterPostUpdate
```

---

# 110. Lifecycle ordering

Recomendado:

```text
preUpdate
↓
stabilize ChangeSet
↓
freeze update
↓
execute
↓
reconcile DB-persisted values
↓
establish new snapshot
↓
postUpdate
↓
detect future mutations
```

---

# 111. Alternative internal ordering

La implementación puede preparar algunos datos antes, pero la semántica observable deberá preservar la regla anterior.

---

# 112. ChangeSet consumed

El `ChangeSet` usado por la operación queda:

```text
consumed
```

tras reconciliación exitosa.

---

# 113. New postUpdate changes

Forman:

```text
future ChangeSet
```

no reutilizan el ChangeSet ya ejecutado.

---

# 114. Execution requirements

```php
final readonly class UpdateExecutionRequirementSet
{
    public function __construct(
        public bool $requiresAffectedRows,
        public bool $requiresGeneratedValues,
        public bool $requiresSameConnection,
        public bool $requiresTransaction,
        public bool $requiresOptimisticLockValidation,
    ) {}
}
```

---

# 115. Write intent

Toda actualización ordinaria declara:

```text
ConnectionIntent = WRITE
```

---

# 116. Sticky reads after update

La política posterior de lectura pertenece al Read/Write Routing System.

No al ORM Update System.

---

# 117. UpdateExecutionOutcome

```php
final readonly class UpdateExecutionOutcome
{
    public function __construct(
        public UpdateExecutionStatus $status,
        public PersistenceExecutionCertainty $certainty,
        public ?int $affectedRows,
        public GeneratedValueResultSet $generatedValues,
        public ExecutionDiagnosticCollection $diagnostics,
    ) {}
}
```

---

# 118. Status

```php
enum UpdateExecutionStatus
{
    case SUCCEEDED;
    case FAILED;
    case UNKNOWN;
}
```

---

# 119. UNKNOWN update

Ejemplo:

```text
UPDATE sent
↓
server may execute
↓
network lost before response
```

---

# 120. UNKNOWN ≠ failure

```text
UNKNOWN
≠
FAILED
```

---

# 121. UNKNOWN ≠ success

```text
UNKNOWN
≠
SUCCEEDED
```

---

# 122. Unknown update problem

La DB podría contener:

```text
new values
```

mientras el snapshot todavía contiene:

```text
old values
```

---

# 123. Tainted EntityManager

Ante incertidumbre no reconciliable:

```text
EntityManager → TAINTED
```

---

# 124. Snapshot after UNKNOWN

No debe actualizarse como si el update hubiera sido exitoso.

---

# 125. ChangeSet after UNKNOWN

Tampoco debe descartarse ciegamente.

---

# 126. Unknown state model

Debe conservar suficiente evidencia para:

```text
rollback
discard context
explicit reconciliation
transaction retry
diagnostics
```

---

# 127. Retry safety

Actualizar puede parecer idempotente:

```text
SET name = Alice
```

pero no siempre lo es.

---

# 128. Non-idempotent expressions

Ejemplo:

```text
counter = counter + 1
```

repetido dos veces produce resultado diferente.

---

# 129. Versioned retry

También:

```text
version 7 → 8
```

puede ayudar a detectar replay, pero la política pertenece a Retry/Transaction systems.

---

# 130. UpdateReplaySafety

```php
enum UpdateReplaySafety
{
    case IDEMPOTENT;
    case CONDITIONALLY_SAFE;
    case UNSAFE;
    case UNKNOWN;
}
```

---

# 131. Retry ownership

Update Persistence describe replay characteristics.

No ejecuta retry global por sí mismo.

---

# 132. Transaction separation

```text
UpdateSucceeded
≠
TransactionCommitted
```

---

# 133. flush ≠ commit

La actualización puede completarse dentro de una transacción aún abierta.

---

# 134. Rollback

Si posteriormente:

```text
ROLLBACK
```

la DB puede volver al estado anterior.

Pero el ORM ya pudo haber reconciliado temporalmente:

```text
snapshot
version
generated fields
```

---

# 135. Rollback reconciliation

Será responsabilidad coordinada de Transaction + UoW architecture.

No deberá improvisarse localmente.

---

# 136. Version after rollback

No asumir universalmente:

```text
version 8 → 7
```

en memoria.

Dependerá de política de reconciliación.

---

# 137. External database changes

Si otro proceso cambia el row:

```text
IdentityMap
```

no se actualiza automáticamente.

---

# 138. Managed entity freshness

```text
MANAGED
≠
latest database state
```

---

# 139. Raw Query Builder update

Una operación:

```php
Database::table('users')
    ->where(...)
    ->update(...);
```

puede dejar entidades administradas stale.

---

# 140. Bulk entity update

Igualmente:

```text
UPDATE users SET status = ...
WHERE ...
```

sin materializar entidades no puede reconciliar automáticamente cada snapshot.

---

# 141. Bulk Update ≠ Entity Update Persistence

Deben permanecer separados.

---

# 142. Bulk synchronization policies

Podrán existir:

```text
CLEAR_AFFECTED_CONTEXT
MARK_POTENTIALLY_STALE
EXPLICIT_REFRESH
NO_AUTOMATIC_RECONCILIATION
```

---

# 143. Default conservative policy

No fingir conocimiento de objetos no inspeccionados.

---

# 144. Refresh

`refresh($entity)` debe actualizar la misma instancia canónica.

No reemplazarla.

---

# 145. Refresh after external update

Puede restablecer:

```text
Entity State
+
Snapshot
```

contra DB.

---

# 146. Partial entities

Una entidad parcialmente hidratada requiere especial cuidado.

---

# 147. Unknown field

Un campo no cargado no deberá actualizarse simplemente porque su propiedad PHP tenga un default accidental.

---

# 148. Partial update safety

Solo podrán persistirse cambios cuyo baseline sea suficientemente conocido bajo la política configurada.

---

# 149. Unknown baseline

Si una propiedad modificada requiere old value pero éste es desconocido:

```text
InsufficientUpdateBaselineException
```

---

# 150. Projection entity

VoltStack debe preferir DTO/projection sobre entidades parcialmente administradas cuando no se necesita tracking.

---

# 151. Immutable properties

Una propiedad:

```text
createdAt
```

puede ser:

```text
INSERT_ONLY
```

Modificar su valor en objeto no significa que pueda persistirse.

---

# 152. Immutable property mutation

Políticas posibles:

```text
THROW
IGNORE_FOR_PERSISTENCE
MARK_DOMAIN_ONLY
```

---

# 153. Default

Para propiedades persistentemente inmutables:

```text
THROW
```

es más seguro cuando la mutación representa inconsistencia.

---

# 154. Computed columns

No deberán incluirse en `SET`.

---

# 155. Read-only properties

Tampoco.

---

# 156. Database generated columns

Solo se recuperan según requirements.

---

# 157. Explicit database expression

Puede existir un escape hatch controlado.

Ejemplo conceptual:

```text
IncrementExpression(1)
```

---

# 158. Expression ≠ arbitrary SQL string

Debe representarse mediante AST/query expression tipada.

---

# 159. Counter update

```text
counter = counter + 1
```

debe ser semánticamente explícito.

---

# 160. Snapshot after expression

Si el resultado final no puede calcularse localmente:

```text
generated/read-back requirement
```

o:

```text
property baseline becomes UNKNOWN
```

---

# 161. Update operation dependency

Un update puede depender de:

```text
insert related entity
update parent
delete relation
```

según Persistence Plan.

---

# 162. Planner owns ordering

Update Persistence no reordena operaciones globales.

---

# 163. Foreign key reassignment

```text
Order.customer:
Customer#10 → Customer#20
```

puede requerir que Customer#20 ya exista.

---

# 164. Update before delete

Una relación nullable puede requerir:

```text
UPDATE child FK = NULL
↓
DELETE parent
```

---

# 165. Planner coordinates

No el Update System individualmente.

---

# 166. Batching

Múltiples updates pueden ser candidatos a batching.

Pero:

```text
Batch Update
```

se formaliza en documento 133.

---

# 167. Update batching constraints

Debe preservar:

```text
entity-specific predicates
optimistic versions
generated values
affected row attribution
outcome correlation
```

---

# 168. Different ChangeSets

```text
Entity A changes name
Entity B changes email
```

no son necesariamente batch-compatible.

---

# 169. Same SQL shape

Puede ayudar:

```text
same target
same changed columns
same concurrency model
```

pero no basta por sí solo.

---

# 170. Per-entity outcome

Si optimistic locking está activo, debe poder determinarse qué entidad falló.

---

# 171. Batching never sacrifices conflict detection

Regla absoluta.

---

# 172. Database triggers

Un trigger puede alterar valores adicionales.

---

# 173. Trigger-mutated state

Si esos valores son relevantes al ORM:

```text
GeneratedValueRequirement
```

debe declararlos.

---

# 174. Trigger side effects

No deberán asumirse reversibles ni idempotentes.

---

# 175. Audit fields

Ejemplos:

```text
updated_at
updated_by
revision
```

pueden ser contribuidos por mapping/extensions.

---

# 176. Audit ≠ authorization

Registrar quién modificó no decide si podía modificar.

---

# 177. Multitenancy integration

Core Database no depende de Multitenancy.

---

# 178. Tenant namespace

`EntityKey` ya contiene el namespace efectivo.

---

# 179. Tenant predicate

En shared-schema, integración puede añadir:

```text
tenant_id = current tenant
```

---

# 180. Tenant predicate is mandatory policy

Cuando el mapping multitenant lo requiere, no podrá deshabilitarse accidentalmente por una operación ORM ordinaria.

---

# 181. Tenant change

Una entidad no deberá migrar silenciosamente de:

```text
tenant A
```

a:

```text
tenant B
```

mediante ordinary update.

---

# 182. Tenant identity mutation

Normalmente:

```text
ContextMismatch / IdentityMutation
```

---

# 183. Shard key mutation

Igualmente peligrosa.

Una shard key que determine ubicación física no deberá actualizarse como campo ordinario salvo sistema explícito de relocation.

---

# 184. Persistent runtime

FrankenPHP exige aislamiento estricto.

---

# 185. Shared immutable state

Puede compartirse:

```text
compiled metadata
type definitions
frozen strategy registry
capability definitions
compiled accessors
```

---

# 186. Scoped mutable state

No compartir:

```text
entity
ChangeSet
snapshot
UpdatePersistenceRequest
PreparedUpdatePersistence
generated values
EntityManager
UnitOfWork
IdentityMap
tenant
transaction
```

---

# 187. No static current ChangeSet

Prohibido.

---

# 188. No static updated entity

Prohibido.

---

# 189. Worker reuse

```text
Request A update state
```

debe desaparecer antes de:

```text
Request B
```

---

# 190. Coroutine isolation

En OpenSwoole:

```text
Coroutine A
≠
Coroutine B PersistenceContext
```

---

# 191. Concurrent EntityManager mutation

Un mismo EntityManager no será thread/coroutine-safe por defecto.

---

# 192. Parallel update preparation

Puede permitirse en el futuro solo bajo coordinación explícita.

---

# 193. Update cancellation

Antes de enviar statement:

```text
safe cancellation
```

puede garantizar no side effect.

---

# 194. Cancellation after send

Puede producir:

```text
UNKNOWN
```

---

# 195. Timeout after send

Igualmente.

---

# 196. Failure phases

```php
enum UpdateFailurePhase
{
    case CHANGESET_VALIDATION;
    case PREPARATION;
    case LIFECYCLE_STABILIZATION;
    case TRANSLATION;
    case COMPILATION;
    case PRE_EXECUTION;
    case EXECUTION;
    case GENERATED_VALUE_CAPTURE;
    case RECONCILIATION;
    case POST_LIFECYCLE;
}
```

---

# 197. Preparation failure

No implica DB modification.

---

# 198. Compilation failure

No implica DB modification.

---

# 199. Execution failure

Puede ser:

```text
CERTAIN FAILED
```

o:

```text
UNKNOWN
```

---

# 200. Reconciliation failure

Puede ocurrir después de DB success.

---

# 201. Outcome dimensions

Preservar:

```text
DatabaseOperationOutcome
ORMReconciliationOutcome
LifecycleOutcome
TransactionOutcome
```

---

# 202. Example

```text
DatabaseOperationOutcome = SUCCEEDED
ORMReconciliationOutcome = SUCCEEDED
LifecycleOutcome = FAILED
TransactionOutcome = PENDING
```

es válido.

---

# 203. Another example

```text
DatabaseOperationOutcome = SUCCEEDED
ORMReconciliationOutcome = FAILED
TransactionOutcome = PENDING
```

requiere taint/recovery.

---

# 204. No fake rollback

Un fallo de `postUpdate` no significa que DB update no ocurrió.

---

# 205. No fake snapshot rollback

Tampoco debe manipularse el snapshot arbitrariamente para fingir atomicidad inexistente.

---

# 206. Extension architecture

```php
interface UpdatePersistenceExtension
{
    public function contribute(
        UpdatePersistenceExtensionContext $context,
    ): UpdatePersistenceContribution;
}
```

---

# 207. Extension uses

Podrán contribuir:

```text
audit values
tenant predicates
custom version strategy
generated values
custom update expressions
domain-specific persistence metadata
```

---

# 208. Extension restrictions

No podrán:

```text
execute arbitrary SQL
commit
rollback
replace EntityKey
silently remove optimistic lock predicate
silently bypass tenant predicate
silently rewrite ChangeSet
```

---

# 209. Contribution provenance

Cada modificación debe ser rastreable.

---

# 210. Conflict detection

Dos extensiones intentando asignar:

```text
updated_by = A
updated_by = B
```

sin estrategia de composición:

```text
UpdateExtensionConflictException
```

---

# 211. Telemetry

Métricas propuestas:

```text
orm.persistence.update.prepared
orm.persistence.update.noop
orm.persistence.update.executed
orm.persistence.update.succeeded
orm.persistence.update.failed
orm.persistence.update.unknown
orm.persistence.update.reconciled
orm.persistence.update.optimistic_conflict
orm.persistence.update.changed_fields
orm.persistence.update.generated_values
orm.persistence.update.duration
orm.persistence.update.reconciliation_duration
orm.persistence.update.tainted_context
```

---

# 212. Slow update telemetry

Debe poder correlacionarse con:

```text
PersistenceOperationId
QueryId
ExecutionId
TransactionId
```

---

# 213. No sensitive values

Telemetry no deberá registrar por defecto:

```text
old password
new password
tokens
PII values
entity identifiers with high cardinality
```

---

# 214. Diagnostics

Ejemplo:

```text
Update Persistence
────────────────────────────────

Entity:
  App\Domain\User

Identity:
  User#42

State:
  MANAGED / DIRTY

Snapshot:
  generation: 17

ChangeSet:
  name:
    changed: yes
  email:
    changed: yes

Projection:
  DIRTY_FIELDS

Optimistic Lock:
  enabled
  expected version: 7
  next version: 8

Generated Values:
  updatedAt:
    REQUIRED_FOR_SNAPSHOT

Execution:
  status: SUCCEEDED
  certainty: CERTAIN
  affected rows: 1

Reconciliation:
  generated values: OK
  snapshot: generation 18
  UnitOfWork: CLEAN
  entity state: MANAGED

Transaction:
  PENDING
```

---

# 215. Error hierarchy

```text
DatabaseOrmException
└── PersistenceException
    └── UpdatePersistenceException
        ├── UpdatePersistencePreparationException
        ├── UpdatePersistenceValidationException
        ├── InvalidUpdateStateException
        ├── MissingUpdateIdentityException
        ├── EntityIdentityMutationException
        ├── NonUpdatablePropertyException
        ├── ImmutablePropertyMutationException
        ├── StaleChangeSetException
        ├── InvalidChangeSetException
        ├── InsufficientUpdateBaselineException
        ├── UpdateValueConversionException
        ├── UpdateRelationshipException
        ├── UpdateRelationshipNullabilityException
        ├── UpdateDependencyException
        ├── UpdateVersionException
        ├── InvalidVersionTransitionException
        ├── VersionOverflowException
        ├── OptimisticLockException
        ├── UpdateAffectedRowException
        ├── UpdateGeneratedValueException
        ├── MissingUpdateGeneratedValueException
        ├── UpdateTranslationException
        ├── UpdateExecutionException
        ├── UpdateOutcomeUnknownException
        ├── UpdateReplaySafetyException
        ├── UpdateReconciliationException
        ├── UpdateSnapshotReconciliationException
        ├── UpdateUnitOfWorkReconciliationException
        ├── UpdateLifecycleException
        ├── UpdateExtensionException
        ├── UpdateExtensionConflictException
        ├── UpdateContextMismatchException
        ├── UpdateRuntimeIsolationException
        └── UpdatePersistenceInvariantException
```

---

# 216. Estructura de directorios

```text
src/Quantum/Database/ORM/Persistence/Update/
│
├── Contract/
│   ├── UpdatePersistenceSystem.php
│   ├── UpdatePersistencePreparer.php
│   ├── UpdatePersistenceReconciler.php
│   └── UpdatePersistenceQueryTranslator.php
│
├── Operation/
│   ├── UpdateEntityOperation.php
│   └── UpdatePersistenceId.php
│
├── Preparation/
│   ├── DefaultUpdatePersistenceSystem.php
│   ├── UpdatePersistenceRequest.php
│   ├── PreparedUpdatePersistence.php
│   ├── UpdatePersistenceFingerprint.php
│   └── UpdatePersistenceValidator.php
│
├── Value/
│   ├── UpdateValue.php
│   ├── UpdateValueKind.php
│   ├── UpdateValueSet.php
│   ├── UpdateValueResolver.php
│   ├── PropertyUpdatability.php
│   └── UpdateProjectionStrategy.php
│
├── Predicate/
│   ├── UpdatePredicate.php
│   ├── UpdatePredicateSet.php
│   ├── IdentityUpdatePredicate.php
│   └── UpdatePredicateResolver.php
│
├── Version/
│   ├── VersionTransition.php
│   ├── VersionTransitionStrategy.php
│   ├── VersionTransitionResolver.php
│   └── OptimisticUpdatePolicy.php
│
├── Relationship/
│   ├── UpdateRelationshipValueResolver.php
│   ├── UpdateRelationshipChange.php
│   └── UpdateRelationshipValidator.php
│
├── Generated/
│   ├── UpdateGeneratedValueStrategy.php
│   ├── UpdateGeneratedValueStrategyResolver.php
│   └── UpdateGeneratedValueReconciler.php
│
├── Query/
│   ├── DefaultUpdatePersistenceQueryTranslator.php
│   ├── UpdatePersistenceTranslationContext.php
│   └── UpdateQueryMetadata.php
│
├── Execution/
│   ├── UpdateExecutionOutcome.php
│   ├── UpdateExecutionStatus.php
│   ├── UpdateExecutionRequirementSet.php
│   ├── UpdateAffectedRowSemantics.php
│   ├── UpdateReplaySafety.php
│   └── UpdateFailurePhase.php
│
├── Reconciliation/
│   ├── UpdatePersistenceReconciler.php
│   ├── UpdateReconciliationContext.php
│   ├── UpdateReconciliationResult.php
│   ├── UpdateSnapshotReconciler.php
│   └── UpdateUnitOfWorkReconciler.php
│
├── Extension/
│   ├── UpdatePersistenceExtension.php
│   ├── UpdatePersistenceExtensionRegistry.php
│   ├── UpdatePersistenceExtensionContext.php
│   └── UpdatePersistenceContribution.php
│
├── Telemetry/
│   ├── UpdatePersistenceTelemetry.php
│   └── UpdatePersistenceDiagnostics.php
│
└── Exception/
    └── ...
```

---

# 217. Testing Strategy

Debe cubrir:

```text
dirty-field updates
full-row strategies
no-op updates
identity immutability
immutable properties
relationships
optimistic locking
version transitions
generated values
snapshot reconciliation
lifecycle
unknown outcomes
persistent runtime
extensions
batch compatibility
```

---

# 218. Test — MANAGED clean

```text
MANAGED + empty ChangeSet
→ NO_OP
```

---

# 219. Test — one dirty field

Solo la propiedad modificada entra al `UpdateValueSet`.

---

# 220. Test — multiple dirty fields

Todas las propiedades persistentes cambiadas son proyectadas.

---

# 221. Test — domain-only property

No produce update.

---

# 222. Test — identifier mutation

Falla antes de execution.

---

# 223. Test — insert-only property mutation

Se aplica política configurada.

---

# 224. Test — computed property

Nunca aparece en assignments.

---

# 225. Test — DB-generated property

No aparece como assignment ordinario.

---

# 226. Test — stale ChangeSet

Es rechazado/recalculado.

---

# 227. Test — preUpdate mutation

ChangeSet final contiene nueva mutación.

---

# 228. Test — lifecycle stabilization

Converge correctamente.

---

# 229. Test — lifecycle infinite mutation

Falla por límite.

---

# 230. Test — postUpdate mutation

Entidad queda dirty para próximo flush.

---

# 231. Test — postUpdate no recursive SQL

Garantizado.

---

# 232. Test — relationship reassignment

FK correcta.

---

# 233. Test — relationship to NULL

Permitido solo si mapping lo permite.

---

# 234. Test — unresolved related ID

No ejecuta update prematuramente.

---

# 235. Test — inverse relation

No produce physical update incorrecto.

---

# 236. Test — optimistic success

```text
expected version = 7
affected rows = 1
new version = 8
```

---

# 237. Test — optimistic conflict

```text
expected version = 7
affected rows = 0
→ OptimisticLockException
```

---

# 238. Test — version overflow

Error explícito.

---

# 239. Test — matched-row semantics

Interpretación correcta.

---

# 240. Test — changed-row semantics

No produce falsos conflictos.

---

# 241. Test — generated updatedAt

Snapshot recibe valor real.

---

# 242. Test — UPDATE RETURNING

Funciona mediante capability, no vendor conditional.

---

# 243. Test — follow-up generated value

Preserva same-connection requirement.

---

# 244. Test — successful reconciliation

```text
MANAGED / DIRTY
→ MANAGED / CLEAN
```

---

# 245. Test — snapshot baseline

Refleja estado persistido.

---

# 246. Test — postUpdate current state differs

Snapshot conserva estado DB; entidad queda dirty.

---

# 247. Test — UNKNOWN execution

Snapshot no se avanza ciegamente.

---

# 248. Test — UNKNOWN ChangeSet

No se descarta ciegamente.

---

# 249. Test — timeout after send

Produce incertidumbre cuando corresponda.

---

# 250. Test — cancellation before send

No DB side effect.

---

# 251. Test — non-idempotent expression

Replay safety = UNSAFE/CONDITIONAL.

---

# 252. Test — flush does not commit

Garantizado.

---

# 253. Test — postUpdate does not mean commit

Garantizado.

---

# 254. Test — raw external update

Managed entity puede quedar stale.

---

# 255. Test — refresh

Reutiliza misma instancia canónica.

---

# 256. Test — partial entity unknown field

No sobrescribe DB accidentalmente.

---

# 257. Test — tenant predicate

Aplicado mediante scoped integration.

---

# 258. Test — tenant mutation

No se permite ordinary update.

---

# 259. Test — shard key mutation

No se trata como field update ordinario.

---

# 260. Test — extension conflict

Error determinista.

---

# 261. Test — persistent worker isolation

Request A no contamina Request B.

---

# 262. Test — no hidden DB I/O during preparation

Preparación es pura respecto a ejecución salvo servicios explícitamente permitidos.

---

# 263. Anti-patterns

## 263.1 Actualizar todas las columnas siempre

Evitar.

## 263.2 Usar entidad completa como ChangeSet

Incorrecto.

## 263.3 Modificar primary key mediante ordinary update

Prohibido.

## 263.4 Tratar `MANAGED` como `DIRTY`

Incorrecto.

## 263.5 Ejecutar UPDATE para ChangeSet vacío

Evitar.

## 263.6 Generar SQL desde ORM

Prohibido.

## 263.7 Hardcodear PostgreSQL/MySQL

Prohibido.

## 263.8 `affectedRows = 0` siempre significa missing row

Incorrecto.

## 263.9 postUpdate = commit

Incorrecto.

## 263.10 Timeout = update failed

Incorrecto.

## 263.11 Retry ciego

Prohibido.

## 263.12 Snapshot = clone de entidad después de callback

Incorrecto.

## 263.13 Sobrescribir campos no cargados

Prohibido.

## 263.14 Bulk update como entity update

Incorrecto.

## 263.15 Tenant global estático

Prohibido.

## 263.16 Bypass de optimistic lock por extensión

Prohibido.

---

# 264. Architectural Invariants

## DB-ORM-UPDATE-001
Update Persistence será distinto de Change Tracking.

## DB-ORM-UPDATE-002
Update Persistence será distinto de Snapshot System.

## DB-ORM-UPDATE-003
Update Persistence será distinto de SQL Compiler.

## DB-ORM-UPDATE-004
Update Persistence será distinto de Execution Engine.

## DB-ORM-UPDATE-005
Update Persistence será distinto de Transaction Manager.

## DB-ORM-UPDATE-006
Update Persistence no generará SQL.

## DB-ORM-UPDATE-007
Update Persistence no utilizará PDO directamente.

## DB-ORM-UPDATE-008
Update Persistence no realizará commit.

## DB-ORM-UPDATE-009
Update Persistence no realizará rollback.

## DB-ORM-UPDATE-010
MANAGED no implicará DIRTY.

## DB-ORM-UPDATE-011
DIRTY será una condición de cambios persistentes.

## DB-ORM-UPDATE-012
ChangeSet vacío producirá normalmente NO_OP.

## DB-ORM-UPDATE-013
flush no implicará UPDATE.

## DB-ORM-UPDATE-014
ChangeSet será distinto de SQL assignments.

## DB-ORM-UPDATE-015
Prepared update no implicará executed update.

## DB-ORM-UPDATE-016
Persistent property será distinta de updatable property.

## DB-ORM-UPDATE-017
Established identifier será immutable por defecto.

## DB-ORM-UPDATE-018
Identifier mutation no será ordinary update.

## DB-ORM-UPDATE-019
IdentityMap canonicality será preservada.

## DB-ORM-UPDATE-020
Dirty-field projection será estrategia preferida cuando sea segura.

## DB-ORM-UPDATE-021
Full-row update será explícito.

## DB-ORM-UPDATE-022
Minimal update requerirá ChangeSet fiable.

## DB-ORM-UPDATE-023
Update Persistence no detectará dirty state por sí mismo.

## DB-ORM-UPDATE-024
ChangeSet tendrá baseline identificable.

## DB-ORM-UPDATE-025
Stale ChangeSet no será ejecutado silenciosamente.

## DB-ORM-UPDATE-026
preUpdate podrá modificar persistent state.

## DB-ORM-UPDATE-027
preUpdate mutation requerirá recomputación.

## DB-ORM-UPDATE-028
Lifecycle stabilization será bounded.

## DB-ORM-UPDATE-029
Infinite lifecycle mutation será error.

## DB-ORM-UPDATE-030
postUpdate será posterior a semantic update reconciliation.

## DB-ORM-UPDATE-031
postUpdate será distinto de commit.

## DB-ORM-UPDATE-032
postUpdate mutation será future dirty state.

## DB-ORM-UPDATE-033
postUpdate no producirá recursive update automático.

## DB-ORM-UPDATE-034
Relationship object será distinto de FK value.

## DB-ORM-UPDATE-035
Relationship updates usarán metadata.

## DB-ORM-UPDATE-036
Unresolved related identity respetará planner dependency.

## DB-ORM-UPDATE-037
Nullable relationship será validada.

## DB-ORM-UPDATE-038
Owning-side semantics serán respetadas.

## DB-ORM-UPDATE-039
Inverse-side mutation no producirá physical update arbitrario.

## DB-ORM-UPDATE-040
Many-to-many collection mutation podrá generar operaciones distintas.

## DB-ORM-UPDATE-041
Persistence Planner controlará ordering global.

## DB-ORM-UPDATE-042
Update target incluirá canonical entity identity.

## DB-ORM-UPDATE-043
Composite identity será soportada.

## DB-ORM-UPDATE-044
Tenant predicates serán integraciones tipadas.

## DB-ORM-UPDATE-045
Authorization será distinta de update identity.

## DB-ORM-UPDATE-046
Optimistic locking será una preocupación first-class.

## DB-ORM-UPDATE-047
Expected version será distinta de next version.

## DB-ORM-UPDATE-048
Version transition será tipada.

## DB-ORM-UPDATE-049
Version overflow no será silencioso.

## DB-ORM-UPDATE-050
Optimistic conflict no inventará causa no observable.

## DB-ORM-UPDATE-051
Affected-row semantics serán capability-aware.

## DB-ORM-UPDATE-052
Zero affected rows no significará universalmente missing row.

## DB-ORM-UPDATE-053
Matched rows será distinto de changed rows.

## DB-ORM-UPDATE-054
No-op DB assignment no producirá falso conflicto sin fundamento.

## DB-ORM-UPDATE-055
Update será representado mediante Query Model.

## DB-ORM-UPDATE-056
ORM no producirá SQL strings.

## DB-ORM-UPDATE-057
New values usarán Database Type System.

## DB-ORM-UPDATE-058
Predicate values usarán Database Type System.

## DB-ORM-UPDATE-059
NULL transitions serán explícitas.

## DB-ORM-UPDATE-060
Type-aware equality será respetada.

## DB-ORM-UPDATE-061
Generated update values serán first-class.

## DB-ORM-UPDATE-062
UPDATE RETURNING será capability-driven.

## DB-ORM-UPDATE-063
ORM no hardcodeará vendors para generated values.

## DB-ORM-UPDATE-064
Follow-up generated-value query preservará correlation safety.

## DB-ORM-UPDATE-065
Required DB-generated timestamp será recuperado.

## DB-ORM-UPDATE-066
Optional generated value podrá permanecer unknown según policy.

## DB-ORM-UPDATE-067
Snapshot reconciliation usará persisted state.

## DB-ORM-UPDATE-068
Snapshot no será simplemente current object state.

## DB-ORM-UPDATE-069
postUpdate mutation no contaminará baseline persistido.

## DB-ORM-UPDATE-070
Consumed ChangeSet no será reutilizado.

## DB-ORM-UPDATE-071
Future mutations generarán future ChangeSet.

## DB-ORM-UPDATE-072
Update tendrá WRITE connection intent.

## DB-ORM-UPDATE-073
Read/write routing permanecerá separado.

## DB-ORM-UPDATE-074
Execution outcome será estructurado.

## DB-ORM-UPDATE-075
UNKNOWN será first-class.

## DB-ORM-UPDATE-076
UNKNOWN será distinto de FAILED.

## DB-ORM-UPDATE-077
UNKNOWN será distinto de SUCCEEDED.

## DB-ORM-UPDATE-078
UNKNOWN no avanzará snapshot ciegamente.

## DB-ORM-UPDATE-079
UNKNOWN no descartará ChangeSet ciegamente.

## DB-ORM-UPDATE-080
Uncertainty podrá taint EntityManager.

## DB-ORM-UPDATE-081
Replay safety será explícita.

## DB-ORM-UPDATE-082
Update no será asumido idempotente universalmente.

## DB-ORM-UPDATE-083
Database expressions podrán ser non-idempotent.

## DB-ORM-UPDATE-084
Retry policy permanecerá separada.

## DB-ORM-UPDATE-085
Update success será distinto de transaction commit.

## DB-ORM-UPDATE-086
flush será distinto de commit.

## DB-ORM-UPDATE-087
Rollback reconciliation permanecerá coordinada con Transaction System.

## DB-ORM-UPDATE-088
Rollback no revertirá versiones en memoria ciegamente.

## DB-ORM-UPDATE-089
MANAGED será distinto de DB-fresh.

## DB-ORM-UPDATE-090
Raw updates podrán dejar IdentityMap stale.

## DB-ORM-UPDATE-091
Bulk update será distinto de entity update persistence.

## DB-ORM-UPDATE-092
Bulk synchronization no fingirá conocimiento inexistente.

## DB-ORM-UPDATE-093
refresh reutilizará canonical instance.

## DB-ORM-UPDATE-094
Partial entity no sobrescribirá unknown fields.

## DB-ORM-UPDATE-095
Unknown baseline podrá impedir update.

## DB-ORM-UPDATE-096
Projection será preferible cuando tracking parcial no sea necesario.

## DB-ORM-UPDATE-097
Immutable persistent properties tendrán política explícita.

## DB-ORM-UPDATE-098
Computed columns no serán ordinary assignments.

## DB-ORM-UPDATE-099
Read-only columns no serán ordinary assignments.

## DB-ORM-UPDATE-100
Raw expressions serán AST tipado.

## DB-ORM-UPDATE-101
Expression result unknown requerirá read-back o unknown snapshot state.

## DB-ORM-UPDATE-102
Update dependencies serán respetadas.

## DB-ORM-UPDATE-103
Update system no reordenará PersistencePlan global.

## DB-ORM-UPDATE-104
Batching preservará entity-specific concurrency semantics.

## DB-ORM-UPDATE-105
Batching preservará generated-value correlation.

## DB-ORM-UPDATE-106
Batching preservará optimistic conflict attribution.

## DB-ORM-UPDATE-107
Batching nunca tendrá prioridad sobre correctness.

## DB-ORM-UPDATE-108
Trigger-mutated relevant fields serán declarados como generated values.

## DB-ORM-UPDATE-109
Trigger side effects no serán asumidos idempotentes.

## DB-ORM-UPDATE-110
Audit fields podrán ser extension contributions.

## DB-ORM-UPDATE-111
Audit será distinto de Authorization.

## DB-ORM-UPDATE-112
Multitenancy no será core dependency.

## DB-ORM-UPDATE-113
EntityKey namespace será preservado.

## DB-ORM-UPDATE-114
Required tenant predicates no serán bypassed silenciosamente.

## DB-ORM-UPDATE-115
Tenant relocation no será ordinary update.

## DB-ORM-UPDATE-116
Shard relocation no será ordinary update.

## DB-ORM-UPDATE-117
Compiled metadata podrá compartirse entre requests.

## DB-ORM-UPDATE-118
ChangeSets serán scoped.

## DB-ORM-UPDATE-119
Snapshots serán scoped.

## DB-ORM-UPDATE-120
Prepared updates serán scoped.

## DB-ORM-UPDATE-121
EntityManager será scoped.

## DB-ORM-UPDATE-122
UnitOfWork será scoped.

## DB-ORM-UPDATE-123
IdentityMap será scoped.

## DB-ORM-UPDATE-124
Tenant context será scoped.

## DB-ORM-UPDATE-125
Transaction context será scoped.

## DB-ORM-UPDATE-126
No existirá static current ChangeSet.

## DB-ORM-UPDATE-127
No existirá static current entity.

## DB-ORM-UPDATE-128
Persistent workers limpiarán mutable ORM state.

## DB-ORM-UPDATE-129
OpenSwoole coroutine contexts permanecerán aislados.

## DB-ORM-UPDATE-130
EntityManager no será concurrent-mutation-safe por defecto.

## DB-ORM-UPDATE-131
Cancellation before send podrá ser side-effect-free.

## DB-ORM-UPDATE-132
Cancellation after send podrá producir UNKNOWN.

## DB-ORM-UPDATE-133
Timeout after send podrá producir UNKNOWN.

## DB-ORM-UPDATE-134
Failure phase será preservada.

## DB-ORM-UPDATE-135
Preparation failure no implicará DB mutation.

## DB-ORM-UPDATE-136
Compilation failure no implicará DB mutation.

## DB-ORM-UPDATE-137
Execution failure podrá tener outcome unknown.

## DB-ORM-UPDATE-138
Reconciliation failure podrá ocurrir después de DB success.

## DB-ORM-UPDATE-139
DB operation outcome será distinto de reconciliation outcome.

## DB-ORM-UPDATE-140
Lifecycle outcome será distinto de DB operation outcome.

## DB-ORM-UPDATE-141
Transaction outcome será distinto de update outcome.

## DB-ORM-UPDATE-142
Reconciliation failure podrá taint manager.

## DB-ORM-UPDATE-143
postUpdate failure no fingirá DB failure.

## DB-ORM-UPDATE-144
Update extensions serán deterministas.

## DB-ORM-UPDATE-145
Extensions no ejecutarán arbitrary SQL.

## DB-ORM-UPDATE-146
Extensions no harán commit.

## DB-ORM-UPDATE-147
Extensions no harán rollback.

## DB-ORM-UPDATE-148
Extensions no podrán cambiar EntityKey silenciosamente.

## DB-ORM-UPDATE-149
Extensions no podrán eliminar optimistic predicates silenciosamente.

## DB-ORM-UPDATE-150
Extensions no podrán eliminar tenant predicates silenciosamente.

## DB-ORM-UPDATE-151
Extension conflicts serán explícitos.

## DB-ORM-UPDATE-152
Extension provenance será observable.

## DB-ORM-UPDATE-153
Telemetry no alterará persistence semantics.

## DB-ORM-UPDATE-154
Sensitive values no serán metrics/log labels por defecto.

## DB-ORM-UPDATE-155
PersistenceOperationId permitirá correlación.

## DB-ORM-UPDATE-156
Snapshot generation avanzará solo tras reconciliación válida.

## DB-ORM-UPDATE-157
No-op updates no deberán incrementar versión salvo policy explícita.

## DB-ORM-UPDATE-158
Version increment no ocurrirá sin una operación que lo requiera.

## DB-ORM-UPDATE-159
Database-generated values no serán inventados.

## DB-ORM-UPDATE-160
Una entidad administrada solo será considerada nuevamente limpia cuando el ChangeSet ejecutado haya sido reconciliado contra un resultado suficientemente cierto y el nuevo snapshot represente fielmente el estado persistido conocido.

---

# 265. Fórmulas fundamentales

## 265.1 Update candidate

```text
UpdateCandidate(E)
=
State(E) = MANAGED
∧
PersistentChangeSet(E) ≠ ∅
∧
IdentityEstablished(E)
```

---

# 266. ChangeSet

```text
Δ(E)
=
PersistentStateCurrent(E)
-
PersistentSnapshot(E)
```

conceptualmente, bajo comparadores tipados.

---

# 267. No-op

```text
Δ(E) = ∅
⇒
UpdateOperation(E) = NO_OP
```

salvo policy explícita.

---

# 268. Update projection

```text
UpdateProjection(E)
=
Filter(
    Δ(E),
    PropertyUpdatability = UPDATABLE
)
```

más contribuciones controladas como versionado/auditoría.

---

# 269. Identity preservation

```text
IdentityBefore(E)
=
IdentityAfter(E)
```

para ordinary update.

---

# 270. Optimistic predicate

```text
OptimisticUpdatePredicate
=
EntityIdentity
∧
ExpectedVersion
```

---

# 271. Version transition

```text
VersionTransition
=
ExpectedVersion
→
NextVersion
```

---

# 272. Successful optimistic update

```text
SuccessfulOptimisticUpdate
=
ExecutionSucceeded
∧
OutcomeCertain
∧
ExpectedRowMatched
∧
VersionTransitionReconciled
```

---

# 273. Snapshot reconciliation

```text
Snapshotₙ₊₁
=
Apply(
    Snapshotₙ,
    PersistedChangeSet,
    GeneratedDatabaseValues
)
```

---

# 274. Post-lifecycle dirty state

```text
CurrentEntityStateAfterPostUpdate
≠
Snapshotₙ₊₁
⇒
EntityRemainsDirty
```

---

# 275. Update success

```text
SuccessfulUpdate
=
ExecutionSucceeded
∧
OutcomeCertaintySufficient
∧
ConcurrencySemanticsSatisfied
∧
RequiredGeneratedValuesResolved
∧
SnapshotReconciled
```

---

# 276. Durability

```text
SuccessfulUpdate
⇏
TransactionCommitted
```

y:

```text
DurableUpdate
=
SuccessfulUpdate
∧
RequiredTransactionOutcome = COMMITTED
```

---

# 277. Unknown outcome

```text
UNKNOWN
⇒
¬ AssumeUpdated
∧
¬ AssumeNotUpdated
```

---

# 278. Safe retry

```text
SafeUpdateReplay
=
ReplaySemanticsKnown
∧
OperationIdempotencySatisfied
∧
VersionSemanticsCompatible
∧
TransactionStateCompatible
∧
SideEffectsSafe
```

---

# 279. Safe partial update

```text
SafePartialUpdate
=
ReliableChangeSet
∧
KnownRequiredBaseline
∧
IdentityStable
∧
ConcurrencyPolicySatisfied
```

---

# 280. Reconciliation consistency

```text
UpdateReconciliationConsistent
=
EntityIdentityConsistent
∧
GeneratedValuesConsistent
∧
SnapshotConsistent
∧
UnitOfWorkConsistent
∧
LifecycleStateConsistent
```

---

# 281. Arquitectura maestra

```text
                   MANAGED Entity
                         │
                         ▼
                 Change Tracking
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
         Current State          Snapshot
              │                     │
              └──────────┬──────────┘
                         ▼
                      ChangeSet
                         │
                         ▼
                   preUpdate
                         │
                         ▼
               ChangeSet Recompute
                         │
                         ▼
                 Stabilization
                         │
                         ▼
                 Persistence Engine
                         │
                         ▼
                UpdateEntityOperation
                         │
                         ▼
                Persistence Planner
                         │
                         ▼
                UpdatePersistenceStep
                         │
                         ▼
              ┌─────────────────────┐
              │ Update Persistence  │
              │ Preparation         │
              └─────────────────────┘
                         │
       ┌─────────────────┼──────────────────┐
       ▼                 ▼                  ▼
    Values           Predicates          Version
       │                 │                  │
       └─────────────────┼──────────────────┘
                         ▼
              Generated Requirements
                         │
                         ▼
                 Update Query Model
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
             ┌────────────────────────┐
             │ Update Reconciliation  │
             └────────────────────────┘
                         │
            ┌────────────┼────────────┐
            ▼            ▼            ▼
       Generated       Version      Snapshot
        Values        Reconcile      S(n+1)
            │            │            │
            └────────────┼────────────┘
                         ▼
                     UnitOfWork
                         │
                         ▼
                  MANAGED / CLEAN
                         │
                         ▼
                    postUpdate
                         │
                  mutation occurred?
                    ┌────┴────┐
                   yes        no
                    │          │
                    ▼          ▼
             MANAGED/DIRTY  MANAGED/CLEAN

             Transaction remains separate
```

---

# 282. Master Formula

```text
Database Update Persistence System
=
Managed Entity Update Intent
+
Entity ChangeSet
+
Snapshot Baseline
+
ChangeSet Validation
+
Updatable Property Resolution
+
Dirty Field Projection
+
Identifier Immutability
+
Relationship Change Resolution
+
Persistence Dependency Integration
+
Update Predicate Construction
+
Optimistic Lock Predicates
+
Version Transition
+
Lifecycle Stabilization
+
Update Query Model Translation
+
Database Type Conversion
+
Parameter Binding
+
Generated Update Value Requirements
+
UPDATE RETURNING Capability
+
Affected Row Semantics
+
Execution Outcome Certainty
+
Optimistic Conflict Detection
+
Replay Safety
+
Generated Value Reconciliation
+
Version Reconciliation
+
Snapshot Reconciliation
+
UnitOfWork Reconciliation
+
PostUpdate Future Dirty Detection
+
Bulk Update Separation
+
External Mutation Awareness
+
Partial Entity Safety
+
Tenant/Shard Context Protection
+
Transaction Boundary Separation
+
Rollback Awareness
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

# 283. Master Rule

> **En VoltStack, un `UPDATE` ORM representa la sincronización controlada de un `ChangeSet` contra un baseline persistente conocido. La identidad establecida no se modifica, únicamente se proyectan propiedades persistentemente actualizables, las relaciones y versiones participan mediante semántica explícita, y el snapshot solo avanza cuando el resultado de ejecución es suficientemente cierto y la reconciliación ha sido completada. Un `UPDATE` exitoso tampoco constituye evidencia de que la transacción haya sido comprometida.**

---

# 284. Estado del bloque 11

Con este documento:

```text
123_DATABASE_IDENTITY_MAP_SYSTEM.md
124_DATABASE_UNIT_OF_WORK_ARCHITECTURE.md
125_DATABASE_CHANGE_TRACKING_SYSTEM.md
126_DATABASE_ENTITY_SNAPSHOT_SYSTEM.md
127_DATABASE_PERSISTENCE_ENGINE.md
128_DATABASE_PERSISTENCE_PLANNER_SYSTEM.md
129_DATABASE_INSERT_PERSISTENCE_SYSTEM.md
130_DATABASE_UPDATE_PERSISTENCE_SYSTEM.md
```

el flujo de persistencia soporta ya:

```text
                 UnitOfWork
                     │
             Persistence Engine
                     │
              Persistence Plan
                 ┌───┴───┐
                 ▼       ▼
              INSERT   UPDATE
                 │       │
                 ▼       ▼
             Execution Engine
                 │       │
                 └───┬───┘
                     ▼
                Reconciliation
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
      IdentityMap  Snapshot   UnitOfWork
```

Todavía falta formalizar la tercera operación fundamental de persistencia:

```text
DELETE
```

seguida por:

```text
FLUSH
BATCH
CONSISTENCY
```

---

# 285. Siguiente documento

```text
131_DATABASE_DELETE_PERSISTENCE_SYSTEM.md
```

El siguiente documento deberá formalizar:

```text
MANAGED Entity
      ↓
remove(entity)
      ↓
REMOVED / SCHEDULED
      ↓
Relationship Dependency Analysis
      ↓
Cascade Remove
      ↓
Orphan Removal
      ↓
DeleteEntityOperation
      ↓
Persistence Planner
      ↓
DeletePersistenceStep
      ↓
Identity Predicate
      ↓
Optimistic Lock Predicate
      ↓
Soft vs Physical Delete Strategy
      ↓
Delete Query Model
      ↓
Execution
      ↓
Affected Row Validation
      ↓
Outcome Certainty
      ↓
IdentityMap Detachment
      ↓
Snapshot Removal
      ↓
UnitOfWork Reconciliation
      ↓
DETACHED
```

incluyendo especialmente:

```text
remove() ≠ immediate DELETE
REMOVED state
physical delete
soft delete integration
cascade remove
orphan removal
relationship dependency ordering
foreign-key constraints
delete-before-insert/update dependencies
identifier preservation
optimistic locking on delete
version predicates
zero affected rows
already-missing rows
idempotent delete semantics
unknown delete outcomes
retry safety
preRemove/postRemove
postRemove ≠ commit
IdentityMap removal
snapshot disposal
detached entities
generated identifiers after deletion
rollback implications
bulk delete separation
tenant/shard protection
persistent runtime isolation
telemetry
diagnostics
extensions
testing
```

Regla central:

> **`remove()` en VoltStack registra intención de eliminación; no ejecuta inmediatamente un `DELETE`. La entidad solo abandona de forma coherente el contexto administrado cuando la operación de eliminación ha sido ejecutada con certeza suficiente y el `IdentityMap`, snapshot y `UnitOfWork` han sido reconciliados, sin confundir `postRemove` con el commit de la transacción.**