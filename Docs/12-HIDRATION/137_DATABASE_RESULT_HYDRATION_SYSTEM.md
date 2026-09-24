# 137_DATABASE_RESULT_HYDRATION_SYSTEM.md

# VoltStack Quantum Database
## Database Result Hydration System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 137 — Database Result Hydration System  
**Bloque:** 12 — Hydration  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Result Hydration System` define la arquitectura responsable de transformar un resultado físico producido por el `Execution Engine` en el resultado lógico solicitado por una consulta ORM.

Su responsabilidad fundamental es:

```text
Physical Result
      │
      ▼
ResultHydrator
      │
      ├── Row iteration
      ├── Result shape interpretation
      ├── Root identity resolution
      ├── Root deduplication
      ├── JOIN amplification handling
      ├── Relationship assembly
      ├── Scalar extraction
      ├── Tuple assembly
      ├── Projection assembly
      ├── DTO construction
      ├── Cardinality enforcement
      └── Resource lifecycle
      │
      ▼
Logical Result
```

La idea central es que:

```text
Database Row
≠
Logical ORM Result
```

Una sola entidad puede estar representada por múltiples rows físicas.

Una sola row puede representar múltiples elementos lógicos:

```text
Entity
+
Scalar
+
Projection
```

dependiendo del `HydrationPlan`.

---

# 2. Principio central

> **`ResultHydrator` interpreta la estructura lógica de un conjunto de resultados; nunca debe asumir que la cardinalidad física de las filas coincide con la cardinalidad lógica del resultado ORM.**

Formalmente:

```text
PhysicalRowCount
≠
LogicalResultCount
```

en general.

Ejemplo:

```text
SQL result:

User#1 Order#10
User#1 Order#11
User#1 Order#12
```

contiene:

```text
Physical rows = 3
```

pero puede representar:

```text
Logical root entities = 1
```

---

# 3. Posición arquitectónica

```text
Entity Query
     │
     ▼
Query Model
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
Result
     │
     ▼
┌────────────────────────────┐
│ Result Hydration System    │
└─────────────┬──────────────┘
              │
       ┌──────┼───────────┐
       ▼      ▼           ▼
    Entity  Scalar     Projection
   Hydrator Hydrator    Hydrator
       │
       ▼
 IdentityMap
       │
       ▼
 Logical Result
```

---

# 4. ResultHydrator ≠ EntityHydrator

El documento anterior definió:

```text
EntityHydrator
```

como responsable de:

```text
row/entity fragment
→
canonical entity
```

El `ResultHydrator` opera un nivel superior:

```text
Result
→
logical result structure
```

---

# 5. Diferencia fundamental

```text
EntityHydrator
    ↓
materializa una entidad

ResultHydrator
    ↓
materializa la forma completa del resultado
```

---

# 6. Ejemplo

Consulta:

```php
$users = User::query()
    ->with('orders')
    ->get();
```

SQL conceptual:

```sql
SELECT ...
FROM users
LEFT JOIN orders ...
```

Resultado:

```text
User#1 Order#10
User#1 Order#11
User#2 Order#20
User#3 NULL
```

`EntityHydrator` resuelve:

```text
User#1
Order#10
Order#11
User#2
Order#20
User#3
```

`ResultHydrator` construye:

```text
[
    User#1 {
        orders: [Order#10, Order#11]
    },

    User#2 {
        orders: [Order#20]
    },

    User#3 {
        orders: []
    }
]
```

---

# 7. ResultHydrator ≠ Result

`Result` pertenece al Execution Engine.

Representa acceso físico al resultado.

Puede proporcionar:

```text
rows
cursor
column metadata
affected rows
driver result state
```

El `ResultHydrator` interpreta esos datos.

---

# 8. ResultHydrator ≠ Query Executor

No ejecuta SQL.

No prepara statements.

No administra bindings.

No selecciona conexiones.

---

# 9. ResultHydrator ≠ SQL Compiler

No conoce cómo se generó el SQL.

Trabaja sobre:

```text
Result
+
CompiledResultHydrationPlan
```

---

# 10. ResultHydrator ≠ Serializer

Su salida puede ser:

- entidades;
- escalares;
- arrays;
- tuples;
- DTOs;
- projections;
- streams.

No representa necesariamente una estructura serializable.

---

# 11. ResultHydrationRequest

Contrato conceptual:

```php
final readonly class ResultHydrationRequest
{
    public function __construct(
        public Result $result,
        public CompiledResultHydrationPlan $plan,
        public ResultHydrationContext $context,
    ) {}
}
```

---

# 12. ResultHydrationContext

```php
final readonly class ResultHydrationContext
{
    public function __construct(
        public HydrationScope $scope,
        public PersistenceContext $persistenceContext,
        public ResultHydrationMode $mode,
        public ResultCardinalityExpectation $cardinality,
    ) {}
}
```

---

# 13. CompiledResultHydrationPlan

El plan describe cómo interpretar cada row.

```php
final readonly class CompiledResultHydrationPlan
{
    public function __construct(
        public ResultShape $shape,
        public array $components,
        public ?RootHydrationPlan $root,
        public ResultGroupingPlan $grouping,
        public ResultDeduplicationPlan $deduplication,
        public ResultCardinalityPlan $cardinality,
        public ResultStreamingPlan $streaming,
    ) {}
}
```

---

# 14. Result Shape

VoltStack deberá modelar explícitamente la forma lógica esperada.

```php
enum ResultShape
{
    case ENTITY;
    case ENTITY_COLLECTION;
    case SCALAR;
    case SCALAR_COLLECTION;
    case TUPLE;
    case TUPLE_COLLECTION;
    case PROJECTION;
    case PROJECTION_COLLECTION;
    case DTO;
    case DTO_COLLECTION;
    case MIXED;
}
```

---

# 15. ENTITY

Ejemplo:

```php
$user = $query->one();
```

Salida:

```text
User
```

---

# 16. ENTITY_COLLECTION

```php
$users = $query->get();
```

Salida:

```text
Collection<User>
```

---

# 17. SCALAR

```php
$count = User::query()->count();
```

Resultado:

```text
42
```

No debe crear entidades.

---

# 18. SCALAR_COLLECTION

Ejemplo:

```text
SELECT email FROM users
```

Resultado lógico:

```text
[
    "a@example.com",
    "b@example.com",
    "c@example.com"
]
```

---

# 19. TUPLE

Una tuple representa múltiples componentes lógicos.

Ejemplo conceptual:

```text
(User, OrderCount)
```

---

# 20. Tuple result

```text
[
    User#1,
    15
]
```

o preferentemente mediante un tipo explícito:

```php
final readonly class ResultTuple
{
    public function __construct(
        private array $values,
    ) {}
}
```

---

# 21. PROJECTION

Una projection contiene únicamente campos seleccionados.

Ejemplo:

```php
User::query()
    ->select('id', 'name')
    ->projection();
```

Resultado:

```text
UserProjection {
    id
    name
}
```

No necesariamente es una entidad managed.

---

# 22. DTO

Consulta:

```text
User
+
Order statistics
```

podría producir:

```php
new UserOrderSummary(
    userId: ...,
    userName: ...,
    orders: ...,
);
```

---

# 23. DTO ≠ Entity

Un DTO no deberá:

```text
enter IdentityMap
```

salvo que contenga entidades como componentes separados.

---

# 24. MIXED

VoltStack podrá soportar resultados como:

```text
Entity + Scalar
```

Ejemplo:

```text
User#1 | order_count=15
User#2 | order_count=7
```

---

# 25. Result component model

Un resultado podrá componerse de:

```text
Root
├── EntityComponent
├── ScalarComponent
├── ProjectionComponent
├── DTOComponent
└── RelationshipComponent
```

---

# 26. ResultComponent

Contrato conceptual:

```php
interface ResultComponentHydrator
{
    public function hydrate(
        ResultRow $row,
        ResultComponentPlan $plan,
        ResultHydrationContext $context
    ): ComponentHydrationResult;
}
```

---

# 27. Component types

```text
ENTITY
SCALAR
TUPLE
PROJECTION
DTO
RELATIONSHIP
EMBEDDED
```

---

# 28. Result iteration

Pipeline general:

```text
Result
  │
  ▼
open
  │
  ▼
fetch row
  │
  ▼
interpret row
  │
  ▼
hydrate components
  │
  ▼
resolve root
  │
  ▼
deduplicate root
  │
  ▼
assemble relationships
  │
  ▼
accumulate / yield
  │
  ▼
next row
  │
  ▼
finalize
  │
  ▼
enforce cardinality
  │
  ▼
close resources
```

---

# 29. Physical Row

El sistema utilizará una abstracción:

```php
interface ResultRow
{
    public function get(ResultSlot $slot): mixed;

    public function has(ResultSlot $slot): bool;
}
```

---

# 30. Positional access

Para rendimiento:

```text
slot #0
slot #1
slot #2
```

deberá ser preferible a buscar nombres repetidamente.

---

# 31. ResultSlot

```php
final readonly class ResultSlot
{
    public function __construct(
        public int $index,
        public ?string $alias = null,
    ) {}
}
```

---

# 32. Physical result metadata

El plan podrá validar:

```text
expected columns
actual columns
column order
aliases
types
```

cuando sea necesario.

---

# 33. Root concept

En resultados ORM complejos se define:

```text
RootResultComponent
```

Ejemplo:

```text
User
```

en:

```text
User JOIN Orders
```

---

# 34. Root identity

La identidad raíz normalmente se determina mediante:

```text
EntityKey
```

---

# 35. Root grouping

Rows consecutivas pueden pertenecer al mismo root.

```text
Row 1 → User#1
Row 2 → User#1
Row 3 → User#1
Row 4 → User#2
```

---

# 36. ResultGroupingPlan

```php
final readonly class ResultGroupingPlan
{
    public function __construct(
        public ResultGroupingStrategy $strategy,
        public bool $requiresStableOrdering,
    ) {}
}
```

---

# 37. Grouping strategies

```text
NONE
IDENTITY
ORDERED_IDENTITY
BUFFERED_IDENTITY
CUSTOM
```

---

# 38. NONE

Utilizado cuando:

```text
one row
=
one logical result
```

Ejemplo:

```text
scalar collection
```

---

# 39. IDENTITY

Agrupa ocurrencias por identidad lógica.

---

# 40. ORDERED_IDENTITY

Asume que todas las rows de una root aparecen contiguamente.

Esto permite streaming eficiente.

---

# 41. BUFFERED_IDENTITY

Permite que:

```text
User#1
User#2
User#1
```

aparezcan fuera de orden.

Requiere buffering adicional.

---

# 42. Grouping precondition

Si un plan utiliza:

```text
ORDERED_IDENTITY
```

deberá existir evidencia de que el resultado cumple esa precondición.

---

# 43. Hydrator no cambia SQL

Si falta orden requerido:

```text
ResultHydrator
```

no añadirá:

```sql
ORDER BY
```

ocultamente.

---

# 44. Planner responsibility

La necesidad de ordering deberá propagarse previamente al:

```text
Query Planner
```

cuando la estrategia lo requiera.

---

# 45. Root deduplication

Ejemplo:

```text
User#1 Order#10
User#1 Order#11
User#1 Order#12
```

produce:

```text
RootResult:
User#1
```

una sola vez.

---

# 46. Deduplication ≠ DISTINCT

`ResultHydrator` deduplication es semántica ORM.

No equivale a:

```sql
SELECT DISTINCT
```

---

# 47. SQL DISTINCT

Opera sobre rows SQL.

---

# 48. ORM deduplication

Opera sobre:

```text
logical identities
```

---

# 49. Example

```text
User#1 Order#10
User#1 Order#11
```

son rows SQL distintas.

Pero:

```text
RootEntityKey = User#1
```

es igual.

---

# 50. ResultDeduplicationPlan

```php
final readonly class ResultDeduplicationPlan
{
    public function __construct(
        public ResultDeduplicationStrategy $strategy,
    ) {}
}
```

---

# 51. Strategies

```text
NONE
ROOT_ENTITY_IDENTITY
FULL_TUPLE
STRUCTURAL
CUSTOM
```

---

# 52. ROOT_ENTITY_IDENTITY

La estrategia estándar para colecciones de entidades con eager JOIN.

---

# 53. FULL_TUPLE

Para resultados tuple:

```text
(User#1, Role#1)
(User#1, Role#2)
```

ambas tuples pueden ser distintas aunque compartan root.

---

# 54. Result identity ≠ entity identity

Debe distinguirse:

```text
EntityIdentity
```

de:

```text
LogicalResultIdentity
```

---

# 55. JOIN amplification

Definimos:

```text
JoinAmplification
=
PhysicalRows
/
UniqueRootResults
```

---

# 56. Example

```text
PhysicalRows = 10,000
UniqueUsers = 100
```

Entonces:

```text
JoinAmplification = 100
```

---

# 57. Importance

Una query aparentemente pequeña:

```text
100 users
```

puede generar:

```text
10,000 rows
```

por eager JOINs.

---

# 58. Cartesian amplification

Con:

```text
User
├── Orders
└── Roles
```

si un usuario tiene:

```text
20 orders
5 roles
```

el JOIN puede producir:

```text
20 × 5 = 100 rows
```

para una sola root.

---

# 59. Relationship assembly

Cada row puede aportar:

```text
new relationship edges
```

aunque la root ya exista.

---

# 60. Therefore

No se puede hacer:

```text
if root already seen:
    skip row
```

porque se perderían relaciones.

---

# 61. Correct behavior

```text
root already exists
↓
reuse root
↓
continue hydrating relationship components
↓
deduplicate edges
```

---

# 62. Relationship assembly state

El ResultHydrator mantendrá estado temporal por scope:

```text
Root
Relationship
Related EntityKey
```

---

# 63. AssemblyKey

```text
RootEntityKey
×
RelationshipId
×
RelatedEntityKey
```

---

# 64. Relationship duplicates

La misma relación no deberá agregarse múltiples veces por multiplicación del JOIN.

---

# 65. Multiple relationships

Ejemplo:

```text
User#1 Order#10 Role#1
User#1 Order#10 Role#2
User#1 Order#11 Role#1
User#1 Order#11 Role#2
```

debe producir:

```text
User#1
├── Orders
│   ├── Order#10
│   └── Order#11
│
└── Roles
    ├── Role#1
    └── Role#2
```

---

# 66. Not

```text
Orders:
10
10
11
11

Roles:
1
2
1
2
```

---

# 67. LEFT JOIN absence

Una relación ausente se determina por:

```text
related identifier absence
```

no por una row completa vacía.

---

# 68. Example

```text
User#1
Order.id = NULL
```

produce:

```text
User#1.orders = []
```

cuando el plan determina colección completamente observada.

---

# 69. Collection completeness

No toda relación hidratada parcialmente deberá marcarse como completa.

---

# 70. Complete eager relation

Consulta:

```text
User
LEFT JOIN all Orders
```

puede producir:

```text
Orders = COMPLETE
```

---

# 71. Filtered relation

```text
User
JOIN Orders
WHERE Order.status = 'PAID'
```

no necesariamente significa:

```text
User.orders = COMPLETE
```

---

# 72. Completeness states

```php
enum RelationshipResultCompleteness
{
    case COMPLETE;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 73. Rule

```text
ObservedRelationshipRows
≠
CompleteRelationship
```

---

# 74. Query semantics determine completeness

El `HydrationPlan` deberá recibir esta información desde fases anteriores.

---

# 75. Scalar Hydration

Un scalar component realiza:

```text
ResultSlot
↓
Database Type
↓
Type Conversion
↓
PHP Scalar / Value
```

---

# 76. Scalar null

```text
NULL
```

se conserva según mapping del resultado.

---

# 77. Aggregate scalar

Ejemplo:

```sql
COUNT(*)
```

puede convertirse a:

```php
int
```

aunque el driver entregue:

```php
"42"
```

---

# 78. Scalar alias

El alias deberá resolverse durante compilación cuando sea posible.

---

# 79. Tuple Hydration

Una tuple puede contener:

```text
Entity
Scalar
Scalar
```

Ejemplo:

```text
[
    User#42,
    15,
    1200.50
]
```

---

# 80. Tuple plan

```php
final readonly class TupleHydrationPlan
{
    public function __construct(
        public array $components,
        public TupleConstructionStrategy $strategy,
    ) {}
}
```

---

# 81. Tuple representation

VoltStack deberá evitar depender exclusivamente de arrays posicionales para API avanzada.

Podrá ofrecer:

```php
$result->get('user');
$result->get('orderCount');
```

---

# 82. Projection Hydration

Projection representa:

```text
selected data
without full managed entity semantics
```

---

# 83. Projection example

```php
final readonly class UserSummary
{
    public function __construct(
        public UserId $id,
        public string $name,
    ) {}
}
```

---

# 84. Projection IdentityMap

Una projection no entra automáticamente al IdentityMap.

---

# 85. Projection snapshot

Tampoco requiere:

```text
EntitySnapshot
```

por defecto.

---

# 86. DTO Hydration

DTO construction deberá utilizar un plan compilado:

```text
Result slots
↓
Type conversion
↓
Constructor/factory
↓
DTO
```

---

# 87. DTO constructor safety

No se seleccionarán constructores dinámicamente a partir de datos de DB.

---

# 88. DTO errors

Incompatibilidades deberán producir errores de hidratación explícitos.

---

# 89. Mixed results

Ejemplo:

```text
User + COUNT(Order)
```

podrá producir:

```php
ResultTuple(
    user: User#42,
    orderCount: 15
)
```

---

# 90. Root semantics in mixed results

La deduplicación dependerá del shape.

Por ejemplo:

```text
User#1 category=A
User#1 category=B
```

pueden ser dos resultados lógicos si la tuple completa es significativa.

---

# 91. Critical rule

> La presencia de una entidad root no implica automáticamente que el resultado deba deduplicarse únicamente por esa entidad.

---

# 92. Result cardinality

VoltStack deberá modelar explícitamente las expectativas de cardinalidad.

```php
enum ResultCardinalityExpectation
{
    case MANY;
    case ZERO_OR_ONE;
    case EXACTLY_ONE;
    case FIRST;
}
```

---

# 93. MANY

```text
0..N
```

resultados lógicos.

---

# 94. ZERO_OR_ONE

```text
0..1
```

resultado lógico.

---

# 95. EXACTLY_ONE

Debe existir:

```text
1
```

resultado lógico.

---

# 96. FIRST

Devuelve el primer resultado lógico según semántica de orden de la query.

---

# 97. Physical cardinality trap

Para:

```text
EXACTLY_ONE User
```

un eager JOIN puede producir:

```text
5 rows
```

sin violar cardinalidad.

---

# 98. Therefore

Incorrecto:

```text
if rowCount > 1:
    throw NonUniqueResultException
```

---

# 99. Correct

Debe evaluarse:

```text
LogicalRootCount
```

---

# 100. Formula

```text
CardinalityValidation
=
ExpectedLogicalCardinality
vs
HydratedLogicalCardinality
```

---

# 101. ZERO_OR_ONE

Resultado:

```text
0 logical roots
→ null

1 logical root
→ result

>1 logical roots
→ NonUniqueResultException
```

---

# 102. EXACTLY_ONE

```text
0
→ NoResultException

1
→ result

>1
→ NonUniqueResultException
```

---

# 103. FIRST

`FIRST` no significa:

```text
first physical row
```

cuando un logical result necesita múltiples rows para completarse.

---

# 104. Correct FIRST

```text
first complete logical result
```

según el plan.

---

# 105. one() optimization

El Query Planner podrá limitar la consulta para detectar:

```text
up to 2 logical roots
```

pero el ResultHydrator no deberá asumir que:

```sql
LIMIT 2
```

sobre rows físicas siempre equivale a dos roots.

---

# 106. JOIN pagination problem

```sql
LIMIT 2
```

sobre:

```text
User#1 Order#10
User#1 Order#11
User#2 Order#20
```

podría devolver únicamente `User#1`.

---

# 107. Consequence

Pagination/cardinality con collection JOIN requiere estrategias específicas.

---

# 108. ResultHydrator does not solve query planning

Debe interpretar correctamente el plan recibido.

---

# 109. Empty Result

Para collection:

```text
[]
```

---

# 110. Empty scalar aggregate

Dependerá de semántica SQL y del plan.

---

# 111. Empty ZERO_OR_ONE

```text
null
```

---

# 112. Empty EXACTLY_ONE

```text
NoResultException
```

---

# 113. Buffered Hydration

Modo:

```text
BUFFERED
```

consume el resultado completo antes de devolver.

---

# 114. Advantages

Permite:

- arbitrary root ordering;
- full deduplication;
- complex graph assembly;
- complete cardinality knowledge;
- easier error recovery.

---

# 115. Cost

```text
Memory
≈
LogicalResults
+
IdentityMap
+
AssemblyState
+
ResultBuffers
```

---

# 116. Streaming Hydration

Modo:

```text
STREAMING
```

produce resultados progresivamente.

---

# 117. Streaming requirement

Debe existir una estrategia para determinar cuándo un resultado lógico está completo.

---

# 118. Simple scalar stream

```text
row
→ scalar
→ yield
```

es trivial.

---

# 119. Simple entity stream

Sin collection JOIN:

```text
row
→ entity
→ yield
```

también puede ser directo.

---

# 120. Graph stream

Con collection JOIN:

```text
multiple rows
→ one root
```

requiere grouping.

---

# 121. Ordered graph streaming

Si rows están ordenadas por root:

```text
User#1
User#1
User#1
User#2
User#2
```

al detectar:

```text
User#2
```

se sabe que:

```text
User#1
```

está completo.

---

# 122. Streaming state

```text
CurrentRoot
CurrentRootKey
RelationshipAssemblyState
```

---

# 123. Root transition

```text
new root key
↓
finalize previous root
↓
yield previous root
↓
begin new root
```

---

# 124. End-of-result

El último root se finaliza al recibir EOF.

---

# 125. Unordered graph streaming

Resultado:

```text
User#1
User#2
User#1
```

impide saber en row 2 que `User#1` está completo.

---

# 126. Therefore

No se deberá utilizar ordered streaming sin precondición.

---

# 127. Strategies

Ante unordered result:

```text
BUFFER
```

o:

```text
REJECT STREAMING PLAN
```

---

# 128. No silent incomplete yield

Prohibido devolver:

```text
User#1
```

y posteriormente descubrir otra relación que debió formar parte del mismo resultado.

---

# 129. Cursor integration

`ResultHydrator` deberá consumir:

```text
ResultCursor
```

sin apropiarse indebidamente de la conexión.

---

# 130. Cursor ownership

El request/plan deberá indicar:

```text
who owns cursor lifecycle
```

---

# 131. Default

Para una operación normal:

```text
ResultHydrator
```

deberá garantizar cierre del cursor al:

- completar;
- fallar;
- cancelar;
- abandonar el stream según contrato.

---

# 132. Generator issue

Ejemplo:

```php
foreach ($query->stream() as $user) {
    break;
}
```

El cursor deberá liberarse determinísticamente cuando el stream sea cerrado/destruido según mecanismos soportados.

---

# 133. Resource lease

Puede modelarse:

```php
final class HydrationResourceLease
{
    public function release(): void;
}
```

---

# 134. Idempotent release

```text
release()
release()
```

no deberá causar error.

---

# 135. Cancellation

El sistema deberá cooperar con:

```text
QueryCancellation
ResultCursorCancellation
Runtime cancellation
```

---

# 136. Hydration cancellation

Si se cancela después de 500 rows:

```text
temporary assembly state
```

deberá liberarse.

---

# 137. Managed entities already hydrated

Las entidades ya activadas antes de cancelación no deberán quedar en un estado ORM corrupto.

---

# 138. Incomplete root

Si el root actual todavía no estaba finalizado:

```text
do not yield incomplete logical result
```

---

# 139. Partial failure

Supongamos:

```text
Root #1 → hydrated
Root #2 → hydrated
Root #3 → conversion failure
```

---

# 140. Buffered mode

La operación completa puede fallar sin devolver la colección.

---

# 141. Streaming mode

Root #1 y #2 pueden haber sido entregados al consumidor.

Esto deberá formar parte explícita del contrato de streaming.

---

# 142. Streaming atomicity

```text
StreamingHydration
≠
AtomicResultMaterialization
```

---

# 143. Database transaction

Incluso dentro de una transaction:

```text
stream yielded entity
```

no significa que el resultado completo será hidratado correctamente.

---

# 144. HydrationScope

Todo proceso deberá tener un scope.

```php
final class HydrationScope
{
    // runtime-local mutable state
}
```

---

# 145. Scope contains

Conceptualmente:

```text
HydrationScopeId
RootRegistry
TupleRegistry
RelationshipAssemblyState
TemporaryDeduplicationState
CurrentStreamingRoot
Statistics
CancellationState
ResourceLease
```

---

# 146. Scope ≠ PersistenceContext

`HydrationScope` vive normalmente durante:

```text
one result hydration operation
```

Mientras:

```text
PersistenceContext
```

puede abarcar múltiples queries.

---

# 147. RootRegistry ≠ IdentityMap

`RootRegistry` responde:

```text
¿ya agregué este root al resultado actual?
```

IdentityMap responde:

```text
¿ya existe representación canónica de esta entidad
en el PersistenceContext?
```

---

# 148. Important distinction

Una entidad puede estar en IdentityMap antes de iniciar la query.

Pero todavía no estar:

```text
RootRegistry
```

del resultado actual.

---

# 149. Example

```text
IdentityMap:
User#1

New query:
User#1
User#2
```

Logical result:

```text
[existing User#1, new User#2]
```

---

# 150. Result root deduplication state

Debe ser local al resultado.

No deberá reutilizarse entre queries.

---

# 151. TupleRegistry

Solo necesario cuando el shape exige deduplicación de tuples.

---

# 152. Structural deduplication

Debe evitar comparar objetos completos recursivamente.

Preferir:

```text
typed structural keys
```

---

# 153. ResultLogicalKey

Concepto:

```php
interface ResultLogicalKey
{
}
```

Implementaciones:

```text
EntityRootResultKey
TupleResultKey
ProjectionResultKey
CustomResultKey
```

---

# 154. Hashing

Podrá utilizarse hashing para lookup rápido.

Pero:

```text
Hash
≠
LogicalKey
```

---

# 155. Hash collision

Debe verificarse igualdad estructural real cuando exista posibilidad de colisión.

---

# 156. Pagination integration

ResultHydrator consume resultados ya planificados para pagination.

No implementa:

```text
OFFSET
CURSOR predicate
LIMIT planning
```

---

# 157. Pagination + eager relationships

Puede requerir:

```text
query root IDs
↓
query roots + relationships
↓
hydrate
```

en lugar de un JOIN paginado directo.

---

# 158. Responsibility boundary

Esto pertenece a:

```text
Query Planner
Entity Query
Pagination System
```

El Hydrator únicamente respeta la semántica resultante.

---

# 159. Duplicate physical rows

Un driver o query puede retornar rows físicamente idénticas.

El tratamiento depende del plan.

---

# 160. No universal deduplication

No deberá existir:

```text
always remove duplicate rows
```

porque en algunos resultados los duplicados son semánticamente válidos.

---

# 161. Example scalar

```text
["A", "A", "B"]
```

puede ser el resultado correcto.

---

# 162. Deduplication policy

Siempre explícita.

---

# 163. Scalar type system integration

Toda conversión deberá delegarse al:

```text
Database Type System
```

o al plan de conversión compilado.

---

# 164. No PHP loose conversion

Evitar:

```php
(int) $value;
(bool) $value;
```

como semántica universal.

---

# 165. Numeric precision

Tipos:

```text
DECIMAL
NUMERIC
BIGINT
```

podrán requerir:

- string;
- decimal object;
- big integer object.

No convertir automáticamente a `float`.

---

# 166. Date/time

El plan deberá conocer:

```text
timezone policy
precision
mutability
database representation
```

---

# 167. JSON

Un JSON result puede producir:

```text
array
object
value object
JSON document abstraction
```

según type mapping.

---

# 168. Projection type safety

Los valores de una projection también deberán pasar por Type System.

---

# 169. DTO type safety

Igualmente.

---

# 170. Query aliases

Los aliases físicos podrán diferir de los nombres lógicos.

Ejemplo:

```text
u__id
u__name
o__id
```

No deben filtrarse necesariamente a la API pública.

---

# 171. Internal result aliases

Son parte del contrato:

```text
Compiler
↔
HydrationPlan
```

---

# 172. Compiler coordination

El SQL Compiler podrá producir:

```text
CompiledQuery
+
ResultShapeMetadata
```

que permita construir o validar el plan de hidratación.

---

# 173. But compiler does not hydrate

```text
Compiler
≠
ResultHydrator
```

---

# 174. Query Planner coordination

Planner puede decidir:

```text
JOIN eager loading
batch eager loading
root ordering
pagination strategy
```

que afectan el plan.

---

# 175. HydrationPlan Compiler

Debe existir una fase que convierta metadata semántica en instrucciones eficientes.

```text
Semantic Result Shape
+
Entity Metadata
+
Compiled Query Result Layout
      │
      ▼
Hydration Plan Compiler
      │
      ▼
CompiledResultHydrationPlan
```

---

# 176. ResultHydrator receives compiled plan

No debería resolver toda la metadata nuevamente por row.

---

# 177. Plan cache

Los planes podrán cachearse cuando dependencias sean estables.

---

# 178. Plan cache key

Puede incluir:

```text
QueryShapeId
EntityMetadataGeneration
TypeRegistryGeneration
PlatformResultSemantics
HydrationMode
```

---

# 179. Plan cache ≠ result cache

No contiene:

- rows;
- entities;
- scalar values;
- DTO instances.

---

# 180. Immutable plan

Requisito fundamental para persistent runtimes.

---

# 181. Error hierarchy

```text
DatabaseResultHydrationException
├── ResultHydrationPlanException
├── ResultShapeMismatchException
├── ResultColumnMismatchException
├── ResultSlotException
├── ResultTypeConversionException
├── ResultGroupingException
├── ResultOrderingRequirementException
├── ResultDeduplicationException
├── ResultRootResolutionException
├── ResultRelationshipAssemblyException
├── ResultTupleHydrationException
├── ResultProjectionHydrationException
├── ResultDtoHydrationException
├── ResultCardinalityException
│   ├── NoResultException
│   └── NonUniqueResultException
├── ResultStreamingException
├── ResultStreamingPreconditionException
├── ResultIncompleteRootException
├── ResultCancellationException
├── ResultResourceException
├── ResultHydrationRuntimeIsolationException
└── ResultHydrationInvariantException
```

---

# 182. Error context

Puede contener:

```text
HydrationPlanId
ResultShape
ComponentId
ResultSlot
LogicalRootNumber
PhysicalRowNumber
HydrationScopeId
```

---

# 183. No raw row dumps

Los errores no deberán incluir automáticamente:

```text
password
token
personal information
full JSON documents
```

---

# 184. Resource failure

Si falla:

```text
cursor close
```

después de una hidratación exitosa, el sistema deberá preservar ambas dimensiones:

```text
HydrationOutcome = SUCCEEDED
ResourceReleaseOutcome = FAILED
```

cuando sea operacionalmente relevante.

---

# 185. Failure dimensions

No reducir todo a:

```text
FAILED
```

si se necesita diagnóstico preciso.

---

# 186. Telemetry

Métricas propuestas:

```text
orm.result_hydration.operations
orm.result_hydration.duration

orm.result_hydration.physical_rows
orm.result_hydration.logical_results

orm.result_hydration.root_occurrences
orm.result_hydration.unique_roots

orm.result_hydration.join_amplification

orm.result_hydration.entity_components
orm.result_hydration.scalar_components
orm.result_hydration.tuple_components
orm.result_hydration.dto_components
orm.result_hydration.projection_components

orm.result_hydration.relationship_edges
orm.result_hydration.relationship_duplicates

orm.result_hydration.buffered
orm.result_hydration.streaming

orm.result_hydration.cardinality_failures
orm.result_hydration.cancellations
orm.result_hydration.failures
```

---

# 187. Logical compression ratio

```text
LogicalCompressionRatio
=
PhysicalRows
/
LogicalResults
```

---

# 188. Relationship duplication ratio

```text
RelationshipDuplicationRatio
=
ObservedRelationshipOccurrences
/
UniqueRelationshipEdges
```

---

# 189. Memory telemetry

Podrá observar:

```text
peak roots buffered
peak assembly keys
peak tuples buffered
estimated hydration memory
```

---

# 190. Avoid high cardinality

No utilizar:

```text
entity IDs
tenant IDs
query parameter values
```

como labels métricas por defecto.

---

# 191. Debug toolbar

Ejemplo:

```text
Result Hydration
────────────────────────────────────

Shape:               ENTITY_COLLECTION
Mode:                BUFFERED

Physical rows:       12,430
Logical roots:          250

JOIN amplification:    49.72x

Entities constructed: 1,820
IdentityMap reused:  10,610

Relationship edges:
  observed:          21,400
  unique:             3,820

Hydration plan:       HIT
Duration:             18.4 ms
Peak hydration state: 6.2 MB
```

---

# 192. High amplification warning

El profiler podrá advertir:

```text
High JOIN amplification detected
```

cuando exceda thresholds configurables.

---

# 193. Suggested remediation

Diagnóstico podrá sugerir:

```text
batch eager loading
split query
projection
pagination redesign
```

sin modificar automáticamente la consulta.

---

# 194. Persistent runtime

En FrankenPHP:

```text
Worker
│
├── Shared
│   ├── CompiledResultHydrationPlans
│   ├── immutable component plans
│   └── immutable converters
│
├── Request A
│   └── HydrationScope A
│
└── Request B
    └── HydrationScope B
```

---

# 195. Shared immutable

Permitido:

```text
CompiledResultHydrationPlan
CompiledTuplePlan
CompiledProjectionPlan
CompiledDtoPlan
CompiledResultSlots
```

---

# 196. Scoped mutable

Obligatoriamente scoped:

```text
RootRegistry
TupleRegistry
CurrentRoot
AssemblyState
Cursor
ResourceLease
CancellationState
ResultAccumulator
```

---

# 197. Forbidden process state

Prohibido:

```php
ResultHydrator::$currentRoot;
ResultHydrator::$currentResult;
ResultHydrator::$seenEntities;
```

---

# 198. Request cleanup

Al finalizar:

```text
HydrationScope
```

deberá liberar:

```text
buffers
dedup registries
assembly registries
cursor lease
temporary tuples
```

---

# 199. PersistenceContext survives only according to scope

El ResultHydrator no decide si EntityManager continúa vivo.

---

# 200. RoadRunner

Misma regla:

```text
Job/Request A hydration state
≠
Job/Request B hydration state
```

---

# 201. OpenSwoole

Cada coroutine lógica deberá tener contexto correctamente aislado.

---

# 202. Concurrency

Un `HydrationScope` mutable no será concurrency-safe por defecto.

---

# 203. Parallel row processing

No se deberá paralelizar ingenuamente:

```text
row1
row2
row3
```

porque:

- comparten root;
- comparten IdentityMap;
- comparten relationship assembly;
- ordering importa.

---

# 204. Future parallelism

Solo mediante planes explícitamente:

```text
partitionable
```

podrá considerarse.

---

# 205. Determinism

Mismo:

```text
Result
+
HydrationPlan
+
PersistenceContext initial state
```

deberá producir la misma estructura lógica observable, salvo factores explícitamente externos.

---

# 206. Result ordering

El ResultHydrator preservará el orden lógico definido por el query/plan.

---

# 207. Deduplication order

Cuando se deduplican roots:

```text
first logical occurrence
```

determina normalmente la posición del root.

---

# 208. Example

```text
User#3
User#1
User#3
User#2
```

produce:

```text
[
    User#3,
    User#1,
    User#2
]
```

bajo first-occurrence ordering.

---

# 209. No implicit sorting

Hydration no reordenará entidades por ID salvo instrucción explícita.

---

# 210. Result Collection

La salida MANY podrá utilizar una abstracción:

```php
interface EntityCollection extends iterable, Countable
{
}
```

o una colección general de VoltStack.

---

# 211. Collection ≠ Relationship Collection

Una colección de resultados:

```text
QueryResultCollection
```

no es necesariamente la misma abstracción que:

```text
PersistentRelationshipCollection
```

---

# 212. Reason

Relationship Collection posee semántica ORM adicional:

- initialized;
- partial;
- dirty;
- owner;
- mapping;
- lazy loading.

---

# 213. Result materialization policy

La colección de resultados podrá ser:

```text
ARRAY
COLLECTION
ITERATOR
STREAM
```

según API.

---

# 214. API examples

```php
$users = User::query()->get();
```

```php
$user = User::query()
    ->where('id', 42)
    ->one();
```

```php
$user = User::query()
    ->where('email', $email)
    ->oneOrNull();
```

```php
foreach (User::query()->stream() as $user) {
    // ...
}
```

---

# 215. Scalar examples

```php
$count = User::query()->count();
```

```php
$emails = User::query()
    ->select('email')
    ->scalars();
```

---

# 216. Projection example

```php
$summaries = User::query()
    ->select('id', 'name')
    ->as(UserSummary::class)
    ->get();
```

---

# 217. Tuple example

```php
$results = User::query()
    ->withAggregate('orders', 'count')
    ->tuples()
    ->get();
```

---

# 218. Hydration mode

```php
enum ResultHydrationMode
{
    case BUFFERED;
    case STREAMING;
}
```

Modos especializados pueden agregarse posteriormente.

---

# 219. Buffer policy

```php
final readonly class ResultBufferPolicy
{
    public function __construct(
        public ?int $maximumLogicalResults,
        public ?int $maximumPhysicalRows,
        public ?int $maximumEstimatedBytes,
    ) {}
}
```

---

# 220. Resource governance

Un resultado inesperadamente grande podrá provocar:

```text
ResultHydrationResourceLimitException
```

en lugar de agotar memoria.

---

# 221. Resource limits ≠ pagination

No se utilizarán límites de seguridad como sustituto de pagination.

---

# 222. Backpressure

Streaming deberá permitir que:

```text
consumer speed
```

controle el ritmo de lectura cuando el driver/cursor lo permita.

---

# 223. No prefetch explosion

El sistema no deberá convertir streaming en:

```text
load everything first
```

salvo que el plan lo requiera explícitamente.

---

# 224. Cancellation propagation

```text
consumer cancellation
↓
hydration stream
↓
ResultCursor
↓
Execution Engine
```

cuando sea soportado.

---

# 225. Result ownership state

Podrá modelarse:

```text
OPEN
CONSUMING
EXHAUSTED
CLOSED
FAILED
CANCELLED
```

---

# 226. Hydration state ≠ Result state

El estado del proceso de hidratación y el estado del cursor deberán permanecer conceptualmente separados.

---

# 227. Extension model

VoltStack podrá soportar:

```text
CustomResultShape
CustomResultComponentHydrator
CustomProjectionHydrator
CustomDtoHydrator
CustomDeduplicationStrategy
CustomGroupingStrategy
```

---

# 228. Extension Registry

```php
interface ResultHydrationExtensionRegistry
{
    public function register(
        ResultHydrationExtension $extension
    ): void;
}
```

---

# 229. Registry freezing

En producción:

```text
bootstrap
↓
register extensions
↓
validate
↓
freeze
```

---

# 230. Extension constraints

Una extensión no podrá:

- romper IdentityMap canonicality;
- acceder a PDO arbitrariamente;
- ejecutar queries ocultas;
- cambiar transaction state;
- compartir mutable request state globalmente.

---

# 231. Security model

Result hydration deberá considerarse una frontera de interpretación de datos.

---

# 232. Untrusted database state

Aunque la DB sea interna:

```text
database value
```

no deberá asumirse automáticamente compatible con el modelo PHP.

---

# 233. Type validation

Los converters deberán rechazar datos incompatibles.

---

# 234. DTO target validation

Solo DTOs autorizados por el plan compilado podrán instanciarse.

---

# 235. Projection target validation

Igualmente.

---

# 236. No dynamic execution

Prohibido utilizar datos de DB como:

```text
class name
method name
service id
PHP expression
```

sin resolución mediante metadata prevalidada.

---

# 237. Testing architecture

La suite deberá cubrir al menos:

```text
result shape
cardinality
deduplication
JOIN amplification
relationship assembly
scalars
tuples
projections
DTOs
buffering
streaming
cancellation
resource release
runtime isolation
```

---

# 238. Simple entity collection test

Rows:

```text
User#1
User#2
User#3
```

Resultado:

```text
[User#1, User#2, User#3]
```

---

# 239. JOIN root dedup test

Rows:

```text
User#1 Order#10
User#1 Order#11
```

Resultado roots:

```text
[User#1]
```

---

# 240. Relationship assembly test

`User#1.orders` contiene exactamente:

```text
Order#10
Order#11
```

---

# 241. Multi-JOIN amplification test

Rows:

```text
User#1 Order#10 Role#1
User#1 Order#10 Role#2
User#1 Order#11 Role#1
User#1 Order#11 Role#2
```

produce dos orders y dos roles.

---

# 242. IdentityMap preloaded test

Si `User#1` ya existe:

```text
result root
```

utiliza la misma instancia.

---

# 243. Scalar duplicate test

```text
A
A
B
```

permanece:

```text
[A, A, B]
```

cuando no existe dedup policy.

---

# 244. ZERO_OR_ONE JOIN test

Cinco rows para una misma root siguen representando:

```text
one logical result
```

---

# 245. Non-unique test

```text
User#1
User#2
```

bajo `ZERO_OR_ONE` produce:

```text
NonUniqueResultException
```

---

# 246. EXACTLY_ONE empty test

Produce:

```text
NoResultException
```

---

# 247. FIRST graph test

No devuelve root antes de completar sus eager relationships.

---

# 248. Streaming ordered test

Roots se producen progresivamente sin buffering global.

---

# 249. Streaming unordered rejection test

Plan que requiere ordering falla si la precondición no está garantizada.

---

# 250. Cursor early break test

```php
foreach ($stream as $item) {
    break;
}
```

libera recursos.

---

# 251. Cancellation test

Cancela cursor y limpia temporary state.

---

# 252. Conversion failure test

Una row inválida produce error tipado y resource cleanup.

---

# 253. Projection test

Projection no entra en IdentityMap.

---

# 254. DTO test

DTO no entra en UnitOfWork.

---

# 255. Mixed result test

Entity component reutiliza IdentityMap mientras scalar component conserva su semántica.

---

# 256. Persistent worker test

HydrationScope A no aparece en request B.

---

# 257. Memory limit test

Un buffer que excede policy falla de forma controlada.

---

# 258. Performance tests

Benchmarks:

```text
10k scalar rows
100k scalar rows

10k simple entities
100k simple entities

1:N JOIN
1:N + M:N JOIN

high IdentityMap hit ratio
high IdentityMap miss ratio

buffered vs streaming

DTO hydration
projection hydration
tuple hydration
```

---

# 259. Architectural Invariants

## DB-ORM-RESULT-HYDRATION-001

ResultHydrator no ejecutará SQL.

## DB-ORM-RESULT-HYDRATION-002

ResultHydrator no generará SQL.

## DB-ORM-RESULT-HYDRATION-003

ResultHydrator no administrará conexiones.

## DB-ORM-RESULT-HYDRATION-004

ResultHydrator consumirá Result mediante contratos del Execution Engine.

## DB-ORM-RESULT-HYDRATION-005

Physical row no equivaldrá universalmente a logical result.

## DB-ORM-RESULT-HYDRATION-006

Physical row count no definirá universalmente logical cardinality.

## DB-ORM-RESULT-HYDRATION-007

Result shape será explícito.

## DB-ORM-RESULT-HYDRATION-008

Hydration plan será explícito.

## DB-ORM-RESULT-HYDRATION-009

Hydration plan podrá ser compilado.

## DB-ORM-RESULT-HYDRATION-010

Compiled plan será inmutable.

## DB-ORM-RESULT-HYDRATION-011

Compiled plan no almacenará entities.

## DB-ORM-RESULT-HYDRATION-012

Compiled plan no almacenará EntityManager scoped.

## DB-ORM-RESULT-HYDRATION-013

ResultHydrator coordinará EntityHydrator.

## DB-ORM-RESULT-HYDRATION-014

ResultHydrator no duplicará EntityHydrator.

## DB-ORM-RESULT-HYDRATION-015

Entity result respetará IdentityMap.

## DB-ORM-RESULT-HYDRATION-016

Root deduplication será distinta de IdentityMap.

## DB-ORM-RESULT-HYDRATION-017

RootRegistry será result-scoped.

## DB-ORM-RESULT-HYDRATION-018

IdentityMap será persistence-context-scoped.

## DB-ORM-RESULT-HYDRATION-019

SQL DISTINCT será distinto de ORM deduplication.

## DB-ORM-RESULT-HYDRATION-020

Deduplication policy será explícita.

## DB-ORM-RESULT-HYDRATION-021

Duplicados escalares no se eliminarán universalmente.

## DB-ORM-RESULT-HYDRATION-022

JOIN amplification no generará duplicate root objects.

## DB-ORM-RESULT-HYDRATION-023

JOIN amplification no generará duplicate relationship edges.

## DB-ORM-RESULT-HYDRATION-024

Root ya visto no provocará skip de relationship processing.

## DB-ORM-RESULT-HYDRATION-025

Relationship absence utilizará identifier semantics.

## DB-ORM-RESULT-HYDRATION-026

LEFT JOIN null related ID no creará entidad vacía.

## DB-ORM-RESULT-HYDRATION-027

Relationship completeness será explícita.

## DB-ORM-RESULT-HYDRATION-028

Filtered JOIN no implicará complete collection.

## DB-ORM-RESULT-HYDRATION-029

Grouping strategy será explícita.

## DB-ORM-RESULT-HYDRATION-030

Ordered grouping declarará ordering precondition.

## DB-ORM-RESULT-HYDRATION-031

ResultHydrator no añadirá ORDER BY ocultamente.

## DB-ORM-RESULT-HYDRATION-032

Unordered graph no se streamed como ordered graph.

## DB-ORM-RESULT-HYDRATION-033

Incomplete root no será yielded.

## DB-ORM-RESULT-HYDRATION-034

End-of-result finalizará root pendiente.

## DB-ORM-RESULT-HYDRATION-035

Scalar hydration utilizará type conversion explícita.

## DB-ORM-RESULT-HYDRATION-036

DECIMAL no se convertirá universalmente a float.

## DB-ORM-RESULT-HYDRATION-037

Tuple shape preservará todos sus componentes.

## DB-ORM-RESULT-HYDRATION-038

Tuple identity será distinta de entity identity.

## DB-ORM-RESULT-HYDRATION-039

Projection no será managed entity por defecto.

## DB-ORM-RESULT-HYDRATION-040

Projection no entrará automáticamente al IdentityMap.

## DB-ORM-RESULT-HYDRATION-041

Projection no requerirá EntitySnapshot por defecto.

## DB-ORM-RESULT-HYDRATION-042

DTO no será entity por defecto.

## DB-ORM-RESULT-HYDRATION-043

DTO no entrará automáticamente al UnitOfWork.

## DB-ORM-RESULT-HYDRATION-044

DTO construction utilizará plan validado.

## DB-ORM-RESULT-HYDRATION-045

DB data no seleccionará clases arbitrarias.

## DB-ORM-RESULT-HYDRATION-046

Mixed result tendrá semántica explícita.

## DB-ORM-RESULT-HYDRATION-047

Root entity no determinará siempre logical result identity.

## DB-ORM-RESULT-HYDRATION-048

MANY aceptará cero resultados.

## DB-ORM-RESULT-HYDRATION-049

ZERO_OR_ONE validará logical results.

## DB-ORM-RESULT-HYDRATION-050

EXACTLY_ONE validará logical results.

## DB-ORM-RESULT-HYDRATION-051

FIRST devolverá first logical result.

## DB-ORM-RESULT-HYDRATION-052

FIRST no significará first physical row universalmente.

## DB-ORM-RESULT-HYDRATION-053

Multiple physical rows podrán representar EXACTLY_ONE.

## DB-ORM-RESULT-HYDRATION-054

NonUniqueResult se decidirá después de logical grouping cuando sea necesario.

## DB-ORM-RESULT-HYDRATION-055

NoResult se decidirá según logical cardinality.

## DB-ORM-RESULT-HYDRATION-056

ResultHydrator no asumirá que SQL LIMIT coincide con logical root limit.

## DB-ORM-RESULT-HYDRATION-057

JOIN pagination planning no pertenecerá al ResultHydrator.

## DB-ORM-RESULT-HYDRATION-058

Buffered mode podrá consumir resultado completo.

## DB-ORM-RESULT-HYDRATION-059

Streaming mode no deberá bufferizar todo silenciosamente.

## DB-ORM-RESULT-HYDRATION-060

Streaming graph requerirá completion strategy.

## DB-ORM-RESULT-HYDRATION-061

Ordered streaming requerirá stable root ordering.

## DB-ORM-RESULT-HYDRATION-062

Unordered streaming deberá bufferizar o rechazarse.

## DB-ORM-RESULT-HYDRATION-063

Streaming hydration no prometerá atomic result materialization.

## DB-ORM-RESULT-HYDRATION-064

Previously yielded results podrán existir antes de later failure.

## DB-ORM-RESULT-HYDRATION-065

Streaming failure semantics serán documentadas.

## DB-ORM-RESULT-HYDRATION-066

Result cursor será liberado determinísticamente.

## DB-ORM-RESULT-HYDRATION-067

Resource release será idempotente cuando corresponda.

## DB-ORM-RESULT-HYDRATION-068

Early stream termination liberará cursor.

## DB-ORM-RESULT-HYDRATION-069

Cancellation liberará temporary hydration state.

## DB-ORM-RESULT-HYDRATION-070

Cancellation no yield incomplete root.

## DB-ORM-RESULT-HYDRATION-071

HydrationScope será operation-scoped.

## DB-ORM-RESULT-HYDRATION-072

HydrationScope será distinto de PersistenceContext.

## DB-ORM-RESULT-HYDRATION-073

RootRegistry no será IdentityMap.

## DB-ORM-RESULT-HYDRATION-074

TupleRegistry no será IdentityMap.

## DB-ORM-RESULT-HYDRATION-075

Result-local deduplication no sobrevivirá entre queries.

## DB-ORM-RESULT-HYDRATION-076

Hash será distinto de logical result key.

## DB-ORM-RESULT-HYDRATION-077

Hash collision no implicará logical equality.

## DB-ORM-RESULT-HYDRATION-078

Result ordering será preservado según plan.

## DB-ORM-RESULT-HYDRATION-079

Deduplication no reordenará arbitrariamente roots.

## DB-ORM-RESULT-HYDRATION-080

No habrá implicit sorting por entity ID.

## DB-ORM-RESULT-HYDRATION-081

QueryResultCollection será distinta de PersistentRelationshipCollection.

## DB-ORM-RESULT-HYDRATION-082

Result collection no tendrá automáticamente relationship semantics.

## DB-ORM-RESULT-HYDRATION-083

Result materialization policy será explícita.

## DB-ORM-RESULT-HYDRATION-084

Buffer limits serán gobernables.

## DB-ORM-RESULT-HYDRATION-085

Resource limits no sustituirán pagination.

## DB-ORM-RESULT-HYDRATION-086

Streaming permitirá backpressure cuando infraestructura lo soporte.

## DB-ORM-RESULT-HYDRATION-087

Streaming no realizará hidden prefetch ilimitado.

## DB-ORM-RESULT-HYDRATION-088

Cancellation podrá propagarse al cursor.

## DB-ORM-RESULT-HYDRATION-089

Hydration state será distinto de cursor state.

## DB-ORM-RESULT-HYDRATION-090

Result component extensions respetarán canonicalidad.

## DB-ORM-RESULT-HYDRATION-091

Extensions no ejecutarán queries ocultas.

## DB-ORM-RESULT-HYDRATION-092

Extensions no modificarán transaction state.

## DB-ORM-RESULT-HYDRATION-093

Extensions no almacenarán mutable request state global.

## DB-ORM-RESULT-HYDRATION-094

Database values serán type validated.

## DB-ORM-RESULT-HYDRATION-095

Invalid values no tendrán coerción silenciosa universal.

## DB-ORM-RESULT-HYDRATION-096

Aliases físicos podrán permanecer internos.

## DB-ORM-RESULT-HYDRATION-097

Compiler podrá proporcionar result layout metadata.

## DB-ORM-RESULT-HYDRATION-098

Compiler no hidratará resultados.

## DB-ORM-RESULT-HYDRATION-099

Planner podrá influir en hydration strategy.

## DB-ORM-RESULT-HYDRATION-100

Hydrator no reemplazará Planner.

## DB-ORM-RESULT-HYDRATION-101

HydrationPlan Compiler podrá precompilar result slots.

## DB-ORM-RESULT-HYDRATION-102

Metadata no se resolverá innecesariamente por row.

## DB-ORM-RESULT-HYDRATION-103

Plan cache será distinto de result cache.

## DB-ORM-RESULT-HYDRATION-104

Plan cache no almacenará rows.

## DB-ORM-RESULT-HYDRATION-105

Plan cache no almacenará entities.

## DB-ORM-RESULT-HYDRATION-106

JOIN amplification será observable.

## DB-ORM-RESULT-HYDRATION-107

Logical result count será observable.

## DB-ORM-RESULT-HYDRATION-108

Physical row count será observable.

## DB-ORM-RESULT-HYDRATION-109

Telemetry distinguirá physical y logical cardinality.

## DB-ORM-RESULT-HYDRATION-110

Telemetry no expondrá IDs sensibles por defecto.

## DB-ORM-RESULT-HYDRATION-111

Debugging podrá reportar amplification.

## DB-ORM-RESULT-HYDRATION-112

Debugging no modificará automáticamente query strategy.

## DB-ORM-RESULT-HYDRATION-113

Shared runtime state será inmutable.

## DB-ORM-RESULT-HYDRATION-114

RootRegistry será scoped.

## DB-ORM-RESULT-HYDRATION-115

CurrentRoot será scoped.

## DB-ORM-RESULT-HYDRATION-116

Cursor será scoped.

## DB-ORM-RESULT-HYDRATION-117

ResultAccumulator será scoped.

## DB-ORM-RESULT-HYDRATION-118

Request A no reutilizará hydration state de Request B.

## DB-ORM-RESULT-HYDRATION-119

FrankenPHP worker reuse no implicará hydration state reuse.

## DB-ORM-RESULT-HYDRATION-120

RoadRunner worker reuse no implicará hydration state reuse.

## DB-ORM-RESULT-HYDRATION-121

OpenSwoole coroutine state estará aislado.

## DB-ORM-RESULT-HYDRATION-122

HydrationScope no será concurrency-safe por defecto.

## DB-ORM-RESULT-HYDRATION-123

Parallel row hydration no será asumida segura.

## DB-ORM-RESULT-HYDRATION-124

Deterministic result semantics serán preservadas.

## DB-ORM-RESULT-HYDRATION-125

Physical duplicates no serán eliminados universalmente.

## DB-ORM-RESULT-HYDRATION-126

Logical duplicates dependerán del plan.

## DB-ORM-RESULT-HYDRATION-127

Entity canonicality será preservada en mixed results.

## DB-ORM-RESULT-HYDRATION-128

Entity component podrá reutilizar objeto existente.

## DB-ORM-RESULT-HYDRATION-129

Scalar component no alterará EntityState.

## DB-ORM-RESULT-HYDRATION-130

DTO component no alterará EntityState.

## DB-ORM-RESULT-HYDRATION-131

Projection component no alterará EntityState por defecto.

## DB-ORM-RESULT-HYDRATION-132

Relationship assembly inicial no creará dirty state artificial.

## DB-ORM-RESULT-HYDRATION-133

Relationship completeness procederá del plan semántico.

## DB-ORM-RESULT-HYDRATION-134

ResultHydrator no inferirá completeness solo por observar EOF.

## DB-ORM-RESULT-HYDRATION-135

EOF finalizará únicamente estructuras cuya semántica permita finalización.

## DB-ORM-RESULT-HYDRATION-136

ResultHydrator no realizará flush.

## DB-ORM-RESULT-HYDRATION-137

ResultHydrator no realizará commit.

## DB-ORM-RESULT-HYDRATION-138

ResultHydrator no realizará rollback.

## DB-ORM-RESULT-HYDRATION-139

ResultHydrator no implicará transaction success.

## DB-ORM-RESULT-HYDRATION-140

Hydration success no implicará DB freshness futura.

## DB-ORM-RESULT-HYDRATION-141

Hydration success no implicará authorization.

## DB-ORM-RESULT-HYDRATION-142

Hydration success no implicará domain validation.

## DB-ORM-RESULT-HYDRATION-143

Hydration failure no cerrará EntityManager arbitrariamente.

## DB-ORM-RESULT-HYDRATION-144

Hydration failure no cerrará Connection arbitrariamente.

## DB-ORM-RESULT-HYDRATION-145

Resource cleanup será independiente de transaction management.

## DB-ORM-RESULT-HYDRATION-146

Entity result shape respetará EntityHydrator invariants.

## DB-ORM-RESULT-HYDRATION-147

Scalar result shape evitará EntityHydrator innecesario.

## DB-ORM-RESULT-HYDRATION-148

Projection result shape evitará managed entity creation innecesaria.

## DB-ORM-RESULT-HYDRATION-149

DTO result shape evitará managed entity creation innecesaria.

## DB-ORM-RESULT-HYDRATION-150

Result shape deberá conocerse antes de interpretar logical cardinality.

## DB-ORM-RESULT-HYDRATION-151

Logical cardinality se calculará después de aplicar grouping relevante.

## DB-ORM-RESULT-HYDRATION-152

Logical root count no será row count.

## DB-ORM-RESULT-HYDRATION-153

JOIN multiplication no cambiará entity canonicality.

## DB-ORM-RESULT-HYDRATION-154

JOIN multiplication no cambiará logical root ordering indebidamente.

## DB-ORM-RESULT-HYDRATION-155

ResultHydrator deberá poder explicar physical-to-logical reduction.

## DB-ORM-RESULT-HYDRATION-156

Hydration diagnostics serán correlacionables con query telemetry.

## DB-ORM-RESULT-HYDRATION-157

Result shape mismatch fallará explícitamente.

## DB-ORM-RESULT-HYDRATION-158

Missing required result slots fallarán explícitamente.

## DB-ORM-RESULT-HYDRATION-159

Temporary result state será liberado deterministicamente.

## DB-ORM-RESULT-HYDRATION-160

VoltStack nunca confundirá una fila física con un resultado lógico cuando el `HydrationPlan` establezca agrupación, deduplicación, relaciones o una cardinalidad lógica distinta.

---

# 260. Fórmulas fundamentales

## 260.1 Result hydration

```text
LogicalResult
=
Hydrate(
    PhysicalResult,
    CompiledResultHydrationPlan,
    HydrationContext
)
```

---

# 261. Physical vs logical cardinality

```text
PhysicalCardinality
=
Count(ResultRows)
```

mientras:

```text
LogicalCardinality
=
Count(LogicalResultsAfterHydrationSemantics)
```

y:

```text
PhysicalCardinality
≠
LogicalCardinality
```

en general.

---

# 262. Root entity collection

```text
LogicalRoots
=
UniqueByRootIdentity(
    HydratedRootOccurrences
)
```

cuando el plan utiliza root identity deduplication.

---

# 263. Relationship graph

```text
HydratedGraph
=
CanonicalEntities
+
UniqueRelationshipEdges
+
RelationshipCompleteness
```

---

# 264. JOIN amplification

```text
JoinAmplification
=
PhysicalRows
/
max(1, UniqueLogicalRoots)
```

---

# 265. Result compression

```text
PhysicalToLogicalCompression
=
PhysicalRowCount
/
max(1, LogicalResultCount)
```

---

# 266. Cardinality correctness

```text
ValidResult
=
ExpectedLogicalCardinality
matches
ActualLogicalCardinality
```

---

# 267. ZERO_OR_ONE

```text
ZERO_OR_ONE(r)
=
null                  if |r| = 0
result                if |r| = 1
NonUniqueResult       if |r| > 1
```

donde:

```text
|r|
```

es cardinalidad lógica.

---

# 268. EXACTLY_ONE

```text
EXACTLY_ONE(r)
=
NoResult              if |r| = 0
result                if |r| = 1
NonUniqueResult       if |r| > 1
```

---

# 269. Safe streaming root

```text
MayYield(root)
=
RootHydrationComplete(root)
∧
NoFutureRowCanExtendRoot(root)
```

---

# 270. Ordered streaming

```text
RootKey(currentRow)
≠
RootKey(previousRows)

+
StableRootOrdering

⇒

PreviousRootMayFinalize
```

---

# 271. Safe resource lifecycle

```text
SafeHydrationResources
=
Acquire
→
Consume
→
FinalizeOrFail
→
Release
```

---

# 272. Result state isolation

```text
HydrationState(RequestA)
∩
HydrationState(RequestB)
=
∅
```

para mutable runtime state.

---

# 273. Master Formula

```text
Database Result Hydration System
=
Physical Result Consumption
+
Compiled Result Hydration Plans
+
Result Shape Semantics
+
Result Component Hydration
+
Entity Hydration Coordination
+
Scalar Hydration
+
Tuple Hydration
+
Projection Hydration
+
DTO Hydration
+
Mixed Result Hydration
+
Root Identity Resolution
+
Root Grouping
+
Root Deduplication
+
Logical Result Identity
+
JOIN Amplification Handling
+
Relationship Graph Assembly
+
Relationship Edge Deduplication
+
Relationship Completeness
+
Logical Cardinality Enforcement
+
Buffered Hydration
+
Streaming Hydration
+
Root Completion Detection
+
Cursor Integration
+
Resource Ownership
+
Cancellation
+
Partial Failure Semantics
+
Memory Governance
+
Type Conversion
+
Plan Compilation
+
Plan Caching
+
Persistent Runtime Isolation
+
Extension Governance
+
Security Validation
+
Telemetry
+
Diagnostics
+
Testing
```

---

# 274. Arquitectura final

```text
                       Execution Engine
                              │
                              ▼
                            Result
                              │
                              ▼
                CompiledResultHydrationPlan
                              │
                              ▼
                     ResultHydrator
                              │
             ┌────────────────┼─────────────────┐
             │                │                 │
             ▼                ▼                 ▼
         Row Reader       Shape Model       HydrationScope
             │                │                 │
             └────────────────┼─────────────────┘
                              ▼
                     Component Hydration
                 ┌────────────┼──────────────┐
                 │            │              │
                 ▼            ▼              ▼
              Entity        Scalar       Projection/DTO
                 │
                 ▼
           EntityHydrator
                 │
                 ▼
            IdentityMap
                 │
                 ▼
         Canonical Entities
                 │
                 └─────────────┐
                               ▼
                      Root Resolution
                               │
                               ▼
                         Root Registry
                               │
                               ▼
                    Relationship Assembly
                               │
                               ▼
                       Edge Deduplication
                               │
                               ▼
                    Completeness Tracking
                               │
                               ▼
                   Logical Result Assembly
                               │
                 ┌─────────────┴─────────────┐
                 │                           │
                 ▼                           ▼
              BUFFERED                   STREAMING
                 │                           │
                 ▼                           ▼
          Full Materialization         Root Finalization
                 │                           │
                 └─────────────┬─────────────┘
                               ▼
                    Cardinality Validation
                               │
                               ▼
                         Logical Result
                               │
                               ▼
                      Resource Release
```

---

# 275. Regla maestra

> **En VoltStack, el `ResultHydrator` no transforma filas en objetos de forma mecánica; interpreta un resultado físico conforme a un `HydrationPlan` para reconstruir su verdadera estructura lógica. Identidad, agrupación, deduplicación, relaciones y cardinalidad deberán evaluarse en el nivel lógico, por lo que el número de filas producido por la base de datos nunca será tratado automáticamente como el número de resultados visibles por la aplicación.**

---

# 276. Siguiente documento

```text
138_DATABASE_SCALAR_HYDRATION_SYSTEM.md
```

El siguiente documento deberá formalizar específicamente:

```text
ScalarHydrator
ScalarHydrationPlan
ScalarResult
ScalarType
database-to-PHP conversion
NULL semantics
MISSING semantics
numeric precision
BIGINT
DECIMAL / NUMERIC
boolean normalization
string/binary values
date/time scalars
UUID scalars
enum scalars
JSON scalars
custom scalar types
aggregate results
COUNT/SUM/AVG/MIN/MAX
scalar aliases
single scalar
scalar collections
scalar tuples
platform differences
overflow protection
precision preservation
conversion failures
streaming scalars
memory governance
telemetry
compiled converters
persistent runtime isolation
```

Su regla central será:

> **La hidratación escalar de VoltStack deberá preservar el significado y la precisión del valor producido por la base de datos; convertir un valor a un tipo PHP conveniente nunca podrá justificar pérdida silenciosa de precisión, rango, nulabilidad o semántica.**