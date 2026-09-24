# 183_DATABASE_DISTRIBUTED_DATABASE_ARCHITECTURE.md

# VoltStack Quantum Database
## Distributed Database Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 183 — Distributed Database Architecture  
**Bloque:** 16 — Read/Write and Distribution  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `182_DATABASE_LOAD_BALANCING_SYSTEM.md`  
**Siguiente documento:** `184_DATABASE_SHARDING_SYSTEM.md`

---

# 1. Propósito

`Distributed Database Architecture` define el modelo arquitectónico mediante el cual VoltStack podrá operar sobre bases de datos desplegadas en múltiples:

- servidores;
- endpoints;
- replicas;
- grupos de replicación;
- zonas;
- regiones;
- shards;
- particiones;
- roles de lectura/escritura;
- dominios de fallo.

Este documento no implementa un nuevo protocolo distribuido.

Su función es establecer el **modelo común de distribución** que conecta los sistemas definidos anteriormente:

```text
176 Read/Write Connection System
177 Read/Write Routing System
178 Replica System
179 Replica Lag Awareness System
180 Sticky Connection System
181 Failover System
182 Load Balancing System
```

con los siguientes:

```text
184 Sharding System
185 Partition Routing System
```

La regla central será:

> **VoltStack representará explícitamente la topología, autoridad, replicación, partición, consistencia y ubicación de los datos; nunca asumirá que múltiples endpoints son intercambiables simplemente porque pertenecen a la misma configuración lógica de base de datos.**

Formalmente:

```text
LogicalDatabase
≠
PhysicalDatabaseServer
≠
Endpoint
≠
ReplicationGroup
≠
Shard
≠
Replica
```

---

# 2. Problema fundamental

En una aplicación simple:

```text
Application
    │
    ▼
Database
```

puede parecer suficiente.

En producción distribuida:

```text
Application
    │
    ▼
Logical Database
    │
    ├── Region MX
    │      ├── Primary
    │      ├── Replica A
    │      └── Replica B
    │
    └── Region US
           ├── Replica C
           └── Replica D
```

y posteriormente:

```text
Logical Database
    │
    ├── Shard 01
    │      ├── Writer
    │      └── Replicas
    │
    ├── Shard 02
    │      ├── Writer
    │      └── Replicas
    │
    └── Shard 03
           ├── Writer
           └── Replicas
```

la pregunta:

```text
"¿qué conexión debo usar?"
```

deja de ser suficiente.

VoltStack deberá responder:

```text
¿Qué dominio de datos?
¿Qué shard?
¿Qué grupo de replicación?
¿Qué autoridad?
¿Qué consistencia requiere la operación?
¿Qué endpoints son elegibles?
¿Qué replica tiene suficiente frescura?
¿Existe sticky requirement?
¿Existe una transacción activa?
¿Existe failover?
¿Qué endpoint conviene seleccionar?
```

---

# 3. Principio arquitectónico

La arquitectura distribuida seguirá:

```text
Correctness
    ↓
Data Location
    ↓
Authority
    ↓
Consistency
    ↓
Eligibility
    ↓
Optimization
```

Nunca:

```text
Lowest Latency
    ↓
Select Database
    ↓
Hope Data Is Correct
```

---

# 4. Objetivos

El sistema deberá permitir:

1. representar topologías distribuidas;
2. separar base lógica de infraestructura física;
3. representar clusters;
4. representar replication groups;
5. representar shards;
6. representar replicas;
7. representar roles;
8. representar autoridad;
9. representar regiones y zonas;
10. representar failure domains;
11. mantener topology generations;
12. resolver execution domains;
13. integrar read/write routing;
14. integrar replica awareness;
15. integrar replica lag;
16. integrar sticky consistency;
17. integrar failover;
18. integrar load balancing;
19. preparar sharding;
20. preparar partition routing;
21. mantener transaction affinity;
22. preservar tenant boundaries;
23. detectar cross-domain operations;
24. evitar implicit distributed transactions;
25. ofrecer diagnostics;
26. ofrecer telemetry;
27. soportar persistent runtimes;
28. permitir extensiones.

---

# 5. No objetivos

Esta arquitectura no implementará directamente:

- consensus algorithms;
- Raft;
- Paxos;
- database replication;
- binlog replication;
- WAL replication;
- distributed SQL engine;
- distributed JOIN engine;
- automatic distributed transactions;
- universal two-phase commit;
- distributed lock manager;
- database proxy;
- service mesh;
- database-specific cluster manager.

VoltStack podrá integrarse con infraestructura que implemente estas capacidades.

---

# 6. Distinciones fundamentales

```text
Distributed Database
≠
Replication
≠
Sharding
≠
Partitioning
≠
Load Balancing
≠
Failover
≠
Multitenancy
≠
Distributed Transaction
```

---

# 7. Distributed database

En VoltStack, una base distribuida significa:

> Un dominio lógico de datos cuya ejecución puede involucrar múltiples ubicaciones físicas conocidas por la arquitectura de routing.

No implica necesariamente sharding.

---

# 8. Replication

Replication significa:

```text
same logical dataset
→
multiple physical copies
```

---

# 9. Sharding

Sharding significa:

```text
logical dataset
→
multiple disjoint/partially disjoint data partitions
```

---

# 10. Replication vs sharding

```text
Replication:
    Data A → Server 1
    Data A → Server 2

Sharding:
    Data A → Server 1
    Data B → Server 2
```

Combinados:

```text
Shard A
├── Writer A
├── Replica A1
└── Replica A2

Shard B
├── Writer B
├── Replica B1
└── Replica B2
```

---

# 11. Multitenancy

Multitenancy responde:

```text
¿a qué tenant pertenecen los datos?
```

Sharding responde:

```text
¿en qué partición están físicamente?
```

No son equivalentes.

---

# 12. Distributed transaction

Una operación que toca:

```text
Shard A
+
Shard B
```

no deberá considerarse automáticamente una única transacción ACID.

---

# 13. Regla de transacción

Por defecto:

```text
One VoltStack Transaction
=
One Physical Transaction Resource
```

---

# 14. Arquitectura general

```text
Application / ORM / Query Engine
              │
              ▼
      DatabaseExecutionContext
              │
              ▼
      ExecutionDomainResolver
              │
              ▼
        LogicalDatabase
              │
              ▼
      DistributedTopology
              │
       ┌──────┴───────┐
       ▼              ▼
 Partition Routing   Replication
       │              │
       ▼              ▼
     Shard      ReplicationGroup
       │              │
       └──────┬───────┘
              ▼
       Candidate Endpoints
              │
              ▼
        Read/Write Routing
              │
              ▼
        Consistency Gates
              │
              ├── Lag
              ├── Sticky
              ├── Transaction
              ├── Authority
              └── Health
              │
              ▼
       Eligible Candidates
              │
              ▼
        Load Balancing
              │
              ▼
       Selected Endpoint
              │
              ▼
       Connection Manager
              │
              ▼
        Connection Lease
```

---

# 15. LogicalDatabase

```php
final readonly class LogicalDatabase
{
    public function __construct(
        public LogicalDatabaseId $id,
        public DistributedTopologyId $topology,
        public DatabaseCapabilityProfile $capabilities,
    ) {}
}
```

`LogicalDatabase` representa el recurso conocido por la aplicación.

---

# 16. LogicalDatabase ≠ endpoint

Una configuración:

```php
DB::connection('default');
```

podrá representar:

```text
LogicalDatabase(default)
```

aunque físicamente existan:

```text
writer-1
replica-1
replica-2
replica-3
```

---

# 17. Endpoint

`DatabaseEndpoint` representa un destino físico direccionable.

Conceptualmente:

```php
final readonly class DatabaseEndpoint
{
    public function __construct(
        public EndpointId $id,
        public EndpointAddress $address,
        public EndpointRoleSet $roles,
        public FailureDomain $failureDomain,
        public EndpointMetadata $metadata,
    ) {}
}
```

---

# 18. Endpoint identity

La identidad del endpoint no deberá depender únicamente de:

```text
hostname
```

porque infraestructura dinámica puede cambiar direcciones.

---

# 19. Stable EndpointId

Preferible:

```text
EndpointId
```

estable durante la vida lógica del endpoint.

---

# 20. DistributedTopology

Objeto principal:

```php
final readonly class DistributedDatabaseTopology
{
    public function __construct(
        public DistributedTopologyId $id,
        public TopologyGeneration $generation,
        public LogicalDatabaseId $database,
        public ReplicationGroupMap $replicationGroups,
        public ShardMap $shards,
        public EndpointMap $endpoints,
        public FailureDomainMap $failureDomains,
    ) {}
}
```

---

# 21. Immutable topology snapshot

Cada versión publicada será immutable.

```text
Topology Generation 100
```

no se modifica.

Se publica:

```text
Topology Generation 101
```

---

# 22. Build-and-swap

Modelo:

```text
Discover
   ↓
Build Candidate Topology
   ↓
Validate
   ↓
Compile
   ↓
Publish Atomically
```

---

# 23. No partial topology mutation

Nunca:

```text
remove replica
request runs
add writer
request runs
change shard
```

sobre un objeto compartido mutable.

---

# 24. TopologyGeneration

```php
final readonly class TopologyGeneration
{
    public function __construct(
        public int|string $value,
    ) {}
}
```

---

# 25. Generation semantics

Permite correlacionar:

```text
RoutingDecision
LoadBalancingDecision
FailoverDecision
ReplicaLagSnapshot
```

con la topología usada.

---

# 26. Topology identity

```text
TopologyId
≠
TopologyGeneration
```

Ejemplo:

```text
TopologyId = production-main
Generation = 5021
```

---

# 27. Cluster

`DatabaseCluster` podrá agrupar infraestructura relacionada.

```text
Cluster
├── replication groups
├── shards
├── endpoints
└── failure domains
```

---

# 28. Cluster ≠ replication group

Un cluster puede contener múltiples grupos de replicación.

---

# 29. ReplicationGroup

```php
final readonly class ReplicationGroup
{
    public function __construct(
        public ReplicationGroupId $id,
        public AuthorityDescriptor $authority,
        public EndpointSet $members,
        public ReplicationSemantics $semantics,
    ) {}
}
```

---

# 30. Replication group invariant

Los miembros deberán representar copias de un mismo dominio lógico de datos dentro del alcance definido.

---

# 31. Authority

`AuthorityDescriptor` describe quién posee autoridad para ciertas operaciones.

---

# 32. Authority ≠ role label

Un endpoint configurado:

```text
role = writer
```

no significa necesariamente:

```text
currently authoritative writer
```

---

# 33. Effective authority

Debe derivarse de evidencia/topología efectiva.

---

# 34. Authority state

Conceptualmente:

```php
enum AuthorityState
{
    case AUTHORITATIVE;
    case NON_AUTHORITATIVE;
    case TRANSITIONING;
    case UNKNOWN;
}
```

---

# 35. UNKNOWN authority

Nunca deberá interpretarse como:

```text
probably writer
```

---

# 36. Split-brain safety

Si VoltStack observa evidencia contradictoria sobre múltiples writers donde la topología exige single-writer:

```text
fail closed
```

---

# 37. No winner by latency

Nunca:

```text
two possible primaries
→ choose fastest
```

---

# 38. Replica

Replica representa una copia no necesariamente autoritativa.

---

# 39. Replica state

El documento 178 definió la semántica específica.

Aquí se integra como componente de topology/routing.

---

# 40. Replica lag

El documento 179 aporta:

```text
replication position
lag
freshness
confidence
```

---

# 41. Replica lag ≠ topology

Topology responde:

```text
quién existe
```

Lag responde:

```text
qué tan actualizada está una copia
```

---

# 42. Sticky consistency

El documento 180 agrega continuidad causal/aplicativa cuando una operación previa requiere observar datos suficientemente recientes.

---

# 43. Sticky ≠ endpoint preference

Sticky puede producir:

```text
hard consistency constraint
```

no simplemente:

```text
prefer this server
```

---

# 44. Failover

El documento 181 modifica la autoridad efectiva/topología utilizable ante fallos.

---

# 45. Load balancing

El documento 182 selecciona únicamente entre candidatos ya válidos.

---

# 46. DistributedExecutionDomain

VoltStack necesitará una identidad explícita del dominio físico/lógico de ejecución.

```php
final readonly class DistributedExecutionDomain
{
    public function __construct(
        public LogicalDatabaseId $database,
        public ?ShardId $shard,
        public ?ReplicationGroupId $replicationGroup,
        public ?TenantDomainId $tenant,
    ) {}
}
```

---

# 47. Domain equality

Dos operaciones pertenecen al mismo execution domain únicamente cuando los componentes relevantes son compatibles.

---

# 48. Transaction domain

Al iniciar una transacción:

```text
TransactionContext
    ↓
pins
    ↓
DistributedExecutionDomain
```

---

# 49. Transaction domain immutability

Una transacción activa no podrá cambiar:

```text
database
shard
replication group
```

arbitrariamente.

---

# 50. Cross-domain transaction

Si una operación intenta:

```text
Transaction Domain = Shard A
Query Domain       = Shard B
```

VoltStack deberá rechazarla por defecto.

---

# 51. Error

Ejemplo:

```text
CrossDatabaseExecutionDomainException
```

o:

```text
CrossShardTransactionException
```

---

# 52. No implicit distributed transaction

VoltStack no deberá convertir automáticamente lo anterior en:

```text
2PC
```

---

# 53. Explicit distributed coordination

Si en el futuro se soporta:

```text
Saga
Outbox
2PC Adapter
Distributed Transaction Coordinator
```

deberá ser un subsistema explícito.

---

# 54. FailureDomain

Concepto fundamental:

```php
final readonly class FailureDomain
{
    public function __construct(
        public FailureDomainId $id,
        public FailureDomainType $type,
        public ?FailureDomainId $parent,
    ) {}
}
```

---

# 55. FailureDomainType

```php
enum FailureDomainType
{
    case PROCESS;
    case HOST;
    case RACK;
    case ZONE;
    case REGION;
    case PROVIDER;
    case CUSTOM;
}
```

---

# 56. Failure domains hierarchy

Ejemplo:

```text
Provider
└── Region
    └── Zone
        └── Host
            └── Endpoint
```

---

# 57. Failure domain purpose

Permite razonar sobre:

- redundancy;
- placement;
- failover;
- locality;
- correlated failures.

---

# 58. Different endpoints ≠ independent failures

Dos replicas en:

```text
same host
```

no proporcionan la misma resiliencia que:

```text
different zones
```

---

# 59. Topology metadata

VoltStack podrá conocer:

```text
region
zone
provider
role
capacity
replication group
shard
```

sin conocer detalles internos del proveedor.

---

# 60. Provider-neutral model

Core:

```text
RegionId("mx-north")
```

no:

```text
if AWS...
if DigitalOcean...
if GCP...
```

---

# 61. Provider adapters

Integraciones podrán traducir infraestructura externa hacia el modelo canónico.

---

# 62. Topology source

La topología podrá provenir de:

```text
Static Configuration
Service Discovery
Database Metadata
Cluster Manager
Cloud Provider
Database Proxy
Custom Provider
```

---

# 63. TopologyProvider

```php
interface DistributedTopologyProvider
{
    public function discover(
        LogicalDatabaseId $database
    ): DistributedDatabaseTopology;
}
```

---

# 64. Static provider

Para configuraciones simples:

```php
'database' => [
    'writer' => 'db-primary',
    'readers' => [
        'db-replica-1',
        'db-replica-2',
    ],
];
```

---

# 65. Dynamic provider

Para clusters dinámicos podrá consultar un control plane.

---

# 66. Discovery ≠ routing

Topology Provider descubre.

Routing Engine decide.

---

# 67. Discovery ≠ failover

Topology Provider puede observar cambios.

No necesariamente inicia promociones.

---

# 68. Topology validation

Antes de publicar una generación deberá comprobarse:

- IDs únicos;
- endpoints existentes;
- replication groups válidos;
- shard references válidas;
- authority consistency;
- failure-domain references válidas;
- no ciclos inválidos;
- capability compatibility.

---

# 69. Invalid topology

No deberá reemplazar una topología válida por una nueva generación corrupta.

---

# 70. Last known topology

Policy podrá permitir conservar temporalmente:

```text
LastKnownGoodTopology
```

cuando discovery falle.

---

# 71. Last known good ≠ current truth

Diagnostics deberán mostrar:

```text
STALE_TOPOLOGY
```

o equivalente.

---

# 72. Topology freshness

Cada snapshot deberá incluir:

```text
discoveredAt
publishedAt
```

y opcional:

```text
validUntil
```

---

# 73. Stale topology policy

Podrá ser:

```text
ALLOW_READ_ONLY
ALLOW_WITH_WARNING
FAIL_CLOSED
CUSTOM
```

según criticidad.

---

# 74. Writes and stale topology

Por defecto deberán ser más conservadoras.

---

# 75. Unknown authority + stale topology

No deberá resultar en:

```text
best guess writer
```

---

# 76. Routing pipeline

```text
Query / ORM Operation
        │
        ▼
DatabaseExecutionIntent
        │
        ▼
LogicalDatabase Resolution
        │
        ▼
Partition/Shard Resolution
        │
        ▼
Replication Group Resolution
        │
        ▼
Read/Write Role Resolution
        │
        ▼
Transaction Constraints
        │
        ▼
Consistency Constraints
        │
        ▼
Replica Freshness
        │
        ▼
Sticky Requirements
        │
        ▼
Authority / Failover
        │
        ▼
Health
        │
        ▼
Eligible Candidate Set
        │
        ▼
Load Balancing
        │
        ▼
Endpoint
```

---

# 77. Order matters

Por ejemplo:

```text
load balance
→ then discover wrong shard
```

es arquitectónicamente incorrecto.

Primero debe conocerse la ubicación lógica de los datos.

---

# 78. Partition first

Cuando exista sharding:

```text
Partition Routing
```

deberá ocurrir antes del balancing entre replicas de ese shard.

---

# 79. Example

```text
User ID = 9001
        │
        ▼
ShardResolver
        │
        ▼
Shard 7
        │
        ▼
ReplicationGroup 7
        │
        ├── writer-7
        ├── replica-7a
        └── replica-7b
```

Solo entonces:

```text
Read Routing
+
Lag
+
Sticky
+
Load Balancing
```

---

# 80. Query scope

Una query podrá tener:

```text
SINGLE_PARTITION
MULTI_PARTITION
UNKNOWN_PARTITION
GLOBAL
```

---

# 81. PartitionScope

```php
enum PartitionScope
{
    case SINGLE;
    case MULTIPLE;
    case GLOBAL;
    case UNKNOWN;
}
```

---

# 82. SINGLE

Ruta normal y preferida.

---

# 83. MULTIPLE

Requerirá ejecución coordinada explícita.

---

# 84. GLOBAL

Operaciones administrativas/analíticas podrán abarcar múltiples particiones.

---

# 85. UNKNOWN

No deberá convertirse automáticamente en:

```text
query all shards
```

---

# 86. Scatter-gather

Si se soporta:

```text
scatter
→ execute
→ gather
→ merge
```

deberá ser una operación explícita.

---

# 87. Scatter-gather ≠ normal query

Tiene implicaciones:

- latencia;
- ordenamiento;
- paginación;
- límites;
- agregaciones;
- fallos parciales;
- memoria.

---

# 88. No accidental fan-out

Una query sin shard key no deberá causar automáticamente cientos de queries sin policy.

---

# 89. Resource governance

Toda operación distribuida deberá tener budgets.

---

# 90. DistributedExecutionBudget

```php
final readonly class DistributedExecutionBudget
{
    public function __construct(
        public int $maxPartitions,
        public int $maxEndpoints,
        public int $maxParallelOperations,
        public Duration $timeout,
    ) {}
}
```

---

# 91. Fan-out limit

```text
requested partitions > budget
```

deberá fallar antes de una expansión peligrosa.

---

# 92. Partial failures

En ejecución multi-partition:

```text
Shard A → success
Shard B → success
Shard C → timeout
```

resultado no será simplemente:

```text
SUCCESS
```

---

# 93. Distributed outcome

Podrá modelarse:

```php
enum DistributedExecutionOutcome
{
    case SUCCESS;
    case PARTIAL_SUCCESS;
    case FAILED;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 94. Partial success

No deberá ocultarse.

---

# 95. Query semantics

Una query distribuida puede requerir merge semántico.

---

# 96. ORDER BY

```text
ORDER BY created_at
LIMIT 20
```

sobre 10 shards no equivale a:

```text
LIMIT 20 on each shard
concatenate
```

---

# 97. Distributed ordering

Requerirá:

```text
local candidate results
→ global merge ordering
```

---

# 98. Distributed LIMIT

Puede requerir over-fetch controlado.

---

# 99. Distributed aggregate

```text
COUNT
SUM
MIN
MAX
```

pueden tener merge semantics.

---

# 100. AVG

No debe combinarse como:

```text
average(local averages)
```

sin pesos.

Correctamente:

```text
Σ local_sum
/
Σ local_count
```

---

# 101. Query Planner integration

El Query Planner podrá producir:

```text
DistributedExecutionPlan
```

cuando la query abarque múltiples partitions.

---

# 102. DistributedExecutionPlan

```php
final readonly class DistributedExecutionPlan
{
    public function __construct(
        public PartitionTargetSet $targets,
        public DistributedExecutionStrategy $strategy,
        public ResultMergePlan $merge,
        public DistributedExecutionBudget $budget,
    ) {}
}
```

---

# 103. Planner ≠ executor

Planner describe.

Executor ejecuta.

---

# 104. Distributed executor

Si se implementa:

```text
DistributedQueryExecutor
```

coordinará múltiples ejecuciones locales.

---

# 105. Local execution reuse

Cada target deberá reutilizar:

```text
Query Engine
→ Compiler
→ Executor
→ Connection Manager
```

existentes.

---

# 106. No second SQL engine

La distribución no creará otro compilador SQL.

---

# 107. ORM integration

El ORM deberá conocer el execution domain de cada entidad administrada cuando sea relevante.

---

# 108. EntityKey

En arquitectura distribuida:

```text
EntityKey
```

podrá necesitar incluir domain/shard identity.

---

# 109. Critical invariant

Dos entidades con:

```text
same entity type
same database ID
different shards
```

no son necesariamente la misma identidad ORM.

---

# 110. Distributed EntityKey

Conceptualmente:

```text
EntityKey
=
EntityType
×
CanonicalIdentifier
×
ExecutionDomain
```

---

# 111. IdentityMap integration

IdentityMap no deberá colisionar:

```text
Shard A / User 100
```

con:

```text
Shard B / User 100
```

---

# 112. Tenant integration

Si tenant forma parte del identity domain:

```text
EntityKey
```

también deberá preservar ese boundary.

---

# 113. Entity movement

Mover una entidad entre shards no será simplemente:

```text
UPDATE shard_id
```

---

# 114. Re-sharding

Podrá requerir:

```text
copy
dual-read/write policy
verify
cutover
cleanup
```

---

# 115. Re-sharding belongs elsewhere

Este documento solo establece que topology generations deberán poder representar el cambio.

---

# 116. UnitOfWork

Un `UnitOfWork` que contenga entidades de múltiples shards no implica que `flush()` pueda persistirlas atómicamente.

---

# 117. Flush planning

Persistence Planner deberá detectar:

```text
multiple execution domains
```

antes de asumir una sola transaction.

---

# 118. Default behavior

Para atomic flush:

```text
multiple physical transaction domains
→ reject
```

por defecto.

---

# 119. Explicit distributed persistence

En el futuro podrá ofrecer:

- saga;
- outbox;
- compensating operations;
- explicit distributed coordinator.

Pero nunca implícitamente.

---

# 120. Relationship constraints

Una relación ORM entre entidades de shards distintos requiere semántica especial.

---

# 121. Cross-shard relationship

No podrá asumirse que existe:

```text
physical foreign key
```

entre servidores independientes.

---

# 122. Relationship metadata

Deberá poder expresar:

```text
LOCAL_ONLY
CROSS_PARTITION_ALLOWED
EXTERNAL_REFERENCE
```

si en el futuro se habilita.

---

# 123. Default relationship policy

```text
cross-shard relationship
→ reject unless explicitly supported
```

---

# 124. JOINs

Un SQL JOIN normal solo puede operar dentro de un execution domain compatible con el motor.

---

# 125. Cross-shard JOIN

No deberá enviarse ingenuamente a un único endpoint.

---

# 126. Distributed JOIN

Si algún día se implementa, será una capacidad separada del Query Planner/Distributed Execution Engine.

---

# 127. Schema

Cada shard puede compartir:

```text
same logical schema
```

pero eso deberá verificarse.

---

# 128. Schema generation

Puede existir:

```text
SchemaGeneration
```

por shard/replication group.

---

# 129. Schema drift

```text
Shard A schema v10
Shard B schema v9
```

es una condición operativa relevante.

---

# 130. Schema drift awareness

Distributed routing podrá necesitar conocer compatibilidad mínima.

---

# 131. Migrations

Una migration distribuida puede requerir ejecución:

```text
per shard
```

---

# 132. Migration success

```text
99/100 shards migrated
```

no significa:

```text
migration globally complete
```

---

# 133. Zero-downtime migration

El documento 110 sigue aplicando, pero ahora:

```text
compatibility window
```

debe contemplar múltiples partitions.

---

# 134. Capability heterogeneity

Idealmente un logical cluster utiliza motores compatibles.

Pero VoltStack no deberá asumirlo ciegamente.

---

# 135. Endpoint capabilities

Cada endpoint/platform podrá declarar capabilities.

---

# 136. Distributed capability intersection

Para una operación multi-target:

```text
EffectiveCapabilities
=
Intersection(
    capabilities(target₁),
    ...
    capabilities(targetₙ)
)
```

para operaciones que requieren semántica uniforme.

---

# 137. Capability union is dangerous

No:

```text
if one shard supports feature
→ distributed query supports feature
```

---

# 138. Platform heterogeneity

Ejemplo conceptual:

```text
Shard A → PostgreSQL
Shard B → PostgreSQL
```

simple.

Pero una arquitectura:

```text
Shard A → PostgreSQL
Shard B → MySQL
```

requeriría restricciones mucho mayores.

---

# 139. Core policy

VoltStack deberá favorecer homogeneous distributed execution domains.

---

# 140. Heterogeneous federation

Si se soporta en el futuro, deberá tratarse como:

```text
Federated Database
```

no como sharding ordinario.

---

# 141. Consistency model

Distributed architecture deberá representar requirements explícitos.

---

# 142. ConsistencyRequirement

Podrá incluir:

```text
authoritative
read-your-writes
bounded staleness
replication position
transaction-bound
eventual
```

---

# 143. Consistency ≠ isolation

Transaction isolation describe interacción dentro de una base transaccional.

Distributed consistency describe visibilidad/autoridad entre copias o dominios.

---

# 144. SERIALIZABLE ≠ globally serializable

Una transacción:

```text
SERIALIZABLE on Shard A
```

no garantiza serialización con:

```text
independent transaction on Shard B
```

---

# 145. Replica consistency

Una replica puede ofrecer:

```text
eventual visibility
```

aunque la query local use un transaction isolation fuerte.

---

# 146. CAP terminology caution

VoltStack no deberá etiquetar arbitrariamente cada deployment como:

```text
CP
AP
```

sin modelo formal suficiente.

---

# 147. Availability ≠ successful connection

Un endpoint conectado puede no ser correcto para una operación.

---

# 148. Consistency first

La selección seguirá:

```text
Data Location
→ Authority
→ Consistency
→ Health
→ Optimization
```

---

# 149. Health model

Health será multidimensional.

---

# 150. Health dimensions

Ejemplos:

```text
network reachability
authentication
database availability
read capability
write capability
replication state
resource saturation
```

---

# 151. Healthy ≠ eligible

Una replica perfectamente healthy puede ser demasiado stale.

---

# 152. Eligible ≠ preferred

Una replica válida puede perder contra otra por load balancing.

---

# 153. Preferred ≠ authoritative

Un reader preferido no obtiene autoridad de escritura.

---

# 154. Three-stage model

```text
Healthy?
   ↓
Eligible?
   ↓
Preferred?
```

son preguntas distintas.

---

# 155. Multi-region architecture

VoltStack deberá poder representar:

```text
Region MX
├── Zone A
│   ├── writer
│   └── replica
└── Zone B
    └── replica

Region US
├── Zone A
│   └── replica
└── Zone B
    └── replica
```

---

# 156. Region preference

Read operations podrán preferir:

```text
local region
```

solo después de cumplir consistency requirements.

---

# 157. Region failover

Un region outage puede cambiar authority y routing.

---

# 158. No automatic cross-region writes

El framework no asumirá que cualquier remote replica puede promoverse.

---

# 159. Promotion authority

Corresponderá al cluster manager/failover provider.

---

# 160. Topology event

Cuando promoción sea confirmada:

```text
new topology generation
```

deberá reflejarla.

---

# 161. Control plane

La arquitectura distingue:

```text
Control Plane
```

de:

```text
Data Plane
```

---

# 162. Control plane

Gestiona/descubre:

- topology;
- authority;
- health;
- failover;
- configuration;
- placement.

---

# 163. Data plane

Ejecuta:

- routing;
- endpoint selection;
- connection acquisition;
- query execution.

---

# 164. Separation

```text
Control Plane State
       ↓
Immutable Snapshots
       ↓
Data Plane
```

---

# 165. Data plane should be fast

No deberá consultar un control plane remoto para cada query si puede utilizar snapshots válidos.

---

# 166. Control plane failure

No deberá bloquear necesariamente reads si existe topology snapshot todavía aceptable bajo policy.

---

# 167. Writes more conservative

Para authority-sensitive operations puede requerirse evidencia fresca.

---

# 168. TopologySnapshotRepository

```php
interface TopologySnapshotRepository
{
    public function current(
        LogicalDatabaseId $database
    ): DistributedDatabaseTopology;
}
```

---

# 169. Atomic publication

Readers deberán observar:

```text
Generation N
```

o:

```text
Generation N+1
```

Nunca una mezcla parcial.

---

# 170. RoutingDecision topology binding

Cada decisión deberá registrar:

```text
TopologyGeneration
```

usada.

---

# 171. Stale decision

Si entre routing y lease acquisition cambia la topología, podrá requerirse revalidation.

---

# 172. Revalidation policy

Dependerá del tipo de operación.

---

# 173. Write revalidation

Más estricta para evitar utilizar old authority.

---

# 174. Read revalidation

Puede ser más permisiva si endpoint sigue siendo seguro.

---

# 175. Authority epoch

Además de topology generation podrá existir:

```text
AuthorityEpoch
```

para fencing.

---

# 176. Fencing

Una nueva autoridad puede obtener un epoch/token mayor.

---

# 177. Fencing purpose

Evitar que old writers continúen actuando como autoridad válida.

---

# 178. VoltStack boundary

VoltStack podrá transportar/verificar fencing metadata cuando infraestructura lo soporte.

No inventará un fencing protocol si el DB cluster no lo ofrece.

---

# 179. AuthorityEpoch

```php
final readonly class AuthorityEpoch
{
    public function __construct(
        public int|string $value,
    ) {}
}
```

---

# 180. Epoch monotonicity

Cuando el provider garantice monotonicidad:

```text
newEpoch > oldEpoch
```

podrá utilizarse para invalidar decisiones antiguas.

---

# 181. Endpoint lease validation

Una lease podrá conservar:

```text
endpoint
topologyGeneration
authorityEpoch
```

para diagnostics/revalidation.

---

# 182. Connection pooling

Pools deberán ser endpoint-scoped o distinguir claramente endpoint identity.

---

# 183. Pool ≠ logical database

No mezclar conexiones de:

```text
replica A
replica B
writer
```

en un pool indistinguible.

---

# 184. Pool key

Conceptualmente:

```text
PoolKey
=
LogicalDatabase
×
Endpoint
×
DriverConfiguration
×
SecurityContext
```

---

# 185. Topology removal

Si endpoint desaparece:

```text
pool
```

deberá poder entrar en draining.

---

# 186. Draining

No entregar nuevas leases.

Las leases existentes se gestionarán según seguridad/contexto.

---

# 187. Transaction exception

Una transacción activa no puede simplemente migrarse porque endpoint esté draining.

---

# 188. Endpoint failure mid-transaction

Resultado sigue las reglas de Transaction System:

```text
outcome may become UNKNOWN
```

---

# 189. Failover does not rescue active transaction

Promover otro writer no hace que la transacción previa continúe allí.

---

# 190. Retry after failover

Una nueva transaction attempt podrá resolverse contra nueva authority si retry policy lo permite.

---

# 191. Operation identity

Para retries distribuidos deberá distinguirse:

```text
LogicalOperationId
TransactionOperationId
TransactionId
RoutingOperationId
ExecutionAttemptId
```

---

# 192. Same logical operation

Puede ejecutarse:

```text
Attempt 1 → old writer
Attempt 2 → new writer
```

sin afirmar que ambos attempts son la misma transacción física.

---

# 193. Idempotency

Distributed retries pueden requerir application-level idempotency.

---

# 194. Idempotency ≠ transaction retry

VoltStack no asumirá que una closure transaccional con side effects externos es idempotente.

---

# 195. Outbox

Para eventos externos:

```text
Database Transaction
+
Outbox Record
```

puede proporcionar coordinación local confiable.

---

# 196. Outbox across shards

Cada shard puede necesitar su propio outbox.

---

# 197. Global ordering

Eventos de múltiples shards no tendrán automáticamente un único orden total.

---

# 198. Distributed clocks

VoltStack no asumirá relojes perfectamente sincronizados entre servidores.

---

# 199. Timestamp ordering

```text
created_at
```

no deberá utilizarse como global causal ordering sin garantías adicionales.

---

# 200. Replication position

Cuando exista, puede proporcionar evidencia más fuerte dentro de un replication domain.

---

# 201. Global sequence

Si una aplicación necesita:

```text
globally monotonic sequence
```

deberá utilizar un mecanismo diseñado para ello.

---

# 202. ID generation

Sharded architectures requieren IDs sin colisiones globales o identity domain explícito.

---

# 203. Supported strategies

Podrán incluir:

```text
database-generated IDs per shard
UUID
ULID
application-generated IDs
distributed ID generators
```

---

# 204. Per-shard auto increment

Puede ser válido si:

```text
EntityKey includes shard
```

---

# 205. Global IDs

Simplifican referencias cross-partition pero no eliminan routing requirements.

---

# 206. Query cache

Cache keys deberán incluir suficiente execution-domain identity.

---

# 207. Cache invariant

Resultado de:

```text
Shard A
```

no deberá reutilizarse como:

```text
Shard B
```

por key collision.

---

# 208. Cache key

Podrá incluir:

```text
logical database
shard
query fingerprint
parameters
consistency context
```

según cache semantics.

---

# 209. Cache architecture later

Los documentos 186–192 profundizarán estas reglas.

---

# 210. Metadata cache

Topology generation puede participar en invalidation cuando sea necesario.

---

# 211. Compiled query cache

SQL compilado no necesariamente depende del endpoint si dialect/capabilities son iguales.

---

# 212. Capability fingerprint

Cuando endpoints difieran:

```text
CompiledQueryCacheKey
```

deberá incluir platform capability fingerprint relevante.

---

# 213. Security

Distributed topology contiene información sensible operacional.

---

# 214. Sensitive fields

No deberán exponerse indiscriminadamente:

- credentials;
- raw passwords;
- TLS secrets;
- provider tokens;
- private connection strings.

---

# 215. Diagnostics sanitization

Mostrar:

```text
endpoint-id
region
role
state
```

sin necesidad de revelar:

```text
username/password
```

---

# 216. Tenant security

Tenant context nunca deberá convertirse en un simple load-balancing hint.

---

# 217. Tenant boundary

Si tenant determina database/shard:

```text
TenantResolution
```

ocurre antes del load balancer.

---

# 218. No cross-tenant fallback

Nunca:

```text
tenant database unavailable
→ use another tenant database
```

---

# 219. Shard boundary

Misma regla:

```text
shard unavailable
→ choose another shard
```

es inválido salvo reconfiguration explícita que indique que los datos fueron movidos.

---

# 220. Replica fallback is different

```text
replica A unavailable
→ replica B
```

puede ser válido porque ambas sirven el mismo replication domain.

---

# 221. Data equivalence requirement

Fallback físico requiere demostrar:

```text
same logical data domain
```

con consistency suficiente.

---

# 222. Observability

Toda ejecución distribuida deberá poder responder:

```text
Why this database?
Why this shard?
Why this replication group?
Why this role?
Why this endpoint?
Which topology generation?
Which consistency requirement?
Which failover state?
Which load balancing policy?
```

---

# 223. DistributedRoutingTrace

```php
final readonly class DistributedRoutingTrace
{
    public function __construct(
        public LogicalDatabaseId $database,
        public TopologyGeneration $topologyGeneration,
        public DistributedExecutionDomain $domain,
        public RoutingDecision $routing,
        public ?LoadBalancingDecision $balancing,
    ) {}
}
```

---

# 224. Trace correlation

Deberá correlacionarse con:

```text
QueryOperationId
TransactionOperationId
Request/Job TraceId
```

cuando existan.

---

# 225. Telemetry

Eventos conceptuales:

```text
DistributedTopologyDiscovered
DistributedTopologyPublished
DistributedTopologyRejected
DistributedRoutingStarted
DistributedDomainResolved
DistributedEndpointSelected
DistributedExecutionStarted
DistributedExecutionCompleted
DistributedExecutionPartiallyFailed
DistributedTopologyStale
DistributedAuthorityConflictDetected
```

---

# 226. Telemetry cardinality

No convertir automáticamente:

```text
shard id
tenant id
endpoint id
```

en labels métricos de cardinalidad ilimitada.

---

# 227. Traces vs metrics

Detalles de alta cardinalidad pueden vivir mejor en:

```text
traces
structured diagnostics
```

que en metric labels.

---

# 228. Metrics

Ejemplos:

```text
db.distributed.routing.count
db.distributed.routing.duration
db.distributed.topology.generation_age
db.distributed.topology.refresh_failures
db.distributed.execution.fanout
db.distributed.execution.partial_failures
db.distributed.authority.conflicts
```

---

# 229. Health diagnostics

Conceptualmente:

```php
DB::distributed()->health();
```

---

# 230. Topology diagnostics

```php
DB::distributed()->topology()->inspect();
```

---

# 231. Example

```text
DISTRIBUTED DATABASE TOPOLOGY

Logical Database:
    main

Topology:
    production-main

Generation:
    1042

Age:
    3.2s

Replication Groups:
    3

Shards:
    3

Endpoints:
    9

Shard 01:
    Replication Group: rg-01
    Authority: db-01-primary
    Replicas:
        db-01-r1
        db-01-r2

Shard 02:
    Replication Group: rg-02
    Authority: db-02-primary
    Replicas:
        db-02-r1
        db-02-r2

Shard 03:
    Replication Group: rg-03
    Authority: db-03-primary
    Replicas:
        db-03-r1
        db-03-r2
```

---

# 232. Explain routing

```php
DB::distributed()->explain($query);
```

---

# 233. Example explanation

```text
DISTRIBUTED ROUTING EXPLANATION

Logical Database:
    main

Operation:
    SELECT

Partition Key:
    user_id = 93411

Resolved Shard:
    shard-07

Replication Group:
    rg-07

Required Role:
    READ

Consistency:
    READ_YOUR_WRITES

Sticky Position:
    0/9821345

Candidates:
    replica-07-a
        position satisfied: yes

    replica-07-b
        position satisfied: no

    writer-07
        authoritative: yes

Eligible:
    replica-07-a
    writer-07

Selected:
    replica-07-a

Selection Reason:
    local region + lower load

Topology Generation:
    772
```

---

# 234. Explainability rule

La decisión final deberá poder descomponerse:

```text
Domain Resolution
+
Partition Resolution
+
Replication Resolution
+
Consistency Filtering
+
Authority Filtering
+
Load Balancing
```

---

# 235. Error hierarchy

```text
DistributedDatabaseException
├── DistributedTopologyException
│   ├── TopologyDiscoveryException
│   ├── TopologyValidationException
│   ├── TopologyUnavailableException
│   ├── TopologyStaleException
│   └── TopologyGenerationException
│
├── DistributedAuthorityException
│   ├── AuthorityUnknownException
│   ├── AuthorityConflictException
│   └── AuthorityEpochMismatchException
│
├── DistributedDomainException
│   ├── ExecutionDomainResolutionException
│   ├── ExecutionDomainMismatchException
│   ├── CrossDatabaseExecutionDomainException
│   └── CrossShardTransactionException
│
├── DistributedRoutingException
│   ├── PartitionResolutionException
│   ├── ReplicationGroupResolutionException
│   └── NoEligibleDistributedEndpointException
│
├── DistributedExecutionException
│   ├── DistributedExecutionBudgetExceededException
│   ├── DistributedPartialFailureException
│   ├── DistributedMergeException
│   └── DistributedExecutionUnknownException
│
└── DistributedDatabaseInvariantViolationException
```

---

# 236. Persistent runtime architecture

En FrankenPHP:

```text
Worker
├── immutable topology snapshots
├── immutable compiled routing policies
├── bounded shared health/lag state
│
└── Request Scope
    ├── DistributedExecutionContext
    ├── TransactionContext
    ├── StickyContext
    ├── RoutingDecision
    └── selected leases
```

---

# 237. Shared vs scoped

Compartible:

```text
Topology snapshots
Compiled policies
Capability descriptors
Immutable endpoint metadata
```

Scoped:

```text
Transaction pinning
Sticky state
Current routing decision
Execution attempts
Temporary fan-out state
```

---

# 238. Static mutable context forbidden

Nunca:

```php
DistributedContext::$currentShard
```

como estado global compartido.

---

# 239. RoadRunner

Misma regla de worker reuse.

---

# 240. OpenSwoole

Contextos deberán ser coroutine/fiber scoped.

---

# 241. Topology publication concurrency

Publicar nueva topología deberá ser atomic respecto a readers concurrentes.

---

# 242. Request consistency

Una operación podrá fijar:

```text
TopologyGeneration
```

durante su routing lifecycle cuando sea necesario.

---

# 243. Long-running operation

Podrá necesitar revalidation si topology generation expira.

---

# 244. No topology mutation under feet

Una estructura ya usada por un execution plan no deberá mutarse internamente.

---

# 245. Resource cleanup

Fan-out execution deberá limpiar:

- leases;
- cursors;
- buffers;
- cancellation tokens;
- temporary merge state.

---

# 246. Cancellation

Si una operación multi-shard se cancela:

```text
cancel remaining targets
```

best-effort.

---

# 247. Cancellation ≠ rollback

Queries ya completadas no se deshacen por cancelar otras.

---

# 248. Distributed writes

Por defecto deberán evitar fan-out write implícito.

---

# 249. Multi-shard write

Requerirá API/policy explícita.

---

# 250. Failure semantics

Si:

```text
Shard A write committed
Shard B write failed
```

no afirmar:

```text
global rollback
```

sin distributed transaction protocol.

---

# 251. Compensating action

Puede ser necesaria a nivel aplicación.

---

# 252. Saga readiness

La arquitectura deberá ser compatible con futura coordinación tipo saga.

---

# 253. Outbox readiness

Cada local transaction podrá producir eventos confiables para coordinación posterior.

---

# 254. Testing architecture

Suite propuesta:

```text
DistributedTopologyTests
TopologyGenerationTests
TopologyValidationTests
TopologyProviderTests
ReplicationGroupTests
AuthorityTests
FailureDomainTests
DistributedExecutionDomainTests
DistributedRoutingTests
DistributedTransactionBoundaryTests
DistributedORMIdentityTests
DistributedCapabilityTests
DistributedExecutionBudgetTests
DistributedPartialFailureTests
DistributedPersistentRuntimeTests
DistributedConcurrencyTests
DistributedTelemetryTests
DistributedDiagnosticsTests
```

---

# 255. Topology immutability test

Una generation publicada no podrá modificarse.

---

# 256. Atomic publication test

Concurrent readers deberán observar generaciones completas.

---

# 257. Invalid topology test

Nueva topology inválida no reemplazará last-known-good válida.

---

# 258. Authority conflict test

Dos authoritative writers bajo single-writer semantics deberán producir fail-closed.

---

# 259. Replica routing test

Solo replicas del replication group correcto podrán participar.

---

# 260. Shard boundary test

Endpoint de otro shard jamás podrá seleccionarse por load balancing.

---

# 261. Tenant boundary test

Endpoint de otro tenant domain jamás podrá usarse como fallback.

---

# 262. Transaction pinning test

Active transaction permanecerá en el mismo execution domain.

---

# 263. Cross-shard transaction test

Intento implícito deberá rechazarse.

---

# 264. IdentityMap test

```text
Shard A/User 1
```

y:

```text
Shard B/User 1
```

no deberán colisionar.

---

# 265. Topology-change test

Routing posterior podrá utilizar nueva generation.

---

# 266. Stale-routing test

Authority-sensitive decision deberá revalidarse cuando policy lo exija.

---

# 267. Failure-domain test

Topology deberá conservar hierarchy correctamente.

---

# 268. Fan-out budget test

No superar `maxPartitions`.

---

# 269. Partial failure test

Un shard failure deberá producir `PARTIAL_SUCCESS` o failure explícito según operación.

---

# 270. Distributed aggregate test

Merge semantics deberán ser matemáticamente correctas.

---

# 271. Persistent worker test

Miles de operaciones no deberán acumular:

- topology snapshots sin límite;
- routing contexts;
- stale leases;
- sticky contexts;
- fan-out state.

---

# 272. Coroutine test

Dos coroutines resolviendo shards distintos deberán permanecer aisladas.

---

# 273. Directory structure

```text
src/Quantum/Database/Distributed/
│
├── DistributedDatabaseManager.php
├── DistributedDatabaseContext.php
├── DistributedExecutionDomain.php
├── DistributedExecutionBudget.php
│
├── Topology/
│   ├── DistributedDatabaseTopology.php
│   ├── DistributedTopologyId.php
│   ├── TopologyGeneration.php
│   ├── TopologySnapshotRepository.php
│   ├── TopologyValidator.php
│   ├── TopologyCompiler.php
│   ├── LastKnownGoodTopology.php
│   │
│   └── Provider/
│       ├── DistributedTopologyProvider.php
│       ├── StaticTopologyProvider.php
│       └── CompositeTopologyProvider.php
│
├── Cluster/
│   ├── DatabaseCluster.php
│   ├── ClusterId.php
│   └── ClusterMetadata.php
│
├── Endpoint/
│   ├── DatabaseEndpoint.php
│   ├── EndpointId.php
│   ├── EndpointAddress.php
│   ├── EndpointMetadata.php
│   └── EndpointRoleSet.php
│
├── Replication/
│   ├── ReplicationGroup.php
│   ├── ReplicationGroupId.php
│   ├── ReplicationSemantics.php
│   └── ReplicationGroupMap.php
│
├── Authority/
│   ├── AuthorityDescriptor.php
│   ├── AuthorityState.php
│   ├── AuthorityEpoch.php
│   └── AuthorityValidator.php
│
├── FailureDomain/
│   ├── FailureDomain.php
│   ├── FailureDomainId.php
│   ├── FailureDomainType.php
│   └── FailureDomainMap.php
│
├── Routing/
│   ├── DistributedRoutingEngine.php
│   ├── DistributedRoutingContext.php
│   ├── DistributedRoutingDecision.php
│   └── DistributedRoutingTrace.php
│
├── Execution/
│   ├── DistributedExecutionPlan.php
│   ├── DistributedExecutionStrategy.php
│   ├── DistributedExecutionOutcome.php
│   ├── DistributedQueryExecutor.php
│   └── DistributedExecutionCoordinator.php
│
├── Merge/
│   ├── ResultMergePlan.php
│   ├── DistributedResultMerger.php
│   ├── OrderedResultMerger.php
│   └── AggregateResultMerger.php
│
├── Diagnostics/
│   ├── DistributedDatabaseInspector.php
│   ├── DistributedRoutingExplainer.php
│   └── DistributedTopologyReport.php
│
├── Telemetry/
│   ├── DistributedDatabaseTelemetry.php
│   ├── DistributedTopologyPublished.php
│   ├── DistributedRoutingCompleted.php
│   └── DistributedExecutionCompleted.php
│
└── Exception/
    ├── DistributedDatabaseException.php
    ├── DistributedTopologyException.php
    ├── DistributedAuthorityException.php
    ├── DistributedDomainException.php
    ├── DistributedRoutingException.php
    ├── DistributedExecutionException.php
    └── DistributedDatabaseInvariantViolationException.php
```

---

# 274. Invariantes arquitectónicos

## DB-DIST-001
LogicalDatabase será distinto de physical endpoint.

## DB-DIST-002
Endpoint será distinto de replication group.

## DB-DIST-003
Replication group será distinto de shard.

## DB-DIST-004
Replication será distinta de sharding.

## DB-DIST-005
Sharding será distinto de multitenancy.

## DB-DIST-006
Distributed database será distinta de distributed transaction.

## DB-DIST-007
Una base distribuida no implicará sharding obligatorio.

## DB-DIST-008
Una base replicada podrá tener un solo shard.

## DB-DIST-009
Un shard podrá tener múltiples replicas.

## DB-DIST-010
Una transacción local pertenecerá a un único physical transaction resource.

## DB-DIST-011
Cross-shard operation no se convertirá automáticamente en distributed transaction.

## DB-DIST-012
Cross-database operation no se convertirá automáticamente en 2PC.

## DB-DIST-013
Distributed topology será explícita.

## DB-DIST-014
Topology snapshot será immutable.

## DB-DIST-015
Topology publication será atomic.

## DB-DIST-016
Topology tendrá generation identity.

## DB-DIST-017
TopologyId será distinto de TopologyGeneration.

## DB-DIST-018
RoutingDecision conservará topology generation relevante.

## DB-DIST-019
EndpointId deberá ser estable dentro de su identity lifecycle.

## DB-DIST-020
Endpoint address no será necesariamente endpoint identity.

## DB-DIST-021
Cluster será distinto de replication group.

## DB-DIST-022
ReplicationGroup representará un replication domain definido.

## DB-DIST-023
Authority será distinta de configured role.

## DB-DIST-024
Writer label no demostrará authority.

## DB-DIST-025
UNKNOWN authority permanecerá UNKNOWN.

## DB-DIST-026
Authority conflict bajo single-writer semantics deberá fallar cerrado.

## DB-DIST-027
VoltStack no elegirá writer por latency.

## DB-DIST-028
VoltStack no elegirá writer por load.

## DB-DIST-029
Replica state será distinto de replica lag.

## DB-DIST-030
Replica lag será distinto de topology.

## DB-DIST-031
Sticky consistency será distinta de load-balancing affinity.

## DB-DIST-032
Failover será distinto de load balancing.

## DB-DIST-033
Load balancing ocurrirá después de eligibility.

## DB-DIST-034
Partition routing ocurrirá antes de replica load balancing.

## DB-DIST-035
Execution domain será explícito.

## DB-DIST-036
TransactionContext fijará execution domain cuando corresponda.

## DB-DIST-037
Active transaction no cambiará de shard.

## DB-DIST-038
Active transaction no cambiará de database.

## DB-DIST-039
Active transaction no cambiará de replication group arbitrariamente.

## DB-DIST-040
Cross-domain transaction será rechazada por default.

## DB-DIST-041
Savepoint no crea distributed transaction.

## DB-DIST-042
Nested transaction no permite cambiar physical domain cuando se une al outer transaction.

## DB-DIST-043
Failover no migra una active transaction.

## DB-DIST-044
New retry attempt podrá usar nueva authority.

## DB-DIST-045
Transaction retry será distinto de transaction migration.

## DB-DIST-046
FailureDomain será explícito.

## DB-DIST-047
Different endpoints no implicarán independent failure domains.

## DB-DIST-048
Region será distinta de zone.

## DB-DIST-049
Provider metadata no contaminará core abstractions.

## DB-DIST-050
Topology provider será distinto de routing engine.

## DB-DIST-051
Topology discovery será distinta de failover execution.

## DB-DIST-052
Invalid discovered topology no será publicada.

## DB-DIST-053
LastKnownGoodTopology podrá conservarse según policy.

## DB-DIST-054
LastKnownGoodTopology no se presentará como current verified truth cuando esté stale.

## DB-DIST-055
Topology freshness será observable.

## DB-DIST-056
Writes usarán policies conservadoras ante stale authority information.

## DB-DIST-057
Unknown writer authority no producirá best-effort write.

## DB-DIST-058
Routing order será determinista conceptualmente.

## DB-DIST-059
Data location deberá resolverse antes de endpoint optimization.

## DB-DIST-060
UNKNOWN partition no significará all partitions.

## DB-DIST-061
Scatter-gather será explícito.

## DB-DIST-062
Scatter-gather tendrá resource budget.

## DB-DIST-063
Fan-out será bounded.

## DB-DIST-064
Distributed execution podrá representar partial failure.

## DB-DIST-065
Partial success no se presentará como full success.

## DB-DIST-066
Cancellation no equivaldrá a rollback.

## DB-DIST-067
Completed remote operation no será revertida por cancelar otro target.

## DB-DIST-068
Distributed ORDER BY requerirá global merge semantics.

## DB-DIST-069
Distributed LIMIT no será concatenación ingenua.

## DB-DIST-070
Distributed aggregates tendrán merge semantics correctas.

## DB-DIST-071
AVG no será promedio simple de promedios locales.

## DB-DIST-072
DistributedExecutionPlan será distinto de execution.

## DB-DIST-073
DistributedQueryExecutor reutilizará local Query Engine.

## DB-DIST-074
Distributed architecture no creará un segundo SQL compiler.

## DB-DIST-075
Entity identity deberá incluir execution domain cuando sea necesario.

## DB-DIST-076
Mismo ID en distintos shards no implicará misma entidad.

## DB-DIST-077
IdentityMap no colisionará entre shards.

## DB-DIST-078
IdentityMap no colisionará entre tenant domains.

## DB-DIST-079
Entity movement entre shards será operación explícita.

## DB-DIST-080
Re-sharding no será tratado como simple UPDATE local.

## DB-DIST-081
UnitOfWork podrá detectar múltiples execution domains.

## DB-DIST-082
Flush no prometerá atomicidad cross-shard.

## DB-DIST-083
Atomic multi-shard flush será rechazado por default.

## DB-DIST-084
Distributed persistence requerirá mecanismo explícito.

## DB-DIST-085
Cross-shard relationship no implicará physical foreign key.

## DB-DIST-086
Cross-shard relationship estará prohibida por default salvo soporte explícito.

## DB-DIST-087
Cross-shard SQL JOIN no será generado como JOIN local.

## DB-DIST-088
Distributed JOIN será capability separada.

## DB-DIST-089
Schema equivalence entre shards no será asumida ciegamente.

## DB-DIST-090
Schema drift será observable.

## DB-DIST-091
Migration global no estará completa mientras falten targets requeridos.

## DB-DIST-092
Distributed capability será conservadora.

## DB-DIST-093
Multi-target feature compatibility podrá requerir capability intersection.

## DB-DIST-094
Capability union no probará distributed support.

## DB-DIST-095
Homogeneous execution domains serán preferidos.

## DB-DIST-096
Heterogeneous federation será distinta de ordinary sharding.

## DB-DIST-097
Distributed consistency será distinta de transaction isolation.

## DB-DIST-098
SERIALIZABLE local no implicará global serializability.

## DB-DIST-099
Healthy endpoint no implicará eligible endpoint.

## DB-DIST-100
Eligible endpoint no implicará preferred endpoint.

## DB-DIST-101
Preferred endpoint no implicará authoritative endpoint.

## DB-DIST-102
Consistency tendrá prioridad sobre locality.

## DB-DIST-103
Consistency tendrá prioridad sobre load.

## DB-DIST-104
Authority tendrá prioridad sobre latency.

## DB-DIST-105
Local region no podrá violar freshness requirement.

## DB-DIST-106
Region failover no será asumido automáticamente.

## DB-DIST-107
Promotion deberá provenir de autoridad competente.

## DB-DIST-108
Control plane será distinto de data plane.

## DB-DIST-109
Data plane podrá consumir immutable control-plane snapshots.

## DB-DIST-110
Data plane evitará remote discovery por query cuando no sea necesario.

## DB-DIST-111
Control-plane failure no implicará siempre immediate read failure.

## DB-DIST-112
Authority-sensitive writes podrán requerir evidence freshness mayor.

## DB-DIST-113
TopologySnapshotRepository entregará snapshots coherentes.

## DB-DIST-114
Readers concurrentes no observarán partial topology generations.

## DB-DIST-115
Stale routing decisions podrán requerir revalidation.

## DB-DIST-116
Write decisions podrán tener stricter revalidation.

## DB-DIST-117
AuthorityEpoch será distinto de TopologyGeneration.

## DB-DIST-118
Authority epoch solo tendrá monotonic semantics si provider lo garantiza.

## DB-DIST-119
VoltStack no inventará fencing guarantees.

## DB-DIST-120
Connection pools distinguirán endpoint identity.

## DB-DIST-121
Writer y replica connections no se mezclarán en pool indistinguible.

## DB-DIST-122
Removed endpoint podrá entrar en draining.

## DB-DIST-123
Draining endpoint no recibirá nuevas leases.

## DB-DIST-124
Active transaction no será migrada durante draining.

## DB-DIST-125
Endpoint failure mid-transaction podrá producir UNKNOWN.

## DB-DIST-126
Failover no convertirá UNKNOWN old transaction en rolled back.

## DB-DIST-127
Retry podrá generar nueva transaction identity.

## DB-DIST-128
Same logical operation podrá tener múltiples execution attempts.

## DB-DIST-129
Distributed retries podrán requerir idempotency.

## DB-DIST-130
Idempotency será distinta de transaction retry.

## DB-DIST-131
External side effects no serán automáticamente transactional.

## DB-DIST-132
Outbox podrá coordinar side effects locales.

## DB-DIST-133
Multiple shard outboxes no implicarán global total order.

## DB-DIST-134
Wall-clock timestamps no serán global causal ordering.

## DB-DIST-135
Replication positions serán scoped a su replication semantics.

## DB-DIST-136
Global monotonic IDs requerirán mecanismo apropiado.

## DB-DIST-137
Per-shard generated IDs serán válidos si identity domain los distingue.

## DB-DIST-138
Global IDs no eliminarán partition routing.

## DB-DIST-139
Query cache keys deberán preservar execution domain.

## DB-DIST-140
Cache entries no cruzarán shard boundaries accidentalmente.

## DB-DIST-141
Compiled query reuse dependerá de compatible capability fingerprint.

## DB-DIST-142
Distributed topology diagnostics serán sanitizados.

## DB-DIST-143
Credentials no aparecerán en topology reports.

## DB-DIST-144
Tenant identity no será load-balancing preference.

## DB-DIST-145
Tenant resolution ocurrirá antes de endpoint balancing.

## DB-DIST-146
No existirá cross-tenant fallback.

## DB-DIST-147
No existirá arbitrary cross-shard fallback.

## DB-DIST-148
Replica fallback requerirá mismo logical data domain.

## DB-DIST-149
Distributed routing será explainable.

## DB-DIST-150
Routing trace conservará topology generation.

## DB-DIST-151
Telemetry será observational.

## DB-DIST-152
Telemetry no cambiará routing decisions.

## DB-DIST-153
High-cardinality topology identifiers serán gobernados.

## DB-DIST-154
Persistent worker state será bounded.

## DB-DIST-155
Topology snapshots podrán compartirse cuando sean immutable.

## DB-DIST-156
Request routing state será scope-local.

## DB-DIST-157
Sticky state será scope-local.

## DB-DIST-158
Transaction state será scope-local.

## DB-DIST-159
Static mutable current shard estará prohibido.

## DB-DIST-160
OpenSwoole coroutine contexts estarán aislados.

## DB-DIST-161
Topology publication será concurrency-safe.

## DB-DIST-162
Long-running operations podrán revalidar topology según policy.

## DB-DIST-163
Published topology objects no mutarán bajo un execution plan.

## DB-DIST-164
Distributed temporary execution resources serán liberados.

## DB-DIST-165
Fan-out writes no ocurrirán implícitamente.

## DB-DIST-166
Multi-shard writes requerirán API/policy explícita.

## DB-DIST-167
Partial distributed write no se presentará como globally rolled back.

## DB-DIST-168
Compensating operations serán distintas de rollback.

## DB-DIST-169
Saga será distinta de local database transaction.

## DB-DIST-170
Outbox será distinta de distributed transaction.

---

# 275. Modelo formal de topología

Sea:

```text
D = LogicalDatabase
```

con shards:

```text
S(D) = {S₁, S₂, ..., Sₙ}
```

Cada shard puede tener un replication group:

```text
RG(Sᵢ)
```

y cada replication group contiene endpoints:

```text
E(RGᵢ) = {e₁, e₂, ..., eₘ}
```

Para una operación `O`:

```text
Shard(O)
=
PartitionResolver(O)
```

Después:

```text
ReplicationDomain(O)
=
RG(Shard(O))
```

Después:

```text
Eligible(O)
⊆
E(ReplicationDomain(O))
```

Y finalmente:

```text
SelectedEndpoint(O)
=
LoadBalancer(
    Eligible(O)
)
```

Por lo tanto:

```text
SelectedEndpoint(O)
∈
E(RG(Shard(O)))
```

---

# 276. Invariante de seguridad de routing

Debe cumplirse:

```text
SelectedEndpoint(O)
∈
CorrectDataDomain(O)
```

antes de optimizar:

```text
latency
load
locality
cost
```

---

# 277. Modelo de decisión

```text
DistributedDecision(O)
=
TopologyGeneration
+
ExecutionDomain
+
PartitionDecision
+
ReplicationDecision
+
ConsistencyDecision
+
AuthorityDecision
+
EligibilityDecision
+
LoadBalancingDecision
```

---

# 278. Arquitectura final del pipeline

```text
                    APPLICATION
                         │
                         ▼
                 ORM / QUERY ENGINE
                         │
                         ▼
                 EXECUTION INTENT
                         │
                         ▼
               LOGICAL DATABASE
                         │
                         ▼
                TENANT CONTEXT*
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
                 TRANSACTION RULES
                         │
                         ▼
              CONSISTENCY REQUIREMENT
                         │
             ┌───────────┼────────────┐
             ▼           ▼            ▼
         REPLICA       STICKY      AUTHORITY
           LAG        CONTEXT      / FAILOVER
             └───────────┼────────────┘
                         ▼
                      HEALTH
                         │
                         ▼
                ELIGIBLE ENDPOINTS
                         │
                         ▼
                  LOAD BALANCER
                         │
                         ▼
                 SELECTED ENDPOINT
                         │
                         ▼
                CONNECTION MANAGER
                         │
                         ▼
                  CONNECTION LEASE
                         │
                         ▼
                 EXECUTION ENGINE
                         │
                         ▼
                     DATABASE

* Optional integration; Multitenancy remains outside Database core.
```

---

# 279. Regla maestra final

> **La arquitectura distribuida de VoltStack deberá conocer primero dónde pertenecen los datos, después quién posee autoridad sobre ellos, posteriormente qué copias satisfacen las garantías de consistencia requeridas y solamente entonces optimizar dónde ejecutar la operación. Distribución física nunca será tratada como equivalencia automática entre servidores.**

En forma compacta:

```text
Data Domain
    ↓
Partition
    ↓
Replication Group
    ↓
Authority
    ↓
Consistency
    ↓
Eligibility
    ↓
Load Balancing
    ↓
Physical Endpoint
```

Nunca:

```text
Available Servers
    ↓
Pick One
    ↓
Discover Whether Data Was There
```

---

# 280. Resultado arquitectónico

Con este documento VoltStack establece la separación:

```text
Logical Database
        │
        ▼
Distributed Topology
        │
   ┌────┴────┐
   ▼         ▼
Partition  Replication
   │         │
   └────┬────┘
        ▼
Execution Domain
        │
        ▼
Consistency + Authority
        │
        ▼
Eligible Endpoints
        │
        ▼
Load Balancing
        │
        ▼
Physical Execution
```

Esto permite que:

```text
Replication
```

y:

```text
Sharding
```

se combinen sin confundirse.

---

# 281. Estado del Bloque 16

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
```

Pendientes:

```text
184_DATABASE_SHARDING_SYSTEM.md
185_DATABASE_PARTITION_ROUTING_SYSTEM.md
```

---

# 282. Siguiente documento

```text
184_DATABASE_SHARDING_SYSTEM.md
```

El siguiente documento definirá formalmente:

```text
Logical Dataset
        │
        ▼
Sharding Strategy
        │
        ▼
Shard Key
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

incluyendo:

- `ShardId`;
- `ShardKey`;
- `ShardMap`;
- `ShardStrategy`;
- hash sharding;
- range sharding;
- directory sharding;
- composite shard keys;
- virtual shards;
- shard ownership;
- shard states;
- shard placement;
- shard metadata;
- hot shards;
- shard splitting;
- shard merging;
- shard draining;
- rebalancing;
- resharding;
- entity identity;
- ORM integration;
- relationships;
- transactions;
- migrations;
- persistent runtime;
- telemetry;
- diagnostics;
- seguridad;
- extensibilidad.

La regla base será:

```text
Sharding determines where data belongs.

Partition Routing determines where an operation must go.

Replication determines which copies exist.

Load Balancing determines which eligible copy should execute it.
```