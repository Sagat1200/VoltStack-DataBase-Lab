# 184_DATABASE_SHARDING_SYSTEM.md

# VoltStack Quantum Database
## Database Sharding System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 184 — Database Sharding System  
**Bloque:** 16 — Read/Write and Distribution  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `183_DATABASE_DISTRIBUTED_DATABASE_ARCHITECTURE.md`  
**Siguiente documento:** `185_DATABASE_PARTITION_ROUTING_SYSTEM.md`

---

# 1. Propósito

`Database Sharding System` define la arquitectura mediante la cual VoltStack podrá dividir un dominio lógico de datos en múltiples particiones horizontales independientes denominadas **shards**.

El sistema establecerá:

- qué es un shard;
- qué es una shard key;
- cómo se define un shard map;
- cómo se asignan datos a shards;
- cómo evolucionan los shards;
- cómo se representa su ownership;
- cómo se integran con replicas;
- cómo interactúan con ORM;
- cómo afectan IdentityMap y UnitOfWork;
- cómo afectan transacciones;
- cómo se manejan movimientos de datos;
- cómo se detectan hot shards;
- cómo se soportan futuras operaciones de re-sharding.

La regla central será:

> **Sharding determina dónde pertenecen los datos; no determina qué replica ejecutará una operación, no balancea conexiones y no convierte operaciones entre shards en una transacción distribuida.**

Formalmente:

```text
ShardAssignment(DataIdentity)
→
Exactly One Logical Shard
```

para estrategias de partición exclusivas.

---

# 2. Contexto arquitectónico

El documento anterior estableció:

```text
Logical Database
        │
        ▼
Distributed Topology
        │
   ┌────┴─────┐
   ▼          ▼
Partition   Replication
   │          │
   └────┬─────┘
        ▼
Execution Domain
```

Ahora se desarrolla:

```text
Logical Dataset
      │
      ▼
Sharding Model
      │
      ▼
Shard Key
      │
      ▼
Shard Strategy
      │
      ▼
Shard Map
      │
      ▼
Shard
      │
      ▼
Replication Group
```

---

# 3. Distinciones fundamentales

```text
Sharding
≠
Partition Routing
≠
Replication
≠
Load Balancing
≠
Multitenancy
≠
Connection Routing
≠
Table Partitioning
≠
Distributed Transaction
```

---

# 4. Sharding vs partition routing

Sharding responde:

> ¿Cómo está dividido el dataset?

Partition Routing responde:

> ¿A qué partición debe enviarse esta operación concreta?

Por tanto:

```text
Sharding System
→ defines distribution

Partition Routing System
→ consumes distribution
```

---

# 5. Sharding vs replication

Sharding:

```text
Dataset
├── Data A → Shard 1
├── Data B → Shard 2
└── Data C → Shard 3
```

Replication:

```text
Shard 1
├── Primary
├── Replica A
└── Replica B
```

Ambos pueden combinarse:

```text
Logical Database
│
├── Shard 01
│   ├── Writer
│   ├── Replica A
│   └── Replica B
│
├── Shard 02
│   ├── Writer
│   └── Replica A
│
└── Shard 03
    ├── Writer
    ├── Replica A
    └── Replica B
```

---

# 6. Sharding vs database table partitioning

Un motor puede implementar:

```text
PARTITION BY ...
```

dentro de una misma instancia.

Eso no es necesariamente sharding para VoltStack.

VoltStack utilizará `Shard` para representar una partición que posee **identidad de routing propia**.

---

# 7. Sharding vs multitenancy

Un tenant puede:

- compartir shard;
- tener shard dedicado;
- tener database dedicada;
- abarcar múltiples shards bajo una estrategia avanzada.

Por ello:

```text
TenantId
≠
ShardId
```

aunque `TenantId` pueda utilizarse como shard key.

---

# 8. Objetivos

El sistema deberá proporcionar:

1. `ShardId`;
2. `ShardKey`;
3. `ShardKeyDefinition`;
4. `ShardMap`;
5. `ShardDefinition`;
6. estrategias de asignación;
7. hash sharding;
8. range sharding;
9. directory sharding;
10. composite shard keys;
11. virtual shards;
12. shard ownership;
13. shard states;
14. shard topology;
15. shard placement;
16. shard generation;
17. shard splitting;
18. shard merging;
19. shard draining;
20. shard movement;
21. rebalancing;
22. re-sharding readiness;
23. hot-shard awareness;
24. ORM integration;
25. transaction integration;
26. migration integration;
27. persistent-runtime safety;
28. diagnostics;
29. telemetry;
30. extensibility.

---

# 9. No objetivos

Este sistema no deberá:

- ejecutar SQL;
- implementar DB replication;
- decidir read/write replica;
- balancear replicas;
- implementar 2PC;
- implementar consenso distribuido;
- mover datos silenciosamente;
- crear shards automáticamente sin política;
- convertir todas las queries en scatter-gather;
- utilizar tenant como shard automáticamente;
- modificar EntityId para esconder routing;
- hacer network discovery por cada query.

---

# 10. Shard

Un shard representa un dominio lógico de partición.

```php
final readonly class Shard
{
    public function __construct(
        public ShardId $id,
        public ShardState $state,
        public ReplicationGroupId $replicationGroup,
        public ShardPlacement $placement,
        public ShardMetadata $metadata,
    ) {}
}
```

---

# 11. ShardId

```php
final readonly class ShardId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Debe ser estable.

Ejemplos:

```text
shard-001
shard-002
shard-europe-01
customer-shard-07
```

---

# 12. Shard identity ≠ location

Nunca:

```text
ShardId = IP address
```

por diseño.

Un shard puede cambiar de:

```text
server A
```

a:

```text
server B
```

sin cambiar su identidad lógica.

---

# 13. Shard ≠ server

Un servidor puede alojar:

```text
Shard 1
Shard 2
Shard 3
```

y un shard puede estar replicado en múltiples servidores.

---

# 14. ShardDefinition

```php
final readonly class ShardDefinition
{
    public function __construct(
        public ShardId $id,
        public ShardBoundary $boundary,
        public ReplicationGroupId $replicationGroup,
        public ShardState $state,
    ) {}
}
```

---

# 15. ShardKey

`ShardKey` representa el valor semántico utilizado para determinar la partición.

Ejemplos:

```text
user_id
account_id
organization_id
tenant_id
region + account_id
```

---

# 16. ShardKey ≠ primary key

Una tabla puede tener:

```text
PRIMARY KEY = order_id
SHARD KEY   = customer_id
```

---

# 17. ShardKeyDefinition

```php
final readonly class ShardKeyDefinition
{
    public function __construct(
        public ShardKeyId $id,
        public array $components,
        public ShardKeyType $type,
        public ShardKeyNormalizationPolicy $normalization,
    ) {}
}
```

---

# 18. Single-component key

```text
customer_id
```

---

# 19. Composite shard key

```text
region
+
customer_id
```

produce:

```text
ShardKey(region, customer_id)
```

---

# 20. Composite key ordering

El orden forma parte de su semántica.

```text
(region, customer)
≠
(customer, region)
```

por default.

---

# 21. Canonical shard key

Antes de routing:

```text
Raw Value
    ↓
Type Validation
    ↓
Normalization
    ↓
CanonicalShardKey
```

---

# 22. Determinism

Debe cumplirse:

```text
Normalize(x) = Normalize(x)
```

para el mismo valor y policy generation.

---

# 23. No locale-dependent hashing

Una shard key no deberá cambiar por:

- locale;
- timezone implícita;
- formato visual;
- encoding no normalizado.

---

# 24. Null shard keys

`NULL` deberá tener policy explícita.

Opciones:

```text
REJECT
DEFAULT_SHARD
DIRECTORY_LOOKUP
CUSTOM
```

No deberá elegirse un shard aleatoriamente.

---

# 25. Missing shard key

```text
MISSING
≠
NULL
```

Una query que no proporciona shard key tiene un problema de routing distinto de una key explícitamente nula.

---

# 26. ShardingStrategy

```php
enum ShardingStrategy
{
    case HASH;
    case RANGE;
    case DIRECTORY;
    case COMPOSITE;
    case VIRTUAL_NODE;
    case CUSTOM;
}
```

---

# 27. Strategy contract

```php
interface ShardAssignmentStrategy
{
    public function assign(
        CanonicalShardKey $key,
        ShardMap $map,
    ): ShardAssignment;
}
```

---

# 28. ShardAssignment

```php
final readonly class ShardAssignment
{
    public function __construct(
        public ShardId $shard,
        public ShardMapGeneration $generation,
        public ShardAssignmentReason $reason,
    ) {}
}
```

---

# 29. Assignment ≠ routing decision

`ShardAssignment` determina pertenencia.

No determina:

- writer;
- replica;
- connection;
- endpoint.

---

# 30. Hash sharding

Modelo simple:

```text
hash(key) mod N
```

Ejemplo:

```text
hash(user_id) % 4
```

---

# 31. Problem with modulo hashing

Si:

```text
N = 4
```

cambia a:

```text
N = 5
```

una gran proporción de keys puede cambiar de shard.

---

# 32. Modulo hash policy

Podrá soportarse para:

- configuraciones estáticas;
- clusters pequeños;
- datasets sin rebalancing frecuente.

Pero no será la única estrategia.

---

# 33. Stable hashing

VoltStack deberá abstraer:

```php
interface ShardHasher
{
    public function hash(
        CanonicalShardKey $key
    ): ShardHash;
}
```

---

# 34. Hash algorithm version

Debe ser explícito:

```text
HashAlgorithmId
HashAlgorithmVersion
```

---

# 35. Critical rule

Cambiar algoritmo de hash puede cambiar la ubicación de los datos.

Por ello:

```text
Hash Algorithm Change
=
Data Migration Concern
```

---

# 36. No PHP runtime hash dependency

No utilizar funciones cuyo resultado pueda variar entre:

- procesos;
- versiones;
- arquitecturas;
- seeds internos.

---

# 37. Stable byte representation

Hashing deberá operar sobre:

```text
CanonicalShardKeyBytes
```

con encoding definido.

---

# 38. Range sharding

Ejemplo:

```text
0       – 999,999    → shard-1
1M      – 1,999,999  → shard-2
2M      – 2,999,999  → shard-3
```

---

# 39. ShardRange

```php
final readonly class ShardRange
{
    public function __construct(
        public ShardKeyBoundary $lower,
        public ShardKeyBoundary $upper,
        public ShardId $shard,
    ) {}
}
```

---

# 40. Boundary semantics

Debe definirse explícitamente:

```text
[lower, upper)
```

o equivalente.

No mezclar:

```text
inclusive
exclusive
```

de manera implícita.

---

# 41. Range validation

El map deberá detectar:

- gaps;
- overlaps;
- invalid ordering;
- duplicate ranges.

---

# 42. Gap policy

Un valor fuera de todos los ranges:

```text
UNASSIGNED
```

no deberá enviarse a un shard arbitrario.

---

# 43. Range hot spots

Keys monotónicas pueden concentrar writes en el último range.

Ejemplo:

```text
timestamp
auto-increment ID
```

---

# 44. Range strategy tradeoff

Ventaja:

```text
range queries
```

pueden localizarse mejor.

Desventaja:

```text
uneven write distribution
```

---

# 45. Directory sharding

Utiliza un mapa explícito:

```text
customer-1 → shard-7
customer-2 → shard-3
customer-3 → shard-9
```

---

# 46. Directory service

Conceptualmente:

```php
interface ShardDirectory
{
    public function lookup(
        CanonicalShardKey $key
    ): ShardDirectoryEntry;
}
```

---

# 47. Directory sharding advantage

Permite mover una key sin cambiar algoritmo global.

---

# 48. Directory disadvantage

Introduce:

- lookup dependency;
- cache requirements;
- directory availability;
- invalidation complexity.

---

# 49. Directory cache

Podrá existir:

```text
ShardDirectoryCache
```

pero deberá ser generation/version aware.

---

# 50. Stale directory mapping

Puede enviar una operación al shard anterior.

Por ello el sistema deberá poder detectar:

```text
ShardOwnershipMismatch
```

cuando infraestructura lo permita.

---

# 51. Directory entry

```php
final readonly class ShardDirectoryEntry
{
    public function __construct(
        public CanonicalShardKey $key,
        public ShardId $shard,
        public ShardOwnershipEpoch $epoch,
    ) {}
}
```

---

# 52. Composite sharding

Puede combinar estrategias.

Ejemplo:

```text
region
    ↓
range/directory
    ↓
customer hash
```

---

# 53. Composite strategy pipeline

```text
ShardKey
   │
   ▼
Region Partition
   │
   ▼
Hash Bucket
   │
   ▼
Shard
```

---

# 54. Complexity control

Composite strategies deberán ser:

- declarativas;
- compilables;
- explainable;
- bounded.

---

# 55. Virtual shards

VoltStack deberá soportar la idea de:

```text
VirtualShard
```

separada del physical shard placement.

---

# 56. Motivation

En vez de:

```text
key → physical shard
```

usar:

```text
key
 ↓
virtual shard
 ↓
physical shard
```

---

# 57. Example

```text
1024 virtual shards
        │
        ├── 0..255   → shard A
        ├── 256..511 → shard B
        ├── 512..767 → shard C
        └── 768..1023→ shard D
```

---

# 58. Rebalancing benefit

Mover:

```text
64 virtual shards
```

puede ser más controlable que cambiar todo el hash space.

---

# 59. VirtualShardId

```php
final readonly class VirtualShardId
{
    public function __construct(
        public int|string $value,
    ) {}
}
```

---

# 60. Virtual shard ≠ physical shard

Debe mantenerse la distinción.

---

# 61. ShardMap

Objeto canónico:

```php
final readonly class ShardMap
{
    public function __construct(
        public ShardMapId $id,
        public ShardMapGeneration $generation,
        public ShardingStrategyDefinition $strategy,
        public ShardDefinitionMap $shards,
    ) {}
}
```

---

# 62. ShardMap immutability

Una generación publicada será immutable.

---

# 63. ShardMapGeneration

```text
ShardMapGeneration
≠
DistributedTopologyGeneration
```

aunque puedan correlacionarse.

---

# 64. Why separate generations

Puede cambiar:

```text
replica health
```

sin cambiar:

```text
shard ownership
```

y puede cambiar:

```text
shard ownership
```

sin modificar todas las demás partes de topology.

---

# 65. Generation binding

Toda `ShardAssignment` deberá registrar la generación usada.

---

# 66. Stale assignment

Una assignment calculada con:

```text
generation 10
```

puede requerir revalidation si ya existe:

```text
generation 11
```

---

# 67. ShardMap lifecycle

```text
DISCOVER / CONFIGURE
       ↓
NORMALIZE
       ↓
VALIDATE
       ↓
COMPILE
       ↓
PUBLISH
       ↓
FREEZE
```

---

# 68. ShardMap validation

Debe comprobar:

- ShardIds únicos;
- strategy válida;
- boundaries válidos;
- no overlapping ownership;
- references válidas;
- replication groups existentes;
- state transitions válidas;
- virtual shard ownership completa cuando aplique.

---

# 69. Shard ownership

`ShardOwnership` define qué shard posee actualmente una parte del key space.

---

# 70. Ownership ≠ physical server ownership

Significa:

```text
logical data responsibility
```

no propiedad administrativa de hardware.

---

# 71. ShardOwnershipEpoch

```php
final readonly class ShardOwnershipEpoch
{
    public function __construct(
        public int|string $value,
    ) {}
}
```

---

# 72. Ownership epoch

Ayuda a detectar:

```text
stale routing
```

durante movimientos.

---

# 73. Ownership states

```php
enum ShardOwnershipState
{
    case STABLE;
    case COPYING;
    case DUAL_READ;
    case DUAL_WRITE;
    case CUTTING_OVER;
    case MOVED;
    case UNKNOWN;
}
```

No todas deberán habilitarse automáticamente.

---

# 74. Dual-write warning

`DUAL_WRITE` introduce problemas complejos:

- ordering;
- partial failure;
- retries;
- consistency;
- deduplication.

No deberá activarse como mecanismo genérico sin estrategia específica.

---

# 75. ShardState

```php
enum ShardState
{
    case ACTIVE;
    case DRAINING;
    case READ_ONLY;
    case MOVING;
    case SPLITTING;
    case MERGING;
    case OFFLINE;
    case UNKNOWN;
}
```

---

# 76. ACTIVE

Acepta operaciones compatibles normales.

---

# 77. DRAINING

No debería recibir nuevas asignaciones permanentes.

Puede seguir sirviendo operaciones existentes según policy.

---

# 78. READ_ONLY

No acepta writes normales.

---

# 79. MOVING

Parte de sus datos está en proceso de relocalización.

---

# 80. SPLITTING

Shard se divide en varios shards.

---

# 81. MERGING

Múltiples shards convergen.

---

# 82. OFFLINE

No disponible para routing normal.

---

# 83. UNKNOWN

Nunca deberá interpretarse como ACTIVE.

---

# 84. Shard state ≠ endpoint health

Un shard puede estar:

```text
ACTIVE
```

pero todas sus replicas:

```text
UNHEALTHY
```

---

# 85. Shard placement

```php
final readonly class ShardPlacement
{
    public function __construct(
        public ReplicationGroupId $replicationGroup,
        public ?RegionId $preferredRegion,
        public ?FailureDomainId $failureDomain,
    ) {}
}
```

---

# 86. Placement ≠ routing endpoint

Placement describe dónde está alojado el shard.

Read/write routing todavía decide endpoint.

---

# 87. Shard and replication

Modelo:

```text
Shard
  │
  ▼
ReplicationGroup
  │
  ├── Writer
  ├── Replica
  └── Replica
```

---

# 88. Correct order

```text
Shard Resolution
      ↓
Replication Group
      ↓
Read/Write Routing
      ↓
Replica Eligibility
      ↓
Load Balancing
```

---

# 89. Never inverse

No:

```text
choose fastest replica
      ↓
discover which shard it contains
```

---

# 90. ShardKey metadata

El ORM deberá poder declarar shard key.

Ejemplo:

```php
#[ShardKey]
private int $accountId;
```

o mediante metadata declarativa.

---

# 91. No hard dependency on attributes

La metadata compilada seguirá siendo la fuente runtime canónica.

---

# 92. Model API example

Conceptualmente:

```php
final class Order extends Model
{
    protected static function shardKey(): string
    {
        return 'customer_id';
    }
}
```

Ambas APIs convergen en:

```text
CompiledEntityShardMetadata
```

---

# 93. EntityShardMetadata

```php
final readonly class EntityShardMetadata
{
    public function __construct(
        public EntityType $entity,
        public ShardKeyDefinition $key,
        public ShardMapId $map,
    ) {}
}
```

---

# 94. Entity creation

Para una nueva entidad sharded:

```text
ShardKey
```

deberá conocerse antes de persistir cuando sea necesaria para routing.

---

# 95. Missing shard key on INSERT

Debe fallar antes de generar SQL.

Ejemplo:

```text
ShardKeyRequiredException
```

---

# 96. Shard key mutation

Cambiar shard key puede implicar mover la entidad entre shards.

---

# 97. Critical rule

Una shard key persistida no deberá tratarse como un campo ordinario mutable.

---

# 98. Default policy

```text
ShardKeyMutation
→ reject
```

para una entidad managed existente.

---

# 99. Explicit relocation

Si se desea:

```text
entity.moveToShard(...)
```

deberá ser una operación especializada.

---

# 100. Shard relocation ≠ UPDATE

Porque puede requerir:

```text
source read
target insert
relationship migration
verification
cutover
source delete
cache invalidation
```

---

# 101. IdentityMap

El identity domain deberá incorporar shard cuando los IDs no sean globalmente únicos.

```text
EntityKey
=
EntityType
×
Identifier
×
ShardId
×
OtherIdentityDomain
```

---

# 102. Global IDs

Aunque IDs sean UUID globales, conservar shard context puede seguir siendo útil para routing.

---

# 103. Entity reference

Una referencia distribuida puede requerir:

```text
EntityReference
├── EntityType
├── Identifier
└── ShardHint / ShardIdentity
```

---

# 104. Hint ≠ authority

`ShardHint` puede acelerar resolución.

Debe revalidarse cuando sea necesario.

---

# 105. Repository integration

Consulta:

```php
$orders->forCustomer($customerId);
```

puede inferir shard key.

---

# 106. Query Builder integration

Predicados podrán alimentar el futuro `PartitionRoutingSystem`.

Ejemplo:

```php
Order::query()
    ->where('customer_id', 100);
```

Semantic analysis puede inferir:

```text
ShardKey = 100
```

---

# 107. SQL compiler boundary

Compiler no decide shard.

---

# 108. Executor boundary

Executor recibe un execution plan ya asociado al domain correspondiente.

---

# 109. Semantic Query Engine integration

Shard routing hints deberán derivarse de:

```text
Semantic Query Graph
```

cuando sea posible.

---

# 110. Raw SQL

Raw SQL dificulta inferencia.

Policy podrá exigir:

```php
DB::onShard('shard-07')->raw(...);
```

---

# 111. No SQL parsing guess

VoltStack no deberá intentar inferir de forma insegura toda shard key desde SQL arbitrario.

---

# 112. Explicit shard API

Conceptualmente:

```php
DB::shard('shard-07')
    ->table('orders')
    ->where(...)
    ->get();
```

---

# 113. Explicit shard selection validation

Seleccionar manualmente un shard no deberá saltarse:

- tenant constraints;
- transaction domain;
- security;
- shard state.

---

# 114. Transaction integration

Una transacción sharded se fija a:

```text
LogicalDatabase
+
ShardId
+
ReplicationGroup
+
ConnectionLease
```

---

# 115. First operation

Una transacción podrá:

1. requerir shard antes de comenzar; o
2. diferir physical begin hasta resolverlo,

dependiendo del diseño final del transaction/routing integration.

---

# 116. Preferred architecture

Cuando el shard sea conocido:

```php
DB::onShard($shard)->transaction(function () {
    // ...
});
```

---

# 117. Transaction shard affinity

Después de iniciar:

```text
Shard(T) = immutable
```

---

# 118. Cross-shard query in transaction

Debe fallar.

---

# 119. Cross-shard flush

Debe detectarse antes de ejecutar parcialmente cuando sea posible.

---

# 120. UnitOfWork grouping

Persistence Planner podrá agrupar:

```text
PersistenceOperations
    ↓
ExecutionDomain
```

---

# 121. Multiple domains

Si aparecen:

```text
Shard A
Shard B
```

y se requiere atomicidad:

```text
reject
```

por default.

---

# 122. No hidden distributed flush

Nunca:

```text
flush()
→ transaction shard A
→ transaction shard B
→ pretend atomic
```

---

# 123. Relationships

La shard key debe diseñarse considerando relaciones frecuentes.

---

# 124. Co-location

Idealmente entidades altamente relacionadas pueden compartir shard key.

Ejemplo:

```text
Customer
Orders
Invoices
```

por:

```text
customer_id
```

---

# 125. Co-location advantage

Permite:

- local joins;
- local transactions;
- local FK;
- menor fan-out.

---

# 126. Cross-shard relationships

Deben ser explícitas.

---

# 127. Foreign keys

Una FK física normalmente solo puede proteger relaciones dentro de un DB domain compatible.

VoltStack no fingirá FK cross-shard.

---

# 128. Many-to-many

Una M2M cross-shard puede requerir:

- dedicated association shard;
- duplicated references;
- application-level consistency;
- specialized service.

No será comportamiento implícito.

---

# 129. Cascades

Cascade persist/remove no deberá cruzar shards silenciosamente.

---

# 130. Orphan removal

Misma regla.

---

# 131. Lazy loading

Una relación cross-shard, si está soportada, deberá producir un nuevo routing operation explícitamente observable.

---

# 132. N+1

Sharding puede amplificar N+1.

Ejemplo:

```text
100 entities
→ 20 shards
→ repeated cross-shard lazy loads
```

N+1 telemetry deberá conservar shard context.

---

# 133. Batch loading

Batch relation loader deberá agrupar por:

```text
ShardId
+
RelationshipId
+
ExecutionDomain
```

---

# 134. Query cache

Cache keys sharded deberán incorporar shard identity cuando corresponda.

---

# 135. Result cache

Nunca compartir resultados entre shards accidentalmente.

---

# 136. Entity cache

Entity cache deberá utilizar distributed EntityKey.

---

# 137. Schema

Todos los shards de una misma logical entity family deberían normalmente compartir schema compatible.

---

# 138. Schema fingerprint

Podrá existir:

```text
ShardSchemaFingerprint
```

para verificar uniformidad.

---

# 139. Schema drift

Debe ser diagnosticable.

---

# 140. Migration architecture

Una migration sobre sharded database puede convertirse en:

```text
Migration
    ↓
Shard Migration Plan
    ↓
Shard 1
Shard 2
Shard 3
...
```

---

# 141. Migration ordering

Podrá ejecutarse:

- sequential;
- bounded parallel;
- rolling;
- canary-first.

---

# 142. Migration completion

Debe distinguir:

```text
LOCAL_COMPLETE
GLOBAL_COMPLETE
PARTIAL
FAILED
UNKNOWN
```

---

# 143. Zero downtime

Puede utilizar:

```text
expand
deploy compatible app
migrate shards
backfill
verify
contract
```

---

# 144. Canary shard

Puede migrarse primero un subconjunto.

---

# 145. Migration budget

Número de shards concurrentes deberá estar limitado.

---

# 146. Shard splitting

Ejemplo:

```text
Shard A
range [0, 1000)
```

se convierte:

```text
Shard A
[0, 500)

Shard B
[500, 1000)
```

---

# 147. Split lifecycle

Conceptualmente:

```text
PLAN
 ↓
CREATE TARGET
 ↓
COPY
 ↓
CATCH UP
 ↓
VERIFY
 ↓
CUTOVER
 ↓
DRAIN SOURCE RANGE
 ↓
COMPLETE
```

---

# 148. No instant split assumption

Actualizar `ShardMap` sin mover datos no completa un split.

---

# 149. Shard merging

Ejemplo:

```text
Shard A [0,500)
Shard B [500,1000)
```

→

```text
Shard C [0,1000)
```

---

# 150. Merge lifecycle

También requiere:

- copy;
- synchronization;
- verification;
- cutover;
- cleanup.

---

# 151. Shard movement

Mover un shard completo:

```text
ReplicationGroup A
→
ReplicationGroup B
```

puede preservar `ShardId`.

---

# 152. Shard relocation

Por tanto:

```text
Shard Identity
≠
Shard Placement
```

---

# 153. Rebalancing

Busca redistribuir:

- storage;
- traffic;
- CPU;
- write load;
- read load;
- tenant concentration.

---

# 154. Rebalancing ≠ load balancing

Load balancing:

```text
request → replica
```

Rebalancing:

```text
data ownership → shard placement
```

---

# 155. Hot shard

Un shard puede convertirse en hotspot.

---

# 156. HotShardSignal

```php
final readonly class HotShardSignal
{
    public function __construct(
        public ShardId $shard,
        public ShardLoadProfile $load,
        public HotShardConfidence $confidence,
    ) {}
}
```

---

# 157. Hot shard dimensions

Podrán incluir:

- QPS;
- writes/sec;
- reads/sec;
- CPU;
- storage;
- connection pressure;
- latency;
- lock contention;
- growth rate.

---

# 158. Hot shard ≠ unhealthy shard

Un shard puede estar:

```text
HOT
```

pero todavía:

```text
HEALTHY
```

---

# 159. Detection ≠ automatic movement

VoltStack podrá detectar/recomendar.

No deberá mover datos automáticamente sin policy/control plane.

---

# 160. Shard skew

Distribución:

```text
Shard A → 5%
Shard B → 5%
Shard C → 80%
Shard D → 10%
```

puede indicar skew.

---

# 161. Key distribution analysis

Telemetry podrá ayudar a identificar shard key deficiente.

---

# 162. Cardinality

Una buena shard key normalmente necesita cardinalidad suficiente.

---

# 163. Low-cardinality danger

Ejemplo:

```text
country
```

con pocos valores puede producir shards muy desiguales.

---

# 164. Temporal hot spots

Usar:

```text
created_at
```

como range shard key puede concentrar nuevas escrituras.

---

# 165. Tenant-based sharding

Puede ser útil:

```text
hash(tenant_id)
```

pero Database core no dependerá de Multitenancy.

---

# 166. Generic domain key

Core verá:

```text
ShardKey
```

no:

```text
TenantModel
```

---

# 167. Large tenant problem

Un tenant muy grande puede dominar un shard.

---

# 168. Future strategies

Podrá soportarse:

```text
tenant
+
bucket
```

para subdividir grandes tenants.

---

# 169. Dedicated shard

Directory sharding puede permitir:

```text
large tenant
→ dedicated shard
```

sin alterar otros tenants.

---

# 170. Shard capacity

```php
final readonly class ShardCapacityProfile
{
    public function __construct(
        public CapacityClass $class,
        public ?StorageCapacity $storage,
        public ?ThroughputCapacity $throughput,
    ) {}
}
```

---

# 171. Capacity ≠ load

Igual que en Load Balancing:

```text
capacity
≠
current utilization
```

---

# 172. Shard metadata

Podrá incluir:

```text
tags
region
data classification
capacity class
schema generation
creation generation
```

---

# 173. Security classification

Algunos datos podrían requerir placement específico.

Ejemplo:

```text
DataResidencyRequirement
```

---

# 174. Residency constraint

Sharding strategy no podrá mover datos a una región incompatible con security/compliance policy.

---

# 175. Placement policy

```php
interface ShardPlacementPolicy
{
    public function validate(
        ShardDefinition $shard,
        ShardPlacement $placement,
    ): ShardPlacementValidationResult;
}
```

---

# 176. Shard selection vs placement

Assignment:

```text
key → shard
```

Placement:

```text
shard → replication group/infrastructure
```

Routing:

```text
operation → shard → endpoint
```

---

# 177. Three-level model

```text
DATA
 ↓
SHARD
 ↓
PLACEMENT
 ↓
ENDPOINT
```

---

# 178. ShardMapProvider

```php
interface ShardMapProvider
{
    public function current(
        ShardMapId $id
    ): ShardMap;
}
```

---

# 179. Static shard map

Adecuado para:

- simple deployments;
- fixed ranges;
- fixed hash buckets.

---

# 180. Dynamic shard map

Adecuado para:

- rebalancing;
- directory sharding;
- virtual shards;
- large clusters.

---

# 181. No remote lookup per query by default

Dynamic maps deberán poder publicarse/cachearse como snapshots.

---

# 182. Control plane integration

```text
Control Plane
    │
    ▼
Shard Map Provider
    │
    ▼
Immutable ShardMap
    │
    ▼
Data Plane
```

---

# 183. Data plane

No deberá mutar ownership arbitrariamente.

---

# 184. Ownership changes

Deben venir de:

```text
validated control-plane transition
```

---

# 185. ShardMap transition

```php
final readonly class ShardMapTransition
{
    public function __construct(
        public ShardMapGeneration $from,
        public ShardMapGeneration $to,
        public ShardTransitionPlan $plan,
    ) {}
}
```

---

# 186. Transition validation

Debe garantizar que el nuevo map no produzca:

- unintended gaps;
- overlapping stable ownership;
- unknown targets;
- invalid state transitions.

---

# 187. Cutover

Durante cutover deberá existir una definición clara de:

```text
authoritative shard ownership
```

---

# 188. No ambiguous ownership

Dos shards no deberán considerarse simultáneamente authoritative para la misma key salvo un protocolo explícito.

---

# 189. Ownership conflict

Debe producir:

```text
ShardOwnershipConflictException
```

---

# 190. Routing during movement

Puede requerir:

```text
source
target
ownership epoch
movement phase
```

El documento 185 definirá la resolución operativa.

---

# 191. Redirect support

Un shard antiguo podría responder conceptualmente:

```text
MOVED_TO shard-B
```

si la infraestructura/adaptador lo soporta.

---

# 192. Redirect safety

Un redirect deberá validarse contra topology/shard map.

No seguir cadenas arbitrarias sin límite.

---

# 193. Redirect budget

Debe existir:

```text
maxShardRedirects
```

---

# 194. Stale client map

Redirect puede ayudar a detectar:

```text
stale ShardMapGeneration
```

---

# 195. ShardHint

Queries o entidades podrán transportar un hint.

---

# 196. Hint contract

```php
final readonly class ShardHint
{
    public function __construct(
        public ShardId $shard,
        public ?ShardMapGeneration $generation,
    ) {}
}
```

---

# 197. Hint validation

Hint nunca sustituye correctness.

---

# 198. Explicit override

Administradores pueden necesitar:

```php
DB::onShard('shard-12')
```

pero seguirá sujeto a safety rules.

---

# 199. Scatter-gather readiness

Cuando no exista single shard assignment:

```text
PartitionScope
=
MULTIPLE
```

o:

```text
GLOBAL
```

Partition Routing decidirá fan-out.

---

# 200. Sharding System boundary

Este sistema solo debe poder responder:

```text
What shards exist?
How is data assigned?
What shard owns this key?
What is the current shard-map generation?
```

---

# 201. Partition Routing boundary

El siguiente sistema responderá:

```text
Given this query/operation,
which shard or shards must execute it?
```

---

# 202. Shard statistics

```php
final readonly class ShardStatistics
{
    public function __construct(
        public ShardId $shard,
        public ?int $estimatedRows,
        public ?StorageSize $storage,
        public ?ShardLoadProfile $load,
        public Instant $observedAt,
    ) {}
}
```

---

# 203. Statistics ≠ truth

Estimaciones deberán declarar:

- source;
- timestamp;
- confidence.

---

# 204. Statistics use

Podrán alimentar:

- diagnostics;
- rebalancing recommendations;
- hot-shard detection;
- planner estimates.

---

# 205. No assignment based on transient load by default

Cambiar:

```text
key → shard
```

por load momentáneo rompe estabilidad de ubicación.

---

# 206. Stable ownership principle

Shard assignment debe ser estable hasta una transición explícita.

---

# 207. Load balancing vs data balancing

```text
Request balancing
```

puede cambiar cada query.

```text
Data balancing
```

requiere movimiento de estado.

---

# 208. ShardMap cache

Map compilado puede compartirse entre workers si immutable.

---

# 209. Cache invalidation

Nueva generation:

```text
G+1
```

invalida/reemplaza:

```text
G
```

para nuevas operaciones.

---

# 210. In-flight operation

Puede continuar con `G` o revalidar según policy.

---

# 211. Write revalidation

Writes durante ownership transitions deberán ser más estrictos.

---

# 212. Read revalidation

Puede permitir más flexibilidad si source todavía sirve reads válidos.

---

# 213. Shard fencing

Cuando infraestructura lo soporte:

```text
ShardOwnershipEpoch
```

puede utilizarse como fencing token.

---

# 214. Fencing limitation

VoltStack no fabricará atomic fencing si el backend no lo soporta.

---

# 215. Security

Shard identifiers provenientes de HTTP/input externo no deberán confiarse directamente.

---

# 216. User-controlled shard selection

Ejemplo peligroso:

```text
?shard=customer-secret-shard
```

No debe otorgar acceso.

---

# 217. Routing ≠ authorization

Incluso un shard correcto requiere:

```text
Authorization
```

independiente.

---

# 218. Shard metadata exposure

Diagnostics públicos no deberán revelar topology sensible.

---

# 219. Credentials

ShardMap nunca deberá almacenar passwords en metadata diagnosticable.

---

# 220. Query audit

Audit podrá registrar:

```text
logical database
shard
operation type
routing reason
```

sin valores sensibles innecesarios.

---

# 221. Telemetry events

```text
ShardAssignmentStarted
ShardAssigned
ShardAssignmentFailed
ShardMapPublished
ShardMapRejected
ShardStateChanged
ShardHotspotDetected
ShardMovementStarted
ShardMovementCutover
ShardMovementCompleted
ShardOwnershipConflictDetected
```

---

# 222. Metrics

Ejemplos:

```text
db.sharding.assignments
db.sharding.assignment_duration
db.sharding.map_generation_age
db.sharding.map_refresh_failures
db.sharding.hot_shards
db.sharding.redirects
db.sharding.routing_mismatches
```

---

# 223. Cardinality governance

`ShardId` puede tener cardinalidad elevada.

No deberá ser metric label universal por default.

---

# 224. Diagnostics API

Conceptualmente:

```php
DB::sharding()->inspect();
```

---

# 225. Example diagnostics

```text
DATABASE SHARDING

Logical Database:
    main

Shard Map:
    customer-data

Generation:
    48

Strategy:
    VIRTUAL_NODE

Virtual Shards:
    1024

Physical Shards:
    4

Shard A:
    virtual shards: 256
    state: ACTIVE
    replication group: rg-a

Shard B:
    virtual shards: 256
    state: ACTIVE
    replication group: rg-b

Shard C:
    virtual shards: 320
    state: ACTIVE
    replication group: rg-c

Shard D:
    virtual shards: 192
    state: ACTIVE
    replication group: rg-d

Warnings:
    shard-c load above expected distribution
```

---

# 226. Explain assignment

```php
DB::sharding()->explain($customerId);
```

Ejemplo:

```text
SHARD ASSIGNMENT

Shard Map:
    customer-data

Generation:
    48

Shard Key:
    customer_id

Canonical Value:
    [redacted/hash]

Strategy:
    VIRTUAL_NODE

Hash Algorithm:
    voltstack.xxhash.v1

Hash:
    [diagnostic-safe]

Virtual Shard:
    738

Owner:
    shard-c

Ownership Epoch:
    91

State:
    STABLE
```

---

# 227. Sensitive shard keys

Raw shard key podrá ser:

```text
REDACTED
HASHED
OMITTED
```

según telemetry policy.

---

# 228. Error hierarchy

```text
DatabaseShardingException
├── ShardConfigurationException
├── ShardMapException
│   ├── ShardMapUnavailableException
│   ├── ShardMapValidationException
│   ├── ShardMapStaleException
│   └── ShardMapGenerationException
│
├── ShardKeyException
│   ├── ShardKeyRequiredException
│   ├── ShardKeyInvalidException
│   ├── ShardKeyNormalizationException
│   └── ShardKeyMutationException
│
├── ShardAssignmentException
│   ├── ShardNotFoundException
│   ├── ShardUnassignedKeyException
│   ├── ShardAssignmentAmbiguousException
│   └── ShardAssignmentStrategyException
│
├── ShardOwnershipException
│   ├── ShardOwnershipConflictException
│   ├── ShardOwnershipMismatchException
│   └── ShardOwnershipEpochException
│
├── ShardStateException
├── CrossShardTransactionException
├── CrossShardRelationshipException
├── ShardMovementException
├── ShardRedirectLimitExceededException
└── ShardingInvariantViolationException
```

---

# 229. Testing architecture

Suite propuesta:

```text
ShardIdTests
ShardKeyTests
ShardKeyNormalizationTests
ShardMapTests
ShardMapGenerationTests
HashShardingTests
RangeShardingTests
DirectoryShardingTests
CompositeShardingTests
VirtualShardTests
ShardOwnershipTests
ShardStateTests
ShardPlacementTests
ShardMapTransitionTests
ShardMovementTests
ShardSplitTests
ShardMergeTests
HotShardDetectionTests
ShardORMIntegrationTests
ShardTransactionIntegrationTests
ShardMigrationIntegrationTests
ShardPersistentRuntimeTests
ShardConcurrencyTests
ShardTelemetryTests
ShardDiagnosticsTests
```

---

# 230. Deterministic hash test

Misma key deberá producir mismo hash bajo misma algorithm version.

---

# 231. Cross-process hash test

Resultado deberá ser igual entre procesos.

---

# 232. Cross-runtime hash test

Debe verificarse entre runtimes soportados cuando corresponda.

---

# 233. Range boundary tests

Probar:

```text
lower
upper - 1
upper
```

según semántica.

---

# 234. Gap test

Range sin owner deberá producir error explícito.

---

# 235. Overlap test

Dos owners stable para mismo range deberán invalidar map.

---

# 236. Directory test

Lookup deberá respetar ownership epoch.

---

# 237. Virtual shard test

Cada virtual shard deberá tener owner válido.

---

# 238. Map immutability test

Published map no podrá mutarse.

---

# 239. Atomic map publication test

Concurrent readers observarán una generación completa.

---

# 240. Shard key mutation test

Managed entity no podrá cambiar shard key silenciosamente.

---

# 241. IdentityMap test

Mismo ID en shards diferentes no colisionará.

---

# 242. Transaction test

Active transaction no cambiará shard.

---

# 243. Cross-shard flush test

Deberá detectarse antes de partial persistence siempre que sea posible.

---

# 244. Relationship test

Cross-shard cascade será rechazado salvo soporte explícito.

---

# 245. Migration test

Global completion deberá distinguir partial shard migration.

---

# 246. Movement test

Durante cutover no deberá existir ambiguous stable ownership.

---

# 247. Stale map test

Old generation deberá ser detectable.

---

# 248. Redirect loop test

```text
A → B → A
```

deberá detenerse.

---

# 249. Persistent runtime test

Miles de requests no deberán acumular:

- old shard maps;
- routing hints;
- entity shard contexts;
- redirect history.

---

# 250. OpenSwoole concurrency test

Concurrent coroutines deberán compartir únicamente immutable maps/concurrency-safe caches.

---

# 251. Directory structure

```text
src/Quantum/Database/Sharding/
│
├── Shard.php
├── ShardId.php
├── ShardDefinition.php
├── ShardState.php
├── ShardMetadata.php
├── ShardPlacement.php
│
├── Key/
│   ├── ShardKey.php
│   ├── CanonicalShardKey.php
│   ├── ShardKeyId.php
│   ├── ShardKeyDefinition.php
│   ├── ShardKeyType.php
│   ├── ShardKeyNormalizer.php
│   └── CompositeShardKey.php
│
├── Map/
│   ├── ShardMap.php
│   ├── ShardMapId.php
│   ├── ShardMapGeneration.php
│   ├── ShardMapProvider.php
│   ├── ShardMapValidator.php
│   ├── ShardMapCompiler.php
│   └── ShardMapTransition.php
│
├── Assignment/
│   ├── ShardAssignment.php
│   ├── ShardAssignmentReason.php
│   ├── ShardAssignmentStrategy.php
│   └── ShardingStrategy.php
│
├── Hash/
│   ├── HashShardingStrategy.php
│   ├── ShardHasher.php
│   ├── ShardHash.php
│   └── HashAlgorithmId.php
│
├── Range/
│   ├── RangeShardingStrategy.php
│   ├── ShardRange.php
│   └── ShardKeyBoundary.php
│
├── Directory/
│   ├── DirectoryShardingStrategy.php
│   ├── ShardDirectory.php
│   ├── ShardDirectoryEntry.php
│   └── ShardDirectoryCache.php
│
├── Virtual/
│   ├── VirtualShardId.php
│   ├── VirtualShardMap.php
│   ├── VirtualShardAssignment.php
│   └── VirtualNodeShardingStrategy.php
│
├── Ownership/
│   ├── ShardOwnership.php
│   ├── ShardOwnershipState.php
│   ├── ShardOwnershipEpoch.php
│   └── ShardOwnershipValidator.php
│
├── Transition/
│   ├── ShardTransitionPlan.php
│   ├── ShardSplitPlan.php
│   ├── ShardMergePlan.php
│   ├── ShardMovementPlan.php
│   └── ShardRebalancingPlan.php
│
├── Statistics/
│   ├── ShardStatistics.php
│   ├── ShardLoadProfile.php
│   ├── HotShardSignal.php
│   └── HotShardDetector.php
│
├── ORM/
│   ├── EntityShardMetadata.php
│   ├── EntityShardResolver.php
│   └── ShardKeyMutationGuard.php
│
├── Placement/
│   ├── ShardPlacementPolicy.php
│   └── ShardPlacementValidator.php
│
├── Diagnostics/
│   ├── ShardingInspector.php
│   ├── ShardAssignmentExplainer.php
│   └── ShardMapDiagnosticReport.php
│
├── Telemetry/
│   ├── ShardingTelemetry.php
│   ├── ShardAssigned.php
│   ├── ShardMapPublished.php
│   └── ShardHotspotDetected.php
│
└── Exception/
    ├── DatabaseShardingException.php
    ├── ShardConfigurationException.php
    ├── ShardMapException.php
    ├── ShardKeyException.php
    ├── ShardAssignmentException.php
    ├── ShardOwnershipException.php
    ├── ShardStateException.php
    ├── CrossShardTransactionException.php
    └── ShardingInvariantViolationException.php
```

---

# 252. Invariantes arquitectónicos

## DB-SHARD-001
Shard será distinto de server.

## DB-SHARD-002
Shard será distinto de database endpoint.

## DB-SHARD-003
Shard será distinto de replication group.

## DB-SHARD-004
Sharding será distinto de replication.

## DB-SHARD-005
Sharding será distinto de partition routing.

## DB-SHARD-006
Sharding será distinto de load balancing.

## DB-SHARD-007
Sharding será distinto de multitenancy.

## DB-SHARD-008
Sharding será distinto de physical table partitioning.

## DB-SHARD-009
ShardId será estable.

## DB-SHARD-010
ShardId no dependerá de IP por default.

## DB-SHARD-011
Shard identity será distinta de shard placement.

## DB-SHARD-012
ShardKey será distinta de primary key.

## DB-SHARD-013
ShardKey podrá coincidir con primary key, pero no se asumirá.

## DB-SHARD-014
Composite shard key tendrá ordering definido.

## DB-SHARD-015
Shard key será normalizada antes de assignment.

## DB-SHARD-016
Shard key normalization será determinista.

## DB-SHARD-017
Shard key hashing será determinista.

## DB-SHARD-018
Hashing no dependerá de process-randomized state.

## DB-SHARD-019
Hash algorithm tendrá identidad/version.

## DB-SHARD-020
Cambiar hash semantics será migration concern.

## DB-SHARD-021
MISSING shard key será distinto de NULL shard key.

## DB-SHARD-022
NULL shard key tendrá policy explícita.

## DB-SHARD-023
Missing required shard key no producirá random routing.

## DB-SHARD-024
ShardAssignment será distinta de endpoint routing.

## DB-SHARD-025
ShardAssignment no elegirá replica.

## DB-SHARD-026
ShardAssignment no adquirirá ConnectionLease.

## DB-SHARD-027
Hash sharding podrá soportarse.

## DB-SHARD-028
Range sharding podrá soportarse.

## DB-SHARD-029
Directory sharding podrá soportarse.

## DB-SHARD-030
Composite sharding podrá soportarse.

## DB-SHARD-031
Virtual shards podrán soportarse.

## DB-SHARD-032
Range boundaries serán explícitas.

## DB-SHARD-033
Stable ranges no deberán solaparse.

## DB-SHARD-034
Unassigned range no será enviado arbitrariamente.

## DB-SHARD-035
Directory mapping tendrá version/epoch cuando sea necesario.

## DB-SHARD-036
Stale directory entries serán detectables.

## DB-SHARD-037
Virtual shard será distinto de physical shard.

## DB-SHARD-038
Todo virtual shard activo tendrá owner.

## DB-SHARD-039
ShardMap será immutable.

## DB-SHARD-040
ShardMap tendrá generation.

## DB-SHARD-041
ShardMapGeneration será distinta de topology generation.

## DB-SHARD-042
ShardAssignment conservará map generation.

## DB-SHARD-043
Published ShardMap no mutará.

## DB-SHARD-044
ShardMap publication será atomic.

## DB-SHARD-045
Invalid map no reemplazará valid map.

## DB-SHARD-046
Ownership será distinta de placement.

## DB-SHARD-047
Ownership será distinta de endpoint authority.

## DB-SHARD-048
ShardOwnershipEpoch podrá utilizarse para stale-routing detection.

## DB-SHARD-049
UNKNOWN ownership permanecerá UNKNOWN.

## DB-SHARD-050
Ambiguous stable ownership deberá fallar cerrado.

## DB-SHARD-051
Shard state será distinta de endpoint health.

## DB-SHARD-052
UNKNOWN shard state no equivaldrá a ACTIVE.

## DB-SHARD-053
DRAINING shard no recibirá nuevas ownership assignments normales.

## DB-SHARD-054
READ_ONLY shard no aceptará ordinary writes.

## DB-SHARD-055
MOVING state será explícito.

## DB-SHARD-056
SPLITTING state será explícito.

## DB-SHARD-057
MERGING state será explícito.

## DB-SHARD-058
Shard placement referenciará infrastructure válida.

## DB-SHARD-059
Shard resolver se ejecutará antes de replica load balancing.

## DB-SHARD-060
Load balancer no cambiará shard.

## DB-SHARD-061
Failover dentro de replication group no cambiará shard identity.

## DB-SHARD-062
Entity shard metadata será compilada.

## DB-SHARD-063
ORM attributes no serán runtime source of truth después de compilation.

## DB-SHARD-064
New sharded entity requerirá shard key antes de persistence cuando sea necesaria.

## DB-SHARD-065
Shard key mutation no será ordinary field mutation.

## DB-SHARD-066
Managed entity no cambiará shard silenciosamente.

## DB-SHARD-067
Entity relocation será explícita.

## DB-SHARD-068
Entity relocation será distinta de UPDATE.

## DB-SHARD-069
EntityKey preservará shard identity cuando sea parte del identity domain.

## DB-SHARD-070
Mismo local ID en distintos shards no colisionará.

## DB-SHARD-071
Global ID no eliminará shard routing requirements.

## DB-SHARD-072
ShardHint será distinto de verified shard assignment.

## DB-SHARD-073
Stale hint podrá ser rechazado.

## DB-SHARD-074
Repository podrá aportar shard-key semantics.

## DB-SHARD-075
Semantic Query Engine podrá inferir shard constraints.

## DB-SHARD-076
SQL Compiler no decidirá shard.

## DB-SHARD-077
Driver no decidirá shard.

## DB-SHARD-078
Connection no decidirá shard.

## DB-SHARD-079
Raw SQL no dependerá de inferencia insegura.

## DB-SHARD-080
Explicit shard API seguirá security/domain constraints.

## DB-SHARD-081
Active transaction tendrá shard affinity.

## DB-SHARD-082
Active transaction no cambiará shard.

## DB-SHARD-083
Cross-shard transaction será rechazada por default.

## DB-SHARD-084
Cross-shard UnitOfWork no implicará atomic flush.

## DB-SHARD-085
Persistence Planner detectará múltiples domains cuando sea posible.

## DB-SHARD-086
VoltStack no fingirá distributed atomicity.

## DB-SHARD-087
Co-location será una consideración de diseño de shard key.

## DB-SHARD-088
Cross-shard relationship será explícita.

## DB-SHARD-089
Physical FK cross-shard no será fingida.

## DB-SHARD-090
Cross-shard cascade estará prohibido por default.

## DB-SHARD-091
Cross-shard orphan removal estará prohibido por default.

## DB-SHARD-092
Batch relationship loading agrupará por shard.

## DB-SHARD-093
N+1 telemetry podrá conservar shard context.

## DB-SHARD-094
Query cache preservará shard domain.

## DB-SHARD-095
Entity cache preservará shard identity.

## DB-SHARD-096
Shard schema compatibility será verificable.

## DB-SHARD-097
Schema drift será observable.

## DB-SHARD-098
Migration sharded tendrá per-shard outcome.

## DB-SHARD-099
Partial shard migration no será global completion.

## DB-SHARD-100
Migration concurrency será bounded.

## DB-SHARD-101
Shard split será una transición, no simple map edit.

## DB-SHARD-102
Shard merge será una transición, no simple map edit.

## DB-SHARD-103
Shard movement preservará ShardId cuando solo cambie placement.

## DB-SHARD-104
Rebalancing será distinto de load balancing.

## DB-SHARD-105
Hot shard será distinto de unhealthy shard.

## DB-SHARD-106
Hot-shard detection no ejecutará automatic movement por default.

## DB-SHARD-107
Shard statistics declararán freshness.

## DB-SHARD-108
Shard statistics declararán confidence cuando corresponda.

## DB-SHARD-109
Transient load no cambiará ownership automáticamente.

## DB-SHARD-110
Shard ownership será estable hasta transición explícita.

## DB-SHARD-111
TenantId no será automáticamente ShardId.

## DB-SHARD-112
TenantId podrá utilizarse como shard key mediante integración.

## DB-SHARD-113
Database core no dependerá de Multitenancy package.

## DB-SHARD-114
Large-tenant strategy podrá extenderse sin romper core.

## DB-SHARD-115
Shard capacity será distinta de current load.

## DB-SHARD-116
Data residency podrá restringir placement.

## DB-SHARD-117
Placement optimization no violará security policy.

## DB-SHARD-118
Assignment será distinta de placement.

## DB-SHARD-119
Placement será distinto de endpoint selection.

## DB-SHARD-120
Dynamic shard maps se consumirán como snapshots.

## DB-SHARD-121
Data plane no mutará shard ownership arbitrariamente.

## DB-SHARD-122
Ownership changes provendrán de transición validada.

## DB-SHARD-123
ShardMapTransition tendrá from/to generations.

## DB-SHARD-124
Cutover tendrá ownership authority definida.

## DB-SHARD-125
Dual stable ownership será inválida salvo protocolo explícito.

## DB-SHARD-126
DUAL_WRITE no será default migration mechanism.

## DB-SHARD-127
Redirect será bounded.

## DB-SHARD-128
Redirect loop será detectado.

## DB-SHARD-129
Redirect no saltará topology validation.

## DB-SHARD-130
Explicit shard override no saltará tenant constraints.

## DB-SHARD-131
Explicit shard override no saltará transaction constraints.

## DB-SHARD-132
Explicit shard override no saltará authorization.

## DB-SHARD-133
UNKNOWN partition no significará all shards.

## DB-SHARD-134
Scatter-gather pertenecerá a Partition Routing/Distributed Execution.

## DB-SHARD-135
Sharding System describirá ownership, no fan-out execution.

## DB-SHARD-136
Shard statistics no serán exact truth si son estimadas.

## DB-SHARD-137
Map cache será generation-aware.

## DB-SHARD-138
Old generations serán bounded.

## DB-SHARD-139
In-flight operation podrá conservar generation según policy.

## DB-SHARD-140
Writes podrán requerir ownership revalidation.

## DB-SHARD-141
Reads podrán tener revalidation policy distinta.

## DB-SHARD-142
Fencing solo se afirmará cuando backend lo soporte.

## DB-SHARD-143
User-provided shard IDs no otorgarán autorización.

## DB-SHARD-144
Routing será distinto de authorization.

## DB-SHARD-145
Shard diagnostics serán sanitizados.

## DB-SHARD-146
ShardMap no expondrá credentials.

## DB-SHARD-147
Telemetry será observational.

## DB-SHARD-148
Telemetry no modificará ownership.

## DB-SHARD-149
Metric cardinality será gobernada.

## DB-SHARD-150
Persistent runtime compartirá solo immutable/concurrency-safe sharding state.

## DB-SHARD-151
Request-local shard context no se filtrará entre requests.

## DB-SHARD-152
Static mutable current shard estará prohibido.

## DB-SHARD-153
FrankenPHP worker reuse deberá limpiar scoped shard state.

## DB-SHARD-154
RoadRunner worker reuse deberá limpiar scoped shard state.

## DB-SHARD-155
OpenSwoole deberá aislar coroutine shard contexts.

## DB-SHARD-156
Shard assignment será deterministic bajo misma map generation y key.

## DB-SHARD-157
Misma key y misma generation deberán producir misma logical assignment.

## DB-SHARD-158
Una assignment diferente requerirá cambio explícito de map/ownership semantics.

## DB-SHARD-159
Performance nunca tendrá prioridad sobre correct data ownership.

## DB-SHARD-160
VoltStack nunca enviará una operación a un shard solo porque esté menos cargado.

---

# 253. Modelo formal

Sea el dataset lógico:

```text
D
```

y el conjunto de shards:

```text
S = {S₁, S₂, ..., Sₙ}
```

Una función de asignación:

```text
A : K → S
```

mapea una shard key `K` hacia un shard.

Para una key `k`:

```text
A(k) = Sᵢ
```

Debe cumplirse, para ordinary exclusive sharding:

```text
∃! Sᵢ ∈ S : Owns(Sᵢ, k)
```

es decir:

> existe exactamente un owner lógico estable para la key.

---

# 254. Modelo con virtual shards

```text
H : K → V
```

donde:

```text
V = set of virtual shards
```

y:

```text
P : V → S
```

Entonces:

```text
A(k)
=
P(H(k))
```

Esto desacopla:

```text
key distribution
```

de:

```text
physical shard count
```

---

# 255. Modelo de routing posterior

Sharding produce:

```text
Shard(O)
```

Después:

```text
ReplicationGroup(O)
=
ReplicationGroup(Shard(O))
```

Después:

```text
EligibleEndpoints(O)
⊆
Endpoints(ReplicationGroup(O))
```

Finalmente:

```text
SelectedEndpoint(O)
=
LoadBalancer(
    EligibleEndpoints(O)
)
```

---

# 256. Principio de estabilidad

Para una map generation `G`:

```text
Assign(k, G)
=
S
```

deberá permanecer estable.

Una nueva ubicación:

```text
Assign(k, G+1)
=
S₂
```

deberá representar una transición explícita y verificable.

---

# 257. Arquitectura final

```text
                    LOGICAL DATABASE
                           │
                           ▼
                    SHARDING MODEL
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
        Shard Key Definition       Shard Map
              │                         │
              ▼                         ▼
       Canonical Shard Key        Map Generation
              │                         │
              └────────────┬────────────┘
                           ▼
                  Assignment Strategy
                           │
          ┌────────────────┼─────────────────┐
          ▼                ▼                 ▼
        HASH             RANGE           DIRECTORY
          │                │                 │
          └────────────────┼─────────────────┘
                           ▼
                     SHARD IDENTITY
                           │
                           ▼
                    SHARD OWNERSHIP
                           │
                           ▼
                    SHARD PLACEMENT
                           │
                           ▼
                  REPLICATION GROUP
                           │
                           ▼
                 READ / WRITE ROUTING
                           │
                           ▼
                 REPLICA ELIGIBILITY
                           │
                           ▼
                    LOAD BALANCING
                           │
                           ▼
                    PHYSICAL ENDPOINT
```

---

# 258. Regla maestra final

> **VoltStack tratará el sharding como un problema de propiedad y ubicación lógica de datos, no como una optimización de conexiones. Una shard key deberá producir una asignación estable y explicable bajo una generación concreta del ShardMap; cualquier cambio de ownership será una transición explícita que puede requerir movimiento, sincronización, verificación y cutover de datos.**

La jerarquía será:

```text
Data Identity
      ↓
Shard Key
      ↓
Shard Assignment
      ↓
Shard Ownership
      ↓
Shard Placement
      ↓
Replication Group
      ↓
Endpoint Routing
```

Nunca:

```text
Available Endpoint
      ↓
Lowest Load
      ↓
Put Data There
```

---

# 259. Resultado arquitectónico

Con los documentos 183 y 184 queda definida la diferencia entre:

```text
Distributed Database Architecture
            │
            ├── Replication
            │
            └── Sharding
```

y dentro de sharding:

```text
Data Distribution Definition
        │
        ▼
ShardMap
        │
        ▼
Shard Ownership
```

Todavía falta resolver una pregunta crítica:

> Dada una query, una entidad, una operación ORM o una operación de persistencia, ¿cómo determina VoltStack exactamente qué shard o conjunto de shards debe recibirla?

Esa responsabilidad pertenece exclusivamente al siguiente sistema.

---

# 260. Estado del Bloque 16

Completados:

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
```

Pendiente:

```text
185_DATABASE_PARTITION_ROUTING_SYSTEM.md
```

---

# 261. Siguiente documento

```text
185_DATABASE_PARTITION_ROUTING_SYSTEM.md
```

El siguiente documento cerrará el **Bloque 16 — Read/Write and Distribution** definiendo:

```text
Query / ORM Operation
        │
        ▼
Semantic Constraints
        │
        ▼
Shard Key Extraction
        │
        ▼
Partition Routing Analysis
        │
        ├── SINGLE
        ├── MULTIPLE
        ├── GLOBAL
        └── UNKNOWN
        │
        ▼
Shard Target Set
        │
        ▼
Distributed Execution Plan
```

y establecerá las reglas para:

- shard-key inference;
- routing desde Query AST;
- routing desde Semantic Query Graph;
- routing de `SELECT`;
- routing de `INSERT`;
- routing de `UPDATE`;
- routing de `DELETE`;
- ORM entity routing;
- repository routing;
- explicit shard hints;
- single-shard operations;
- multi-shard operations;
- scatter-gather;
- fan-out budgets;
- `IN (...)` sobre shard keys;
- range routing;
- composite keys;
- joins;
- aggregates;
- pagination;
- transactions;
- UnitOfWork;
- stale ShardMaps;
- ownership transitions;
- redirects;
- routing cache;
- telemetry;
- diagnostics;
- persistent runtimes;
- seguridad.

La regla central del documento 185 será:

```text
Sharding defines where data belongs.

Partition Routing proves where an operation must execute.

Replication determines which copies can serve it.

Load Balancing chooses among those valid copies.
```