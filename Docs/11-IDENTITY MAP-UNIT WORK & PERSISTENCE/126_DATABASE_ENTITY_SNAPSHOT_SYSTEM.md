# 126_DATABASE_ENTITY_SNAPSHOT_SYSTEM.md

# VoltStack Quantum Database
## Database Entity Snapshot System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 126 — Database Entity Snapshot System  
**Bloque:** 11 — Identity Map, Unit of Work & Persistence  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Entity Snapshot System` define la arquitectura mediante la cual el ORM de VoltStack conserva una representación estable del último estado persistente conocido de una entidad dentro de un `PersistenceContext`.

Su responsabilidad principal es proporcionar el baseline utilizado por:

```text
Change Tracking
UnitOfWork
Optimistic Locking
Persistence Reconciliation
Diagnostics
```

El principio central será:

> **Un Entity Snapshot representa el baseline persistente conocido de una entidad dentro de un PersistenceContext; no representa necesariamente el estado actual de la base de datos y nunca deberá compartir estado mutable de forma que una mutación de la entidad pueda modificar retroactivamente su propio baseline.**

---

# 2. Contexto arquitectónico

La secuencia actual es:

```text
123 Identity Map
        ↓
124 Unit of Work
        ↓
125 Change Tracking
        ↓
126 Entity Snapshot
        ↓
127 Persistence Engine
```

Los documentos anteriores establecieron:

```text
IdentityMap
=
canonical managed object identity

UnitOfWork
=
pending persistence coordination

ChangeTracking
=
persistent baseline vs current state
```

El Snapshot System aporta:

```text
persistent baseline
```

---

# 3. Problema fundamental

Supongamos:

```php
$user = $repository->find(10);
```

La base de datos devuelve:

```text
id    = 10
name  = Bob
email = bob@example.com
```

Después de hydration:

```text
Entity User#10

name  = Bob
email = bob@example.com
```

VoltStack deberá conservar:

```text
Snapshot User#10

name  = Bob
email = bob@example.com
```

Posteriormente:

```php
$user->rename('Alice');
```

La entidad queda:

```text
name = Alice
```

pero el snapshot deberá permanecer:

```text
name = Bob
```

para permitir:

```text
Bob → Alice
```

---

# 4. Snapshot ≠ Entity

Regla:

```text
EntitySnapshot
≠
Entity
```

El snapshot no será:

```text
clone $entity
```

como arquitectura universal.

Representará únicamente:

```text
persistent baseline state
```

---

# 5. Snapshot ≠ database row

Aunque pueda originarse en una fila:

```text
EntitySnapshot
≠
RawDatabaseRow
```

porque el snapshot opera con:

```text
ORM persistent semantics
```

y no necesariamente con representación física del driver.

---

# 6. Snapshot ≠ current database state

Una vez cargada la entidad:

```text
T1:
DB = Bob
Snapshot = Bob
Entity = Bob
```

otro proceso podría ejecutar:

```text
T2:
DB = Charlie
```

mientras el PersistenceContext conserva:

```text
Snapshot = Bob
Entity = Bob
```

Por tanto:

```text
Snapshot
≠
LiveDatabaseState
```

---

# 7. Snapshot ≠ cache

El Snapshot System no es:

```text
Query Cache
Result Cache
Entity Cache
Second-Level Cache
```

Su función no es evitar queries entre requests.

Su función es:

```text
track baseline inside persistence scope
```

---

# 8. Snapshot ≠ Identity Map

```text
IdentityMap
```

responde:

> ¿Cuál es la instancia managed correspondiente a esta identidad?

`SnapshotRegistry` responde:

> ¿Cuál es el baseline persistente conocido para esta entidad managed?

---

# 9. Snapshot ≠ ChangeSet

```text
Snapshot
=
baseline

ChangeSet
=
difference between baseline and current state
```

---

# 10. Snapshot ≠ audit history

VoltStack no conservará automáticamente:

```text
Snapshot v1
Snapshot v2
Snapshot v3
...
```

como historial de negocio.

Eso sería:

```text
versioning
audit
temporal data
```

y pertenece a otros sistemas.

---

# 11. Snapshot ≠ serialization

Un snapshot no será:

```text
serialize($entity)
```

ni:

```text
json_encode($entity)
```

como contrato de persistencia.

---

# 12. Arquitectura general

```text
Database Result
      │
      ▼
Entity Hydrator
      │
      ├──────────────→ Entity
      │                   │
      │                   ▼
      │              Identity Map
      │
      ▼
Persistent State Extraction
      │
      ▼
Snapshot Plan
      │
      ▼
Snapshot Value Normalization
      │
      ▼
Entity Snapshot
      │
      ▼
Snapshot Registry
      │
      ├──────────────→ Change Tracking
      │
      ├──────────────→ UnitOfWork
      │
      ├──────────────→ Optimistic Locking
      │
      └──────────────→ Persistence Reconciliation
```

---

# 13. Core terminology

El sistema distinguirá:

```text
EntitySnapshot
SnapshotId
SnapshotRegistry
SnapshotEntry
SnapshotValue
SnapshotPlan
SnapshotProperty
SnapshotCompleteness
SnapshotCertainty
SnapshotOrigin
SnapshotGeneration
SnapshotFingerprint
SnapshotCopyStrategy
SnapshotLifecycle
```

---

# 14. EntitySnapshot

Objeto principal:

```php
final readonly class EntitySnapshot
{
    public function __construct(
        public SnapshotId $id,
        public EntityKey $entityKey,
        public SnapshotGeneration $generation,
        public SnapshotPropertyCollection $properties,
        public SnapshotCompleteness $completeness,
        public SnapshotOrigin $origin,
        public SnapshotFingerprint $fingerprint,
    ) {}
}
```

---

# 15. SnapshotId

`SnapshotId` identifica una instancia lógica de snapshot.

```php
final readonly class SnapshotId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 16. SnapshotId ≠ EntityKey

Una misma entidad puede tener:

```text
EntityKey = User#10
```

y diferentes snapshots a través del tiempo:

```text
Snapshot A
Snapshot B
Snapshot C
```

---

# 17. SnapshotId ≠ SnapshotGeneration

`SnapshotId` identifica el snapshot.

`SnapshotGeneration` representa su evolución dentro del contexto.

---

# 18. Snapshot generation

Ejemplo:

```text
User#10

Generation 1:
name = Bob

flush successful

Generation 2:
name = Alice
```

---

# 19. Snapshot generation ≠ entity version field

Muy importante:

```text
SnapshotGeneration
≠
OptimisticLockVersion
```

Uno es infraestructura ORM.

El otro puede ser:

```text
database concurrency token
```

---

# 20. Snapshot generation ≠ business version

Tampoco será:

```text
document revision
domain aggregate version
```

---

# 21. SnapshotProperty

```php
final readonly class SnapshotProperty
{
    public function __construct(
        public PersistentPropertyId $property,
        public SnapshotValue $value,
        public SnapshotPropertyState $state,
    ) {}
}
```

---

# 22. SnapshotPropertyState

Se propone:

```php
enum SnapshotPropertyState
{
    case LOADED;
    case NULL;
    case UNLOADED;
    case UNINITIALIZED;
    case UNKNOWN;
}
```

---

# 23. NULL ≠ UNLOADED

Regla fundamental:

```text
NULL
≠
UNLOADED
```

---

# 24. NULL ≠ UNKNOWN

También:

```text
NULL
≠
UNKNOWN
```

---

# 25. UNINITIALIZED ≠ UNLOADED

Una propiedad PHP puede existir pero no estar inicializada.

Una propiedad ORM puede no haber sido cargada.

Son conceptos distintos.

---

# 26. UNKNOWN

`UNKNOWN` significa:

> VoltStack no posee evidencia suficiente para afirmar el baseline.

Nunca significará:

```text
probably null
```

---

# 27. SnapshotValue

Se recomienda que los valores se representen mediante contratos explícitos:

```php
interface SnapshotValue
{
}
```

Implementaciones:

```text
ScalarSnapshotValue
ObjectSnapshotValue
StructuredSnapshotValue
EntityReferenceSnapshotValue
CanonicalSnapshotValue
HashedSnapshotValue
UnknownSnapshotValue
```

---

# 28. Snapshot representation

El snapshot puede almacenar:

```text
original PHP value
canonical value
structural representation
stable digest
entity reference
```

dependiendo del tipo y estrategia.

---

# 29. Snapshot representation ≠ one universal format

No todas las propiedades deberán almacenarse mediante:

```text
mixed $value
```

sin semántica adicional.

---

# 30. Snapshot plan

Cada `EntityType` deberá disponer de un:

```text
CompiledSnapshotPlan
```

derivado de metadata.

---

# 31. CompiledSnapshotPlan

```php
final readonly class CompiledSnapshotPlan
{
    public function __construct(
        public EntityType $entityType,
        public SnapshotPropertyPlanCollection $properties,
        public EmbeddedSnapshotPlanCollection $embedded,
        public RelationshipSnapshotPlanCollection $relationships,
        public SnapshotPlanFingerprint $fingerprint,
    ) {}
}
```

---

# 32. SnapshotPropertyPlan

Puede declarar:

```text
property accessor
mapped type
copy strategy
canonicalization strategy
loaded-state strategy
mutability classification
sensitivity
generated-field policy
snapshot participation
```

---

# 33. Metadata-driven

Solo se snapshotearán:

```text
persistent properties relevant to tracking/persistence
```

No:

```text
all PHP object state
```

---

# 34. Runtime reflection

No deberá redescubrirse mapping mediante Reflection en cada snapshot.

Pipeline:

```text
Entity Metadata
      ↓
Snapshot Plan Compiler
      ↓
CompiledSnapshotPlan
      ↓
Metadata Cache
      ↓
Runtime Snapshot Factory
```

---

# 35. Snapshot creation

Contrato conceptual:

```php
interface EntitySnapshotFactory
{
    public function create(
        object $entity,
        CompiledSnapshotPlan $plan,
        SnapshotCreationContext $context,
    ): EntitySnapshot;
}
```

---

# 36. Snapshot creation ≠ entity mutation

Crear snapshot será:

```text
read-only against entity state
```

---

# 37. Snapshot creation side effects

No deberá:

```text
execute SQL
lazy-load relations
dispatch domain commands
change tenant
modify entity
flush UnitOfWork
```

---

# 38. Snapshot origins

```php
enum SnapshotOrigin
{
    case HYDRATION;
    case PERSISTENCE_RECONCILIATION;
    case EXPLICIT_REFRESH;
    case REATTACH;
    case GENERATED;
    case CUSTOM;
}
```

---

# 39. HYDRATION

Snapshot creado a partir del estado cargado por ORM.

---

# 40. PERSISTENCE_RECONCILIATION

Snapshot establecido después de una operación de persistencia cuyo resultado sea suficientemente cierto.

---

# 41. EXPLICIT_REFRESH

Después de:

```php
$entityManager->refresh($user);
```

puede sustituirse el baseline.

---

# 42. REATTACH

Si una futura versión soporta reattachment, deberá definir explícitamente cómo se establece el nuevo baseline.

---

# 43. Snapshot completeness

```php
enum SnapshotCompleteness
{
    case COMPLETE;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 44. COMPLETE

Todos los persistent members necesarios para el tracking completo están representados.

---

# 45. PARTIAL

Solo una parte fue cargada o puede afirmarse.

---

# 46. UNKNOWN

VoltStack no puede establecer si el baseline está completo.

---

# 47. Completeness ≠ entity validity

Una entidad parcial puede ser válida para:

```text
read projection-like use
partial persistence
specific operations
```

dependiendo de política.

---

# 48. Partial snapshot

Ejemplo:

```text
EntitySnapshot User#10

id:
    LOADED = 10

name:
    LOADED = Alice

email:
    UNLOADED

address:
    UNLOADED

status:
    LOADED = ACTIVE

Completeness:
PARTIAL
```

---

# 49. Partial snapshot safety

Nunca:

```text
UNLOADED
→ NULL
```

---

# 50. Snapshot certainty

Además de completeness puede existir:

```php
enum SnapshotCertainty
{
    case CERTAIN;
    case PARTIALLY_CERTAIN;
    case UNKNOWN;
}
```

---

# 51. Completeness ≠ certainty

Ejemplo:

```text
all fields represented
```

pero algún valor proviene de una fuente cuya ejecución terminó con:

```text
UNKNOWN outcome
```

Entonces podría existir:

```text
Completeness = COMPLETE
Certainty = UNKNOWN
```

---

# 52. Snapshot certainty after hydration

Una query exitosa normal puede producir:

```text
CERTAIN
```

respecto al resultado observado.

Eso no significa que la DB no cambie inmediatamente después.

---

# 53. Snapshot freshness

Opcionalmente:

```php
final readonly class SnapshotObservation
{
    public function __construct(
        public SnapshotOrigin $origin,
        public ?DateTimeImmutable $observedAt,
        public ?ExecutionId $execution,
    ) {}
}
```

---

# 54. observedAt ≠ current DB guarantee

Timestamp solo indica:

```text
when state was observed
```

---

# 55. Snapshot immutability

Una vez construido:

```text
EntitySnapshot
```

será immutable.

---

# 56. Why immutability

Permite:

```text
deterministic Change Tracking
safe fingerprints
stable diagnostics
predictable UnitOfWork
```

---

# 57. Snapshot replacement

El snapshot no se modifica campo por campo como estado compartido.

Preferencia:

```text
SnapshotGeneration N
        ↓
new immutable snapshot
        ↓
SnapshotGeneration N+1
```

---

# 58. Registry replacement

```text
SnapshotRegistry
User#10 → SnapshotGeneration 1

successful flush

SnapshotRegistry
User#10 → SnapshotGeneration 2
```

---

# 59. Old snapshot retention

Por defecto:

```text
Generation 1
```

podrá liberarse después de una transición segura.

No será historial permanente.

---

# 60. Snapshot registry

```php
interface EntitySnapshotRegistry
{
    public function has(EntityKey $key): bool;

    public function get(EntityKey $key): EntitySnapshot;

    public function replace(
        EntityKey $key,
        EntitySnapshot $snapshot,
    ): void;

    public function remove(EntityKey $key): void;

    public function clear(): void;
}
```

---

# 61. Registry ownership

El registry pertenecerá a:

```text
PersistenceContext / UnitOfWork scope
```

---

# 62. Registry ≠ process-global

Nunca:

```php
static array $snapshots;
```

---

# 63. EntityKey association

Normalmente:

```text
EntityKey
→ EntitySnapshot
```

pero NEW entities sin ID persistente requieren tratamiento especial.

---

# 64. NEW entities

Una entidad NEW puede no tener:

```text
established EntityKey
```

si su ID será generado por DB.

---

# 65. Temporary UnitOfWork identity

Doc 123/124 permite utilizar:

```text
UnitOfWork internal object identity
```

para rastrear NEW entities.

Por tanto:

```text
ManagedEntityHandle
→ optional EntityKey
→ optional EntitySnapshot
```

---

# 66. NEW entity snapshot

Por defecto, una NEW entity no necesita un baseline de fila existente.

---

# 67. Initial state ≠ snapshot

Su estado inicial pertenece principalmente al:

```text
Insert Persistence System
```

---

# 68. Optional new-entity snapshot

Podría crearse temporalmente para:

```text
lifecycle stabilization
mutation diagnostics
```

pero deberá marcarse:

```text
not a persisted baseline
```

---

# 69. PersistedBaseline flag

Si se requiere:

```php
enum SnapshotBaselineKind
{
    case PERSISTED;
    case TRANSIENT;
    case RECONCILED;
}
```

---

# 70. Transient snapshot ≠ persisted snapshot

Nunca deberá utilizarse para afirmar:

```text
this value existed in database
```

---

# 71. Hydration integration

Pipeline recomendado:

```text
Database Result
      ↓
Hydration Plan
      ↓
Entity Instantiation
      ↓
Persistent Field Assignment
      ↓
Relationship Initialization
      ↓
Lifecycle Hydration Phase
      ↓
Snapshot Establishment
      ↓
IdentityMap / UnitOfWork Registration
```

El orden exacto se coordinará con Hydration y Lifecycle.

---

# 72. Snapshot before IdentityMap?

Implementación puede necesitar registrar identidad temprano para resolver ciclos.

Por tanto, la arquitectura lógica puede ser:

```text
instantiate
→ establish provisional identity
→ register hydration identity
→ hydrate
→ finalize snapshot
→ mark managed
```

---

# 73. Provisional hydration registration

No deberá confundirse con:

```text
fully managed synchronized entity
```

---

# 74. Hydration failure

Si hydration falla antes de completar baseline:

```text
no valid managed snapshot
```

deberá publicarse.

---

# 75. Atomic publication

Preferencia:

> Construir el snapshot fuera del registry y publicarlo únicamente cuando sea válido.

---

# 76. Snapshot publication

```text
Build
→ Validate
→ Fingerprint
→ Publish
```

---

# 77. Invalid snapshot

Nunca deberá almacenarse parcialmente como si fuera válido.

---

# 78. Lifecycle callbacks

El momento exacto del baseline respecto a `postLoad` deberá ser determinista.

V1 recomendado:

```text
Hydrate
→ controlled postLoad normalization
→ Snapshot
→ Managed baseline
```

---

# 79. Domain mutation during postLoad

Si `postLoad` ejecuta una verdadera mutación de negocio, deberá existir una política explícita.

No deberá quedar ambiguo si forma parte del baseline.

---

# 80. Snapshot extraction

Snapshot Factory utilizará:

```text
CompiledPropertyAccessor
```

compatible con Change Tracking.

---

# 81. Shared accessor semantics

Snapshot y Change Tracking deberán utilizar semántica compatible.

Incorrecto:

```text
Snapshot reads field A one way
ChangeTracker reads field A another way
```

---

# 82. Canonical snapshot representation

Para valores simples puede almacenarse:

```text
canonical persistent value
```

---

# 83. Canonical representation benefits

```text
smaller comparison logic
deterministic fingerprinting
stable type comparison
less aliasing
```

---

# 84. Canonical representation risk

No deberá perder información necesaria para:

```text
correct comparison
optimistic locking
persistence planning
```

---

# 85. Snapshot copy strategies

```php
enum SnapshotCopyStrategyType
{
    case DIRECT;
    case SHALLOW_COPY;
    case DEEP_COPY;
    case CANONICALIZE;
    case STRUCTURAL;
    case HASH;
    case CUSTOM;
}
```

---

# 86. DIRECT

Adecuado cuando el valor es realmente immutable.

Ejemplos:

```text
int
string
bool
immutable enum
```

---

# 87. SHALLOW_COPY

Solo cuando el contrato demuestre que las referencias internas no permiten alterar el baseline.

---

# 88. DEEP_COPY

Puede utilizarse para estructuras mutables pequeñas.

---

# 89. Deep copy ≠ serialize/unserialize

No se usará:

```php
unserialize(serialize($value))
```

como mecanismo ORM universal.

---

# 90. CANONICALIZE

Transforma el valor a una representación immutable de comparación.

Ejemplo:

```text
DateTime object
→ canonical timestamp + precision semantics
```

---

# 91. STRUCTURAL

Adecuado para:

```text
embedded Value Objects
JSON
nested persistent structures
```

---

# 92. HASH

Adecuado potencialmente para valores grandes.

---

# 93. HASH snapshot

```text
Large Value
    ↓
Canonicalization
    ↓
Digest
    ↓
HashedSnapshotValue
```

---

# 94. Hash-only limitation

Si igualdad exacta es requerida:

```text
same hash
```

no deberá asumirse universalmente como prueba matemática de igualdad.

---

# 95. Hybrid snapshot

Puede conservar:

```text
digest
+
additional verification metadata
```

o realizar comparación exacta cuando sea necesario.

---

# 96. CUSTOM

Permitirá estrategias especializadas bajo contratos deterministas.

---

# 97. SnapshotCopyStrategy

```php
interface SnapshotCopyStrategy
{
    public function snapshot(
        mixed $value,
        SnapshotValueContext $context,
    ): SnapshotValue;
}
```

---

# 98. Strategy purity

No deberá:

```text
query database
modify entity
perform HTTP
access mutable global state
change runtime context
```

---

# 99. Mutable aliasing problem

Supongamos:

```php
$preferences = $user->preferences();
```

Snapshot incorrecto:

```text
Entity.preferences ─────┐
                       ├── same mutable object
Snapshot.preferences ──┘
```

Después:

```php
$preferences->setTheme('dark');
```

ambos parecen contener:

```text
dark
```

y el cambio desaparece.

---

# 100. Required invariant

```text
Mutable Current Value
```

y:

```text
Snapshot Baseline
```

no podrán compartir mutabilidad observable que invalide comparación.

---

# 101. Immutable Value Objects

Para un Value Object contractualmente immutable:

```text
same object reference
```

puede ser segura si el objeto no puede mutar.

---

# 102. Runtime immutability claim

El framework no deberá asumir que una clase es immutable solo porque:

```text
no setters were discovered
```

---

# 103. Immutable metadata

La mutabilidad deberá declararse/inferirse bajo reglas verificables.

---

# 104. PHP readonly

`readonly` puede ser evidencia útil, pero:

```text
readonly property
```

no garantiza que el objeto referenciado sea profundamente immutable.

---

# 105. Structural mutability

Ejemplo:

```php
readonly class Preferences
{
    public function __construct(
        public MutableCollection $items,
    ) {}
}
```

El objeto externo es readonly, pero su grafo puede ser mutable.

---

# 106. PersistentValueMutability

Reutilizará la clasificación del documento 125:

```text
IMMUTABLE
MUTABLE
STRUCTURALLY_MUTABLE
UNKNOWN
```

---

# 107. Arrays

PHP arrays tienen copy-on-write, pero el framework no deberá depender de detalles accidentales sin un contrato claro.

---

# 108. Snapshot array

Una representación estructural immutable puede ser preferible.

---

# 109. JSON snapshots

Para JSON:

```text
PHP array/object
      ↓
Canonical JSON Structure
      ↓
Immutable Snapshot
```

---

# 110. JSON object keys

Podrán canonicalizarse.

---

# 111. JSON array order

Deberá preservarse.

---

# 112. Decimal snapshots

No convertir a float.

Ejemplo:

```text
Decimal
→ canonical decimal representation
```

---

# 113. Date/time snapshots

Podrán conservar:

```text
instant
timezone semantics
precision
```

según mapping.

---

# 114. Enum snapshots

Podrán conservar:

```text
canonical enum persistence value
```

sin guardar necesariamente toda la instancia.

---

# 115. Identifier snapshots

Identificadores establecidos deberán formar parte del baseline de identidad.

---

# 116. Identifier snapshot ≠ ordinary field snapshot

Su mutación tiene semántica especial.

---

# 117. EntityIdentitySnapshot

Opcionalmente:

```php
final readonly class EntityIdentitySnapshot
{
    public function __construct(
        public EntityType $type,
        public EntityIdentifier $identifier,
        public EntityIdentityNamespace $namespace,
    ) {}
}
```

---

# 118. Identity snapshot purpose

Permite verificar:

```text
EntityKey did not mutate
```

---

# 119. Generated identifier reconciliation

Para NEW entity:

```text
Before INSERT:
id = UNASSIGNED

INSERT succeeds:
id = 101

Result System:
generated id = 101
```

Después:

```text
assign generated identifier
establish EntityKey
register IdentityMap
create persisted snapshot
```

---

# 120. Generated ID certainty

Solo después de un outcome suficientemente cierto.

---

# 121. UNKNOWN insert outcome

Si:

```text
INSERT may have committed
connection lost before confirmation
```

VoltStack no deberá crear un snapshot afirmando:

```text
persisted id = X
state synchronized
```

sin evidencia suficiente.

---

# 122. UNKNOWN snapshot transition

El UnitOfWork/EntityManager podrá quedar:

```text
TAINTED
REQUIRES_RECOVERY
```

según Persistence Outcome rules.

---

# 123. Database-generated fields

Después de INSERT/UPDATE pueden existir:

```text
generated ID
generated timestamp
computed value
version token
database default
RETURNING value
```

---

# 124. Snapshot reconciliation

El nuevo snapshot deberá incorporar:

```text
confirmed database-generated values
```

cuando estén disponibles.

---

# 125. Missing generated values

Si la DB genera un valor pero el ORM no lo recupera:

```text
baseline may be PARTIAL
```

o requerir:

```text
refresh
```

según mapping.

---

# 126. Database-authoritative values

Si una propiedad es:

```text
DATABASE_AUTHORITATIVE
```

el snapshot no deberá reemplazarse con un valor local no confirmado.

---

# 127. Snapshot advancement after flush

Pipeline:

```text
Old Snapshot
      +
Stable ChangeSet
      +
Persistence Plan
      +
Execution Results
      ↓
Outcome Reconciliation
      ↓
New Snapshot
```

---

# 128. Critical rule

> Un snapshot solo avanzará cuando VoltStack tenga evidencia suficiente de que el baseline nuevo representa un estado persistente válido.

---

# 129. Flush success

Si la persistencia fue confirmada:

```text
Snapshot N
→ Snapshot N+1
```

---

# 130. Flush failure before effects

Si se sabe:

```text
no database effect occurred
```

mantener:

```text
Snapshot N
```

---

# 131. Partial persistence

Si algunas operaciones fueron confirmadas y otras no:

```text
single simplistic snapshot replacement
```

puede ser incorrecto.

---

# 132. Partial reconciliation

Persistence Reconciliation deberá determinar por entidad:

```text
CONFIRMED_UNCHANGED
CONFIRMED_UPDATED
PARTIAL
UNKNOWN
```

---

# 133. Snapshot reconciliation state

```php
enum SnapshotReconciliationState
{
    case PRESERVE;
    case ADVANCE;
    case PARTIAL;
    case INVALIDATE;
    case UNKNOWN;
}
```

---

# 134. UNKNOWN persistence outcome

Nunca:

```text
assume success
```

ni:

```text
assume failure
```

para actualizar baseline.

---

# 135. Invalidation

Cuando el baseline deja de ser confiable:

```text
SnapshotState = INVALID
```

o el EntityManager puede quedar tainted.

---

# 136. Invalid snapshot

Un snapshot invalidado no deberá utilizarse para:

```text
normal dirty checking
```

como si fuera confiable.

---

# 137. Snapshot lifecycle state

Se propone:

```php
enum EntitySnapshotState
{
    case BUILDING;
    case VALID;
    case SUPERSEDED;
    case INVALID;
    case UNKNOWN;
}
```

---

# 138. Immutable snapshot vs lifecycle state

Para preservar immutability, el lifecycle state puede residir en:

```text
SnapshotRegistryEntry
```

y no dentro del snapshot.

---

# 139. SnapshotRegistryEntry

```php
final class SnapshotRegistryEntry
{
    public function __construct(
        public readonly EntityKey $key,
        private EntitySnapshot $snapshot,
        private EntitySnapshotState $state,
    ) {}
}
```

---

# 140. SUPERSEDED

Un snapshot reemplazado deja de ser baseline actual.

---

# 141. Transaction integration

Snapshot advancement y transaction commit requieren cuidado.

---

# 142. flush() ≠ commit()

Como ya estableció ORM Architecture:

```text
flush()
≠
transaction commit()
```

---

# 143. Example transaction

```php
$tx->begin();

$user->rename('Alice');

$em->flush();

$tx->rollback();
```

Después de `flush()`:

```text
database inside transaction:
Alice
```

pero después de rollback:

```text
database:
Bob
```

---

# 144. Snapshot problem

Si inmediatamente después de flush se cambia baseline a:

```text
Alice
```

y luego transaction rollback:

```text
snapshot must not remain Alice
```

---

# 145. Transaction-aware snapshots

VoltStack necesitará integrar snapshots con:

```text
TransactionContext
```

---

# 146. Transaction-local baseline

Una estrategia robusta:

```text
Committed Baseline
        +
Transaction-local Snapshot Advancement
```

---

# 147. Snapshot transaction frame

Conceptualmente:

```text
Transaction
├── entry baseline
├── current transactional baseline
└── reconciliation log
```

---

# 148. Flush inside transaction

Puede avanzar:

```text
transaction-local baseline
```

sin afirmar todavía:

```text
committed external state
```

---

# 149. Transaction commit

Al commit confirmado:

```text
transaction-local baseline
→ accepted baseline
```

---

# 150. Transaction rollback

Al rollback confirmado:

```text
restore transaction-entry baseline
```

pero también debe reconciliar:

```text
entity in-memory state
```

---

# 151. Entity state after rollback

Importante:

DB rollback no significa automáticamente:

```text
entity object reverted
```

---

# 152. Example

```text
Before:
Entity = Bob
Snapshot = Bob

Change:
Entity = Alice

flush in transaction:
DB(tx) = Alice

rollback:
DB = Bob
Entity = Alice
```

Entonces:

```text
Snapshot = Bob
Entity = Alice
```

y Change Tracking deberá volver a detectar:

```text
Bob → Alice
```

---

# 153. Correct rollback behavior

No deberá resetear silenciosamente la entidad a Bob salvo política explícita.

---

# 154. Nested transactions/savepoints

El Snapshot System deberá soportar conceptualmente:

```text
transaction snapshot frames
```

o mecanismos equivalentes.

Detalles se formalizarán en:

```text
168_DATABASE_NESTED_TRANSACTION_SYSTEM.md
169_DATABASE_SAVEPOINT_SYSTEM.md
```

---

# 155. Savepoint rollback

Puede requerir restaurar:

```text
transactional baseline generation
```

al frame correspondiente.

---

# 156. Snapshot frame ≠ PHP object clone

Se almacenará únicamente el baseline ORM necesario.

---

# 157. Transaction outcome UNKNOWN

Si commit outcome es desconocido:

```text
snapshot certainty = UNKNOWN
```

o manager tainted.

Nunca se promoverá silenciosamente un baseline.

---

# 158. Optimistic locking

Snapshot deberá conservar:

```text
original version token
```

---

# 159. Version snapshot

Ejemplo:

```text
version = 7
```

Luego Persistence Planner puede construir:

```text
expectedVersion = 7
nextVersion = 8
```

---

# 160. Successful version update

Después de outcome confirmado:

```text
snapshot.version = 8
```

---

# 161. Optimistic lock failure

Si:

```text
UPDATE ... WHERE version = 7
```

afecta cero filas:

```text
snapshot remains version 7
```

pero la entidad/contexto puede requerir:

```text
refresh
conflict resolution
detach
```

---

# 162. Version token ≠ snapshot generation

Reiteración:

```text
version 7
```

puede coincidir accidentalmente con:

```text
SnapshotGeneration 7
```

pero son dominios independientes.

---

# 163. Relationships

Snapshots de relaciones to-one deberán preferir:

```text
EntityKey / EntityReference
```

en lugar de guardar todo el objeto relacionado.

---

# 164. To-one snapshot

Ejemplo:

```text
Order.customer
→ EntityReference(Customer#20)
```

---

# 165. Relationship object reference

No deberá depender de:

```text
spl_object_id()
```

como identidad persistente.

---

# 166. Unloaded relationship

Se conserva:

```text
UNLOADED
```

sin lazy-load.

---

# 167. Missing relationship

Debe distinguirse:

```text
NULL
```

de:

```text
UNLOADED
```

y:

```text
MISSING_TARGET
```

cuando dicha información sea conocida.

---

# 168. To-many snapshots

Las colecciones requieren arquitectura especializada.

---

# 169. Collection snapshot

Puede representar:

```text
known member EntityKeys
collection initialization state
ordering information
collection version/generation
```

---

# 170. Unloaded collection

No deberá snapshotearse cargándola.

---

# 171. Partial collection

Si una colección fue cargada parcialmente:

```text
LIMIT
filtered relation
slice
```

no deberá presentarse como snapshot completo de membership.

---

# 172. Collection completeness

Puede necesitar:

```text
COMPLETE
PARTIAL
UNLOADED
UNKNOWN
```

independiente de entity snapshot completeness.

---

# 173. Collection ordering

Para relaciones ordenadas:

```text
membership
+
order
```

pueden ser persistentes.

---

# 174. Collection snapshot ≠ collection object

No conservará necesariamente:

```text
PersistentCollection instance
```

como baseline.

---

# 175. Relationship ownership

Solo relaciones relevantes para persistence tracking deberán influir en snapshots.

---

# 176. Inverse relation

Una relación inverse puede necesitar tracking para:

```text
graph consistency
cascade
orphan detection
```

aunque no produzca directamente FK UPDATE.

---

# 177. Snapshot scope

Cada snapshot pertenece a un:

```text
PersistenceContextId
```

de forma lógica.

---

# 178. Same EntityKey across contexts

Dos EntityManagers pueden tener:

```text
User#10
```

cada uno con snapshot distinto.

Esto es válido.

---

# 179. Snapshot context identity

Por tanto:

```text
SnapshotRegistryKey
=
PersistenceContext
+
EntityKey
```

aunque el registry ya esté físicamente scoped y no necesite duplicar el context ID en cada key.

---

# 180. Tenant identity

Como `EntityKey` incorpora effective identity namespace:

```text
Tenant A / User#10
≠
Tenant B / User#10
```

---

# 181. No hardcoded tenant_id

Snapshot core no asumirá:

```text
tenant_id
```

---

# 182. Shard identity

La misma regla aplica a:

```text
shard
database target
logical persistence namespace
```

---

# 183. Refresh

`EntityManager::refresh()` puede:

```text
query current database state
hydrate managed entity
replace snapshot
```

---

# 184. Refresh ≠ Snapshot System query

Snapshot System por sí mismo no ejecuta la query.

Pipeline:

```text
EntityManager
    ↓
Entity Loader
    ↓
Query Engine
    ↓
Hydration
    ↓
Snapshot Replacement
```

---

# 185. Refresh discards local changes

Dependiendo de API, `refresh()` puede reemplazar estado local.

Debe ser explícito.

---

# 186. Snapshot refresh conflict

Si existen cambios locales:

```text
refresh
```

podrá:

```text
REJECT
DISCARD_LOCAL
MERGE_EXPLICITLY
```

según policy.

---

# 187. No implicit merge

V1 deberá evitar:

```text
magical database/local merge
```

---

# 188. Snapshot invalidation

Puede ocurrir por:

```text
raw SQL mutation
bulk update
bulk delete
external refresh
unknown persistence outcome
cross-context mutation
transaction uncertainty
manual database operation
```

---

# 189. Bulk operations

Como estableció ORM Architecture:

```text
bulk update/delete
```

puede dejar managed entities stale.

---

# 190. Bulk snapshot policy

Opciones:

```php
enum BulkSnapshotPolicy
{
    case CLEAR_AFFECTED;
    case INVALIDATE_AFFECTED;
    case REFRESH_AFFECTED;
    case REJECT_IF_MANAGED;
    case ALLOW_STALE_EXPLICITLY;
}
```

---

# 191. Recommended default

Para operaciones donde no pueda conocerse exactamente el nuevo estado:

```text
INVALIDATE_AFFECTED
```

o:

```text
CLEAR_AFFECTED
```

serán más seguros que fingir sincronización.

---

# 192. Raw SQL

Si el usuario modifica una tabla mediante raw SQL:

```text
Snapshot System
```

no podrá inferir automáticamente qué entidades cambiaron.

---

# 193. Explicit invalidation API

Podrá existir:

```php
$entityManager->invalidate(User::class, $id);
```

o:

```php
$entityManager->clear(User::class);
```

según API final.

---

# 194. Snapshot invalidation ≠ entity deletion

Invalidar baseline significa:

```text
we no longer trust synchronization state
```

no:

```text
row deleted
```

---

# 195. Read-only entities

Para:

```text
READ_ONLY
```

el ORM puede omitir snapshots.

---

# 196. Why

Si no habrá:

```text
dirty checking
flush persistence
```

no se necesita baseline completo.

---

# 197. Identity-only snapshot

Algunas estrategias read-only podrían conservar únicamente:

```text
EntityKey
```

para IdentityMap.

Eso no será un full persistence snapshot.

---

# 198. Detached-after-yield

Streaming:

```text
hydrate
yield
detach
```

puede liberar inmediatamente snapshot.

---

# 199. Snapshot lifetime

Normalmente:

```text
managed lifetime
```

dentro del PersistenceContext.

---

# 200. Detach

Al hacer:

```php
$entityManager->detach($entity);
```

deberá removerse:

```text
IdentityMap entry
UnitOfWork entry
SnapshotRegistry entry
```

según ownership architecture.

---

# 201. clear()

```php
$entityManager->clear();
```

deberá eliminar snapshots scoped.

---

# 202. close()

También.

---

# 203. Request termination

En FrankenPHP:

```text
request end
→ clear EntityManager
→ clear UnitOfWork
→ clear IdentityMap
→ clear SnapshotRegistry
```

---

# 204. No implicit flush

El cleanup del snapshot system nunca deberá:

```text
flush pending changes
```

---

# 205. Persistent runtime architecture

```text
FrankenPHP Worker
│
├── Shared Immutable
│   ├── Snapshot Plans
│   ├── Copy Strategies
│   ├── Type Definitions
│   └── Frozen Registries
│
├── Request A
│   └── PersistenceContext A
│       └── SnapshotRegistry A
│
└── Request B
    └── PersistenceContext B
        └── SnapshotRegistry B
```

---

# 206. RoadRunner

Misma arquitectura:

```text
worker reusable
mutable snapshots scoped
```

---

# 207. OpenSwoole

Especial cuidado con:

```text
coroutines
```

Cada logical execution scope deberá tener su propio registry.

---

# 208. No static snapshot cache

Prohibido:

```php
private static array $snapshots = [];
```

---

# 209. Memory architecture

Snapshot memory puede convertirse en un coste importante:

```text
SnapshotMemory
≈
Σ ManagedEntities × PersistentSnapshotSize
```

---

# 210. Large UnitOfWork problem

Si se gestionan:

```text
100,000 entities
```

los snapshots pueden consumir memoria significativa.

---

# 211. Operational guidance

Para procesos grandes:

```text
load chunk
process
flush
clear
repeat
```

---

# 212. Compact snapshots

VoltStack podrá optimizar mediante:

```text
typed compact values
interned metadata keys
canonical scalar representation
hashing large immutable structures
shared immutable constants
bitsets for loaded-state
```

---

# 213. Property IDs

No será necesario almacenar repetidamente strings largos si el compiled plan permite:

```text
property slot index
```

---

# 214. Snapshot slots

Ejemplo:

```text
slot 0 → id
slot 1 → name
slot 2 → email
slot 3 → version
```

Runtime snapshot:

```text
[10, "Alice", "a@example.com", 7]
```

más loaded-state bitmap.

---

# 215. Internal compact form

Puede existir:

```php
final readonly class CompactEntitySnapshot
{
    public function __construct(
        public SnapshotLayoutId $layout,
        public array $values,
        public SnapshotStateBitmap $states,
    ) {}
}
```

---

# 216. Public diagnostics

Deberán traducir slots nuevamente a:

```text
PersistentPropertyId
```

---

# 217. Optimization ≠ leaked API

Los slot indexes son implementación interna.

---

# 218. Snapshot layout

```text
CompiledSnapshotPlan
→ SnapshotLayout
```

podrá compartirse entre snapshots del mismo EntityType.

---

# 219. Layout version

Cambios en metadata pueden producir:

```text
SnapshotLayoutVersion
```

---

# 220. Snapshot compatibility

Un snapshot creado con:

```text
layout v1
```

no deberá compararse ciegamente con:

```text
tracking plan v2
```

---

# 221. Metadata fingerprint

Snapshot deberá estar asociado directa o indirectamente a:

```text
EntityMetadataFingerprint
SnapshotPlanFingerprint
```

---

# 222. Stale metadata detection

En desarrollo/hot reload:

```text
snapshot plan changed
```

puede requerir:

```text
clear EntityManager
```

---

# 223. Production immutable metadata

En producción:

```text
metadata registry frozen
```

evita ese problema durante una request.

---

# 224. Snapshot fingerprint

Se propone:

```php
final readonly class SnapshotFingerprint
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 225. Fingerprint inputs

Puede incluir:

```text
EntityKey
SnapshotGeneration
SnapshotLayout
Property states
Canonical persistent values
Relevant metadata fingerprint
```

---

# 226. Sensitive values

No deberán incluirse en claro en fingerprints.

---

# 227. Fingerprint purposes

```text
stale baseline detection
debugging
transaction frame verification
determinism tests
reconciliation
telemetry correlation
```

---

# 228. Fingerprint ≠ optimistic version

No sustituye:

```text
version column
```

---

# 229. Fingerprint ≠ cryptographic audit proof

A menos que otro sistema añada ese contrato.

---

# 230. Determinism

Mismos:

```text
entity persistent state
snapshot plan
type semantics
loaded-state information
context
```

deberán producir snapshot semánticamente equivalente.

---

# 231. Snapshot ordering

Las propiedades deberán seguir:

```text
compiled stable order
```

---

# 232. No reflection-order dependency

Reflection order no será contrato de snapshot.

---

# 233. Snapshot serialization

Puede existir una representación interna para:

```text
debugging
testing
compiled artifacts
```

pero:

```text
PHP serialize()
```

no será contrato.

---

# 234. Cross-process snapshots

V1 no deberá mover snapshots de EntityManager entre workers.

---

# 235. Why

Contienen baseline scoped a:

```text
specific persistence context
specific managed identity
specific transaction state
```

---

# 236. Queue serialization

Serializar una Entity y su snapshot hacia un Job será anti-pattern.

Preferir:

```text
EntityType + EntityIdentifier
```

y recargar en el nuevo contexto.

---

# 237. EntityReference ≠ Snapshot

Para jobs:

```text
EntityReference
```

es apropiado.

`EntitySnapshot` no.

---

# 238. Snapshot extension system

Se podrán extender:

```text
copy strategies
snapshot value types
embedded snapshot strategies
large-value strategies
diagnostic renderers
```

---

# 239. Extension registry

```text
SnapshotCopyStrategyRegistry
SnapshotValueFactoryRegistry
SnapshotExtensionRegistry
```

---

# 240. Frozen registries

Durante runtime normal:

```text
frozen
```

---

# 241. No last-wins

Dos estrategias con la misma resolución deberán producir:

```text
SnapshotStrategyCollisionException
```

---

# 242. Extension safety

Una extensión no podrá:

```text
execute SQL
lazy-load entity
mutate entity
change PersistenceContext
hide unknown values
convert unloaded to null
bypass sensitive redaction
```

---

# 243. Custom snapshot value

Debe ser:

```text
immutable
deterministic
comparison-compatible
```

---

# 244. Telemetry

Métricas:

```text
orm.snapshot.created
orm.snapshot.replaced
orm.snapshot.invalidated
orm.snapshot.partial
orm.snapshot.unknown
orm.snapshot.bytes.estimated
orm.snapshot.properties
orm.snapshot.copy.duration
orm.snapshot.canonicalization.duration
orm.snapshot.registry.size
orm.snapshot.large_value.hashes
orm.snapshot.transaction.frames
orm.snapshot.reconciliation.advanced
orm.snapshot.reconciliation.preserved
orm.snapshot.reconciliation.unknown
```

---

# 245. Tracing

Spans opcionales:

```text
orm.snapshot.create
orm.snapshot.reconcile
orm.snapshot.refresh
orm.snapshot.invalidate
```

---

# 246. Telemetry cardinality

No usar:

```text
entity ID
email
customer number
```

como metric labels de alta cardinalidad.

---

# 247. Diagnostics

Ejemplo:

```text
Entity:
App\Entity\User

Entity Key:
User#10

Snapshot:
Generation 4

Origin:
HYDRATION

Completeness:
PARTIAL

Certainty:
CERTAIN

Properties:
id       LOADED
name     LOADED
email    UNLOADED
status   LOADED
version  LOADED

Fingerprint:
8a91...
```

---

# 248. Sensitive diagnostics

```text
passwordHash:
LOADED [REDACTED]
```

---

# 249. Snapshot diff diagnostics

El Snapshot System podrá mostrar baseline, pero:

```text
snapshot diff
```

pertenece conceptualmente a Change Tracking.

---

# 250. Resource governance

Configuración:

```php
final readonly class SnapshotResourceLimits
{
    public function __construct(
        public int $maxSnapshotBytesPerEntity,
        public int $maxSnapshotBytesPerContext,
        public int $maxEmbeddedDepth,
        public int $maxCollectionMembers,
        public int $largeValueThreshold,
    ) {}
}
```

---

# 251. Limit exceeded

Nunca truncar silenciosamente.

Opciones:

```text
reject
switch to approved compact strategy
hash under explicit contract
mark unsupported
```

---

# 252. Snapshot budget

El UnitOfWork puede conocer:

```text
current estimated snapshot memory
```

para diagnostics/resource governance.

---

# 253. Automatic degradation

No deberá cambiar de:

```text
exact snapshot
```

a:

```text
lossy hash snapshot
```

silenciosamente por presión de memoria.

---

# 254. Security

Snapshots pueden contener:

```text
password hashes
tokens
PII
financial data
private metadata
```

---

# 255. Memory sensitivity

Aunque no se escriban en logs, permanecen en memoria.

El sistema deberá minimizar:

```text
unnecessary copies
long retention
debug exposure
```

---

# 256. Snapshot encryption in memory

No será requisito universal del core.

Si existe una extensión especializada, deberá reconocer que:

```text
application needs plaintext/canonical values to operate
```

y no prometer protección absoluta de memoria.

---

# 257. Redaction

Aplica a:

```text
debugging
exceptions
telemetry
profiler
developer toolbar
```

---

# 258. Snapshot access

La API pública de aplicación no debería exponer libremente snapshots internos.

---

# 259. Snapshot inspection API

Podrá existir para:

```text
debugging
testing
framework extensions
```

bajo contratos controlados.

---

# 260. Error hierarchy

```text
DatabaseOrmException
└── EntitySnapshotException
    ├── SnapshotCreationException
    ├── SnapshotPlanException
    ├── SnapshotMetadataException
    ├── SnapshotValueException
    ├── SnapshotCopyException
    ├── SnapshotCanonicalizationException
    ├── SnapshotMutableAliasingException
    ├── SnapshotUnknownValueException
    ├── SnapshotPartialStateException
    ├── SnapshotRegistryException
    ├── SnapshotNotFoundException
    ├── SnapshotAlreadyExistsException
    ├── SnapshotReplacementException
    ├── SnapshotInvalidationException
    ├── SnapshotReconciliationException
    ├── SnapshotOutcomeUncertaintyException
    ├── SnapshotTransactionException
    ├── SnapshotTransactionFrameException
    ├── SnapshotLayoutException
    ├── SnapshotLayoutMismatchException
    ├── SnapshotFingerprintException
    ├── SnapshotStrategyException
    ├── SnapshotStrategyCollisionException
    ├── SnapshotResourceLimitException
    ├── SnapshotRuntimeIsolationException
    └── SnapshotInvariantException
```

---

# 261. Directory architecture

```text
src/Quantum/Database/ORM/
│
├── Snapshot/
│   ├── Contract/
│   │   ├── EntitySnapshotFactory.php
│   │   ├── EntitySnapshotRegistry.php
│   │   ├── SnapshotCopyStrategy.php
│   │   ├── SnapshotValueFactory.php
│   │   └── SnapshotReconciler.php
│   │
│   ├── Core/
│   │   ├── EntitySnapshot.php
│   │   ├── SnapshotId.php
│   │   ├── SnapshotGeneration.php
│   │   ├── SnapshotOrigin.php
│   │   ├── SnapshotBaselineKind.php
│   │   ├── SnapshotCompleteness.php
│   │   ├── SnapshotCertainty.php
│   │   └── SnapshotObservation.php
│   │
│   ├── Property/
│   │   ├── SnapshotProperty.php
│   │   ├── SnapshotPropertyCollection.php
│   │   ├── SnapshotPropertyState.php
│   │   └── SnapshotStateBitmap.php
│   │
│   ├── Value/
│   │   ├── SnapshotValue.php
│   │   ├── ScalarSnapshotValue.php
│   │   ├── CanonicalSnapshotValue.php
│   │   ├── StructuredSnapshotValue.php
│   │   ├── EntityReferenceSnapshotValue.php
│   │   ├── HashedSnapshotValue.php
│   │   ├── UnknownSnapshotValue.php
│   │   └── SnapshotValueContext.php
│   │
│   ├── Plan/
│   │   ├── CompiledSnapshotPlan.php
│   │   ├── SnapshotPlanCompiler.php
│   │   ├── SnapshotPlanRegistry.php
│   │   ├── SnapshotPlanFingerprint.php
│   │   ├── SnapshotPropertyPlan.php
│   │   ├── SnapshotPropertyPlanCollection.php
│   │   ├── SnapshotLayout.php
│   │   ├── SnapshotLayoutId.php
│   │   └── SnapshotLayoutVersion.php
│   │
│   ├── Copy/
│   │   ├── SnapshotCopyStrategyRegistry.php
│   │   ├── DirectSnapshotStrategy.php
│   │   ├── ShallowCopySnapshotStrategy.php
│   │   ├── DeepCopySnapshotStrategy.php
│   │   ├── CanonicalSnapshotStrategy.php
│   │   ├── StructuralSnapshotStrategy.php
│   │   ├── HashSnapshotStrategy.php
│   │   └── CustomSnapshotStrategyAdapter.php
│   │
│   ├── Registry/
│   │   ├── DefaultEntitySnapshotRegistry.php
│   │   ├── SnapshotRegistryEntry.php
│   │   └── EntitySnapshotState.php
│   │
│   ├── Identity/
│   │   └── EntityIdentitySnapshot.php
│   │
│   ├── Embedded/
│   │   ├── EmbeddedSnapshotPlan.php
│   │   ├── EmbeddedSnapshotValue.php
│   │   └── EmbeddedSnapshotFactory.php
│   │
│   ├── Relationship/
│   │   ├── RelationshipSnapshotPlan.php
│   │   ├── ToOneRelationshipSnapshot.php
│   │   ├── CollectionSnapshot.php
│   │   ├── CollectionSnapshotState.php
│   │   └── CollectionSnapshotCompleteness.php
│   │
│   ├── Reconciliation/
│   │   ├── DefaultSnapshotReconciler.php
│   │   ├── SnapshotReconciliationState.php
│   │   ├── SnapshotReconciliationResult.php
│   │   └── SnapshotReconciliationContext.php
│   │
│   ├── Transaction/
│   │   ├── SnapshotTransactionFrame.php
│   │   ├── SnapshotTransactionStack.php
│   │   └── SnapshotTransactionCoordinator.php
│   │
│   ├── Runtime/
│   │   ├── SnapshotRuntimeScope.php
│   │   ├── SnapshotRuntimeResetter.php
│   │   └── SnapshotOwnershipGuard.php
│   │
│   ├── Resource/
│   │   ├── SnapshotResourceLimits.php
│   │   ├── SnapshotMemoryEstimator.php
│   │   └── SnapshotResourceGuard.php
│   │
│   ├── Telemetry/
│   │   ├── SnapshotTelemetry.php
│   │   ├── SnapshotProfiler.php
│   │   └── SnapshotDiagnostics.php
│   │
│   └── Exception/
│       └── ...
```

---

# 262. Integration with Change Tracking

```text
Snapshot Registry
       │
       ▼
Entity Snapshot
       │
       ├────────────┐
       │            │
       ▼            ▼
Original State   Current Entity
       │            │
       └─────┬──────┘
             ▼
       Change Tracker
             │
             ▼
          ChangeSet
```

---

# 263. Integration with UnitOfWork

```text
UnitOfWork
├── Entity Entries
├── Entity States
├── Snapshot Registry
├── ChangeSet Registry
└── Persistence Preparation
```

---

# 264. Integration with IdentityMap

```text
EntityKey
   │
   ├──→ IdentityMap → object
   │
   └──→ SnapshotRegistry → baseline
```

---

# 265. Integration with Entity State

Typical invariant:

```text
MANAGED synchronized entity
→ valid snapshot expected
```

except explicitly supported:

```text
read-only
partial special mode
new entity
tainted state
```

---

# 266. Integration with Persistence Engine

Persistence Engine no deberá mutar snapshot directamente.

Debe producir:

```text
Execution Outcome
+
Persistence Reconciliation Information
```

---

# 267. Reconciler

```text
Persistence Outcome
        +
Old Snapshot
        +
Stable ChangeSet
        +
Generated Values
        ↓
Snapshot Reconciler
        ↓
New Snapshot / Preserve / Invalidate / Unknown
```

---

# 268. Integration with Transaction Manager

Transaction Manager deberá informar:

```text
begin
savepoint
commit
rollback
unknown outcome
```

para gestionar snapshot frames.

---

# 269. Integration with Type System

```text
Mapped Type
    ↓
Snapshot Strategy
    ↓
Canonical Snapshot Value
```

---

# 270. Integration with Hydration

Hydrator proporciona:

```text
loaded state
persistent values
generated values
relationship initialization state
```

para snapshot creation.

---

# 271. Integration with Model API

`Model::save()` no manipulará snapshots directamente.

Ruta:

```text
Model API
→ EntityManager
→ UnitOfWork
→ Persistence
→ Snapshot Reconciliation
```

---

# 272. Integration with raw Query Builder

Query Builder no conoce snapshots.

Si una operación externa invalida entidades managed:

```text
EntityManager integration layer
```

deberá aplicar policy.

---

# 273. Testing architecture

Se requieren:

```text
unit tests
integration tests
runtime isolation tests
transaction tests
mutable value tests
partial entity tests
memory tests
determinism tests
platform-independent ORM tests
```

---

# 274. Test — immutable scalar

```text
name = Alice
```

debe producir baseline independiente correcto.

---

# 275. Test — mutable object

Mutar current value no deberá alterar snapshot.

---

# 276. Test — array mutation

```php
$user->settings['theme'] = 'dark';
```

deberá dejar snapshot original intacto.

---

# 277. Test — partial entity

Unloaded properties permanecen `UNLOADED`.

---

# 278. Test — null

Real `NULL` permanece distinto de unloaded.

---

# 279. Test — uninitialized typed property

Debe conservar estado correcto.

---

# 280. Test — JSON canonicalization

Object key order deberá respetar policy.

---

# 281. Test — JSON array

Order debe preservarse.

---

# 282. Test — decimal

No debe perder precisión.

---

# 283. Test — DateTime

Debe conservar comparación según mapping.

---

# 284. Test — entity relationship

Debe snapshotear identidad, no object graph.

---

# 285. Test — unloaded relationship

No query oculta.

---

# 286. Test — unloaded collection

No query oculta.

---

# 287. Test — partial collection

No presentarla como completa.

---

# 288. Test — generated identifier

Crear snapshot persistente solo tras outcome válido.

---

# 289. Test — generated field

Nuevo baseline incorpora valor confirmado.

---

# 290. Test — unknown insert

No afirmar baseline persistido.

---

# 291. Test — successful update

Snapshot generation avanza.

---

# 292. Test — failed update

Snapshot se preserva.

---

# 293. Test — partial persistence

Reconciliation conserva incertidumbre.

---

# 294. Test — optimistic lock conflict

No avanzar snapshot como si update hubiera funcionado.

---

# 295. Test — transaction flush + commit

Promover baseline correctamente.

---

# 296. Test — transaction flush + rollback

Restaurar baseline transaccional correcto.

---

# 297. Test — entity after rollback

Entidad modificada puede volver a resultar dirty.

---

# 298. Test — savepoint rollback

Restaurar frame correspondiente.

---

# 299. Test — commit unknown

No promover baseline como cierto.

---

# 300. Test — refresh

Reemplazar snapshot correctamente.

---

# 301. Test — bulk invalidation

Managed snapshots afectados deberán seguir policy.

---

# 302. Test — raw mutation

Explicit invalidation deberá funcionar.

---

# 303. Test — detach

Snapshot eliminado.

---

# 304. Test — clear

Registry vacío.

---

# 305. Test — request reset

Request B no ve snapshots de Request A.

---

# 306. Test — tenant isolation

Misma identidad numérica en tenants distintos no colisiona.

---

# 307. Test — large UnitOfWork

Medir memoria y cleanup.

---

# 308. Test — compact layout

Representación compacta conserva semántica.

---

# 309. Test — metadata layout mismatch

Debe rechazarse.

---

# 310. Test — fingerprint determinism

Mismos inputs producen mismo fingerprint semántico.

---

# 311. Test — sensitive diagnostics

No filtrar valores.

---

# 312. Anti-pattern: clone entity as universal snapshot

```php
$snapshot = clone $entity;
```

No será arquitectura base.

---

# 313. Anti-pattern: serialize entity

```php
$snapshot = serialize($entity);
```

Prohibido como contrato universal.

---

# 314. Anti-pattern: raw database row snapshot

El row físico no reemplaza necesariamente la semántica ORM.

---

# 315. Anti-pattern: snapshot points to mutable object

```text
Entity ──────┐
             ├→ MutableValue
Snapshot ────┘
```

Prohibido cuando puede ocultar cambios.

---

# 316. Anti-pattern: unloaded as null

Riesgo directo de pérdida de datos.

---

# 317. Anti-pattern: snapshot refresh by hidden query

Snapshot System no deberá consultar DB silenciosamente.

---

# 318. Anti-pattern: lazy load during snapshot creation

Prohibido.

---

# 319. Anti-pattern: advance snapshot before persistence certainty

Prohibido.

---

# 320. Anti-pattern: assume failed execution means no effect

Si outcome es UNKNOWN, preservar incertidumbre.

---

# 321. Anti-pattern: flush means commit

Incorrecto.

---

# 322. Anti-pattern: transaction rollback restores entity automatically

No necesariamente.

---

# 323. Anti-pattern: snapshot generation = optimistic version

Dominios distintos.

---

# 324. Anti-pattern: process-global registry

Incompatible con persistent runtimes.

---

# 325. Anti-pattern: snapshot history as audit log

Snapshot System no es temporal storage.

---

# 326. Anti-pattern: entity snapshots in queue payloads

Recargar por identidad en nuevo contexto.

---

# 327. Anti-pattern: silent lossy compression

Memory optimization no puede alterar semántica sin contrato.

---

# 328. Anti-pattern: full collection loading

No cargar relaciones solo para snapshot.

---

# 329. Anti-pattern: object identity for relationships

`spl_object_id()` no representa identidad persistente.

---

# 330. Anti-pattern: table name as snapshot identity

Snapshot se asocia a:

```text
EntityKey
```

no a:

```text
table + row
```

como concepto ORM primario.

---

# 331. Architectural invariants

## DB-ORM-SNAPSHOT-001
EntitySnapshot representará un baseline persistente conocido.

## DB-ORM-SNAPSHOT-002
EntitySnapshot será distinto de Entity.

## DB-ORM-SNAPSHOT-003
EntitySnapshot será distinto de raw database row.

## DB-ORM-SNAPSHOT-004
EntitySnapshot será distinto del estado actual garantizado de la DB.

## DB-ORM-SNAPSHOT-005
EntitySnapshot será distinto de cache.

## DB-ORM-SNAPSHOT-006
EntitySnapshot será distinto de IdentityMap.

## DB-ORM-SNAPSHOT-007
EntitySnapshot será distinto de ChangeSet.

## DB-ORM-SNAPSHOT-008
EntitySnapshot será distinto de audit history.

## DB-ORM-SNAPSHOT-009
EntitySnapshot será distinto de serialization.

## DB-ORM-SNAPSHOT-010
Snapshots estables serán immutable.

## DB-ORM-SNAPSHOT-011
Snapshot replacement producirá una nueva generación lógica.

## DB-ORM-SNAPSHOT-012
SnapshotGeneration será distinta de entity version.

## DB-ORM-SNAPSHOT-013
SnapshotGeneration será distinta de business version.

## DB-ORM-SNAPSHOT-014
SnapshotId será distinto de EntityKey.

## DB-ORM-SNAPSHOT-015
SnapshotProperty será metadata-driven.

## DB-ORM-SNAPSHOT-016
Solo persistent state relevante participará.

## DB-ORM-SNAPSHOT-017
NULL será distinto de UNLOADED.

## DB-ORM-SNAPSHOT-018
NULL será distinto de UNKNOWN.

## DB-ORM-SNAPSHOT-019
UNINITIALIZED será distinto de UNLOADED.

## DB-ORM-SNAPSHOT-020
UNKNOWN nunca se interpretará como NULL.

## DB-ORM-SNAPSHOT-021
Snapshot completeness será explícita.

## DB-ORM-SNAPSHOT-022
COMPLETE será distinto de PARTIAL.

## DB-ORM-SNAPSHOT-023
PARTIAL será distinto de UNKNOWN.

## DB-ORM-SNAPSHOT-024
Completeness será distinta de certainty.

## DB-ORM-SNAPSHOT-025
Partial snapshots preservarán unloaded state.

## DB-ORM-SNAPSHOT-026
Snapshot creation no mutará la entidad.

## DB-ORM-SNAPSHOT-027
Snapshot creation no ejecutará SQL.

## DB-ORM-SNAPSHOT-028
Snapshot creation no disparará lazy loading.

## DB-ORM-SNAPSHOT-029
Snapshot creation no cambiará PersistenceContext.

## DB-ORM-SNAPSHOT-030
Snapshot creation será metadata-driven.

## DB-ORM-SNAPSHOT-031
Snapshot plans serán compilables.

## DB-ORM-SNAPSHOT-032
Compiled plans serán immutable.

## DB-ORM-SNAPSHOT-033
Compiled plans podrán compartirse entre requests.

## DB-ORM-SNAPSHOT-034
Snapshot registries no se compartirán entre requests.

## DB-ORM-SNAPSHOT-035
Runtime reflection repetitiva deberá minimizarse.

## DB-ORM-SNAPSHOT-036
Snapshot y Change Tracking utilizarán accessor semantics compatibles.

## DB-ORM-SNAPSHOT-037
Snapshot representation será type-aware.

## DB-ORM-SNAPSHOT-038
Snapshot representation no será universalmente raw PHP value.

## DB-ORM-SNAPSHOT-039
Snapshot representation no será universalmente DB binding value.

## DB-ORM-SNAPSHOT-040
Snapshot copy strategy será explícita.

## DB-ORM-SNAPSHOT-041
DIRECT solo se utilizará cuando sharing sea semánticamente seguro.

## DB-ORM-SNAPSHOT-042
Mutable aliasing que oculte cambios estará prohibido.

## DB-ORM-SNAPSHOT-043
Deep copy no utilizará serialize/unserialize como contrato universal.

## DB-ORM-SNAPSHOT-044
Canonical snapshots deberán preservar comparison semantics.

## DB-ORM-SNAPSHOT-045
Structural snapshots deberán preservar persistent structure semantics.

## DB-ORM-SNAPSHOT-046
Hash snapshots requerirán contrato explícito.

## DB-ORM-SNAPSHOT-047
Hash equality no sustituirá exact equality sin garantía.

## DB-ORM-SNAPSHOT-048
Custom snapshot strategies serán deterministas.

## DB-ORM-SNAPSHOT-049
Custom strategies serán side-effect-free.

## DB-ORM-SNAPSHOT-050
Custom strategies no ejecutarán I/O oculto.

## DB-ORM-SNAPSHOT-051
PHP readonly no implicará deep immutability automáticamente.

## DB-ORM-SNAPSHOT-052
Unknown mutability no se tratará como immutable.

## DB-ORM-SNAPSHOT-053
Arrays mutables conservarán baseline independiente.

## DB-ORM-SNAPSHOT-054
JSON snapshots preservarán object/array semantics.

## DB-ORM-SNAPSHOT-055
Decimals no serán convertidos automáticamente a float.

## DB-ORM-SNAPSHOT-056
Date/time snapshots respetarán type semantics.

## DB-ORM-SNAPSHOT-057
Identifier baseline tendrá tratamiento especial.

## DB-ORM-SNAPSHOT-058
Established identifier mutation no será normal field change.

## DB-ORM-SNAPSHOT-059
EntityIdentitySnapshot podrá preservar identidad original.

## DB-ORM-SNAPSHOT-060
NEW entity no requerirá persisted snapshot.

## DB-ORM-SNAPSHOT-061
Transient snapshot será distinto de persisted snapshot.

## DB-ORM-SNAPSHOT-062
Generated identifier no establecerá persisted baseline antes de outcome válido.

## DB-ORM-SNAPSHOT-063
UNKNOWN insert outcome no producirá snapshot sincronizado ficticio.

## DB-ORM-SNAPSHOT-064
Database-generated values se incorporarán solo con evidencia suficiente.

## DB-ORM-SNAPSHOT-065
Database-authoritative values no serán reemplazados por valores locales no confirmados.

## DB-ORM-SNAPSHOT-066
Snapshot advancement requerirá persistence reconciliation.

## DB-ORM-SNAPSHOT-067
Successful confirmed persistence podrá avanzar snapshot.

## DB-ORM-SNAPSHOT-068
Known no-effect failure preservará snapshot.

## DB-ORM-SNAPSHOT-069
Partial persistence será first-class.

## DB-ORM-SNAPSHOT-070
UNKNOWN persistence outcome será first-class.

## DB-ORM-SNAPSHOT-071
UNKNOWN outcome no asumirá éxito.

## DB-ORM-SNAPSHOT-072
UNKNOWN outcome no asumirá fracaso.

## DB-ORM-SNAPSHOT-073
Invalid baseline no se utilizará como válido.

## DB-ORM-SNAPSHOT-074
Snapshot lifecycle state podrá residir en registry entry.

## DB-ORM-SNAPSHOT-075
flush será distinto de commit.

## DB-ORM-SNAPSHOT-076
Snapshot system será transaction-aware.

## DB-ORM-SNAPSHOT-077
Flush dentro de transaction no implicará committed baseline externo.

## DB-ORM-SNAPSHOT-078
Transaction rollback restaurará baseline apropiado.

## DB-ORM-SNAPSHOT-079
Transaction rollback no revertirá automáticamente el objeto Entity.

## DB-ORM-SNAPSHOT-080
Después de rollback, current entity podrá diferir nuevamente del snapshot.

## DB-ORM-SNAPSHOT-081
Nested transactions podrán requerir snapshot frames.

## DB-ORM-SNAPSHOT-082
Savepoint rollback deberá preservar baseline correcto.

## DB-ORM-SNAPSHOT-083
Unknown commit outcome no promoverá baseline como cierto.

## DB-ORM-SNAPSHOT-084
Optimistic version token será parte del baseline cuando aplique.

## DB-ORM-SNAPSHOT-085
Optimistic version será distinta de SnapshotGeneration.

## DB-ORM-SNAPSHOT-086
Optimistic lock failure no avanzará snapshot como éxito.

## DB-ORM-SNAPSHOT-087
To-one relationships preferirán identity snapshots.

## DB-ORM-SNAPSHOT-088
Relationship snapshots no dependerán de PHP object identity.

## DB-ORM-SNAPSHOT-089
Unloaded relationship permanecerá unloaded.

## DB-ORM-SNAPSHOT-090
To-many snapshots tendrán semántica especializada.

## DB-ORM-SNAPSHOT-091
Unloaded collections no serán cargadas para snapshot.

## DB-ORM-SNAPSHOT-092
Partial collection no será presentada como complete collection.

## DB-ORM-SNAPSHOT-093
Collection ordering será preservado cuando sea persistente.

## DB-ORM-SNAPSHOT-094
Collection snapshot será distinto de collection object.

## DB-ORM-SNAPSHOT-095
Snapshot registry será PersistenceContext-scoped.

## DB-ORM-SNAPSHOT-096
Same EntityKey podrá tener snapshots distintos en contexts distintos.

## DB-ORM-SNAPSHOT-097
Tenant identity namespace será respetado.

## DB-ORM-SNAPSHOT-098
Snapshot core no hardcodeará tenant_id.

## DB-ORM-SNAPSHOT-099
Shard identity namespace será respetado.

## DB-ORM-SNAPSHOT-100
Refresh será coordinado por EntityManager/Loader, no por hidden Snapshot I/O.

## DB-ORM-SNAPSHOT-101
Refresh con local changes tendrá policy explícita.

## DB-ORM-SNAPSHOT-102
V1 no realizará implicit merge.

## DB-ORM-SNAPSHOT-103
Bulk operations podrán invalidar snapshots.

## DB-ORM-SNAPSHOT-104
Raw SQL podrá requerir explicit invalidation.

## DB-ORM-SNAPSHOT-105
Snapshot invalidation será distinta de entity deletion.

## DB-ORM-SNAPSHOT-106
Read-only entities podrán omitir snapshots.

## DB-ORM-SNAPSHOT-107
Projection results no necesitarán snapshots ORM.

## DB-ORM-SNAPSHOT-108
Detach eliminará snapshot scoped.

## DB-ORM-SNAPSHOT-109
EntityManager clear eliminará snapshots.

## DB-ORM-SNAPSHOT-110
EntityManager close eliminará snapshots.

## DB-ORM-SNAPSHOT-111
Request cleanup eliminará snapshots mutable-scoped.

## DB-ORM-SNAPSHOT-112
Snapshot cleanup no ejecutará implicit flush.

## DB-ORM-SNAPSHOT-113
FrankenPHP requests no compartirán snapshots.

## DB-ORM-SNAPSHOT-114
RoadRunner requests no compartirán snapshots.

## DB-ORM-SNAPSHOT-115
OpenSwoole execution scopes no compartirán snapshots accidentalmente.

## DB-ORM-SNAPSHOT-116
Process-global snapshot registry estará prohibido.

## DB-ORM-SNAPSHOT-117
Snapshot memory será observable.

## DB-ORM-SNAPSHOT-118
Large UnitOfWork deberá soportar flush/clear operational pattern.

## DB-ORM-SNAPSHOT-119
Compact snapshots podrán utilizar compiled layouts.

## DB-ORM-SNAPSHOT-120
Snapshot slot indexes serán internal implementation details.

## DB-ORM-SNAPSHOT-121
Snapshot layout será versionable/fingerprintable.

## DB-ORM-SNAPSHOT-122
Layout mismatch será detectado.

## DB-ORM-SNAPSHOT-123
Metadata drift no será ignorado.

## DB-ORM-SNAPSHOT-124
Snapshot fingerprint será determinista.

## DB-ORM-SNAPSHOT-125
Fingerprint no dependerá de memory address.

## DB-ORM-SNAPSHOT-126
Sensitive values no se expondrán en fingerprints/logs.

## DB-ORM-SNAPSHOT-127
Snapshot fingerprint será distinto de optimistic lock version.

## DB-ORM-SNAPSHOT-128
Snapshot fingerprint no será audit proof por defecto.

## DB-ORM-SNAPSHOT-129
Property ordering será determinista.

## DB-ORM-SNAPSHOT-130
Reflection order no será contrato.

## DB-ORM-SNAPSHOT-131
Snapshots no serán persistence payloads cross-process por defecto.

## DB-ORM-SNAPSHOT-132
Queue jobs deberán preferir EntityReference sobre EntitySnapshot.

## DB-ORM-SNAPSHOT-133
Extension registries serán frozen durante runtime normal.

## DB-ORM-SNAPSHOT-134
Extension collisions no usarán last-wins.

## DB-ORM-SNAPSHOT-135
Extensions no podrán ocultar unknown state.

## DB-ORM-SNAPSHOT-136
Extensions no convertirán unloaded a null.

## DB-ORM-SNAPSHOT-137
Extensions no ejecutarán lazy loading oculto.

## DB-ORM-SNAPSHOT-138
Extensions no cambiarán tenant context.

## DB-ORM-SNAPSHOT-139
Custom snapshot values serán immutable.

## DB-ORM-SNAPSHOT-140
Telemetry no cambiará snapshot semantics.

## DB-ORM-SNAPSHOT-141
High-cardinality entity IDs no serán metric labels por defecto.

## DB-ORM-SNAPSHOT-142
Diagnostics soportarán redaction.

## DB-ORM-SNAPSHOT-143
Resource limits no producirán silent truncation.

## DB-ORM-SNAPSHOT-144
Memory pressure no degradará snapshot accuracy silenciosamente.

## DB-ORM-SNAPSHOT-145
Snapshots sensibles tendrán lifetime mínimo razonable.

## DB-ORM-SNAPSHOT-146
Internal snapshot access estará controlado.

## DB-ORM-SNAPSHOT-147
Snapshot publication será validada antes de registrarse.

## DB-ORM-SNAPSHOT-148
Hydration failure no publicará snapshot válido parcial accidentalmente.

## DB-ORM-SNAPSHOT-149
Snapshot/current comparison utilizará semántica compatible.

## DB-ORM-SNAPSHOT-150
Baseline advancement será explícito.

## DB-ORM-SNAPSHOT-151
Baseline replacement será distinto de entity mutation.

## DB-ORM-SNAPSHOT-152
Snapshot observation timestamp no garantizará DB freshness.

## DB-ORM-SNAPSHOT-153
Snapshot certainty preservará uncertainty.

## DB-ORM-SNAPSHOT-154
Snapshot completeness preservará partial loading.

## DB-ORM-SNAPSHOT-155
Snapshot system no será un second-level cache.

## DB-ORM-SNAPSHOT-156
Snapshot system no será un audit/versioning system.

## DB-ORM-SNAPSHOT-157
Snapshot system no generará SQL.

## DB-ORM-SNAPSHOT-158
Snapshot system no ejecutará queries.

## DB-ORM-SNAPSHOT-159
Snapshot system no decidirá persistence ordering.

## DB-ORM-SNAPSHOT-160
Entity Snapshot será un baseline immutable, tipado, context-scoped, uncertainty-aware y transaction-aware del último estado persistente conocido.

---

# 332. Fórmula fundamental

```text
EntitySnapshot(E, C, t)
=
KnownPersistentBaseline(
    Entity = E,
    PersistenceContext = C,
    Observation = t
)
```

---

# 333. Snapshot independence

Para cualquier valor mutable `V`:

```text
Mutate(Current(V))
⇒
Snapshot(V) remains semantically unchanged
```

---

# 334. Change Tracking relation

```text
ChangeSet(E)
=
Compare(
    Snapshot(E),
    CurrentPersistentState(E)
)
```

---

# 335. Partial snapshot formula

```text
Snapshot(P) = UNLOADED
⇒
¬ Infer(Snapshot(P) = NULL)
```

---

# 336. Snapshot advancement

```text
AdvanceSnapshot
⇔
PersistenceEffectKnown
∧
ResultReconciled
∧
GeneratedValuesReconciled
∧
TransactionSemanticsSatisfied
```

---

# 337. Failure preservation

```text
KnownNoEffectFailure
⇒
PreservePreviousSnapshot
```

---

# 338. Unknown outcome rule

```text
UnknownPersistenceOutcome
⇒
¬ AssumeSuccess
∧
¬ AssumeFailure
∧
PreserveUncertainty
```

---

# 339. Transaction rollback formula

```text
EntityStateAfterRollback
≠
DatabaseStateAfterRollback
```

necessarily.

Therefore:

```text
Rollback(DB)
→ RestoreAppropriateBaseline
→ RecompareCurrentEntity
```

---

# 340. Safe mutable snapshot

```text
SafeMutableSnapshot(V)
=
IndependentRepresentation(V)
∧
DeterministicCopyOrCanonicalization(V)
∧
NoObservableMutableAliasing(V)
```

---

# 341. Snapshot scope formula

```text
SnapshotIdentity
=
PersistenceContext
×
EntityKey
×
SnapshotGeneration
```

---

# 342. Memory formula

```text
SnapshotMemory
≈
Σ(
    ManagedEntitySnapshotSize
)
+
RegistryOverhead
+
TransactionFrames
+
TemporarySnapshotConstruction
```

---

# 343. Safe persistent runtime

```text
SafeSnapshotRuntime
=
ImmutableSharedPlans
∧
ImmutableSharedStrategies
∧
ScopedSnapshotRegistry
∧
ScopedTransactionFrames
∧
ScopedReconciliationState
∧
DeterministicCleanup
∧
NoCrossRequestSnapshotLeakage
```

---

# 344. Snapshot consistency formula

```text
UsableSnapshot
=
Valid
∧
MappingCompatible
∧
ContextCompatible
∧
IdentityCompatible
∧
SufficientlyCertainForRequestedOperation
```

---

# 345. Master Formula

```text
Database Entity Snapshot System
=
Persistent Baseline Model
+
Typed Snapshot Values
+
Explicit Loaded States
+
Completeness
+
Certainty
+
Compiled Snapshot Plans
+
Property Accessors
+
Copy Strategies
+
Canonicalization
+
Mutable Value Isolation
+
Embedded Snapshots
+
Relationship Identity Snapshots
+
Collection Snapshot Boundaries
+
Identifier Baselines
+
Generated Value Reconciliation
+
Optimistic Version Baselines
+
Immutable Snapshot Generations
+
Snapshot Registry
+
Persistence Reconciliation
+
Transaction Frames
+
Rollback Awareness
+
Invalidation
+
Partial Entity Awareness
+
Runtime Isolation
+
Memory Governance
+
Deterministic Fingerprinting
+
Security Redaction
+
Telemetry
+
Diagnostics
+
Extension Governance
```

---

# 346. Master Rule

> **VoltStack utilizará Entity Snapshots como baselines ORM inmutables y explícitos del estado persistente conocido de cada entidad managed. Un snapshot nunca será tratado como una copia viva de la entidad, una garantía del estado actual de la base de datos, un cache global ni un historial de auditoría; cualquier avance del baseline deberá preservar loaded-state, tipos, incertidumbre, contexto transaccional y certeza real del resultado de persistencia.**

---

# 347. Resultado arquitectónico

Con los documentos 123–126 queda establecido el núcleo de seguimiento de estado:

```text
                    Entity
                      │
             ┌────────┴────────┐
             │                 │
             ▼                 ▼
        IdentityMap       Entity State
             │                 │
             └────────┬────────┘
                      ▼
                  UnitOfWork
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
    Snapshot Registry       Current Entity
          │                       │
          └───────────┬───────────┘
                      ▼
                Change Tracking
                      │
                      ▼
                  ChangeSet
                      │
                      ▼
              Prepared UnitOfWork
```

Ahora VoltStack dispone de suficiente información para comenzar a convertir:

```text
managed entity changes
```

en:

```text
structured persistence operations
```

sin romper la separación:

```text
Entity
≠
ChangeSet
≠
Persistence Operation
≠
Query Model
≠
SQL
```

---

# 348. Transición al Persistence Engine

La siguiente etapa será:

```text
Entity State
     +
Snapshots
     +
ChangeSets
     +
Relationship Changes
     +
UnitOfWork
     ↓
Persistence Engine
```

El Persistence Engine será responsable de convertir el estado ORM preparado en operaciones persistibles estructuradas.

No deberá generar SQL directamente.

---

# 349. Siguiente documento

```text
127_DATABASE_PERSISTENCE_ENGINE.md
```

Este documento deberá formalizar:

```text
Persistence Engine architecture
persistence responsibilities
Persistence Engine ≠ UnitOfWork
Persistence Engine ≠ Persistence Planner
Persistence Engine ≠ Query Engine
Persistence Engine ≠ SQL Compiler
Persistence Engine ≠ Executor

Prepared UnitOfWork consumption
Entity Change Graph
Persistence Operation Model
Persistence Operation Graph
insert/update/delete operations
relationship persistence operations
collection persistence operations
generated-value requirements
dependency discovery
operation normalization
operation validation
persistence intent
persistence scope
entity-to-operation transformation
change-set consumption
snapshot integration
identity integration
cascade integration
orphan integration
version/locking integration
query model generation boundary
execution boundary
result reconciliation
outcome certainty
partial persistence
transaction integration
batch persistence boundaries
platform capability integration
runtime isolation
extension model
telemetry
diagnostics
security
testing
failure model
```

con la regla central:

> **El Persistence Engine transforma un UnitOfWork preparado en operaciones ORM de persistencia estructuradas y reconciliables; no genera SQL, no ejecuta queries y no sustituye al Persistence Planner, Query Engine, Transaction Manager ni Execution Engine.**