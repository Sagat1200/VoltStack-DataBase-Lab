# 245_DATABASE_HYDRATION_PERFORMANCE_SYSTEM.md

# VoltStack Quantum Database
## Database Hydration Performance System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 245 — Database Hydration Performance System  
**Bloque:** 24 — Performance  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `244_DATABASE_ORM_PERFORMANCE_SYSTEM.md`  
**Siguiente documento:** `246_DATABASE_METADATA_COMPILATION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura del **Hydration Performance System** de VoltStack Database.

Su objetivo es controlar y optimizar el costo de transformar resultados físicos provenientes del Execution Engine en estructuras utilizables por la aplicación:

```text
Database
    ↓
Driver
    ↓
Execution Engine
    ↓
Result / ResultCursor
    ↓
Hydration Engine
    ↓
Application Result
```

Los resultados pueden convertirse en:

```text
Scalar
Tuple
Array
Projection
DTO
Entity
Entity Graph
Collection
```

El sistema deberá optimizar:

- lectura de filas;
- decodificación;
- conversión de tipos;
- resolución de metadata;
- resolución de identidad;
- allocation de objetos;
- acceso a propiedades;
- deduplicación;
- snapshots;
- relación con IdentityMap;
- registro en UnitOfWork;
- ensamblaje de relaciones;
- memoria temporal;
- procesamiento incremental;
- throughput.

Sin alterar las garantías semánticas definidas por el sistema de hidratación.

Regla central:

> **La hidratación de VoltStack deberá minimizar el costo de transformar resultados físicos en representaciones de aplicación sin sacrificar tipos, identidad canónica, loaded-field knowledge, snapshots, relaciones, EntityState ni las garantías del UnitOfWork.**

---

# 2. Principio fundamental

Una optimización de hidratación nunca podrá transformar:

```text
fast hydration
```

en:

```text
incorrect application state
```

Por tanto:

```text
Hydration Performance
≠
Skipping Hydration Semantics
```

---

# 3. Recordatorio arquitectónico

La arquitectura definida anteriormente establece:

```text
Query
 ↓
Execution Engine
 ↓
Result
 ↓
Hydration Plan
 ↓
Hydrator
 ↓
Application Result
```

El Hydrator:

```text
NO ejecuta queries
NO genera SQL
NO decide routing
NO administra transacciones
NO reemplaza IdentityMap
NO reemplaza UnitOfWork
```

---

# 4. Hydration ≠ Query Execution

La consulta ya fue ejecutada cuando comienza hidratación.

Por tanto:

```text
Query Execution Time
≠
Hydration Time
```

Esta separación será fundamental para profiling.

---

# 5. Hydration ≠ ORM

La hidratación puede producir:

```text
entities
```

pero también:

```text
scalars
tuples
arrays
DTOs
projections
```

Por tanto:

```text
Hydration Performance
≠
ORM Performance
```

---

# 6. Hydration ≠ Serialization

Hydration:

```text
Database Result
→
Application Representation
```

Serialization:

```text
Application Representation
→
External Representation
```

Son procesos distintos.

---

# 7. Hydration ≠ Type Conversion

Type Conversion participa en hidratación, pero es un subsistema independiente.

```text
Hydration
    ↓
Type Conversion
```

no:

```text
Hydration = Type Conversion
```

---

# 8. Objetivos

El sistema deberá permitir responder:

```text
¿Cuántas filas fueron leídas?

¿Cuántas filas fueron hidratadas?

¿Cuántas entidades fueron creadas?

¿Cuántas entidades fueron reutilizadas desde IdentityMap?

¿Cuántos DTOs fueron creados?

¿Cuántas conversiones de tipos ocurrieron?

¿Cuántos snapshots fueron generados?

¿Cuántas relaciones fueron ensambladas?

¿Cuántas filas duplicadas aparecieron por JOIN?

¿Cuánto tiempo consumió la hidratación?

¿Cuánta memoria consumió?

¿Cuál fue el throughput?

¿Cuánto costó cada entidad hidratada?
```

---

# 9. Modelo de costo

Conceptualmente:

```text
Chydration
=
CrowRead
+
Cdecode
+
Cconversion
+
Cmetadata
+
Cidentity
+
Callocation
+
Cassignment
+
Cdeduplication
+
Crelationship
+
Csnapshot
+
Cuow
+
CtemporaryMemory
```

---

# 10. Tiempo total

```text
Thydration
=
Trow
+
Tconversion
+
Tidentity
+
Tallocation
+
Tassignment
+
Trelations
+
Tsnapshot
+
Tuow
```

---

# 11. Hydration throughput

Una métrica importante será:

```text
HydrationThroughput
=
HydratedRows
/
HydrationDuration
```

expresada como:

```text
rows / second
```

---

# 12. Entity throughput

También:

```text
EntityThroughput
=
HydratedEntities
/
HydrationDuration
```

---

# 13. Rows ≠ entities

Con JOIN:

```text
1 User
20 Orders
```

puede producir:

```text
20 physical rows
```

pero solamente:

```text
1 User
20 Order entities
```

Por tanto:

```text
RowsHydrated
≠
EntitiesCreated
```

---

# 14. Hydration amplification

Definimos:

```text
HydrationAmplification
=
PhysicalRows
/
LogicalRootResults
```

Ejemplo:

```text
Physical rows: 10,000
Root entities: 100
```

Entonces:

```text
Amplification = 100
```

---

# 15. High amplification

Un valor alto puede indicar:

```text
JOIN explosion
```

pero no necesariamente un error.

---

# 16. Performance findings

Podrá detectarse:

```text
HIGH_HYDRATION_AMPLIFICATION
```

---

# 17. Arquitectura general

```text
Result
  │
  ▼
HydrationPerformanceSession
  │
  ▼
HydrationPlan
  │
  ├── Column Mapping
  ├── Type Mapping
  ├── Identity Mapping
  ├── Assignment Mapping
  └── Relationship Mapping
  │
  ▼
Hydrator
  │
  ├── Row Reader
  ├── Type Converter
  ├── Identity Resolver
  ├── Object Allocator
  ├── Property Writer
  ├── Relationship Assembler
  └── Snapshot Builder
  │
  ▼
Application Result
```

---

# 18. HydrationPerformanceContext

```php
final readonly class HydrationPerformanceContext
{
    public function __construct(
        public HydrationOperationId $operationId,
        public HydrationPerformancePolicy $policy,
        public HydrationPerformanceBudget $budget,
        public HydrationPerformanceMeasurementMode $mode,
    ) {}
}
```

---

# 19. Operation identity

```php
final readonly class HydrationOperationId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 20. Correlación

Una operación podrá correlacionarse con:

```text
ORMOperationId
QueryOperationId
ExecutionOperationId
HydrationOperationId
```

---

# 21. HydrationPerformanceBudget

```php
final readonly class HydrationPerformanceBudget
{
    public function __construct(
        public ?Duration $maxDuration = null,
        public ?int $maxRows = null,
        public ?int $maxEntities = null,
        public ?int $maxObjects = null,
        public ?int $maxMemoryBytes = null,
        public ?float $maxAmplification = null,
    ) {}
}
```

---

# 22. Measurement modes

```php
enum HydrationPerformanceMeasurementMode
{
    case OFF;
    case MINIMAL;
    case STANDARD;
    case DETAILED;
    case PROFILE;
}
```

---

# 23. MINIMAL

Podrá medir:

```text
duration
rows
result count
```

---

# 24. STANDARD

Podrá agregar:

```text
entities
objects
IdentityMap hits
type conversions
snapshots
relationships
```

---

# 25. DETAILED

Podrá agregar:

```text
entity types
conversion categories
allocation counts
deduplication
memory
relationship assembly
```

---

# 26. PROFILE

Podrá medir:

```text
per-column cost
per-type cost
per-entity cost
property assignment cost
individual relationship assembly
```

con mayor overhead.

---

# 27. Hydration Plan

El `HydrationPlan` será fundamental para performance.

Debe describir anticipadamente:

```text
result shape
columns
types
entity metadata
identifiers
property accessors
relationships
snapshot strategy
```

---

# 28. Plan before rows

La información estructural deberá resolverse antes del hot loop cuando sea posible.

No:

```text
foreach row
    inspect metadata
    parse attributes
    discover property
    resolve type
```

Preferido:

```text
prepare plan
    ↓
foreach row
    execute prepared instructions
```

---

# 29. Hydration Plan ≠ Query Plan

Continúa vigente:

```text
Hydration Plan
≠
Query Plan
```

---

# 30. Query Plan

Describe:

```text
cómo obtener datos
```

Hydration Plan describe:

```text
cómo convertir los resultados obtenidos
```

---

# 31. Hydration Plan caching

Los planes podrán almacenarse en:

```text
Hydration Plan Cache
```

---

# 32. Cache contents

Puede contener:

```text
column indexes
column names
type handlers
compiled property writers
identifier extractors
relationship assemblers
snapshot instructions
```

---

# 33. Cache no entities

Nunca:

```text
HydrationPlanCache
→
Entity objects
```

---

# 34. Cache key

Deberá incluir información como:

```text
result shape
mapping generation
metadata generation
platform/type generation
selected fields
relationship structure
```

---

# 35. Stale plan prevention

Si cambia metadata:

```text
MetadataGeneration
```

deberá invalidar planes incompatibles.

---

# 36. Reflection

Reflection repetitiva dentro del row loop será evitada.

---

# 37. Anti-pattern

```php
foreach ($rows as $row) {
    $reflection = new ReflectionClass(User::class);
}
```

estará prohibido en implementación normal.

---

# 38. Attribute parsing

Tampoco:

```text
parse PHP attributes
for every row
```

---

# 39. Metadata compilation

La preparación profunda de metadata será responsabilidad principal de:

```text
246_DATABASE_METADATA_COMPILATION_SYSTEM.md
```

---

# 40. Property assignment

Uno de los hot paths será:

```text
database value
    ↓
converted value
    ↓
entity property
```

---

# 41. Assignment strategies

VoltStack podrá evaluar:

```text
ReflectionProperty
Closure accessor
Generated accessor
Direct constructor
Hydration constructor
Custom hydrator
```

---

# 42. Strategy abstraction

```php
interface PropertyWriter
{
    public function write(
        object $target,
        mixed $value,
    ): void;
}
```

---

# 43. Compiled writer

Ejemplo conceptual:

```php
$writer = static function (User $user, string $name): void {
    $user->name = $name;
};
```

según visibility y mapping strategy.

---

# 44. Generated accessors

Podrán evaluarse para hot paths.

Pero:

> **Generated code no será adoptado únicamente porque intuitivamente parezca más rápido.**

Debe existir benchmark.

---

# 45. Constructor hydration

Para DTOs:

```php
new UserView(
    id: $id,
    name: $name,
);
```

puede ser más apropiado.

---

# 46. Entity constructor semantics

Las entidades pueden requerir constructores de dominio.

Hydration no deberá invocarlos incorrectamente solo por performance.

---

# 47. Constructor strategy

Metadata deberá definir:

```text
NORMAL_CONSTRUCTOR
HYDRATION_CONSTRUCTOR
WITHOUT_CONSTRUCTOR
FACTORY
CUSTOM
```

según modelo soportado.

---

# 48. Allocation cost

Object allocation será medible.

```text
ObjectsAllocated
```

podrá incluir:

```text
entities
DTOs
value objects
collections
snapshots
temporary structures
```

---

# 49. Allocation amplification

```text
AllocationAmplification
=
ObjectsAllocated
/
LogicalResults
```

---

# 50. Excessive allocation

Finding:

```text
HIGH_OBJECT_ALLOCATION
```

---

# 51. Temporary allocations

El hot loop deberá minimizar:

```text
temporary arrays
temporary strings
temporary metadata wrappers
temporary identifier objects
```

cuando sea seguro.

---

# 52. Optimization ≠ mutable reuse bugs

No deberán reutilizarse estructuras mutables entre entidades de forma que compartan accidentalmente estado.

---

# 53. Row representation

El Driver/Result System puede entregar:

```text
associative array
numeric array
driver-native row
```

según implementación.

---

# 54. Numeric indexes

Una representación por índices numéricos puede ser más eficiente en ciertos casos.

Ejemplo:

```php
$row[0]
$row[1]
$row[2]
```

frente a:

```php
$row['id']
$row['name']
$row['email']
```

---

# 55. But implementation-specific

VoltStack no asumirá universalmente que numeric arrays son superiores.

Se benchmarkeará por driver/runtime.

---

# 56. Column map

El plan podrá resolver:

```text
column position
→
mapped field
```

una sola vez.

---

# 57. Column lookup

Evitar:

```text
string column resolution
```

repetitiva cuando ya existe mapping estable.

---

# 58. Type conversion

Cada valor puede requerir:

```text
database representation
→
canonical ORM value
```

---

# 59. Conversion examples

```text
VARCHAR → string
BIGINT → int/string
DATETIME → temporal value
JSON → structured value
UUID → UUID value object
ENUM → enum
```

---

# 60. Conversion performance

Se medirán:

```text
conversion count
conversion duration
conversion by logical type
```

en modos apropiados.

---

# 61. Fast path types

Tipos simples pueden tener fast paths.

Ejemplo conceptual:

```text
string → string
int → int
```

si el driver ya entrega representación correcta.

---

# 62. Fast path correctness

Antes de omitir conversión debe probarse:

```text
driver value
=
canonical ORM representation
```

---

# 63. Platform differences

MySQL, MariaDB, PostgreSQL y SQLite pueden entregar representaciones distintas.

Por tanto:

```text
same logical type
≠
same driver raw representation
```

---

# 64. Driver-aware conversion

La estrategia podrá depender de:

```text
Logical Type
+
Platform
+
Driver Capability
```

---

# 65. Type handler lookup

No deberá resolverse TypeRegistry desde cero por cada celda.

---

# 66. Pre-bound converters

HydrationPlan podrá contener:

```php
interface ValueConverter
{
    public function convert(mixed $raw): mixed;
}
```

ya resuelto.

---

# 67. Custom types

Custom Type podrá ser más costoso.

Performance System podrá identificar:

```text
EXPENSIVE_CUSTOM_TYPE_CONVERSION
```

---

# 68. Conversion telemetry

No registrará valores convertidos.

Solo:

```text
type
count
duration
failures
```

con cardinalidad bounded.

---

# 69. NULL handling

`NULL` tendrá fast path cuando mapping lo permita.

---

# 70. NULL ≠ missing

Continúa vigente:

```text
Database NULL
≠
Field Not Selected
```

---

# 71. LoadedFieldMask

Partial projection/entity hydration utilizará:

```text
LoadedFieldMask
```

para distinguir campos cargados.

---

# 72. Loaded mask performance

Podrá implementarse mediante:

```text
bitset
compact integer mask
immutable field set
```

según número de campos.

---

# 73. Bitset optimization

Para metadata estable, un bitset puede hacer:

```text
loaded(field)
```

muy eficiente.

---

# 74. LoadedFieldMask ≠ correctness shortcut

Nunca deberá marcarse un campo como loaded si no estaba presente.

---

# 75. Entity identity extraction

Antes de crear una entidad managed deberá extraerse su identidad.

---

# 76. Identity extraction

HydrationPlan podrá precompilar:

```text
row columns
→
IdentifierTuple
```

---

# 77. Identifier types

Debe soportar:

```text
single ID
composite ID
UUID
string ID
tenant-qualified ID
shard-qualified identity context
```

---

# 78. IdentityMap lookup

Pipeline:

```text
row
 ↓
extract identifier
 ↓
canonicalize identifier
 ↓
IdentityMap lookup
 ├── HIT
 └── MISS
```

---

# 79. HIT

Si existe managed entity:

```text
reuse canonical instance
```

---

# 80. MISS

Entonces:

```text
allocate entity
reserve identity
hydrate
activate identity
```

según arquitectura definida.

---

# 81. Identity reservation

Para ciclos:

```text
RESERVED
→
ACTIVE
```

evitará crear identidades duplicadas.

---

# 82. Performance importance

La reserva temprana también reduce duplicación durante JOIN graph hydration.

---

# 83. Existing dirty entity

Una entidad ya managed y dirty no deberá ser sobrescrita ciegamente por una nueva fila.

---

# 84. Performance does not override state

Nunca:

```text
overwrite dirty entity
```

para evitar una rama condicional.

---

# 85. IdentityMap hit metrics

```text
identity lookups
identity hits
identity misses
identity reservations
```

---

# 86. Entity allocation savings

```text
AllocationsAvoided
=
IdentityMapHitsThatAvoidedAllocation
```

---

# 87. Deduplication

JOINs pueden repetir la misma identidad.

Ejemplo:

```text
User#1 Order#1
User#1 Order#2
User#1 Order#3
```

`User#1` se crea una sola vez.

---

# 88. Root deduplication

Hydration session podrá mantener:

```text
RootResultRegistry
```

para evitar agregar varias veces el mismo root.

---

# 89. Root registry ≠ IdentityMap

```text
RootResultRegistry
≠
IdentityMap
```

El primero organiza el resultado actual.

El segundo garantiza identidad managed dentro del scope ORM.

---

# 90. Relationship assembly registry

También podrá existir:

```text
RelationshipAssemblyRegistry
```

para evitar agregar repetidamente la misma relación durante un JOIN.

---

# 91. Session-local structures

Estas estructuras serán:

```text
HydrationSession scoped
```

---

# 92. No static deduplication

Nunca:

```php
Hydrator::$seen;
```

---

# 93. Relationship assembly

JOIN hydration puede requerir:

```text
resolve owner
resolve related entity
resolve collection
check membership
attach relation
update coverage
```

---

# 94. Collection membership check

Debe evitar:

```text
linear scan
```

por cada fila cuando pueda producir O(N²).

---

# 95. Assembly index

Podrá existir:

```text
owner identity
+
relationship ID
+
related identity
```

como índice temporal.

---

# 96. Expected complexity

Con hashing adecuado:

```text
O(rows)
```

promedio para ensamblaje típico.

---

# 97. Assembly memory

Pero estos índices consumen memoria.

Deberán liberarse al finalizar la sesión.

---

# 98. Join amplification example

```text
User
 ├── 100 Orders
 │     └── 20 Items
```

un JOIN completo puede producir:

```text
2,000 rows
```

para:

```text
1 User
100 Orders
2,000 Items
```

---

# 99. Multiple collections

Peor:

```text
User
 ├── 100 Orders
 └── 50 Roles
```

puede producir:

```text
100 × 50 = 5,000 rows
```

para un solo root.

---

# 100. Cartesian amplification

Finding:

```text
CARTESIAN_HYDRATION_AMPLIFICATION
```

---

# 101. Hydration system cannot fix query shape alone

Podrá reportarlo.

Query/Relationship Planner decidirá alternativas.

---

# 102. Suggested alternative

```text
Evaluate SELECT_IN or batch relationship loading.
```

---

# 103. Snapshot creation

Managed entities pueden requerir snapshot inicial.

---

# 104. Hydration snapshot

Debe representar:

```text
canonical persistent state
```

después de hidratación inicial.

---

# 105. Snapshot timing

Pipeline:

```text
assign hydrated state
 ↓
establish baseline snapshot
 ↓
postLoad
```

según contrato definido.

---

# 106. postLoad mutations

Si `postLoad` modifica la entidad:

```text
mutation
```

deberá poder aparecer como dirty.

---

# 107. Snapshot before postLoad

Esto preserva dicha semántica.

---

# 108. Snapshot cost

Hydration Performance medirá:

```text
snapshots created
snapshot values
snapshot bytes estimate
snapshot duration
```

---

# 109. Read-only hydration

Si query es:

```text
READ_ONLY
```

puede evitar snapshot/UoW tracking según contrato.

---

# 110. Performance benefit

Pipeline:

```text
normal entity hydration
=
allocation
+
assignment
+
identity
+
snapshot
+
UoW registration
```

frente a:

```text
read-only
=
allocation
+
assignment
+
reduced tracking
```

según diseño.

---

# 111. Read-only ≠ scalar

Todavía puede producir entidades.

---

# 112. DTO hydration

DTOs no necesitan:

```text
IdentityMap
UnitOfWork
Snapshots
EntityState
```

salvo comportamiento especial explícito.

---

# 113. DTO advantage

Para read workloads pueden reducir overhead significativamente.

---

# 114. DTO semantics

Pero:

```text
DTO
≠
Entity
```

---

# 115. Projection hydration

Projection podrá contener:

```text
selected scalar fields
nested DTOs
computed values
```

sin convertirse en managed graph.

---

# 116. Scalar hydration

Es el camino más simple.

Ejemplo:

```php
$count = $query->scalar();
```

---

# 117. Scalar hot path

No deberá construir:

```text
EntityMetadata
IdentityMap entries
Snapshots
```

innecesariamente.

---

# 118. Scalar list

Ejemplo:

```text
[10, 20, 30]
```

deberá utilizar un hydrator especializado.

---

# 119. Tuple hydration

Ejemplo:

```text
[
    id,
    name,
    total
]
```

---

# 120. Tuple ≠ Entity

No deberá pasar por EntityManager.

---

# 121. Array hydration

Podrá devolver:

```text
associative arrays
```

con type conversion.

---

# 122. Specialized hydrators

VoltStack evitará un único hydrator gigante lleno de ramas.

Arquitectura:

```text
ScalarHydrator
TupleHydrator
ArrayHydrator
DTOHydrator
EntityHydrator
GraphHydrator
```

---

# 123. Shared infrastructure

Todos podrán reutilizar:

```text
HydrationPlan
Type Conversion
Result Cursor
Performance Session
```

---

# 124. Specialized hot loops

Cada hydrator podrá optimizar su loop según shape.

---

# 125. Generic flexibility vs specialization

La arquitectura deberá permitir:

```text
generic fallback
+
optimized specialized paths
```

---

# 126. Entity hydration loop

Conceptualmente:

```php
while ($row = $result->next()) {
    $id = $idExtractor->extract($row);

    $entity = $identityMap->get($type, $id);

    if ($entity === null) {
        $entity = $allocator->allocate();
        $identityMap->reserve($type, $id, $entity);
        $fieldWriter->hydrate($entity, $row);
        $snapshotter->snapshot($entity);
        $unitOfWork->registerManaged($entity);
    }

    $assembler->consume($entity, $row);
}
```

La implementación concreta podrá estar más especializada.

---

# 127. Branch minimization

El hot loop podrá reducir ramas redundantes.

Pero:

```text
branch reduction
```

no justificará omitir validaciones necesarias.

---

# 128. Hydration instructions

Un plan podría compilarse conceptualmente a:

```text
READ column 0
CONVERT int
ASSIGN User.id

READ column 1
CONVERT string
ASSIGN User.name

READ column 2
CONVERT datetime
ASSIGN User.createdAt
```

---

# 129. Instruction interpreter

Una implementación inicial podría interpretar instrucciones.

---

# 130. Compiled hydrator

Posteriormente podría generar un callable optimizado.

```text
HydrationPlan
    ↓
HydratorCompiler
    ↓
CompiledHydrator
```

---

# 131. Generated PHP

Podría evaluarse generar PHP optimizado.

No será requisito de V1.

---

# 132. Cacheability

Compiled hydrators podrán ser reutilizados cuando:

```text
metadata generation
result shape
type mapping
```

sean compatibles.

---

# 133. Persistent runtime advantage

FrankenPHP permitirá mantener:

```text
compiled hydration plans
compiled accessors
immutable converters
```

entre requests.

---

# 134. Persistent runtime restriction

Nunca podrán compartirse:

```text
current entity
IdentityMap
HydrationSession
result cursor
relationship assembly state
tenant state
```

---

# 135. Streaming hydration

Con:

```text
StreamingResult
```

el Hydrator podrá procesar:

```text
row
→
hydrate
→
yield
```

sin materializar todo.

---

# 136. Streaming ≠ constant memory

Si se producen managed entities:

```text
IdentityMap
```

puede seguir creciendo.

---

# 137. Streaming entity hydration

Por tanto:

```text
Streaming Rows
≠
Bounded ORM State
```

---

# 138. Streaming DTO hydration

Puede aproximarse mejor a memoria acotada si el consumidor no retiene resultados.

---

# 139. Consumer retention

Incluso:

```text
yield DTO
```

no garantiza memoria constante si el consumidor guarda todos los objetos.

---

# 140. Lazy Collection integration

Lazy Collection podrá utilizar:

```text
Hydrator
```

incrementalmente.

---

# 141. Chunk integration

Chunk Processing podrá hidratar:

```text
N rows
→
N results
→
process
→
release chunk
```

---

# 142. ORM memory caveat

Nuevamente:

```text
release chunk
≠
release managed entities
```

---

# 143. Batch hydration

Batch hydration podrá procesar bloques de filas.

---

# 144. Batch size

Debe balancear:

```text
function-call overhead
memory
cache locality
latency
result buffering
```

---

# 145. Universal batch size

No existirá una constante universal.

---

# 146. Driver buffering

El driver puede ya estar buffering resultados.

Por tanto:

```text
Hydration Batch
≠
Driver Fetch Buffer
```

---

# 147. Result buffering

Debe medirse por separado cuando sea observable.

---

# 148. Database Cursor

Un cursor DB puede mantener recursos abiertos.

Hydration Performance no deberá ocultar este costo.

---

# 149. Early termination

Si consumidor deja de iterar:

```text
break
```

deberá cerrarse correctamente:

```text
Hydration Session
Result Cursor
Streaming Result
```

según ownership.

---

# 150. Resource ownership

Debe estar definido explícitamente.

---

# 151. Destructor

No será mecanismo primario de correctness para cerrar recursos.

---

# 152. Cancellation

Hydration loop deberá respetar:

```text
CancellationToken
Deadline
ResourceBudget
```

en puntos razonables.

---

# 153. Cancellation frequency

Comprobar cancelación en cada celda podría ser excesivo.

Puede hacerse:

```text
every N rows
```

o en boundaries adecuados.

---

# 154. Bounded responsiveness

La policy definirá equilibrio entre:

```text
overhead
vs
cancellation latency
```

---

# 155. Memory accounting

Hydration podrá observar:

```text
start memory
peak memory
end memory
rows
objects
```

---

# 156. Per-row memory estimate

```text
ApproxMemoryPerRow
=
MemoryDelta
/
Rows
```

será solo una estimación.

---

# 157. GC interference

PHP GC puede distorsionar esta métrica.

Debe marcarse como estimada.

---

# 158. Peak memory

Será especialmente importante para:

```text
large result sets
JOIN amplification
large graphs
DTO collections
array hydration
```

---

# 159. Array hydration memory

Arrays PHP pueden tener overhead significativo.

No deberá asumirse que:

```text
array hydration
```

es siempre más ligera que DTO hydration.

---

# 160. Benchmark required

Debe medirse.

---

# 161. Entity memory

Entity hydration agrega además:

```text
IdentityMap
Snapshots
UnitOfWork
Relationships
```

---

# 162. Hydration memory model

Conceptualmente:

```text
Mhydration
=
MresultBuffer
+
Mobjects
+
Mtemporary
+
Mdedup
+
MrelationshipAssembly
+
Msnapshots
+
Midentity
```

---

# 163. Memory pressure

Finding:

```text
HYDRATION_MEMORY_PRESSURE
```

---

# 164. Large result set

Finding:

```text
LARGE_MATERIALIZED_RESULT
```

---

# 165. Recommendation

Puede sugerir:

```text
Lazy Collection
Chunk Processing
Streaming
Projection
DTO
```

según caso.

---

# 166. Recommendation ≠ automatic rewrite

Nunca cambiará:

```text
get()
```

a:

```text
lazy()
```

silenciosamente.

---

# 167. Result cardinality

El Hydrator puede conocer cardinalidad progresivamente.

No deberá asumir:

```text
rowCount()
```

fiable para SELECT en todos los drivers.

---

# 168. Preallocation

Preasignar arrays según cardinalidad podría ser útil si existe evidencia fiable.

No será universal.

---

# 169. `count()` on result

No deberá utilizarse como supuesto universal para obtener filas totales.

---

# 170. Hydration failures

Podrán ocurrir por:

```text
type conversion
missing identifier
invalid NULL
constructor failure
property assignment
unknown discriminator
relationship assembly
snapshot failure
lifecycle callback
```

---

# 171. Failure after partial hydration

Puede existir:

```text
100 entities hydrated
101st fails
```

---

# 172. Partial hydration state

El sistema deberá manejar correctamente:

```text
IdentityMap reservations
partially assembled relationships
UnitOfWork registrations
```

---

# 173. No half-active identity

Una entidad que falla durante hidratación no deberá quedar silenciosamente:

```text
ACTIVE
```

como si estuviera completa.

---

# 174. Reservation rollback

La sesión deberá poder retirar/cancelar:

```text
RESERVED identity
```

cuando falla la creación.

---

# 175. Existing entities

Si la sesión modificó estructuras de ensamblaje antes del fallo, deberá mantener invariantes.

---

# 176. Hydration exception

```php
class DatabaseHydrationPerformanceException
    extends DatabaseException
{
}
```

Performance failures normalmente no deberán sustituir errores reales de hidratación.

---

# 177. Instrumentation failure

Si falla:

```text
performance recorder
```

la hidratación normal deberá continuar cuando sea seguro.

---

# 178. Performance instrumentation ≠ correctness dependency

Regla:

```text
Telemetry Failure
≠
Hydration Failure
```

---

# 179. HydrationPerformanceMeasurement

```php
final readonly class HydrationPerformanceMeasurement
{
    public function __construct(
        public HydrationOperationId $operationId,
        public HydrationShape $shape,
        public int $rowsRead,
        public int $logicalResults,
        public int $objectsAllocated,
        public int $entitiesCreated,
        public int $entitiesReused,
        public int $typeConversions,
        public int $snapshotsCreated,
        public int $relationshipsAssembled,
        public Duration $duration,
        public HydrationMemoryStatistics $memory,
        public HydrationOutcome $outcome,
    ) {}
}
```

---

# 180. HydrationShape

```php
enum HydrationShape
{
    case SCALAR;
    case SCALAR_LIST;
    case TUPLE;
    case ARRAY;
    case DTO;
    case PROJECTION;
    case ENTITY;
    case ENTITY_COLLECTION;
    case ENTITY_GRAPH;
}
```

---

# 181. HydrationOutcome

```php
enum HydrationOutcome
{
    case COMPLETED;
    case COMPLETED_EARLY;
    case CANCELLED;
    case FAILED;
}
```

---

# 182. Identity statistics

```php
final readonly class HydrationIdentityStatistics
{
    public function __construct(
        public int $lookups,
        public int $hits,
        public int $misses,
        public int $reservations,
        public int $activations,
    ) {}
}
```

---

# 183. Conversion statistics

```php
final readonly class HydrationConversionStatistics
{
    public function __construct(
        public int $values,
        public int $nullValues,
        public int $fastPathValues,
        public int $customTypeValues,
        public Duration $duration,
    ) {}
}
```

---

# 184. Assignment statistics

```php
final readonly class HydrationAssignmentStatistics
{
    public function __construct(
        public int $fieldAssignments,
        public int $constructorArguments,
        public int $valueObjectAssignments,
        public Duration $duration,
    ) {}
}
```

---

# 185. Relationship statistics

```php
final readonly class HydrationRelationshipStatistics
{
    public function __construct(
        public int $owners,
        public int $relatedEntities,
        public int $collectionMembershipChecks,
        public int $deduplicatedMemberships,
        public Duration $duration,
    ) {}
}
```

---

# 186. HydrationMemoryStatistics

```php
final readonly class HydrationMemoryStatistics
{
    public function __construct(
        public ?int $startBytes,
        public ?int $peakBytes,
        public ?int $endBytes,
        public ?int $estimatedTemporaryBytes,
    ) {}
}
```

---

# 187. HydrationPerformanceReport

```php
final readonly class HydrationPerformanceReport
{
    public function __construct(
        public HydrationPerformanceMeasurement $measurement,
        public array $findings,
    ) {}
}
```

---

# 188. Finding codes

```php
enum HydrationPerformanceFindingCode
{
    case HIGH_HYDRATION_AMPLIFICATION;
    case CARTESIAN_HYDRATION_AMPLIFICATION;
    case HIGH_OBJECT_ALLOCATION;
    case HIGH_IDENTITY_LOOKUP_COST;
    case HIGH_TYPE_CONVERSION_COST;
    case EXPENSIVE_CUSTOM_TYPE_CONVERSION;
    case HIGH_PROPERTY_ASSIGNMENT_COST;
    case HIGH_SNAPSHOT_COST;
    case HIGH_RELATIONSHIP_ASSEMBLY_COST;
    case LARGE_MATERIALIZED_RESULT;
    case HYDRATION_MEMORY_PRESSURE;
    case LOW_HYDRATION_THROUGHPUT;
    case EXCESSIVE_TEMPORARY_ALLOCATION;
    case LARGE_MANAGED_HYDRATION;
}
```

---

# 189. Finding evidence

Ejemplo:

```text
Finding:
    HIGH_HYDRATION_AMPLIFICATION

Physical Rows:
    125,000

Root Entities:
    500

Amplification:
    250x

Relationships:
    Order.items
    Order.tags

Confidence:
    HIGH
```

---

# 190. Suggestion

```text
Evaluate splitting multiple collection JOINs into
SELECT_IN or batch relationship loads.
```

---

# 191. Low throughput

Finding:

```text
LOW_HYDRATION_THROUGHPUT
```

deberá considerar shape.

No es comparable directamente:

```text
scalar hydration
```

contra:

```text
large entity graph hydration
```

---

# 192. Benchmark class

Comparaciones deberán agruparse por:

```text
shape
field count
relationship count
type complexity
runtime
driver
platform
```

---

# 193. Telemetry

Métricas posibles:

```text
database.hydration.duration
database.hydration.rows
database.hydration.results
database.hydration.entities.created
database.hydration.entities.reused
database.hydration.objects.allocated
database.hydration.identity.hits
database.hydration.identity.misses
database.hydration.type_conversions
database.hydration.snapshots
database.hydration.relationships
database.hydration.amplification
database.hydration.memory.peak
```

---

# 194. Histograms

Duraciones y tamaños podrán utilizar histogramas.

---

# 195. Labels

Permitidos de forma bounded:

```text
hydration_shape
platform
driver
entity_type
```

cuando la cardinalidad esté controlada.

---

# 196. Forbidden labels

No:

```text
entity_id
tenant_id arbitrary
email
query parameters
field values
```

---

# 197. No per-row events

Production no emitirá:

```text
RowHydratedEvent
```

hacia Event System por cada fila.

---

# 198. Why

Para 1 millón de filas:

```text
1,000,000 events
```

sería contraproducente.

---

# 199. Local counters

Preferido:

```text
hydration session
    ↓
local counters
    ↓
aggregate report
```

---

# 200. Debug information

Debug System podrá mostrar:

```text
Hydration

Shape               ENTITY_GRAPH
Rows                 8,420
Root Entities          200
Entities Created     2,114
Entities Reused      6,306
Duration             34.8 ms
Throughput          241k rows/s
Amplification        42.1x
Peak Memory          31 MB
```

---

# 201. Debug Toolbar

Podrá existir panel:

```text
Hydration
```

o subsección dentro de:

```text
Database / ORM
```

---

# 202. Security

Nunca deberán mostrarse:

```text
hydrated passwords
tokens
API keys
PII
raw JSON secrets
```

en profiling.

---

# 203. Value sampling

Por defecto:

```text
DISABLED
```

---

# 204. Profile values

Incluso PROFILE no debería capturar valores salvo herramienta explícita y política de seguridad independiente.

---

# 205. Query Audit separation

Hydration Performance no será mecanismo de auditoría.

---

# 206. Tenant context

El contexto de tenant deberá permanecer estable durante la hidratación.

---

# 207. Tenant drift

Nunca:

```text
row 1 → tenant A
row 2 → tenant B
```

dentro de una operación tenant-scoped por cambio accidental del contexto runtime.

---

# 208. Cross-tenant result

Solo será posible mediante una operación explícitamente diseñada y autorizada.

---

# 209. Shard context

La identidad deberá conservar:

```text
database domain
shard
tenant
```

cuando formen parte del contexto efectivo.

---

# 210. Same local ID across shards

```text
Shard A User#10
Shard B User#10
```

no deberán colisionar.

---

# 211. Distributed hydration

Resultados distribuidos pueden requerir:

```text
merge
deduplication
global ordering
hydration
```

---

# 212. Merge ≠ hydration

La fusión distribuida pertenece al execution/distribution layer.

Hydration consume el resultado lógico correspondiente.

---

# 213. Cross-shard entity graph

No se ensamblará como si fuera una relación local normal sin soporte explícito.

---

# 214. Relationship coverage

Hydration deberá conservar:

```text
UNINITIALIZED
PARTIAL
COMPLETE
```

correctamente.

---

# 215. Performance cannot fake completeness

Nunca:

```text
PARTIAL → COMPLETE
```

para evitar futuras cargas.

---

# 216. Optional joined relation

Si todos los identificadores relacionados son NULL:

```text
related entity absent
```

sin allocation.

---

# 217. Null-ID fast path

Esto puede evitar:

```text
IdentityMap lookup
entity allocation
property assignments
```

para relaciones ausentes.

---

# 218. Composite identifier NULL semantics

La regla deberá seguir metadata.

No bastará:

```text
any identifier component NULL
→ absent
```

universalmente.

---

# 219. Discriminator resolution

Herencia/polimorfismo puede requerir:

```text
discriminator
→
EntityType
```

---

# 220. Discriminator registry

Debe estar pre-resuelto cuando sea posible.

---

# 221. No FQCN injection

Valores de DB no podrán instanciar clases arbitrarias.

---

# 222. Unknown discriminator

Debe producir error definido.

Performance no podrá usar:

```text
fallback class
```

silencioso.

---

# 223. Polymorphic relationships

Pueden aumentar costo porque:

```text
type resolution
+
identity resolution
+
metadata dispatch
```

ocurren por resultado.

---

# 224. Polymorphic fast path

Registry podrá mapear:

```text
stable discriminator
→
compiled hydration plan
```

---

# 225. Value Objects

Value Object hydration puede implicar allocations adicionales.

---

# 226. Inline value objects

Ejemplo:

```text
Money
Address
Coordinates
```

---

# 227. Value Object allocation

Será contabilizado separadamente cuando profiling lo permita.

---

# 228. Immutable value objects

Podrían permitir optimizaciones internas.

Pero no se hará interning global arbitrario.

---

# 229. JSON hydration

JSON puede ser costoso por:

```text
decode
validation
nested allocations
```

---

# 230. JSON projection

Si la aplicación no necesita decodificar JSON completo, futuras optimizaciones podrían usar:

```text
database-side JSON projection
```

a través de Query Engine.

Hydrator no reescribirá la query.

---

# 231. Temporal hydration

Date/time conversion también puede ser costosa.

---

# 232. Temporal objects

Ejemplo:

```text
100,000 rows
×
3 temporal fields
=
300,000 temporal object conversions
```

---

# 233. Temporal optimization

Converters podrán reutilizar:

```text
timezone metadata
format parsers
```

inmutables.

No objetos temporales mutables compartidos.

---

# 234. Enum hydration

Registry deberá resolver enum mapping eficientemente.

---

# 235. Enum cases

PHP enum cases son singletons del lenguaje cuando corresponde, lo cual puede reducir allocations.

---

# 236. Custom hydrators

VoltStack permitirá:

```php
interface CustomHydrator
{
    public function hydrate(
        ResultCursor $result,
        HydrationContext $context,
    ): mixed;
}
```

---

# 237. Custom hydrator contract

Deberá respetar:

```text
resource ownership
cancellation
type semantics
security
telemetry
```

---

# 238. Custom hydrator performance

Podrá integrarse al mismo Performance Session.

---

# 239. Hydration extension

Extensiones no podrán acceder directamente al Driver para ejecutar queries durante hidratación.

---

# 240. No hidden query

Regla:

> **Hydration nunca realizará una consulta oculta para completar datos faltantes.**

Eso correspondería a relationship/lazy loading posterior.

---

# 241. Hydration purity boundary

Idealmente:

```text
Result + Plan + Context
→
Hydrated Representation
```

sin nuevo I/O de DB.

---

# 242. Lifecycle events

`postLoad` puede ejecutarse después de hidratación.

Su costo deberá distinguirse.

---

# 243. Hydration time vs postLoad time

Podrán reportarse:

```text
core hydration
lifecycle callbacks
total ORM materialization
```

por separado.

---

# 244. Callback query

Si `postLoad` ejecuta una query:

```text
Query Telemetry
```

la correlacionará.

No deberá contarse como core hydration SQL.

---

# 245. Performance policy

```php
interface HydrationPerformancePolicy
{
    public function budgetFor(
        HydrationShape $shape,
        HydrationContext $context,
    ): HydrationPerformanceBudget;
}
```

---

# 246. Different budgets

Ejemplo:

```text
HTTP entity query
    max 10,000 rows

CLI export
    max 10,000,000 rows
    but streaming required

Admin report
    projection preferred
```

---

# 247. Materialization budget

Podrá existir:

```text
maxMaterializedResults
```

---

# 248. Exceeding budget

Policy:

```text
ALLOW
WARN
REJECT
```

---

# 249. No silent strategy switch

Nunca:

```text
materialized get()
→
streaming()
```

automáticamente por exceder threshold.

---

# 250. Explain integration

Una query podrá mostrar:

```text
HYDRATION PLAN

Shape
    ENTITY_GRAPH

Root
    Order

Fields
    14

Relationships
    customer
    items.product

Identity Resolution
    ENABLED

Snapshots
    ENABLED

Expected Root Cardinality
    UNKNOWN

Warnings
    Multiple collection JOIN may amplify rows
```

---

# 251. Runtime diagnostics

Durante ejecución:

```text
HYDRATION ACTUAL

Rows
    42,511

Roots
    500

Amplification
    85x

Peak Memory
    94 MB
```

---

# 252. Estimated vs actual

Debe diferenciarse:

```text
ESTIMATED
ACTUAL
```

---

# 253. Query planner estimates

Si Query Planner proporciona cardinalidad estimada, Hydration Planner podrá utilizarla.

---

# 254. Unknown estimate

```text
UNKNOWN
```

no será convertido a cero.

---

# 255. Adaptive buffer sizing

Podría explorarse utilizando estimaciones.

No será requisito de V1.

---

# 256. HydrationPerformanceAnalyzer

```php
interface HydrationPerformanceAnalyzer
{
    public function analyze(
        HydrationPerformanceMeasurement $measurement,
        HydrationPerformanceContext $context,
    ): HydrationPerformanceReport;
}
```

---

# 257. Analyzer read-only

No podrá:

```text
rerun query
change HydrationPlan
clear EntityManager
detach entities
change relationship strategy
```

---

# 258. Analyzer after operation

Normalmente analizará:

```text
measurement
```

una vez terminada la operación.

---

# 259. Online guardrails

Resource budgets podrán actuar durante hidratación.

Eso será responsabilidad de:

```text
Resource Governance
```

no del analyzer post-operation.

---

# 260. Benchmark dimensions

Hydration benchmarks deberán variar:

```text
rows
columns
types
shape
relationships
IdentityMap hit ratio
snapshot mode
result buffering
runtime
driver
platform
```

---

# 261. Scalar benchmark

Ejemplo:

```text
1M integers
```

---

# 262. Tuple benchmark

```text
1M rows
×
5 scalar fields
```

---

# 263. DTO benchmark

```text
100k rows
×
10 fields
```

---

# 264. Entity benchmark

```text
100k entities
×
10 fields
+
IdentityMap
+
snapshot
+
UoW
```

---

# 265. IdentityMap-hit benchmark

```text
100k rows
90% existing identities
```

---

# 266. JOIN graph benchmark

```text
1,000 roots
10 children/root
10 grandchildren/child
```

---

# 267. Multiple collection benchmark

Debe probar escenarios de cartesian amplification.

---

# 268. Custom type benchmark

Especialmente:

```text
JSON
UUID
DateTime
ValueObject
Enum
```

---

# 269. Warm vs cold

Deberán distinguirse:

```text
cold metadata/plan
warm metadata/plan
```

---

# 270. Persistent runtime benchmark

FrankenPHP deberá probar:

```text
first request
warm requests
10k sequential requests
```

---

# 271. Memory reset benchmark

Después de cada request deberá comprobarse:

```text
HydrationSession released
Result released
temporary assembly registries released
```

---

# 272. Compiled plan reuse

Podrá sobrevivir si es:

```text
immutable
context-independent
generation-compatible
```

---

# 273. Tenant-specific plan state

No deberá incluir:

```text
current tenant entity instances
tenant credentials
request state
```

---

# 274. Plan key tenant awareness

Si el mapping físico varía por tenant:

```text
tenant schema generation
```

puede formar parte de la clave.

---

# 275. Shared-schema multitenancy

Si mapping es idéntico:

```text
plan
```

podrá compartirse estructuralmente.

---

# 276. Hydration testing

Se deberán probar:

```text
identity deduplication
JOIN deduplication
partial fields
NULL handling
composite IDs
polymorphism
relationship assembly
snapshot correctness
postLoad mutation
read-only hydration
DTO hydration
tuple hydration
scalar hydration
streaming cleanup
cancellation
failure cleanup
persistent runtime reset
```

---

# 277. Performance regression tests

Podrán establecer:

```text
maximum allocation growth
maximum relative latency regression
maximum memory regression
```

con tolerancias.

---

# 278. Absolute benchmark caution

No deberá fallarse CI únicamente porque:

```text
operation > 5ms
```

en hardware distinto.

---

# 279. Relative comparisons

Será preferible evaluar:

```text
baseline
vs
candidate
```

en entorno controlado.

---

# 280. Statistical benchmarks

Documento 250 definirá metodología formal.

---

# 281. Estructura de directorios

```text
src/Quantum/Database/Performance/Hydration/
│
├── Contract/
│   ├── HydrationPerformanceManager.php
│   ├── HydrationPerformanceAnalyzer.php
│   ├── HydrationPerformancePolicy.php
│   └── HydrationPerformanceRecorder.php
│
├── Context/
│   └── HydrationPerformanceContext.php
│
├── Budget/
│   └── HydrationPerformanceBudget.php
│
├── Model/
│   ├── HydrationOperationId.php
│   ├── HydrationPerformanceMeasurement.php
│   ├── HydrationPerformanceMeasurementMode.php
│   ├── HydrationIdentityStatistics.php
│   ├── HydrationConversionStatistics.php
│   ├── HydrationAssignmentStatistics.php
│   ├── HydrationRelationshipStatistics.php
│   ├── HydrationMemoryStatistics.php
│   └── HydrationOutcome.php
│
├── Measurement/
│   ├── DefaultHydrationPerformanceManager.php
│   ├── HydrationPerformanceSession.php
│   ├── HydrationCounterSet.php
│   └── HydrationMemoryObserver.php
│
├── Analysis/
│   ├── DefaultHydrationPerformanceAnalyzer.php
│   ├── HydrationAmplificationAnalyzer.php
│   ├── HydrationAllocationAnalyzer.php
│   ├── HydrationConversionAnalyzer.php
│   ├── HydrationIdentityAnalyzer.php
│   ├── HydrationRelationshipAnalyzer.php
│   └── HydrationMemoryAnalyzer.php
│
├── Finding/
│   ├── HydrationPerformanceFinding.php
│   ├── HydrationPerformanceFindingCode.php
│   ├── HighAmplificationDetector.php
│   ├── CartesianAmplificationDetector.php
│   ├── HighAllocationDetector.php
│   ├── SlowConversionDetector.php
│   ├── SnapshotCostDetector.php
│   ├── RelationshipAssemblyCostDetector.php
│   └── HydrationMemoryPressureDetector.php
│
├── Telemetry/
│   └── HydrationPerformanceTelemetryBridge.php
│
├── Testing/
│   ├── FakeHydrationPerformanceRecorder.php
│   ├── HydrationPerformanceAssertions.php
│   └── HydrationBenchmarkFixture.php
│
└── Exception/
    ├── DatabaseHydrationPerformanceException.php
    ├── HydrationPerformanceMeasurementException.php
    ├── HydrationPerformanceBudgetException.php
    └── HydrationPerformanceInstrumentationException.php
```

---

# 282. Integración con Hydration Engine

```text
Quantum/Database/Hydration
             │
             ├── Plan
             ├── Entity
             ├── Scalar
             ├── Tuple
             ├── Result
             └── Cache
                    │
                    ▼
          Performance/Hydration
```

La dependencia deberá diseñarse para evitar que el core de hidratación dependa de una implementación pesada de telemetry.

---

# 283. No-op implementation

Cuando performance instrumentation esté desactivada:

```php
final class NullHydrationPerformanceRecorder
    implements HydrationPerformanceRecorder
{
}
```

---

# 284. Hot-path optimization

El `NullHydrationPerformanceRecorder` deberá permitir que:

```text
OFF
```

tenga costo mínimo.

---

# 285. Conditional instrumentation

La implementación podrá seleccionar previamente:

```text
OFF path
STANDARD path
PROFILE path
```

en vez de comprobar el modo para cada campo.

---

# 286. Example

Preferible:

```text
choose hydrator instrumentation mode once
    ↓
run hot loop
```

frente a:

```text
for each field
    if profiling enabled...
```

---

# 287. Compiled instrumentation

En el futuro, compiled hydrators podrían generarse:

```text
without instrumentation
```

o:

```text
with counters
```

según entorno.

---

# 288. Complexity model

Para hidratación simple:

```text
R = rows
F = fields per row
```

costo base aproximado:

```text
O(R × F)
```

---

# 289. Entity graph hydration

Con relaciones:

```text
O(R × F + R × I + A)
```

donde:

```text
I = identity/relationship lookup cost
A = assembly operations
```

con hash lookups esperados O(1).

---

# 290. Bad collection implementation

Si membership usa linear scan:

```text
O(R × C)
```

y puede aproximarse a:

```text
O(N²)
```

para colecciones grandes.

Esto deberá evitarse.

---

# 291. Performance priorities

Orden general:

```text
1. Correctness
2. Bounded resource behavior
3. Avoid algorithmic pathologies
4. Reduce unnecessary work
5. Reduce allocations
6. Optimize hot loops
7. Micro-optimize only after benchmarks
```

---

# 292. Anti-patterns

Quedan desaconsejados:

```text
Parse metadata for every row

Use Reflection for every field assignment

Resolve TypeRegistry for every cell

Allocate duplicate entities from JOIN rows

Ignore IdentityMap during entity hydration

Use linear collection membership checks

Create snapshots for DTOs

Register DTOs in UnitOfWork

Register scalars in EntityManager

Treat NULL as missing field

Treat missing field as NULL

Mark partial relationship COMPLETE

Hydrate full entity when projection is enough

Use partial managed entities as universal optimization

Materialize millions of rows unnecessarily

Assume streaming means constant memory

Assume chunk means bounded ORM state

Emit telemetry event per row

Emit telemetry event per field

Log hydrated values

Cache entities inside HydrationPlan

Share HydrationSession between requests

Share relationship assembly registries between requests

Leave failed RESERVED identities active

Execute hidden queries during hydration

Use destructor as primary resource cleanup

Generate code without benchmarking it

Assume arrays always use less memory than DTOs

Assume fewer allocations always means faster code

Assume numeric row indexes always outperform associative rows

Ignore driver-specific raw types

Skip type conversion without proving equivalence

Ignore postLoad semantics for speed

Overwrite dirty managed entities

Ignore tenant/shard identity context

Treat UNKNOWN cardinality as zero

Use rowCount() as universal SELECT cardinality

Assume high throughput alone means good hydration
```

---

# 293. Invariantes arquitectónicas

## DB-HYDPERF-001
Hydration Performance no ejecutará queries.

## DB-HYDPERF-002
Hydration Performance no generará SQL.

## DB-HYDPERF-003
Hydration Performance no será ORM Performance.

## DB-HYDPERF-004
Hydration Performance no será Query Performance.

## DB-HYDPERF-005
Hydration Performance no será Serialization.

## DB-HYDPERF-006
Hydration Performance preservará Type System.

## DB-HYDPERF-007
Hydration Performance preservará canonical identity.

## DB-HYDPERF-008
Hydration Performance preservará LoadedFieldMask.

## DB-HYDPERF-009
Hydration Performance preservará snapshots.

## DB-HYDPERF-010
Hydration Performance preservará relationship coverage.

## DB-HYDPERF-011
Hydration Performance preservará EntityState.

## DB-HYDPERF-012
Hydration Performance preservará UnitOfWork semantics.

## DB-HYDPERF-013
Rows no serán equivalentes a entities.

## DB-HYDPERF-014
Hydration amplification será observable.

## DB-HYDPERF-015
HydrationPlan será distinto de QueryPlan.

## DB-HYDPERF-016
HydrationPlan podrá pre-resolver hot-path metadata.

## DB-HYDPERF-017
HydrationPlanCache no almacenará entities.

## DB-HYDPERF-018
HydrationPlanCache será generation-aware.

## DB-HYDPERF-019
Reflection repetitiva será evitada en row loops.

## DB-HYDPERF-020
Attribute parsing repetitivo será evitado.

## DB-HYDPERF-021
Property accessors podrán precompilarse.

## DB-HYDPERF-022
Generated accessors requerirán benchmarks.

## DB-HYDPERF-023
Entity constructor semantics no serán sacrificadas.

## DB-HYDPERF-024
Object allocation será observable.

## DB-HYDPERF-025
Temporary allocations deberán minimizarse cuando sea seguro.

## DB-HYDPERF-026
Mutable state no será reutilizado inseguramente.

## DB-HYDPERF-027
Column mappings podrán pre-resolverse.

## DB-HYDPERF-028
Type handlers podrán pre-resolverse.

## DB-HYDPERF-029
Driver raw type no se asumirá universal.

## DB-HYDPERF-030
Fast-path conversion requerirá equivalencia semántica.

## DB-HYDPERF-031
Custom Type conversion será observable.

## DB-HYDPERF-032
NULL no será missing.

## DB-HYDPERF-033
Missing no será NULL.

## DB-HYDPERF-034
LoadedFieldMask será correcto.

## DB-HYDPERF-035
Identity será extraída antes de crear managed entity cuando aplique.

## DB-HYDPERF-036
Composite IDs serán soportados.

## DB-HYDPERF-037
Tenant/shard context formará parte de identidad cuando corresponda.

## DB-HYDPERF-038
IdentityMap HIT reutilizará canonical instance.

## DB-HYDPERF-039
IdentityMap MISS podrá reservar identidad antes de graph assembly.

## DB-HYDPERF-040
RESERVED y ACTIVE serán estados distintos.

## DB-HYDPERF-041
Dirty managed entity no será sobrescrita ciegamente.

## DB-HYDPERF-042
JOIN duplicates no crearán duplicate managed entities.

## DB-HYDPERF-043
RootResultRegistry no será IdentityMap.

## DB-HYDPERF-044
RelationshipAssemblyRegistry será session-scoped.

## DB-HYDPERF-045
No existirá deduplication static state.

## DB-HYDPERF-046
Relationship assembly evitará O(N²) cuando sea posible.

## DB-HYDPERF-047
Assembly indexes se liberarán al terminar.

## DB-HYDPERF-048
Cartesian amplification será diagnosticable.

## DB-HYDPERF-049
Hydrator no cambiará query strategy.

## DB-HYDPERF-050
Snapshot representará baseline persistente.

## DB-HYDPERF-051
Snapshot será establecido conforme a lifecycle semantics.

## DB-HYDPERF-052
postLoad mutation podrá producir dirty state.

## DB-HYDPERF-053
Read-only hydration podrá reducir tracking.

## DB-HYDPERF-054
Read-only entity no será DTO.

## DB-HYDPERF-055
DTO no será Entity.

## DB-HYDPERF-056
DTO no entrará en IdentityMap por defecto.

## DB-HYDPERF-057
DTO no entrará en UnitOfWork por defecto.

## DB-HYDPERF-058
Projection no será managed entity.

## DB-HYDPERF-059
Scalar hydration tendrá specialized path.

## DB-HYDPERF-060
Tuple hydration tendrá specialized path.

## DB-HYDPERF-061
Array hydration tendrá specialized path.

## DB-HYDPERF-062
DTO hydration tendrá specialized path.

## DB-HYDPERF-063
Entity hydration tendrá specialized path.

## DB-HYDPERF-064
Graph hydration tendrá specialized path.

## DB-HYDPERF-065
Specialized hydrators podrán compartir infraestructura.

## DB-HYDPERF-066
Generic fallback permanecerá disponible.

## DB-HYDPERF-067
Branch optimization no omitirá correctness checks.

## DB-HYDPERF-068
Compiled hydrators serán opcionales.

## DB-HYDPERF-069
Generated PHP no será requisito V1.

## DB-HYDPERF-070
Compiled plans podrán sobrevivir requests si son inmutables.

## DB-HYDPERF-071
HydrationSession no sobrevivirá requests.

## DB-HYDPERF-072
ResultCursor no será compartido entre requests.

## DB-HYDPERF-073
Current entity no será shared state.

## DB-HYDPERF-074
Streaming hydration no implicará constant memory.

## DB-HYDPERF-075
Streaming managed entities podrán hacer crecer IdentityMap.

## DB-HYDPERF-076
Consumer retention podrá hacer crecer memoria.

## DB-HYDPERF-077
Chunk release no implicará entity detach.

## DB-HYDPERF-078
Hydration batch no será driver fetch buffer.

## DB-HYDPERF-079
Batch size no será universal.

## DB-HYDPERF-080
Resource ownership será explícito.

## DB-HYDPERF-081
Early termination cerrará recursos owned.

## DB-HYDPERF-082
Destructor no será primary correctness mechanism.

## DB-HYDPERF-083
Cancellation será soportada.

## DB-HYDPERF-084
Cancellation checks evitarán overhead excesivo.

## DB-HYDPERF-085
Memory measurements podrán ser estimadas.

## DB-HYDPERF-086
Estimated memory será identificada como tal.

## DB-HYDPERF-087
GC no será confundido con hydration cleanup.

## DB-HYDPERF-088
Arrays no se asumirán más ligeros que DTOs.

## DB-HYDPERF-089
Entity hydration incluirá tracking costs cuando aplique.

## DB-HYDPERF-090
Large materialized results serán detectables.

## DB-HYDPERF-091
Performance analyzer no cambiará get() a lazy().

## DB-HYDPERF-092
rowCount no será cardinalidad universal de SELECT.

## DB-HYDPERF-093
UNKNOWN cardinality no será cero.

## DB-HYDPERF-094
Hydration failures podrán ocurrir después de trabajo parcial.

## DB-HYDPERF-095
Failed reserved identity no quedará ACTIVE.

## DB-HYDPERF-096
Failure cleanup preservará IdentityMap invariants.

## DB-HYDPERF-097
Instrumentation failure no será hydration failure por defecto.

## DB-HYDPERF-098
Performance telemetry no será correctness dependency.

## DB-HYDPERF-099
Performance findings tendrán evidence.

## DB-HYDPERF-100
Hydration shape será parte del análisis.

## DB-HYDPERF-101
Scalar y entity graph throughput no se compararán ingenuamente.

## DB-HYDPERF-102
Telemetry tendrá cardinalidad bounded.

## DB-HYDPERF-103
Entity IDs no serán metric labels.

## DB-HYDPERF-104
PII no será metric label.

## DB-HYDPERF-105
No habrá Event System event por fila en production default.

## DB-HYDPERF-106
Counters serán agregados localmente.

## DB-HYDPERF-107
Debug output será redacted.

## DB-HYDPERF-108
Hydrated sensitive values no serán registrados.

## DB-HYDPERF-109
Tenant context permanecerá estable.

## DB-HYDPERF-110
Cross-tenant hydration requerirá operación explícita.

## DB-HYDPERF-111
Shard identity context será preservado.

## DB-HYDPERF-112
Same local ID on different shards no colisionará.

## DB-HYDPERF-113
Distributed merge no será responsabilidad del Hydrator.

## DB-HYDPERF-114
Cross-shard relationships no se fingirán locales.

## DB-HYDPERF-115
PARTIAL relationship no será marcada COMPLETE.

## DB-HYDPERF-116
Absent optional relation evitará entity allocation.

## DB-HYDPERF-117
Composite NULL identity seguirá metadata.

## DB-HYDPERF-118
Discriminator resolution utilizará registry seguro.

## DB-HYDPERF-119
DB discriminator no podrá instanciar FQCN arbitrario.

## DB-HYDPERF-120
Unknown discriminator producirá error.

## DB-HYDPERF-121
Polymorphic hydration podrá usar compiled dispatch.

## DB-HYDPERF-122
Value Object allocations serán observables cuando aplique.

## DB-HYDPERF-123
Value Objects no serán interned globalmente de forma insegura.

## DB-HYDPERF-124
JSON conversion será observable.

## DB-HYDPERF-125
Hydrator no reescribirá query para optimizar JSON.

## DB-HYDPERF-126
Temporal converters podrán reutilizar metadata inmutable.

## DB-HYDPERF-127
Temporal mutable objects no serán compartidos.

## DB-HYDPERF-128
Enum mapping será pre-resolvable.

## DB-HYDPERF-129
Custom Hydrator respetará resource ownership.

## DB-HYDPERF-130
Custom Hydrator respetará cancellation.

## DB-HYDPERF-131
Custom Hydrator respetará security.

## DB-HYDPERF-132
Custom Hydrator podrá integrarse a performance telemetry.

## DB-HYDPERF-133
Hydration extensions no ejecutarán SQL directamente.

## DB-HYDPERF-134
Hydration no hará hidden queries.

## DB-HYDPERF-135
Hydration será I/O-free respecto a nuevas consultas DB.

## DB-HYDPERF-136
Lifecycle callback cost será separable de core hydration.

## DB-HYDPERF-137
Callback query será correlacionable.

## DB-HYDPERF-138
Hydration budgets serán configurables.

## DB-HYDPERF-139
Budgets podrán variar por workload.

## DB-HYDPERF-140
Exceder budget no cambiará strategy silenciosamente.

## DB-HYDPERF-141
Explain distinguirá estimated y actual.

## DB-HYDPERF-142
Hydration analyzer será read-only.

## DB-HYDPERF-143
Analyzer no reejecutará queries.

## DB-HYDPERF-144
Analyzer no hará clear de EntityManager.

## DB-HYDPERF-145
Analyzer no hará detach.

## DB-HYDPERF-146
Resource Governance controlará online guardrails.

## DB-HYDPERF-147
Hydration benchmarks incluirán multiple shapes.

## DB-HYDPERF-148
Benchmarks incluirán multiple cardinalities.

## DB-HYDPERF-149
Benchmarks incluirán complex types.

## DB-HYDPERF-150
Benchmarks incluirán IdentityMap hit ratios.

## DB-HYDPERF-151
Benchmarks incluirán JOIN amplification.

## DB-HYDPERF-152
Benchmarks distinguirán cold y warm plans.

## DB-HYDPERF-153
FrankenPHP será runtime principal de benchmark.

## DB-HYDPERF-154
Persistent runtime tests comprobarán cleanup.

## DB-HYDPERF-155
Compiled plans solo se compartirán si son context-safe.

## DB-HYDPERF-156
Tenant mutable state no estará en shared plan.

## DB-HYDPERF-157
Hydration correctness tendrá tests específicos.

## DB-HYDPERF-158
Performance regressions se medirán con metodología controlada.

## DB-HYDPERF-159
CI evitará thresholds absolutos ingenuos.

## DB-HYDPERF-160
Performance improvements deberán validarse con benchmarks.

## DB-HYDPERF-161
No-op instrumentation tendrá overhead mínimo.

## DB-HYDPERF-162
Instrumentation mode podrá seleccionarse antes del hot loop.

## DB-HYDPERF-163
PROFILE no será production default.

## DB-HYDPERF-164
Core hydration complexity buscará O(R×F).

## DB-HYDPERF-165
Relationship assembly evitará patologías O(N²).

## DB-HYDPERF-166
Correctness tendrá prioridad sobre micro-optimization.

## DB-HYDPERF-167
Bounded resource behavior tendrá prioridad sobre micro-optimization.

## DB-HYDPERF-168
Algorithmic optimization tendrá prioridad sobre micro-optimization.

## DB-HYDPERF-169
Allocation optimization será posterior a correctness.

## DB-HYDPERF-170
Hot-loop optimization requerirá evidencia.

## DB-HYDPERF-171
Hydration no será considerada gratuita.

## DB-HYDPERF-172
Hydration cost será observable.

## DB-HYDPERF-173
Hydration memory será observable.

## DB-HYDPERF-174
Hydration amplification será observable.

## DB-HYDPERF-175
Type conversion cost será observable.

## DB-HYDPERF-176
Identity resolution cost será observable.

## DB-HYDPERF-177
Snapshot cost será observable.

## DB-HYDPERF-178
Relationship assembly cost será observable.

## DB-HYDPERF-179
Allocation cost será observable.

## DB-HYDPERF-180
Hydration performance nunca debilitará hydration semantics.

---

# 294. Modelo formal

Sea:

```text
R = {r1, r2, ..., rn}
```

el conjunto ordenado de filas físicas.

Sea:

```text
H
```

el Hydration Plan.

Entonces:

```text
Hydrate(R, H, C)
→
A
```

donde:

```text
C = Hydration Context
A = Application Result
```

---

# 295. Entity identity

Para cada fila `r`:

```text
id(r)
```

representa la identidad extraída.

Si:

```text
IdentityMap[id(r)] = e
```

entonces:

```text
hydrateEntity(r)
→
reuse(e)
```

en lugar de crear otra instancia managed.

---

# 296. Allocation bound

Para una entidad repetida `k` veces por JOIN:

```text
rows(entity) = k
```

pero idealmente:

```text
entity allocations = 1
```

dentro del scope.

---

# 297. Root result deduplication

Si:

```text
RootIdentity(ri)
=
RootIdentity(rj)
```

la colección raíz deberá contener una sola entrada cuando la semántica del result shape lo requiera.

---

# 298. Throughput

```text
ThroughputRows
=
n / T
```

y:

```text
ThroughputResults
=
|A| / T
```

cuando `A` sea cardinalizable.

---

# 299. Amplification

```text
A_h
=
PhysicalRows
/
LogicalRoots
```

para `LogicalRoots > 0`.

---

# 300. Empty result

Para:

```text
LogicalRoots = 0
```

amplification no deberá dividir entre cero.

Se representará:

```text
NOT_APPLICABLE
```

---

# 301. Allocation amplification

```text
A_alloc
=
AllocatedObjects
/
LogicalResults
```

cuando sea semánticamente útil.

---

# 302. Performance target

El objetivo no será:

```text
minimize T at any cost
```

sino:

```text
minimize
(
  latency,
  CPU,
  allocations,
  memory,
  redundant work
)
```

sujeto a:

```text
correctness constraints
+
resource constraints
+
ORM invariants
```

---

# 303. Pipeline final

```text
                        QUERY ENGINE
                             │
                             ▼
                      EXECUTION ENGINE
                             │
                             ▼
                          RESULT
                             │
                             ▼
                    HYDRATION PLAN
                             │
             ┌───────────────┼───────────────┐
             ▼               ▼               ▼
       Row Mapping      Type Mapping    Identity Mapping
             │               │               │
             └───────────────┼───────────────┘
                             ▼
                       HOT HYDRATION LOOP
                             │
             ┌───────────────┼────────────────┐
             ▼               ▼                ▼
         Allocate         Assign          Deduplicate
             │               │                │
             └───────────────┼────────────────┘
                             ▼
                    Relationship Assembly
                             │
                             ▼
                         Snapshot
                             │
                             ▼
                    UnitOfWork Registration
                             │
                             ▼
                     APPLICATION RESULT

                ┌──────────────────────────┐
                │ Hydration Performance    │
                │                          │
                │ rows                     │
                │ allocations              │
                │ conversions              │
                │ identity                 │
                │ relationships            │
                │ snapshots                │
                │ memory                   │
                │ duration                 │
                └──────────────────────────┘
```

---

# 304. Resultado arquitectónico

VoltStack podrá diferenciar claramente una consulta rápida de una hidratación costosa.

Ejemplo:

```text
DATABASE OPERATION

Query Execution
    18.4 ms

Hydration
    126.7 ms

Total
    145.1 ms
```

y explicar:

```text
HYDRATION ANALYSIS

Shape
    ENTITY_GRAPH

Physical Rows
    94,820

Root Entities
    400

Entities Created
    12,341

Entities Reused
    82,479

Type Conversions
    613,214

Snapshots
    12,341

Relationship Membership Operations
    91,734

Hydration Amplification
    237x

Memory Peak
    186 MB

Findings
    HIGH_HYDRATION_AMPLIFICATION
    CARTESIAN_HYDRATION_AMPLIFICATION
    HYDRATION_MEMORY_PRESSURE

Recommendation
    Evaluate splitting collection JOINs into
    SELECT_IN/BATCH relationship loading.
```

Esto permitirá detectar un problema que un simple:

```text
slow query detector
```

no podría explicar correctamente.

---

# 305. Filosofía final

VoltStack deberá considerar la hidratación como una fase de ejecución de alto rendimiento por derecho propio.

No será tratada simplemente como:

```text
foreach row → new Entity
```

sino como:

```text
Result
   ↓
Precompiled Structural Knowledge
   ↓
Canonical Type Conversion
   ↓
Identity Resolution
   ↓
Minimal Necessary Allocation
   ↓
Efficient Assignment
   ↓
Graph Deduplication
   ↓
Relationship Assembly
   ↓
Snapshot / State Registration
   ↓
Application Result
```

La meta será:

> **hacer que el costo de hidratación crezca principalmente con la cantidad de información que realmente debe transformarse, evitando trabajo estructural repetitivo, allocations redundantes, búsquedas innecesarias y explosiones algorítmicas.**

---

# 306. Regla arquitectónica final

```text
Fast Hydration
≠
Unsafe Hydration

Rows
≠
Entities

Hydration
≠
Query Execution

Hydration
≠
ORM

Hydration Plan
≠
Query Plan

IdentityMap
≠
Dedup Registry

NULL
≠
Missing

PARTIAL
≠
COMPLETE

DTO
≠
Entity

Projection
≠
Partial Managed Entity

Streaming
≠
Constant Memory

Chunk
≠
Bounded ORM State

Generated Code
≠
Automatically Faster

Fewer Allocations
≠
Automatically Faster

More JOINs
≠
Better Hydration

Performance
≠
Weaker Semantics
```

---

# 307. Relación con el Bloque 24

```text
242 DATABASE PERFORMANCE ARCHITECTURE
             │
             ├── 243 QUERY PERFORMANCE
             │
             ├── 244 ORM PERFORMANCE
             │
             ├── 245 HYDRATION PERFORMANCE
             │       │
             │       ├── Hydration Plans
             │       ├── Row Processing
             │       ├── Type Conversion
             │       ├── Identity Resolution
             │       ├── Allocation
             │       ├── Assignment
             │       ├── Deduplication
             │       ├── Relationship Assembly
             │       ├── Snapshots
             │       └── Memory
             │
             ├── 246 METADATA COMPILATION
             ├── 247 QUERY COMPILATION OPTIMIZATION
             ├── 248 MEMORY MANAGEMENT
             ├── 249 RESOURCE GOVERNANCE
             └── 250 PERFORMANCE BENCHMARK
```

---

# 308. Estado del Bloque 24

```text
BLOCK 24 — PERFORMANCE

✓ 242_DATABASE_PERFORMANCE_ARCHITECTURE.md
✓ 243_DATABASE_QUERY_PERFORMANCE_SYSTEM.md
✓ 244_DATABASE_ORM_PERFORMANCE_SYSTEM.md
✓ 245_DATABASE_HYDRATION_PERFORMANCE_SYSTEM.md
│
├── 246_DATABASE_METADATA_COMPILATION_SYSTEM.md
├── 247_DATABASE_QUERY_COMPILATION_OPTIMIZATION_SYSTEM.md
├── 248_DATABASE_MEMORY_MANAGEMENT_SYSTEM.md
├── 249_DATABASE_RESOURCE_GOVERNANCE_SYSTEM.md
└── 250_DATABASE_PERFORMANCE_BENCHMARK_SYSTEM.md
```

---

# 309. Siguiente documento

```text
246_DATABASE_METADATA_COMPILATION_SYSTEM.md
```

El siguiente documento deberá formalizar cómo VoltStack transforma metadata declarativa relativamente costosa:

```text
PHP Classes
Attributes
Mappings
Relationships
Types
Identifiers
Lifecycle Definitions
Accessors
Persistence Metadata
```

en estructuras:

```text
compiled
immutable
validated
indexed
cacheable
runtime-efficient
```

para evitar que operaciones como:

```text
reflection
attribute parsing
mapping validation
relationship resolution
type resolution
property discovery
```

se repitan durante:

```text
query building
hydration
dirty checking
relationship loading
flush
persistence planning
```

bajo la regla:

> **VoltStack deberá pagar el costo de interpretar metadata principalmente durante bootstrap/compilación y no repetidamente en los hot paths de ejecución de cada entidad, query o fila.**