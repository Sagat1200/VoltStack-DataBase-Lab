# 140_DATABASE_HYDRATION_PLAN_SYSTEM.md

# VoltStack Quantum Database
## Database Hydration Plan System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 140 — Database Hydration Plan System  
**Bloque:** 12 — Hydration  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Hydration Plan System` define la arquitectura utilizada por VoltStack para describir, validar, normalizar, optimizar y compilar la transformación entre:

```text id="4x3u0b"
Physical Result Layout
```

y:

```text id="x1vte9"
Logical Application Result
```

El sistema constituye la capa declarativa y compilada que indica a los hydrators:

- qué representa cada columna;
- qué tipo lógico tiene;
- qué entidad representa cada grupo de columnas;
- dónde se encuentra su identificador;
- cómo convertir cada valor;
- cómo construir una entidad;
- cómo escribir cada propiedad;
- cómo materializar DTOs;
- cómo construir tuples;
- cómo ensamblar relaciones;
- cómo detectar ausencia en JOINs;
- cómo agrupar rows;
- cómo deduplicar resultados;
- qué cardinalidad lógica se espera;
- qué estrategia de streaming es válida;
- cuándo un resultado lógico puede considerarse completo.

La regla fundamental será:

> **Los hydrators ejecutan planes; no reconstruyen la semántica del resultado durante el hot path.**

---

# 2. Principio central

> **`HydrationPlan` es la representación estructural, validada y compilable de cómo convertir un `Result` físico en un resultado lógico. Ningún Hydrator deberá depender de reflection, SQL parsing o búsquedas abiertas de metadata para descubrir esa semántica mientras procesa filas.**

Por tanto:

```text id="7h3am8"
HydrationPlan
≠
QueryPlan
```

```text id="z1iz10"
HydrationPlan
≠
ExecutionPlan
```

```text id="t1t4n6"
HydrationPlan
≠
EntityMetadata
```

```text id="iy63k9"
HydrationPlan
≠
Result
```

---

# 3. Problema arquitectónico

Supongamos una consulta conceptual:

```text id="9366e5"
User
+
Orders
+
COUNT(Payments)
```

El SQL Compiler puede producir columnas físicas:

```text id="ofnazp"
c0
c1
c2
c3
c4
c5
c6
```

El Hydrator no debería descubrir dinámicamente:

```text id="enl72p"
c0 = User.id
c1 = User.name
c2 = Order.id
...
```

por introspección tardía.

Debe recibir un plan parecido a:

```text id="5ctim8"
Root:
  Entity User

Identifier:
  ResultSlot 0
  → User.id
  → int

Fields:
  Slot 1
  → User.name
  → string

Relationship:
  User.orders
  → Order node

Aggregate:
  Slot 6
  → paymentCount
  → int
```

---

# 4. Posición arquitectónica

```text id="jry2gi"
Entity Query / Projection Definition
              │
              ▼
         Query Model
              │
              ▼
      Semantic Query Engine
              │
              ▼
         Query Planner
              │
              ▼
          SQL Compiler
              │
              ├──────────────┐
              ▼              │
        Compiled Query       │
                             │
              ┌──────────────┘
              ▼
       Result Shape Metadata
              │
              ▼
┌────────────────────────────────┐
│ Hydration Plan System          │
│                                │
│ Definition                     │
│ Normalization                  │
│ Validation                     │
│ Optimization                   │
│ Compilation                    │
└───────────────┬────────────────┘
                │
                ▼
       CompiledHydrationPlan
                │
                ▼
        ResultHydrator
                │
        ┌───────┼────────┐
        ▼       ▼        ▼
      Entity  Scalar   Tuple/DTO
     Hydrator Hydrator  Hydrator
```

---

# 5. HydrationPlan ≠ QueryPlan

`QueryPlan` responde:

```text id="qtddut"
¿Cómo debe ejecutarse la consulta?
```

`HydrationPlan` responde:

```text id="ka8q8o"
¿Cómo debe interpretarse su resultado?
```

---

# 6. HydrationPlan ≠ Result Layout únicamente

El layout físico es solo una parte.

Un plan completo también contiene:

```text id="o5qolq"
logical shape
type conversion
identity semantics
construction
assignment
relationship assembly
grouping
deduplication
cardinality
streaming constraints
```

---

# 7. HydrationPlan ≠ EntityMetadata

`EntityMetadata` describe una entidad globalmente.

`HydrationPlan` describe qué parte de esa metadata participa en una consulta concreta.

---

# 8. Ejemplo

`User` puede tener 20 campos.

Consulta:

```text id="n3fgt8"
SELECT User.id, User.name
```

no requiere un plan de 20 campos si la estrategia es projection.

---

# 9. Objetivos

El sistema deberá proporcionar:

1. definición tipada del resultado;
2. bindings físicos-lógicos;
3. normalización;
4. validación;
5. compilación;
6. caching;
7. fingerprinting;
8. optimización;
9. plan sharing;
10. integración con metadata;
11. soporte para entities;
12. scalars;
13. tuples;
14. projections;
15. DTOs;
16. relationships;
17. grouping;
18. deduplication;
19. cardinality;
20. streaming;
21. diagnostics;
22. extensibilidad;
23. seguridad;
24. runtime persistente.

---

# 10. No objetivos

Hydration Plan System no:

```text id="vx2vrg"
executes SQL
executes hydration itself
manages IdentityMap
manages UnitOfWork
opens cursors
owns connections
begins transactions
serializes results
```

---

# 11. Arquitectura de fases

```text id="7i25hl"
HydrationPlanDefinition
        │
        ▼
Plan Normalization
        │
        ▼
Semantic Validation
        │
        ▼
Result Layout Binding
        │
        ▼
Hydration Strategy Resolution
        │
        ▼
Plan Optimization
        │
        ▼
Plan Compilation
        │
        ▼
Plan Fingerprinting
        │
        ▼
CompiledHydrationPlan
        │
        ▼
Hydration Plan Cache
```

---

# 12. HydrationPlanDefinition

Es la representación declarativa antes de compilación.

```php id="hzkl3o"
final readonly class HydrationPlanDefinition
{
    public function __construct(
        public HydrationShapeDefinition $shape,
        public ResultLayoutDefinition $resultLayout,
        public HydrationCardinalityDefinition $cardinality,
        public HydrationStreamingDefinition $streaming,
    ) {}
}
```

---

# 13. Definition ≠ Compiled Plan

La Definition puede contener:

```text id="18p6mf"
logical field references
metadata identifiers
logical aliases
high-level strategy declarations
```

El plan compilado contendrá:

```text id="6jo5oj"
physical result indexes
compiled converters
compiled field writers
resolved metadata handles
precomputed identity readers
fast alias lookup
```

---

# 14. Hydration Shape

Debe representar la forma del resultado.

```php id="99h2ra"
sealed interface HydrationShapeDefinition
{
}
```

Implementaciones conceptuales:

```text id="oiad7y"
EntityHydrationShape
EntityCollectionHydrationShape
ScalarHydrationShape
ScalarCollectionHydrationShape
TupleHydrationShape
ProjectionHydrationShape
DtoHydrationShape
MixedHydrationShape
```

---

# 15. Shape tree

Un shape puede formar un árbol:

```text id="yktbnr"
Tuple
├── Entity(User)
├── Scalar(orderCount)
└── DTO(BillingSummary)
```

---

# 16. Shape node identity

Cada nodo deberá tener:

```text id="8pjejz"
HydrationNodeId
```

estable dentro del plan.

---

# 17. Why node IDs

Permiten:

- diagnostics;
- telemetry;
- relation edges;
- cache fingerprints;
- error paths.

---

# 18. EntityHydrationNodeDefinition

```php id="fqz34n"
final readonly class EntityHydrationNodeDefinition
{
    public function __construct(
        public HydrationNodeId $id,
        public EntityMetadataId $entity,
        public IdentifierHydrationDefinition $identifier,
        public array $fields,
        public array $relationships,
        public HydrationCompleteness $completeness,
    ) {}
}
```

---

# 19. ScalarHydrationNodeDefinition

```php id="ovxwf9"
final readonly class ScalarHydrationNodeDefinition
{
    public function __construct(
        public HydrationNodeId $id,
        public ResultSlotReference $source,
        public TypeId $type,
        public bool $nullable,
    ) {}
}
```

---

# 20. TupleHydrationNodeDefinition

```php id="9uxzrt"
final readonly class TupleHydrationNodeDefinition
{
    public function __construct(
        public HydrationNodeId $id,
        public array $slots,
    ) {}
}
```

---

# 21. Result Layout

El plan necesita correlacionarse con el layout generado por compilación.

```php id="k6ovgf"
final readonly class ResultLayoutDefinition
{
    /**
     * @param list<ResultColumnDefinition> $columns
     */
    public function __construct(
        public array $columns,
    ) {}
}
```

---

# 22. ResultColumnDefinition

```php id="culsyr"
final readonly class ResultColumnDefinition
{
    public function __construct(
        public ResultColumnId $id,
        public int $physicalIndex,
        public string $physicalAlias,
        public ?TypeId $databaseType,
    ) {}
}
```

---

# 23. Logical alias ≠ physical alias

Ejemplo:

```text id="4yw55q"
logical:
user.id

physical:
vs_c17
```

El usuario no necesita conocer `vs_c17`.

---

# 24. Physical aliases

Pueden ser generados para:

```text id="akldux"
collision avoidance
compiler determinism
driver compatibility
```

---

# 25. ResultSlotBinding

```php id="8o8cd3"
final readonly class ResultSlotBinding
{
    public function __construct(
        public ResultColumnId $column,
        public int $physicalIndex,
        public HydrationNodeId $targetNode,
        public HydrationMemberId $targetMember,
    ) {}
}
```

---

# 26. Column binding invariant

Cada required logical member deberá tener:

```text id="2l0iqn"
exactly one valid source binding
```

salvo estrategias expresamente compuestas.

---

# 27. Duplicate binding

Debe fallar durante validación.

---

# 28. Missing binding

Igualmente.

---

# 29. Identifier Hydration Plan

Una entidad administrada necesita identificador completo.

```php id="5vjs53"
final readonly class IdentifierHydrationDefinition
{
    public function __construct(
        public array $components,
        public IdentityDomainType $identityDomain,
    ) {}
}
```

---

# 30. Identifier component

Cada componente define:

```text id="g6s1lu"
ResultSlot
Type
Canonicalizer
Identity member
Nullability
```

---

# 31. Composite IDs

Orden lógico será estable.

Ejemplo:

```text id="3dazhu"
country
number
```

No dependerá del orden accidental del SQL.

---

# 32. Identifier completeness

Para entidades normales:

```text id="7uzj11"
AllIdentifierComponentsBound
=
true
```

---

# 33. Optional JOIN entity

En un `LEFT JOIN`, el plan deberá saber cómo determinar ausencia.

---

# 34. Absence policy

Ejemplo:

```php id="23fpfn"
enum JoinedEntityAbsencePolicy
{
    case ALL_IDENTIFIER_COMPONENTS_NULL;
    case CUSTOM;
}
```

---

# 35. Partial composite identity

Si:

```text id="cchzzo"
idA = 10
idB = NULL
```

y el identificador no permite tal estado:

```text id="1mpf6s"
PartialIdentifierHydrationException
```

---

# 36. Type Conversion Plan

Cada binding deberá resolver su converter antes del hot path.

```php id="vampml"
final readonly class CompiledValueConversionPlan
{
    public function __construct(
        public TypeId $type,
        public ValueConverter $converter,
        public ConversionOptions $options,
    ) {}
}
```

---

# 37. Type converter lookup

No:

```text id="qc8nt3"
TypeRegistry.lookup()
```

por cada row si puede resolverse durante compilación.

---

# 38. Platform context

Algunos converters pueden depender de:

```text id="fyd3db"
platform representation
timezone policy
numeric policy
```

pero no de mutable request data arbitrario.

---

# 39. Field Hydration Plan

```php id="5czcpq"
final readonly class CompiledFieldHydrationPlan
{
    public function __construct(
        public FieldId $field,
        public int $resultIndex,
        public ValueConverter $converter,
        public EntityFieldWriter $writer,
        public bool $required,
    ) {}
}
```

---

# 40. Writer resolution

Durante compilación se selecciona:

```text id="x5f8xy"
constructor argument
property writer
generated accessor
reflection accessor
setter
custom
```

---

# 41. Reflection

Puede participar en plan compilation.

No deberá dominar el row-processing hot path.

---

# 42. Constructor Plan

```php id="912k1s"
final readonly class CompiledEntityConstructionPlan
{
    public function __construct(
        public EntityConstructionStrategy $strategy,
        public EntityInstantiator $instantiator,
        public array $arguments,
    ) {}
}
```

---

# 43. DTO Construction Plan

Separado de Entity construction.

```php id="zyia9q"
final readonly class CompiledDtoHydrationPlan
{
    public function __construct(
        public string $dtoClass,
        public DtoInstantiator $instantiator,
        public array $arguments,
    ) {}
}
```

---

# 44. Projection Plan

Puede tener una estructura más ligera.

---

# 45. Relationship Hydration Plan

```php id="zhdy1v"
final readonly class RelationshipHydrationPlan
{
    public function __construct(
        public RelationshipId $relationship,
        public HydrationNodeId $owner,
        public HydrationNodeId $related,
        public RelationshipHydrationMode $mode,
        public RelationshipResultCompleteness $completeness,
    ) {}
}
```

---

# 46. Relationship mode

Ejemplos:

```text id="qfof3k"
TO_ONE_JOINED
TO_MANY_JOINED
REFERENCE_ONLY
DEFERRED
```

---

# 47. Relationship Plan ≠ Relationship Loader

El plan solo describe lo que puede ensamblarse con el `Result` actual.

No ejecuta queries adicionales.

---

# 48. Completeness metadata

Crítica para:

```text id="03zsbe"
filtered eager joins
partial relation loading
```

---

# 49. Rule

```text id="87h24r"
RelationshipRowsObserved
```

no permiten inferir:

```text id="a6a0ex"
RelationshipComplete
```

sin plan semántico.

---

# 50. Grouping Plan

```php id="z8191c"
final readonly class HydrationGroupingPlan
{
    public function __construct(
        public HydrationGroupingStrategy $strategy,
        public array $keyNodes,
        public bool $requiresStableOrdering,
    ) {}
}
```

---

# 51. Grouping strategies

```text id="tnx347"
NONE
ROOT_ENTITY
SELECTED_COMPONENTS
FULL_TUPLE
CUSTOM
```

---

# 52. Grouping key

Determina:

```text id="wrumbv"
which physical rows
belong to one logical result
```

---

# 53. Grouping ≠ Deduplication

Debe preservarse permanentemente:

```text id="xei6bc"
Grouping
≠
Deduplication
```

---

# 54. Deduplication Plan

```php id="j015cy"
final readonly class HydrationDeduplicationPlan
{
    public function __construct(
        public HydrationDeduplicationStrategy $strategy,
        public array $keyNodes,
    ) {}
}
```

---

# 55. Deduplication strategies

```text id="w0bl2f"
NONE
ROOT_ENTITY_IDENTITY
FULL_TUPLE
SELECTED_COMPONENTS
STRUCTURAL
CUSTOM
```

---

# 56. Logical result key compiler

El plan podrá incluir:

```text id="khm9km"
LogicalResultKeyFactory
```

precompilada.

---

# 57. Entity key component

Usará:

```text id="6b7rom"
EntityKey
```

no serialización profunda del objeto.

---

# 58. Scalar key component

Usará forma canonicalizada.

---

# 59. Cardinality Plan

```php id="pny1q7"
final readonly class HydrationCardinalityPlan
{
    public function __construct(
        public ResultCardinalityExpectation $expectation,
    ) {}
}
```

---

# 60. Cardinality is logical

Nunca se valida contra:

```text id="a5hwfg"
physical row count
```

directamente cuando hay grouping.

---

# 61. Streaming Plan

```php id="jzd6vq"
final readonly class HydrationStreamingPlan
{
    public function __construct(
        public HydrationStreamingMode $mode,
        public bool $requiresOrderedGroups,
        public HydrationFinalizationStrategy $finalization,
    ) {}
}
```

---

# 62. Streaming modes

```text id="23sdvu"
BUFFERED_ONLY
STREAMABLE
STREAMABLE_IF_ORDERED
CUSTOM
```

---

# 63. Finalization strategy

Determina cuándo:

```text id="mavrik"
logical result
```

puede exponerse.

---

# 64. Safe streaming condition

```text id="6ieyib"
MayYield(result)
=
ResultComplete
∧
PlanCanProveNoFutureExtension
```

---

# 65. Result completion plan

Puede depender de:

```text id="3a9cbe"
root identity transition
EOF
known row boundary
scalar row completion
custom grouping boundary
```

---

# 66. Lifecycle finalization

Para entities con eager graph:

```text id="wu4ogg"
postLoad
```

deberá coordinarse con un punto de finalización semánticamente seguro.

---

# 67. LifecyclePlan

Podrá existir:

```php id="0ff5xg"
final readonly class HydrationLifecyclePlan
{
    public function __construct(
        public bool $dispatchPostLoad,
        public LifecycleFinalizationPoint $finalizationPoint,
    ) {}
}
```

---

# 68. Critical issue

Si se dispara `postLoad` inmediatamente en la primera row de:

```text id="tcjdte"
User#1 Order#10
User#1 Order#11
User#1 Order#12
```

el callback podría observar una colección incompleta.

---

# 69. Recommended architecture

Distinguir:

```text id="iyg6wg"
EntityMaterialized
```

de:

```text id="ld72yd"
EntityGraphFinalized
```

---

# 70. Materialization

Significa:

```text id="26ikhw"
identity resolved
fields initialized
entity registered
```

---

# 71. Graph finalization

Significa:

```text id="huotn7"
all relationship data promised by current result
has been assembled
```

---

# 72. postLoad strategy

Para entidades sin joined to-many:

```text id="ctwps8"
after materialization
```

puede ser suficiente.

Para graph hydration:

```text id="nrkj9h"
after graph finalization
```

es más seguro.

---

# 73. Plan must decide

No deberá decidirse mediante heurística dentro de `EntityHydrator`.

---

# 74. Partial Entity Plan

Aunque el documento 139 del master original estaba reservado a partial entities, el plan deberá soportarlas arquitectónicamente.

```php id="iptn4a"
final readonly class PartialEntityHydrationPlan
{
    public function __construct(
        public EntityMetadataId $entity,
        public LoadedFieldMask $loadedFields,
        public PartialEntityHydrationPolicy $policy,
    ) {}
}
```

---

# 75. LoadedFieldMask

Será precomputable.

Ejemplo:

```text id="g89pc4"
id         loaded
name       loaded
email      missing
createdAt  missing
```

---

# 76. Missing ≠ NULL

El plan deberá mantener esta distinción incluso si el objeto PHP contiene propiedades no inicializadas.

---

# 77. Read-only Plan

Podrá incluir:

```text id="nmwd7a"
HydrationTrackingMode
```

---

# 78. Tracking modes

```text id="fz8q69"
MANAGED
READ_ONLY_MANAGED
UNMANAGED
```

---

# 79. Tracking mode affects plan

Puede cambiar:

- IdentityMap behavior;
- snapshot requirement;
- UoW registration;
- lifecycle;
- memory behavior.

---

# 80. Existing Entity Policy

El plan podrá incorporar:

```text id="62o2if"
REUSE_ONLY
INITIALIZE_MISSING
REFRESH
REJECT_CONFLICT
```

---

# 81. Refresh plan

No debe reutilizar accidentalmente el plan de una consulta normal si las semánticas difieren.

---

# 82. Plan context separation

Algunas decisiones son estructurales.

Otras son runtime policy.

Debe evitarse compilar valores request-specific innecesariamente.

---

# 83. Structural vs contextual plan data

## Structural

```text id="v6cwhl"
Result indexes
Entity metadata IDs
field bindings
type converters
relationship nodes
tuple slots
constructor plans
```

## Contextual

```text id="733tvs"
PersistenceContext
tenant identity namespace
request cancellation
current deadline
EntityManager
```

---

# 84. Critical invariant

```text id="m4fjjp"
CompiledHydrationPlan
```

no deberá capturar mutable runtime context.

---

# 85. Plan Normalization

Antes de validar se deberá convertir distintas representaciones equivalentes a una forma canónica.

---

# 86. Example

Input logical reference:

```text id="8kvber"
User.email
```

puede normalizarse a:

```text id="px4uva"
EntityMetadataId(17)
FieldId(8)
```

---

# 87. Normalization goals

- eliminar aliases ambiguos;
- resolver metadata IDs;
- ordenar componentes determinísticamente;
- resolver defaults;
- canonicalizar type IDs;
- producir shapes comparables.

---

# 88. Deterministic normalization

Mismo input semántico deberá producir:

```text id="7v5pvu"
same normalized representation
```

independientemente del orden no significativo de configuración.

---

# 89. Validation

Debe ocurrir antes de hot path.

---

# 90. Validation categories

```text id="nfphwc"
structural
metadata
type
identity
result layout
construction
relationship
grouping
deduplication
cardinality
streaming
lifecycle
runtime shareability
```

---

# 91. Structural validation

Comprueba:

```text id="j1nejn"
duplicate node IDs
cycles where forbidden
missing root
invalid slot references
duplicate aliases
```

---

# 92. Metadata validation

Comprueba:

```text id="kc5i65"
EntityMetadata exists
Field exists
Relationship exists
Identifier compatible
```

---

# 93. Type validation

Comprueba:

```text id="obg607"
converter exists
PHP target compatible
nullability compatible
```

---

# 94. Identity validation

Para entity nodes managed:

```text id="hjyc20"
identifier complete
identity domain known
namespace source valid
```

---

# 95. Result layout validation

Comprueba:

```text id="yd7dlw"
expected column exists
physical index valid
alias mapping unambiguous
```

---

# 96. Construction validation

Comprueba:

```text id="4qls52"
constructor/factory callable
arguments available
readonly rules valid
```

---

# 97. Relationship validation

Comprueba:

```text id="2veibd"
owner node exists
related node exists
mapping compatible
collection semantics known
```

---

# 98. Grouping validation

Si:

```text id="7porl8"
ordered streaming
```

requiere:

```text id="ve10ql"
stable root ordering guarantee
```

---

# 99. Dedup validation

No podrá utilizarse un component como key si no tiene semántica de key estable.

---

# 100. Cardinality validation

Ejemplo:

```text id="g0pw1v"
SCALAR
+
MANY
```

puede ser inválido salvo scalar collection.

---

# 101. Lifecycle validation

Un `postLoad after graph finalization` debe tener un mecanismo de graph finalization disponible.

---

# 102. Fail fast

Preferencia:

```text id="gg9kv7"
bootstrap/query compilation
```

sobre:

```text id="3w5hcu"
row 500,000
```

---

# 103. Plan Optimization

Después de validar:

```text id="iv8s0j"
NormalizedPlan
→ OptimizedPlan
```

---

# 104. Optimization goals

Reducir:

- dynamic lookup;
- reflection;
- alias resolution;
- object allocations;
- branches;
- metadata traversal.

---

# 105. Column index precomputation

En vez de:

```php id="apswk4"
$row['user_name'];
```

puede compilarse:

```php id="bh47i9"
$row[3];
```

---

# 106. Alias map

Para APIs que lo necesiten:

```text id="868o8m"
alias
→ slot index
```

se compila una vez.

---

# 107. Converter specialization

Puede seleccionarse:

```text id="9yrdag"
FastIntConverter
UuidConverter
DecimalConverter
DateTimeConverter
```

antes del hot path.

---

# 108. Field writer specialization

Igualmente.

---

# 109. Identity reader specialization

Composite IDs pueden recibir:

```text id="bmc1z9"
CompiledCompositeIdentifierReader
```

---

# 110. Branch elimination

Si un field es:

```text id="y43v2r"
NOT NULL
```

y el driver contract garantiza representation válida, algunos checks podrán moverse fuera del hot path, siempre sin sacrificar seguridad.

---

# 111. Optimization ≠ semantic weakening

Nunca:

```text id="exq70e"
faster
```

a costa de:

```text id="m3md5m"
precision
identity
nullability
completeness
```

---

# 112. Compilation

La compilación produce un objeto ejecutable inmutable.

```php id="pde13w"
interface HydrationPlanCompiler
{
    public function compile(
        NormalizedHydrationPlan $plan,
        HydrationCompilationContext $context,
    ): CompiledHydrationPlan;
}
```

---

# 113. Compilation Context

Puede incluir:

```text id="46uhdh"
MetadataGeneration
TypeRegistryGeneration
PlatformCapabilities
ResultLayout
CompilerVersion
```

---

# 114. No request-scoped services

No deberá contener:

```text id="h2hquw"
current EntityManager
current user
current tenant object
HTTP request
```

---

# 115. CompiledHydrationPlan

Contrato base:

```php id="esvohs"
interface CompiledHydrationPlan
{
    public function id(): HydrationPlanId;

    public function shape(): HydrationShape;

    public function fingerprint(): HydrationPlanFingerprint;
}
```

---

# 116. Specialized plans

```text id="170bz7"
CompiledEntityHydrationPlan
CompiledScalarHydrationPlan
CompiledTupleHydrationPlan
CompiledProjectionHydrationPlan
CompiledDtoHydrationPlan
CompiledResultHydrationPlan
```

---

# 117. Composition

Un `CompiledResultHydrationPlan` puede contener referencias inmutables a planes especializados.

---

# 118. Plan immutability

Después de compilación:

```text id="k5hpbd"
PlanMutation
=
forbidden
```

---

# 119. Why immutability

Permite:

```text id="6gn5rq"
safe sharing
cache reuse
deterministic behavior
worker persistence
simple invalidation
```

---

# 120. Fingerprinting

Cada plan deberá tener una huella estable.

```php id="99xlzl"
final readonly class HydrationPlanFingerprint
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 121. Fingerprint inputs

Puede incluir:

```text id="82y301"
normalized shape
result layout
metadata generation
type registry generation
construction strategies
relationship semantics
grouping
deduplication
cardinality
streaming requirements
compiler version
```

---

# 122. Fingerprint ≠ runtime identity

No incluir:

```text id="dqneyd"
EntityManager object ID
request ID
current tenant object
current transaction
```

---

# 123. Deterministic fingerprint

Mismo plan semántico deberá producir:

```text id="w5hvi3"
same fingerprint
```

bajo misma generación de dependencias.

---

# 124. Cache key

```text id="34tjpf"
HydrationPlanCacheKey
=
HydrationPlanFingerprint
+
MetadataGeneration
+
TypeRegistryGeneration
+
HydrationCompilerVersion
```

cuando esos elementos no estén ya embebidos en fingerprint.

---

# 125. Platform generation

Puede incluirse si el plan depende de representación específica de plataforma.

---

# 126. Plan cache

Será cubierto con mayor profundidad por:

```text id="mrcygm"
141_DATABASE_HYDRATION_CACHE_SYSTEM.md
```

pero este documento establece los contratos.

---

# 127. Cacheable condition

```text id="2zepjc"
Cacheable(plan)
=
Immutable
∧
NoScopedState
∧
Deterministic
∧
DependencyGenerationsKnown
```

---

# 128. Non-cacheable plan

Puede existir para:

```text id="dzn0f4"
custom runtime transformation
```

pero deberá marcarse explícitamente.

---

# 129. Cacheability metadata

```php id="47ymq2"
enum HydrationPlanCacheability
{
    case CACHEABLE;
    case REQUEST_SCOPED;
    case NON_CACHEABLE;
}
```

---

# 130. Cacheability default

Para planes ORM estándar:

```text id="hh8nu0"
CACHEABLE
```

deberá ser objetivo arquitectónico.

---

# 131. Metadata generations

El plan nunca deberá asumir que EntityMetadata permanece eternamente igual.

---

# 132. MetadataGeneration

Cada compilación de metadata puede incrementar:

```text id="t3y5ye"
MetadataGenerationId
```

---

# 133. Type generation

Igualmente:

```text id="b57bt5"
TypeRegistryGenerationId
```

---

# 134. Extension generation

Puede existir:

```text id="p36w47"
HydrationExtensionGenerationId
```

si extensiones afectan el plan.

---

# 135. Stale plan detection

Si:

```text id="z4qn8r"
plan.metadataGeneration
≠
current.metadataGeneration
```

deberá:

```text id="j4vtwn"
miss cache
```

o rechazar plan.

---

# 136. Never use stale plan silently

Especialmente peligroso si:

```text id="wlwawp"
field order changed
mapping changed
identifier changed
type converter changed
```

---

# 137. Plan compatibility

Puede existir:

```php id="2r6f0t"
interface HydrationPlanCompatibilityChecker
{
    public function isCompatible(
        CompiledHydrationPlan $plan,
        HydrationRuntimeDescriptor $runtime,
    ): bool;
}
```

---

# 138. Compatibility ≠ equality

Un plan puede seguir siendo compatible aunque ciertos metadata externos no relevantes hayan cambiado.

Optimización futura.

---

# 139. V1 policy

Preferir invalidación conservadora basada en generaciones.

---

# 140. Plan Explainability

Debe poder explicarse un plan.

---

# 141. explain()

Conceptualmente:

```php id="n85fso"
$plan->explain();
```

o mediante diagnostics service.

---

# 142. Example

```text id="8njvo1"
Hydration Plan hp_7DF91

Shape:
  ENTITY_COLLECTION<User>

Root:
  Entity User

Identity:
  user.id
  ← result[0]
  converter: Int64

Fields:
  User.name
  ← result[1]

Relationship:
  User.orders
  ← Order node

Grouping:
  ROOT_ENTITY

Deduplication:
  ROOT_ENTITY_IDENTITY

Streaming:
  STREAMABLE_IF_ORDERED

Lifecycle:
  postLoad AFTER_GRAPH_FINALIZATION
```

---

# 143. Explainability purpose

Ayuda a detectar:

```text id="xjnq7h"
wrong aliases
unexpected partial hydration
JOIN amplification risk
streaming restrictions
incorrect collection completeness
```

---

# 144. Diagnostics levels

```text id="4u3x91"
SUMMARY
STANDARD
DETAILED
```

---

# 145. Production diagnostics

No deberán revelar datos concretos del Result.

---

# 146. Security

El Hydration Plan System se construye desde metadata confiable.

---

# 147. No arbitrary class instantiation

DTO/entity classes deberán ser resueltas durante compilación.

---

# 148. No row-driven method calls

Los datos DB no podrán seleccionar:

```text id="68p5jm"
method
service
class
converter
hydrator
```

arbitrariamente.

---

# 149. Discriminator exception

Un discriminator DB puede seleccionar un tipo solo dentro de:

```text id="hohr6h"
compiled whitelist
```

---

# 150. Generated code

Si se generan hydrators PHP:

```text id="lsegdb"
metadata
→ validated AST/code template
→ generated PHP
```

Nunca:

```text id="h1ctzy"
database value
→ PHP source code
```

---

# 151. Extension System

Extensiones podrán participar mediante contribuciones estructuradas.

---

# 152. HydrationPlanContributor

```php id="mj751h"
interface HydrationPlanContributor
{
    public function contribute(
        HydrationPlanBuilder $builder,
        HydrationPlanContributionContext $context,
    ): void;
}
```

---

# 153. Contribution timing

Solo durante:

```text id="ieo0g5"
plan construction / compilation
```

No por row.

---

# 154. Extension constraints

Una extensión no deberá:

- capturar request state;
- generar SQL;
- ejecutar queries;
- resolver services arbitrarios por row;
- romper IdentityMap semantics.

---

# 155. Contribution collisions

Dos extensiones que modifiquen el mismo binding deberán:

```text id="c1ii5v"
fail
```

o usar un resolution protocol explícito.

---

# 156. No last-write-wins silencioso

Regla general.

---

# 157. Plan Builder

```php id="sfiurb"
interface HydrationPlanBuilder
{
    public function entity(...): HydrationNodeId;

    public function scalar(...): HydrationNodeId;

    public function tuple(...): HydrationNodeId;

    public function relationship(...): void;

    public function build(): HydrationPlanDefinition;
}
```

---

# 158. Builder mutable

Durante construcción puede ser mutable.

---

# 159. Built definition immutable

Al llamar:

```text id="8x8ex8"
build()
```

se produce una Definition inmutable.

---

# 160. Builder ≠ Plan

Evitar exponer mutable builder como runtime plan.

---

# 161. Automatic plan generation

En la mayoría de consultas ORM:

```text id="c07i85"
developer
```

no construirá el plan manualmente.

---

# 162. Typical flow

```text id="6cn1sh"
EntityQuery
↓
semantic result description
↓
HydrationPlanFactory
↓
Definition
↓
Compiler
```

---

# 163. Manual plan API

Puede existir para:

```text id="q30cl5"
advanced raw query integrations
```

pero deberá ser fuertemente tipada.

---

# 164. Raw SQL integration

Si el usuario ejecuta SQL raw y quiere entity hydration, deberá proporcionar:

```text id="owc9sv"
explicit result mapping
```

No se inferirá de nombres de columnas libremente.

---

# 165. Raw mapping example

```php id="fg2akm"
$result = DB::raw($sql)
    ->mapEntity(User::class, [
        'u_id' => 'id',
        'u_name' => 'name',
    ]);
```

Internamente:

```text id="98bybw"
mapping
→ HydrationPlanDefinition
```

---

# 166. Raw entity hydration safety

Debe requerir identificador válido y metadata compatible.

---

# 167. No magic guessing

No:

```text id="i2n9se"
column "id"
→ probably User.id
```

sin mapping explícito.

---

# 168. Plan versioning

Puede existir:

```text id="ffqdoa"
HydrationPlanFormatVersion
```

para soporte de cache persistente futuro.

---

# 169. Compiled plan serialization

V1 puede mantener planes en memoria.

Una versión posterior podría persistir compilados.

---

# 170. Serialization requirement

Si se implementa:

```text id="i6f92f"
SerializedCompiledHydrationPlan
```

deberá ser:

- versionado;
- checksumed;
- generation-bound;
- safe to deserialize.

---

# 171. No serialized services

No deberán persistirse instancias de servicios runtime dentro del plan serializado.

---

# 172. Persistent Runtime

La arquitectura está especialmente diseñada para:

```text id="pqna1u"
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 173. Worker-level cache

Podrá almacenar:

```text id="5xzvxt"
CompiledHydrationPlan
```

entre requests.

---

# 174. Request-level context

Se pasa separadamente:

```text id="1b7v16"
HydrationContext
```

---

# 175. Fundamental separation

```text id="2ui1zk"
Shared:
CompiledHydrationPlan

Scoped:
HydrationContext
HydrationScope
PersistenceContext
IdentityMap
EntityManager
```

---

# 176. Thread/coroutine safety

Compiled plans podrán ser concurrency-safe por ser inmutables.

---

# 177. Builder/compiler concurrency

Dependerá de implementación.

No se asumirá mutable builder compartible.

---

# 178. Reset

No hay nada que resetear dentro de un plan compilado inmutable.

---

# 179. This is desirable

Reduce significativamente riesgo de state leaks.

---

# 180. Memory Model

El plan debe ser mucho más pequeño que:

```text id="a691ew"
hydrated result
```

y no mantener referencias a entidades.

---

# 181. Memory components

```text id="w4dq3t"
PlanMemory
≈
Node definitions
+
Slot bindings
+
Converter references
+
Writer references
+
Grouping metadata
+
Alias maps
```

---

# 182. Sharing effect

Si 10,000 requests usan la misma query shape:

```text id="36k0vi"
one compiled plan
```

puede reutilizarse.

---

# 183. Performance

Sin plan compilado:

```text id="9ewqu2"
row
→ resolve alias
→ lookup metadata
→ lookup type
→ lookup writer
→ reflection
→ hydrate
```

---

# 184. With compiled plan

```text id="erzte3"
row[index]
→ converter
→ writer
```

---

# 185. Approximation

Para `n` rows y `f` fields:

```text id="h05q60"
WithoutCompilation
≈
O(n × f × dynamicResolutionCost)
```

Mientras:

```text id="7buu4j"
WithCompilation
≈
O(n × f)
```

con constantes mucho menores.

---

# 186. Compilation cost

La compilación puede ser más costosa:

```text id="stxxep"
C_compile
```

pero amortizada:

```text id="p8umyp"
TotalCost
=
C_compile
+
N × C_hydrate
```

---

# 187. Cache benefit

Para plan reutilizado:

```text id="fr3j78"
TotalCost
≈
C_compile once
+
N_requests × C_hydrate
```

---

# 188. Compilation threshold

No se necesita una heurística compleja en V1.

Compilar siempre los planes estándar es aceptable.

---

# 189. Development mode

Podrá invalidarse con mayor frecuencia para DX.

---

# 190. Production mode

Cache agresivo de immutable plans.

---

# 191. Telemetry

Métricas propuestas:

```text id="cf34ju"
orm.hydration_plan.created
orm.hydration_plan.normalized
orm.hydration_plan.validated
orm.hydration_plan.compiled

orm.hydration_plan.compile.duration
orm.hydration_plan.validation.duration

orm.hydration_plan.cache.hit
orm.hydration_plan.cache.miss

orm.hydration_plan.nodes
orm.hydration_plan.entity_nodes
orm.hydration_plan.scalar_nodes
orm.hydration_plan.tuple_nodes
orm.hydration_plan.relationship_nodes

orm.hydration_plan.invalid
orm.hydration_plan.stale
orm.hydration_plan.recompiled
```

---

# 192. High-cardinality telemetry

No usar:

```text id="ub1cod"
plan fingerprint
```

como metric label si puede producir alta cardinalidad.

Puede aparecer en trace/debug metadata controlada.

---

# 193. Plan complexity

Podrá calcularse:

```text id="2imta3"
PlanComplexity
=
nodes
+
bindings
+
relationships
+
grouping components
```

---

# 194. Debug Toolbar

Ejemplo:

```text id="bs3ifh"
Hydration Plan
─────────────────────────────────

Plan:                 hp_31AF
Shape:                ENTITY_COLLECTION<User>

Nodes:
  Entity:             2
  Scalar:             1
  Relationship:       1

Bindings:             11

Root:
  User

Grouping:
  ROOT_ENTITY

Deduplication:
  ROOT_ENTITY_IDENTITY

Streaming:
  STREAMABLE_IF_ORDERED

Lifecycle:
  postLoad:
    AFTER_GRAPH_FINALIZATION

Compiled:
  YES

Cache:
  HIT

Metadata Generation:
  84

Type Generation:
  11
```

---

# 195. Testing Strategy

Debe existir una suite específica del plan system.

---

# 196. Definition test

Construcción produce shape correcto.

---

# 197. Normalization test

Dos definitions semánticamente equivalentes producen forma canónica equivalente.

---

# 198. Deterministic normalization test

Orden de metadata no significativa no cambia fingerprint.

---

# 199. Entity binding test

Field binding correcto.

---

# 200. Missing field binding test

Falla temprano.

---

# 201. Duplicate field binding test

Falla temprano.

---

# 202. Composite ID test

Componentes quedan ordenados según metadata de identidad.

---

# 203. Partial ID test

Plan inválido es rechazado.

---

# 204. LEFT JOIN absence plan test

Genera política correcta.

---

# 205. Type converter test

Converter queda pre-resuelto.

---

# 206. Decimal converter test

Preserva precision policy.

---

# 207. Field writer test

Writer queda compilado.

---

# 208. Constructor plan test

Constructor arguments quedan ligados correctamente.

---

# 209. Readonly mapping test

Configuraciones inválidas fallan antes de hydration.

---

# 210. DTO plan test

Constructor/factory validado.

---

# 211. Tuple plan test

Slots ordenados y aliases correctos.

---

# 212. Duplicate tuple alias test

Falla.

---

# 213. Relationship plan test

Owner/related mapping correcto.

---

# 214. Filtered collection test

Completeness queda PARTIAL/UNKNOWN según semántica.

---

# 215. Grouping plan test

Root grouping generado correctamente.

---

# 216. Dedup plan test

No se confunde con grouping.

---

# 217. Cardinality plan test

Logical cardinality declarada correctamente.

---

# 218. Streaming precondition test

Plan invalida streaming si ordering no puede garantizarse.

---

# 219. Lifecycle finalization test

Joined to-many produce:

```text id="hgvqx7"
AFTER_GRAPH_FINALIZATION
```

cuando corresponda.

---

# 220. Scalar-only plan test

No incorpora IdentityMap dependencies.

---

# 221. Projection plan test

No incorpora UoW registration.

---

# 222. Read-only plan test

TrackingMode correcto.

---

# 223. Refresh plan test

No es idéntico semánticamente a normal read plan.

---

# 224. Fingerprint test

Misma semántica → mismo fingerprint.

---

# 225. Mapping change test

Cambio estructural → fingerprint/generation incompatible.

---

# 226. Type registry change test

Cambio de converter invalida plan.

---

# 227. Plan cache compatibility test

Plan stale no se reutiliza.

---

# 228. Persistent worker test

Plan compartido no contiene state del request anterior.

---

# 229. Concurrency test

Mismo compiled plan puede leerse concurrentemente.

---

# 230. Custom extension test

Contribution válida participa en fingerprint.

---

# 231. Extension conflict test

Conflicto produce error determinista.

---

# 232. Raw SQL plan test

Mapping explícito funciona sin magic inference.

---

# 233. Security test

Discriminator no permite clase fuera de whitelist.

---

# 234. Performance test

Comparar:

```text id="sp5df1"
dynamic metadata hydration
vs
compiled plan hydration
```

---

# 235. Plan compilation benchmark

Casos:

```text id="44edkd"
simple scalar
simple entity
20-field entity
composite ID
entity + relation
multi-relation graph
tuple
DTO
projection
```

---

# 236. Architectural Invariants

## DB-ORM-HYDRATION-PLAN-001

HydrationPlan describirá cómo interpretar un Result.

## DB-ORM-HYDRATION-PLAN-002

HydrationPlan será distinto de QueryPlan.

## DB-ORM-HYDRATION-PLAN-003

HydrationPlan será distinto de ExecutionPlan.

## DB-ORM-HYDRATION-PLAN-004

HydrationPlan será distinto de EntityMetadata.

## DB-ORM-HYDRATION-PLAN-005

HydrationPlan será distinto del Result físico.

## DB-ORM-HYDRATION-PLAN-006

Hydrators ejecutarán planes en vez de redescubrir semántica.

## DB-ORM-HYDRATION-PLAN-007

Hydrators no analizarán SQL para descubrir mappings.

## DB-ORM-HYDRATION-PLAN-008

Hot path evitará reflection siempre que el plan pueda precompilarse.

## DB-ORM-HYDRATION-PLAN-009

Plan Definition será distinta de Compiled Plan.

## DB-ORM-HYDRATION-PLAN-010

Plan Definition será inmutable después de build.

## DB-ORM-HYDRATION-PLAN-011

Compiled Plan será inmutable.

## DB-ORM-HYDRATION-PLAN-012

Compiled Plan será determinista respecto a sus inputs estructurales.

## DB-ORM-HYDRATION-PLAN-013

Result shape será explícito.

## DB-ORM-HYDRATION-PLAN-014

Shape nodes tendrán IDs estables dentro del plan.

## DB-ORM-HYDRATION-PLAN-015

Entity node referenciará metadata validada.

## DB-ORM-HYDRATION-PLAN-016

Scalar node tendrá type semantics explícitas.

## DB-ORM-HYDRATION-PLAN-017

Tuple node tendrá slot ordering explícito.

## DB-ORM-HYDRATION-PLAN-018

DTO node tendrá construction semantics explícitas.

## DB-ORM-HYDRATION-PLAN-019

Projection node será distinta de Entity node.

## DB-ORM-HYDRATION-PLAN-020

Logical alias será distinto de physical alias.

## DB-ORM-HYDRATION-PLAN-021

Physical result slots serán ligados explícitamente.

## DB-ORM-HYDRATION-PLAN-022

Required logical member tendrá source binding válido.

## DB-ORM-HYDRATION-PLAN-023

Duplicate source binding ambiguo fallará.

## DB-ORM-HYDRATION-PLAN-024

Missing required binding fallará.

## DB-ORM-HYDRATION-PLAN-025

Entity identifier será completamente descrito.

## DB-ORM-HYDRATION-PLAN-026

Composite ID ordering seguirá identity metadata.

## DB-ORM-HYDRATION-PLAN-027

Optional joined entity tendrá absence policy explícita.

## DB-ORM-HYDRATION-PLAN-028

Partial composite identifier inválido será rechazado.

## DB-ORM-HYDRATION-PLAN-029

Type converters serán resolubles antes del hot path.

## DB-ORM-HYDRATION-PLAN-030

Plan no duplicará Type System.

## DB-ORM-HYDRATION-PLAN-031

Field writer será precompilable.

## DB-ORM-HYDRATION-PLAN-032

Writer strategy procederá de validated metadata.

## DB-ORM-HYDRATION-PLAN-033

Constructor strategy será explícita.

## DB-ORM-HYDRATION-PLAN-034

DTO construction strategy será distinta de entity construction.

## DB-ORM-HYDRATION-PLAN-035

Readonly constraints serán validables antes de hydration.

## DB-ORM-HYDRATION-PLAN-036

Relationship plan solo describirá relaciones presentes en current result.

## DB-ORM-HYDRATION-PLAN-037

Relationship plan no ejecutará lazy queries.

## DB-ORM-HYDRATION-PLAN-038

Relationship completeness será explícita.

## DB-ORM-HYDRATION-PLAN-039

Filtered relation no será marcada COMPLETE por heurística.

## DB-ORM-HYDRATION-PLAN-040

Grouping plan será explícito.

## DB-ORM-HYDRATION-PLAN-041

Grouping será distinto de deduplication.

## DB-ORM-HYDRATION-PLAN-042

Deduplication plan será explícito.

## DB-ORM-HYDRATION-PLAN-043

Logical result key podrá precompilarse.

## DB-ORM-HYDRATION-PLAN-044

Entity components de keys usarán EntityKey.

## DB-ORM-HYDRATION-PLAN-045

Scalar key components usarán canonical scalar semantics.

## DB-ORM-HYDRATION-PLAN-046

Cardinality plan operará sobre logical results.

## DB-ORM-HYDRATION-PLAN-047

Physical row count no definirá cardinality del plan.

## DB-ORM-HYDRATION-PLAN-048

Streaming capability será explícita.

## DB-ORM-HYDRATION-PLAN-049

Streaming ordering requirements serán explícitos.

## DB-ORM-HYDRATION-PLAN-050

Plan no afirmará streamability si no puede probar completion boundaries.

## DB-ORM-HYDRATION-PLAN-051

Result finalization strategy será explícita.

## DB-ORM-HYDRATION-PLAN-052

Entity materialization será distinta de graph finalization.

## DB-ORM-HYDRATION-PLAN-053

Lifecycle finalization point será planificable.

## DB-ORM-HYDRATION-PLAN-054

Joined graph postLoad no se disparará prematuramente por diseño del plan.

## DB-ORM-HYDRATION-PLAN-055

Partial entity plan tendrá LoadedFieldMask explícito.

## DB-ORM-HYDRATION-PLAN-056

MISSING será distinto de NULL en partial plan.

## DB-ORM-HYDRATION-PLAN-057

Tracking mode será explícito cuando afecte hydration semantics.

## DB-ORM-HYDRATION-PLAN-058

MANAGED será distinto de UNMANAGED.

## DB-ORM-HYDRATION-PLAN-059

REFRESH plan podrá diferir de normal read plan.

## DB-ORM-HYDRATION-PLAN-060

Existing entity policy será explícita cuando corresponda.

## DB-ORM-HYDRATION-PLAN-061

Compiled plan no capturará PersistenceContext.

## DB-ORM-HYDRATION-PLAN-062

Compiled plan no capturará EntityManager.

## DB-ORM-HYDRATION-PLAN-063

Compiled plan no capturará IdentityMap.

## DB-ORM-HYDRATION-PLAN-064

Compiled plan no capturará current tenant object.

## DB-ORM-HYDRATION-PLAN-065

Compiled plan no capturará current transaction.

## DB-ORM-HYDRATION-PLAN-066

Plan normalization será determinista.

## DB-ORM-HYDRATION-PLAN-067

Equivalent semantic definitions producirán normalized representation equivalente.

## DB-ORM-HYDRATION-PLAN-068

Plan validation ocurrirá antes del hot path cuando sea posible.

## DB-ORM-HYDRATION-PLAN-069

Structural validation detectará duplicate node IDs.

## DB-ORM-HYDRATION-PLAN-070

Metadata validation detectará missing entity metadata.

## DB-ORM-HYDRATION-PLAN-071

Type validation detectará missing converters.

## DB-ORM-HYDRATION-PLAN-072

Identity validation detectará incomplete identifiers.

## DB-ORM-HYDRATION-PLAN-073

Result layout validation detectará invalid slots.

## DB-ORM-HYDRATION-PLAN-074

Construction validation detectará impossible constructors.

## DB-ORM-HYDRATION-PLAN-075

Relationship validation detectará invalid graph bindings.

## DB-ORM-HYDRATION-PLAN-076

Grouping validation comprobará required ordering.

## DB-ORM-HYDRATION-PLAN-077

Dedup validation comprobará stable key semantics.

## DB-ORM-HYDRATION-PLAN-078

Cardinality validation comprobará compatible result shapes.

## DB-ORM-HYDRATION-PLAN-079

Lifecycle validation comprobará finalization capability.

## DB-ORM-HYDRATION-PLAN-080

Plan optimization no debilitará semántica.

## DB-ORM-HYDRATION-PLAN-081

Column indexes podrán precomputarse.

## DB-ORM-HYDRATION-PLAN-082

Alias maps podrán precomputarse.

## DB-ORM-HYDRATION-PLAN-083

Converter references podrán pre-resolverse.

## DB-ORM-HYDRATION-PLAN-084

Field writers podrán pre-resolverse.

## DB-ORM-HYDRATION-PLAN-085

Identity readers podrán precompilarse.

## DB-ORM-HYDRATION-PLAN-086

Plan compiler será distinto de Hydrator.

## DB-ORM-HYDRATION-PLAN-087

Plan compiler no ejecutará results.

## DB-ORM-HYDRATION-PLAN-088

Compiled plan tendrá fingerprint estable.

## DB-ORM-HYDRATION-PLAN-089

Fingerprint no incluirá mutable request state.

## DB-ORM-HYDRATION-PLAN-090

Fingerprint no incluirá EntityManager object identity.

## DB-ORM-HYDRATION-PLAN-091

Plan cache key será generation-aware.

## DB-ORM-HYDRATION-PLAN-092

Stale metadata plan no se reutilizará silenciosamente.

## DB-ORM-HYDRATION-PLAN-093

Stale type registry plan no se reutilizará silenciosamente.

## DB-ORM-HYDRATION-PLAN-094

V1 podrá usar conservative generation invalidation.

## DB-ORM-HYDRATION-PLAN-095

Plan cacheability será explícita.

## DB-ORM-HYDRATION-PLAN-096

Standard ORM plans deberán aspirar a CACHEABLE.

## DB-ORM-HYDRATION-PLAN-097

Cached plan será immutable.

## DB-ORM-HYDRATION-PLAN-098

Cached plan no almacenará hydrated entities.

## DB-ORM-HYDRATION-PLAN-099

Cached plan no almacenará Result.

## DB-ORM-HYDRATION-PLAN-100

Cached plan no almacenará request-local services.

## DB-ORM-HYDRATION-PLAN-101

Plan explainability será soportada.

## DB-ORM-HYDRATION-PLAN-102

Explain output no mostrará actual row values por defecto.

## DB-ORM-HYDRATION-PLAN-103

Generated hydration code derivará de trusted metadata.

## DB-ORM-HYDRATION-PLAN-104

Database values no generarán executable PHP.

## DB-ORM-HYDRATION-PLAN-105

Discriminator resolution utilizará compiled whitelist.

## DB-ORM-HYDRATION-PLAN-106

Extensions contribuirán durante plan construction/compilation.

## DB-ORM-HYDRATION-PLAN-107

Extensions no resolverán metadata dinámicamente por row salvo contrato especializado.

## DB-ORM-HYDRATION-PLAN-108

Extension conflicts no usarán last-write-wins silencioso.

## DB-ORM-HYDRATION-PLAN-109

Plan builder será distinto de built definition.

## DB-ORM-HYDRATION-PLAN-110

Builder mutable no será runtime plan.

## DB-ORM-HYDRATION-PLAN-111

ORM normal generará planes automáticamente.

## DB-ORM-HYDRATION-PLAN-112

Manual plans serán escape hatch avanzado.

## DB-ORM-HYDRATION-PLAN-113

Raw SQL entity hydration requerirá mapping explícito.

## DB-ORM-HYDRATION-PLAN-114

Raw SQL hydration no utilizará magic column guessing.

## DB-ORM-HYDRATION-PLAN-115

Plan format podrá versionarse.

## DB-ORM-HYDRATION-PLAN-116

Serialized compiled plans, si existen, serán version-bound.

## DB-ORM-HYDRATION-PLAN-117

Serialized plans no contendrán runtime service instances.

## DB-ORM-HYDRATION-PLAN-118

Compiled plans podrán compartirse entre requests.

## DB-ORM-HYDRATION-PLAN-119

HydrationContext será separado del plan compartido.

## DB-ORM-HYDRATION-PLAN-120

HydrationScope será separado del plan compartido.

## DB-ORM-HYDRATION-PLAN-121

Persistent workers no requerirán reset de immutable plans.

## DB-ORM-HYDRATION-PLAN-122

Mutable plan builders no serán process-global.

## DB-ORM-HYDRATION-PLAN-123

Compiled plans podrán ser concurrent-read-safe.

## DB-ORM-HYDRATION-PLAN-124

Plan compilation deberá ser benchmarkeable.

## DB-ORM-HYDRATION-PLAN-125

Plan execution deberá reducir dynamic resolution en hot path.

## DB-ORM-HYDRATION-PLAN-126

Plan memory no deberá retener entity graphs.

## DB-ORM-HYDRATION-PLAN-127

Plan memory deberá ser compartible cuando sea seguro.

## DB-ORM-HYDRATION-PLAN-128

Compilation cost deberá amortizarse mediante reuse.

## DB-ORM-HYDRATION-PLAN-129

Production mode podrá usar aggressive immutable plan caching.

## DB-ORM-HYDRATION-PLAN-130

Development mode podrá invalidar planes más frecuentemente.

## DB-ORM-HYDRATION-PLAN-131

Telemetry distinguirá plan creation de plan execution.

## DB-ORM-HYDRATION-PLAN-132

Telemetry distinguirá cache hit de miss.

## DB-ORM-HYDRATION-PLAN-133

Telemetry no utilizará plan IDs de alta cardinalidad como labels por defecto.

## DB-ORM-HYDRATION-PLAN-134

Plan complexity será observable.

## DB-ORM-HYDRATION-PLAN-135

Hydration Plan no ejecutará SQL.

## DB-ORM-HYDRATION-PLAN-136

Hydration Plan no administrará transaction.

## DB-ORM-HYDRATION-PLAN-137

Hydration Plan no administrará Connection.

## DB-ORM-HYDRATION-PLAN-138

Hydration Plan no hará flush.

## DB-ORM-HYDRATION-PLAN-139

Hydration Plan no creará entities por sí mismo.

## DB-ORM-HYDRATION-PLAN-140

Hydration Plan no registrará IdentityMap state.

## DB-ORM-HYDRATION-PLAN-141

Hydration Plan no registrará UoW state.

## DB-ORM-HYDRATION-PLAN-142

Hydration Plan podrá referenciar contracts requeridos para esos procesos.

## DB-ORM-HYDRATION-PLAN-143

Result aliases físicos serán tratados como compiler-hydration contract.

## DB-ORM-HYDRATION-PLAN-144

Logical aliases serán tratados como application-facing semantics.

## DB-ORM-HYDRATION-PLAN-145

Logical and physical alias namespaces podrán diferir.

## DB-ORM-HYDRATION-PLAN-146

Field order físico no sustituirá metadata lógica.

## DB-ORM-HYDRATION-PLAN-147

Plan deberá poder describir one-row and multi-row logical results.

## DB-ORM-HYDRATION-PLAN-148

Plan deberá poder describir root grouping.

## DB-ORM-HYDRATION-PLAN-149

Plan deberá poder describir graph finalization.

## DB-ORM-HYDRATION-PLAN-150

Plan deberá poder describir partial collection completeness.

## DB-ORM-HYDRATION-PLAN-151

Plan deberá poder describir projection results sin entity semantics.

## DB-ORM-HYDRATION-PLAN-152

Plan deberá poder describir scalar-only results sin ORM state dependencies.

## DB-ORM-HYDRATION-PLAN-153

Plan deberá poder describir tuple results sin confundir tuple identity con entity identity.

## DB-ORM-HYDRATION-PLAN-154

Plan deberá poder describir DTO construction sin convertir DTO en managed entity.

## DB-ORM-HYDRATION-PLAN-155

Plan compilation failures serán diagnósticos y tipados.

## DB-ORM-HYDRATION-PLAN-156

Plan validation failures no deberán llegar innecesariamente al row hot path.

## DB-ORM-HYDRATION-PLAN-157

Plan generation deberá ser reproducible.

## DB-ORM-HYDRATION-PLAN-158

Plan compatibility deberá ser verificable.

## DB-ORM-HYDRATION-PLAN-159

Plan runtime sharing nunca podrá implicar shared mutable ORM state.

## DB-ORM-HYDRATION-PLAN-160

Toda hidratación estándar de VoltStack deberá estar gobernada por una representación estructurada y validada que describa explícitamente cómo pasar del layout físico del Result a la estructura lógica esperada sin reconstruir esa semántica durante cada row.

---

# 237. Exception Architecture

```text id="qyo8se"
DatabaseHydrationPlanException
├── HydrationPlanDefinitionException
├── HydrationPlanNormalizationException
├── HydrationPlanValidationException
│   ├── HydrationPlanStructuralException
│   ├── HydrationPlanMetadataException
│   ├── HydrationPlanTypeException
│   ├── HydrationPlanIdentityException
│   ├── HydrationPlanResultLayoutException
│   ├── HydrationPlanConstructionException
│   ├── HydrationPlanRelationshipException
│   ├── HydrationPlanGroupingException
│   ├── HydrationPlanDeduplicationException
│   ├── HydrationPlanCardinalityException
│   ├── HydrationPlanStreamingException
│   └── HydrationPlanLifecycleException
├── HydrationPlanCompilationException
├── HydrationPlanFingerprintException
├── HydrationPlanCompatibilityException
├── StaleHydrationPlanException
├── HydrationPlanExtensionConflictException
├── HydrationPlanSecurityException
└── HydrationPlanInvariantException
```

---

# 238. Directory Structure

```text id="lwpdzl"
src/Quantum/Database/ORM/Hydration/Plan/
│
├── Contract/
│   ├── HydrationPlanCompiler.php
│   ├── HydrationPlanNormalizer.php
│   ├── HydrationPlanValidator.php
│   ├── HydrationPlanOptimizer.php
│   ├── HydrationPlanCompatibilityChecker.php
│   └── HydrationPlanContributor.php
│
├── Definition/
│   ├── HydrationPlanDefinition.php
│   ├── HydrationShapeDefinition.php
│   ├── EntityHydrationNodeDefinition.php
│   ├── ScalarHydrationNodeDefinition.php
│   ├── TupleHydrationNodeDefinition.php
│   ├── ProjectionHydrationNodeDefinition.php
│   ├── DtoHydrationNodeDefinition.php
│   ├── RelationshipHydrationDefinition.php
│   ├── HydrationCardinalityDefinition.php
│   └── HydrationStreamingDefinition.php
│
├── Model/
│   ├── HydrationPlanId.php
│   ├── HydrationNodeId.php
│   ├── HydrationPlanFingerprint.php
│   ├── HydrationPlanFormatVersion.php
│   ├── HydrationPlanCacheability.php
│   └── HydrationPlanComplexity.php
│
├── Result/
│   ├── ResultLayoutDefinition.php
│   ├── ResultColumnDefinition.php
│   ├── ResultSlotBinding.php
│   ├── ResultColumnId.php
│   └── ResultSlotReference.php
│
├── Entity/
│   ├── IdentifierHydrationDefinition.php
│   ├── CompiledIdentifierHydrationPlan.php
│   ├── CompiledFieldHydrationPlan.php
│   ├── CompiledEntityConstructionPlan.php
│   └── PartialEntityHydrationPlan.php
│
├── Relationship/
│   ├── RelationshipHydrationPlan.php
│   ├── RelationshipHydrationMode.php
│   └── RelationshipResultCompleteness.php
│
├── Scalar/
│   └── CompiledScalarHydrationPlan.php
│
├── Tuple/
│   └── CompiledTupleHydrationPlan.php
│
├── Projection/
│   └── CompiledProjectionHydrationPlan.php
│
├── Dto/
│   └── CompiledDtoHydrationPlan.php
│
├── Grouping/
│   ├── HydrationGroupingPlan.php
│   └── HydrationGroupingStrategy.php
│
├── Deduplication/
│   ├── HydrationDeduplicationPlan.php
│   └── HydrationDeduplicationStrategy.php
│
├── Cardinality/
│   └── HydrationCardinalityPlan.php
│
├── Streaming/
│   ├── HydrationStreamingPlan.php
│   ├── HydrationStreamingMode.php
│   └── HydrationFinalizationStrategy.php
│
├── Lifecycle/
│   ├── HydrationLifecyclePlan.php
│   └── LifecycleFinalizationPoint.php
│
├── Compilation/
│   ├── DefaultHydrationPlanCompiler.php
│   ├── HydrationCompilationContext.php
│   └── CompiledHydrationPlan.php
│
├── Normalization/
│   └── DefaultHydrationPlanNormalizer.php
│
├── Validation/
│   └── DefaultHydrationPlanValidator.php
│
├── Optimization/
│   └── DefaultHydrationPlanOptimizer.php
│
├── Fingerprint/
│   └── HydrationPlanFingerprinter.php
│
├── Builder/
│   └── DefaultHydrationPlanBuilder.php
│
├── Extension/
│   └── HydrationPlanContributionRegistry.php
│
├── Diagnostics/
│   ├── HydrationPlanExplainer.php
│   └── HydrationPlanDiagnostics.php
│
└── Exception/
    └── ...
```

---

# 239. Fórmulas fundamentales

## 239.1 Plan generation

```text id="z3fn1k"
HydrationPlan
=
Normalize(
    ResultShape
    +
    EntityMetadata
    +
    TypeMetadata
    +
    ResultLayout
    +
    QuerySemantics
)
```

---

# 240. Compilation

```text id="opw20a"
CompiledHydrationPlan
=
Compile(
    Validate(
        Optimize(
            Normalize(
                HydrationPlanDefinition
            )
        )
    )
)
```

El orden real puede validar antes y después de optimización; conceptualmente ambas fases deberán garantizar corrección.

---

# 241. Runtime execution

```text id="axwd7v"
LogicalResult
=
Execute(
    CompiledHydrationPlan,
    Result,
    HydrationContext
)
```

---

# 242. Entity field binding

```text id="81jb3d"
FieldValue
=
FieldWriter(
    TypeConverter(
        Result[PhysicalIndex]
    )
)
```

---

# 243. Entity identity

```text id="ww6tz5"
EntityKey
=
IdentityNamespace
×
IdentityDomain
×
Canonicalize(
    Convert(
        IdentifierResultSlots
    )
)
```

---

# 244. Relationship plan

```text id="m7xpqh"
RelationshipAssembly
=
OwnerNode
+
RelatedNode
+
RelationshipMetadata
+
CompletenessSemantics
```

---

# 245. Grouping

```text id="7ttncs"
LogicalGroup
=
Group(
    PhysicalRows,
    HydrationGroupingPlan
)
```

---

# 246. Deduplication

```text id="g3r7of"
UniqueLogicalResults
=
Deduplicate(
    LogicalResults,
    HydrationDeduplicationPlan
)
```

---

# 247. Streaming safety

```text id="4nfmk9"
Streamable(plan)
=
CompletionBoundaryKnown
∧
OrderingRequirementsSatisfied
∧
BoundedStatePossible
```

---

# 248. Cacheability

```text id="fnvh09"
Cacheable(plan)
=
Immutable(plan)
∧
Deterministic(plan)
∧
NoScopedState(plan)
∧
KnownDependencyGenerations(plan)
```

---

# 249. Fingerprint

```text id="h8wdb1"
Fingerprint
=
Hash(
    NormalizedShape
    +
    ResultLayout
    +
    MetadataGeneration
    +
    TypeGeneration
    +
    StrategyConfiguration
    +
    CompilerVersion
)
```

---

# 250. Runtime safety

```text id="7n2d7d"
SafeHydrationPlanRuntime
=
ImmutableSharedPlan
∧
SeparateScopedHydrationContext
∧
SeparateScopedPersistenceContext
∧
GenerationValidation
∧
DeterministicPlanReuse
```

---

# 251. Master Formula

```text id="h8di9f"
Database Hydration Plan System
=
Hydration Plan Definitions
+
Hydration Shapes
+
Hydration Nodes
+
Result Layout Definitions
+
Logical-to-Physical Bindings
+
Identifier Plans
+
Type Conversion Plans
+
Field Hydration Plans
+
Construction Plans
+
DTO Plans
+
Projection Plans
+
Relationship Plans
+
Loaded Field Masks
+
Tracking Modes
+
Grouping Plans
+
Deduplication Plans
+
Logical Result Keys
+
Cardinality Plans
+
Streaming Plans
+
Completion Boundaries
+
Lifecycle Finalization Plans
+
Plan Normalization
+
Plan Validation
+
Plan Optimization
+
Plan Compilation
+
Plan Fingerprinting
+
Metadata Generations
+
Type Registry Generations
+
Plan Compatibility
+
Cacheability
+
Plan Explainability
+
Extension Contributions
+
Raw Query Mapping
+
Security Validation
+
Persistent Runtime Sharing
+
Telemetry
+
Diagnostics
+
Testing
```

---

# 252. Master Rule

> **En VoltStack, toda hidratación estándar deberá estar gobernada por un `HydrationPlan` compilado que describa explícitamente la relación entre el layout físico del resultado y la estructura lógica esperada. Los hydrators serán ejecutores especializados de ese plan y no deberán redescubrir tipos, campos, identidades, relaciones, aliases, agrupación o cardinalidad durante cada fila.**

---

# 253. Arquitectura resultante

```text id="s1qhvx"
             Entity Query / Projection / DTO Request
                              │
                              ▼
                      Semantic Result Shape
                              │
                              ▼
                         Query Planner
                              │
                              ▼
                         SQL Compiler
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
              Compiled Query       Result Layout
                                         │
                                         ▼
                             HydrationPlanDefinition
                                         │
                                         ▼
                                  Normalization
                                         │
                                         ▼
                                    Validation
                                         │
                                         ▼
                                   Optimization
                                         │
                                         ▼
                                    Compilation
                                         │
                                         ▼
                               Fingerprint / Cache
                                         │
                                         ▼
                             CompiledHydrationPlan
                                         │
               ┌─────────────────────────┼─────────────────────────┐
               ▼                         ▼                         ▼
       Entity Hydration          Scalar Hydration            Tuple/DTO
               │                         │                         │
               └─────────────────────────┼─────────────────────────┘
                                         ▼
                               Result Hydration
                                         │
                                         ▼
                                  Logical Result
```

Con esta separación:

```text id="aa6mez"
Query Planner
decides how to obtain data

SQL Compiler
decides physical SQL representation

Execution Engine
obtains Result

Hydration Plan
describes interpretation

Hydrators
perform interpretation

ORM
receives canonical logical objects/results
```

---

# 254. Siguiente documento

```text id="zb9roo"
141_DATABASE_HYDRATION_CACHE_SYSTEM.md
```

El siguiente documento cerrará el **Bloque 12 — Hydration** y deberá formalizar:

```text id="4r2vbt"
HydrationPlanCache
CompiledHydrationPlan cache
cache namespaces
cache keys
HydrationPlanFingerprint
metadata generations
type registry generations
extension generations
compiler versioning
cache compatibility
cache invalidation
stale plan detection
memory cache
persistent worker cache
optional persistent cache
warmup
precompilation
cache admission policy
cache eviction
LRU/LFU strategies
memory budgets
stampede prevention
concurrent compilation
negative caching
development invalidation
production immutable caching
plan serialization
plan format versioning
checksum validation
security
telemetry
cache hit/miss diagnostics
FrankenPHP reuse
RoadRunner reuse
OpenSwoole isolation
testing
```

Su regla central deberá ser:

> **El Hydration Cache de VoltStack almacenará únicamente artefactos de hidratación compilados e inmutables; nunca entidades, resultados, `EntityManager`, `PersistenceContext`, IdentityMap ni cualquier estado mutable de request.**