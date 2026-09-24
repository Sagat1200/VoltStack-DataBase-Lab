# 125_DATABASE_CHANGE_TRACKING_SYSTEM.md

# VoltStack Quantum Database
## Database Change Tracking System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 125 — Database Change Tracking System  
**Bloque:** 11 — Identity Map, Unit of Work & Persistence  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Change Tracking System` define la arquitectura mediante la cual el ORM de VoltStack determina qué parte del estado persistente de una entidad ha cambiado desde el último estado sincronizado conocido.

Su responsabilidad fundamental es transformar:

```text
Original Persistent State
        +
Current Entity State
        ↓
Change Detection
        ↓
Typed ChangeSet
```

sin decidir todavía:

```text
SQL
physical persistence strategy
transaction boundaries
query execution
```

La regla central será:

> **Change Tracking determina qué estado persistente de una entidad cambió desde su baseline conocido; no decide cómo ese cambio será convertido en SQL ni si debe ejecutarse dentro de una transacción.**

---

# 2. Contexto arquitectónico

El documento anterior estableció:

```text
124_DATABASE_UNIT_OF_WORK_ARCHITECTURE
```

donde:

```text
UnitOfWork
=
coordination of pending persistence work
```

Change Tracking es uno de sus subsistemas especializados:

```text
UnitOfWork
    │
    ├── Entity Registration
    ├── Change Tracking
    ├── Snapshot System
    ├── Relationship Tracking
    ├── Cascade Discovery
    ├── Orphan Analysis
    └── Change Graph
```

---

# 3. Problema

Supongamos:

```php
$user = $repository->find(10);

$user->rename('Alice');
$user->changeEmail('alice@example.com');
```

Cuando la entidad fue hidratada:

```text
name  = Bob
email = bob@example.com
```

Actualmente:

```text
name  = Alice
email = alice@example.com
```

VoltStack necesita producir:

```text
User#10 ChangeSet

name:
    Bob
    →
    Alice

email:
    bob@example.com
    →
    alice@example.com
```

No deberá producir todavía:

```sql
UPDATE users
SET name = ?, email = ?
WHERE id = ?
```

---

# 4. Change Tracking ≠ SQL generation

Regla:

```text
ChangeSet
≠
SQL
```

Change Tracking trabaja con conceptos ORM:

```text
Entity
Field
Persistent Property
Relationship
Embedded Value
Original Value
Current Value
```

SQL pertenece al:

```text
Persistence Engine
→ Query Engine
→ SQL Compiler
```

---

# 5. Change Tracking ≠ UnitOfWork

UnitOfWork responde:

> ¿Qué trabajo de persistencia está pendiente?

Change Tracking responde:

> ¿Qué cambió dentro de esta entidad?

Por tanto:

```text
UnitOfWork
        │
        ▼
ChangeTracker
        │
        ▼
ChangeSet
```

---

# 6. Change Tracking ≠ Snapshot System

El Snapshot System conserva:

```text
baseline state
```

Change Tracking compara:

```text
baseline
vs
current state
```

El documento siguiente:

```text
126_DATABASE_ENTITY_SNAPSHOT_SYSTEM.md
```

formalizará snapshots.

---

# 7. Change Tracking ≠ Entity State

Una entidad puede estar:

```text
MANAGED
```

y no haber cambiado.

También puede estar:

```text
MANAGED
+
dirty
```

Por ello:

```text
EntityState
≠
ChangeTrackingState
```

---

# 8. Dirty ≠ lifecycle state

`DIRTY` deberá considerarse principalmente una condición de tracking:

```text
Dirty(E)
=
ChangeSet(E) ≠ ∅
```

y no necesariamente un estado fundamental equivalente a:

```text
NEW
MANAGED
REMOVED
DETACHED
```

---

# 9. Change Tracking ≠ Persistence Decision

Que exista un ChangeSet no significa automáticamente:

```text
execute UPDATE
```

Persistence Planner podría determinar:

```text
UPDATE
NO_OP
combined operation
bulk strategy
relationship operation
special persistence strategy
```

---

# 10. Arquitectura general

```text
                    Entity Metadata
                          │
                          ▼
                 Change Tracking Plan
                          │
            ┌─────────────┴─────────────┐
            │                           │
            ▼                           ▼
     Original Snapshot             Current Entity
            │                           │
            └─────────────┬─────────────┘
                          ▼
                   Value Extraction
                          │
                          ▼
                   Normalization
                          │
                          ▼
                  Typed Comparison
                          │
                          ▼
                   Change Detection
                          │
                          ▼
                      ChangeSet
                          │
          ┌───────────────┼────────────────┐
          ▼               ▼                ▼
     FieldChanges   EmbeddedChanges   RelationChanges
                          │
                          ▼
                     UnitOfWork
                          │
                          ▼
                  EntityChangeGraph
```

---

# 11. Core terminology

El sistema distinguirá:

```text
Persistent Property
Original Value
Current Value
Canonical Value
Comparison Value
Change
ChangeSet
Dirty Entity
Baseline
Snapshot
Change Tracking Strategy
Change Tracking Plan
```

---

# 12. Persistent property

No toda propiedad PHP es persistente.

Ejemplo:

```php
final class User
{
    private UserId $id;

    private string $name;

    private string $email;

    private bool $profileExpanded = false;
}
```

`profileExpanded` puede ser:

```text
runtime-only
```

y no deberá entrar al Change Tracking ORM.

---

# 13. Metadata-driven tracking

Las propiedades rastreadas deberán derivarse de:

```text
Compiled Entity Metadata
```

No de:

```text
all PHP properties
```

---

# 14. PersistentPropertyId

Se recomienda un identificador lógico:

```php
final readonly class PersistentPropertyId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 15. Property ID ≠ PHP property name

Esto permitirá desacoplar:

```text
logical persistent field
```

de:

```text
physical PHP representation
```

cuando sea necesario.

---

# 16. Property ID ≠ column name

También:

```text
PersistentPropertyId
≠
DatabaseColumnName
```

Ejemplo:

```text
Entity property:
createdAt

Database column:
created_at
```

---

# 17. Baseline

El baseline representa:

> el último estado persistente conocido contra el cual puede compararse el estado actual.

Normalmente proviene del:

```text
Entity Snapshot System
```

---

# 18. Baseline ≠ database current state

Un snapshot representa:

```text
what this PersistenceContext knows
```

No necesariamente:

```text
what the database contains right now
```

porque otro proceso pudo modificar la fila.

---

# 19. Baseline certainty

Podrá existir:

```php
enum BaselineCertainty
{
    case KNOWN;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 20. KNOWN

El ORM conoce el baseline necesario para comparar una propiedad.

---

# 21. PARTIAL

Solo parte de la entidad fue cargada.

Ejemplo:

```text
User#10

loaded:
    id
    name

not loaded:
    email
    address
```

---

# 22. UNKNOWN

VoltStack no puede afirmar el valor original.

Regla:

```text
UNKNOWN
≠
null
```

---

# 23. Unloaded ≠ null

Especialmente para partial entities:

```text
property not loaded
```

no deberá convertirse en:

```text
original = null
```

---

# 24. OriginalValue

Se propone una representación explícita:

```php
interface OriginalValueState
{
}
```

con:

```text
KnownOriginalValue
UnloadedOriginalValue
UnknownOriginalValue
```

---

# 25. Ejemplo

```php
final readonly class KnownOriginalValue
{
    public function __construct(
        public mixed $value,
    ) {}
}
```

---

# 26. Sentinel explícito

Nunca deberá utilizarse un valor ambiguo como:

```php
null
```

para representar simultáneamente:

```text
real database NULL
unloaded
unknown
uninitialized
```

---

# 27. Value states

VoltStack deberá poder distinguir:

```text
VALUE
NULL
UNLOADED
UNINITIALIZED
UNKNOWN
```

cuando la semántica lo requiera.

---

# 28. ChangeSet

Objeto central:

```php
final readonly class ChangeSet
{
    public function __construct(
        public EntityHandle $entity,
        public FieldChangeCollection $fields,
        public EmbeddedChangeCollection $embedded,
        public RelationshipChangeCollection $relationships,
        public ChangeSetFingerprint $fingerprint,
    ) {}
}
```

---

# 29. ChangeSet immutable

Una vez producido para una fase estable:

```text
ChangeSet
```

deberá ser immutable.

---

# 30. Mutable detection context ≠ ChangeSet

Durante detección puede existir:

```text
MutableChangeDetectionContext
```

pero el resultado será:

```text
Immutable ChangeSet
```

---

# 31. FieldChange

```php
final readonly class FieldChange
{
    public function __construct(
        public PersistentPropertyId $property,
        public ValueState $original,
        public ValueState $current,
        public ChangeKind $kind,
    ) {}
}
```

---

# 32. ChangeKind

```php
enum ChangeKind
{
    case VALUE_CHANGED;
    case VALUE_ASSIGNED;
    case VALUE_CLEARED;
    case VALUE_INITIALIZED;
    case VALUE_REPLACED;
    case UNKNOWN_ORIGINAL;
}
```

---

# 33. VALUE_CHANGED

Ejemplo:

```text
Bob → Alice
```

---

# 34. VALUE_ASSIGNED

Ejemplo:

```text
uninitialized → Alice
```

cuando esa transición sea persistente y válida.

---

# 35. VALUE_CLEARED

Ejemplo:

```text
alice@example.com → null
```

---

# 36. UNKNOWN_ORIGINAL

Debe conservar incertidumbre:

```text
? → Alice
```

No deberá fingirse:

```text
null → Alice
```

---

# 37. ChangeSet empty

```php
$changeSet->isEmpty();
```

significa:

```text
no persistent differences detected
```

bajo el tracking plan utilizado.

---

# 38. Empty ChangeSet ≠ DB unchanged globally

Otro proceso pudo modificar la DB.

Solo significa:

```text
this persistence context detected no local persistent change
```

---

# 39. Change tracking strategies

VoltStack soportará:

```text
SNAPSHOT
EXPLICIT
NOTIFY
CUSTOM
```

---

# 40. SNAPSHOT strategy

Es la estrategia general recomendada.

```text
Original Snapshot
       +
Current State
       ↓
Compare
       ↓
ChangeSet
```

---

# 41. Ventajas SNAPSHOT

```text
POPO friendly
no entity instrumentation required
works with private properties
domain model stays ORM-light
simple mental model
```

---

# 42. Coste SNAPSHOT

Puede requerir:

```text
O(number of tracked properties)
```

por entidad inspeccionada.

---

# 43. EXPLICIT strategy

La entidad solo se inspecciona cuando fue marcada:

```text
explicitly dirty
```

---

# 44. Ejemplo conceptual

```php
$changeTracker->markDirty($user);
```

o mediante infraestructura interna:

```text
UnitOfWork::scheduleForDirtyCheck()
```

---

# 45. EXPLICIT ≠ manually define SQL

Solo limita:

```text
which entities/properties need comparison
```

---

# 46. NOTIFY strategy

Las mutaciones notifican cambios.

Conceptualmente:

```php
$this->name = $name;

$this->changeNotifier->propertyChanged(
    'name',
    $old,
    $name,
);
```

---

# 47. Entity pollution concern

VoltStack no deberá exigir NOTIFY para todas las entidades.

Sería incompatible con el objetivo:

```text
POPO entities without mandatory ORM base class
```

---

# 48. Instrumentation alternative

En futuras versiones NOTIFY podría provenir de:

```text
generated proxies
instrumented models
compiled accessors
specialized Model API
```

sin obligar al dominio puro.

---

# 49. CUSTOM strategy

Permite:

```text
specialized immutable models
external state containers
event-sourced projections
custom property semantics
```

siempre bajo contratos explícitos.

---

# 50. Strategy selection

Se define en:

```text
Entity Metadata
```

y se compila en:

```text
ChangeTrackingPlan
```

---

# 51. ChangeTrackingPlan

```php
final readonly class ChangeTrackingPlan
{
    public function __construct(
        public EntityType $entityType,
        public ChangeTrackingStrategy $strategy,
        public TrackedPropertyCollection $properties,
        public ComparisonPlanCollection $comparisons,
        public EmbeddedTrackingPlanCollection $embedded,
        public RelationshipTrackingPlanCollection $relationships,
        public ChangeTrackingPlanFingerprint $fingerprint,
    ) {}
}
```

---

# 52. Runtime reflection avoidance

VoltStack deberá evitar repetir:

```text
ReflectionProperty discovery
attribute parsing
mapping interpretation
```

en cada dirty check.

---

# 53. Compilation pipeline

```text
Entity Metadata
      ↓
Change Tracking Metadata Analyzer
      ↓
Tracking Plan Compiler
      ↓
Compiled ChangeTrackingPlan
      ↓
Metadata Cache
      ↓
Runtime ChangeTracker
```

---

# 54. Compiled plan

Puede contener:

```text
property accessors
type comparators
normalizers
embedded plans
loaded-state rules
mutability rules
version rules
generated-field rules
```

---

# 55. Shared immutable plans

En FrankenPHP:

```text
Worker
├── Compiled ChangeTrackingPlans  ← shared immutable
│
├── Request A
│   └── Change Detection Context A
│
└── Request B
    └── Change Detection Context B
```

---

# 56. No shared snapshots

Aunque los plans puedan compartirse:

```text
Snapshots
ChangeSets
dirty markers
notification buffers
```

serán scoped.

---

# 57. Property extraction

Change Tracking necesita leer estado actual.

Se utilizará:

```text
CompiledPropertyAccessor
```

o abstracción equivalente.

---

# 58. No getter requirement

Una entidad no deberá requerir:

```php
public function getName(): string
```

solo para que ORM detecte cambios.

---

# 59. Private properties

Deberán poder soportarse mediante:

```text
compiled reflection access
generated accessor
mapping accessor
custom accessor
```

---

# 60. Property accessor ≠ serialization

Leer una propiedad para persistence tracking no equivale a:

```text
JSON serialization
API serialization
```

---

# 61. Canonical comparison

La comparación deberá realizarse con semántica del tipo persistente.

Ejemplo:

```text
PHP DateTimeImmutable
```

puede requerir comparación mediante:

```text
instant
timezone policy
precision
```

---

# 62. PHP strict equality no siempre basta

Incorrecto como regla universal:

```php
$original !== $current;
```

---

# 63. Typed comparator

Se propone:

```php
interface PersistentValueComparator
{
    public function equals(
        mixed $original,
        mixed $current,
        ComparisonContext $context,
    ): bool;
}
```

---

# 64. Comparator source

Normalmente deberá derivarse del:

```text
Database Type System
+
ORM Mapping
```

---

# 65. One type system

Como estableció ORM Architecture:

> El ORM no deberá crear un segundo sistema de tipos incompatible con Database Type System.

---

# 66. Canonicalization

Puede utilizarse:

```text
PHP Value
   ↓
Type Normalizer
   ↓
Canonical Persistent Value
```

antes de comparar.

---

# 67. Canonicalization ≠ DB serialization

No necesariamente será exactamente la representación física del driver.

---

# 68. Example UUID

```text
UserId object
UUID string
binary UUID database representation
```

pueden representar la misma identidad/valor persistente bajo mappings distintos.

---

# 69. String comparison

La semántica deberá ser explícita.

No deberá asumir:

```text
case-insensitive DB collation
```

como equivalencia automática de objetos PHP.

---

# 70. Database equality ≠ ORM equality

Ejemplo:

```text
DB collation:
"Alice" = "alice"

Domain:
"Alice" ≠ "alice"
```

VoltStack no deberá borrar esa diferencia accidentalmente.

---

# 71. Decimal values

No deberán compararse mediante float si el mapping representa:

```text
DECIMAL
```

---

# 72. Decimal canonical form

Ejemplo:

```text
"10.00"
"10.0"
```

pueden considerarse equivalentes o distintos según:

```text
type semantics
scale policy
```

---

# 73. Float tracking

Para propiedades realmente float:

```text
NaN
Infinity
-0.0
precision
```

deberán tener política explícita.

---

# 74. Approximate comparison

No se usará:

```text
epsilon comparison
```

universalmente.

Solo cuando el type contract lo declare.

---

# 75. Date/time precision

DB podría almacenar:

```text
microseconds
milliseconds
seconds
```

El comparator deberá conocer la precisión persistente efectiva cuando sea relevante.

---

# 76. Timezone normalization

Ejemplo:

```text
2026-09-06T10:00:00-06:00
2026-09-06T16:00:00Z
```

pueden representar el mismo instant.

La igualdad dependerá del type mapping.

---

# 77. JSON

Comparar JSON requiere política.

Dos estructuras:

```json
{"a":1,"b":2}
```

y:

```json
{"b":2,"a":1}
```

podrían ser semánticamente equivalentes para un JSON object.

---

# 78. JSON array

En cambio:

```json
[1,2]
```

y:

```json
[2,1]
```

normalmente no son equivalentes.

---

# 79. Canonical JSON comparator

El Type System podrá proporcionar:

```text
canonical object key ordering
typed scalar normalization
array-order preservation
```

---

# 80. Mutable arrays

PHP arrays pueden modificarse in-place:

```php
$user->settings['theme'] = 'dark';
```

Snapshot tracking deberá poder detectar el cambio.

---

# 81. Snapshot aliasing problem

Incorrecto:

```text
snapshot
and
entity property

point to same mutable object
```

porque una mutación podría alterar ambos.

---

# 82. Snapshot isolation

El Snapshot System deberá preservar un baseline semánticamente independiente.

Los detalles corresponden al documento 126.

---

# 83. Mutable objects

Ejemplo:

```php
$user->preferences()->setTheme('dark');
```

Si `preferences` es Value Object mutable, el sistema necesita saber si:

```text
object reference equality
```

es suficiente.

Normalmente no lo será.

---

# 84. Mutability classification

Se propone:

```php
enum PersistentValueMutability
{
    case IMMUTABLE;
    case MUTABLE;
    case STRUCTURALLY_MUTABLE;
    case UNKNOWN;
}
```

---

# 85. IMMUTABLE

Ejemplos típicos:

```text
int
string
bool
immutable ID object
DateTimeImmutable
```

según type contract.

---

# 86. MUTABLE

Puede requerir:

```text
deep snapshot
canonical serialization
copy strategy
custom comparator
```

---

# 87. UNKNOWN mutability

No deberá asumirse segura.

Puede generar:

```text
mapping validation warning/error
```

si snapshot tracking depende de ello.

---

# 88. Value Object tracking

Un Value Object embebido puede rastrearse:

```text
as a whole
```

o:

```text
by persistent members
```

según mapping.

---

# 89. Whole-value strategy

```text
Address(old)
vs
Address(new)
```

---

# 90. Member strategy

```text
Address.street
Address.city
Address.postalCode
```

pueden generar cambios individuales.

---

# 91. EmbeddedChange

```php
final readonly class EmbeddedChange
{
    public function __construct(
        public EmbeddedPath $path,
        public ChangeSet $changes,
    ) {}
}
```

---

# 92. Embedded path

Ejemplo:

```text
billingAddress.city
```

No deberá confundirse con:

```text
database column path
```

---

# 93. Nested embedded objects

Deberán soportarse con:

```text
bounded recursive tracking plans
```

---

# 94. Cyclic embedded values

Un embedded Value Object graph persistido por valor debería ser acíclico por defecto.

Si se detecta un ciclo no soportado:

```text
EmbeddedValueCycleException
```

---

# 95. Entity relationships

Una propiedad que referencia otra Entity:

```php
private Customer $customer;
```

no deberá compararse mediante deep object equality.

---

# 96. Entity relationship comparison

Debe utilizar:

```text
EntityKey
```

cuando exista identidad estable.

---

# 97. Same entity different instance

Dentro de un PersistenceContext normal IdentityMap debería evitarlo.

Aun así:

```text
different PHP instance
same EntityKey
```

no deberá interpretarse automáticamente como cambio de FK si la política admite esa comparación.

---

# 98. Relationship reference

Puede compararse mediante:

```text
EntityReference
EntityKey
CanonicalIdentifier
```

según estado de carga.

---

# 99. Relationship change

Ejemplo:

```text
Order.customer

Customer#10
    →
Customer#20
```

---

# 100. Relationship change ≠ field change

Aunque físicamente pueda convertirse en:

```text
customer_id UPDATE
```

semánticamente seguirá siendo:

```text
RelationshipChange
```

hasta Persistence Planning.

---

# 101. To-many relationships

Las colecciones tienen sistema especializado.

Change Tracking puede detectar:

```text
collection potentially changed
```

pero:

```text
Collection Change Tracking
```

deberá resolver:

```text
added
removed
reordered
```

---

# 102. Collection ≠ scalar field

Nunca comparar:

```php
$originalCollection != $currentCollection
```

como política ORM universal.

---

# 103. Collection tracking boundary

```text
Change Tracking
        │
        ▼
CollectionChangeAnalyzer
        │
        ▼
CollectionChangeSet
```

---

# 104. Collection loaded state

Una colección:

```text
UNLOADED
```

no deberá materializarse automáticamente solo para dirty checking.

---

# 105. No hidden lazy loading

Regla crítica:

> **Change detection no deberá disparar consultas de base de datos ocultas únicamente para comparar propiedades o relaciones no cargadas.**

---

# 106. Unloaded relationship

Si no fue cargada y no fue modificada:

```text
no change
```

puede inferirse mediante instrumentation/loaded-state metadata.

---

# 107. Explicit assignment to unloaded relationship

Si una relación unloaded recibe un nuevo valor:

```text
original = UNLOADED
current = Customer#20
```

no deberá inventarse el original.

---

# 108. Unknown original relationship

Puede requerir:

```text
blind assignment persistence
```

o:

```text
original-value requirement
```

según optimistic locking, orphan removal y mapping.

La decisión final pertenece a Persistence Planning.

---

# 109. Partial entities

Supongamos:

```text
User#10

loaded:
id
name

unloaded:
email
status
```

---

# 110. Dirty checking partial entity

Solo propiedades cuyo baseline sea conocido o cuya mutación explícita esté registrada deberán evaluarse normalmente.

---

# 111. Partial entity safety

Nunca:

```text
unloaded email
→ current PHP default null
→ persist NULL
```

---

# 112. Uninitialized typed properties

PHP permite:

```php
private string $email;
```

sin inicializar.

VoltStack deberá distinguir:

```text
UNINITIALIZED
```

de:

```text
NULL
```

---

# 113. Accessor safety

Compiled accessor deberá poder inspeccionar:

```text
initialized?
```

antes de leer una typed property.

---

# 114. Uninitialized persistent property

Puede significar:

```text
not loaded
not assigned yet
invalid entity state
generated later
```

según metadata.

---

# 115. Identifier fields

Los identificadores requieren reglas especiales.

---

# 116. Identifier mutation detection

Una vez establecida identidad persistente:

```text
ID old ≠ ID current
```

deberá generar:

```text
EntityIdentifierMutationException
```

por defecto.

---

# 117. Identifier change ≠ normal ChangeSet

Nunca deberá convertirse automáticamente en:

```text
UPDATE primary_key
```

---

# 118. Composite identifiers

Cada parte deberá compararse de forma tipada.

```text
OrderLineId(
    orderId,
    lineNumber
)
```

---

# 119. Composite ID mutation

Cualquier cambio en una parte establecida deberá considerarse identity mutation.

---

# 120. Generated identifiers

Para una entidad NEW:

```text
id = UNASSIGNED
```

puede ser válido.

Después del INSERT confirmado:

```text
id = GENERATED
```

no debe interpretarse como un user field change normal.

---

# 121. Generated field tracking

Campos generados por DB pueden incluir:

```text
auto IDs
timestamps
computed columns
database defaults
generated columns
version values
```

---

# 122. GeneratedFieldPolicy

Se propone:

```php
enum GeneratedFieldPolicy
{
    case NEVER_TRACK_AS_USER_CHANGE;
    case TRACK_AFTER_SYNCHRONIZATION;
    case DATABASE_AUTHORITATIVE;
    case APPLICATION_AUTHORITATIVE;
    case CUSTOM;
}
```

---

# 123. DB authoritative

Si la DB controla un campo:

```text
current entity mutation
```

puede ser:

```text
ignored
rejected
diagnosed
```

según mapping.

Nunca deberá aceptarse accidentalmente.

---

# 124. Computed columns

Normalmente:

```text
read-only persistence property
```

y no generan updates.

---

# 125. Read-only field

Modificar un read-only mapped field puede generar:

```text
ReadOnlyPersistentPropertyMutationException
```

o warning según política.

---

# 126. Version fields

Optimistic locking requiere distinguir:

```text
business change
```

de:

```text
ORM-generated version advancement
```

---

# 127. Version baseline

El ChangeSet puede transportar:

```text
expectedVersion
```

sin tratarla necesariamente como un user modification.

---

# 128. Version advancement

Persistence Engine puede solicitar:

```text
version 7 → 8
```

como persistence-generated change.

---

# 129. Persistence-generated change ≠ domain change

Deberán distinguirse.

---

# 130. Change origin

Se propone:

```php
enum ChangeOrigin
{
    case DOMAIN;
    case ORM;
    case DATABASE;
    case HYDRATION;
    case EXTENSION;
    case UNKNOWN;
}
```

---

# 131. FieldChange origin

Puede incluir:

```text
origin = DOMAIN
```

o:

```text
origin = ORM
```

para version advancement.

---

# 132. Hydration ≠ modification

Durante hydration:

```text
assign fields
```

no deberá generar dirty state.

---

# 133. Hydration suppression

El Entity Hydrator deberá operar bajo:

```text
hydration context
```

que no registre cada assignment como domain mutation.

---

# 134. Snapshot establishment

Después de hydration:

```text
hydrated state
→ baseline snapshot
```

---

# 135. postLoad mutation

Si un lifecycle callback modifica una propiedad después de hydration, la política deberá determinar si:

```text
snapshot before callback
```

o:

```text
snapshot after callback
```

representa el baseline.

---

# 136. Recommended rule

Para V1:

```text
hydrate persistent state
→ execute controlled postLoad
→ establish managed baseline
```

si `postLoad` solo normaliza representación y está contractualmente permitido.

Pero cualquier callback que represente una verdadera domain mutation deberá ser explícito.

---

# 137. No hidden normalization ambiguity

La fase deberá quedar definida por Lifecycle System.

---

# 138. Property normalization

Antes de comparación puede aplicarse:

```text
PersistentValueNormalizer
```

---

# 139. Normalization examples

```text
enum normalization
UUID normalization
decimal normalization
date precision normalization
JSON canonicalization
```

---

# 140. Normalizer ≠ mutator

No deberá modificar la entidad.

Debe producir:

```text
comparison representation
```

---

# 141. Comparison representation

Se propone:

```php
final readonly class ComparisonValue
{
    public function __construct(
        public mixed $value,
        public ComparisonValueType $type,
    ) {}
}
```

---

# 142. Comparison representation ≠ database binding value

Puede ser distinta de:

```text
driver parameter
```

---

# 143. Type conversion failures

Si una propiedad no puede normalizarse:

```text
ChangeTrackingTypeConversionException
```

---

# 144. Invalid current value

Ejemplo:

```text
mapped UUID
current value = invalid string
```

Change Tracking no deberá fingir:

```text
changed
```

simplemente.

Debe reportar:

```text
invalid persistent value
```

---

# 145. Validation ≠ Change Tracking

Change Tracking no reemplaza:

```text
Validation System
```

pero debe proteger sus propios invariantes de comparación.

---

# 146. New entities

Una entidad NEW no requiere necesariamente comparar contra snapshot.

---

# 147. NEW persistence state

Puede generar:

```text
InitialPersistentState
```

en lugar de:

```text
ChangeSet from old row
```

---

# 148. InitialEntityState

Se recomienda separar:

```text
NewEntityState
```

de:

```text
ManagedEntityChangeSet
```

cuando simplifique Persistence Planning.

---

# 149. NEW entity ≠ all fields changed

Conceptualmente:

```text
NEW
```

significa:

```text
requires insertion
```

no:

```text
every property changed from null
```

---

# 150. Removed entities

Una entidad REMOVED no necesita dirty checking para decidir que debe eliminarse.

---

# 151. Changes before removal

Dependiendo de cascades/audit/events:

```text
changes made before remove()
```

podrían seguir siendo relevantes para diagnostics/events.

Pero no necesariamente requieren UPDATE antes de DELETE.

---

# 152. Planner responsibility

UnitOfWork/Persistence Planner decidirán si esos changes son:

```text
ignored
needed
invalid
event-relevant
```

---

# 153. Detached entities

No deberán dirty-checkearse dentro del UnitOfWork.

---

# 154. Reattached entities

Si en futuro se soporta reattach:

```text
new baseline policy
```

deberá ser explícita.

---

# 155. No implicit merge

VoltStack V1 no deberá comparar detached object vs database para implementar un merge mágico.

---

# 156. Notification tracking architecture

```text
Entity Mutation
      ↓
Change Notification
      ↓
Notification Buffer
      ↓
Change Normalizer
      ↓
ChangeSet
```

---

# 157. Notification buffer scoped

Nunca:

```text
static global notifications
```

---

# 158. Notification trust

Un NOTIFY strategy puede:

```text
trust notification
```

o:

```text
verify notification against current value
```

según política.

---

# 159. Recommended V1

Para integridad:

```text
notification narrows candidate fields
+
typed comparison verifies final state
```

---

# 160. Why

Evita:

```text
duplicate notification
reverted change
incorrect old value
intermediate mutation noise
```

---

# 161. Example

```text
Bob → Alice
Alice → Bob
```

Durante la request hubo dos mutaciones.

El final es:

```text
Bob
```

Por tanto:

```text
final ChangeSet = empty
```

si el baseline era Bob.

---

# 162. Mutation log ≠ ChangeSet

Importante:

```text
Mutation History
≠
Final ChangeSet
```

---

# 163. Event sourcing distinction

Change Tracking ORM no será un event store.

---

# 164. Intermediate mutations

Pueden ser útiles para:

```text
domain events
audit
debugging
```

pero no deberán confundirse con final persistence delta.

---

# 165. Explicit tracking architecture

```text
Application / Model API
       ↓
markDirty(entity)
       ↓
DirtyCandidateRegistry
       ↓
flush preparation
       ↓
typed comparison
       ↓
ChangeSet
```

---

# 166. Dirty candidate ≠ dirty entity

Una entidad marcada como candidata puede terminar con:

```text
empty ChangeSet
```

---

# 167. DirtyCandidateRegistry

Deberá pertenecer al PersistenceContext.

---

# 168. Property-level dirty candidates

Opcionalmente:

```text
Entity → [name, email]
```

puede reducir trabajo.

---

# 169. Property-level hints ≠ truth

El sistema puede verificar el estado final.

---

# 170. Snapshot strategy optimization

No necesariamente se deben comparar todas las entidades managed.

---

# 171. Candidate selection

Podrán usarse:

```text
explicit mutation hints
Model API instrumentation
collection dirty markers
relationship assignment markers
known immutable entity behavior
read-only flags
```

para reducir trabajo.

---

# 172. Optimization ≠ semantic change

La optimización no deberá cambiar qué ChangeSet resulta.

---

# 173. ChangeTrackingScope

Se propone:

```php
final class ChangeTrackingScope
{
    public function __construct(
        public readonly PersistenceContextId $context,
        public readonly FlushCycleId $flushCycle,
    ) {}
}
```

---

# 174. Tracking session

Cada preparación de flush puede crear:

```text
ChangeDetectionSession
```

---

# 175. Session state

Puede contener:

```text
visited entities
dirty candidates
comparison cache
temporary normalized values
diagnostics
resource counters
```

---

# 176. Session ≠ shared runtime cache

Se destruye al terminar la detección.

---

# 177. Comparison cache

Puede evitar repetir normalización costosa dentro del mismo cycle.

---

# 178. No stale comparison cache

Nunca reutilizar:

```text
current-value comparison cache
```

entre flush cycles sin demostrar validez.

---

# 179. ChangeSet fingerprint

Cada ChangeSet podrá tener:

```text
ChangeSetFingerprint
```

---

# 180. Fingerprint purpose

```text
stale plan detection
debugging
telemetry correlation
persistence reconciliation
determinism testing
```

---

# 181. Fingerprint inputs

```text
EntityHandle
TrackedProperty IDs
Canonical original values
Canonical current values
Change kinds
Relevant metadata version
```

---

# 182. Sensitive fingerprinting

Sensitive values deberán:

```text
hash/canonical digest
```

sin almacenarse en claro cuando no sea necesario.

---

# 183. Fingerprint ≠ security hash guarantee

A menos que el contrato lo especifique, su función principal es:

```text
semantic identity / drift detection
```

---

# 184. ChangeSet ordering

Para determinismo:

```text
FieldChanges
```

deberán ordenarse por un criterio estable:

```text
PersistentPropertyId
```

o compiled metadata order.

---

# 185. Reflection order

No deberá ser la única autoridad accidental.

---

# 186. Relationship ordering

También deberá normalizarse.

---

# 187. ChangeSet determinism

Mismos:

```text
metadata
baseline
current state
tracking strategy
```

deberán producir el mismo ChangeSet semántico.

---

# 188. Concurrency

Un EntityManager/UnitOfWork mutable no será thread/coroutine safe por defecto.

---

# 189. Concurrent entity mutation

Si una entidad cambia mientras está siendo dirty-checked:

```text
race
```

puede invalidar el ChangeSet.

---

# 190. Mutation generation

Puede existir:

```text
EntityMutationGeneration
```

para detectar cambios durante preparación cuando instrumentation lo permita.

---

# 191. Generic POPO limitation

Para entidades no instrumentadas no siempre será posible detectar una carrera arbitraria.

Por tanto:

> Un PersistenceContext mutable pertenece a un único flujo lógico de ejecución.

---

# 192. Coroutine safety

En OpenSwoole futuro:

```text
one coroutine
→ one scoped EntityManager
→ one UnitOfWork
```

salvo implementación específicamente concurrency-safe.

---

# 193. Change tracking during flush

Una vez alcanzado:

```text
FROZEN
```

el ChangeSet utilizado por ese flush no cambiará.

---

# 194. Mutations after freeze

Deberán:

```text
be rejected during critical phase
```

o:

```text
belong to next flush cycle
```

según lifecycle policy.

---

# 195. Lifecycle callback interaction

`preUpdate` puede necesitar conocer:

```text
ChangeSet
```

---

# 196. Lifecycle mutation problem

Callback:

```php
#[PreUpdate]
public function normalizeName(): void
{
    $this->name = trim($this->name);
}
```

cambia el estado después de una detección inicial.

---

# 197. Stabilization

Pipeline:

```text
Initial Change Detection
        ↓
Pre-Lifecycle Hooks
        ↓
Affected Entity Recheck
        ↓
Updated ChangeSet
        ↓
Repeat if necessary
        ↓
Stable ChangeSet
```

---

# 198. Stable ChangeSet

Solo después de convergencia:

```text
ChangeSet
→ PreparedUnitOfWork
```

---

# 199. Lifecycle iteration limit

Como doc 124:

```text
maxStabilizationPasses
```

---

# 200. Oscillation detection

Ejemplo:

```text
Pass 1:
A → B

Pass 2:
B → A

Pass 3:
A → B
```

podrá detectarse mediante:

```text
ChangeSet fingerprints
```

---

# 201. Oscillation exception

```text
ChangeTrackingNonConvergenceException
```

---

# 202. Relationship lifecycle mutation

Callbacks también pueden cambiar relaciones.

Entonces:

```text
field ChangeSet stabilization
```

y:

```text
relationship graph stabilization
```

deberán coordinarse mediante UnitOfWork.

---

# 203. Change detection pipeline

```text
Entity
  │
  ▼
Resolve Entity Metadata
  │
  ▼
Resolve Tracking Plan
  │
  ▼
Resolve Baseline
  │
  ▼
Select Candidate Properties
  │
  ▼
Extract Current Values
  │
  ▼
Check Loaded/Initialized State
  │
  ▼
Normalize Original Values
  │
  ▼
Normalize Current Values
  │
  ▼
Typed Compare
  │
  ├── Equal ────────→ No Change
  │
  └── Different
         │
         ▼
     Build Change
         │
         ▼
     Validate Change
         │
         ▼
      ChangeSet
```

---

# 204. Validation stage

Debe detectar:

```text
identifier mutation
read-only mutation
invalid generated-field mutation
unknown baseline restrictions
invalid type values
unsupported mutable value
```

---

# 205. Change validation ≠ business validation

No deberá validar:

```text
email format
minimum age
business credit limit
```

salvo que esas reglas pertenezcan a otro sistema.

---

# 206. Tracking policy

Se propone:

```php
final readonly class ChangeTrackingPolicy
{
    public function __construct(
        public UnknownBaselinePolicy $unknownBaseline,
        public ReadOnlyMutationPolicy $readOnlyMutation,
        public GeneratedFieldMutationPolicy $generatedFieldMutation,
        public MutableValuePolicy $mutableValues,
        public PartialEntityPolicy $partialEntities,
    ) {}
}
```

---

# 207. UnknownBaselinePolicy

Opciones conceptuales:

```text
REJECT
ALLOW_EXPLICIT_ASSIGNMENT
REQUIRE_REFRESH
DEFER_TO_PERSISTENCE_PLAN
CUSTOM
```

---

# 208. ReadOnlyMutationPolicy

```text
REJECT
WARN_AND_IGNORE
IGNORE
CUSTOM
```

En producción segura:

```text
REJECT
```

deberá preferirse para mutaciones claramente inválidas.

---

# 209. Generated field policy

No deberá ser implícita.

---

# 210. Metadata validation

Muchas inconsistencias deberán detectarse en boot/metadata compilation:

```text
mutable type without snapshot strategy
invalid comparator
generated + writable conflict
identifier + mutable conflict
partial loading incompatible with tracking strategy
```

---

# 211. Boot-time validation

Mejor:

```text
fail during metadata compilation
```

que descubrirlo después de modificar producción.

---

# 212. Tracking completeness

Un ChangeSet podrá declarar:

```php
enum ChangeSetCompleteness
{
    case COMPLETE;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 213. COMPLETE

Todos los persistent members relevantes fueron evaluados.

---

# 214. PARTIAL

Solo una parte puede afirmarse.

---

# 215. UNKNOWN

No existe evidencia suficiente para declarar un delta completo.

---

# 216. Completeness ≠ empty

```text
PARTIAL + zero detected changes
```

no equivale necesariamente a:

```text
complete no-op
```

---

# 217. Persistence Planner consumes completeness

Puede decidir:

```text
safe partial UPDATE
require refresh
reject operation
```

---

# 218. Partial UPDATE

Una ventaja del ChangeSet tipado es poder generar:

```text
UPDATE only changed known fields
```

sin sobrescribir unloaded state.

---

# 219. Lost update

Aun así:

```text
partial UPDATE
```

no evita automáticamente lost updates.

Eso requiere:

```text
optimistic locking
transaction isolation
concurrency policy
```

---

# 220. Change tracking ≠ concurrency control

Regla:

```text
Detecting a change
≠
protecting against concurrent change
```

---

# 221. Optimistic locking integration

ChangeSet puede aportar:

```text
original version token
```

al Persistence Planner.

---

# 222. Original value predicates

Una estrategia avanzada podría generar:

```text
WHERE field = originalValue
```

para compare-and-swap.

Pero esto pertenece a Persistence/Concurrency, no Change Tracking.

---

# 223. Database-generated defaults

Para NEW entity:

```text
UNASSIGNED property
```

puede significar:

```text
allow DB default
```

No deberá convertirse automáticamente en:

```text
INSERT NULL
```

---

# 224. Assignment state

Se requiere distinguir:

```text
UNASSIGNED
ASSIGNED_NULL
ASSIGNED_VALUE
```

para inserts.

---

# 225. Insert state ≠ ChangeSet

Esto refuerza que NEW entity persistence será tratada por:

```text
129_DATABASE_INSERT_PERSISTENCE_SYSTEM.md
```

---

# 226. Delete state

Igualmente:

```text
REMOVED
```

no necesita representarse como fields cambiados.

---

# 227. Audit integration

ChangeSet es una buena fuente para:

```text
audit descriptions
```

pero Audit System no deberá depender de valores sin redaction.

---

# 228. Audit ≠ snapshot

No almacenar automáticamente snapshots completos como audit logs.

---

# 229. Telemetry

Métricas propuestas:

```text
orm.change_tracking.entities.scanned
orm.change_tracking.entities.dirty
orm.change_tracking.entities.clean
orm.change_tracking.properties.compared
orm.change_tracking.properties.changed
orm.change_tracking.relationships.changed
orm.change_tracking.embedded.changed
orm.change_tracking.partial_entities
orm.change_tracking.unknown_baselines
orm.change_tracking.duration
orm.change_tracking.normalization.duration
orm.change_tracking.comparison.duration
orm.change_tracking.stabilization.passes
orm.change_tracking.plan_cache.hit
orm.change_tracking.plan_cache.miss
```

---

# 230. Performance model

Aproximadamente:

```text
SnapshotDirtyCheckingCost
≈
Σ TrackedProperties(entity)
```

sobre las entidades candidatas.

---

# 231. Optimization objective

Reducir:

```text
CandidateEntities
×
ComparedProperties
×
ComparisonCost
```

sin cambiar semántica.

---

# 232. Compiled accessor optimization

Evitar reflexión repetitiva es importante para:

```text
FrankenPHP persistent workers
```

---

# 233. Metadata compilation

Los plans podrán permanecer:

```text
process-shared immutable
```

durante múltiples requests.

---

# 234. Comparator specialization

Para tipos comunes podrán existir comparadores optimizados:

```text
IntegerComparator
StringComparator
BooleanComparator
UuidComparator
DateTimeComparator
DecimalComparator
EnumComparator
JsonComparator
ValueObjectComparator
EntityReferenceComparator
```

---

# 235. Comparator registry

```text
PersistentValueComparatorRegistry
```

deberá estar:

```text
frozen
```

durante runtime normal.

---

# 236. Comparator collision

Dos comparadores para la misma resolución no deberán resolverse mediante:

```text
last registration wins
```

---

# 237. Extension contract

```php
interface ChangeTrackingStrategy
{
    public function detect(
        object $entity,
        EntitySnapshot $baseline,
        ChangeTrackingPlan $plan,
        ChangeDetectionContext $context,
    ): ChangeSet;
}
```

---

# 238. Strategy registry

```text
ChangeTrackingStrategyRegistry
```

contendrá:

```text
SNAPSHOT
EXPLICIT
NOTIFY
CUSTOM providers
```

---

# 239. Custom strategy restrictions

No podrá:

```text
execute arbitrary SQL
mutate EntityManager global state
switch tenant context
hide unknown baseline
bypass identifier immutability
```

---

# 240. Custom comparator restrictions

Debe ser:

```text
deterministic
side-effect-free
context-bounded
```

---

# 241. Comparator purity

Idealmente:

```text
equals(a,b)
```

no deberá:

```text
query database
modify entity
dispatch business commands
read mutable globals
```

---

# 242. Deterministic comparison

Para mismos valores canónicos:

```text
equals(a,b)
```

deberá producir el mismo resultado.

---

# 243. External services

Un comparator no deberá llamar:

```text
HTTP API
Redis
database
filesystem
```

para decidir igualdad.

---

# 244. Why

Dirty checking debe permanecer:

```text
fast
local
predictable
reproducible
```

---

# 245. Runtime isolation

Estado mutable scoped:

```text
DirtyCandidateRegistry
NotificationBuffer
ChangeDetectionSession
ComparisonCache
ChangeSets
Diagnostics
```

---

# 246. Shared immutable

Podrán compartirse:

```text
Compiled ChangeTrackingPlans
Comparator definitions
Accessor definitions
Type metadata
Frozen strategy registry
```

---

# 247. Request reset

Al terminar request:

```text
dirty candidates → clear
notification buffers → clear
session caches → clear
ChangeSets → discard with UoW
```

---

# 248. No cross-request ChangeSet

Nunca:

```text
Request A ChangeSet
→ Request B UnitOfWork
```

---

# 249. Tenant isolation

Si el PersistenceContext pertenece a:

```text
Tenant A
```

su tracking state no podrá reutilizarse para:

```text
Tenant B
```

---

# 250. ChangeSet does not need tenant field everywhere

El tenant pertenece al:

```text
PersistenceContext / IdentityNamespace
```

No deberá hardcodearse `tenant_id` en Change Tracking core.

---

# 251. Sharding

Misma regla:

```text
ShardContext
```

es externo al field delta.

---

# 252. Debug information

Ejemplo:

```text
Entity:
App\Entity\User

Identity:
User#10

Tracking Strategy:
SNAPSHOT

Properties Compared:
8

Changes:
2

name:
    Bob
    → Alice

email:
    bob@example.com
    → alice@example.com

Completeness:
COMPLETE
```

---

# 253. Sensitive values

Ejemplo:

```text
passwordHash:
    [REDACTED]
    →
    [REDACTED]
```

---

# 254. Diagnostic reasons

El sistema deberá poder explicar:

```text
why property was considered changed
```

---

# 255. Example explanation

```text
Property:
price

Type:
decimal(12,2)

Original:
10.00

Current:
10.01

Comparator:
DecimalPersistentValueComparator

Result:
DIFFERENT

Reason:
Canonical decimal values differ.
```

---

# 256. Unknown baseline diagnostic

```text
Property:
email

Original:
UNLOADED

Current:
alice@example.com

Completeness:
PARTIAL

Policy:
ALLOW_EXPLICIT_ASSIGNMENT
```

---

# 257. Identifier mutation diagnostic

```text
DB-ORM-CHANGE-ID-001

Entity:
App\Entity\User

Original Identifier:
UserId(10)

Current Identifier:
UserId(20)

Reason:
The identifier of a managed entity changed after persistent identity
was established.

Action:
Entity identifiers are immutable by default. Create a new entity or
use an explicitly supported identity migration workflow.
```

---

# 258. Mutable value diagnostic

```text
DB-ORM-CHANGE-MUTABLE-002

Entity:
App\Entity\User

Property:
preferences

Mapped Type:
App\Domain\Preferences

Mutability:
MUTABLE

Problem:
The configured snapshot strategy cannot guarantee an independent
baseline for this mutable value.

Action:
Configure a copy strategy, canonical comparator, immutable value
object, or custom tracking strategy.
```

---

# 259. Partial entity diagnostic

```text
DB-ORM-CHANGE-PARTIAL-003

Entity:
App\Entity\User#10

Property:
email

Baseline:
UNLOADED

Current:
NULL

Reason:
VoltStack cannot determine whether NULL represents a deliberate
assignment or an uninitialized/unloaded property.

Action:
Explicitly assign the property, refresh the entity, or use a mapping
policy compatible with partial loading.
```

---

# 260. Exception hierarchy

```text
DatabaseOrmException
└── ChangeTrackingException
    ├── ChangeTrackingPlanException
    ├── ChangeTrackingMetadataException
    ├── ChangeTrackingStrategyException
    ├── ChangeDetectionException
    ├── ChangeTrackingTypeConversionException
    ├── PersistentValueComparisonException
    ├── PersistentValueNormalizationException
    ├── UnknownBaselineException
    ├── PartialEntityChangeTrackingException
    ├── UnloadedPropertyMutationException
    ├── UninitializedPersistentPropertyException
    ├── EntityIdentifierMutationException
    ├── ReadOnlyPersistentPropertyMutationException
    ├── GeneratedPropertyMutationException
    ├── UnsupportedMutableValueException
    ├── EmbeddedValueCycleException
    ├── RelationshipChangeTrackingException
    ├── CollectionChangeTrackingException
    ├── ChangeTrackingNonConvergenceException
    ├── ChangeSetValidationException
    ├── ChangeSetFingerprintException
    ├── ChangeTrackingResourceLimitException
    ├── ChangeTrackingRuntimeIsolationException
    ├── ChangeTrackingConcurrencyException
    └── ChangeTrackingInvariantException
```

---

# 261. Arquitectura de directorios

```text
src/Quantum/Database/ORM/
│
├── ChangeTracking/
│   ├── Contract/
│   │   ├── ChangeTracker.php
│   │   ├── ChangeTrackingStrategy.php
│   │   ├── PersistentValueComparator.php
│   │   ├── PersistentValueNormalizer.php
│   │   └── ChangeSetValidator.php
│   │
│   ├── Core/
│   │   ├── DefaultChangeTracker.php
│   │   ├── ChangeDetectionContext.php
│   │   ├── ChangeDetectionSession.php
│   │   ├── ChangeTrackingScope.php
│   │   ├── ChangeTrackingStrategyType.php
│   │   └── ChangeSetCompleteness.php
│   │
│   ├── ChangeSet/
│   │   ├── ChangeSet.php
│   │   ├── ChangeSetBuilder.php
│   │   ├── ChangeSetCollection.php
│   │   ├── ChangeSetFingerprint.php
│   │   ├── FieldChange.php
│   │   ├── FieldChangeCollection.php
│   │   ├── EmbeddedChange.php
│   │   ├── EmbeddedChangeCollection.php
│   │   ├── RelationshipChange.php
│   │   ├── RelationshipChangeCollection.php
│   │   ├── ChangeKind.php
│   │   └── ChangeOrigin.php
│   │
│   ├── Plan/
│   │   ├── ChangeTrackingPlan.php
│   │   ├── ChangeTrackingPlanCompiler.php
│   │   ├── ChangeTrackingPlanRegistry.php
│   │   ├── ChangeTrackingPlanFingerprint.php
│   │   ├── TrackedProperty.php
│   │   ├── TrackedPropertyCollection.php
│   │   ├── ComparisonPlan.php
│   │   └── ComparisonPlanCollection.php
│   │
│   ├── Strategy/
│   │   ├── SnapshotChangeTrackingStrategy.php
│   │   ├── ExplicitChangeTrackingStrategy.php
│   │   ├── NotifyChangeTrackingStrategy.php
│   │   └── CustomChangeTrackingStrategyAdapter.php
│   │
│   ├── Candidate/
│   │   ├── DirtyCandidateRegistry.php
│   │   ├── DirtyCandidate.php
│   │   ├── DirtyPropertyCandidate.php
│   │   └── DirtyCandidateReason.php
│   │
│   ├── Notification/
│   │   ├── ChangeNotification.php
│   │   ├── ChangeNotificationBuffer.php
│   │   └── ChangeNotificationNormalizer.php
│   │
│   ├── Value/
│   │   ├── ValueState.php
│   │   ├── KnownValue.php
│   │   ├── NullValue.php
│   │   ├── UnloadedValue.php
│   │   ├── UninitializedValue.php
│   │   ├── UnknownValue.php
│   │   ├── ComparisonValue.php
│   │   └── PersistentValueMutability.php
│   │
│   ├── Comparison/
│   │   ├── PersistentValueComparatorRegistry.php
│   │   ├── IntegerComparator.php
│   │   ├── StringComparator.php
│   │   ├── BooleanComparator.php
│   │   ├── DecimalComparator.php
│   │   ├── FloatComparator.php
│   │   ├── DateTimeComparator.php
│   │   ├── EnumComparator.php
│   │   ├── JsonComparator.php
│   │   ├── ValueObjectComparator.php
│   │   └── EntityReferenceComparator.php
│   │
│   ├── Normalization/
│   │   ├── PersistentValueNormalizerRegistry.php
│   │   ├── ComparisonValueNormalizer.php
│   │   └── CanonicalValue.php
│   │
│   ├── Accessor/
│   │   ├── CompiledPropertyAccessor.php
│   │   ├── PropertyAccessorCompiler.php
│   │   ├── PropertyInitializationInspector.php
│   │   └── PropertyAccessorRegistry.php
│   │
│   ├── Embedded/
│   │   ├── EmbeddedTrackingPlan.php
│   │   ├── EmbeddedPath.php
│   │   ├── EmbeddedChangeTracker.php
│   │   └── EmbeddedCycleGuard.php
│   │
│   ├── Relationship/
│   │   ├── RelationshipTrackingPlan.php
│   │   ├── RelationshipChangeTracker.php
│   │   └── RelationshipValueComparator.php
│   │
│   ├── Policy/
│   │   ├── ChangeTrackingPolicy.php
│   │   ├── UnknownBaselinePolicy.php
│   │   ├── ReadOnlyMutationPolicy.php
│   │   ├── GeneratedFieldMutationPolicy.php
│   │   ├── MutableValuePolicy.php
│   │   └── PartialEntityPolicy.php
│   │
│   ├── Stabilization/
│   │   ├── ChangeSetStabilizer.php
│   │   ├── ChangeSetConvergenceDetector.php
│   │   └── ChangeSetOscillationDetector.php
│   │
│   ├── Runtime/
│   │   ├── ChangeTrackingRuntimeScope.php
│   │   ├── ChangeTrackingRuntimeResetter.php
│   │   └── ChangeTrackingOwnershipGuard.php
│   │
│   ├── Telemetry/
│   │   ├── ChangeTrackingTelemetry.php
│   │   ├── ChangeTrackingProfiler.php
│   │   └── ChangeTrackingDiagnostics.php
│   │
│   └── Exception/
│       └── ...
```

---

# 262. Public internal contract

Conceptualmente:

```php
interface ChangeTracker
{
    public function detectChanges(
        object $entity,
        EntitySnapshot $snapshot,
        ChangeTrackingContext $context,
    ): ChangeSet;
}
```

---

# 263. Batch detection contract

También:

```php
interface ChangeSetDetector
{
    public function detect(
        iterable $entities,
        ChangeDetectionContext $context,
    ): ChangeSetCollection;
}
```

---

# 264. Separation of contracts

`ChangeTracker` podrá trabajar por entidad.

`UnitOfWorkChangeDetector` podrá coordinar:

```text
many entities
candidate selection
resource budgets
stabilization
```

---

# 265. Resource governance

Configuración conceptual:

```php
final readonly class ChangeTrackingLimits
{
    public function __construct(
        public int $maxEntitiesPerPass,
        public int $maxPropertiesPerFlush,
        public int $maxEmbeddedDepth,
        public int $maxStabilizationPasses,
        public int $maxComparisonBytes,
    ) {}
}
```

---

# 266. Limits ≠ silent truncation

Si se excede:

```text
ChangeTrackingResourceLimitException
```

---

# 267. Comparison bytes

Tipos grandes:

```text
JSON
BLOB-like mapped values
large serialized value objects
```

pueden hacer costoso el comparison.

---

# 268. Large value strategies

Posibles:

```text
canonical hash
immutable version token
explicit dirty marker
custom comparator
```

---

# 269. Hash comparison

Puede optimizar valores grandes:

```text
Hash(original)
vs
Hash(current)
```

---

# 270. Hash collision semantics

No deberá utilizarse un hash no adecuado como prueba absoluta si el contrato requiere igualdad exacta.

---

# 271. Two-stage comparison

Podrá utilizarse:

```text
fast fingerprint comparison
        ↓ if uncertain/different
exact comparison
```

---

# 272. Memory model

Aproximadamente:

```text
ChangeTrackingMemory
=
SnapshotReferences
+
DirtyCandidateRegistry
+
CurrentComparisonValues
+
ChangeSets
+
ComparisonCache
+
Diagnostics
```

---

# 273. Avoid duplicate state

No deberán conservarse innecesariamente:

```text
full snapshot
+
full normalized snapshot
+
full serialized snapshot
+
full ChangeSet copy
```

si una representación compacta es suficiente.

---

# 274. Copy-on-write opportunities

Cuando sea seguro podrán utilizarse estrategias que reduzcan copias.

Pero:

```text
memory optimization
```

nunca deberá romper:

```text
snapshot independence
```

---

# 275. Long-running UnitOfWork

Dirty checking de:

```text
100,000 managed entities
```

es un anti-pattern.

---

# 276. Recommended operational model

Para grandes procesos:

```text
load chunk
process
flush
clear
repeat
```

---

# 277. Batch persistence

Se formalizará en:

```text
133_DATABASE_BATCH_PERSISTENCE_SYSTEM.md
```

---

# 278. Read-only query optimization

Entidades obtenidas en:

```text
READ_ONLY
```

podrán evitar:

```text
snapshot creation
dirty checking
ChangeSet allocation
```

---

# 279. Projection optimization

DTO/projection results:

```text
do not enter UnitOfWork
```

y por tanto:

```text
no Change Tracking
```

---

# 280. Streaming

Para streaming de muchas entidades:

```text
DETACHED_AFTER_YIELD
READ_ONLY
PROJECTION
```

reducen tracking memory.

---

# 281. Security boundary

Change Tracking no decide:

```text
who may change property
```

---

# 282. Sensitive field marker

Metadata puede declarar:

```text
sensitive = true
```

para:

```text
diagnostic redaction
telemetry redaction
audit policy
```

---

# 283. ChangeSet exposure

No deberá exponerse directamente a usuarios finales sin transformación/policy.

---

# 284. Password example

Nunca:

```text
oldPasswordHash = ...
newPasswordHash = ...
```

en Debug Toolbar de producción.

---

# 285. Deterministic redaction

Puede mostrar:

```text
[CHANGED]
```

sin valores.

---

# 286. Testing strategy

Se requieren pruebas para:

```text
Snapshot Tracking
Explicit Tracking
Notify Tracking
Custom Tracking
Scalar Types
Value Objects
Mutable Values
Relationships
Collections
Partial Entities
Unloaded Properties
Identifiers
Generated Fields
Version Fields
Lifecycle Stabilization
Runtime Isolation
Performance
Determinism
Extensions
```

---

# 287. Test: unchanged scalar

```text
Bob → Bob
```

produce:

```text
no change
```

---

# 288. Test: changed scalar

```text
Bob → Alice
```

produce `FieldChange`.

---

# 289. Test: null transition

```text
Alice → null
```

produce `VALUE_CLEARED`.

---

# 290. Test: null unchanged

```text
null → null
```

no produce change.

---

# 291. Test: unloaded ≠ null

Debe preservarse la distinción.

---

# 292. Test: uninitialized ≠ null

También.

---

# 293. Test: partial entity

Unloaded properties no deberán sobrescribirse.

---

# 294. Test: same-value mutation

```text
Alice → Bob → Alice
```

deberá terminar clean.

---

# 295. Test: notification noise

Múltiples notifications deberán normalizarse al estado final.

---

# 296. Test: explicit candidate clean

Una entidad marcada dirty pero restaurada al baseline produce empty ChangeSet.

---

# 297. Test: identifier mutation

Debe rechazarse.

---

# 298. Test: generated ID assignment

No deberá aparecer como user field mutation después de reconciliation.

---

# 299. Test: read-only mutation

Debe seguir policy.

---

# 300. Test: DB-generated property

No deberá persistirse como normal field si es DB authoritative.

---

# 301. Test: decimal equality

Debe respetar type semantics.

---

# 302. Test: UUID equality

Debe utilizar canonical representation.

---

# 303. Test: DateTime equality

Debe respetar precision/timezone policy.

---

# 304. Test: JSON object ordering

Debe respetar canonical JSON semantics.

---

# 305. Test: JSON array ordering

Debe preservar order semantics.

---

# 306. Test: mutable array

Una mutación in-place debe detectarse.

---

# 307. Test: mutable Value Object

Debe utilizar snapshot/comparator adecuado.

---

# 308. Test: immutable Value Object

Puede optimizar comparación cuando contract lo permita.

---

# 309. Test: relationship same EntityKey

No deberá generar cambio solo por representación distinta compatible.

---

# 310. Test: relationship different EntityKey

Debe generar RelationshipChange.

---

# 311. Test: unloaded relationship

No deberá disparar lazy load.

---

# 312. Test: unloaded collection

No deberá disparar query solo por dirty checking.

---

# 313. Test: embedded change

Debe producir path correcto.

---

# 314. Test: embedded cycle

Debe rechazarse cuando no esté soportado.

---

# 315. Test: lifecycle mutation

Debe recalcular ChangeSet.

---

# 316. Test: lifecycle convergence

Debe alcanzar stable ChangeSet.

---

# 317. Test: lifecycle oscillation

Debe fallar explícitamente.

---

# 318. Test: deterministic ordering

Mismos inputs producen mismo ChangeSet ordering/fingerprint.

---

# 319. Test: FrankenPHP isolation

Dirty candidates de Request A no aparecen en Request B.

---

# 320. Test: RoadRunner isolation

Misma garantía.

---

# 321. Test: OpenSwoole coroutine isolation

Misma garantía por scope.

---

# 322. Test: tenant isolation

No compartir tracking context.

---

# 323. Test: comparator side effects

Conformance test deberá rechazar/detectar comportamientos inválidos cuando sea posible.

---

# 324. Test: resource limit

No deberá truncar ChangeSet silenciosamente.

---

# 325. Test: large JSON

Debe respetar memory/performance budgets.

---

# 326. Test: read-only entities

No deberán generar tracking innecesario.

---

# 327. Test: projections

No deberán entrar al Change Tracker.

---

# 328. Anti-pattern: compare every PHP property

Incorrecto:

```php
foreach ((array) $entity as $property => $value) {
    // compare everything
}
```

El tracking debe ser metadata-driven.

---

# 329. Anti-pattern: serialize entity

Incorrecto:

```php
if (serialize($entity) !== $snapshot) {
    // dirty
}
```

---

# 330. Por qué

`serialize()` puede incluir:

```text
runtime state
lazy loaders
non-persistent properties
manager references
implementation details
```

---

# 331. Anti-pattern: PHP == universal comparator

Incorrecto:

```php
$old == $new
```

como semántica universal.

---

# 332. Anti-pattern: PHP === universal comparator

También incorrecto universalmente.

---

# 333. Anti-pattern: unloaded becomes null

Provoca:

```text
accidental data loss
```

---

# 334. Anti-pattern: deep compare entities

Incorrecto:

```text
Customer object graph
vs
Customer object graph
```

para decidir si cambió `Order.customer`.

Debe compararse identidad.

---

# 335. Anti-pattern: lazy-load during dirty check

Puede producir:

```text
unexpected queries
N+1
huge graph expansion
side effects during flush
```

---

# 336. Anti-pattern: all NEW fields changed from null

Incorrecto conceptualmente.

NEW tiene:

```text
initial persistence state
```

no un ficticio registro previo lleno de nulls.

---

# 337. Anti-pattern: dirty means UPDATE

Incorrecto:

```text
ChangeSet exists
→ direct UPDATE
```

Persistence Planner decide.

---

# 338. Anti-pattern: notify event = final delta

Incorrecto:

```text
Bob → Alice → Bob
```

no debe producir necesariamente UPDATE.

---

# 339. Anti-pattern: static dirty registry

Especialmente peligroso con FrankenPHP.

---

# 340. Anti-pattern: comparator queries DB

Dirty checking debe ser local y side-effect-free.

---

# 341. Anti-pattern: hash-only equality without contract

Puede introducir errores semánticos.

---

# 342. Anti-pattern: snapshots inside entity

No exigir:

```php
private array $__voltstackOriginal;
```

a todas las entidades.

---

# 343. Anti-pattern: authorization inside ChangeTracker

Incorrecto:

```php
if (!$currentUser->can('change-email')) ...
```

---

# 344. Anti-pattern: silent mutable aliasing

Snapshot y current value no deberán compartir mutable state cuando eso impida detectar cambios.

---

# 345. Anti-pattern: overwrite partial entity

Nunca:

```text
unloaded fields
→ default PHP values
→ UPDATE
```

---

# 346. Dependency architecture

Permitido:

```text
Change Tracking
    ├── Entity Metadata
    ├── Entity Mapping
    ├── Entity Model
    ├── Entity State
    ├── Snapshot contracts
    ├── Type System
    └── Relationship metadata
```

---

# 347. Prohibido

```text
Change Tracking → PDO
Change Tracking → SQL Compiler
Change Tracking → Driver
Change Tracking → concrete MySQL
Change Tracking → concrete PostgreSQL
Change Tracking → HTTP globals
Change Tracking → mutable global tenant
```

---

# 348. Integration with UnitOfWork

```text
UnitOfWork
    │
    ▼
Select Change Candidates
    │
    ▼
ChangeTracker
    │
    ▼
ChangeSetCollection
    │
    ▼
UnitOfWork Stabilizer
    │
    ▼
EntityChangeGraph
```

---

# 349. Integration with Snapshot System

```text
SnapshotRegistry
      │
      ▼
EntitySnapshot
      │
      ▼
ChangeTracker
      │
      ▼
ChangeSet
```

---

# 350. Integration with Persistence Engine

Indirecta:

```text
ChangeSet
    ↓
PreparedUnitOfWork
    ↓
Persistence Planner
    ↓
Persistence Plan
    ↓
Persistence Engine
```

---

# 351. Integration with Type System

```text
Field Mapping
    ↓
Database Type
    ↓
Normalizer / Comparator
    ↓
Canonical Comparison
```

---

# 352. Integration with Hydration

```text
Database Result
    ↓
Hydrator
    ↓
Entity
    ↓
Snapshot Establishment
    ↓
Managed Entity
```

No:

```text
hydration assignments
→ dirty changes
```

---

# 353. Integration with Relationships

```text
Entity Field
    ↓
Relationship Metadata
    ↓
EntityKey Comparison
    ↓
RelationshipChange
```

---

# 354. Integration with Lifecycle

```text
Initial ChangeSet
       ↓
preUpdate callbacks
       ↓
Affected State Mutation
       ↓
Re-detection
       ↓
Stable ChangeSet
```

---

# 355. Integration with Telemetry

Solo observacional:

```text
ChangeTracker
    ↓
Telemetry events/metrics
```

Telemetry nunca será necesaria para corrección semántica.

---

# 356. Invariantes arquitectónicas

## DB-ORM-CHANGE-001
Change Tracking determinará diferencias de estado persistente, no SQL.

## DB-ORM-CHANGE-002
Change Tracking será distinto de UnitOfWork.

## DB-ORM-CHANGE-003
Change Tracking será distinto de Snapshot System.

## DB-ORM-CHANGE-004
Change Tracking será distinto de Entity State.

## DB-ORM-CHANGE-005
Change Tracking será distinto de Persistence Planner.

## DB-ORM-CHANGE-006
Change Tracking será distinto de Transaction Manager.

## DB-ORM-CHANGE-007
ChangeSet será distinto de SQL.

## DB-ORM-CHANGE-008
ChangeSet será distinto de mutation history.

## DB-ORM-CHANGE-009
Dirty será una condición de cambio y no implicará UPDATE.

## DB-ORM-CHANGE-010
Solo propiedades persistentes participarán en dirty checking.

## DB-ORM-CHANGE-011
PersistentPropertyId será distinto de PHP property name cuando el mapping lo requiera.

## DB-ORM-CHANGE-012
PersistentPropertyId será distinto de DB column name.

## DB-ORM-CHANGE-013
Baseline representará estado conocido por el PersistenceContext.

## DB-ORM-CHANGE-014
Baseline no afirmará ser necesariamente el estado DB actual.

## DB-ORM-CHANGE-015
UNKNOWN será distinto de NULL.

## DB-ORM-CHANGE-016
UNLOADED será distinto de NULL.

## DB-ORM-CHANGE-017
UNINITIALIZED será distinto de NULL.

## DB-ORM-CHANGE-018
Los estados especiales no se representarán mediante null ambiguo.

## DB-ORM-CHANGE-019
ChangeSet estable será immutable.

## DB-ORM-CHANGE-020
FieldChange preservará original y current semantics.

## DB-ORM-CHANGE-021
Unknown original value no será inventado.

## DB-ORM-CHANGE-022
Empty ChangeSet solo afirmará ausencia de cambio local detectado.

## DB-ORM-CHANGE-023
Empty ChangeSet no afirmará ausencia de cambios externos.

## DB-ORM-CHANGE-024
SNAPSHOT será soportado.

## DB-ORM-CHANGE-025
EXPLICIT será soportado.

## DB-ORM-CHANGE-026
NOTIFY será soportado.

## DB-ORM-CHANGE-027
CUSTOM será extensible bajo contrato.

## DB-ORM-CHANGE-028
VoltStack no exigirá NOTIFY a todas las entidades.

## DB-ORM-CHANGE-029
POPO entities no requerirán base class para snapshot tracking.

## DB-ORM-CHANGE-030
Tracking strategy será metadata-driven.

## DB-ORM-CHANGE-031
ChangeTrackingPlan será compilable.

## DB-ORM-CHANGE-032
Compiled tracking plans serán immutable.

## DB-ORM-CHANGE-033
Compiled plans podrán compartirse entre requests.

## DB-ORM-CHANGE-034
Snapshots no se compartirán entre requests.

## DB-ORM-CHANGE-035
ChangeSets no se compartirán entre requests.

## DB-ORM-CHANGE-036
Dirty candidates no se compartirán entre requests.

## DB-ORM-CHANGE-037
Notification buffers no serán process-global.

## DB-ORM-CHANGE-038
Dirty checking evitará reflection metadata discovery repetitivo.

## DB-ORM-CHANGE-039
Entidades no requerirán getters públicos para tracking.

## DB-ORM-CHANGE-040
Private properties serán soportables.

## DB-ORM-CHANGE-041
Property accessor ORM será distinto de serializer.

## DB-ORM-CHANGE-042
Persistent comparison será type-aware.

## DB-ORM-CHANGE-043
PHP `==` no será comparator universal.

## DB-ORM-CHANGE-044
PHP `===` no será comparator universal.

## DB-ORM-CHANGE-045
ORM no creará un segundo type system incompatible.

## DB-ORM-CHANGE-046
Canonical comparison será distinta de driver binding cuando corresponda.

## DB-ORM-CHANGE-047
Database collation no definirá automáticamente domain equality.

## DB-ORM-CHANGE-048
DECIMAL no será reducido automáticamente a float.

## DB-ORM-CHANGE-049
Float approximate comparison no será default universal.

## DB-ORM-CHANGE-050
Date/time comparison respetará mapped precision semantics.

## DB-ORM-CHANGE-051
Timezone comparison será type-policy driven.

## DB-ORM-CHANGE-052
JSON object comparison podrá canonicalizar key order.

## DB-ORM-CHANGE-053
JSON array comparison preservará order semantics por defecto.

## DB-ORM-CHANGE-054
Mutable values requerirán baseline independiente.

## DB-ORM-CHANGE-055
Snapshot/current mutable aliasing no deberá ocultar cambios.

## DB-ORM-CHANGE-056
Mutable Value Objects requerirán estrategia explícita.

## DB-ORM-CHANGE-057
Unknown mutability no será asumida immutable.

## DB-ORM-CHANGE-058
Embedded Value Objects podrán rastrearse estructuralmente.

## DB-ORM-CHANGE-059
Embedded paths serán distintos de column names.

## DB-ORM-CHANGE-060
Unsupported embedded cycles serán rechazados.

## DB-ORM-CHANGE-061
Entity relationships no se compararán mediante deep object equality.

## DB-ORM-CHANGE-062
Entity relationship identity utilizará EntityKey/identifier semantics.

## DB-ORM-CHANGE-063
RelationshipChange será distinto de scalar FieldChange.

## DB-ORM-CHANGE-064
To-many collections tendrán análisis especializado.

## DB-ORM-CHANGE-065
Collection comparison no será simple PHP array comparison universal.

## DB-ORM-CHANGE-066
Dirty checking no disparará lazy loading oculto.

## DB-ORM-CHANGE-067
Unloaded relationships no se materializarán solo para comparación.

## DB-ORM-CHANGE-068
Unloaded collections no se materializarán solo para comparación.

## DB-ORM-CHANGE-069
Explicit assignment sobre unloaded state preservará unknown original.

## DB-ORM-CHANGE-070
Partial entities preservarán loaded-state information.

## DB-ORM-CHANGE-071
Partial entity tracking no sobrescribirá unloaded properties.

## DB-ORM-CHANGE-072
Typed uninitialized properties serán detectadas explícitamente.

## DB-ORM-CHANGE-073
Uninitialized será distinto de unloaded.

## DB-ORM-CHANGE-074
Established entity identifiers serán immutable por defecto.

## DB-ORM-CHANGE-075
Identifier mutation no será un normal UPDATE ChangeSet.

## DB-ORM-CHANGE-076
Composite identifier parts serán type-compared.

## DB-ORM-CHANGE-077
Generated ID assignment no será domain mutation.

## DB-ORM-CHANGE-078
Generated fields tendrán política explícita.

## DB-ORM-CHANGE-079
DB-authoritative fields no serán writable accidentalmente.

## DB-ORM-CHANGE-080
Computed fields podrán ser read-only.

## DB-ORM-CHANGE-081
Read-only persistent property mutation será gobernada por policy.

## DB-ORM-CHANGE-082
Optimistic version advancement será distinto de domain mutation.

## DB-ORM-CHANGE-083
Change origin podrá distinguir DOMAIN/ORM/DATABASE.

## DB-ORM-CHANGE-084
Hydration no generará dirty state.

## DB-ORM-CHANGE-085
Hydration establecerá baseline según lifecycle contract.

## DB-ORM-CHANGE-086
Normalizers no mutarán entidades.

## DB-ORM-CHANGE-087
Comparison values serán distintos de driver binding values.

## DB-ORM-CHANGE-088
Invalid persistent values producirán error tipado.

## DB-ORM-CHANGE-089
Change Tracking no reemplazará business validation.

## DB-ORM-CHANGE-090
NEW entity no será modelada como todos los campos cambiando desde null.

## DB-ORM-CHANGE-091
NEW persistence input será distinto de managed ChangeSet cuando convenga.

## DB-ORM-CHANGE-092
REMOVED entity no requerirá dirty check para justificar deletion.

## DB-ORM-CHANGE-093
DETACHED entities no serán dirty-checked por un UnitOfWork ajeno.

## DB-ORM-CHANGE-094
VoltStack V1 no implementará implicit detached merge.

## DB-ORM-CHANGE-095
NOTIFY strategy será scoped.

## DB-ORM-CHANGE-096
Notification history será distinto de final ChangeSet.

## DB-ORM-CHANGE-097
Final state podrá cancelar intermediate mutations.

## DB-ORM-CHANGE-098
Dirty candidate será distinto de confirmed dirty entity.

## DB-ORM-CHANGE-099
Property-level dirty hints no reemplazarán semántica final.

## DB-ORM-CHANGE-100
Tracking optimizations no cambiarán ChangeSet semantics.

## DB-ORM-CHANGE-101
Cada flush podrá utilizar una ChangeDetectionSession independiente.

## DB-ORM-CHANGE-102
Session caches no se reutilizarán de forma stale.

## DB-ORM-CHANGE-103
ChangeSet podrá tener deterministic fingerprint.

## DB-ORM-CHANGE-104
ChangeSet fingerprint no dependerá de memory address.

## DB-ORM-CHANGE-105
ChangeSet ordering será determinista.

## DB-ORM-CHANGE-106
Reflection order no será autoridad accidental.

## DB-ORM-CHANGE-107
Mismos inputs semánticos producirán mismo ChangeSet.

## DB-ORM-CHANGE-108
PersistenceContext mutable pertenecerá a un único flujo lógico.

## DB-ORM-CHANGE-109
ChangeTracker no será thread-safe por defecto.

## DB-ORM-CHANGE-110
ChangeSet utilizado por frozen flush será immutable.

## DB-ORM-CHANGE-111
Lifecycle mutations antes de freeze requerirán redetection.

## DB-ORM-CHANGE-112
Lifecycle stabilization deberá converger.

## DB-ORM-CHANGE-113
Non-convergent tracking será error.

## DB-ORM-CHANGE-114
Oscillation podrá detectarse mediante fingerprints.

## DB-ORM-CHANGE-115
Field y relationship stabilization se coordinarán mediante UnitOfWork.

## DB-ORM-CHANGE-116
Identifier mutation será validada durante detection.

## DB-ORM-CHANGE-117
Read-only mutation será validada durante detection.

## DB-ORM-CHANGE-118
Generated-field mutation será validada durante detection.

## DB-ORM-CHANGE-119
Unknown baseline será gobernado explícitamente.

## DB-ORM-CHANGE-120
Tracking completeness será first-class.

## DB-ORM-CHANGE-121
PARTIAL será distinto de COMPLETE.

## DB-ORM-CHANGE-122
UNKNOWN completeness será distinto de COMPLETE.

## DB-ORM-CHANGE-123
Partial empty ChangeSet no implicará complete no-op.

## DB-ORM-CHANGE-124
Persistence Planner podrá consumir completeness.

## DB-ORM-CHANGE-125
Change Tracking no evitará por sí mismo lost updates.

## DB-ORM-CHANGE-126
Change Tracking será distinto de concurrency control.

## DB-ORM-CHANGE-127
Optimistic locking podrá consumir original version.

## DB-ORM-CHANGE-128
Original-value predicates pertenecerán a persistence/concurrency planning.

## DB-ORM-CHANGE-129
UNASSIGNED será distinto de ASSIGNED_NULL.

## DB-ORM-CHANGE-130
Database default handling será explícito.

## DB-ORM-CHANGE-131
ChangeSet podrá alimentar audit integrations sin ser audit log.

## DB-ORM-CHANGE-132
Sensitive values serán redactables.

## DB-ORM-CHANGE-133
Telemetry no cambiará tracking semantics.

## DB-ORM-CHANGE-134
Tracking performance será medible.

## DB-ORM-CHANGE-135
Compiled plans reducirán runtime reflection.

## DB-ORM-CHANGE-136
Comparator registry será frozen en runtime normal.

## DB-ORM-CHANGE-137
Comparator collisions serán deterministas.

## DB-ORM-CHANGE-138
Custom strategies serán tipadas.

## DB-ORM-CHANGE-139
Custom strategies no ejecutarán SQL ocultamente.

## DB-ORM-CHANGE-140
Custom strategies no cambiarán tenant context.

## DB-ORM-CHANGE-141
Custom strategies no ocultarán unknown baselines.

## DB-ORM-CHANGE-142
Custom comparators serán side-effect-free.

## DB-ORM-CHANGE-143
Comparators no consultarán servicios externos para igualdad.

## DB-ORM-CHANGE-144
Comparators serán deterministas.

## DB-ORM-CHANGE-145
DirtyCandidateRegistry será scoped.

## DB-ORM-CHANGE-146
ComparisonCache será scoped al detection cycle.

## DB-ORM-CHANGE-147
FrankenPHP requests no compartirán mutable tracking state.

## DB-ORM-CHANGE-148
RoadRunner requests no compartirán mutable tracking state.

## DB-ORM-CHANGE-149
OpenSwoole execution scopes no compartirán tracking state accidentalmente.

## DB-ORM-CHANGE-150
Tenant contexts no compartirán tracking state.

## DB-ORM-CHANGE-151
Shard contexts no compartirán tracking state incompatible.

## DB-ORM-CHANGE-152
Change Tracking core no hardcodeará `tenant_id`.

## DB-ORM-CHANGE-153
ChangeSet diagnostics serán explainable.

## DB-ORM-CHANGE-154
Sensitive diagnostics respetarán redaction policy.

## DB-ORM-CHANGE-155
Resource limits no truncarán silenciosamente detección.

## DB-ORM-CHANGE-156
Large values podrán utilizar optimizaciones contractualmente seguras.

## DB-ORM-CHANGE-157
Hash optimization no reemplazará exact equality sin garantía.

## DB-ORM-CHANGE-158
Read-only entities podrán evitar tracking innecesario.

## DB-ORM-CHANGE-159
DTO/projection results no entrarán al Change Tracker.

## DB-ORM-CHANGE-160
Change Tracking será una transformación semántica local, tipada, determinista y side-effect-free del baseline conocido y el estado persistente actual hacia un ChangeSet.

---

# 357. Fórmula fundamental

```text
ChangeSet(E)
=
Compare(
    PersistentBaseline(E),
    CurrentPersistentState(E),
    CompiledTrackingPlan(E)
)
```

---

# 358. Dirty formula

```text
Dirty(E)
⇔
ChangeSet(E) ≠ ∅
```

para una entidad managed bajo una estrategia de comparación completa.

---

# 359. Typed equality

```text
Changed(P)
=
¬ Comparator(Type(P)).equals(
    Normalize(Original(P)),
    Normalize(Current(P))
)
```

---

# 360. Partial baseline formula

```text
Original(P) = UNLOADED
⇒
DoNotAssume(NULL)
```

---

# 361. Relationship formula

```text
RelationshipChanged(R)
=
CanonicalEntityIdentity(Original(R))
≠
CanonicalEntityIdentity(Current(R))
```

cuando ambas identidades sean conocidas.

---

# 362. Embedded formula

```text
EmbeddedChanged(V)
=
∃ persistent member p ∈ V :
Changed(p)
```

para member-based embedded tracking.

---

# 363. Notification formula

```text
NotifyCandidates(E)
→
FinalTypedComparison(E)
→
FinalChangeSet(E)
```

de modo que:

```text
MutationHistory
≠
FinalChangeSet
```

---

# 364. Safe partial tracking

```text
SafePartialChangeTracking
=
KnownLoadedState
∧
NoUnloadedValueFabrication
∧
ExplicitUnknownRepresentation
∧
TypedChangedFields
∧
PersistencePlannerAwareOfCompleteness
```

---

# 365. Safe mutable value tracking

```text
SafeMutableTracking(V)
=
IndependentBaseline(V)
∧
DeterministicComparator(V)
∧
NoSnapshotAliasing(V)
```

---

# 366. Safe persistent runtime

```text
SafeChangeTrackingRuntime
=
ImmutableSharedPlans
∧
ImmutableSharedComparators
∧
ScopedSnapshots
∧
ScopedDirtyCandidates
∧
ScopedNotificationBuffers
∧
ScopedChangeSets
∧
ScopedComparisonCaches
∧
DeterministicReset
```

---

# 367. Performance formula

```text
ChangeTrackingCost
≈
CandidateEntitySelection
+
Σ PropertyExtraction
+
Σ ValueNormalization
+
Σ TypedComparison
+
ChangeSetAllocation
+
StabilizationCost
```

---

# 368. Optimization rule

```text
OptimizationAllowed
⇔
ResultingChangeSetSemanticsRemainEquivalent
```

---

# 369. Master Formula

```text
Database Change Tracking System
=
Metadata-Driven Persistent Property Selection
+
Compiled Tracking Plans
+
Baseline Resolution
+
Explicit Value States
+
Snapshot Comparison
+
Explicit Dirty Candidates
+
Notification Tracking
+
Typed Value Normalization
+
Typed Equality
+
Mutable Value Handling
+
Embedded Value Tracking
+
Relationship Identity Comparison
+
Collection Tracking Integration
+
Partial Entity Awareness
+
Loaded-State Awareness
+
Identifier Mutation Protection
+
Generated Field Governance
+
Version Field Integration
+
Lifecycle Stabilization
+
ChangeSet Construction
+
ChangeSet Completeness
+
Deterministic Fingerprinting
+
Runtime Isolation
+
Resource Governance
+
Telemetry
+
Diagnostics
+
Extension Governance
```

---

# 370. Master Rule

> **VoltStack determina cambios ORM comparando únicamente el estado persistente relevante de una entidad contra un baseline explícitamente conocido, mediante metadata compilada y semántica de tipos; nunca convierte valores no cargados en `null`, nunca dispara lazy loading para descubrir cambios y nunca confunde un ChangeSet con la operación SQL que eventualmente podría persistirlo.**

---

# 371. Resultado arquitectónico

Después de este sistema, VoltStack dispone de:

```text
Managed Entity
      │
      ├── Persistent Metadata
      │
      ├── Tracking Plan
      │
      └── Original Baseline
              │
              ▼
       Change Tracking
              │
              ▼
         Typed ChangeSet
              │
              ▼
          UnitOfWork
              │
              ▼
      Entity Change Graph
```

Ahora falta formalizar de manera independiente la fuente fundamental del baseline:

```text
Entity Snapshot
```

---

# 372. Relación con el siguiente documento

```text
124 UnitOfWork Architecture
          │
          ▼
125 Change Tracking System
          │
          ▼
126 Entity Snapshot System
          │
          ▼
127 Persistence Engine
          │
          ▼
128 Persistence Planner
```

---

# 373. Siguiente documento

```text
126_DATABASE_ENTITY_SNAPSHOT_SYSTEM.md
```

El siguiente documento deberá formalizar:

```text
Entity Snapshot architecture
snapshot identity
snapshot lifecycle
snapshot creation
snapshot baseline semantics
snapshot completeness
full snapshots
partial snapshots
field snapshots
embedded snapshots
relationship snapshots
collection snapshot boundaries
typed snapshot values
canonical snapshot representation
mutable value copying
copy strategies
structural snapshots
snapshot immutability
snapshot fingerprints
snapshot versioning
snapshot replacement
snapshot advancement after flush
snapshot preservation after failure
snapshot behavior after transaction rollback
generated identifier reconciliation
database-generated fields
optimistic version snapshots
read-only entities
partial entities
lazy/unloaded properties
snapshot memory optimization
large-value snapshot strategies
snapshot registry
IdentityMap integration
UnitOfWork integration
Hydration integration
Change Tracking integration
persistent runtime isolation
resource governance
telemetry
diagnostics
extension contracts
```

con la regla central:

> **Un Entity Snapshot representa el baseline persistente conocido de una entidad dentro de un PersistenceContext; no representa necesariamente el estado actual de la base de datos y nunca deberá compartir estado mutable de forma que una mutación de la entidad pueda modificar retroactivamente su propio baseline.**