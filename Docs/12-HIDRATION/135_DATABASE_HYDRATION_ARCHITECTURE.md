# 135_DATABASE_HYDRATION_ARCHITECTURE.md

# VoltStack Quantum Database
## Database Hydration Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 135 — Database Hydration Architecture  
**Bloque:** 12 — Hydration  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Hydration Architecture` define la arquitectura mediante la cual VoltStack transforma resultados tipados provenientes del `Execution Engine` en representaciones PHP utilizables por las capas superiores.

La hidratación constituye el camino conceptual inverso a la persistencia:

```text
Persistence

Entity
  ↓
ChangeSet
  ↓
PersistencePlan
  ↓
Query Model
  ↓
Execution
  ↓
Database


Hydration

Database
  ↓
Execution
  ↓
Result
  ↓
HydrationPlan
  ↓
Value Conversion
  ↓
Identity Resolution
  ↓
Entity / Projection / Scalar / Tuple / DTO
```

La hidratación deberá soportar:

- entidades;
- resultados escalares;
- proyecciones;
- tuples;
- DTOs;
- relaciones;
- resultados parciales;
- streaming;
- JOINs;
- aliases;
- conversiones de tipos;
- Identity Map;
- snapshots;
- lifecycle;
- proxies/references;
- consultas read-only;
- datasets grandes.

---

# 2. Principio central

> **El Hydration System convierte un resultado estructurado en una representación PHP según un `HydrationPlan`; no ejecuta consultas, no genera SQL, no decide qué consultar y no constituye un segundo motor ORM.**

Por tanto:

```text
Hydration
≠
Query Execution
```

```text
Hydration
≠
Persistence
```

```text
Hydration
≠
Entity Mapping
```

```text
Hydration
≠
Identity Map
```

```text
Hydration
≠
Serialization
```

---

# 3. Posición arquitectónica

```text
                    Query / Entity Query
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
                     Result System
                           │
                           ▼
                 ┌───────────────────┐
                 │ Hydration System  │
                 └─────────┬─────────┘
                           │
             ┌─────────────┼──────────────┐
             │             │              │
             ▼             ▼              ▼
          Scalar       Projection        Entity
                                            │
                                ┌───────────┼───────────┐
                                ▼           ▼           ▼
                           IdentityMap   Snapshot      UoW
                                │
                                ▼
                            Lifecycle
                                │
                                ▼
                           Application
```

---

# 4. Regla de dependencias

La dirección será:

```text
ORM
 ↓
Hydration
 ↓
Result System
```

y, para hidratación de entidades:

```text
Entity Hydration
 ├── Entity Metadata
 ├── Type System
 ├── IdentityMap
 ├── EntityState
 ├── Snapshot System
 └── Lifecycle System
```

Pero nunca:

```text
Hydrator
→ SQL Compiler
```

ni:

```text
Hydrator
→ Driver
```

ni:

```text
Hydrator
→ raw PDO
```

---

# 5. Objetivos

La arquitectura deberá proporcionar:

1. un pipeline único de hidratación;
2. múltiples estrategias de resultado;
3. planes compilables;
4. hidratación determinista;
5. integración segura con IdentityMap;
6. deduplicación de entidades;
7. soporte para grafos relacionales;
8. conversiones de tipos;
9. snapshots correctos;
10. prevención de dirty tracking artificial;
11. lifecycle `postLoad`;
12. streaming;
13. soporte para partial entities;
14. DTO/projection hydration;
15. rendimiento predecible;
16. bajo overhead;
17. extensibilidad;
18. aislamiento en runtimes persistentes.

---

# 6. No objetivos

Hydration System no será responsable de:

```text
query building
query optimization
SQL generation
statement execution
transaction management
schema discovery
migration execution
entity persistence
authorization
serialization for HTTP
validation
business logic
```

---

# 7. Hydration ≠ Construction

Construir un objeto es solo una operación posible dentro de la hidratación.

```text
ObjectConstruction
⊂
Hydration
```

Una hidratación completa puede requerir:

```text
resolve identity
reuse existing object
construct object
convert values
assign fields
assemble relationships
capture snapshot
register state
dispatch lifecycle
```

---

# 8. Hydration ≠ Serialization

Serialization normalmente transforma:

```text
PHP Object
→
external representation
```

Hydration transforma:

```text
database result
→
PHP representation
```

No deben compartir contratos accidentalmente.

---

# 9. Hydration ≠ Mapping

Mapping responde:

```text
¿Qué significa este campo?
```

Hydration responde:

```text
¿Cómo materializo este resultado usando ese mapping?
```

Por tanto:

```text
EntityMetadata
→ input to HydrationPlan
```

---

# 10. Hydration ≠ Type Conversion

La conversión de tipos es un servicio utilizado por hidratación.

Ejemplo:

```text
"2026-09-06 14:30:00"
       ↓
DateTimeType
       ↓
DateTimeImmutable
```

El Hydrator coordina la conversión; no redefine el Type System.

---

# 11. Hydration ≠ Identity Resolution

El Hydrator solicita:

```text
EntityKey
→ IdentityMap
```

pero el IdentityMap conserva la garantía de canonicalidad.

---

# 12. Hydration ≠ Entity State

El Hydrator puede provocar que una entidad recién cargada termine como:

```text
MANAGED
```

pero no es propietario del modelo completo de estados.

---

# 13. Hydration ≠ Change Tracking

Asignar datos provenientes de DB no deberá producir artificialmente:

```text
DIRTY
```

La inicialización y el baseline deberán coordinarse con Snapshot/Change Tracking.

---

# 14. Hydration Modes

VoltStack definirá modos explícitos.

```php
enum HydrationMode
{
    case ENTITY;
    case SCALAR;
    case PROJECTION;
    case TUPLE;
    case DTO;
    case PARTIAL_ENTITY;
    case ARRAY;
    case CUSTOM;
}
```

---

# 15. ENTITY

Ejemplo:

```php
$users = User::query()
    ->where('active', true)
    ->get();
```

Resultado:

```text
Result Rows
   ↓
User entities
```

con integración ORM completa.

---

# 16. SCALAR

Ejemplo conceptual:

```php
$count = User::query()
    ->count();
```

Resultado:

```text
"150"
 ↓
int
 ↓
150
```

---

# 17. PROJECTION

```php
$result = User::query()
    ->select('id', 'email')
    ->get();
```

Puede producir una representación explícita que no pretenda ser una entidad completa.

---

# 18. TUPLE

Ejemplo:

```text
[
    User,
    OrderCount,
    LastOrderDate
]
```

Debe existir metadata que describa cada slot.

---

# 19. DTO

Ejemplo:

```php
final readonly class UserSummary
{
    public function __construct(
        public int $id,
        public string $name,
        public int $orders,
    ) {}
}
```

Hydration podrá materializar directamente:

```text
UserSummary
```

sin introducirlo al IdentityMap.

---

# 20. PARTIAL_ENTITY

Representa una entidad cuya información persistente está incompleta.

Será una modalidad explícita y restringida.

Nunca:

```text
missing field
=
null
```

automáticamente.

---

# 21. ARRAY

Modo estructural para APIs low-level donde el usuario desea valores convertidos sin semántica de entidad.

---

# 22. CUSTOM

Extensiones podrán registrar estrategias propias mediante contratos explícitos.

---

# 23. Arquitectura del pipeline

```text
ExecutionResult
      │
      ▼
Result Metadata
      │
      ▼
Hydration Request
      │
      ▼
Hydration Plan Resolver
      │
      ▼
Compiled Hydration Plan
      │
      ▼
Row Reader
      │
      ▼
Column Resolution
      │
      ▼
Type Conversion
      │
      ▼
Identity Extraction
      │
      ▼
Identity Resolution
      │
      ├──────── IdentityMap HIT
      │                  │
      │                  ▼
      │          canonical entity
      │
      └──────── IdentityMap MISS
                         │
                         ▼
                 entity construction
                         │
                         ▼
                    field hydration
                         │
                         ▼
                relationship assembly
                         │
                         ▼
                   ORM registration
                         │
                         ▼
                    snapshot baseline
                         │
                         ▼
                       postLoad
                         │
                         ▼
                    hydrated result
```

---

# 24. Hydration Request

La operación deberá comenzar mediante un objeto explícito.

```php
final readonly class HydrationRequest
{
    public function __construct(
        public HydrationMode $mode,
        public ResultInterface $result,
        public HydrationPlan $plan,
        public HydrationContext $context,
    ) {}
}
```

---

# 25. HydrationContext

Contendrá únicamente información scoped necesaria.

Ejemplo:

```php
final readonly class HydrationContext
{
    public function __construct(
        public PersistenceContextId $persistenceContext,
        public HydrationScopeId $scope,
        public HydrationPolicy $policy,
        public bool $readOnly = false,
    ) {}
}
```

---

# 26. Hydration Plan

`HydrationPlan` será la descripción ejecutable de cómo interpretar un Result.

```text
Result Shape
+
Query Metadata
+
Entity Metadata
+
Type Metadata
+
Alias Metadata
+
Relationship Metadata
+
Hydration Mode
      ↓
HydrationPlan
```

---

# 27. HydrationPlan ≠ QueryPlan

Son conceptos diferentes.

```text
QueryPlan
=
how query will be executed
```

```text
HydrationPlan
=
how result will be materialized
```

---

# 28. HydrationPlan ≠ Result

El plan describe.

El Result contiene datos.

---

# 29. Hydration Plan ejemplo

```text
Entity: User

Identifier:
    result column u_id
    → User.id
    → int

Fields:
    u_name
    → User.name
    → string

    u_email
    → User.email
    → EmailAddress

Relationship:
    o_id...
    → User.orders[]
    → Order
```

---

# 30. Compiled Hydration Plan

El hot path no deberá inspeccionar repetidamente:

```text
Reflection
Attributes
EntityMetadata
Query AST
```

por cada row.

Se generará:

```text
CompiledHydrationPlan
```

---

# 31. Compilation

```text
Hydration Metadata
      ↓
Plan Builder
      ↓
Plan Validation
      ↓
Plan Optimization
      ↓
CompiledHydrationPlan
```

---

# 32. Plan immutability

Una vez compilado:

```text
CompiledHydrationPlan = immutable
```

y podrá reutilizarse cuando el contexto estructural sea compatible.

---

# 33. Plan cache

Puede existir:

```text
HydrationPlanCache
```

para planes inmutables.

No debe almacenar:

```text
entities
EntityManager
tenant mutable state
request
Result
```

---

# 34. Cache key

Conceptualmente:

```text
HydrationPlanCacheKey
=
QueryResultShape
+
HydrationMode
+
MetadataGeneration
+
TypeSystemGeneration
+
PlatformRelevantShape
```

---

# 35. Tenant consideration

El tenant no deberá introducirse en el cache key si el plan es estructuralmente idéntico.

Si cambia mapping/schema efectivo, deberá existir un discriminator estructural adecuado.

---

# 36. Result Metadata

Hydration consume metadata de Result.

Ejemplo:

```text
column alias
column index
logical type
nullable
source expression
entity field binding
```

---

# 37. Positional vs named access

El plan compilado puede preferir:

```text
column position
```

para rendimiento.

Pero la semántica deberá derivarse de aliases/metadatos estables, no de suposiciones accidentales.

---

# 38. Row Reader

Contrato conceptual:

```php
interface HydrationRowReader
{
    public function read(ResultRow $row, CompiledHydrationPlan $plan): HydrationRow;
}
```

---

# 39. Row Reader responsibilities

Debe:

- obtener valores;
- respetar posiciones;
- detectar columnas ausentes;
- distinguir NULL;
- producir errores diagnósticos.

No debe:

- crear entidades;
- ejecutar queries;
- persistir objetos.

---

# 40. Missing ≠ NULL

Regla crítica:

```text
COLUMN_NOT_PRESENT
≠
SQL_NULL
```

Especialmente para:

```text
partial entities
LEFT JOIN
projections
```

---

# 41. Value Presence

Puede modelarse:

```php
enum HydrationValuePresence
{
    case PRESENT;
    case NULL;
    case MISSING;
}
```

---

# 42. Type conversion pipeline

```text
Database Value
      ↓
Driver Normalization
      ↓
Database Type Descriptor
      ↓
VoltStack Type
      ↓
PHP Value
```

---

# 43. Example

```text
PostgreSQL UUID string
      ↓
UuidType
      ↓
UserId
```

---

# 44. Conversion error

Debe producir un error contextual:

```text
entity: User
field: id
source column: u_id
expected type: UserId
database value type: string
```

sin revelar valores sensibles por defecto.

---

# 45. Entity Hydration Pipeline

La hidratación de entidad seguirá conceptualmente:

```text
Row
 ↓
Identifier Extraction
 ↓
Identifier Conversion
 ↓
EntityKey Construction
 ↓
IdentityMap Resolution
 ↓
┌─────────────────────┐
│ HIT                 │ MISS
│                     │
▼                     ▼
Reuse               Reserve Identity
Entity                 ↓
                    Construct
                       ↓
                    Populate
                       ↓
                    Register
└───────────┬───────────┘
            ▼
     Relationship Assembly
            ↓
       Snapshot Baseline
            ↓
      EntityState MANAGED
            ↓
          postLoad
```

---

# 46. Identifier first

Cuando sea posible, el identificador deberá procesarse antes del resto de campos.

Razón:

```text
EntityKey
```

determina si debe crearse un objeto.

---

# 47. IdentityMap hit

Si:

```text
IdentityMap
User#42
→ Object A
```

y otra row representa `User#42`:

```text
Hydrator
→ Object A
```

No:

```text
new User()
```

---

# 48. IdentityMap hit ≠ overwrite

Una entidad ya managed puede contener cambios locales.

Por tanto:

```text
IdentityMap HIT
```

no autoriza automáticamente:

```text
overwrite all fields from row
```

---

# 49. Existing Entity Hydration Policy

Se definirá:

```php
enum ExistingEntityHydrationPolicy
{
    case REUSE_ONLY;
    case INITIALIZE_MISSING;
    case REFRESH;
    case REJECT_CONFLICT;
}
```

---

# 50. Default normal query

Para una entidad completamente managed:

```text
REUSE_ONLY
```

deberá evitar sobrescribir cambios locales.

---

# 51. Explicit refresh

`EntityManager::refresh()` podrá usar:

```text
REFRESH
```

bajo las reglas de `DirtyRefreshPolicy`.

---

# 52. IdentityMap miss

En un miss:

```text
EntityKey
→ no canonical instance
```

se inicia materialización.

---

# 53. Hydration reservation

Para evitar duplicados durante grafos circulares:

```text
reserve EntityKey
```

antes de completar toda la entidad.

---

# 54. Why reservation

Ejemplo:

```text
User
 └── Profile
      └── User
```

Sin reserva:

```text
hydrate User A
  ↓
hydrate Profile
  ↓
hydrate User again
  ↓
create User B
```

violando canonicalidad.

---

# 55. Two-phase identity registration

Propuesta:

```text
RESERVED
   ↓
HYDRATING
   ↓
ACTIVE
```

Estos son estados internos de materialización.

No sustituyen:

```text
EntityState
```

---

# 56. Reserved entity visibility

Una entidad parcialmente hidratada no deberá escapar a APIs normales.

Solo subsistemas internos autorizados podrán resolverla para cerrar ciclos.

---

# 57. Hydration failure

Si falla antes de activación:

```text
reservation
→ abort
```

y deberán limpiarse:

```text
forward identity reservation
reverse object registration
temporary relationship links
temporary hydration state
```

---

# 58. Entity Construction

VoltStack no deberá requerir que todas las entidades tengan:

```php
public function __construct() {}
```

---

# 59. Construction Strategy

```php
interface EntityConstructionStrategy
{
    public function construct(
        EntityMetadata $metadata,
        EntityConstructionContext $context
    ): object;
}
```

---

# 60. Strategies

Podrán existir:

```text
CONSTRUCTOR
BYPASS_CONSTRUCTOR
FACTORY
CUSTOM
```

---

# 61. Constructor strategy

Adecuada cuando el query contiene todos los argumentos necesarios.

---

# 62. Constructor bypass

Puede ser necesario para Data Mapper.

Pero deberá estar:

- controlado;
- compilado;
- validado;
- compatible con propiedades PHP;
- explícitamente soportado por metadata.

---

# 63. No arbitrary constructor execution

Hydration no deberá ejecutar constructores de dominio accidentalmente cuando esos constructores representen:

```text
new domain object creation
```

en lugar de reconstitución.

---

# 64. Reconstitution

VoltStack distinguirá conceptualmente:

```text
Creation
```

de:

```text
Reconstitution
```

---

# 65. Field Assignment

La escritura podrá usar estrategias compiladas.

```text
PropertyAccess
Setter
ConstructorArgument
ReflectionAccessor
GeneratedAccessor
CustomAccessor
```

---

# 66. Setter caution

Un setter puede:

- validar;
- mutar otros campos;
- emitir domain events;
- modificar timestamps;
- ejecutar lógica.

Por tanto no deberá utilizarse implícitamente como mecanismo universal de hydration.

---

# 67. Field Access Metadata

Mapping definirá:

```text
read strategy
write strategy
hydration strategy
```

separadamente cuando sea necesario.

---

# 68. Readonly properties

PHP readonly requiere tratamiento explícito.

La arquitectura deberá validar si:

- puede inicializarse durante construcción;
- puede inicializarse mediante estrategia soportada;
- requiere constructor hydration;
- es incompatible.

---

# 69. Typed properties

El Hydrator nunca deberá introducir un valor incompatible deliberadamente para "arreglarlo después".

---

# 70. Uninitialized properties

Podrán ser válidas temporalmente durante construcción interna.

No deberán escapar al usuario si metadata declara que son obligatorias.

---

# 71. Hydration completeness

Antes de activar la entidad:

```text
RequiredPersistentFieldsInitialized
=
true
```

salvo `PARTIAL_ENTITY`.

---

# 72. Full Entity Hydration

Una entidad completa deberá poseer todos los campos persistentes requeridos por su mapping, salvo valores:

- lazy;
- deferred;
- virtual;
- generated-later;
- explícitamente no seleccionados bajo una estrategia soportada.

---

# 73. Partial Entity

Partial entity será explícita:

```text
User {
    id: loaded
    name: loaded
    email: MISSING
}
```

No:

```text
email = null
```

---

# 74. Partial state metadata

La entidad no necesita guardar flags ORM.

El PersistenceContext puede mantener:

```text
LoadedFieldSet
```

externamente.

---

# 75. LoadedFieldSet

```php
final readonly class LoadedFieldSet
{
    // persistent field identifiers
}
```

---

# 76. Partial entity persistence

Por defecto:

```text
partial entity
→ restricted persistence
```

para evitar:

```text
MISSING
→ NULL
→ accidental UPDATE
```

---

# 77. Recommended V1

Las proyecciones deberán preferirse sobre partial entities.

```text
Projection > PartialEntity
```

para lecturas incompletas.

---

# 78. DTO hydration

DTO:

```text
does not enter IdentityMap
does not become MANAGED
does not receive EntityState
does not receive ORM Snapshot
```

---

# 79. Projection hydration

Misma regla, salvo que la projection sea explícitamente un wrapper sobre una entidad managed.

---

# 80. Scalar hydration

Debe ser extremadamente liviana.

Pipeline:

```text
Result Value
 ↓
Type Conversion
 ↓
PHP Scalar / Value Object
```

No necesita:

```text
IdentityMap
UoW
Lifecycle
Snapshot
```

---

# 81. Tuple hydration

Cada slot tendrá descriptor.

```php
final readonly class TupleSlot
{
    public function __construct(
        public string $alias,
        public HydrationValuePlan $plan,
    ) {}
}
```

---

# 82. Relationship Hydration

Los JOINs producen un problema importante:

```text
User 1 + Order 10
User 1 + Order 11
User 1 + Order 12
```

El resultado contiene tres rows.

El objeto debe ser:

```text
User#1
 └── orders
      ├── Order#10
      ├── Order#11
      └── Order#12
```

No tres usuarios.

---

# 83. Root Deduplication

La entidad raíz se deduplicará por:

```text
EntityKey
```

---

# 84. Child Deduplication

Las relaciones to-many también deberán evitar:

```text
Order#10
Order#10
```

por JOIN multiplicativo.

---

# 85. Collection Assembly

El Hydrator podrá utilizar:

```text
RelationshipAssemblyState
```

scoped al proceso de hidratación.

---

# 86. RelationshipAssemblyKey

Conceptualmente:

```text
OwnerEntityKey
+
RelationshipId
+
ChildEntityKey
```

---

# 87. No collection duplicates

Para relaciones set-like:

```text
same child identity
```

no se añadirá repetidamente.

---

# 88. Bag semantics

Si una mapping declara semántica bag/multiset, la política podrá diferir.

La metadata decide.

---

# 89. LEFT JOIN

Ejemplo:

```text
User row exists
Order columns = NULL
```

No deberá crear:

```text
Order(null)
```

---

# 90. Null joined identity

Si todos los componentes identificadores relevantes del joined entity indican ausencia:

```text
related entity = absent
```

---

# 91. Composite identifiers

La ausencia deberá determinarse según reglas del identificador compuesto.

No simplemente:

```text
first column is null
```

si el mapping permite otra semántica.

---

# 92. Relationship fix-up

Al hidratar:

```text
User.orders → Order
```

podrá requerirse:

```text
Order.user → User
```

si la relación bidireccional lo exige.

---

# 93. Fix-up ≠ domain mutation

El Relationship Hydrator deberá diferenciar inicialización ORM de una mutación realizada por la aplicación.

---

# 94. No artificial dirty state

Relationship fix-up inicial:

```text
must not mark relationship dirty
```

---

# 95. Relationship baseline

Después de completar la materialización:

```text
RelationshipSnapshot
```

deberá representar la colección cargada.

---

# 96. Incomplete collections

Una colección puede ser:

```text
UNINITIALIZED
PARTIALLY_INITIALIZED
INITIALIZED
```

---

# 97. JOIN-filtered collection

Ejemplo:

```sql
... JOIN orders
WHERE orders.status = 'open'
```

No necesariamente significa:

```text
User.orders contains all orders
```

---

# 98. Critical rule

Una colección filtrada por query no deberá marcarse automáticamente como:

```text
FULLY_INITIALIZED
```

si no existe evidencia de completitud.

---

# 99. Collection Completeness

```php
enum CollectionHydrationCompleteness
{
    case COMPLETE;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 100. Why

Evita que:

```text
partially loaded collection
```

sea tratada durante persistence como la colección completa y produzca deletes accidentales.

---

# 101. Snapshot Creation

Una entidad recién hidratada deberá obtener un baseline.

```text
HydratedPersistentState
      ↓
EntitySnapshot
```

---

# 102. Snapshot timing

El snapshot deberá establecerse cuando:

```text
persistent field hydration
```

haya finalizado suficientemente.

---

# 103. Snapshot before postLoad

Recomendación:

```text
hydrate
→ register managed baseline
→ postLoad
```

De esta manera, una mutación persistente realizada en `postLoad` puede ser detectada como cambio posterior.

---

# 104. postLoad mutation

Si:

```php
#[PostLoad]
public function initialize(): void
{
    $this->status = 'processed';
}
```

y `status` es persistente:

```text
entity becomes dirty
```

No debe incorporarse silenciosamente al snapshot inicial.

---

# 105. postLoad semantics

Por tanto:

```text
SnapshotBaseline
=
DatabaseHydratedState
```

no:

```text
StateAfterPostLoadMutation
```

---

# 106. Entity State transition

Una entidad recién materializada:

```text
UNTRACKED
   ↓
HYDRATION INTERNAL
   ↓
MANAGED
```

sin introducir necesariamente `HYDRATION INTERNAL` como estado público.

---

# 107. Registration ordering

La activación final deberá coordinar:

```text
IdentityMap
EntityStateRegistry
UnitOfWork
SnapshotRegistry
```

como transición interna coherente.

---

# 108. Failure atomicity

No deberá quedar:

```text
IdentityMap contains entity
```

pero:

```text
UoW missing entity
```

tras fallo de hidratación.

---

# 109. Hydration Registration Transaction

Conceptualmente:

```text
HydrationRegistrationTransaction
```

coordina las mutaciones internas.

No es una transacción DB.

---

# 110. Registration phases

```text
reserve identity
construct
populate
validate completeness
prepare snapshot
prepare state registration
commit internal registration
postLoad
```

---

# 111. Circular graph complication

En ciclos puede necesitarse una instancia antes del commit completo.

Por ello:

```text
reservation
```

y:

```text
active registration
```

deben distinguirse.

---

# 112. Internal graph visibility

Durante hydration:

```text
reserved entity
```

puede ser visible para:

```text
relationship assembler
proxy resolver
nested hydrator
```

pero no para APIs públicas normales.

---

# 113. Hydration Scope

Cada operación tendrá:

```text
HydrationScope
```

con estado temporal.

---

# 114. Scope contents

Puede contener:

```text
root deduplication
relationship assembly
reservations
row position
partial collections
hydration statistics
temporary object graph
```

---

# 115. Scope lifetime

```text
one hydration execution
```

No:

```text
entire worker lifetime
```

---

# 116. Hydration Scope ≠ PersistenceContext

El PersistenceContext puede sobrevivir a múltiples consultas.

El HydrationScope normalmente corresponde a una materialización concreta.

---

# 117. Query 1

```text
PersistenceContext P1
 └── HydrationScope H1
```

---

# 118. Query 2

```text
PersistenceContext P1
 └── HydrationScope H2
```

Ambas comparten:

```text
IdentityMap P1
```

pero no:

```text
temporary row assembly state
```

---

# 119. Streaming Hydration

Para datasets grandes:

```text
ResultCursor
   ↓
Row
   ↓
Hydrate
   ↓
Yield
```

---

# 120. Problem

Si cada entidad se mantiene en IdentityMap:

```text
1,000,000 rows
→ 1,000,000 managed entities
```

puede agotar memoria.

---

# 121. Streaming Identity Policy

```php
enum StreamingIdentityPolicy
{
    case MANAGED;
    case DETACH_AFTER_YIELD;
    case UNMANAGED_READ_ONLY;
    case WINDOWED;
}
```

---

# 122. MANAGED

Garantía ORM normal.

Mayor consumo de memoria.

---

# 123. DETACH_AFTER_YIELD

Después de entregar una entidad:

```text
yield entity
↓
detach when iteration advances
```

según contrato exacto.

---

# 124. UNMANAGED_READ_ONLY

Puede producir objetos reconstituidos sin registrarlos permanentemente.

Debe ser explícito porque:

```text
same EntityKey
```

ya no garantiza misma instancia entre rows/ventanas.

---

# 125. WINDOWED

Mantiene canonicalidad dentro de una ventana limitada.

---

# 126. Canonicality scope

Debe documentarse:

```text
normal ENTITY hydration
→ PersistenceContext canonicality

streaming unmanaged
→ no managed canonicality guarantee
```

---

# 127. Streaming relationships

Hydration de JOINs to-many en streaming es compleja porque una entidad raíz puede abarcar varias rows.

---

# 128. Root grouping

Puede requerirse:

```text
read rows until root EntityKey changes
↓
complete root graph
↓
yield root
```

---

# 129. Ordering requirement

Ese algoritmo solo será seguro si el Result garantiza agrupación adecuada.

---

# 130. Otherwise

El planner puede:

- añadir ordering;
- usar buffering;
- rechazar esa estrategia;
- elegir otra hydration strategy.

Hydrator no modifica la query por sí mismo.

---

# 131. Hydration Strategy Selection

```text
HydrationMode
+
ResultShape
+
RelationshipShape
+
StreamingMode
+
MemoryPolicy
      ↓
HydrationStrategy
```

---

# 132. Strategies

Ejemplos:

```text
SingleEntityHydrator
EntityCollectionHydrator
JoinedEntityGraphHydrator
ScalarHydrator
TupleHydrator
ProjectionHydrator
DtoHydrator
StreamingEntityHydrator
PartialEntityHydrator
```

Todos forman parte del mismo framework de hidratación.

---

# 133. Strategy Registry

```php
interface HydrationStrategyRegistry
{
    public function resolve(
        HydrationMode $mode,
        HydrationShape $shape
    ): HydrationStrategy;
}
```

---

# 134. No service locator

El registry deberá estar compilado/resuelto por infraestructura.

Los hydrators no buscarán servicios arbitrarios.

---

# 135. Hydration Plan Builder

Responsable de convertir metadata en plan.

```php
interface HydrationPlanBuilder
{
    public function build(
        HydrationPlanDefinition $definition
    ): HydrationPlan;
}
```

---

# 136. Plan validation

Debe detectar antes de ejecutar:

- aliases duplicados;
- identifier incompleto;
- field mapping inexistente;
- conversiones incompatibles;
- constructor imposible;
- DTO constructor incompatible;
- readonly assignment inválido;
- relación imposible;
- ambiguous column;
- unsupported partial hydration.

---

# 137. Fail early

Siempre que sea posible:

```text
query compilation/planning time
```

en lugar de:

```text
row 50,000
```

---

# 138. Plan Optimization

Puede precalcular:

```text
column indexes
field writers
type converters
identifier readers
EntityKey factory
relationship assemblers
constructor invokers
snapshot extractors
```

---

# 139. Generated hydrators

VoltStack podrá introducir posteriormente:

```text
generated PHP hydration code
```

para reducir:

- Reflection;
- dynamic dispatch;
- metadata lookup;
- associative-array access.

---

# 140. Example generated concept

En vez de:

```php
foreach ($metadata->fields() as $field) {
    // dynamic resolution
}
```

podrá compilar conceptualmente:

```php
$id = $intConverter($row[0]);
$name = $stringConverter($row[1]);
$email = $emailConverter($row[2]);
```

---

# 141. Generated code safety

El código generado deberá derivarse únicamente de metadata validada.

Nunca interpolar datos del usuario como código PHP.

---

# 142. Reflection

Permitida durante:

```text
metadata compilation
plan compilation
development fallback
```

Debe minimizarse en hot path.

---

# 143. Hydration Cache

`Hydration Cache` no significará:

```text
cache hydrated entities
```

Eso pertenece a otras capas.

---

# 144. Hydration Cache purpose

Podrá almacenar:

```text
compiled hydration plans
accessor plans
constructor plans
column maps
converter plans
relationship assembly plans
```

---

# 145. Hydration Cache ≠ Entity Cache

Regla:

```text
HydrationPlanCache
≠
IdentityMap
≠
SecondLevelEntityCache
≠
QueryResultCache
```

---

# 146. Read-only Hydration

Una consulta puede declararse:

```text
readOnly
```

---

# 147. Read-only semantics

Dependiendo de policy:

```text
entity can enter IdentityMap
```

pero:

```text
change tracking may be reduced/disabled
```

o producir objetos unmanaged.

---

# 148. Read-only ≠ immutable entity

No significa que PHP impida mutarla.

Significa que ORM no pretende persistir cambios bajo ese modo.

---

# 149. Read-only managed entity

Si se permite, el UoW debe conocer:

```text
read-only registration
```

para no generar ChangeSets.

---

# 150. Mixing read-only and managed

Si una entidad ya existe como managed writable en IdentityMap:

```text
read-only query
```

no deberá degradarla silenciosamente.

---

# 151. Reverse case

Si una entidad read-only managed ya existe y posteriormente se solicita writable, la transición deberá tener policy explícita.

---

# 152. Lock-aware hydration

Pessimistic lock pertenece al Query/Transaction layer.

Hydration consume el Result.

No "crea" el lock.

---

# 153. Version fields

Optimistic locking version fields deberán hidratarse como cualquier campo persistente especial.

Su snapshot será importante posteriormente para UPDATE.

---

# 154. Generated fields

Campos:

```text
database default
computed column
trigger-generated
RETURNING
```

pueden ser hidratados cuando formen parte del Result.

---

# 155. Entity Query Integration

```text
EntityQuery
   ↓
Query Model
   ↓
Query Metadata
   ↓
Execution
   ↓
Result
   ↓
HydrationPlan
   ↓
Entity
```

---

# 156. Query Metadata handoff

Entity Query deberá conservar suficiente metadata para saber:

```text
which alias represents which entity
which column represents which field
which expression represents scalar/projection
which joins represent relationships
```

---

# 157. No SQL parsing

Hydrator nunca deberá analizar el SQL generado para descubrir:

```text
what column means what
```

La semántica deberá viajar mediante metadata estructurada.

---

# 158. Alias stability

SQL Compiler puede renombrar aliases físicos.

El mapping:

```text
logical result slot
↔
physical result alias/index
```

deberá conservarse mediante Result Metadata.

---

# 159. Result Shape

```php
final readonly class HydrationShape
{
    public function __construct(
        public HydrationMode $mode,
        public array $roots,
        public array $scalars,
        public array $relationships,
    ) {}
}
```

---

# 160. Multiple root entities

Consultas avanzadas pueden retornar:

```text
User + Organization
```

como tuple.

Cada root deberá resolverse independientemente por identidad.

---

# 161. Duplicate rows

La deduplicación no debe basarse en:

```text
row equality
```

sino en identidad semántica.

---

# 162. EntityKey construction

```text
Identifier Result Values
       ↓
Type Conversion
       ↓
Canonical Identifier
       ↓
Identity Namespace
       ↓
Identity Domain Type
       ↓
EntityKey
```

---

# 163. Inheritance

El IdentityMap deberá recibir:

```text
canonical identity domain type
```

para evitar:

```text
Person#1
Employee#1
```

como dos instancias cuando representan la misma identidad ORM.

---

# 164. Discriminator

Para inheritance mapping:

```text
discriminator
```

deberá resolverse antes de seleccionar el concrete entity metadata cuando corresponda.

---

# 165. Unknown discriminator

Error explícito.

Nunca:

```text
fallback to base class silently
```

salvo policy declarada.

---

# 166. Embeddables / Value Objects

Hydration deberá poder reconstruir:

```text
Address
Money
Coordinates
```

como partes de una entidad.

---

# 167. Value Object identity

Normalmente no entran en IdentityMap.

Se hidratan como valores.

---

# 168. Nullable embeddable

Debe existir policy para determinar:

```text
all columns NULL
→ null embeddable?
```

o:

```text
embeddable with nullable members?
```

según mapping.

---

# 169. Custom Types

El Type Registry podrá suministrar:

```text
database → PHP converter
```

---

# 170. Conversion purity

Idealmente los converters deberán ser:

```text
deterministic
side-effect free
context bounded
```

---

# 171. Converter context

Puede incluir:

```text
platform
timezone policy
type options
```

pero no servicios arbitrarios de aplicación.

---

# 172. Hydration Security

Aunque los valores vienen de DB:

```text
database data
≠
trusted application data
```

---

# 173. Type safety

Valores incompatibles deberán fallar de manera controlada.

---

# 174. No arbitrary class instantiation

DTO/entity class targets deben proceder de metadata/plan confiable.

Nunca directamente de un string retornado por DB.

---

# 175. Discriminator whitelist

Inheritance discriminators deberán mapearse mediante:

```text
validated discriminator map
```

No:

```php
new $row['class']();
```

---

# 176. Sensitive values

Errores de hydration no deberán incluir por defecto:

```text
password hash
token
secret
full personal data
```

---

# 177. Hydration Limits

La policy podrá establecer:

```text
maximum graph depth
maximum root buffer
maximum relationship buffer
maximum rows per eager graph window
```

para proteger memoria.

---

# 178. Graph explosion

JOINs múltiples:

```text
User
 × Orders
 × Roles
 × Addresses
```

pueden generar producto cartesiano.

---

# 179. Example

```text
1 User
10 Orders
5 Roles
3 Addresses

= 150 result rows
```

para una sola entidad raíz.

---

# 180. Hydration does not solve bad query shape

El sistema deduplicará correctamente, pero no elimina el costo de transferir 150 rows.

---

# 181. Planner integration

Telemetry/N+1/Query Planner podrán recomendar:

```text
batch relation loading
multiple queries
```

en vez de mega-JOIN.

Hydrator no tomará unilateralmente esa decisión.

---

# 182. Memory Model

Coste aproximado:

```text
HydrationMemory
≈
ManagedEntities
+
Snapshots
+
IdentityMapEntries
+
RelationshipCollections
+
HydrationScopeBuffers
+
ResultBuffers
```

---

# 183. Streaming objective

Reducir:

```text
ResultBuffers
+
HydrationScopeBuffers
```

y, con policy adecuada:

```text
ManagedEntities
```

---

# 184. Complexity

Para `n` rows y lookup O(1) promedio:

```text
Entity identity resolution ≈ O(n)
```

---

# 185. Relationship deduplication

Con hash sets:

```text
≈ O(n)
```

promedio.

Evitar:

```text
array linear scan per child
```

que puede degradar a O(n²).

---

# 186. Composite IDs

La canonicalización debe realizarse eficientemente y, cuando sea posible, usando planes precompilados.

---

# 187. Hydration Result

La ejecución deberá producir un contrato adecuado.

```php
interface HydrationResult
{
    public function mode(): HydrationMode;
}
```

Especializaciones:

```text
EntityHydrationResult
ScalarHydrationResult
ProjectionHydrationResult
TupleHydrationResult
StreamingHydrationResult
```

---

# 188. Collection result

La API pública podrá devolver:

```text
Collection
array
iterator
cursor-backed iterable
```

según el sistema de colecciones de VoltStack.

Eso es distinto del mecanismo interno de hidratación.

---

# 189. Empty Result

Entity collection:

```text
[]
```

Single entity:

```text
null
```

cuando la API lo permita.

Scalar aggregate deberá respetar semántica del query.

---

# 190. Duplicate scalar aliases

Deberán rechazarse o resolverse mediante aliases estructurados.

Nunca last-write-wins accidental.

---

# 191. Error Architecture

Jerarquía:

```text
DatabaseHydrationException
├── HydrationPlanException
├── HydrationPlanCompilationException
├── HydrationPlanValidationException
├── HydrationShapeException
├── HydrationColumnException
├── HydrationMissingColumnException
├── HydrationAmbiguousColumnException
├── HydrationTypeConversionException
├── HydrationIdentifierException
├── HydrationIdentityConflictException
├── HydrationReservationException
├── HydrationConstructionException
├── HydrationFieldAssignmentException
├── HydrationReadonlyPropertyException
├── HydrationIncompleteEntityException
├── HydrationPartialEntityException
├── HydrationRelationshipException
├── HydrationCollectionException
├── HydrationCircularGraphException
├── HydrationSnapshotException
├── HydrationRegistrationException
├── HydrationLifecycleException
├── HydrationDtoException
├── HydrationStreamingException
├── HydrationMemoryLimitException
├── HydrationRuntimeIsolationException
└── HydrationInvariantException
```

---

# 192. Error context

Un error podrá contener:

```text
HydrationPlanId
EntityType
FieldId
RelationshipId
ResultSlot
RowNumber
HydrationScopeId
```

sin valores sensibles.

---

# 193. Row number

En streaming puede ser útil:

```text
row 15482
```

para diagnóstico.

---

# 194. Failure behavior

Si falla una entidad durante hydration:

```text
abort reservations
rollback temporary registration
discard temporary relationship assembly
close/release result resources as required
```

---

# 195. Existing managed entities

No deben ser destruidas porque una consulta posterior falle hidratando otra row.

---

# 196. Partial graph failure

Si el HydrationScope ya modificó colecciones managed antes de fallar, deberá existir estrategia de rollback interno o marcar el scope/contexto para reconciliation.

---

# 197. Preferred strategy

Siempre que sea viable:

```text
stage relationship assembly
↓
validate
↓
commit assembly
```

para reducir mutaciones parciales.

---

# 198. Hydration atomicity

No se promete atomicidad de toda una consulta a nivel DB.

Se busca:

```text
internal ORM structural consistency
```

ante fallos.

---

# 199. Hydration and Transactions

Hydration puede ocurrir:

```text
inside transaction
outside transaction
```

No comienza transacciones automáticamente.

---

# 200. Hydration and lazy loading

Lazy loading inicia una nueva Entity Query/Execution/Hydration operation.

No es una continuación mágica del Result original.

---

# 201. Lazy relation flow

```text
Proxy/Collection access
      ↓
Relationship Loader
      ↓
Entity Query
      ↓
Execution
      ↓
Hydration
      ↓
IdentityMap reconciliation
```

---

# 202. Hydration recursion

El sistema deberá limitar recursion accidental.

---

# 203. Circular relationships

Se resuelven mediante:

```text
IdentityMap
reservations
relationship fix-up
```

no mediante recursión infinita.

---

# 204. Hydration Depth

```php
final readonly class HydrationDepth
{
    public function __construct(
        public int $value
    ) {}
}
```

Puede utilizarse para políticas y diagnóstico.

---

# 205. Telemetry

Métricas propuestas:

```text
orm.hydration.operations
orm.hydration.rows
orm.hydration.entities
orm.hydration.scalars
orm.hydration.dtos
orm.hydration.duration
orm.hydration.failures

orm.hydration.identity.hit
orm.hydration.identity.miss
orm.hydration.identity.reservation

orm.hydration.entity.constructed
orm.hydration.entity.reused

orm.hydration.relationship.assembled
orm.hydration.relationship.deduplicated

orm.hydration.partial_entities
orm.hydration.streaming.rows

orm.hydration.plan.cache.hit
orm.hydration.plan.cache.miss

orm.hydration.type_conversion.failure
```

---

# 206. Useful ratios

```text
IdentityReuseRatio
=
ReusedEntities
/
ResolvedEntities
```

---

# 207. Row amplification

```text
RowAmplification
=
ResultRows
/
UniqueRootEntities
```

---

# 208. Example

```text
Rows: 1500
Unique Users: 100

RowAmplification = 15
```

Puede indicar eager JOIN costoso.

---

# 209. Hydration throughput

```text
HydratedRowsPerSecond
```

será útil para benchmarks.

---

# 210. Memory telemetry

En profiler/debug mode:

```text
peak hydration scope memory
identity map growth
snapshot growth
relationship buffer size
```

---

# 211. Debug Toolbar

Ejemplo:

```text
Hydration

Mode: ENTITY
Plan: hp_48F1

Rows read:              2,540
Root entities:            200
Related entities:       1,103

Identity hits:          1,237
Identity misses:          266

Objects constructed:      266
Objects reused:         1,237

Row amplification:       12.7x

Duration:               34.2 ms

Hydration plan:
  cache: HIT

Collections:
  complete:               188
  partial:                 12
```

---

# 212. Persistent Runtime

La arquitectura deberá ser segura bajo:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 213. Shared state allowed

Solo estructuras inmutables:

```text
CompiledHydrationPlan
HydrationStrategyDefinition
CompiledAccessor
CompiledConverterPlan
HydrationMetadata
```

---

# 214. Scoped state

Nunca compartir:

```text
HydrationScope
Result
current row
current entity
reservations
relationship buffers
PersistenceContext
IdentityMap
EntityManager
tenant context
```

---

# 215. Worker reset

Al terminar request/job:

```text
temporary hydration state
→ destroyed
```

---

# 216. No static current hydrator context

Prohibido:

```php
Hydrator::$currentEntity
```

o:

```php
HydrationContext::current()
```

mediante global mutable state.

---

# 217. Coroutine safety

Cada coroutine tendrá su contexto lógico independiente.

---

# 218. Concurrent hydration

Un mismo EntityManager/PersistenceContext no admitirá por defecto dos hydrations concurrentes que puedan registrar las mismas identidades.

---

# 219. Why

Podrían producir:

```text
Query A → User#42 MISS
Query B → User#42 MISS
```

simultáneamente.

---

# 220. Resolution

V1 puede:

```text
reject concurrent mutable hydration
```

en el mismo PersistenceContext.

Posteriormente podría existir coordinación mediante reservations.

---

# 221. Extension System

Extensiones podrán registrar:

- HydrationMode;
- HydrationStrategy;
- Type converter;
- DTO constructor strategy;
- Value Object hydrator;
- Result shape handler;
- field writer;
- relationship assembler.

---

# 222. Extension rule

Una extensión no podrá:

```text
bypass IdentityMap canonicality
```

para entidades managed.

---

# 223. Extension rule II

No podrá marcar una entidad:

```text
MANAGED
```

sin completar el protocolo de registro ORM.

---

# 224. Extension collisions

Dos handlers para el mismo:

```text
mode + shape
```

deberán resolverse mediante prioridad determinista o error explícito.

Nunca:

```text
last registered wins
```

silenciosamente.

---

# 225. Proposed Contracts

```php
interface Hydrator
{
    public function hydrate(
        HydrationRequest $request
    ): HydrationResult;
}
```

---

# 226. HydrationStrategy

```php
interface HydrationStrategy
{
    public function supports(HydrationShape $shape): bool;

    public function hydrate(
        ResultInterface $result,
        CompiledHydrationPlan $plan,
        HydrationContext $context
    ): HydrationResult;
}
```

---

# 227. EntityHydrator

```php
interface EntityHydrator
{
    public function hydrateEntity(
        ResultRow $row,
        CompiledEntityHydrationPlan $plan,
        HydrationContext $context
    ): object;
}
```

---

# 228. ScalarHydrator

```php
interface ScalarHydrator
{
    public function hydrateScalar(
        ResultValue $value,
        ScalarHydrationPlan $plan
    ): mixed;
}
```

---

# 229. HydrationPlanCompiler

```php
interface HydrationPlanCompiler
{
    public function compile(
        HydrationPlan $plan
    ): CompiledHydrationPlan;
}
```

---

# 230. Directory Structure

```text
src/Quantum/Database/ORM/Hydration/
│
├── Contract/
│   ├── Hydrator.php
│   ├── EntityHydrator.php
│   ├── ScalarHydrator.php
│   ├── HydrationStrategy.php
│   ├── HydrationPlanBuilder.php
│   └── HydrationPlanCompiler.php
│
├── Model/
│   ├── HydrationMode.php
│   ├── HydrationShape.php
│   ├── HydrationRequest.php
│   ├── HydrationContext.php
│   ├── HydrationScopeId.php
│   ├── HydrationValuePresence.php
│   └── HydrationResult.php
│
├── Plan/
│   ├── HydrationPlan.php
│   ├── HydrationPlanId.php
│   ├── HydrationPlanDefinition.php
│   ├── CompiledHydrationPlan.php
│   ├── HydrationPlanValidator.php
│   ├── DefaultHydrationPlanBuilder.php
│   └── DefaultHydrationPlanCompiler.php
│
├── Entity/
│   ├── DefaultEntityHydrator.php
│   ├── EntityConstructionStrategy.php
│   ├── EntityConstructionContext.php
│   ├── EntityFieldWriter.php
│   ├── ExistingEntityHydrationPolicy.php
│   └── LoadedFieldSet.php
│
├── Identity/
│   ├── HydrationIdentityResolver.php
│   ├── HydrationIdentityReservation.php
│   └── HydrationRegistrationCoordinator.php
│
├── Scalar/
│   └── DefaultScalarHydrator.php
│
├── Projection/
│   └── ProjectionHydrator.php
│
├── Tuple/
│   ├── TupleHydrator.php
│   └── TupleSlot.php
│
├── Dto/
│   ├── DtoHydrator.php
│   └── DtoConstructionPlan.php
│
├── Partial/
│   ├── PartialEntityHydrator.php
│   └── PartialEntityPolicy.php
│
├── Relationship/
│   ├── RelationshipHydrator.php
│   ├── RelationshipAssembler.php
│   ├── RelationshipAssemblyState.php
│   └── CollectionHydrationCompleteness.php
│
├── Conversion/
│   ├── HydrationValueConverter.php
│   └── HydrationConversionContext.php
│
├── Streaming/
│   ├── StreamingHydrator.php
│   └── StreamingIdentityPolicy.php
│
├── Scope/
│   ├── HydrationScope.php
│   ├── HydrationScopeFactory.php
│   └── HydrationScopeResetter.php
│
├── Cache/
│   ├── HydrationPlanCache.php
│   └── HydrationPlanCacheKey.php
│
├── Registry/
│   └── HydrationStrategyRegistry.php
│
├── Telemetry/
│   ├── HydrationTelemetry.php
│   ├── HydrationStatistics.php
│   └── HydrationDiagnostics.php
│
├── Extension/
│   └── HydrationExtensionRegistry.php
│
└── Exception/
    └── ...
```

---

# 231. Integración con otros módulos

```text
ORM/Hydration
│
├── ORM/EntityMetadata
├── ORM/IdentityMap
├── ORM/EntityState
├── ORM/Snapshot
├── ORM/UnitOfWork
├── ORM/Lifecycle
│
├── Query/Metadata
├── Result
├── TypeSystem
│
├── Telemetry
└── Runtime
```

---

# 232. Prohibited Dependencies

Hydration no dependerá directamente de:

```text
PDO
MySQL driver
PostgreSQL driver
SQLite driver
SQL compiler internals
HTTP request
Authorization
Cache implementation
Multitenancy package
```

---

# 233. Multitenancy

Hydration solo recibe:

```text
EntityIdentityNamespace
```

mediante PersistenceContext.

No necesita conocer:

```text
Tenant model
Tenant package
tenant database strategy
```

---

# 234. Identity isolation

La misma row identity:

```text
User#42
```

en:

```text
Tenant A
```

y:

```text
Tenant B
```

deberá producir diferentes `EntityKey`.

---

# 235. Testing Strategy

Se deberán cubrir como mínimo los siguientes escenarios.

---

# 236. Basic entity

Row completa produce entidad managed correcta.

---

# 237. Identity reuse

Dos rows con mismo `EntityKey` reutilizan la misma instancia.

---

# 238. Different entity type

```text
User#1
Order#1
```

no colisionan.

---

# 239. Different namespace

Mismo type/ID en dos namespaces no colisiona.

---

# 240. Composite ID

Canonicalización correcta.

---

# 241. Custom ID type

Strongly typed identifier correcto.

---

# 242. SQL NULL

Se convierte según mapping nullable.

---

# 243. Missing field

No se interpreta como NULL.

---

# 244. Type conversion

DB representation produce PHP representation correcta.

---

# 245. Conversion failure

Error contextual sin sensitive value.

---

# 246. Existing managed dirty entity

Una query normal no sobrescribe cambios locales.

---

# 247. Refresh

Actualiza la misma instancia canónica.

---

# 248. Circular graph

No crea duplicados ni recursion infinita.

---

# 249. Reservation failure

Limpia estado temporal.

---

# 250. Duplicate JOIN rows

No duplica root entity.

---

# 251. Duplicate child

No duplica child en colección set-like.

---

# 252. LEFT JOIN null child

No crea entidad fantasma.

---

# 253. Filtered collection

No se marca completa sin evidencia.

---

# 254. Bidirectional relationship

Fix-up no genera dirty state artificial.

---

# 255. Snapshot

Representa estado proveniente de DB.

---

# 256. postLoad mutation

Produce cambio posterior al baseline.

---

# 257. Constructor bypass

Respeta metadata.

---

# 258. Constructor hydration

Pasa argumentos correctos.

---

# 259. Readonly

Configuraciones válidas funcionan; inválidas fallan temprano.

---

# 260. Typed properties

No reciben tipos incompatibles.

---

# 261. DTO

No entra al IdentityMap.

---

# 262. Scalar

No activa UoW.

---

# 263. Projection

No se registra como entidad.

---

# 264. Partial entity

Conserva LoadedFieldSet.

---

# 265. Partial entity persistence

No convierte missing fields en NULL.

---

# 266. Streaming managed

Mantiene semántica documentada.

---

# 267. Streaming detach

Limita crecimiento del IdentityMap.

---

# 268. Streaming graph

Respeta root grouping.

---

# 269. Plan cache

Mismo shape reutiliza plan compatible.

---

# 270. Metadata generation

Cambio de metadata invalida plan incompatible.

---

# 271. Runtime isolation

Request A no comparte HydrationScope con B.

---

# 272. Concurrent hydration

Uso no soportado se rechaza.

---

# 273. Hydration failure

No deja IdentityMap/UoW corruptos.

---

# 274. Custom extension

Respeta canonicalidad.

---

# 275. Telemetry

No incluye sensitive values.

---

# 276. Architectural Invariants

## DB-ORM-HYDRATION-001

Hydration no ejecutará SQL.

## DB-ORM-HYDRATION-002

Hydration no generará SQL.

## DB-ORM-HYDRATION-003

Hydration no decidirá qué query ejecutar.

## DB-ORM-HYDRATION-004

Hydration no será un persistence engine.

## DB-ORM-HYDRATION-005

Hydration no será serialization.

## DB-ORM-HYDRATION-006

Hydration no será entity mapping.

## DB-ORM-HYDRATION-007

Hydration consumirá metadata estructurada.

## DB-ORM-HYDRATION-008

Hydration no analizará SQL para reconstruir semántica ORM.

## DB-ORM-HYDRATION-009

HydrationMode será explícito.

## DB-ORM-HYDRATION-010

ENTITY será distinto de SCALAR.

## DB-ORM-HYDRATION-011

ENTITY será distinto de DTO.

## DB-ORM-HYDRATION-012

ENTITY será distinto de PROJECTION.

## DB-ORM-HYDRATION-013

PARTIAL_ENTITY será explícito.

## DB-ORM-HYDRATION-014

Missing value será distinto de NULL.

## DB-ORM-HYDRATION-015

HydrationPlan será distinto de QueryPlan.

## DB-ORM-HYDRATION-016

CompiledHydrationPlan será inmutable.

## DB-ORM-HYDRATION-017

Hot path evitará reflection repetitiva.

## DB-ORM-HYDRATION-018

Hydration plan podrá cachearse cuando sea estructuralmente seguro.

## DB-ORM-HYDRATION-019

Plan cache no almacenará entidades.

## DB-ORM-HYDRATION-020

Plan cache no almacenará EntityManager.

## DB-ORM-HYDRATION-021

Plan cache no almacenará request state.

## DB-ORM-HYDRATION-022

Identifier se resolverá antes de construir entidad cuando sea posible.

## DB-ORM-HYDRATION-023

IdentityMap hit reutilizará instancia canónica.

## DB-ORM-HYDRATION-024

IdentityMap hit no autorizará overwrite arbitrario.

## DB-ORM-HYDRATION-025

IdentityMap miss podrá iniciar reservation.

## DB-ORM-HYDRATION-026

Reservation evitará duplicados durante grafos circulares.

## DB-ORM-HYDRATION-027

Reserved entity no escapará normalmente al usuario.

## DB-ORM-HYDRATION-028

Hydration failure abortará reservation.

## DB-ORM-HYDRATION-029

Hydration no requerirá constructor vacío universal.

## DB-ORM-HYDRATION-030

Construction strategy será explícita.

## DB-ORM-HYDRATION-031

Constructor bypass será controlado.

## DB-ORM-HYDRATION-032

Hydration distinguirá creation de reconstitution.

## DB-ORM-HYDRATION-033

Field assignment será metadata-driven.

## DB-ORM-HYDRATION-034

Setters no serán usados universalmente de forma implícita.

## DB-ORM-HYDRATION-035

Readonly properties serán validadas.

## DB-ORM-HYDRATION-036

Typed properties recibirán valores compatibles.

## DB-ORM-HYDRATION-037

Full entity no escapará con required fields no inicializados.

## DB-ORM-HYDRATION-038

Partial entity conservará información de campos cargados.

## DB-ORM-HYDRATION-039

Missing partial field no será convertido en NULL.

## DB-ORM-HYDRATION-040

Projection será preferida para lecturas incompletas cuando sea posible.

## DB-ORM-HYDRATION-041

DTO no entrará al IdentityMap.

## DB-ORM-HYDRATION-042

DTO no será MANAGED.

## DB-ORM-HYDRATION-043

Scalar hydration no utilizará UoW innecesariamente.

## DB-ORM-HYDRATION-044

Tuple slots estarán descritos estructuralmente.

## DB-ORM-HYDRATION-045

JOIN duplicate rows no crearán duplicate root entities.

## DB-ORM-HYDRATION-046

Relationship child deduplication utilizará identidad.

## DB-ORM-HYDRATION-047

LEFT JOIN sin child no creará entidad fantasma.

## DB-ORM-HYDRATION-048

Relationship fix-up inicial no generará dirty state artificial.

## DB-ORM-HYDRATION-049

Collection completeness será explícita.

## DB-ORM-HYDRATION-050

Filtered JOIN collection no será considerada completa automáticamente.

## DB-ORM-HYDRATION-051

Snapshot baseline representará estado hidratado de DB.

## DB-ORM-HYDRATION-052

postLoad ocurrirá después del baseline inicial.

## DB-ORM-HYDRATION-053

Mutación persistente en postLoad será detectable como cambio.

## DB-ORM-HYDRATION-054

Entity activation coordinará IdentityMap, State, Snapshot y UoW.

## DB-ORM-HYDRATION-055

Hydration registration será internamente coherente.

## DB-ORM-HYDRATION-056

Hydration registration transaction no será DB transaction.

## DB-ORM-HYDRATION-057

HydrationScope será temporal.

## DB-ORM-HYDRATION-058

HydrationScope será distinto de PersistenceContext.

## DB-ORM-HYDRATION-059

Múltiples HydrationScopes podrán compartir un PersistenceContext.

## DB-ORM-HYDRATION-060

HydrationScope state no sobrevivirá accidentalmente al request.

## DB-ORM-HYDRATION-061

Streaming no implicará necesariamente full managed identity semantics.

## DB-ORM-HYDRATION-062

Streaming identity policy será explícita.

## DB-ORM-HYDRATION-063

DETACH_AFTER_YIELD tendrá semántica explícita.

## DB-ORM-HYDRATION-064

UNMANAGED_READ_ONLY no prometerá canonicalidad managed.

## DB-ORM-HYDRATION-065

WINDOWED limitará la garantía a su ventana.

## DB-ORM-HYDRATION-066

Streaming joined graphs respetará requisitos de agrupación.

## DB-ORM-HYDRATION-067

Hydrator no modificará query para obtener ordering ocultamente.

## DB-ORM-HYDRATION-068

Hydration strategy será seleccionada por mode y shape.

## DB-ORM-HYDRATION-069

Múltiples strategies no constituirán múltiples ORM engines.

## DB-ORM-HYDRATION-070

Strategy registry será determinista.

## DB-ORM-HYDRATION-071

Hydration plans inválidos fallarán temprano cuando sea posible.

## DB-ORM-HYDRATION-072

Generated hydrators solo usarán metadata confiable.

## DB-ORM-HYDRATION-073

Database data nunca será interpolada como PHP code.

## DB-ORM-HYDRATION-074

Hydration Cache será distinto de Entity Cache.

## DB-ORM-HYDRATION-075

Hydration Cache será distinto de IdentityMap.

## DB-ORM-HYDRATION-076

Hydration Cache será distinto de Query Result Cache.

## DB-ORM-HYDRATION-077

Read-only hydration tendrá semántica explícita.

## DB-ORM-HYDRATION-078

Read-only no significará PHP immutable.

## DB-ORM-HYDRATION-079

Read-only query no degradará silenciosamente entidad writable existente.

## DB-ORM-HYDRATION-080

Hydration no será propietario de pessimistic locks.

## DB-ORM-HYDRATION-081

Version fields formarán parte del baseline cuando sean cargados.

## DB-ORM-HYDRATION-082

Generated fields podrán hidratarse desde Result.

## DB-ORM-HYDRATION-083

Query semantics llegarán mediante metadata, no SQL parsing.

## DB-ORM-HYDRATION-084

Physical aliases estarán correlacionados con logical result slots.

## DB-ORM-HYDRATION-085

Multiple root entities se resolverán independientemente.

## DB-ORM-HYDRATION-086

Deduplication será identity-based, no row-based.

## DB-ORM-HYDRATION-087

EntityKey utilizará identifier canonicalizado.

## DB-ORM-HYDRATION-088

EntityKey incluirá IdentityNamespace.

## DB-ORM-HYDRATION-089

Inheritance utilizará canonical identity domain.

## DB-ORM-HYDRATION-090

Unknown discriminator fallará explícitamente.

## DB-ORM-HYDRATION-091

Database discriminator no podrá instanciar clases arbitrarias.

## DB-ORM-HYDRATION-092

Embeddables no entrarán al IdentityMap por defecto.

## DB-ORM-HYDRATION-093

Nullable embeddable tendrá mapping explícito.

## DB-ORM-HYDRATION-094

Custom types utilizarán Type System.

## DB-ORM-HYDRATION-095

Hydration no duplicará lógica de Type System.

## DB-ORM-HYDRATION-096

Conversion errors serán diagnósticos.

## DB-ORM-HYDRATION-097

Sensitive values no aparecerán por defecto en errores.

## DB-ORM-HYDRATION-098

Target entity/DTO class procederá de metadata confiable.

## DB-ORM-HYDRATION-099

Hydration podrá imponer límites de recursos.

## DB-ORM-HYDRATION-100

Hydration no ocultará row amplification.

## DB-ORM-HYDRATION-101

Hydrator no resolverá unilateralmente query graph explosion.

## DB-ORM-HYDRATION-102

Relationship deduplication evitará algoritmos O(n²) innecesarios.

## DB-ORM-HYDRATION-103

HydrationResult será explícito.

## DB-ORM-HYDRATION-104

Public collection API será distinta del hydration mechanism.

## DB-ORM-HYDRATION-105

Duplicate aliases no usarán last-write-wins accidental.

## DB-ORM-HYDRATION-106

Hydration errors tendrán contexto estructurado.

## DB-ORM-HYDRATION-107

Hydration failure limpiará temporary state.

## DB-ORM-HYDRATION-108

Hydration failure no destruirá managed entities previas no relacionadas.

## DB-ORM-HYDRATION-109

Partial graph failure deberá preservar consistencia ORM.

## DB-ORM-HYDRATION-110

Hydration no prometerá DB transaction atomicity.

## DB-ORM-HYDRATION-111

Hydration no iniciará transaction automáticamente.

## DB-ORM-HYDRATION-112

Lazy loading utilizará un nuevo query/execution/hydration cycle.

## DB-ORM-HYDRATION-113

Circular relationships no producirán recursion infinita.

## DB-ORM-HYDRATION-114

Hydration depth podrá ser gobernada.

## DB-ORM-HYDRATION-115

Telemetry distinguirá rows de unique entities.

## DB-ORM-HYDRATION-116

Telemetry medirá identity hits y misses.

## DB-ORM-HYDRATION-117

Telemetry podrá medir row amplification.

## DB-ORM-HYDRATION-118

Telemetry no expondrá identifiers sensibles.

## DB-ORM-HYDRATION-119

Compiled plans inmutables podrán compartirse entre requests.

## DB-ORM-HYDRATION-120

HydrationScope mutable nunca será process-global.

## DB-ORM-HYDRATION-121

Current row nunca será process-global.

## DB-ORM-HYDRATION-122

Current entity nunca será process-global.

## DB-ORM-HYDRATION-123

FrankenPHP aislará hydration state por request.

## DB-ORM-HYDRATION-124

RoadRunner aislará hydration state por request/job.

## DB-ORM-HYDRATION-125

OpenSwoole aislará hydration state por contexto lógico.

## DB-ORM-HYDRATION-126

Concurrent mutable hydration en mismo PersistenceContext no será asumida segura.

## DB-ORM-HYDRATION-127

Unsupported concurrent hydration será rechazada.

## DB-ORM-HYDRATION-128

Extensions no podrán romper IdentityMap canonicality.

## DB-ORM-HYDRATION-129

Extensions no podrán registrar entidades MANAGED fuera del protocolo ORM.

## DB-ORM-HYDRATION-130

Extension resolution será determinista.

## DB-ORM-HYDRATION-131

Entity hydration no implicará entity freshness perpetua.

## DB-ORM-HYDRATION-132

IdentityMap hit no implicará DB query.

## DB-ORM-HYDRATION-133

IdentityMap miss no implicará row absence fuera del Result actual.

## DB-ORM-HYDRATION-134

Hydrated entity no implicará transaction commit.

## DB-ORM-HYDRATION-135

Hydrated entity no implicará authorization.

## DB-ORM-HYDRATION-136

Hydrated entity no implicará validation success de negocio.

## DB-ORM-HYDRATION-137

Hydration será determinista respecto al Result, plan y contexto observable permitido.

## DB-ORM-HYDRATION-138

Hydration no ejecutará side effects externos por defecto.

## DB-ORM-HYDRATION-139

Hydration callbacks deberán respetar Lifecycle System.

## DB-ORM-HYDRATION-140

postLoad no se disparará repetidamente por duplicate JOIN rows.

## DB-ORM-HYDRATION-141

Refresh no utilizará postLoad como sustituto de postRefresh.

## DB-ORM-HYDRATION-142

Proxy initialization respetará canonicalidad.

## DB-ORM-HYDRATION-143

Proxy class no alterará canonical EntityType.

## DB-ORM-HYDRATION-144

Partial collections conservarán su incompletitud.

## DB-ORM-HYDRATION-145

Unknown collection completeness no será convertida a COMPLETE.

## DB-ORM-HYDRATION-146

Snapshot no incluirá valores MISSING como NULL.

## DB-ORM-HYDRATION-147

Hydration baseline no incluirá postLoad mutations.

## DB-ORM-HYDRATION-148

Read-only unmanaged hydration no contaminará UoW.

## DB-ORM-HYDRATION-149

Streaming detach liberará referencias ORM según contrato.

## DB-ORM-HYDRATION-150

HydrationPlan incompatibility invalidará cache reutilizado.

## DB-ORM-HYDRATION-151

Metadata generation participará en compatibilidad del plan.

## DB-ORM-HYDRATION-152

Type system generation participará cuando afecte conversiones.

## DB-ORM-HYDRATION-153

Plan cache key no dependerá de mutable request state.

## DB-ORM-HYDRATION-154

Hydration temporary state será liberado determinísticamente.

## DB-ORM-HYDRATION-155

Result resources serán liberados conforme al Result System.

## DB-ORM-HYDRATION-156

Hydrator no cerrará conexiones arbitrariamente.

## DB-ORM-HYDRATION-157

Hydrator no hará commit.

## DB-ORM-HYDRATION-158

Hydrator no hará rollback.

## DB-ORM-HYDRATION-159

Hydrator no hará flush.

## DB-ORM-HYDRATION-160

Toda materialización de entidad managed en VoltStack deberá preservar identidad canónica, baseline de persistencia, estado ORM coherente y aislamiento de runtime sin confundir hidratación con consulta, persistencia, transacción o serialización.

---

# 277. Fórmulas fundamentales

## 277.1 Hydration

```text
Hydration
=
Result
+
HydrationPlan
+
HydrationContext
→
PHP Representation
```

---

# 278. Entity Hydration

```text
EntityHydration
=
ResultRow
+
CompiledEntityHydrationPlan
+
TypeConversion
+
IdentityResolution
+
EntityConstructionOrReuse
+
FieldPopulation
+
RelationshipAssembly
+
SnapshotRegistration
+
EntityStateRegistration
+
Lifecycle
```

---

# 279. Entity Key

```text
EntityKey
=
IdentityNamespace
×
IdentityDomainType
×
CanonicalIdentifier
```

---

# 280. Canonical entity resolution

```text
ResolveEntity(k)
=
IdentityMap[k]
    if present

otherwise
    Reserve(k)
    → Construct
    → Hydrate
    → Register
```

---

# 281. Canonicality

```text
∀ a,b,k:

Managed(a,k)
∧
Managed(b,k)

⇒

a === b
```

---

# 282. Safe activation

```text
SafeEntityActivation
=
IdentityReserved
∧
RequiredFieldsHydrated
∧
TypesValid
∧
IdentityUnique
∧
SnapshotPrepared
∧
UoWRegistrationPrepared
∧
EntityStateRegistrationPrepared
```

---

# 283. Baseline

```text
InitialSnapshot
=
PersistentStateImmediatelyAfterDatabaseHydration
```

antes de mutaciones lifecycle posteriores.

---

# 284. Partial entity

```text
PartialEntity
=
EntityIdentity
+
LoadedFieldSet
+
LoadedPersistentValues
+
ExplicitIncompleteState
```

---

# 285. Missing semantics

```text
MISSING
≠
NULL
≠
DEFAULT
≠
UNINITIALIZED
```

---

# 286. Relationship assembly

```text
RelationshipGraph
=
UniqueRootEntities
+
UniqueRelatedEntities
+
RelationshipEdges
+
CollectionCompleteness
```

---

# 287. Row amplification

```text
RowAmplification
=
ResultRowCount
/
UniqueRootEntityCount
```

---

# 288. Hydration complexity

Con índices hash adecuados:

```text
T(n)
≈
O(n)
```

respecto al número de rows, más costos de conversión y construcción.

---

# 289. Memory

```text
M
≈
ManagedEntities
+
Snapshots
+
IdentityEntries
+
RelationshipState
+
HydrationBuffers
+
ResultBuffers
```

---

# 290. Streaming

```text
StreamingHydration
=
Cursor
+
BoundedHydrationState
+
ExplicitIdentityPolicy
+
DeterministicResourceRelease
```

---

# 291. Safe Runtime

```text
SafeHydrationRuntime
=
ImmutableSharedPlans
∧
ScopedMutableHydrationState
∧
ScopedPersistenceContext
∧
NoCrossRequestReservations
∧
DeterministicReset
```

---

# 292. Master Formula

```text
Database Hydration Architecture
=
Hydration Modes
+
Hydration Requests
+
Hydration Context
+
Hydration Scope
+
Result Metadata
+
Hydration Shape
+
Hydration Plans
+
Compiled Hydration Plans
+
Plan Validation
+
Plan Optimization
+
Row Reading
+
Value Presence Semantics
+
Type Conversion
+
Identifier Extraction
+
EntityKey Construction
+
IdentityMap Resolution
+
Identity Reservations
+
Entity Construction
+
Reconstitution Strategies
+
Field Assignment
+
Readonly Property Support
+
Partial Entity Semantics
+
Loaded Field Tracking
+
Scalar Hydration
+
Projection Hydration
+
Tuple Hydration
+
DTO Hydration
+
Relationship Assembly
+
Root Deduplication
+
Child Deduplication
+
Collection Completeness
+
Relationship Fix-Up
+
Snapshot Creation
+
UnitOfWork Registration
+
Entity State Registration
+
Lifecycle Integration
+
Streaming Hydration
+
Memory Governance
+
Hydration Plan Cache
+
Read-Only Hydration
+
Inheritance Resolution
+
Embeddable Hydration
+
Extension System
+
Error Architecture
+
Persistent Runtime Isolation
+
Telemetry
+
Diagnostics
+
Testing
```

---

# 293. Master Rule

> **En VoltStack, la hidratación será un proceso planificado, tipado y determinista que transforma resultados del Execution Engine en representaciones PHP sin conocer SQL ni ejecutar persistencia. Cuando el resultado represente entidades administradas, toda materialización deberá integrarse con el `IdentityMap`, reutilizar la instancia canónica existente, establecer un baseline de persistencia correcto y registrar de forma coherente `EntityState`, `Snapshot` y `UnitOfWork`. Los JOINs, relaciones circulares, resultados parciales y streaming nunca podrán utilizarse como justificación para romper la identidad canónica o confundir `MISSING` con `NULL`.**

---

# 294. Resultado arquitectónico

Después de este documento, el camino completo de lectura ORM queda:

```text
Entity Query
    │
    ▼
Query Model / AST
    │
    ▼
Semantic Engine
    │
    ▼
Optimizer
    │
    ▼
Planner
    │
    ▼
SQL Compiler
    │
    ▼
Execution Engine
    │
    ▼
Result System
    │
    ▼
Hydration Plan
    │
    ▼
Hydration Engine
    │
    ├── Scalar
    ├── Projection
    ├── Tuple
    ├── DTO
    ├── Partial Entity
    └── Entity
          │
          ▼
     Identity Resolution
          │
          ▼
       IdentityMap
          │
          ▼
   Construction / Reuse
          │
          ▼
     Type Conversion
          │
          ▼
    Field Population
          │
          ▼
 Relationship Assembly
          │
          ▼
      Snapshot
          │
          ▼
     EntityState
          │
          ▼
      UnitOfWork
          │
          ▼
       postLoad
          │
          ▼
      Application
```

La arquitectura queda simétrica respecto a persistencia:

```text
READ PATH

Database
   ↓
Execution
   ↓
Result
   ↓
Hydration
   ↓
Entity


WRITE PATH

Entity
   ↓
UnitOfWork
   ↓
Persistence
   ↓
Query Engine
   ↓
Execution
   ↓
Database
```

Y ambos caminos convergen sobre:

```text
Entity Metadata
Type System
IdentityMap
EntityState
Snapshots
PersistenceContext
```

sin duplicar responsabilidades.

---

# 295. Documentos del Bloque 12

```text
135_DATABASE_HYDRATION_ARCHITECTURE.md
136_DATABASE_ENTITY_HYDRATOR_SYSTEM.md
137_DATABASE_RESULT_HYDRATION_SYSTEM.md
138_DATABASE_SCALAR_HYDRATION_SYSTEM.md
139_DATABASE_PARTIAL_ENTITY_HYDRATION_SYSTEM.md
140_DATABASE_HYDRATION_PLAN_SYSTEM.md
141_DATABASE_HYDRATION_CACHE_SYSTEM.md
```

Este documento `135` establece la arquitectura común.

Los siguientes documentos profundizarán cada componente sin modificar las invariantes definidas aquí.

---

# 296. Siguiente documento

```text
136_DATABASE_ENTITY_HYDRATOR_SYSTEM.md
```

El siguiente documento deberá profundizar específicamente en:

```text
EntityHydrator
EntityHydrationRequest
EntityHydrationPlan
identifier-first hydration
EntityKey construction
IdentityMap lookup
identity reservations
two-phase hydration
entity construction
constructor bypass
field writers
readonly properties
typed properties
embedded values
inheritance/discriminators
existing managed entities
dirty entity protection
relationship graph assembly
duplicate JOIN rows
circular references
snapshot baseline
UoW registration
EntityState registration
postLoad
refresh integration
proxy initialization
failure atomicity
streaming entity hydration
persistent runtime isolation
telemetry
performance
```

La regla que deberá gobernarlo será:

> **El `EntityHydrator` nunca crea una segunda instancia administrada cuando el `PersistenceContext` ya posee una representación canónica de la misma identidad; construye únicamente cuando la identidad no está representada y completa su registro mediante un protocolo atómico de hidratación.**