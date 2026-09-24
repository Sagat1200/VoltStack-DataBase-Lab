# 139_DATABASE_TUPLE_HYDRATION_SYSTEM.md

# VoltStack Quantum Database
## Database Tuple Hydration System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 139 — Database Tuple Hydration System  
**Bloque:** 12 — Hydration  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Tuple Hydration System` define la arquitectura utilizada por VoltStack para materializar resultados compuestos formados por múltiples componentes lógicos dentro de una misma unidad de resultado.

Una tuple puede contener:

```text
Entity
Scalar
Projection
DTO
Value Object
Embedded Value
Nested Tuple
```

Ejemplo:

```text
User
+
COUNT(Order)
+
MAX(Order.created_at)
```

puede transformarse en:

```text
ResultTuple
├── user       → User#42
├── orderCount → 15
└── lastOrder  → DateTimeImmutable
```

La responsabilidad del sistema es conservar la semántica individual de cada componente mientras crea una representación compuesta, tipada, determinista e interpretable.

---

# 2. Principio central

> **Una tuple de VoltStack representa la composición explícita de varios resultados lógicos; la presencia de una entidad dentro de ella no convierte toda la tuple en una entidad ni permite deduplicar automáticamente el resultado únicamente por identidad de esa entidad.**

Por tanto:

```text
Tuple
≠
Entity
```

```text
Tuple
≠
Array arbitrario
```

```text
Tuple identity
≠
Entity identity
```

---

# 3. Problema arquitectónico

Considérese:

```sql
SELECT
    u.id,
    u.name,
    COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON ...
GROUP BY u.id, u.name
```

El resultado lógico puede ser:

```text
(User#1, 12)
(User#2, 4)
(User#3, 0)
```

Aquí:

```text
User#1
```

es una entidad administrada.

Pero:

```text
(User#1, 12)
```

es una tuple.

El ORM debe distinguir ambos conceptos.

---

# 4. Caso más complejo

Consulta:

```text
User
+
Order
+
OrderTotal
```

puede producir:

```text
(User#1, Order#10, 500.00)
(User#1, Order#11, 800.00)
```

Aunque:

```text
User#1
```

sea la misma instancia canónica en ambas filas, las tuples son resultados lógicos distintos.

Por ello:

```text
SameEntityComponent
⇏
SameTuple
```

---

# 5. Posición arquitectónica

```text
Execution Engine
      │
      ▼
    Result
      │
      ▼
ResultHydrator
      │
      ▼
┌──────────────────────┐
│ Tuple Hydrator       │
└──────────┬───────────┘
           │
    ┌──────┼─────────────┐
    ▼      ▼             ▼
 Entity  Scalar      Projection
Hydrator Hydrator      / DTO
    │      │             │
    └──────┼─────────────┘
           ▼
      Tuple Builder
           │
           ▼
      ResultTuple
```

---

# 6. TupleHydrator ≠ ResultHydrator

`ResultHydrator`:

```text
Result
→ logical result collection
```

`TupleHydrator`:

```text
row / logical row-group
→ one tuple
```

---

# 7. TupleHydrator ≠ EntityHydrator

El EntityHydrator resuelve:

```text
EntityKey
→ canonical entity
```

TupleHydrator solo consume esa entidad como uno de sus componentes.

---

# 8. TupleHydrator ≠ DTO Hydrator

Una tuple es una estructura genérica de resultado.

Un DTO posee un tipo de aplicación específico.

Ejemplo:

```text
ResultTuple(User, 15)
```

vs:

```php
new UserSummary(
    user: $user,
    orderCount: 15,
);
```

---

# 9. Objetivos

El sistema deberá proporcionar:

```text
typed tuple slots
named aliases
positional access
entity components
scalar components
projection components
DTO components
nested tuples
tuple shape validation
tuple construction plans
logical tuple identity
tuple deduplication
tuple ordering
tuple cardinality
streaming
memory governance
extension support
telemetry
```

---

# 10. No objetivos

Tuple Hydration no será responsable de:

```text
executing SQL
building queries
managing transactions
persisting entities
tracking entity changes
creating IdentityMap entries directly
deciding eager-loading strategy
serializing HTTP responses
```

---

# 11. Tuple Shape

Toda tuple deberá poseer una forma explícita.

```php
final readonly class TupleShape
{
    /**
     * @param list<TupleSlotDefinition> $slots
     */
    public function __construct(
        public array $slots,
    ) {}
}
```

---

# 12. TupleSlotDefinition

```php
final readonly class TupleSlotDefinition
{
    public function __construct(
        public int $position,
        public ?string $alias,
        public TupleComponentType $type,
        public TupleComponentPlan $plan,
        public bool $nullable,
    ) {}
}
```

---

# 13. TupleComponentType

```php
enum TupleComponentType
{
    case ENTITY;
    case SCALAR;
    case PROJECTION;
    case DTO;
    case VALUE_OBJECT;
    case TUPLE;
    case CUSTOM;
}
```

---

# 14. Tuple positions

Una tuple tendrá orden estable.

Ejemplo:

```text
position 0 → user
position 1 → orderCount
position 2 → total
```

---

# 15. Named aliases

Además del acceso posicional:

```php
$tuple[0];
```

VoltStack deberá permitir:

```php
$tuple->get('user');
$tuple->get('orderCount');
```

---

# 16. Alias uniqueness

Dentro de una tuple:

```text
alias
```

deberá ser único.

No:

```text
user
user
```

---

# 17. Duplicate alias

Resultado:

```text
DuplicateTupleAliasException
```

en compilación del plan cuando sea posible.

---

# 18. Missing alias

Los aliases son opcionales.

Una tuple puramente posicional seguirá siendo válida si la API lo permite.

---

# 19. ResultTuple

Propuesta:

```php
final readonly class ResultTuple implements ArrayAccess, Countable, IteratorAggregate
{
    /**
     * @param list<mixed> $values
     * @param array<string, int> $aliases
     */
    public function __construct(
        private array $values,
        private array $aliases = [],
    ) {}

    public function get(int|string $slot): mixed
    {
        // ...
    }
}
```

---

# 20. ResultTuple immutable

Después de construirse:

```text
ResultTuple
=
immutable
```

Esto evita que la estructura del resultado cambie accidentalmente.

---

# 21. Tuple value mutability

La tuple puede contener:

```text
mutable Entity
```

aunque la tuple en sí sea immutable.

Por tanto:

```text
ImmutableTuple
≠
ImmutableComponents
```

---

# 22. Tuple plan

```php
final readonly class TupleHydrationPlan
{
    /**
     * @param list<CompiledTupleSlotPlan> $slots
     */
    public function __construct(
        public TupleShape $shape,
        public array $slots,
        public TupleConstructionStrategy $construction,
        public TupleDeduplicationStrategy $deduplication,
    ) {}
}
```

---

# 23. CompiledTupleSlotPlan

```php
final readonly class CompiledTupleSlotPlan
{
    public function __construct(
        public int $position,
        public ?string $alias,
        public TupleComponentHydrator $hydrator,
        public TupleComponentPlan $component,
    ) {}
}
```

---

# 24. Tuple hydration pipeline

```text
Result Row
    │
    ▼
CompiledTupleHydrationPlan
    │
    ▼
Slot #0 Hydration
    │
    ▼
Slot #1 Hydration
    │
    ▼
Slot #2 Hydration
    │
    ▼
Validate Slot Results
    │
    ▼
Tuple Construction
    │
    ▼
Logical Tuple Key
    │
    ▼
Optional Deduplication
    │
    ▼
ResultTuple
```

---

# 25. Entity slot

Para:

```text
slot user
```

el TupleHydrator delega:

```text
EntityHydrator
```

---

# 26. Identity preservation

Si dos tuples contienen:

```text
User#42
```

ambas deberán apuntar a:

```text
same canonical object
```

dentro del mismo `PersistenceContext`.

---

# 27. Example

```php
$tupleA->get('user') === $tupleB->get('user');
```

deberá ser `true` si representan la misma entidad managed.

---

# 28. Tuple duplication remains possible

Aunque compartan entidad:

```text
(User#42, 5)
(User#42, 8)
```

son tuples distintas.

---

# 29. Scalar slot

Usa:

```text
Scalar Hydration System
```

por ejemplo:

```text
"15"
↓
int(15)
```

---

# 30. Decimal slot

```text
"9500.75"
↓
Decimal
```

sin pérdida de precisión.

---

# 31. Projection slot

Una projection será hidratada según su propio plan.

No entra automáticamente al IdentityMap.

---

# 32. DTO slot

Igualmente:

```text
DTO
```

puede coexistir con:

```text
Entity
```

dentro de una tuple.

---

# 33. Nested tuple

Una tuple puede contener otra.

Ejemplo:

```text
(
    User,
    (
        OrderCount,
        TotalRevenue
    )
)
```

---

# 34. Nested plan

Debe ser explícito.

No se inferirá recursivamente a partir de arrays arbitrarios.

---

# 35. Tuple Depth

Podrá existir:

```php
final readonly class TupleDepth
{
    public function __construct(
        public int $value,
    ) {}
}
```

para gobernanza de recursos.

---

# 36. Maximum nesting

Una policy podrá restringir profundidad.

Ejemplo:

```text
maxTupleDepth = 8
```

para evitar estructuras patológicas.

---

# 37. Null tuple components

Cada slot define su propia nullability.

Ejemplo:

```text
(User, ?LastOrderDate)
```

---

# 38. SQL NULL scalar

Si el slot es nullable:

```text
NULL → null
```

---

# 39. NULL entity component

En un LEFT JOIN puede existir:

```text
User + ?Organization
```

Si el identificador de `Organization` representa ausencia:

```text
organization slot = null
```

---

# 40. Entity absence ≠ empty entity

Nunca:

```text
Organization(
    id: null,
    ...
)
```

---

# 41. Missing tuple component

Debe distinguirse:

```text
MISSING
```

de:

```text
NULL
```

---

# 42. Missing required slot

Produce:

```text
TupleMissingSlotException
```

---

# 43. Optional structural slot

Si VoltStack soporta slots opcionales, deberán estar declarados explícitamente.

No desaparecer arbitrariamente dependiendo de la row.

---

# 44. Stable shape

Toda tuple producida por un mismo plan deberá mantener:

```text
same logical slot structure
```

aunque valores concretos sean `null`.

---

# 45. Tuple Construction Strategy

```php
enum TupleConstructionStrategy
{
    case RESULT_TUPLE;
    case ARRAY;
    case CUSTOM;
}
```

---

# 46. RESULT_TUPLE

Default recomendado.

Ventajas:

```text
typed semantics
aliases
diagnostics
immutability
future metadata
```

---

# 47. ARRAY

Puede utilizarse como API low-level.

Pero pierde:

```text
explicit tuple type
metadata richness
strong diagnostics
```

---

# 48. CUSTOM

Extensiones podrán proporcionar estructuras propias.

---

# 49. Tuple identity

Una tuple puede requerir una identidad lógica para deduplicación.

No será EntityKey.

---

# 50. TupleLogicalKey

```php
final readonly class TupleLogicalKey
{
    public function __construct(
        public array $components,
    ) {}
}
```

---

# 51. Key components

Podrán derivarse de:

```text
entity keys
scalar canonical values
projection keys
nested tuple keys
```

según el plan.

---

# 52. Full tuple identity

Ejemplo:

```text
(User#1, "admin")
```

y:

```text
(User#1, "editor")
```

tienen keys diferentes.

---

# 53. Tuple deduplication

Strategies:

```php
enum TupleDeduplicationStrategy
{
    case NONE;
    case FULL_TUPLE;
    case SELECTED_SLOTS;
    case CUSTOM;
}
```

---

# 54. Default

```text
NONE
```

o la estrategia explícitamente derivada de la semántica del query.

---

# 55. No automatic entity-root dedup

Si resultado es tuple:

```text
(User, Role)
```

no deberá deduplicarse únicamente por `User`.

---

# 56. Example

Rows:

```text
User#1 Role#10
User#1 Role#11
```

Resultado correcto:

```text
(User#1, Role#10)
(User#1, Role#11)
```

---

# 57. Wrong result

No:

```text
(User#1, Role#10)
```

solamente.

---

# 58. FULL_TUPLE dedup

Dos tuples se consideran equivalentes si todos los componentes definidos por el plan son equivalentes.

---

# 59. Entity component equality

Para managed entities:

```text
EntityKey
```

deberá preferirse sobre comparación profunda de propiedades.

---

# 60. Scalar component equality

Deberá respetar:

```text
canonical scalar semantics
```

No PHP loose equality.

---

# 61. Example

No asumir:

```php
'1' == 1
```

como base universal de deduplicación.

El ScalarHydrator ya habrá canonicalizado valores.

---

# 62. Decimal equality

Utilizar semántica de `Decimal`, no `float`.

---

# 63. Value Object equality

El tipo correspondiente deberá proporcionar semántica estable de key/equality cuando se use para dedup.

---

# 64. Selected-slot dedup

Ejemplo:

```text
Tuple:
User
Department
LastLogin
```

un plan avanzado podría deduplicar por:

```text
User + Department
```

solo si esa semántica fue explícitamente declarada.

---

# 65. Hashing

Un tuple key podrá tener hash.

```text
Hash(TupleLogicalKey)
```

para lookup O(1) promedio.

---

# 66. Hash ≠ equality

Colisiones deberán verificarse estructuralmente.

---

# 67. Tuple ordering

El orden lógico seguirá el orden definido por el Result Hydration Plan.

---

# 68. Dedup order

Cuando exista dedup:

```text
first logical occurrence
```

conserva normalmente la posición.

---

# 69. No implicit sorting

Tuple Hydration no ordenará por:

```text
entity ID
alias
scalar value
```

sin instrucción explícita.

---

# 70. Tuple collection

Para MANY:

```text
Collection<ResultTuple>
```

o:

```text
iterable<ResultTuple>
```

---

# 71. Tuple cardinality

El Result Hydrator sigue siendo responsable de:

```text
MANY
ZERO_OR_ONE
EXACTLY_ONE
FIRST
```

sobre tuples lógicas.

---

# 72. Physical rows ≠ logical tuple count

Especialmente si cada tuple requiere agrupación.

---

# 73. Grouped tuple

Ejemplo:

```text
(User, OrdersCollection, Total)
```

puede requerir múltiples rows para completar una tuple.

---

# 74. Tuple grouping plan

Podrá definir:

```text
NONE
ENTITY_ROOT
SELECTED_SLOTS
CUSTOM
```

---

# 75. Entity-root grouped tuple

Ejemplo:

```text
(User, Orders, OrderCount)
```

puede agrupar varias rows de un mismo `User`.

---

# 76. But

Agrupación por root es distinta de deduplicación final.

---

# 77. Grouping

Responde:

```text
¿qué filas pertenecen al mismo resultado compuesto?
```

---

# 78. Deduplication

Responde:

```text
¿dos resultados ya construidos representan
el mismo resultado lógico?
```

---

# 79. Keep distinction

```text
Grouping
≠
Deduplication
```

---

# 80. Relationship-containing tuple

Ejemplo:

```text
(User with Orders, Revenue)
```

puede necesitar:

```text
relationship assembly
```

antes de finalizar la tuple.

---

# 81. Entity component lifecycle

El TupleHydrator no dispara lifecycle adicional sobre una entidad solo porque aparezca en una tuple.

---

# 82. postLoad

Pertenece a:

```text
EntityHydrator
```

---

# 83. Repeated entity occurrence

Si `User#1` aparece en 20 tuples:

```text
postLoad
```

no se ejecuta 20 veces.

---

# 84. DTO inside tuple

DTO lifecycle ORM:

```text
none
```

salvo extensión específica no-ORM.

---

# 85. Tuple itself

No entra en:

```text
IdentityMap
UnitOfWork
EntityState
SnapshotRegistry
```

---

# 86. Critical rule

```text
ResultTuple
≠
Managed Object
```

---

# 87. Entity references inside tuple

Las entidades internas sí mantienen sus propias garantías ORM.

---

# 88. Tuple immutability and UoW

Modificar una entidad dentro de la tuple:

```php
$tuple->get('user')->rename('Alice');
```

puede generar dirty tracking normalmente.

Modificar la tuple:

```php
$tuple['user'] = ...
```

no debería permitirse en `ResultTuple` default.

---

# 89. Tuple projections

Un slot puede contener:

```text
Projection<UserSummary>
```

sin registrar entidad.

---

# 90. Entity vs Projection in same tuple

Ejemplo:

```text
(User entity, BillingSummary projection)
```

es válido.

---

# 91. Alias API

Ejemplo:

```php
$tuple->get('user');
$tuple->get('totalRevenue');
```

---

# 92. Positional API

```php
$tuple->get(0);
$tuple->get(1);
```

---

# 93. Invalid alias

Debe producir:

```text
UnknownTupleAliasException
```

No devolver `null` ambiguamente.

---

# 94. Null value ambiguity

Porque:

```text
alias exists and value = null
```

es diferente de:

```text
alias does not exist
```

---

# 95. has()

API propuesta:

```php
$tuple->has('lastOrder');
```

---

# 96. getOrNull()

Opcionalmente:

```php
$tuple->getOrNull('lastOrder');
```

con semántica explícita.

---

# 97. Tuple metadata

ResultTuple podrá exponer metadata ligera:

```php
$tuple->aliases();
$tuple->count();
```

Pero no deberá exponer internals mutables del HydrationPlan.

---

# 98. Named tuple type

En el futuro VoltStack podría proporcionar:

```text
NamedTuple
```

como una representation optimizada.

---

# 99. DTO conversion

El usuario podrá convertir una tuple a DTO explícitamente.

Eso será distinto de hidratar directamente como DTO.

---

# 100. Why distinction

```text
Tuple → DTO
```

es transformación posterior.

```text
Result → DTO Hydrator
```

es una estrategia de hidratación directa.

---

# 101. Performance model

Para una tuple con `s` slots:

```text
T(tuple)
≈
Σ HydrationCost(slot_i)
+
ConstructionCost
+
OptionalKeyCost
```

---

# 102. Simple scalar tuple

Para:

```text
(int, string, Decimal)
```

la complejidad será aproximadamente:

```text
O(s)
```

---

# 103. Entity tuple

Incluye:

```text
IdentityMap lookups
```

con costo promedio O(1) por entity component.

---

# 104. Compiled slot access

El plan deberá usar:

```text
ResultSlot index
```

para evitar resolución repetida de aliases.

---

# 105. No metadata scanning per tuple

No:

```php
foreach ($metadata as ...)
```

en hot path cuando el plan ya está compilado.

---

# 106. Generated tuple hydrators

VoltStack podrá generar código especializado.

Ejemplo conceptual:

```php
$user = $entityHydrator->hydrate(...);
$count = $intConverter($row[4]);
$total = $decimalConverter($row[5]);

return new ResultTuple(
    [$user, $count, $total],
    [
        'user' => 0,
        'orderCount' => 1,
        'total' => 2,
    ],
);
```

---

# 107. Generated code safety

Nunca utilizar values DB como código PHP.

---

# 108. Tuple Plan Cache

Planes inmutables podrán cachearse.

---

# 109. Cache key

Puede incluir:

```text
ResultShapeId
TupleShapeFingerprint
MetadataGeneration
TypeRegistryGeneration
HydrationPlanGeneration
```

---

# 110. Tuple shape fingerprint

```text
TupleShapeFingerprint
=
Hash(
    ordered slot types,
    aliases,
    component plan IDs,
    nullability,
    construction strategy
)
```

---

# 111. Cache invalidation

Cambio en:

```text
entity metadata
scalar type mapping
projection mapping
DTO constructor plan
```

deberá invalidar planes incompatibles.

---

# 112. Cache contents

Permitido:

```text
compiled slot plans
converter references
hydrator strategy references
alias lookup indexes
```

---

# 113. Cache forbidden

No:

```text
actual tuples
actual entities
Result
EntityManager
PersistenceContext
current tenant mutable object
```

---

# 114. Buffered tuple hydration

```text
Result
→ hydrate all tuples
→ collection
```

---

# 115. Streaming tuples

```text
ResultCursor
→ tuple
→ yield
```

cuando:

```text
one logical tuple
=
one completed row/group
```

---

# 116. Grouped streaming tuple

Si una tuple requiere varias rows:

```text
current tuple group
```

deberá completarse antes de yield.

---

# 117. Safe yield formula

```text
MayYield(tuple)
=
TupleComplete
∧
NoFutureRowCanExtendTuple
```

---

# 118. Ordered streaming requirement

Cuando agrupación depende de root:

```text
stable root ordering
```

debe estar garantizado.

---

# 119. Unordered result

Debe:

```text
buffer
```

o:

```text
reject streaming strategy
```

---

# 120. No incomplete tuple yield

Regla absoluta.

---

# 121. Streaming memory

Para tuple simple:

```text
O(1)
```

estado adicional, aparte de componentes ORM retenidos.

---

# 122. Managed entity components

Aunque Tuple Hydrator sea streaming, las entidades pueden seguir acumulándose en IdentityMap.

Por tanto:

```text
TupleStreamingMemory
≠
ORMManagedMemory
```

---

# 123. Streaming entity policy

Debe coordinarse con:

```text
StreamingIdentityPolicy
```

definida por Entity Hydration.

---

# 124. Cancellation

Si el consumidor cancela:

```text
current incomplete tuple
```

se descarta.

---

# 125. Previously yielded tuples

Pueden haber sido entregadas.

No existe atomicidad global del stream.

---

# 126. Tuple failure

Si falla un slot:

```text
tuple hydration fails
```

---

# 127. Example

```text
user slot → success
count slot → success
decimal slot → conversion failure
```

La tuple no deberá entregarse parcialmente.

---

# 128. Existing entity side effect

Si el entity slot materializó una entidad antes de fallar otro slot, esa entidad puede ya estar correctamente managed.

---

# 129. Important distinction

```text
TupleOutcome = FAILED
```

puede coexistir con:

```text
EntityHydrationOutcome = SUCCEEDED
```

---

# 130. No fake rollback

Tuple Hydrator no deberá intentar eliminar una entidad válida del IdentityMap solo porque otro slot falló.

---

# 131. Temporary tuple state

Sí deberá descartar:

```text
partially built tuple
temporary tuple key
tuple-local buffers
```

---

# 132. Entity partial hydration failure

Si el fallo está dentro del EntityHydrator, se aplican sus reglas de rollback interno.

---

# 133. Failure dimensions

Conviene conservar:

```text
TupleHydrationOutcome
ComponentHydrationOutcomes
ResourceOutcome
```

para diagnóstico.

---

# 134. Error hierarchy

```text
DatabaseTupleHydrationException
├── TupleHydrationPlanException
├── TupleShapeException
├── TupleSlotException
├── TupleMissingSlotException
├── TupleNullViolationException
├── TupleAliasException
│   ├── DuplicateTupleAliasException
│   └── UnknownTupleAliasException
├── TupleComponentHydrationException
├── TupleEntityComponentException
├── TupleScalarComponentException
├── TupleProjectionComponentException
├── TupleDtoComponentException
├── TupleNestedHydrationException
├── TupleDepthException
├── TupleConstructionException
├── TupleKeyException
├── TupleDeduplicationException
├── TupleGroupingException
├── TupleStreamingException
├── TupleStreamingPreconditionException
├── TupleIncompleteException
├── TupleCancellationException
├── TupleResourceLimitException
├── TupleRuntimeIsolationException
└── TupleHydrationInvariantException
```

---

# 135. Error context

Puede incluir:

```text
HydrationPlanId
TuplePlanId
TupleSlot
Alias
ComponentType
PhysicalRowNumber
LogicalTupleNumber
HydrationScopeId
```

---

# 136. Sensitive data

No incluir por defecto el valor concreto de:

```text
password
token
email
personal data
JSON document
```

en excepciones.

---

# 137. Resource governance

Policy:

```php
final readonly class TupleHydrationResourcePolicy
{
    public function __construct(
        public int $maximumDepth,
        public ?int $maximumTupleCount,
        public ?int $maximumBufferedBytes,
    ) {}
}
```

---

# 138. Maximum tuple count

No sustituye pagination.

Es una protección operacional.

---

# 139. Deep nesting protection

Una estructura accidentalmente recursiva deberá fallar antes de provocar agotamiento de memoria.

---

# 140. Extension system

Podrán registrarse:

```text
CustomTupleComponentHydrator
CustomTupleConstructionStrategy
CustomTupleKeyStrategy
CustomTupleDeduplicationStrategy
```

---

# 141. Extension constraints

Una extensión no podrá:

```text
bypass EntityHydrator for managed entities
break IdentityMap canonicality
execute SQL
commit/rollback
mutate global request state
```

---

# 142. Custom component

Ejemplo:

```text
GeometrySummary
```

podrá tener su propio component hydrator.

---

# 143. Component registry

```php
interface TupleComponentHydratorRegistry
{
    public function resolve(
        TupleComponentType $type
    ): TupleComponentHydrator;
}
```

---

# 144. Registry freezing

En producción:

```text
register
→ validate
→ freeze
```

---

# 145. Persistent runtime

En FrankenPHP se podrán compartir:

```text
CompiledTupleHydrationPlan
CompiledTupleSlotPlan
TupleShape
AliasLookupMap
```

si son inmutables.

---

# 146. Scoped mutable state

Nunca compartir:

```text
current tuple
current row
tuple registry
grouping state
current root
HydrationScope
EntityManager
ResultCursor
```

---

# 147. Request isolation

```text
TupleHydrationState(Request A)
∩
TupleHydrationState(Request B)
=
∅
```

---

# 148. RoadRunner

Misma garantía.

---

# 149. OpenSwoole

Todo mutable state será coroutine/logical-context scoped.

---

# 150. No static current tuple

Prohibido:

```php
TupleHydrator::$currentTuple;
```

---

# 151. Concurrency

Un mismo mutable `HydrationScope` no será concurrency-safe por defecto.

---

# 152. Parallel tuple slots

Podría parecer atractivo hidratar slots en paralelo.

No deberá hacerse por defecto porque varios slots pueden:

```text
touch same IdentityMap
share related entities
require ordering
share hydration state
```

---

# 153. V1

Slots se hidratan determinísticamente en orden compilado.

---

# 154. Slot order

El orden interno puede optimizarse solo si no altera:

```text
observable lifecycle
failure semantics
identity coordination
```

---

# 155. Default

Preferir:

```text
plan order
```

estable.

---

# 156. Telemetry

Métricas propuestas:

```text
orm.tuple_hydration.operations
orm.tuple_hydration.tuples
orm.tuple_hydration.slots
orm.tuple_hydration.duration
orm.tuple_hydration.failures

orm.tuple_hydration.entity_slots
orm.tuple_hydration.scalar_slots
orm.tuple_hydration.dto_slots
orm.tuple_hydration.projection_slots
orm.tuple_hydration.nested_slots

orm.tuple_hydration.deduplicated
orm.tuple_hydration.grouped
orm.tuple_hydration.streaming
orm.tuple_hydration.buffered
```

---

# 157. Average slot count

```text
AverageTupleWidth
=
TotalHydratedSlots
/
HydratedTuples
```

---

# 158. Tuple dedup ratio

```text
TupleDedupRatio
=
ObservedTupleCandidates
/
UniqueLogicalTuples
```

cuando dedup está activo.

---

# 159. Entity reuse inside tuples

Puede medirse mediante las métricas generales de EntityHydrator.

No duplicar innecesariamente telemetría.

---

# 160. Debug toolbar

Ejemplo:

```text
Tuple Hydration
──────────────────────────────

Shape:
  (User, orderCount, revenue)

Tuples:              1,250
Slots hydrated:      3,750

Entity slots:        1,250
Scalar slots:        2,500

Unique User entities:  410
IdentityMap reuse:      840

Tuple dedup:
  strategy: NONE

Mode:
  BUFFERED

Duration:
  7.8 ms
```

---

# 161. Testing Strategy

Debe probarse:

```text
scalar-only tuples
entity + scalar
entity + entity
entity + DTO
entity + projection
nested tuples
NULL slots
missing slots
aliases
deduplication
grouping
streaming
failure
persistent runtime
```

---

# 162. Simple scalar tuple

Input:

```text
1 | Alice | 500.25
```

Output:

```text
(int, string, Decimal)
```

---

# 163. Entity + scalar

```text
User#1 | 15
```

produce:

```text
(User canonical instance, int 15)
```

---

# 164. Same entity multiple tuples

```text
User#1 | ADMIN
User#1 | EDITOR
```

produce dos tuples.

---

# 165. Identity reuse test

Ambas tuples contienen el mismo objeto `User`.

---

# 166. Entity + entity

```text
User#1 | Organization#5
```

ambos componentes respetan IdentityMap.

---

# 167. Nullable entity slot

LEFT JOIN sin Organization produce:

```text
(User#1, null)
```

---

# 168. Alias test

```php
$tuple->get('user');
```

retorna el slot correcto.

---

# 169. Duplicate alias test

Plan falla temprano.

---

# 170. Unknown alias test

No devuelve null silenciosamente.

---

# 171. Missing required slot test

Falla explícitamente.

---

# 172. NULL allowed test

Se conserva null.

---

# 173. Nested tuple test

Construcción recursiva correcta dentro de profundidad permitida.

---

# 174. Max depth test

Falla deterministicamente.

---

# 175. FULL_TUPLE dedup test

Tuples idénticas se consolidan cuando policy lo exige.

---

# 176. NONE dedup test

Tuples duplicadas físicamente permanecen duplicadas.

---

# 177. Selected-slot test

Solo slots seleccionados participan en logical key.

---

# 178. Grouping test

Múltiples rows construyen una tuple lógica cuando el plan lo exige.

---

# 179. Grouping vs dedup test

Se verifican como fases distintas.

---

# 180. Entity lifecycle test

Tuple occurrence repetida no dispara `postLoad` nuevamente.

---

# 181. DTO slot test

DTO no entra a UoW.

---

# 182. Projection slot test

Projection no entra al IdentityMap.

---

# 183. Failure slot 3 test

Tuple no se entrega parcialmente.

---

# 184. Existing entity after tuple failure

Entidad correctamente materializada permanece coherente.

---

# 185. Streaming tuple test

Cada tuple completa se yield correctamente.

---

# 186. Streaming grouped tuple test

No yield hasta completar group.

---

# 187. Unordered streaming test

Se rechaza cuando required ordering no está garantizado.

---

# 188. Cancellation test

Current incomplete tuple se descarta.

---

# 189. Persistent runtime test

Request A no comparte tuple state con B.

---

# 190. Performance tests

Benchmarks mínimos:

```text
2 scalar slots
10 scalar slots
entity + scalar
2 entities + 4 scalars
nested tuple
100k scalar tuples
100k entity/scalar tuples
IdentityMap hit-heavy tuples
buffered vs streaming
```

---

# 191. Architectural Invariants

## DB-ORM-TUPLE-HYDRATION-001

Tuple Hydration no ejecutará SQL.

## DB-ORM-TUPLE-HYDRATION-002

Tuple Hydration no generará SQL.

## DB-ORM-TUPLE-HYDRATION-003

Tuple Hydration será distinta de Result Hydration.

## DB-ORM-TUPLE-HYDRATION-004

Tuple Hydration será distinta de Entity Hydration.

## DB-ORM-TUPLE-HYDRATION-005

Tuple será distinta de Entity.

## DB-ORM-TUPLE-HYDRATION-006

Tuple será distinta de DTO.

## DB-ORM-TUPLE-HYDRATION-007

Tuple tendrá shape explícito.

## DB-ORM-TUPLE-HYDRATION-008

Tuple slots tendrán orden estable.

## DB-ORM-TUPLE-HYDRATION-009

Aliases serán únicos cuando existan.

## DB-ORM-TUPLE-HYDRATION-010

Duplicate alias fallará explícitamente.

## DB-ORM-TUPLE-HYDRATION-011

Unknown alias será distinto de null value.

## DB-ORM-TUPLE-HYDRATION-012

Tuple podrá permitir acceso posicional.

## DB-ORM-TUPLE-HYDRATION-013

Tuple podrá permitir acceso nominal.

## DB-ORM-TUPLE-HYDRATION-014

Default ResultTuple será immutable.

## DB-ORM-TUPLE-HYDRATION-015

Tuple immutability no implicará component immutability.

## DB-ORM-TUPLE-HYDRATION-016

Entity slots utilizarán EntityHydrator.

## DB-ORM-TUPLE-HYDRATION-017

Scalar slots utilizarán ScalarHydrator.

## DB-ORM-TUPLE-HYDRATION-018

Projection slots utilizarán projection semantics.

## DB-ORM-TUPLE-HYDRATION-019

DTO slots utilizarán DTO semantics.

## DB-ORM-TUPLE-HYDRATION-020

Nested tuple utilizará plan explícito.

## DB-ORM-TUPLE-HYDRATION-021

Nested depth será gobernable.

## DB-ORM-TUPLE-HYDRATION-022

Same EntityKey en múltiples tuples reutilizará object instance.

## DB-ORM-TUPLE-HYDRATION-023

Same entity component no implicará same tuple.

## DB-ORM-TUPLE-HYDRATION-024

Tuple identity será distinta de EntityKey.

## DB-ORM-TUPLE-HYDRATION-025

Logical tuple key será explícita cuando se necesite.

## DB-ORM-TUPLE-HYDRATION-026

Tuple deduplication será policy-driven.

## DB-ORM-TUPLE-HYDRATION-027

Entity-root dedup no será automática para tuples.

## DB-ORM-TUPLE-HYDRATION-028

FULL_TUPLE comparará todos los slots relevantes.

## DB-ORM-TUPLE-HYDRATION-029

Entity equality dentro de tuple key utilizará identidad semántica.

## DB-ORM-TUPLE-HYDRATION-030

Scalar equality utilizará valores canonicalizados.

## DB-ORM-TUPLE-HYDRATION-031

PHP loose equality no gobernará dedup universalmente.

## DB-ORM-TUPLE-HYDRATION-032

Hash será distinto de logical tuple key.

## DB-ORM-TUPLE-HYDRATION-033

Hash collision deberá verificarse.

## DB-ORM-TUPLE-HYDRATION-034

Tuple ordering respetará result semantics.

## DB-ORM-TUPLE-HYDRATION-035

Dedup conservará first logical occurrence salvo policy distinta.

## DB-ORM-TUPLE-HYDRATION-036

TupleHydrator no hará implicit sorting.

## DB-ORM-TUPLE-HYDRATION-037

Grouping será distinto de deduplication.

## DB-ORM-TUPLE-HYDRATION-038

Grouping determinará qué rows forman una tuple.

## DB-ORM-TUPLE-HYDRATION-039

Deduplication determinará equivalencia entre tuples construidas.

## DB-ORM-TUPLE-HYDRATION-040

Tuple con relaciones podrá requerir múltiples rows.

## DB-ORM-TUPLE-HYDRATION-041

Grouped tuple no se finalizará antes de estar completa.

## DB-ORM-TUPLE-HYDRATION-042

Result cardinality se aplicará sobre tuples lógicas.

## DB-ORM-TUPLE-HYDRATION-043

Physical row count no será tuple cardinality universalmente.

## DB-ORM-TUPLE-HYDRATION-044

Tuple no entrará al IdentityMap.

## DB-ORM-TUPLE-HYDRATION-045

Tuple no entrará al UnitOfWork.

## DB-ORM-TUPLE-HYDRATION-046

Tuple no recibirá EntityState.

## DB-ORM-TUPLE-HYDRATION-047

Tuple no recibirá EntitySnapshot.

## DB-ORM-TUPLE-HYDRATION-048

Entity components conservarán sus garantías ORM.

## DB-ORM-TUPLE-HYDRATION-049

Tuple occurrence no disparará lifecycle adicional sobre entity reutilizada.

## DB-ORM-TUPLE-HYDRATION-050

Repeated entity tuple occurrence no repetirá postLoad.

## DB-ORM-TUPLE-HYDRATION-051

DTO component no tendrá ORM lifecycle por defecto.

## DB-ORM-TUPLE-HYDRATION-052

Projection component no tendrá ORM lifecycle por defecto.

## DB-ORM-TUPLE-HYDRATION-053

NULL será distinto de MISSING.

## DB-ORM-TUPLE-HYDRATION-054

Missing required tuple slot fallará.

## DB-ORM-TUPLE-HYDRATION-055

Nullable tuple slot podrá contener null.

## DB-ORM-TUPLE-HYDRATION-056

Absent related entity podrá hidratarse como null.

## DB-ORM-TUPLE-HYDRATION-057

Absent entity no se materializará como empty entity.

## DB-ORM-TUPLE-HYDRATION-058

Tuple shape será estable entre resultados del mismo plan.

## DB-ORM-TUPLE-HYDRATION-059

Slot positions no cambiarán por row.

## DB-ORM-TUPLE-HYDRATION-060

Compiled tuple plan será inmutable.

## DB-ORM-TUPLE-HYDRATION-061

Compiled tuple plan podrá cachearse.

## DB-ORM-TUPLE-HYDRATION-062

Tuple plan cache no almacenará actual tuples.

## DB-ORM-TUPLE-HYDRATION-063

Tuple plan cache no almacenará entities.

## DB-ORM-TUPLE-HYDRATION-064

Tuple plan cache no almacenará EntityManager.

## DB-ORM-TUPLE-HYDRATION-065

Metadata generation participará en cache compatibility.

## DB-ORM-TUPLE-HYDRATION-066

Type generation participará cuando afecte slots.

## DB-ORM-TUPLE-HYDRATION-067

Hot path utilizará compiled slot access.

## DB-ORM-TUPLE-HYDRATION-068

Metadata no se escaneará innecesariamente por tuple.

## DB-ORM-TUPLE-HYDRATION-069

Generated tuple hydration code utilizará metadata validada.

## DB-ORM-TUPLE-HYDRATION-070

Database values no serán ejecutados como código.

## DB-ORM-TUPLE-HYDRATION-071

Tuple construction strategy será explícita.

## DB-ORM-TUPLE-HYDRATION-072

RESULT_TUPLE será representación default recomendada.

## DB-ORM-TUPLE-HYDRATION-073

ARRAY mode podrá existir como escape hatch.

## DB-ORM-TUPLE-HYDRATION-074

ARRAY mode no cambiará component semantics.

## DB-ORM-TUPLE-HYDRATION-075

Streaming tuple hydration será explícita.

## DB-ORM-TUPLE-HYDRATION-076

Simple completed tuple podrá yield por row.

## DB-ORM-TUPLE-HYDRATION-077

Grouped tuple requerirá completion detection.

## DB-ORM-TUPLE-HYDRATION-078

Ordered grouped streaming requerirá ordering guarantee.

## DB-ORM-TUPLE-HYDRATION-079

Unordered grouped streaming deberá bufferizar o rechazarse.

## DB-ORM-TUPLE-HYDRATION-080

Incomplete tuple no será yielded.

## DB-ORM-TUPLE-HYDRATION-081

Cancellation descartará current incomplete tuple.

## DB-ORM-TUPLE-HYDRATION-082

Streaming no prometerá global materialization atomicity.

## DB-ORM-TUPLE-HYDRATION-083

Managed entity components podrán aumentar IdentityMap aunque tuple stream sea bounded.

## DB-ORM-TUPLE-HYDRATION-084

Streaming memory y IdentityMap memory serán dimensiones distintas.

## DB-ORM-TUPLE-HYDRATION-085

Failure en un slot impedirá entregar tuple parcial.

## DB-ORM-TUPLE-HYDRATION-086

Tuple failure no deshará artificialmente entity hydration exitosa.

## DB-ORM-TUPLE-HYDRATION-087

Tuple outcome será distinto de component outcome.

## DB-ORM-TUPLE-HYDRATION-088

TupleHydrator no hará rollback.

## DB-ORM-TUPLE-HYDRATION-089

TupleHydrator no hará commit.

## DB-ORM-TUPLE-HYDRATION-090

TupleHydrator no hará flush.

## DB-ORM-TUPLE-HYDRATION-091

TupleHydrator no administrará Connection.

## DB-ORM-TUPLE-HYDRATION-092

TupleHydrator no conocerá PDO.

## DB-ORM-TUPLE-HYDRATION-093

TupleHydrator no conocerá SQL dialect.

## DB-ORM-TUPLE-HYDRATION-094

TupleHydrator no tendrá dependencia obligatoria de Multitenancy.

## DB-ORM-TUPLE-HYDRATION-095

Entity component IdentityNamespace vendrá del PersistenceContext.

## DB-ORM-TUPLE-HYDRATION-096

Custom component no podrá romper Entity canonicality.

## DB-ORM-TUPLE-HYDRATION-097

Custom component no ejecutará hidden queries.

## DB-ORM-TUPLE-HYDRATION-098

Extension registry será determinista.

## DB-ORM-TUPLE-HYDRATION-099

Extension registry podrá congelarse en producción.

## DB-ORM-TUPLE-HYDRATION-100

Resource limits serán explícitos.

## DB-ORM-TUPLE-HYDRATION-101

Tuple count safety limit no sustituirá pagination.

## DB-ORM-TUPLE-HYDRATION-102

Nested tuple depth será bounded.

## DB-ORM-TUPLE-HYDRATION-103

Tuple hydration deberá ser deterministicamente reproducible dado mismo input y contexto.

## DB-ORM-TUPLE-HYDRATION-104

Current tuple será HydrationScope-scoped.

## DB-ORM-TUPLE-HYDRATION-105

Tuple registry será HydrationScope-scoped.

## DB-ORM-TUPLE-HYDRATION-106

Grouping state será HydrationScope-scoped.

## DB-ORM-TUPLE-HYDRATION-107

No existirá process-global current tuple.

## DB-ORM-TUPLE-HYDRATION-108

FrankenPHP request state estará aislado.

## DB-ORM-TUPLE-HYDRATION-109

RoadRunner request/job state estará aislado.

## DB-ORM-TUPLE-HYDRATION-110

OpenSwoole logical contexts estarán aislados.

## DB-ORM-TUPLE-HYDRATION-111

Tuple hydration no será concurrency-safe por defecto.

## DB-ORM-TUPLE-HYDRATION-112

Slot parallelization no se asumirá segura.

## DB-ORM-TUPLE-HYDRATION-113

V1 hidratará slots en orden determinista.

## DB-ORM-TUPLE-HYDRATION-114

Telemetry distinguirá tuple count de slot count.

## DB-ORM-TUPLE-HYDRATION-115

Telemetry no expondrá tuple values sensibles.

## DB-ORM-TUPLE-HYDRATION-116

Tuple diagnostics podrán mostrar aliases sin mostrar valores sensibles.

## DB-ORM-TUPLE-HYDRATION-117

Entity slot telemetry reutilizará métricas del EntityHydrator cuando sea posible.

## DB-ORM-TUPLE-HYDRATION-118

Tuple hydration no implicará persistence.

## DB-ORM-TUPLE-HYDRATION-119

Tuple hydration no implicará transaction success.

## DB-ORM-TUPLE-HYDRATION-120

Tuple hydration no implicará authorization.

## DB-ORM-TUPLE-HYDRATION-121

Tuple hydration no implicará domain validation.

## DB-ORM-TUPLE-HYDRATION-122

ResultTuple podrá contener managed entities sin ser managed.

## DB-ORM-TUPLE-HYDRATION-123

Modificación de entity component seguirá reglas normales del UoW.

## DB-ORM-TUPLE-HYDRATION-124

Tuple structure no deberá ser mutable por defecto.

## DB-ORM-TUPLE-HYDRATION-125

Tuple API deberá distinguir missing alias de null value.

## DB-ORM-TUPLE-HYDRATION-126

Alias lookup deberá ser O(1) promedio mediante mapa compilado.

## DB-ORM-TUPLE-HYDRATION-127

Tuple logical key no usará PHP serialization como canonical encoding por defecto.

## DB-ORM-TUPLE-HYDRATION-128

Tuple logical key deberá ser typed.

## DB-ORM-TUPLE-HYDRATION-129

Composite tuple key deberá preservar slot boundaries.

## DB-ORM-TUPLE-HYDRATION-130

Value object key semantics deberán ser explícitas.

## DB-ORM-TUPLE-HYDRATION-131

Decimal key semantics deberán preservar precisión.

## DB-ORM-TUPLE-HYDRATION-132

Entity key semantics deberán preservar IdentityNamespace.

## DB-ORM-TUPLE-HYDRATION-133

Nullable slots deberán preservar null en logical key cuando corresponda.

## DB-ORM-TUPLE-HYDRATION-134

Grouping key y dedup key podrán ser diferentes.

## DB-ORM-TUPLE-HYDRATION-135

Grouping state no se reutilizará entre resultados independientes.

## DB-ORM-TUPLE-HYDRATION-136

Tuple dedup registry no será IdentityMap.

## DB-ORM-TUPLE-HYDRATION-137

Tuple dedup registry no sobrevivirá a HydrationScope.

## DB-ORM-TUPLE-HYDRATION-138

Tuple aliases físicos del SQL podrán diferir de aliases lógicos.

## DB-ORM-TUPLE-HYDRATION-139

Logical aliases procederán del HydrationPlan.

## DB-ORM-TUPLE-HYDRATION-140

TupleHydrator no analizará SQL para descubrir slots.

## DB-ORM-TUPLE-HYDRATION-141

TupleHydrator no inferirá type semantics desde valores concretos cuando exista plan.

## DB-ORM-TUPLE-HYDRATION-142

Component converter failure será explícito.

## DB-ORM-TUPLE-HYDRATION-143

Tuple construction failure será explícito.

## DB-ORM-TUPLE-HYDRATION-144

Tuple failure cleanup liberará tuple-local temporary state.

## DB-ORM-TUPLE-HYDRATION-145

Tuple failure no limpiará PersistenceContext completo arbitrariamente.

## DB-ORM-TUPLE-HYDRATION-146

TupleHydrator preservará ordering de component plans.

## DB-ORM-TUPLE-HYDRATION-147

Nested tuple failure propagará contexto del slot padre.

## DB-ORM-TUPLE-HYDRATION-148

Tuple plan validation detectará aliases duplicados antes del hot path.

## DB-ORM-TUPLE-HYDRATION-149

Tuple plan validation detectará unsupported component types.

## DB-ORM-TUPLE-HYDRATION-150

Tuple plan validation detectará impossible nullability contracts.

## DB-ORM-TUPLE-HYDRATION-151

Tuple width podrá ser observable.

## DB-ORM-TUPLE-HYDRATION-152

Tuple nesting depth podrá ser observable.

## DB-ORM-TUPLE-HYDRATION-153

Tuple dedup cost podrá ser diagnosticable.

## DB-ORM-TUPLE-HYDRATION-154

Tuple streaming mode deberá exponer failure semantics.

## DB-ORM-TUPLE-HYDRATION-155

Early stream termination liberará tuple-local state.

## DB-ORM-TUPLE-HYDRATION-156

Resource release failure no transformará retroactivamente tuples ya hydrated en inexistentes.

## DB-ORM-TUPLE-HYDRATION-157

Tuple component results conservarán sus propios outcome semantics.

## DB-ORM-TUPLE-HYDRATION-158

TupleHydrator será una composición de hydrators especializados, no un reemplazo de ellos.

## DB-ORM-TUPLE-HYDRATION-159

Tuple semantics permanecerán independientes de la representación física de rows.

## DB-ORM-TUPLE-HYDRATION-160

VoltStack solo considerará una tuple correctamente hidratada cuando todos los slots requeridos hayan sido materializados conforme a su propio contrato y la estructura compuesta pueda construirse sin perder identidad, tipo, precisión, nullability, ordering o cardinalidad lógica.

---

# 192. Fórmulas fundamentales

## 192.1 Tuple

```text
Tuple
=
Ordered(
    Component₀,
    Component₁,
    ...,
    Componentₙ
)
```

---

# 193. Tuple hydration

```text
HydratedTuple
=
Construct(
    Hydrate(slot₀),
    Hydrate(slot₁),
    ...,
    Hydrate(slotₙ)
)
```

---

# 194. Safe tuple

```text
SafeTuple
=
AllRequiredSlotsPresent
∧
AllComponentHydrationsValid
∧
NullabilitySatisfied
∧
ShapeSatisfied
∧
ConstructionSuccessful
```

---

# 195. Tuple identity

```text
TupleLogicalKey
=
CanonicalKey(
    SelectedTupleComponents
)
```

---

# 196. Entity component identity

```text
EntityTupleComponentKey
=
EntityKey
```

No:

```text
ObjectPropertyHash
```

---

# 197. Tuple equality

```text
TupleEquivalent(a,b)
=
TupleDeduplicationPolicy(a,b)
```

No existe igualdad universal impuesta por el sistema.

---

# 198. Grouping

```text
TupleGroup(row)
=
GroupingKey(
    SelectedResultComponents(row)
)
```

---

# 199. Grouping vs dedup

```text
GroupingKey
≠
DeduplicationKey
```

en general.

---

# 200. Complete tuple

```text
TupleComplete
=
RequiredComponentsHydrated
∧
GroupedComponentsFinalized
```

---

# 201. Safe streaming

```text
SafeTupleYield
=
TupleComplete
∧
NoFutureRowCanExtendTuple
```

---

# 202. Complexity

Para `s` slots:

```text
T(tuple)
≈
Σᵢ T(slotᵢ)
+
O(s)
```

para construction/key operations.

---

# 203. Runtime safety

```text
SafeTupleRuntime
=
ImmutableSharedPlans
∧
ScopedTupleState
∧
ScopedGroupingState
∧
ScopedResultResources
∧
NoCrossRequestTuples
∧
DeterministicCleanup
```

---

# 204. Master Formula

```text
Database Tuple Hydration System
=
Tuple Shapes
+
Ordered Tuple Slots
+
Named Aliases
+
Compiled Tuple Plans
+
Entity Components
+
Scalar Components
+
Projection Components
+
DTO Components
+
Value Object Components
+
Nested Tuples
+
Type-Safe Component Hydration
+
NULL Semantics
+
MISSING Semantics
+
Tuple Construction Strategies
+
Immutable ResultTuple
+
Logical Tuple Keys
+
Tuple Grouping
+
Tuple Deduplication
+
Tuple Ordering
+
Tuple Cardinality Integration
+
Relationship-Aware Tuple Assembly
+
Streaming Tuple Hydration
+
Tuple Completion Detection
+
Cancellation
+
Failure Isolation
+
Resource Governance
+
Tuple Plan Caching
+
Persistent Runtime Isolation
+
Extension Governance
+
Telemetry
+
Diagnostics
+
Testing
```

---

# 205. Master Rule

> **En VoltStack, una tuple será una estructura lógica compuesta cuyos componentes conservan sus propias garantías. Una entidad dentro de una tuple continuará usando `IdentityMap`, un escalar conservará su precisión y nullability, y un DTO o projection conservará su carácter no administrado. La tuple únicamente los compone; nunca podrá utilizar la identidad de uno de ellos para destruir la semántica de los demás.**

---

# 206. Arquitectura resultante

```text
                        Result Row / Group
                               │
                               ▼
                    CompiledTupleHydrationPlan
                               │
                               ▼
                         TupleHydrator
                               │
              ┌────────────────┼───────────────────┐
              │                │                   │
              ▼                ▼                   ▼
          Entity Slot      Scalar Slot      Projection/DTO
              │                │                   │
              ▼                ▼                   ▼
       EntityHydrator    ScalarHydrator      Specialized
              │                │               Hydrators
              ▼                ▼                   ▼
        IdentityMap      Typed PHP Value      PHP Object
              │                │                   │
              └────────────────┼───────────────────┘
                               ▼
                      Slot Validation
                               │
                               ▼
                        Tuple Builder
                               │
                               ▼
                    Logical Tuple Key
                               │
                    ┌──────────┴─────────┐
                    │                    │
                    ▼                    ▼
                Grouping            Deduplication
                    │                    │
                    └──────────┬─────────┘
                               ▼
                         ResultTuple
                               │
                     ┌─────────┴─────────┐
                     ▼                   ▼
                  BUFFERED            STREAMING
```

---

# 207. Siguiente documento

De acuerdo con el bloque de Hydration, después de este documento conviene continuar con:

```text
140_DATABASE_HYDRATION_PLAN_SYSTEM.md
```

Este documento deberá formalizar la capa que compila toda la semántica de hidratación:

```text
HydrationPlan
HydrationPlanDefinition
HydrationShape
Entity plans
Scalar plans
Tuple plans
Projection plans
DTO plans
Relationship plans
Result slot binding
physical aliases
logical aliases
column positions
type converters
constructor plans
field writers
identifier readers
discriminator readers
grouping plans
deduplication plans
cardinality plans
streaming requirements
plan validation
plan normalization
plan optimization
plan compilation
plan fingerprints
metadata generations
cache keys
persistent-runtime sharing
extension contributions
diagnostics
testing
```

Su regla central deberá ser:

> **El `HydrationPlan` será la única descripción ejecutable que conecta la forma física del `Result` con la forma lógica esperada por el ORM; ningún hydrator deberá reconstruir esa semántica analizando SQL, reflection o metadata dispersa durante el hot path.**