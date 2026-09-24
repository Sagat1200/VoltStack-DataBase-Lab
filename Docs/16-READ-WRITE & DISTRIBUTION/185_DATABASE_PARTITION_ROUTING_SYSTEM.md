# 185_DATABASE_PARTITION_ROUTING_SYSTEM.md

# VoltStack Quantum Database
## Database Partition Routing System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 185 — Database Partition Routing System  
**Bloque:** 16 — Read/Write and Distribution  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `184_DATABASE_SHARDING_SYSTEM.md`  
**Siguiente documento:** `186_DATABASE_CACHE_ARCHITECTURE.md`

---

# 1. Propósito

`Database Partition Routing System` define cómo VoltStack determinará **qué shard, partición o conjunto de shards debe ejecutar una operación concreta de base de datos**.

El documento anterior definió:

```text
Data
 ↓
Shard Key
 ↓
Shard Assignment
 ↓
Shard Ownership
```

Este documento añade:

```text
Database Operation
        ↓
Semantic Analysis
        ↓
Partition Constraints
        ↓
Shard Key Extraction
        ↓
Partition Resolution
        ↓
Shard Target Set
        ↓
Distributed Execution Scope
```

La regla central será:

> **Sharding define dónde pertenecen los datos; Partition Routing demuestra dónde debe ejecutarse una operación concreta. VoltStack nunca enviará una operación a un shard basándose únicamente en conveniencia, carga o proximidad cuando no pueda demostrar que dicho shard pertenece al dominio correcto de datos.**

Formalmente:

```text
Route(Operation, Context, ShardMap)
→
PartitionRoutingDecision
```

---

# 2. Posición arquitectónica

El pipeline distribuido completo queda:

```text
Application / ORM
       │
       ▼
Query Model / AST
       │
       ▼
Semantic Query Engine
       │
       ▼
Partition Routing
       │
       ▼
Shard Target Set
       │
       ▼
Distributed Planner
       │
       ▼
Read / Write Routing
       │
       ▼
Replica Eligibility
       │
       ▼
Load Balancing
       │
       ▼
Connection
       │
       ▼
Execution
```

Por tanto:

```text
Partition Routing
```

ocurre conceptualmente **antes** de elegir:

- writer;
- replica;
- endpoint;
- connection;
- connection pool member.

---

# 3. Distinciones fundamentales

```text
Partition Routing
≠
Sharding
≠
Read/Write Routing
≠
Replica Selection
≠
Load Balancing
≠
Connection Resolution
≠
Query Planning
≠
SQL Compilation
≠
Distributed Transaction
```

---

# 4. Sharding vs Partition Routing

Sharding responde:

```text
Where does key K belong?
```

Partition Routing responde:

```text
Which partitions can affect operation O?
```

Ejemplo:

```sql
SELECT *
FROM orders
WHERE customer_id = 42;
```

Sharding puede establecer:

```text
customer_id=42
→ shard-07
```

Partition Routing utiliza esa información para producir:

```text
TargetSet = { shard-07 }
```

---

# 5. Partition Routing vs Read/Write Routing

Después de determinar:

```text
Shard = shard-07
```

todavía falta determinar:

```text
READ
→ replica?

WRITE
→ writer?
```

Por tanto:

```text
Partition Routing
→ chooses shard domain

Read/Write Routing
→ chooses role inside shard domain
```

---

# 6. Partition Routing vs Load Balancing

Si:

```text
shard-07
├── writer
├── replica-a
├── replica-b
└── replica-c
```

Partition Routing termina en:

```text
shard-07
```

Load Balancing puede posteriormente seleccionar:

```text
replica-b
```

---

# 7. Objetivos

El sistema deberá soportar:

1. single-shard routing;
2. multi-shard routing;
3. global routing;
4. unknown routing;
5. shard-key extraction;
6. AST-based inference;
7. semantic graph inference;
8. ORM routing;
9. repository routing;
10. INSERT routing;
11. UPDATE routing;
12. DELETE routing;
13. SELECT routing;
14. equality routing;
15. `IN` routing;
16. range routing;
17. composite shard keys;
18. JOIN routing;
19. aggregate routing;
20. pagination routing;
21. explicit shard hints;
22. transaction affinity;
23. UnitOfWork routing;
24. scatter-gather planning;
25. fan-out governance;
26. stale ShardMap handling;
27. ownership-transition awareness;
28. routing diagnostics;
29. telemetry;
30. persistent-runtime isolation.

---

# 8. No objetivos

Partition Routing no deberá:

- generar SQL;
- ejecutar queries;
- abrir conexiones;
- elegir replicas;
- medir replica lag;
- hacer load balancing;
- implementar sharding;
- mover shards;
- ejecutar migrations;
- implementar distributed transactions;
- fusionar resultados;
- hidratar entidades;
- autorizar usuarios;
- convertir unknown routing en global routing silenciosamente.

---

# 9. Arquitectura conceptual

```text
             DATABASE OPERATION
                     │
                     ▼
            PartitionRouteRequest
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
     Query Semantics       Routing Context
          │                     │
          └──────────┬──────────┘
                     ▼
         Shard Constraint Analyzer
                     │
                     ▼
          Partition Constraint
                     │
                     ▼
          Partition Resolver
                     │
                     ▼
          Shard Map / Ownership
                     │
                     ▼
         PartitionRoutingDecision
```

---

# 10. PartitionRouteRequest

Objeto canónico:

```php
final readonly class PartitionRouteRequest
{
    public function __construct(
        public DatabaseOperation $operation,
        public QuerySemanticModel $semantics,
        public PartitionRoutingContext $context,
        public ShardMapSnapshot $shardMap,
    ) {}
}
```

---

# 11. PartitionRoutingDecision

```php
final readonly class PartitionRoutingDecision
{
    public function __construct(
        public PartitionScope $scope,
        public ShardTargetSet $targets,
        public RoutingConfidence $confidence,
        public RoutingReason $reason,
        public ShardMapGeneration $generation,
    ) {}
}
```

---

# 12. PartitionScope

Modelo fundamental:

```php
enum PartitionScope
{
    case SINGLE;
    case MULTIPLE;
    case GLOBAL;
    case UNKNOWN;
    case NONE;
}
```

---

# 13. SINGLE

```text
TargetSet = { shard-07 }
```

La operación puede ejecutarse sobre un único shard.

---

# 14. MULTIPLE

```text
TargetSet =
{
    shard-02,
    shard-07,
    shard-09
}
```

La operación requiere varios shards conocidos.

---

# 15. GLOBAL

La semántica de la operación requiere todos los shards elegibles.

```text
TargetSet = ALL_ACTIVE_SHARDS
```

---

# 16. UNKNOWN

VoltStack no puede demostrar correctamente los shards requeridos.

```text
UNKNOWN
≠
GLOBAL
```

Esta distinción será obligatoria.

---

# 17. NONE

La operación puede resolverse sin consultar ningún shard.

Ejemplo:

```text
WHERE shard_key IN ()
```

normalizado semánticamente a:

```text
FALSE
```

puede producir:

```text
PartitionScope::NONE
```

---

# 18. Regla UNKNOWN

Nunca:

```text
UNKNOWN
→ ALL SHARDS
```

automáticamente.

Deberá existir una policy explícita.

---

# 19. Routing confidence

```php
enum RoutingConfidence
{
    case EXACT;
    case PROVEN;
    case CONSERVATIVE;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 20. EXACT

La shard key completa está disponible.

```text
customer_id = 42
```

---

# 21. PROVEN

El Semantic Engine demuestra un conjunto equivalente de constraints.

---

# 22. CONSERVATIVE

El sistema conoce un superconjunto seguro de shards.

Ejemplo:

```text
actual required:
{A}

safe derived set:
{A,B}
```

Puede ser correcto pero menos eficiente.

---

# 23. PARTIAL

Solo se conoce una parte de la información requerida.

No deberá confundirse con safe routing.

---

# 24. UNKNOWN

No existe evidencia suficiente.

---

# 25. Routing correctness

La regla de seguridad será:

```text
RequiredShards(O)
⊆
RoutedShards(O)
```

para routing conservador permitido.

Nunca:

```text
RequiredShards(O)
⊄
RoutedShards(O)
```

porque implicaría resultados incompletos o writes incorrectos.

---

# 26. Minimal routing

Idealmente:

```text
RequiredShards(O)
=
RoutedShards(O)
```

---

# 27. ShardConstraint

El análisis produce:

```php
interface ShardConstraint
{
}
```

Implementaciones:

```text
ExactShardKeyConstraint
ShardKeySetConstraint
ShardKeyRangeConstraint
CompositeShardKeyConstraint
AllPartitionsConstraint
NoPartitionsConstraint
UnknownPartitionConstraint
```

---

# 28. Exact shard-key constraint

Ejemplo:

```sql
WHERE customer_id = 42
```

produce:

```text
ExactShardKeyConstraint(42)
```

---

# 29. Set constraint

```sql
WHERE customer_id IN (10, 20, 30)
```

produce:

```text
ShardKeySetConstraint(
    10,
    20,
    30
)
```

---

# 30. Deduplicación

Si:

```text
10 → shard-A
20 → shard-B
30 → shard-A
```

el target set será:

```text
{ shard-A, shard-B }
```

---

# 31. Range constraint

Para range sharding:

```sql
WHERE customer_id BETWEEN 1000 AND 2500
```

podrá mapearse a:

```text
ShardKeyRangeConstraint
```

y posteriormente:

```text
{ shard-2, shard-3 }
```

---

# 32. Hash sharding y ranges

Con hash sharding:

```text
customer_id BETWEEN 1000 AND 2500
```

normalmente no permite reducir el target set por range.

Esto puede resultar en:

```text
GLOBAL
```

según strategy.

---

# 33. ShardConstraintAnalyzer

```php
interface ShardConstraintAnalyzer
{
    public function analyze(
        QuerySemanticModel $query,
        ShardKeyDefinition $key,
    ): ShardConstraintAnalysis;
}
```

---

# 34. AST integration

Query:

```php
Order::query()
    ->where('customer_id', 42);
```

produce conceptualmente:

```text
Query Builder
    ↓
Query AST
    ↓
Semantic Graph
    ↓
customer_id = :p1
    ↓
Shard Constraint
```

---

# 35. No string inspection

Partition Routing no deberá analizar:

```text
"SELECT ... WHERE customer_id = ?"
```

mediante regex como mecanismo principal.

---

# 36. Semantic routing

La inferencia deberá realizarse sobre:

- Query AST;
- Semantic Query Graph;
- resolved symbols;
- normalized predicates;
- typed parameters.

---

# 37. Parameter resolution

Query:

```text
customer_id = :customer
```

puede producir routing exacto cuando:

```text
:customer = 42
```

esté disponible antes de execution planning.

---

# 38. Parameter type

Shard key parameter deberá pasar por:

```text
Query Parameter
    ↓
Type Validation
    ↓
Shard Key Normalization
    ↓
CanonicalShardKey
```

---

# 39. Shard key coercion

No deberá utilizar coerción insegura.

Ejemplo:

```text
"42abc"
```

no deberá convertirse silenciosamente en:

```text
42
```

---

# 40. Constant folding

Query:

```text
customer_id = 40 + 2
```

podrá convertirse a:

```text
customer_id = 42
```

si el Semantic/Normalization Engine demuestra equivalencia segura.

---

# 41. Contradictory predicates

```sql
WHERE customer_id = 10
AND customer_id = 20
```

puede normalizarse a:

```text
NoPartitionsConstraint
```

si los tipos/semántica garantizan contradicción.

---

# 42. OR predicates

```sql
WHERE customer_id = 10
OR customer_id = 20
```

produce:

```text
ShardKeySetConstraint(10,20)
```

---

# 43. Mixed OR

```sql
WHERE customer_id = 10
OR status = 'pending'
```

La segunda rama no restringe shard.

Por tanto, normalmente:

```text
GLOBAL
```

no:

```text
shard(customer_id=10)
```

---

# 44. AND predicates

```sql
WHERE customer_id = 10
AND status = 'pending'
```

puede conservar:

```text
ExactShardKeyConstraint(10)
```

---

# 45. Predicate algebra

Conceptualmente:

```text
Route(A AND B)
=
Intersect(Route(A), Route(B))
```

cuando ambas expresiones representan constraints compatibles.

Mientras:

```text
Route(A OR B)
=
Union(Route(A), Route(B))
```

con tratamiento conservador de `UNKNOWN`.

---

# 46. UNKNOWN in AND

Ejemplo:

```text
customer_id = 42
AND complex_function(...)
```

Si la segunda condición no afecta shard ownership:

```text
SINGLE shard-07
```

puede seguir siendo válido.

---

# 47. UNKNOWN in OR

```text
customer_id = 42
OR unknown_predicate
```

no permite single-shard routing.

---

# 48. Composite shard keys

Supongamos:

```text
ShardKey =
(region, customer_id)
```

Query:

```sql
WHERE region = 'mx'
AND customer_id = 42
```

permite:

```text
ExactCompositeShardKey
```

---

# 49. Partial composite key

```sql
WHERE region = 'mx'
```

puede o no reducir shards dependiendo de strategy.

---

# 50. Composite routing policy

El strategy deberá declarar qué prefijos/componentes permiten pruning.

---

# 51. Composite key ordering

```text
(region, customer)
```

no deberá asumirse equivalente a:

```text
(customer, region)
```

---

# 52. INSERT routing

Para:

```php
Order::create([
    'customer_id' => 42,
    ...
]);
```

el sistema deberá extraer:

```text
ShardKey = 42
```

antes de execution.

---

# 53. INSERT invariant

Todo INSERT sobre una entidad sharded deberá tener un target exacto salvo estrategia explícita distinta.

Normalmente:

```text
INSERT
→ SINGLE
```

---

# 54. INSERT UNKNOWN

Si no puede determinarse shard:

```text
INSERT
→ reject
```

Nunca scatter.

---

# 55. INSERT multi-shard batch

Un batch:

```text
[
 customer 10,
 customer 20,
 customer 30
]
```

puede dividirse:

```text
Shard A
├── row 10
└── row 30

Shard B
└── row 20
```

---

# 56. Batch INSERT ≠ distributed atomicity

La división anterior no hace atomic el batch global.

---

# 57. UPDATE routing

```sql
UPDATE orders
SET status = 'paid'
WHERE customer_id = 42;
```

produce:

```text
SINGLE
```

---

# 58. UPDATE without shard key

```sql
UPDATE orders
SET status = 'expired'
WHERE expires_at < NOW();
```

puede requerir:

```text
GLOBAL
```

pero solo si global writes están permitidos explícitamente.

---

# 59. Global write safety

Por default:

```text
GLOBAL UPDATE
GLOBAL DELETE
```

deberán requerir policy más estricta que global reads.

---

# 60. Dangerous fan-out writes

Nunca deberán ejecutarse como efecto secundario de:

```text
UNKNOWN routing
```

---

# 61. Shard-key UPDATE

```sql
UPDATE orders
SET customer_id = 100
WHERE customer_id = 42;
```

puede cambiar ownership.

Esto no será un ordinary UPDATE.

---

# 62. Shard-key mutation

Deberá ser rechazada o convertida en una operación explícita de relocation.

---

# 63. DELETE routing

```sql
DELETE FROM orders
WHERE customer_id = 42;
```

produce:

```text
SINGLE
```

---

# 64. DELETE by primary key

Si:

```text
PRIMARY KEY = order_id
SHARD KEY = customer_id
```

entonces:

```sql
DELETE WHERE order_id = 123
```

no implica que el shard pueda derivarse.

---

# 65. Global identifier directory

Una extensión futura podría resolver:

```text
order_id
→ shard
```

mediante directory/index global.

Pero no se asumirá en core.

---

# 66. SELECT routing

SELECT puede producir:

```text
NONE
SINGLE
MULTIPLE
GLOBAL
UNKNOWN
```

---

# 67. SELECT without shard predicate

Ejemplo:

```sql
SELECT *
FROM products
WHERE active = true;
```

Si `products` está sharded por `category_id`:

```text
GLOBAL
```

puede ser correcto.

---

# 68. Global read policy

Podrá existir:

```php
enum GlobalReadPolicy
{
    case ALLOW;
    case ALLOW_BOUNDED;
    case WARN;
    case FORBID;
}
```

---

# 69. Multi-shard read

```text
MULTIPLE
```

deberá producir un conjunto conocido de shards.

---

# 70. Scatter-gather

```text
Query
  │
  ├── shard-A
  ├── shard-B
  ├── shard-C
  └── shard-D
       │
       ▼
Distributed Result Merge
```

Partition Routing determina el fan-out.

No ejecuta ni fusiona resultados.

---

# 71. Scatter ≠ gather

```text
Scatter
=
dispatch to partitions

Gather
=
combine results
```

Son responsabilidades diferentes.

---

# 72. FanOutPlan

Conceptualmente:

```php
final readonly class FanOutPlan
{
    public function __construct(
        public ShardTargetSet $targets,
        public FanOutExecutionPolicy $policy,
        public FanOutBudget $budget,
    ) {}
}
```

---

# 73. Fan-out budget

Debe poder limitar:

- máximo de shards;
- concurrencia;
- tiempo;
- filas;
- memoria;
- bytes;
- partial failures.

---

# 74. Max shard fan-out

Ejemplo:

```text
maxShardsPerOperation = 32
```

Una query que requiera 500 shards puede rechazarse.

---

# 75. Global query budget

`GLOBAL` no significará:

```text
unlimited parallel execution
```

---

# 76. Bounded concurrency

Ejemplo:

```text
100 shards
fanOutConcurrency = 8
```

---

# 77. Failure semantics

Si:

```text
Shard A → success
Shard B → success
Shard C → timeout
Shard D → success
```

el resultado global no deberá presentarse como completo.

---

# 78. Partial result

Deberá existir una semántica explícita si se permite:

```text
PARTIAL_RESULT
```

---

# 79. Default correctness

Para queries normales:

```text
one required shard failed
→ operation failed
```

---

# 80. JOIN routing

Supongamos:

```text
orders.customer_id
customers.id
```

y ambas entidades están co-sharded por customer.

El Semantic Engine puede demostrar:

```text
co-located join
```

---

# 81. Co-located join

```text
JOIN
→ SINGLE shard
```

si existe una shard-key constraint exacta.

---

# 82. Distributed JOIN

Si las tablas no están co-located:

```text
JOIN
→ distributed operation
```

no deberá ejecutarse como local JOIN accidentalmente.

---

# 83. Cross-shard JOIN

Core deberá:

- rechazarlo;
- o delegarlo a una futura distributed query engine;

pero nunca fingir que un SQL local puede resolverlo.

---

# 84. Shard co-location metadata

Podrá existir:

```php
final readonly class ShardColocationGroup
{
    public function __construct(
        public ShardColocationGroupId $id,
        public array $members,
    ) {}
}
```

---

# 85. Co-location proof

Dos entidades no son co-located solo porque sus campos tengan el mismo nombre.

---

# 86. Required proof

Debe comprobarse:

- same shard map;
- compatible shard-key semantics;
- compatible assignment strategy;
- compatible ownership domain.

---

# 87. Aggregate routing

```sql
SELECT COUNT(*)
FROM orders;
```

sobre sharded table puede requerir:

```text
GLOBAL
```

---

# 88. Aggregate decomposition

Algunas agregaciones permiten:

```text
local aggregate
+
merge
```

Ejemplos:

```text
COUNT
SUM
MIN
MAX
```

bajo semántica compatible.

---

# 89. AVG

Puede transformarse:

```text
AVG
→ SUM + COUNT
→ global merge
```

pero esta optimización pertenece al Distributed Planner.

---

# 90. Aggregate correctness

Partition Routing solo determina:

```text
required partitions
```

No define cómo fusionar agregados.

---

# 91. GROUP BY

Puede requerir:

```text
scatter
+
partial group
+
global merge
```

si grouping key no coincide con shard ownership.

---

# 92. DISTINCT

Igualmente puede requerir deduplicación global.

---

# 93. ORDER BY

Cada shard puede ordenar localmente.

Eso no produce orden global.

---

# 94. Global ordering

Requiere merge ordenado.

---

# 95. LIMIT

Query:

```sql
ORDER BY created_at DESC
LIMIT 20
```

sobre múltiples shards no puede ejecutarse correctamente tomando 20 filas de un shard arbitrario.

---

# 96. Distributed top-K

Puede requerir:

```text
local top-K
+
global merge
```

---

# 97. OFFSET pagination

```text
OFFSET 100000
```

sobre múltiples shards puede ser extremadamente costoso.

---

# 98. Cursor pagination

Deberá preferirse cuando sea posible.

---

# 99. Distributed cursor

Puede requerir estado:

```text
Shard A cursor
Shard B cursor
Shard C cursor
```

más merge position.

---

# 100. Pagination ≠ routing

Pagination semantics serán responsabilidad de query/distributed execution layers.

Partition Routing solo determina shards.

---

# 101. Subqueries

Subquery podrá tener su propio routing analysis.

---

# 102. Correlated subquery

Puede requerir:

```text
outer shard context
→ inner routing constraints
```

---

# 103. CTE

CTEs deberán analizarse semánticamente.

No se asumirá que todo CTE es global.

---

# 104. UNION

Cada branch puede tener target set independiente.

Ejemplo:

```text
Branch A → shard-1
Branch B → shard-3
```

Target:

```text
{shard-1, shard-3}
```

---

# 105. Set operation routing

```text
UNION
INTERSECT
EXCEPT
```

deberán preservar sus semánticas al combinar target sets.

---

# 106. Repository routing

Repository:

```php
$orders->findForCustomer(
    customerId: 42,
    orderId: 100
);
```

puede aportar directamente:

```text
ShardKeyConstraint(42)
```

---

# 107. Repository hint

Repository no deberá devolver directamente una Connection.

Debe aportar:

```text
routing semantics
```

---

# 108. Model API routing

```php
Order::onShardKey($customerId)
    ->whereKey($orderId)
    ->first();
```

puede ser una API explícita.

---

# 109. Explicit shard

```php
Order::onShard('shard-07')
```

es distinto de:

```php
Order::onShardKey(42)
```

---

# 110. Prefer shard key

Cuando sea posible deberá preferirse:

```text
logical shard key
```

sobre:

```text
physical/logical ShardId
```

porque el map puede cambiar.

---

# 111. Shard hint

```php
$query->withShardHint($shardId);
```

será un hint.

---

# 112. Hint ≠ override

Por default:

```text
Hint
→ validate
→ use if compatible
```

---

# 113. Forced shard

Una API administrativa podría permitir:

```text
FORCE_SHARD
```

pero deberá ser explícita y auditable.

---

# 114. Forced routing risks

Puede:

- omitir datos;
- leer stale location;
- violar transaction domain;
- afectar tenant isolation.

---

# 115. RoutingContext

```php
final readonly class PartitionRoutingContext
{
    public function __construct(
        public LogicalDatabaseId $database,
        public OperationType $operation,
        public ?TransactionContextView $transaction,
        public ?ShardHint $hint,
        public RoutingPolicy $policy,
        public RuntimeScopeId $scope,
    ) {}
}
```

---

# 116. Context exclusions

No deberá contener:

- EntityManager completo;
- service container;
- HTTP request;
- credentials;
- mutable global state.

---

# 117. Transaction routing

Si existe una transacción activa:

```text
TransactionContext
→ Pinned Shard
```

será una constraint dominante.

---

# 118. Transaction first operation

Si transaction todavía no tiene shard asignado, una operación exacta puede fijarlo.

Conceptualmente:

```text
UNBOUND TRANSACTION
      ↓
first shard-bound operation
      ↓
SHARD AFFINITY
```

solo si el transaction architecture permite deferred binding.

---

# 119. Bound transaction

Después:

```text
Shard(T) = shard-A
```

una operación que requiera:

```text
shard-B
```

deberá fallar.

---

# 120. Transaction mismatch

```text
RequiredShard(O)
≠
TransactionShard(T)
```

produce:

```text
TransactionShardMismatchException
```

---

# 121. Multi-shard inside transaction

```text
MULTIPLE
GLOBAL
```

serán rechazados en una ordinary single-resource transaction.

---

# 122. Savepoints

Savepoint no crea un nuevo shard domain.

---

# 123. Nested transaction

Una nested joined transaction hereda shard affinity.

---

# 124. REQUIRES_NEW

Una transacción independiente podrá tener otra affinity si utiliza un nuevo physical transaction/context conforme a Transaction System.

---

# 125. Retry

Un transaction retry deberá resolver nuevamente routing contra topology/map policy apropiada.

---

# 126. Retry and ownership movement

Si entre intentos cambia:

```text
ShardMapGeneration
```

el nuevo intento puede dirigirse al nuevo owner.

---

# 127. Same attempt

Dentro del mismo transaction attempt no deberá cambiarse silenciosamente de shard.

---

# 128. UnitOfWork

Cada persistence operation deberá tener:

```text
PersistenceOperation
→ ShardAssignment
```

antes de execution.

---

# 129. Persistence grouping

```text
UnitOfWork
    ↓
Persistence Planner
    ↓
Group by Execution Domain
```

---

# 130. Single-domain flush

Caso normal:

```text
all operations
→ shard-A
```

---

# 131. Multi-domain flush

Si:

```text
entity 1 → shard-A
entity 2 → shard-B
```

se detectará:

```text
CrossShardPersistencePlan
```

---

# 132. Default multi-domain policy

Si atomicidad es requerida:

```text
reject before execution
```

---

# 133. Best-effort persistence

No deberá habilitarse implícitamente.

---

# 134. IdentityMap

Partition routing no sustituye IdentityMap.

Pero EntityKey deberá conservar el domain necesario para evitar colisiones.

---

# 135. Hydration

Hydrator recibe resultados de un shard ya determinado.

No decide routing.

---

# 136. Relationship loading

To-one/to-many loading deberá derivar shard context de:

- owner;
- relationship metadata;
- shard-key mapping.

---

# 137. Co-located relationship

Puede permanecer:

```text
SINGLE
```

---

# 138. Cross-shard relationship

Debe generar una nueva routing operation explícita.

---

# 139. Batch relationship loading

Owners deberán agruparse por:

```text
RelationshipId
+
ShardId
+
ExecutionDomain
```

---

# 140. Eager loading

Fetch planner deberá considerar shard boundaries.

---

# 141. Eager JOIN

Solo podrá utilizarse cuando las relaciones estén co-located y SQL-local.

---

# 142. Cross-shard eager loading

Puede transformarse en:

```text
root query
+
grouped relation fetches
```

no en JOIN físico único.

---

# 143. Lazy loading

Deberá conservar:

```text
ShardContext
```

sin depender de global mutable state.

---

# 144. ShardMap snapshot

Cada routing operation utilizará una generación concreta.

---

# 145. Map consistency

```text
RoutingDecision
→ ShardMapGeneration
```

deberá conservarse en execution plan.

---

# 146. Stale map

Antes de dispatch, el sistema podrá detectar que la generación quedó obsoleta.

---

# 147. Stale-read policy

Un read puede permitir generation antigua si ownership transition garantiza compatibilidad.

---

# 148. Stale-write policy

Writes deberán utilizar reglas más estrictas.

---

# 149. Ownership transition

Durante:

```text
COPYING
CUTTING_OVER
MOVED
```

Partition Routing deberá consultar ownership semantics.

---

# 150. Routing target state

Un target podrá tener:

```text
PRIMARY_OWNER
SOURCE_OWNER
TARGET_OWNER
REDIRECT_OWNER
```

según transition protocol.

---

# 151. No arbitrary dual-write

Partition Routing no convertirá un write normal en dual-write automáticamente.

---

# 152. Redirect

Si execution recibe:

```text
MOVED_TO shard-B
```

podrá solicitar una nueva routing resolution.

---

# 153. Redirect validation

Debe validar:

- target;
- generation;
- ownership epoch;
- redirect count;
- transaction constraints.

---

# 154. Redirect inside transaction

Normalmente deberá rechazarse si implicaría cambiar physical shard de una transacción ya activa.

---

# 155. Routing retry

Routing retry será distinto de:

```text
query retry
transaction retry
failover retry
```

---

# 156. Routing cache

Podrá cachearse:

```text
CanonicalShardKey
+
ShardMapGeneration
→
ShardAssignment
```

---

# 157. Cache generation

Nunca:

```text
key → shard forever
```

sin generation/epoch.

---

# 158. Routing plan cache

También podrán cachearse planes semánticos:

```text
QueryFingerprint
→
ShardConstraintExtractionPlan
```

---

# 159. Plan cache ≠ routing result cache

El plan puede decir:

```text
extract parameter :customer_id
```

sin cachear:

```text
customer 42 → shard 7
```

---

# 160. Prepared statements

Ejemplo:

```sql
WHERE customer_id = ?
```

puede reutilizar:

```text
ShardConstraintExtractionPlan
```

para distintos parámetros.

---

# 161. Query fingerprint

Routing plan cache deberá incorporar:

- semantic query shape;
- entity metadata generation;
- shard metadata generation;
- relevant strategy version.

---

# 162. Map changes

No necesariamente invalidan todo semantic extraction plan.

Pero sí pueden cambiar:

```text
key → shard
```

---

# 163. RoutingPolicy

```php
final readonly class RoutingPolicy
{
    public function __construct(
        public UnknownRoutingPolicy $unknown,
        public GlobalReadPolicy $globalReads,
        public GlobalWritePolicy $globalWrites,
        public FanOutBudget $fanOut,
    ) {}
}
```

---

# 164. UnknownRoutingPolicy

```php
enum UnknownRoutingPolicy
{
    case REJECT;
    case EXPLICIT_GLOBAL_ONLY;
    case ALLOW_GLOBAL_READ;
}
```

---

# 165. Safe default

```text
UNKNOWN
→ REJECT
```

---

# 166. GlobalWritePolicy

```php
enum GlobalWritePolicy
{
    case FORBID;
    case EXPLICIT_ONLY;
    case ALLOW_BOUNDED;
}
```

---

# 167. Safe default global writes

```text
FORBID
```

o `EXPLICIT_ONLY`, según API final.

---

# 168. Explicit global API

Conceptualmente:

```php
DB::acrossAllShards()
    ->table('sessions')
    ->where('expired_at', '<', $now)
    ->delete();
```

La intención será visible.

---

# 169. Global mutation guard

Podrá requerir:

- explicit API;
- fan-out budget;
- no active transaction;
- audit event;
- safety policy.

---

# 170. RoutingReason

```php
enum RoutingReason
{
    case EXACT_SHARD_KEY;
    case SHARD_KEY_SET;
    case SHARD_KEY_RANGE;
    case TRANSACTION_AFFINITY;
    case ENTITY_AFFINITY;
    case EXPLICIT_SHARD;
    case DIRECTORY_LOOKUP;
    case GLOBAL_OPERATION;
    case CONSERVATIVE_FANOUT;
    case UNKNOWN;
}
```

---

# 171. Explainability

Toda decisión deberá poder explicar:

```text
Why this shard?
```

---

# 172. Diagnostics API

Conceptualmente:

```php
DB::routing()->explain(
    Order::query()
        ->where('customer_id', 42)
);
```

---

# 173. Diagnostic example

```text
PARTITION ROUTING EXPLAIN

Logical Database:
    main

Entity:
    Order

Shard Map:
    customer-data

Generation:
    48

Shard Key:
    customer_id

Predicate:
    customer_id = :p1

Parameter:
    [redacted]

Constraint:
    EXACT

Partition Scope:
    SINGLE

Target:
    shard-c

Reason:
    EXACT_SHARD_KEY

Confidence:
    EXACT

Transaction:
    none

Fan-Out:
    1
```

---

# 174. Multi-shard explain

```text
Partition Scope:
    MULTIPLE

Targets:
    shard-a
    shard-c
    shard-d

Reason:
    SHARD_KEY_SET

Input Keys:
    12

Unique Shards:
    3
```

---

# 175. Global explain

```text
Partition Scope:
    GLOBAL

Reason:
    NO_SHARD_RESTRICTING_PREDICATE

Eligible Shards:
    64

Policy:
    ALLOW_BOUNDED

Fan-Out Limit:
    64
```

---

# 176. UNKNOWN diagnostic

```text
Partition Scope:
    UNKNOWN

Reason:
    INSUFFICIENT_SHARD_KEY_INFORMATION

Action:
    REJECT

Suggestion:
    Add a shard-key predicate or use an explicit global operation.
```

---

# 177. Developer experience

Error messages deberán ser accionables.

No:

```text
Routing failed.
```

Preferir:

```text
Cannot route Order query to a shard because the entity is sharded by
customer_id and the query does not constrain customer_id.

Use a customer_id predicate, provide a shard-key routing context,
or explicitly opt into a bounded global query.
```

---

# 178. Security

Partition Routing deberá asumir que:

```text
ShardId
ShardHint
ShardKey
```

pueden provenir indirectamente de input no confiable.

---

# 179. Routing ≠ authorization

Encontrar el shard correcto no concede permiso para leerlo.

---

# 180. Tenant isolation

Cuando Multitenancy esté instalado:

```text
TenantContext
```

podrá aportar constraints adicionales.

Pero:

```text
Database core
```

no dependerá del paquete Multitenancy.

---

# 181. Tenant constraint intersection

Conceptualmente:

```text
QueryShardConstraint
∩
TenantShardConstraint
```

deberá ser compatible.

---

# 182. Tenant mismatch

Si query intenta:

```text
Shard A
```

pero tenant pertenece a:

```text
Shard B
```

deberá fallar cerrado.

---

# 183. Explicit shard security

```php
DB::onShard('shard-admin')
```

no deberá saltarse tenant/domain restrictions.

---

# 184. Raw SQL

Raw SQL sobre sharded entities deberá requerir:

- explicit shard;
- shard key context;
- o explicit global policy.

---

# 185. No unsafe SQL inference

No se intentará interpretar arbitrariamente vendor SQL para garantizar routing correctness.

---

# 186. Stored procedures

Una stored procedure sharded deberá declarar routing metadata.

Ejemplo:

```text
Procedure:
    close_customer_orders

Routing:
    shard-key argument #1
```

---

# 187. Custom query extensions

Deberán poder declarar:

```text
ShardConstraintProvider
```

si introducen nuevas expresiones semánticas.

---

# 188. Extension contract

```php
interface ShardConstraintProvider
{
    public function constraints(
        SemanticExpression $expression,
        ShardRoutingMetadata $metadata,
    ): ShardConstraintResult;
}
```

---

# 189. Unknown custom expressions

Sin provider:

```text
UNKNOWN
```

cuando la expresión pueda afectar shard pruning.

---

# 190. Optimizer integration

Partition pruning podrá ejecutarse antes del physical query planning.

---

# 191. Planner integration

Logical plan:

```text
LogicalQueryPlan
      ↓
Partition Routing
      ↓
PartitionedLogicalPlan
      ↓
Physical Planning
```

---

# 192. Alternative integration

Para algunas operaciones:

```text
Semantic Graph
      ↓
Routing Analysis
      ↓
Logical Planner
```

podrá ser más eficiente.

La implementación final deberá preservar la misma semántica.

---

# 193. PartitionedQueryPlan

```php
final readonly class PartitionedQueryPlan
{
    public function __construct(
        public LogicalQueryPlan $logicalPlan,
        public PartitionRoutingDecision $routing,
        public ShardMapGeneration $generation,
    ) {}
}
```

---

# 194. Single-shard physical plan

```text
Partitioned Plan
      ↓
Shard A
      ↓
SQL Compiler
      ↓
Executor
```

---

# 195. Multi-shard physical plan

```text
Partitioned Plan
      │
      ├── Shard A Plan
      ├── Shard B Plan
      └── Shard C Plan
              │
              ▼
      Distributed Merge Plan
```

---

# 196. SQL compilation

Una misma semantic query puede compilarse varias veces si shards utilizan plataformas/dialects distintos.

---

# 197. Preferred homogeneous topology

VoltStack podrá optimizar cuando shards sean platform-compatible, pero no deberá asumirlo universalmente.

---

# 198. Capability checks

Multi-shard planner deberá considerar capabilities por target.

---

# 199. Lowest common denominator

No deberá degradar silenciosamente semántica a la capacidad más baja.

---

# 200. Unsupported target

Si un shard no puede ejecutar la semántica requerida:

```text
global operation
→ fail
```

salvo estrategia de emulación explícita.

---

# 201. Cancellation

Multi-shard operation deberá propagar cancellation a operaciones en vuelo cuando sea posible.

---

# 202. Timeout

Podrá existir:

```text
global deadline
```

del cual se derivan budgets por shard.

---

# 203. Deadline ≠ per-shard timeout × shard count

La operación global deberá respetar un deadline total.

---

# 204. Resource governance

Partition Routing deberá calcular un costo aproximado de fan-out.

---

# 205. RoutingCost

```php
final readonly class RoutingCost
{
    public function __construct(
        public int $shardCount,
        public RoutingCostClass $class,
    ) {}
}
```

---

# 206. Routing cost classes

```text
LOCAL
SMALL_FANOUT
MEDIUM_FANOUT
LARGE_FANOUT
GLOBAL
```

---

# 207. Cost policy

Una aplicación podrá prohibir:

```text
LARGE_FANOUT
GLOBAL
```

en endpoints interactivos.

---

# 208. Background jobs

Podrán permitir budgets mayores.

---

# 209. Request context

HTTP request no deberá estar embebido en routing core.

Solo podrá proporcionar una policy mediante integration layer.

---

# 210. Persistent runtimes

FrankenPHP, RoadRunner y OpenSwoole requieren:

```text
immutable shared routing plans
+
scoped mutable routing state
```

---

# 211. Shared state

Puede compartirse:

- immutable ShardMaps;
- compiled shard metadata;
- semantic extraction plans;
- immutable routing policies;
- thread/coroutine-safe caches.

---

# 212. Scoped state

Debe permanecer por operación:

- current transaction affinity;
- redirect history;
- temporary shard hints;
- fan-out progress;
- cancellation;
- deadlines;
- diagnostic breadcrumbs.

---

# 213. Forbidden global state

Nunca:

```php
static $currentShard;
```

---

# 214. Worker reset

Al finalizar request/job:

```text
RoutingContext
RedirectHistory
FanOutState
TemporaryHints
```

deberán desaparecer.

---

# 215. Coroutine isolation

En OpenSwoole:

```text
Coroutine A → shard-A
Coroutine B → shard-B
```

no podrán contaminarse.

---

# 216. Fiber isolation

Misma regla para Fibers.

---

# 217. Routing state leak

Deberá detectarse como lifecycle violation.

---

# 218. Telemetry

Eventos sugeridos:

```text
PartitionRoutingStarted
PartitionRoutingCompleted
PartitionRoutingRejected
SingleShardRouteSelected
MultiShardRouteSelected
GlobalRouteSelected
UnknownRouteDetected
FanOutBudgetExceeded
TransactionShardMismatchDetected
ShardMapStaleDuringRouting
ShardRedirectObserved
```

---

# 219. Metrics

Ejemplos:

```text
db.routing.operations
db.routing.single_shard
db.routing.multi_shard
db.routing.global
db.routing.unknown
db.routing.fanout_size
db.routing.duration
db.routing.rejections
db.routing.redirects
```

---

# 220. Cardinality

No utilizar `ShardId` como metric label ilimitado por default.

---

# 221. Tracing

Span:

```text
database.partition.route
```

atributos bounded:

```text
scope=SINGLE
target_count=1
reason=EXACT_SHARD_KEY
map_generation=48
```

---

# 222. Sensitive values

No registrar shard keys raw por default.

---

# 223. Query telemetry

Podrá correlacionarse:

```text
QueryOperationId
→ PartitionRoutingDecision
→ ExecutionPlan
→ PhysicalQueries
```

---

# 224. Error hierarchy

```text
DatabasePartitionRoutingException
├── PartitionRoutingUnknownException
├── PartitionRoutingRejectedException
├── PartitionRoutingAmbiguousException
├── PartitionRoutingNoTargetException
├── PartitionRoutingFanOutException
│   ├── FanOutBudgetExceededException
│   └── GlobalRoutingForbiddenException
│
├── ShardConstraintException
│   ├── ShardConstraintExtractionException
│   ├── ShardConstraintConflictException
│   └── IncompleteCompositeShardKeyException
│
├── TransactionShardMismatchException
├── CrossShardTransactionRoutingException
├── ShardRoutingHintMismatchException
├── StaleShardRoutingException
├── ShardRedirectException
├── ShardRedirectLoopException
├── ShardOwnershipRoutingException
└── PartitionRoutingInvariantViolationException
```

---

# 225. Directory structure

```text
src/Quantum/Database/Distribution/PartitionRouting/
│
├── PartitionRouter.php
├── PartitionRouteRequest.php
├── PartitionRoutingDecision.php
├── PartitionRoutingContext.php
├── PartitionScope.php
├── RoutingConfidence.php
├── RoutingReason.php
├── RoutingPolicy.php
├── RoutingCost.php
│
├── Constraint/
│   ├── ShardConstraint.php
│   ├── ExactShardKeyConstraint.php
│   ├── ShardKeySetConstraint.php
│   ├── ShardKeyRangeConstraint.php
│   ├── CompositeShardKeyConstraint.php
│   ├── AllPartitionsConstraint.php
│   ├── NoPartitionsConstraint.php
│   └── UnknownPartitionConstraint.php
│
├── Analysis/
│   ├── ShardConstraintAnalyzer.php
│   ├── PredicateShardAnalyzer.php
│   ├── CompositeShardKeyAnalyzer.php
│   ├── JoinShardAnalyzer.php
│   ├── InsertShardAnalyzer.php
│   ├── UpdateShardAnalyzer.php
│   ├── DeleteShardAnalyzer.php
│   └── SelectShardAnalyzer.php
│
├── Resolution/
│   ├── PartitionResolver.php
│   ├── ShardTargetSet.php
│   ├── ExactPartitionResolver.php
│   ├── RangePartitionResolver.php
│   └── DirectoryPartitionResolver.php
│
├── Plan/
│   ├── PartitionedQueryPlan.php
│   ├── FanOutPlan.php
│   ├── FanOutBudget.php
│   ├── FanOutExecutionPolicy.php
│   └── RoutingPlanCache.php
│
├── Transaction/
│   ├── TransactionShardGuard.php
│   └── TransactionShardAffinity.php
│
├── ORM/
│   ├── EntityPartitionResolver.php
│   ├── PersistencePartitionResolver.php
│   └── RelationshipPartitionResolver.php
│
├── Ownership/
│   ├── OwnershipRoutingResolver.php
│   ├── StaleRoutingDetector.php
│   └── ShardRedirectHandler.php
│
├── Policy/
│   ├── UnknownRoutingPolicy.php
│   ├── GlobalReadPolicy.php
│   ├── GlobalWritePolicy.php
│   └── FanOutPolicy.php
│
├── Diagnostics/
│   ├── PartitionRoutingInspector.php
│   ├── PartitionRoutingExplainer.php
│   └── PartitionRoutingDiagnosticReport.php
│
├── Telemetry/
│   ├── PartitionRoutingTelemetry.php
│   ├── PartitionRoutingCompleted.php
│   └── UnknownRouteDetected.php
│
└── Exception/
    ├── DatabasePartitionRoutingException.php
    ├── PartitionRoutingUnknownException.php
    ├── PartitionRoutingRejectedException.php
    ├── FanOutBudgetExceededException.php
    ├── TransactionShardMismatchException.php
    └── PartitionRoutingInvariantViolationException.php
```

---

# 226. API interna

Contrato principal:

```php
interface PartitionRouter
{
    public function route(
        PartitionRouteRequest $request
    ): PartitionRoutingDecision;
}
```

---

# 227. Pipeline

```text
RouteRequest
    ↓
Resolve Entity/Table Sharding Metadata
    ↓
Analyze Semantic Constraints
    ↓
Normalize Shard Constraints
    ↓
Apply Transaction Affinity
    ↓
Apply Domain Constraints
    ↓
Resolve Through ShardMap
    ↓
Validate Ownership
    ↓
Apply Routing Policy
    ↓
Validate Fan-Out Budget
    ↓
Produce Immutable Decision
```

---

# 228. Constraint normalization

Ejemplo:

```text
customer_id = 10
OR
customer_id = 10
OR
customer_id = 20
```

→

```text
ShardKeySet {10,20}
```

---

# 229. Shard target normalization

```text
10 → shard-A
20 → shard-A
```

→

```text
TargetSet {shard-A}
Scope SINGLE
```

---

# 230. Scope derives from targets

Normalmente:

```text
0 targets → NONE
1 target  → SINGLE
N targets → MULTIPLE
all       → GLOBAL
```

pero `UNKNOWN` será un semantic state separado.

---

# 231. Global marker

`GLOBAL` no deberá inferirse solo comparando count si ShardMap está incompleto.

---

# 232. ShardMap completeness

Routing deberá conocer si el map representa:

```text
COMPLETE
PARTIAL
UNKNOWN
```

---

# 233. Partial ShardMap

No podrá afirmar:

```text
GLOBAL
```

si no conoce todos los shards relevantes.

---

# 234. Query pruning

Partition Routing funcionará como una forma de:

```text
distributed partition pruning
```

---

# 235. Pruning objective

Reducir:

```text
AllShards
```

a:

```text
RequiredShards
```

sin sacrificar correctness.

---

# 236. Optimization hierarchy

```text
Correctness
   >
Isolation
   >
Completeness
   >
Performance
```

---

# 237. No under-routing

Nunca optimizar:

```text
{A,B,C}
```

a:

```text
{A}
```

sin prueba semántica.

---

# 238. Over-routing

Puede permitirse:

```text
{A}
```

→

```text
{A,B}
```

si sigue siendo correcto y policy lo permite.

---

# 239. Write over-routing

Para writes, incluso over-routing puede ser incorrecto.

Por ello las reglas serán más estrictas.

---

# 240. Write target cardinality

Ordinary entity INSERT/UPDATE/DELETE deberá ser:

```text
exactly one shard
```

salvo operación global/multi-shard explícita.

---

# 241. Read target cardinality

Reads pueden ser:

```text
0..N shards
```

según semantics/policy.

---

# 242. Testing architecture

Suite propuesta:

```text
PartitionRouterTests
PartitionScopeTests
ShardConstraintAnalyzerTests
PredicateShardAnalyzerTests
ExactShardRoutingTests
SetShardRoutingTests
RangeShardRoutingTests
CompositeShardRoutingTests
InsertRoutingTests
UpdateRoutingTests
DeleteRoutingTests
SelectRoutingTests
JoinRoutingTests
AggregateRoutingTests
SubqueryRoutingTests
UnionRoutingTests
TransactionShardRoutingTests
PersistenceShardRoutingTests
RelationshipShardRoutingTests
GlobalRoutingPolicyTests
FanOutBudgetTests
ShardMapGenerationTests
ShardOwnershipTransitionTests
ShardRedirectTests
RoutingCacheTests
PersistentRuntimeRoutingTests
RoutingTelemetryTests
RoutingDiagnosticsTests
```

---

# 243. Equality test

```text
customer_id = 42
```

deberá producir exactamente el owner de `42`.

---

# 244. IN test

```text
customer_id IN (1,2,3,4)
```

deberá deduplicar target shards.

---

# 245. Empty IN test

Deberá poder producir:

```text
NONE
```

si Query Semantics lo normaliza a false.

---

# 246. OR safety test

```text
shard_key = 1
OR non_shard_predicate
```

no deberá producir single-shard routing incorrecto.

---

# 247. AND pruning test

```text
shard_key = 1
AND status = 'active'
```

deberá conservar single-shard constraint.

---

# 248. Composite-key test

Partial composite key no deberá asumirse complete.

---

# 249. Hash-range test

Range predicate sobre hash strategy no deberá realizar range pruning falso.

---

# 250. Insert safety test

INSERT sin required shard key deberá fallar antes de execution.

---

# 251. Shard-key mutation test

UPDATE de shard key no será ordinary routed update.

---

# 252. Transaction test

Transaction shard A + operation shard B deberá fallar.

---

# 253. Multi-shard transaction test

MULTIPLE target dentro de ordinary transaction deberá fallar.

---

# 254. Fan-out budget test

Target count mayor al budget deberá fallar antes de dispatch.

---

# 255. Partial failure test

Distributed result no deberá presentarse como complete.

---

# 256. Stale map test

Routing decision deberá conservar generation.

---

# 257. Ownership transition test

Write deberá respetar authoritative owner.

---

# 258. Redirect-loop test

```text
A → B → C → A
```

deberá detenerse.

---

# 259. Cache-generation test

Cached assignment de generación vieja no deberá aplicarse ciegamente a una nueva.

---

# 260. Worker isolation test

Request A:

```text
shard-A
```

Request B:

```text
shard-B
```

en mismo worker no deberán contaminarse.

---

# 261. Coroutine test

Concurrent routing deberá permanecer aislado.

---

# 262. Property-based tests

Para shard constraints:

```text
RequiredShards
⊆
RoutedShards
```

deberá verificarse automáticamente sobre combinaciones generadas.

---

# 263. Invariantes arquitectónicos

## DB-PRT-001
Partition Routing será distinto de Sharding.

## DB-PRT-002
Partition Routing será distinto de Read/Write Routing.

## DB-PRT-003
Partition Routing será distinto de Load Balancing.

## DB-PRT-004
Partition Routing no elegirá Connection.

## DB-PRT-005
Partition Routing no ejecutará SQL.

## DB-PRT-006
Partition Routing no generará SQL.

## DB-PRT-007
Partition Routing no hidratará entidades.

## DB-PRT-008
Partition Routing no implementará distributed transactions.

## DB-PRT-009
Toda decisión tendrá PartitionScope.

## DB-PRT-010
SINGLE tendrá exactamente un target.

## DB-PRT-011
MULTIPLE tendrá múltiples targets conocidos.

## DB-PRT-012
GLOBAL significará todos los shards elegibles conocidos.

## DB-PRT-013
UNKNOWN será distinto de GLOBAL.

## DB-PRT-014
NONE será distinto de UNKNOWN.

## DB-PRT-015
UNKNOWN no se convertirá silenciosamente en GLOBAL.

## DB-PRT-016
Routing conservará ShardMapGeneration.

## DB-PRT-017
Routing deberá ser explainable.

## DB-PRT-018
Routing deberá preservar correctness antes que performance.

## DB-PRT-019
Required shards nunca podrán omitirse por optimización.

## DB-PRT-020
Over-routing solo se permitirá cuando sea semánticamente seguro.

## DB-PRT-021
Write over-routing será prohibido por default.

## DB-PRT-022
Shard constraints derivarán de semántica, no SQL regex.

## DB-PRT-023
Query AST podrá alimentar routing.

## DB-PRT-024
Semantic Query Graph podrá alimentar routing.

## DB-PRT-025
Resolved parameters podrán alimentar routing.

## DB-PRT-026
Shard key parameters serán type validated.

## DB-PRT-027
Shard key normalization será canonical.

## DB-PRT-028
Unsafe coercion estará prohibida.

## DB-PRT-029
Equality sobre complete shard key podrá producir exact routing.

## DB-PRT-030
IN podrá producir known multi-shard routing.

## DB-PRT-031
Duplicate shard targets serán deduplicados.

## DB-PRT-032
Range pruning dependerá de sharding strategy.

## DB-PRT-033
Hash sharding no fingirá range locality.

## DB-PRT-034
Contradictory predicates podrán producir NONE cuando se demuestre.

## DB-PRT-035
OR con unconstrained branch no producirá unsafe pruning.

## DB-PRT-036
AND preservará valid shard constraints.

## DB-PRT-037
Composite shard key requerirá semántica compatible.

## DB-PRT-038
Partial composite key no será tratada como complete.

## DB-PRT-039
Composite pruning dependerá de strategy capabilities.

## DB-PRT-040
INSERT sharded ordinario tendrá single target.

## DB-PRT-041
INSERT sin required shard key será rechazado.

## DB-PRT-042
Batch INSERT podrá agruparse por shard.

## DB-PRT-043
Multi-shard batch INSERT no implicará atomicidad global.

## DB-PRT-044
UPDATE podrá utilizar shard predicates.

## DB-PRT-045
DELETE podrá utilizar shard predicates.

## DB-PRT-046
Primary key no implicará shard key.

## DB-PRT-047
Shard-key mutation no será ordinary UPDATE.

## DB-PRT-048
Global writes requerirán policy explícita.

## DB-PRT-049
UNKNOWN nunca disparará global write.

## DB-PRT-050
SELECT podrá producir NONE/SINGLE/MULTIPLE/GLOBAL/UNKNOWN.

## DB-PRT-051
Global read será gobernado por policy.

## DB-PRT-052
Scatter-gather será distinto de Partition Routing.

## DB-PRT-053
Partition Routing determinará fan-out targets.

## DB-PRT-054
Partition Routing no fusionará resultados.

## DB-PRT-055
Fan-out tendrá resource budget.

## DB-PRT-056
Fan-out concurrency será bounded.

## DB-PRT-057
Global query no significará unlimited concurrency.

## DB-PRT-058
Partial distributed failure no será complete success.

## DB-PRT-059
Partial result requerirá semántica explícita.

## DB-PRT-060
JOIN co-location deberá demostrarse.

## DB-PRT-061
Same field name no demostrará co-location.

## DB-PRT-062
Cross-shard JOIN no se ejecutará como local JOIN.

## DB-PRT-063
Aggregate routing será distinto de aggregate merge.

## DB-PRT-064
Global ordering requerirá distributed planning.

## DB-PRT-065
LIMIT local no implicará LIMIT global correcto.

## DB-PRT-066
Pagination será distinta de routing.

## DB-PRT-067
Subqueries podrán tener routing analysis propio.

## DB-PRT-068
UNION combinará target semantics de branches.

## DB-PRT-069
Repository podrá aportar routing semantics.

## DB-PRT-070
Repository no seleccionará physical connection directamente.

## DB-PRT-071
Shard key routing será preferible a hardcoded ShardId cuando sea posible.

## DB-PRT-072
ShardHint será distinto de verified assignment.

## DB-PRT-073
Hint deberá validarse.

## DB-PRT-074
Forced shard routing será explícito.

## DB-PRT-075
Forced routing será auditable.

## DB-PRT-076
Forced routing no saltará authorization.

## DB-PRT-077
Forced routing no saltará tenant/domain constraints.

## DB-PRT-078
RoutingContext será scoped.

## DB-PRT-079
RoutingContext no contendrá service container completo.

## DB-PRT-080
Transaction affinity dominará routing normal.

## DB-PRT-081
Bound transaction no cambiará shard.

## DB-PRT-082
Transaction shard mismatch será rechazado.

## DB-PRT-083
MULTIPLE dentro de single-resource transaction será rechazado.

## DB-PRT-084
GLOBAL dentro de single-resource transaction será rechazado.

## DB-PRT-085
Savepoint no creará shard domain nuevo.

## DB-PRT-086
Joined nested transaction heredará shard affinity.

## DB-PRT-087
Retry attempt podrá re-resolver routing según policy.

## DB-PRT-088
Same transaction attempt no migrará silenciosamente entre shards.

## DB-PRT-089
Persistence operation tendrá shard assignment antes de execution.

## DB-PRT-090
Cross-shard UnitOfWork será detectable.

## DB-PRT-091
Cross-shard flush no fingirá atomicidad.

## DB-PRT-092
Hydrator no decidirá shard.

## DB-PRT-093
IdentityMap no decidirá shard.

## DB-PRT-094
Relationship loading preservará shard context.

## DB-PRT-095
Cross-shard relationship load será observable.

## DB-PRT-096
Batch relation loading agrupará por shard.

## DB-PRT-097
Eager JOIN requerirá co-location.

## DB-PRT-098
ShardMap snapshot será immutable durante routing decision.

## DB-PRT-099
Routing decision conservará map generation.

## DB-PRT-100
Stale routing será detectable.

## DB-PRT-101
Writes tendrán stale-map policy estricta.

## DB-PRT-102
Ownership transitions serán consideradas.

## DB-PRT-103
Partition Router no inventará dual-write.

## DB-PRT-104
Redirect será validado.

## DB-PRT-105
Redirect será bounded.

## DB-PRT-106
Redirect loop será detectado.

## DB-PRT-107
Redirect dentro de active transaction no cambiará shard silenciosamente.

## DB-PRT-108
Routing retry será distinto de transaction retry.

## DB-PRT-109
Routing cache será generation-aware.

## DB-PRT-110
Semantic routing-plan cache será distinto de assignment cache.

## DB-PRT-111
Prepared queries podrán reutilizar extraction plans.

## DB-PRT-112
Map generation change podrá cambiar assignment sin cambiar query shape.

## DB-PRT-113
UnknownRoutingPolicy tendrá safe default.

## DB-PRT-114
GlobalWritePolicy tendrá safe default.

## DB-PRT-115
Global mutation será explícita.

## DB-PRT-116
Global mutation podrá requerir audit.

## DB-PRT-117
Toda decision tendrá RoutingReason.

## DB-PRT-118
Diagnostics no expondrán shard keys sensibles por default.

## DB-PRT-119
Routing será distinto de authorization.

## DB-PRT-120
Database core no dependerá de Multitenancy.

## DB-PRT-121
Tenant integration podrá añadir routing constraints.

## DB-PRT-122
Tenant/shard mismatch fallará cerrado.

## DB-PRT-123
Raw SQL requerirá routing context explícito cuando no pueda inferirse.

## DB-PRT-124
Raw SQL no se analizará mediante heurística insegura.

## DB-PRT-125
Stored procedures podrán declarar routing metadata.

## DB-PRT-126
Custom semantic expressions podrán extender shard analysis.

## DB-PRT-127
Unknown custom expression no producirá falsa certeza.

## DB-PRT-128
Partition pruning ocurrirá antes de endpoint selection.

## DB-PRT-129
Partitioned plan será distinto de physical SQL plan.

## DB-PRT-130
Multi-shard platform capabilities serán verificadas.

## DB-PRT-131
Unsupported target no será ignorado silenciosamente.

## DB-PRT-132
Cancellation podrá propagarse a fan-out execution.

## DB-PRT-133
Global deadline será distinto de per-shard timeout acumulado.

## DB-PRT-134
Routing cost será observable.

## DB-PRT-135
Interactive requests podrán tener budgets distintos de background jobs.

## DB-PRT-136
HTTP Request no será dependencia de routing core.

## DB-PRT-137
Immutable routing metadata podrá compartirse entre workers.

## DB-PRT-138
Mutable routing state será operation-scoped.

## DB-PRT-139
Static mutable current shard estará prohibido.

## DB-PRT-140
Worker reuse no filtrará routing state.

## DB-PRT-141
Coroutine routing state estará aislado.

## DB-PRT-142
Fiber routing state estará aislado.

## DB-PRT-143
Telemetry será observational.

## DB-PRT-144
Telemetry no modificará routing.

## DB-PRT-145
Metric cardinality será gobernada.

## DB-PRT-146
QueryOperationId podrá correlacionar routing y execution.

## DB-PRT-147
Shard key raw no será telemetry default.

## DB-PRT-148
Partial ShardMap no permitirá afirmar GLOBAL sin evidencia.

## DB-PRT-149
Map completeness será explícita.

## DB-PRT-150
Partition pruning nunca sacrificará completeness.

## DB-PRT-151
Writes ordinarios requerirán exact target.

## DB-PRT-152
Reads podrán usar conservative supersets cuando policy lo permita.

## DB-PRT-153
Conservative routing deberá incluir todos los required shards.

## DB-PRT-154
Routing será deterministic para misma operación, contexto y map generation.

## DB-PRT-155
Load no modificará partition ownership.

## DB-PRT-156
Replica latency no modificará partition target.

## DB-PRT-157
Failover no cambiará logical shard.

## DB-PRT-158
Read/Write Router consumirá el shard result.

## DB-PRT-159
Replica System consumirá el replication group del shard.

## DB-PRT-160
Load Balancer operará únicamente después de establecer partition correctness.

---

# 264. Modelo formal

Sea:

```text
S = {s₁, s₂, ..., sₙ}
```

el conjunto de shards.

Para una operación `O`, definimos:

```text
R(O) ⊆ S
```

como el conjunto real de shards que contienen datos relevantes para la operación.

Partition Routing produce:

```text
P(O) ⊆ S
```

Debe cumplirse:

```text
R(O) ⊆ P(O)
```

para reads conservadores.

Idealmente:

```text
R(O) = P(O)
```

---

# 265. Under-routing

Si:

```text
R(O) = {A,B}
```

pero:

```text
P(O) = {A}
```

entonces:

```text
UNDER_ROUTING
```

Es un error de correctness.

---

# 266. Over-routing

Si:

```text
R(O) = {A}
```

y:

```text
P(O) = {A,B}
```

entonces:

```text
OVER_ROUTING
```

Puede ser correcto para reads, aunque ineficiente.

---

# 267. Exact routing

```text
P(O) = R(O)
```

representa routing óptimo.

---

# 268. Write formalism

Para un ordinary single-record write `W`:

```text
|P(W)| = 1
```

deberá cumplirse antes de execution.

---

# 269. Transaction formalism

Para una transacción `T`:

```text
Shard(T) = s
```

Entonces toda operación `O` dentro de `T` deberá cumplir:

```text
P(O) ⊆ {s}
```

Para ordinary single-resource transactions.

---

# 270. Fan-out factor

Definimos:

```text
FanOut(O) = |P(O)|
```

---

# 271. Routing efficiency

Una métrica conceptual:

```text
RoutingEfficiency(O)
=
|R(O)| / |P(O)|
```

cuando `R(O)` pueda estimarse/observarse.

Ideal:

```text
1.0
```

---

# 272. Arquitectura final del Bloque 16

Con los documentos 176–185:

```text
                     DATABASE OPERATION
                             │
                             ▼
                  DISTRIBUTED DATABASE
                             │
                             ▼
                         SHARDING
                             │
                             ▼
                    PARTITION ROUTING
                             │
                             ▼
                         SHARD
                             │
                             ▼
                   REPLICATION GROUP
                             │
                             ▼
                   READ / WRITE ROUTING
                             │
                             ▼
                  REPLICA LAG AWARENESS
                             │
                             ▼
                    STICKY CONSTRAINTS
                             │
                             ▼
                       FAILOVER
                             │
                             ▼
                    LOAD BALANCING
                             │
                             ▼
                 CONNECTION / ENDPOINT
                             │
                             ▼
                        EXECUTION
```

---

# 273. Orden lógico de resolución

La secuencia deberá conservarse:

```text
1. What data domain is involved?
                ↓
2. Which shard owns that data?
                ↓
3. Which shard(s) must execute the operation?
                ↓
4. Is this a read or write?
                ↓
5. Which replicas are eligible?
                ↓
6. Is freshness sufficient?
                ↓
7. Are sticky constraints active?
                ↓
8. Has failover changed topology?
                ↓
9. Which eligible endpoint should receive it?
                ↓
10. Acquire connection and execute.
```

---

# 274. Regla maestra final

> **Partition Routing será la frontera de correctness entre una operación lógica y la topología distribuida. Su responsabilidad será demostrar el conjunto de shards necesario para preservar la semántica de la operación. Nunca elegirá menos shards de los requeridos, nunca convertirá incertidumbre en certeza, nunca utilizará carga o latencia para redefinir ownership y nunca confundirá shard selection con replica selection.**

En forma compacta:

```text
Query Semantics
      ↓
Shard Constraints
      ↓
Partition Routing
      ↓
Shard Target Set
      ↓
Replication Group
      ↓
Read / Write Policy
      ↓
Replica Eligibility
      ↓
Load Balancing
      ↓
Physical Execution
```

Y la separación definitiva será:

```text
Sharding
=
Where data belongs.

Partition Routing
=
Where an operation must execute.

Replication
=
Which copies of that shard exist.

Read/Write Routing
=
Which role may serve the operation.

Load Balancing
=
Which eligible copy should receive it.

Execution
=
Actually perform the database operation.
```

---

# 275. Cierre del Bloque 16 — Read/Write and Distribution

Quedan completados:

```text
176_DATABASE_READ_WRITE_CONNECTION_SYSTEM.md
177_DATABASE_READ_WRITE_ROUTING_SYSTEM.md
178_DATABASE_REPLICA_SYSTEM.md
179_DATABASE_REPLICA_LAG_AWARENESS_SYSTEM.md
180_DATABASE_STICKY_CONNECTION_SYSTEM.md
181_DATABASE_FAILOVER_SYSTEM.md
182_DATABASE_LOAD_BALANCING_SYSTEM.md
183_DATABASE_DISTRIBUTED_DATABASE_ARCHITECTURE.md
184_DATABASE_SHARDING_SYSTEM.md
185_DATABASE_PARTITION_ROUTING_SYSTEM.md
```

La arquitectura distribuida resultante puede resumirse:

```text
                    Logical Database
                           │
                           ▼
                 Distributed Topology
                           │
              ┌────────────┴─────────────┐
              │                          │
              ▼                          ▼
           Sharding                 Replication
              │                          │
              ▼                          │
        Partition Routing                │
              │                          │
              └────────────┬─────────────┘
                           ▼
                    Execution Domain
                           │
                           ▼
                   Read/Write Routing
                           │
                           ▼
                 Replica Lag Awareness
                           │
                           ▼
                    Sticky Routing
                           │
                           ▼
                       Failover
                           │
                           ▼
                   Load Balancing
                           │
                           ▼
                 Physical Connection
```

Con esto, VoltStack dispone conceptualmente de una arquitectura capaz de evolucionar desde:

```text
Single Database
```

hacia:

```text
Primary + Replicas
```

y posteriormente:

```text
Sharded
+
Replicated
+
Failure-Aware
+
Lag-Aware
+
Partition-Aware
+
Load-Balanced
```

sin introducir esas capacidades como dependencias obligatorias para una aplicación sencilla.

---

# 276. Siguiente bloque

Con `185_DATABASE_PARTITION_ROUTING_SYSTEM.md` queda cerrado:

```text
BLOCK 16
READ / WRITE AND DISTRIBUTION
```

El siguiente bloque será:

```text
BLOCK 17
DATABASE CACHE
```

Documentos:

```text
186_DATABASE_CACHE_ARCHITECTURE.md
187_DATABASE_QUERY_CACHE_SYSTEM.md
188_DATABASE_RESULT_CACHE_SYSTEM.md
189_DATABASE_METADATA_CACHE_SYSTEM.md
190_DATABASE_ENTITY_CACHE_SYSTEM.md
191_DATABASE_CACHE_INVALIDATION_SYSTEM.md
192_DATABASE_CACHE_CONSISTENCY_SYSTEM.md
```

---

# 277. Siguiente documento

```text
186_DATABASE_CACHE_ARCHITECTURE.md
```

Este documento establecerá la arquitectura general de caching para `VoltStack/Quantum/Database`, manteniendo separaciones críticas:

```text
Query Cache
≠
Compiled Query Cache
≠
Result Cache
≠
Metadata Cache
≠
Entity Cache
≠
IdentityMap
≠
Hydration Cache
≠
Application Cache
```

La arquitectura deberá garantizar especialmente que:

```text
Cache Hit
≠
Database Truth
```

y:

```text
IdentityMap
≠
Second-Level Entity Cache
```

mientras integra:

```text
Query Engine
ORM
Persistence
Transactions
Sharding
Replication
Telemetry
VoltStack Cache
Persistent Runtimes
```

sin convertir el Database Core en dependiente obligatorio de un proveedor concreto de caché.