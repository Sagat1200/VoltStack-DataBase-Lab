# 262_DATABASE_TENANT_CONNECTION_RESOLUTION_SYSTEM.md

# VoltStack Quantum Database
## Database Tenant Connection Resolution System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Integración:** `VoltStack/Quantum/Multitenancy`  
**Documento:** 262 — Database Tenant Connection Resolution System  
**Bloque:** 26 — Multitenancy Integration  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `261_DATABASE_MULTITENANCY_INTEGRATION_ARCHITECTURE.md`  
**Siguiente documento:** `263_DATABASE_TENANT_DATABASE_ISOLATION_SYSTEM.md`

---

# 1. Propósito

Este documento define el sistema responsable de resolver qué infraestructura Database debe utilizar una operación perteneciente a un tenant.

El sistema deberá transformar:

```text
TenantContext
+
Database Operation Intent
```

en:

```text
TenantConnectionResolution
```

capaz de ser consumido por:

```text
ConnectionManager
```

sin abrir directamente una conexión física.

La regla central será:

> **Resolver una conexión tenant-aware significa determinar de forma verificable el destino lógico, placement, shard, rol, endpoint class, credenciales y requisitos de sesión de una operación; adquirir la conexión continúa siendo responsabilidad exclusiva del ConnectionManager.**

Por tanto:

```text
TenantConnectionResolver
≠
ConnectionManager
≠
ConnectionPool
≠
Driver
```

---

# 2. Problema arquitectónico

Una aplicación sencilla podría asumir:

```php
$database = 'tenant_' . $tenantId;
```

Este modelo falla cuando aparecen:

```text
shared databases
dedicated databases
schemas
shards
replicas
regions
failover
credential rotation
tenant mobility
hybrid isolation
persistent workers
```

VoltStack no deberá convertir:

```text
TenantId
```

directamente en:

```text
DatabaseName
```

mediante una convención rígida.

---

# 3. Separación fundamental

La arquitectura distinguirá:

```text
Tenant Identity
        ↓
Tenant Placement
        ↓
Logical Database
        ↓
Shard / Replication Group
        ↓
Read/Write Role
        ↓
Endpoint Eligibility
        ↓
Credential Resolution
        ↓
Connection Pool Resolution
        ↓
Connection Acquisition
```

---

# 4. TenantId ≠ DatabaseName

Ejemplo:

```text
TenantId:
acme
```

no implica:

```text
Database:
tenant_acme
```

El tenant podría estar en:

```text
shared-prod-04
```

o:

```text
enterprise-acme-cluster
```

o:

```text
shard-eu-west-07
```

sin cambiar su identidad lógica.

---

# 5. Objetivos

El sistema deberá soportar:

1. TenantContext resolution.
2. Tenant placement resolution.
3. Logical database resolution.
4. Database-per-tenant.
5. Shared database.
6. Schema-per-tenant.
7. Shard-per-tenant.
8. Dedicated cluster.
9. Hybrid placement.
10. Read/write intent.
11. Writer routing.
12. Replica routing.
13. Sticky routing.
14. Transaction affinity.
15. Shard affinity.
16. Region affinity.
17. credential resolution.
18. credential rotation.
19. connection pool resolution.
20. connection reuse.
21. tenant session initialization.
22. tenant session reset.
23. placement generations.
24. credential generations.
25. failover.
26. tenant mobility.
27. stale placement detection.
28. persistent workers.
29. concurrent runtimes.
30. resource governance.
31. security.
32. telemetry.
33. diagnostics.
34. testing.

---

# 6. No objetivos

Este sistema no será responsable de:

```text
authenticating users
authorizing tenant access
generating SQL
executing SQL
opening PDO directly
managing ORM entities
performing migrations
billing tenants
creating SaaS subscriptions
```

---

# 7. Componentes principales

```text
TenantConnectionResolutionSystem
│
├── TenantContextValidator
├── TenantPlacementResolver
├── TenantDatabaseResolver
├── TenantShardResolver
├── TenantRegionResolver
├── TenantConnectionIntentBuilder
├── TenantCredentialResolver
├── TenantSessionProfileResolver
├── TenantPoolKeyResolver
├── TenantConnectionPolicy
├── TenantConnectionResolutionCache
└── TenantConnectionDiagnostics
```

---

# 8. Pipeline principal

```text
Database Operation
       │
       ▼
TenantContext
       │
       ▼
TenantContextValidator
       │
       ▼
TenantPlacementResolver
       │
       ▼
Logical Database Resolution
       │
       ▼
Shard Resolution
       │
       ▼
Transaction / Sticky Affinity
       │
       ▼
Read/Write Routing
       │
       ▼
Replica Eligibility
       │
       ▼
Credential Resolution
       │
       ▼
Session Profile Resolution
       │
       ▼
Pool Resolution
       │
       ▼
TenantConnectionResolution
       │
       ▼
ConnectionManager
       │
       ▼
ConnectionLease
```

---

# 9. TenantConnectionResolver

Contrato conceptual:

```php
interface TenantConnectionResolver
{
    public function resolve(
        TenantContext $tenant,
        DatabaseOperationIntent $operation,
        DatabaseContext $databaseContext,
    ): TenantConnectionResolution;
}
```

---

# 10. Resolver purity

Idealmente:

```text
resolve()
```

no deberá:

```text
open connection
begin transaction
execute SQL
mutate ORM
modify global tenant
```

Puede consultar infraestructura necesaria para placement/configuración, pero la resolución deberá permanecer conceptualmente separada de acquisition.

---

# 11. TenantConnectionResolution

Resultado inmutable:

```php
final readonly class TenantConnectionResolution
{
    public function __construct(
        public TenantId $tenant,
        public TenantPlacement $placement,
        public LogicalDatabaseId $logicalDatabase,
        public ?ShardId $shard,
        public ConnectionRole $role,
        public ConnectionTarget $target,
        public CredentialReference $credentials,
        public TenantSessionProfile $session,
        public ConnectionPoolKey $poolKey,
        public TenantPlacementGeneration $placementGeneration,
        public CredentialGeneration $credentialGeneration,
    ) {}
}
```

---

# 12. Resolution ≠ Connection

`TenantConnectionResolution` no contendrá necesariamente:

```text
PDO
socket
driver connection
active transaction
```

---

# 13. DatabaseOperationIntent

Debe describir la intención semántica.

Ejemplo:

```php
enum DatabaseOperationKind
{
    case READ;
    case WRITE;
    case LOCKING_READ;
    case SCHEMA;
    case MIGRATION;
    case ADMINISTRATION;
}
```

---

# 14. Read intent

Una operación:

```text
READ
```

puede ser elegible para réplica.

Pero:

```text
READ
≠
REPLICA
```

---

# 15. Write intent

```text
WRITE
```

normalmente requiere:

```text
WRITER
```

---

# 16. Locking read

```text
SELECT ... FOR UPDATE
```

es semánticamente:

```text
LOCKING_READ
```

y deberá dirigirse al writer.

---

# 17. Transaction override

Incluso una lectura normal:

```text
READ
```

dentro de una transaction ya asociada al writer deberá conservar la conexión/rol correspondiente.

---

# 18. Effective connection intent

Formalmente:

```text
EffectiveIntent
=
OperationIntent
+
TransactionContext
+
ConsistencyPolicy
+
StickyState
+
TenantPlacement
+
ShardAffinity
```

---

# 19. TenantPlacement

Representará la ubicación del tenant.

```php
final readonly class TenantPlacement
{
    public function __construct(
        public TenantId $tenant,
        public TenantIsolationStrategy $strategy,
        public LogicalDatabaseId $logicalDatabase,
        public ?DatabaseName $database,
        public ?SchemaName $schema,
        public ?ShardId $shard,
        public ?RegionId $region,
        public TenantPlacementGeneration $generation,
    ) {}
}
```

---

# 20. Placement may be indirect

No es obligatorio almacenar endpoint físico directamente.

Podría indicar:

```text
Logical Database
+
Shard
+
Region
```

y dejar que topology/routing resuelva endpoints.

---

# 21. Placement resolution

```text
TenantId
   ↓
TenantPlacementResolver
   ↓
TenantPlacement
```

---

# 22. Placement sources

Podrán existir:

```text
StaticTenantPlacementProvider
ConfigTenantPlacementProvider
RegistryTenantPlacementProvider
CachedTenantPlacementProvider
ControlPlaneTenantPlacementProvider
CustomTenantPlacementProvider
```

---

# 23. Resolver chain

Conceptualmente:

```text
TenantPlacementResolver
       │
       ├── L0 Execution Cache
       ├── L1 Worker Cache
       ├── L2 Shared Cache
       └── Authoritative Registry
```

---

# 24. Cache semantics

Cada nivel deberá distinguir:

```text
cached
fresh
stale
unknown
authoritative
```

---

# 25. Cache hit ≠ usable placement

Un valor puede existir físicamente en cache pero no ser utilizable bajo la política actual.

```text
PHYSICAL_HIT
≠
VALID_PLACEMENT
```

---

# 26. PlacementGeneration

Cada cambio significativo de ubicación deberá producir una generación nueva.

Ejemplo:

```text
Tenant A
Generation 71
Shard 4
```

después:

```text
Tenant A
Generation 72
Shard 9
```

---

# 27. Generation purpose

Permite detectar:

```text
stale caches
stale jobs
stale connections
stale routing decisions
stale transaction contexts
```

---

# 28. Generation ≠ timestamp

Puede implementarse mediante:

```text
monotonic integer
opaque version
UUID generation token
control-plane revision
```

---

# 29. Stable TenantId

Durante migration:

```text
TenantId
```

permanece estable.

Cambia:

```text
TenantPlacement
```

---

# 30. LogicalDatabaseId

VoltStack deberá preferir una identidad lógica.

Ejemplo:

```text
customer-data
```

en lugar de hacer que la aplicación conozca:

```text
mysql-prod-cluster-17.internal
```

---

# 31. Logical to physical resolution

```text
LogicalDatabaseId
      ↓
Topology
      ↓
Replication Group
      ↓
Endpoint
```

---

# 32. Database-per-tenant

Ejemplo:

```text
Tenant A
  ↓
Placement
  ↓
Logical DB: tenant-a-data
  ↓
Physical DB: prod_tenant_a
```

---

# 33. Shared database

```text
Tenant A ─┐
Tenant B ─┼── Logical DB: shared-customers-1
Tenant C ─┘
```

---

# 34. Shared schema connection resolution

En:

```text
SHARED_DATABASE_SHARED_SCHEMA
```

varios tenants podrán resolver al mismo:

```text
endpoint
database
credentials
pool
```

La separación de filas será responsabilidad adicional del Tenant Query Context.

---

# 35. Separate schema

En:

```text
SHARED_DATABASE_SEPARATE_SCHEMA
```

varios tenants pueden compartir conexión física, pero requerir:

```text
different schema context
```

---

# 36. Session profile

Por ello se introduce:

```text
TenantSessionProfile
```

---

# 37. TenantSessionProfile

Ejemplo:

```php
final readonly class TenantSessionProfile
{
    public function __construct(
        public ?DatabaseName $database,
        public ?SchemaName $schema,
        public ?DatabaseRole $role,
        public array $safeSessionSettings,
    ) {}
}
```

---

# 38. Session profile restrictions

No deberá contener secretos arbitrarios ni SQL libre no validado.

---

# 39. Session initialization

Después de adquirir una conexión:

```text
ConnectionLease
      ↓
TenantSessionInitializer
      ↓
Tenant Session Ready
```

---

# 40. Initialization ≠ resolution

El resolver determina:

```text
required session state
```

El Connection lifecycle aplica ese estado.

---

# 41. Session state examples

Podría incluir:

```text
database
schema/search path
role
timezone
tenant session variable
application name
```

si la plataforma lo soporta.

---

# 42. Platform capabilities

No todas las bases permiten las mismas operaciones de sesión.

Usar:

```text
PlatformCapabilities
```

No:

```php
if ($database === 'postgres') {
}
```

disperso por el sistema.

---

# 43. Tenant schema qualification

Cuando sea preferible, el compiler puede recibir schema ya resuelto en metadata/query plan.

Esto evita modificar session state.

---

# 44. Session switching trade-off

Dos enfoques:

```text
Connection Session Switching
```

vs.

```text
Qualified Object Names
```

La estrategia será capability/policy-driven.

---

# 45. ConnectionRole

Ejemplo:

```php
enum ConnectionRole
{
    case WRITER;
    case REPLICA;
    case ADMIN;
    case MIGRATION;
}
```

---

# 46. Tenant connection role

El tenant no seleccionará arbitrariamente el role mediante input externo.

Será derivado de:

```text
operation
policy
transaction
consistency
```

---

# 47. Read/write routing integration

Reutilizará:

```text
176_DATABASE_READ_WRITE_CONNECTION_SYSTEM.md
177_DATABASE_READ_WRITE_ROUTING_SYSTEM.md
178_DATABASE_REPLICA_SYSTEM.md
179_DATABASE_REPLICA_LAG_AWARENESS_SYSTEM.md
180_DATABASE_STICKY_CONNECTION_SYSTEM.md
```

---

# 48. Routing order

```text
Tenant
 ↓
Placement
 ↓
Replication Group
 ↓
Consistency
 ↓
Transaction Affinity
 ↓
Sticky State
 ↓
Writer/Replica Eligibility
 ↓
Load Balancing
```

---

# 49. No replica before placement

Incorrecto:

```text
Choose Replica
   ↓
Resolve Tenant
```

Correcto:

```text
Resolve Tenant Placement
   ↓
Find Placement Replicas
   ↓
Choose Eligible Replica
```

---

# 50. Replica eligibility

Debe considerar:

```text
health
lag
tenant placement generation
region
read consistency
transaction state
sticky state
```

---

# 51. Tenant sticky state

Ejemplo:

```php
final readonly class TenantStickyState
{
    public function __construct(
        public TenantId $tenant,
        public LogicalDatabaseId $database,
        public ?ShardId $shard,
        public ReplicationPosition $minimumPosition,
    ) {}
}
```

---

# 52. Sticky scope

Será:

```text
execution scoped
```

no:

```text
worker global
```

---

# 53. Tenant A write

```text
Tenant A
WRITE
 ↓
Writer
 ↓
Sticky(A)
```

---

# 54. Tenant B unaffected

```text
Tenant B
READ
```

no deberá volverse writer-sticky por una escritura de A.

---

# 55. Transaction affinity

Al iniciar transaction:

```text
Tenant
+
Placement Generation
+
Shard
+
ConnectionLease
```

quedarán asociados al `TransactionContext`.

---

# 56. TransactionConnectionAffinity

Conceptualmente:

```php
final readonly class TransactionConnectionAffinity
{
    public function __construct(
        public TransactionId $transaction,
        public TenantId $tenant,
        public TenantPlacementGeneration $placementGeneration,
        public ConnectionLeaseId $lease,
        public ConnectionTargetId $target,
    ) {}
}
```

---

# 57. Placement change during transaction

Si control plane mueve tenant:

```text
Generation 41
→
Generation 42
```

una transaction iniciada en 41 no migrará silenciosamente.

---

# 58. Transaction completion

Podrá terminar sobre su conexión original si sigue siendo válida conforme a política.

---

# 59. Hard cutover

Si infraestructura antigua deja de estar disponible:

```text
transaction outcome
```

podrá ser:

```text
FAILED
```

o:

```text
UNKNOWN
```

según evidencia.

---

# 60. Never transparent replay

No repetir automáticamente toda transaction en el nuevo placement salvo que:

```text
RetryPolicy
```

lo autorice y sea seguro.

---

# 61. Shard resolution

Para tenants sharded:

```text
TenantContext
      ↓
Placement
      ↓
Shard Routing
```

---

# 62. Tenant ≠ Shard

Muchos tenants pueden compartir shard.

---

# 63. Tenant dedicated shard

También puede existir:

```text
Tenant A
→
Shard A
```

---

# 64. Internal tenant sharding

Un tenant grande incluso podría ocupar múltiples shards.

Entonces:

```text
TenantContext
+
Entity/Partition Key
```

puede ser necesario.

---

# 65. Connection resolver and partition routing

En ese escenario:

```text
TenantConnectionResolver
```

no inventará un shard.

Consumirá:

```text
PartitionRoutingDecision
```

cuando la operación requiera una partition key.

---

# 66. UNKNOWN shard

Nunca:

```text
UNKNOWN shard
→
default shard
```

para writes.

---

# 67. Multi-shard read

Podrá generar:

```text
DistributedExecutionPlan
```

en capas correspondientes.

No una única conexión falsa.

---

# 68. Region resolution

Placement podrá incluir:

```text
RegionId
```

---

# 69. Region preference

Puede existir:

```text
preferred region
required region
data residency region
```

---

# 70. Data residency

Si política exige:

```text
Tenant A → Mexico Region
```

no deberá fallar silenciosamente a otra región no autorizada.

---

# 71. Region failover

Debe respetar:

```text
residency
security
consistency
tenant policy
```

---

# 72. Credential resolution

El sistema deberá resolver:

```text
CredentialReference
```

no necesariamente secretos directamente.

---

# 73. CredentialReference

Ejemplo:

```php
final readonly class CredentialReference
{
    public function __construct(
        public CredentialId $id,
        public CredentialGeneration $generation,
    ) {}
}
```

---

# 74. Secret retrieval

Los secretos reales podrán provenir de:

```text
secret manager
environment
vault
runtime secret provider
platform credential service
```

---

# 75. Secret separation

```text
TenantPlacement
```

no deberá contener necesariamente:

```text
plaintext password
```

---

# 76. CredentialResolver

```php
interface DatabaseCredentialResolver
{
    public function resolve(
        CredentialReference $reference
    ): DatabaseCredential;
}
```

---

# 77. Credential lifetime

Secretos deberán tener lifetime mínimo necesario.

---

# 78. Credential cache

Si existe deberá:

```text
be bounded
support rotation
avoid logging secrets
respect expiration
```

---

# 79. Credential generation

Ejemplo:

```text
CredentialGeneration 12
```

rota a:

```text
CredentialGeneration 13
```

---

# 80. Pool implication

Conexiones creadas con generación 12 pueden necesitar:

```text
retirement
```

---

# 81. Pool key generation

`CredentialGeneration` puede formar parte del:

```text
ConnectionPoolKey
```

---

# 82. Rotation strategy

```text
New operations
     ↓
Generation 13 pool

Old idle connections
     ↓
retire

Old leased connections
     ↓
finish according to policy
```

---

# 83. No credential mutation

No intentar cambiar mágicamente credenciales de una conexión física ya autenticada.

---

# 84. PoolKey

Conceptualmente:

```php
final readonly class ConnectionPoolKey
{
    public function __construct(
        public ConnectionTargetId $target,
        public LogicalDatabaseId $database,
        public CredentialId $credential,
        public CredentialGeneration $credentialGeneration,
        public ConnectionRole $role,
        public ?DatabaseName $physicalDatabase,
        public ConnectionProfileId $profile,
    ) {}
}
```

---

# 85. TenantId absent from PoolKey

Si dos tenants pueden compartir conexión de forma segura:

```text
TenantId
```

no necesita estar en el pool key.

---

# 86. TenantId included when required

Si existe:

```text
dedicated credentials
dedicated DB
dedicated security context
non-resettable tenant state
```

la resolución podrá producir pools separados.

---

# 87. Pool identity ≠ Tenant identity

Regla:

```text
PoolKey
=
Physical Connection Compatibility
```

no:

```text
PoolKey
=
TenantId always
```

---

# 88. Pool explosion prevention

Con miles de tenants:

```text
pool per tenant
```

no será default.

---

# 89. Example

```text
50,000 tenants
×
2 connections
=
100,000 physical connections
```

aunque estén inactivos.

---

# 90. Shared pool model

```text
Tenant A ─┐
Tenant B ─┼── Compatible Placement → Shared Pool
Tenant C ─┘
```

con tenant session reset seguro.

---

# 91. Dedicated pool model

```text
Enterprise Tenant X
       ↓
Dedicated DB
       ↓
Dedicated Credentials
       ↓
Dedicated Pool
```

---

# 92. Pool registry

Necesitará límites:

```text
max pools
max idle pools
max connections
idle timeout
pool retirement
LRU-like eviction
generation retirement
```

---

# 93. Pool retirement

Un pool puede entrar en:

```text
ACTIVE
DRAINING
RETIRED
CLOSED
```

---

# 94. Placement generation change

Pool antiguo puede:

```text
DRAIN
```

sin aceptar nuevas leases.

---

# 95. New placement

Nuevas operaciones resolverán al nuevo pool.

---

# 96. ConnectionTarget

Representa endpoint/topology target.

Ejemplo:

```php
final readonly class ConnectionTarget
{
    public function __construct(
        public ConnectionTargetId $id,
        public EndpointAddress $endpoint,
        public DatabasePlatformId $platform,
        public ConnectionRole $role,
        public RegionId $region,
        public EndpointGeneration $generation,
    ) {}
}
```

---

# 97. Endpoint address security

No deberá exponerse innecesariamente en errores públicos.

---

# 98. Endpoint health

Connection resolution deberá consumir:

```text
EndpointHealth
```

cuando corresponda.

---

# 99. Endpoint health states

Conceptualmente:

```text
HEALTHY
DEGRADED
UNHEALTHY
UNKNOWN
```

---

# 100. UNKNOWN health

Su tratamiento dependerá de política.

Pero nunca deberá transformarse automáticamente en:

```text
HEALTHY
```

---

# 101. Failover

Cuando writer falla:

```text
Writer A
  ↓
failure
  ↓
Topology Update
  ↓
Writer B
```

---

# 102. Tenant placement vs failover

El tenant puede conservar:

```text
same placement generation
```

si el cambio ocurre dentro del mismo replication group/topology generation, o requerir nueva generación según modelo.

---

# 103. Generation domains

Conviene distinguir:

```text
TenantPlacementGeneration
TopologyGeneration
EndpointGeneration
CredentialGeneration
SchemaGeneration
```

---

# 104. Generation ≠ one global version

No mezclar todas las generaciones en un único contador si representan cambios distintos.

---

# 105. Resolution token

Podrá existir:

```php
final readonly class TenantConnectionResolutionToken
{
    public function __construct(
        public TenantPlacementGeneration $placement,
        public TopologyGeneration $topology,
        public CredentialGeneration $credentials,
    ) {}
}
```

---

# 106. Resolution validation

Antes o durante acquisition podrá verificarse que la resolución siga siendo válida.

---

# 107. TOCTOU problem

Existe una ventana:

```text
resolve
  ↓
topology changes
  ↓
acquire
```

---

# 108. Mitigation

Generation validation podrá detectar:

```text
stale resolution
```

---

# 109. Stale resolution

Podrá:

```text
re-resolve
```

si todavía no existen efectos laterales.

---

# 110. No unsafe re-resolution

Después de comenzar:

```text
transaction
write
stream
```

no se deberá migrar silenciosamente.

---

# 111. Tenant mobility

Mover tenant:

```text
Placement A
→
Placement B
```

requiere coordination.

---

# 112. Mobility phases

Conceptualmente:

```text
STABLE_A
   ↓
PREPARING
   ↓
COPYING
   ↓
CATCHING_UP
   ↓
CUTOVER_PENDING
   ↓
STABLE_B
```

---

# 113. Resolver behavior during mobility

Debe depender de una política explícita.

Ejemplo:

```text
reads → A
writes → A
```

hasta cutover.

Después:

```text
reads → B
writes → B
```

---

# 114. No implicit dual write

El connection resolver no devolverá:

```text
A + B
```

para un write normal y asumirá éxito distribuido.

---

# 115. Dual-write protocol

Si existe, pertenecerá a un sistema explícito de data movement/replication.

---

# 116. Migration cutover

Al cambiar:

```text
A → B
```

deberá cambiar la generación correspondiente.

---

# 117. Existing connection leases

Leases antiguas:

```text
Generation A
```

no deberán volver a ser elegibles para operaciones nuevas de generación B.

---

# 118. Lease ownership

Una vez adquirido:

```text
ConnectionLease
```

tendrá:

```text
execution owner
operation owner
tenant domain
placement generation
```

cuando sea relevante.

---

# 119. TenantConnectionLeaseMetadata

Conceptualmente:

```php
final readonly class TenantConnectionLeaseMetadata
{
    public function __construct(
        public TenantId $tenant,
        public TenantPlacementGeneration $placementGeneration,
        public ConnectionPoolKey $pool,
        public TenantSessionProfile $session,
    ) {}
}
```

---

# 120. Lease ≠ tenant global state

La metadata viaja con el lease.

---

# 121. Cross-tenant lease reuse

Una conexión física puede reutilizarse posteriormente.

Pero:

```text
Lease A
```

nunca se transforma en:

```text
Lease B
```

mientras sigue activa.

---

# 122. Release pipeline

```text
Tenant Operation Ends
        ↓
Close Results
        ↓
Resolve Transaction State
        ↓
Reset Tenant Session
        ↓
Reset Generic Session
        ↓
Verify Connection
        ↓
Release Lease
        ↓
Return to Pool
```

---

# 123. Reset ordering

Tenant-specific reset deberá formar parte del reset general definido en:

```text
255_DATABASE_STATE_RESET_SYSTEM.md
256_DATABASE_CONNECTION_REUSE_SYSTEM.md
```

---

# 124. TenantSessionResetter

Contrato conceptual:

```php
interface TenantSessionResetter
{
    public function reset(
        Connection $connection,
        TenantSessionProfile $profile
    ): TenantSessionResetResult;
}
```

---

# 125. Reset result

```text
CLEAN
DISCARD_REQUIRED
UNKNOWN
FAILED
```

---

# 126. UNKNOWN reset

Regla:

```text
UNKNOWN
⇒
DISCARD
```

---

# 127. Schema reset

Ejemplo:

```text
Tenant A → schema_a
```

al liberar:

```text
schema_a
```

no deberá quedar activo accidentalmente.

---

# 128. Role reset

Si Tenant A usó:

```text
SET ROLE tenant_a_role
```

deberá revertirse.

---

# 129. Session variables

Variables tenant-specific deberán eliminarse/restaurarse.

---

# 130. Temporary objects

Si la estrategia permite temp tables u otros objetos session-local, deberán incluirse en cleanliness policy.

---

# 131. Advisory locks

Locks tenant-related no deberán filtrarse al siguiente lease.

---

# 132. Active result sets

Una conexión con result cursor activo no deberá volver al pool.

---

# 133. Transaction state

Una conexión con transaction activa no deberá volver al pool.

---

# 134. Connection contamination

Definición:

```text
TenantConnectionContamination
=
Residual tenant-specific state
after lease release
```

---

# 135. Severity

Será una violación crítica de aislamiento.

---

# 136. Persistent runtime integration

Aplica a:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 137. Worker state

Nunca almacenar:

```php
static $currentTenantConnection;
```

---

# 138. Execution scope

Cada request/job/command tendrá:

```text
TenantConnectionResolutionContext
```

propio.

---

# 139. OpenSwoole concurrency

Ejemplo:

```text
Coroutine A
Tenant A
Connection Lease C1

Coroutine B
Tenant B
Connection Lease C2
```

---

# 140. Concurrent shared pool

C1 y C2 podrían provenir del mismo pool si son físicamente compatibles.

---

# 141. But

No compartirán simultáneamente la misma conexión física por default.

---

# 142. One active owner

Regla:

```text
Physical Connection
→
One Active Lease Owner
```

salvo futura capability explícita de multiplexing.

---

# 143. Context-local resolution

Coroutine A no deberá observar:

```text
TenantConnectionResolution(B)
```

---

# 144. Connection resolution cache

Puede existir cache por execution.

---

# 145. L0 resolution cache

Muy útil para evitar resolver placement repetidamente dentro de la misma request.

---

# 146. L0 key

Conceptualmente:

```text
TenantId
+
Operation Routing Class
+
Placement Generation
```

---

# 147. Worker cache

Puede compartir placement metadata.

No debe compartir:

```text
current tenant
current connection intent
current lease
```

---

# 148. Shared cache invalidation

Placement update deberá invalidar/obsoletar generaciones anteriores.

---

# 149. Resource governance

Connection resolution deberá integrarse con:

```text
249_DATABASE_RESOURCE_GOVERNANCE_SYSTEM.md
```

---

# 150. Per-tenant limits

Podrán existir:

```text
max leased connections
max waiting acquisitions
max concurrent DB operations
```

---

# 151. Global limits

También:

```text
global pool size
global active connections
global connection creation rate
```

---

# 152. Tenant quota ≠ pool size

Un tenant con:

```text
max concurrent DB operations = 10
```

no necesita un pool físico exclusivo de 10 conexiones.

---

# 153. Noisy neighbor

Tenant A no deberá consumir todas las leases si policy exige fairness.

---

# 154. Admission pipeline

```text
Tenant Operation
      ↓
Tenant Resource Policy
      ↓
Global Resource Policy
      ↓
Connection Resolution
      ↓
Pool Admission
```

---

# 155. Acquisition wait

Debe ser:

```text
bounded
cancellable
deadline-aware
```

---

# 156. Tenant deadline

Una operación puede tener:

```text
request deadline
tenant policy deadline
database acquisition timeout
```

---

# 157. Effective deadline

```text
EffectiveDeadline
=
min(
    RequestDeadline,
    OperationDeadline,
    TenantPolicyDeadline,
    AcquisitionDeadline
)
```

---

# 158. Security

Connection resolution es una frontera sensible.

---

# 159. TenantId untrusted input

Un `TenantId` proveniente de:

```text
HTTP header
URL
API payload
```

no deberá considerarse autorizado.

---

# 160. Authorization before sensitive resolution

La arquitectura superior deberá establecer tenant autorizado antes de operaciones normales.

---

# 161. Enumeration protection

Errores públicos no deberán revelar:

```text
whether tenant exists
database host
database name
cluster name
credentials
schema
internal topology
```

sin necesidad.

---

# 162. Internal diagnostics

Sí podrán contener metadata adicional bajo acceso administrativo.

---

# 163. Credentials

Nunca incluir en:

```text
logs
exceptions
traces
metrics
debug toolbar
```

---

# 164. Connection string

Debe redactarse.

Ejemplo:

```text
mysql://db.internal:3306/customer
```

puede mostrarse internamente según política, pero nunca:

```text
mysql://user:password@...
```

---

# 165. Tenant session injection

Schema/database/role names derivados de metadata deberán validarse como identifiers.

No concatenar input arbitrario.

---

# 166. Query parameterization does not protect identifiers

Importante:

```text
prepared statements
```

protegen valores, no necesariamente:

```text
schema names
database names
role names
```

---

# 167. Identifier validation

Todo identifier dinámico deberá provenir de:

```text
trusted placement metadata
+
platform identifier validation/quoting
```

---

# 168. Fail closed

Si placement es ambiguo:

```text
REJECT
```

---

# 169. No default tenant database

Nunca:

```text
resolution failed
→
default application database
```

para una operación tenant-scoped.

---

# 170. Failure taxonomy

```text
TenantConnectionResolutionException
├── TenantContextMissingException
├── TenantPlacementNotFoundException
├── TenantPlacementStaleException
├── TenantPlacementUnavailableException
├── TenantDatabaseResolutionException
├── TenantShardResolutionException
├── TenantRegionResolutionException
├── TenantCredentialResolutionException
├── TenantCredentialExpiredException
├── TenantTopologyResolutionException
├── TenantConnectionTargetUnavailableException
├── TenantConnectionPoolUnavailableException
├── TenantSessionProfileException
├── TenantConnectionAffinityException
├── TenantResourceLimitException
└── TenantConnectionSecurityException
```

---

# 171. Resolution failure ≠ acquisition failure

Distinguir:

```text
could not determine target
```

de:

```text
target known but connection failed
```

---

# 172. Acquisition failures

Pertenecen principalmente a:

```text
236_DATABASE_CONNECTION_FAILURE_HANDLING_SYSTEM.md
```

---

# 173. Retry resolution

Placement resolution puede reintentarse si:

```text
failure is transient
no DB side effect exists
deadline allows
policy permits
```

---

# 174. Retry target acquisition

Puede usar las reglas de:

```text
238_DATABASE_RETRY_POLICY_SYSTEM.md
240_DATABASE_FAILOVER_AND_RECOVERY_SYSTEM.md
```

---

# 175. Retry ≠ arbitrary tenant migration

Un retry no deberá elegir otro tenant placement incompatible sin generation/topology validation.

---

# 176. Circuit breaker

Puede existir por:

```text
endpoint
cluster
placement
credential provider
control plane
```

---

# 177. Tenant circuit breaker

Evitar uno por tenant cuando miles comparten infraestructura, salvo que exista razón semántica.

---

# 178. Failure domain

Circuit breakers deben alinearse con el verdadero failure domain.

Ejemplo:

```text
Shared Cluster X
```

no:

```text
50,000 independent breakers
```

si todos usan el mismo cluster.

---

# 179. Telemetry

Eventos conceptuales:

```text
TenantConnectionResolutionStarted
TenantPlacementResolved
TenantConnectionTargetResolved
TenantConnectionResolutionCompleted
TenantConnectionResolutionFailed
TenantPlacementStaleDetected
TenantConnectionPoolResolved
TenantSessionInitialized
TenantSessionReset
TenantConnectionContaminationDetected
```

---

# 180. High-volume telemetry

No emitir todos los eventos como logs permanentes por default.

---

# 181. Metrics

Ejemplos:

```text
database_tenant_connection_resolution_duration
database_tenant_connection_resolution_failures_total
database_tenant_placement_cache_hits_total
database_tenant_placement_stale_total
database_tenant_connection_acquisition_wait
database_tenant_session_reset_failures_total
```

---

# 182. Tenant cardinality warning

No usar:

```text
tenant_id
```

como metric label general.

---

# 183. Bounded dimensions

Preferir:

```text
isolation_strategy
connection_role
result
failure_class
runtime
platform
region_class
```

cuando sean bounded.

---

# 184. Tracing

Span:

```text
database.tenant.connection.resolve
```

podrá incluir atributos controlados.

---

# 185. Sensitive attributes

Tenant identity deberá seguir políticas de privacy/observability.

---

# 186. Query telemetry

Una query podrá relacionarse internamente con:

```text
ConnectionResolutionId
```

---

# 187. ResolutionId

Permite correlacionar:

```text
tenant resolution
routing
pool wait
connection lease
query execution
```

sin exponer credenciales.

---

# 188. Diagnostics

Comando conceptual:

```text
php volt database:tenant:connection <tenant>
```

---

# 189. Diagnostic example

```text
Tenant Connection Resolution
────────────────────────────────

Tenant:              acme
Status:              ACTIVE
Isolation:           DATABASE_PER_TENANT

Placement
  Logical Database:  customer-data
  Region:            mx-central
  Shard:             shard-07
  Generation:        42

Routing
  Intent:            READ
  Role:              REPLICA
  Consistency:       EVENTUAL
  Sticky:            no

Credentials
  Reference:         db-customer-read
  Generation:        18
  Secret:            [REDACTED]

Pool
  Key:               pool:***
  State:             ACTIVE

Session
  Database:           [internal]
  Schema:             default
  Role:               customer_reader

Resolution
  Status:             VALID
```

---

# 190. Explain mode

```text
php volt database:tenant:connection acme --explain
```

podrá mostrar:

```text
1. TenantContext validated
2. Placement generation 42 loaded
3. Logical database customer-data selected
4. Shard shard-07 selected
5. READ intent evaluated
6. No transaction affinity
7. No sticky writer requirement
8. Eligible replica set resolved
9. Credential generation 18 selected
10. Compatible connection pool selected
```

---

# 191. Debug toolbar

Podrá mostrar:

```text
Tenant
Placement
Connection Role
Shard
Region
Pool Wait
Resolution Duration
```

sin secretos.

---

# 192. Health checks

Debe distinguir:

```text
control-plane health
placement resolver health
credential provider health
pool health
endpoint health
```

---

# 193. Tenant health ≠ global health

Un tenant puede estar degradado sin que toda la aplicación lo esté.

---

# 194. Shared failure

También:

```text
Shared Cluster Down
```

puede afectar miles de tenants.

---

# 195. Readiness

Aplicación puede permanecer parcialmente ready dependiendo de política.

---

# 196. Testing architecture

Se requerirán:

```text
Unit Tests
Placement Resolution Tests
Routing Tests
Credential Tests
Pool Resolution Tests
Session Initialization Tests
Session Reset Tests
Transaction Affinity Tests
Sticky Routing Tests
Replica Tests
Failover Tests
Mobility Tests
Generation Tests
Persistent Runtime Tests
Concurrency Tests
Security Tests
Resource Governance Tests
Fault Injection Tests
```

---

# 197. Basic resolution test

```text
Tenant A
→
Placement A
→
Database A
```

debe ser determinista.

---

# 198. Shared database test

```text
Tenant A
Tenant B
```

resuelven al mismo pool compatible pero con contextos tenant distintos.

---

# 199. Database-per-tenant test

A y B resuelven a databases diferentes.

---

# 200. Schema test

A y B comparten conexión compatible pero obtienen session profiles distintos.

---

# 201. Credential test

Tenant enterprise con credenciales dedicadas resuelve pool separado.

---

# 202. Rotation test

```text
Credential generation 10
→
11
```

Nuevas conexiones usan 11.

---

# 203. Old pool test

Pool generation 10 entra en drain.

---

# 204. Placement movement test

```text
Tenant A
DB-X generation 5
→
DB-Y generation 6
```

Nuevas operaciones usan Y.

---

# 205. Active transaction test

Transaction iniciada en generation 5 no salta a Y.

---

# 206. Stale cache test

Resolver obtiene generation 5 después de cutover a 6.

Debe detectar staleness cuando exista evidencia suficiente.

---

# 207. TOCTOU test

Cambiar topology entre resolution y acquisition.

El sistema deberá revalidar o fallar según política.

---

# 208. Sticky isolation test

Write de Tenant A no vuelve sticky a Tenant B.

---

# 209. Transaction isolation test

Transaction A no usa conexión de B.

---

# 210. Session contamination test

```text
Lease A → schema_a
release
Lease B → schema_b
```

B nunca observa `schema_a`.

---

# 211. Reset failure test

Si no puede resetear:

```text
schema_a
```

la conexión es descartada.

---

# 212. Pool explosion test

Crear decenas de miles de tenants con shared placement.

Número de pools deberá permanecer bounded según compatibilidad física.

---

# 213. Resource fairness test

Tenant A satura su cuota.

Tenant B debe conservar capacidad conforme a policy.

---

# 214. Concurrent runtime test

OpenSwoole:

```text
Coroutine A → Tenant A
Coroutine B → Tenant B
Coroutine C → Tenant C
```

con resolution simultánea.

---

# 215. Worker reuse test

Secuencia:

```text
A → B → C → A
```

sin current tenant/connection leakage.

---

# 216. Failure test

Control plane no disponible.

Validar comportamiento de cache según freshness policy.

---

# 217. Unknown placement test

Sin placement seguro:

```text
REJECT
```

---

# 218. Security test

TenantId manipulado no deberá seleccionar arbitrariamente un database interno sin authorization/context validation.

---

# 219. Identifier injection test

Malicious schema/database identifier deberá rechazarse.

---

# 220. Proposed directory

```text
src/Quantum/Multitenancy/Database/Connection/
│
├── Contract/
│   ├── TenantConnectionResolver.php
│   ├── TenantPlacementResolver.php
│   ├── TenantCredentialResolver.php
│   ├── TenantSessionProfileResolver.php
│   └── TenantPoolKeyResolver.php
│
├── Resolution/
│   ├── TenantConnectionResolution.php
│   ├── TenantConnectionResolutionId.php
│   ├── TenantConnectionResolutionContext.php
│   ├── TenantConnectionResolutionToken.php
│   └── DefaultTenantConnectionResolver.php
│
├── Placement/
│   ├── TenantPlacement.php
│   ├── TenantPlacementGeneration.php
│   ├── TenantPlacementRegistry.php
│   └── TenantPlacementCache.php
│
├── Intent/
│   ├── TenantConnectionIntent.php
│   └── TenantConnectionIntentBuilder.php
│
├── Credential/
│   ├── CredentialReference.php
│   ├── CredentialGeneration.php
│   └── TenantDatabaseCredentialResolver.php
│
├── Pool/
│   ├── TenantPoolKeyResolver.php
│   ├── TenantPoolPolicy.php
│   └── TenantPoolRetirementPolicy.php
│
├── Session/
│   ├── TenantSessionProfile.php
│   ├── TenantSessionInitializer.php
│   ├── TenantSessionResetter.php
│   └── TenantSessionResetResult.php
│
├── Affinity/
│   ├── TenantTransactionAffinity.php
│   ├── TenantStickyState.php
│   └── TenantShardAffinity.php
│
├── Mobility/
│   ├── TenantMobilityState.php
│   └── TenantPlacementCutoverPolicy.php
│
├── Security/
│   └── TenantConnectionSecurityValidator.php
│
├── Telemetry/
│   └── TenantConnectionTelemetry.php
│
├── Diagnostics/
│   └── TenantConnectionDiagnostics.php
│
└── Exception/
    ├── TenantConnectionResolutionException.php
    ├── TenantPlacementNotFoundException.php
    ├── TenantPlacementStaleException.php
    ├── TenantCredentialResolutionException.php
    ├── TenantSessionException.php
    └── TenantConnectionSecurityException.php
```

---

# 221. Integration with Database Core

```text
Quantum/Multitenancy
        │
        ▼
TenantConnectionResolution
        │
        ▼
Database Connection Contracts
        │
        ▼
ConnectionManager
        │
        ▼
Connection Pool
        │
        ▼
Driver
```

---

# 222. Forbidden dependency

Nunca:

```text
Driver
 ↓
Multitenancy
```

---

# 223. ConnectionManager awareness

ConnectionManager no necesita conocer toda la semántica del tenant.

Debe recibir una resolución suficiente para adquirir/configurar correctamente el recurso.

---

# 224. Tenant connection invariants

## DB-TCR-001

TenantId no será DatabaseName.

## DB-TCR-002

TenantId no será ConnectionName.

## DB-TCR-003

TenantConnectionResolver no abrirá conexiones.

## DB-TCR-004

ConnectionManager será responsable de acquisition.

## DB-TCR-005

Tenant placement será resuelto antes de acquisition.

## DB-TCR-006

Placement será independiente de tenant identity.

## DB-TCR-007

TenantId permanecerá estable durante placement changes.

## DB-TCR-008

Placement tendrá generation.

## DB-TCR-009

Placement generation no será necesariamente timestamp.

## DB-TCR-010

Logical database será preferible a hardcoded physical endpoints.

## DB-TCR-011

Physical endpoint será resuelto mediante topology.

## DB-TCR-012

Read intent no implicará replica automáticamente.

## DB-TCR-013

Write intent requerirá writer salvo capability/policy explícita.

## DB-TCR-014

Locking read utilizará writer.

## DB-TCR-015

Transaction affinity tendrá prioridad sobre generic read routing.

## DB-TCR-016

Sticky routing será tenant-scoped.

## DB-TCR-017

Write de Tenant A no afectará sticky state de B.

## DB-TCR-018

Replica selection ocurrirá dentro del tenant placement.

## DB-TCR-019

Replica health será evaluada antes de selection.

## DB-TCR-020

Replica lag será evaluado según consistency policy.

## DB-TCR-021

UNKNOWN replica freshness no equivaldrá a fresh.

## DB-TCR-022

TenantConnectionResolution será immutable.

## DB-TCR-023

Resolution no contendrá live connection por definición.

## DB-TCR-024

TenantSessionProfile será explícito.

## DB-TCR-025

Session initialization ocurrirá después de acquisition.

## DB-TCR-026

Session reset ocurrirá antes de reuse.

## DB-TCR-027

UNKNOWN session reset causará discard.

## DB-TCR-028

Active transaction impedirá normal pool return.

## DB-TCR-029

Active cursor impedirá normal pool return.

## DB-TCR-030

Residual tenant state será contamination.

## DB-TCR-031

Tenant contamination será critical isolation failure.

## DB-TCR-032

Schema switching será capability-driven.

## DB-TCR-033

Database switching será capability-driven.

## DB-TCR-034

Role switching será capability-driven.

## DB-TCR-035

Tenant-specific identifiers serán validados.

## DB-TCR-036

Untrusted identifiers no serán concatenados.

## DB-TCR-037

Prepared statements no serán considerados protección de identifiers.

## DB-TCR-038

Credential references serán separadas de secrets.

## DB-TCR-039

Plaintext credentials no vivirán en TenantPlacement.

## DB-TCR-040

Credentials no aparecerán en telemetry.

## DB-TCR-041

Credential rotation tendrá generation.

## DB-TCR-042

New operations usarán current credential generation.

## DB-TCR-043

Old credential pools podrán ser drained.

## DB-TCR-044

Credential rotation no mutará live authenticated connection.

## DB-TCR-045

PoolKey representará physical compatibility.

## DB-TCR-046

PoolKey no incluirá TenantId obligatoriamente.

## DB-TCR-047

TenantId entrará en PoolKey sólo cuando isolation lo requiera.

## DB-TCR-048

Pool per tenant no será default.

## DB-TCR-049

Pool count será bounded.

## DB-TCR-050

Idle pools podrán ser retired.

## DB-TCR-051

Placement generation change podrá drain old pools.

## DB-TCR-052

New generation no reutilizará stale incompatible leases.

## DB-TCR-053

ConnectionLease tendrá explicit ownership.

## DB-TCR-054

ConnectionLease podrá portar tenant resolution metadata.

## DB-TCR-055

Lease no cambiará de tenant mientras esté activo.

## DB-TCR-056

Physical connection podrá reutilizarse después de clean release.

## DB-TCR-057

One physical connection tendrá one active owner por default.

## DB-TCR-058

Multiplexing requerirá capability explícita.

## DB-TCR-059

Tenant transaction tendrá connection affinity.

## DB-TCR-060

Tenant transaction tendrá placement affinity.

## DB-TCR-061

Tenant transaction tendrá shard affinity cuando corresponda.

## DB-TCR-062

Transaction no migrará de placement silenciosamente.

## DB-TCR-063

Transaction no migrará de tenant.

## DB-TCR-064

Placement cutover no moverá active transaction automáticamente.

## DB-TCR-065

UNKNOWN transaction outcome permanecerá UNKNOWN.

## DB-TCR-066

Retry no implicará arbitrary placement switch.

## DB-TCR-067

Shard resolution será explícita.

## DB-TCR-068

UNKNOWN shard no implicará default shard.

## DB-TCR-069

Multi-shard operation no será representada como una conexión única falsa.

## DB-TCR-070

Tenant no será equivalente a shard.

## DB-TCR-071

Un tenant podrá residir en múltiples shards si estrategia lo permite.

## DB-TCR-072

Partition routing decidirá shard cuando sea necesario.

## DB-TCR-073

Region constraints serán explícitas.

## DB-TCR-074

Data residency no será ignorada durante failover.

## DB-TCR-075

Failover será topology-aware.

## DB-TCR-076

Failover no violará tenant security policy.

## DB-TCR-077

Generation domains serán distinguibles.

## DB-TCR-078

Placement generation no será confundida con topology generation.

## DB-TCR-079

Topology generation no será confundida con credential generation.

## DB-TCR-080

Resolution podrá portar generation token.

## DB-TCR-081

Stale resolution será detectable cuando exista generation evidence.

## DB-TCR-082

Pre-side-effect stale resolution podrá re-resolverse.

## DB-TCR-083

Post-side-effect resolution no cambiará silenciosamente.

## DB-TCR-084

Tenant mobility será explícita.

## DB-TCR-085

Tenant mobility no implicará hidden dual writes.

## DB-TCR-086

Dual writes requerirán protocolo especializado.

## DB-TCR-087

Cutover cambiará placement generation cuando corresponda.

## DB-TCR-088

Old generation leases no serán usadas para new-generation operations.

## DB-TCR-089

Placement cache no será source of truth por definición.

## DB-TCR-090

Cache hit no equivaldrá a usable placement.

## DB-TCR-091

TTL no equivaldrá a correctness.

## DB-TCR-092

Control-plane outage tendrá política explícita.

## DB-TCR-093

Unknown placement fallará cerrado.

## DB-TCR-094

Resolution failure no usará default tenant database.

## DB-TCR-095

Missing TenantContext no implicará global DB.

## DB-TCR-096

Resolution failure será distinta de acquisition failure.

## DB-TCR-097

Retry policy distinguirá resolution y acquisition.

## DB-TCR-098

Circuit breakers seguirán real failure domains.

## DB-TCR-099

Shared cluster no requerirá breaker por tenant por default.

## DB-TCR-100

Tenant resource limits podrán restringir acquisitions.

## DB-TCR-101

Global resource limits también aplicarán.

## DB-TCR-102

Tenant quota no será equivalente a dedicated pool.

## DB-TCR-103

Acquisition wait será bounded.

## DB-TCR-104

Acquisition wait será cancellable.

## DB-TCR-105

Acquisition wait será deadline-aware.

## DB-TCR-106

Resource fairness será soportable.

## DB-TCR-107

Tenant A no monopolizará capacidad si policy exige fairness.

## DB-TCR-108

Persistent workers no mantendrán current tenant connection global.

## DB-TCR-109

Resolution context será execution-scoped.

## DB-TCR-110

Worker cache podrá compartir placement metadata segura.

## DB-TCR-111

Worker cache no compartirá current tenant intent.

## DB-TCR-112

OpenSwoole coroutines tendrán resolution isolation.

## DB-TCR-113

Shared pool no implicará shared active connection.

## DB-TCR-114

Connection reuse dependerá de verified cleanliness.

## DB-TCR-115

Security tendrá prioridad sobre connection reuse.

## DB-TCR-116

Isolation tendrá prioridad sobre pool efficiency.

## DB-TCR-117

Unknown cleanliness causará discard.

## DB-TCR-118

Tenant session reset failures serán observables.

## DB-TCR-119

Tenant connection resolution será observable.

## DB-TCR-120

TenantId no será metric label por default.

## DB-TCR-121

Telemetry dimensions serán bounded.

## DB-TCR-122

Credentials nunca serán telemetry attributes.

## DB-TCR-123

Diagnostics redactará secretos.

## DB-TCR-124

Public errors no revelarán topology innecesaria.

## DB-TCR-125

ResolutionId permitirá correlación interna.

## DB-TCR-126

Tenant connection resolution tendrá tests de isolation.

## DB-TCR-127

Connection reuse tendrá contamination tests.

## DB-TCR-128

Credential rotation tendrá tests.

## DB-TCR-129

Placement mobility tendrá tests.

## DB-TCR-130

Transaction affinity tendrá tests.

## DB-TCR-131

Sticky isolation tendrá tests.

## DB-TCR-132

Replica routing tendrá tests.

## DB-TCR-133

Failover tendrá tests.

## DB-TCR-134

Pool explosion tendrá tests.

## DB-TCR-135

Persistent runtime tendrá tests.

## DB-TCR-136

Concurrent runtime tendrá tests.

## DB-TCR-137

Identifier injection tendrá tests.

## DB-TCR-138

Unknown placement tendrá fail-closed tests.

## DB-TCR-139

Session reset tendrá fault-injection tests.

## DB-TCR-140

Connection resolution no generará SQL.

## DB-TCR-141

Connection resolution no ejecutará queries de aplicación.

## DB-TCR-142

Connection resolution no manipulará ORM entities.

## DB-TCR-143

Connection resolution no iniciará transaction.

## DB-TCR-144

ConnectionManager seguirá siendo connection lifecycle authority.

## DB-TCR-145

Driver seguirá siendo protocol authority.

## DB-TCR-146

Platform seguirá siendo capability authority.

## DB-TCR-147

Topology seguirá siendo endpoint authority.

## DB-TCR-148

Tenant placement seguirá siendo tenant-location authority.

## DB-TCR-149

Authorization seguirá siendo access authority.

## DB-TCR-150

TenantContext seguirá siendo effective tenant identity.

## DB-TCR-151

Resolution será deterministic bajo mismo context/generation/topology state.

## DB-TCR-152

Resolution será explainable.

## DB-TCR-153

Resolution será testable sin abrir conexión real.

## DB-TCR-154

Resolution podrá ser cacheable bajo generation contract.

## DB-TCR-155

Resolution no esconderá uncertainty.

## DB-TCR-156

UNKNOWN no será convertido en valid target.

## DB-TCR-157

No habrá implicit fallback a global database.

## DB-TCR-158

No habrá implicit fallback a arbitrary shard.

## DB-TCR-159

No habrá implicit fallback a unauthorized region.

## DB-TCR-160

Tenant connection resolution preservará aislamiento antes de optimizar reutilización.

---

# 225. Modelo formal

Sea:

```text
T = TenantContext
O = OperationIntent
P = TenantPlacement
X = TransactionContext
S = StickyState
C = ConsistencyPolicy
G = ResourceGovernance
```

Entonces:

```text
ResolveConnection(T,O,X,S,C,G)
→
TenantConnectionResolution
```

donde:

```text
TenantConnectionResolution
=
(
 LogicalDatabase,
 Placement,
 Shard,
 Region,
 Role,
 Target,
 Credentials,
 SessionProfile,
 PoolKey,
 Generations
)
```

---

# 226. Condición de resolución válida

Una resolución `R` será válida sólo si:

```text
Valid(R)
=
TenantValid
∧
PlacementKnown
∧
PlacementGenerationValid
∧
RoutingCompatible
∧
TransactionAffinityPreserved
∧
ConsistencySatisfied
∧
CredentialGenerationValid
∧
SecurityPolicySatisfied
∧
ResourcePolicySatisfied
```

---

# 227. Condición de acquisition

Posteriormente:

```text
Acquire(R)
→
ConnectionLease
```

será responsabilidad del ConnectionManager.

Por tanto:

```text
Resolve(T)
≠
AcquireConnection(T)
```

---

# 228. Condición de reuse

Una conexión usada por Tenant A podrá ser reutilizada por B únicamente si:

```text
ReusableFor(C,B)
=
LeaseReleased(C)
∧
NoTransaction(C)
∧
NoActiveResult(C)
∧
TenantStateReset(C)
∧
GenericStateReset(C)
∧
CredentialCompatibility(C,B)
∧
PlacementCompatibility(C,B)
∧
SecurityCompatibility(C,B)
∧
VerificationPassed(C)
```

---

# 229. Unknown rule

Si cualquier condición crítica es:

```text
UNKNOWN
```

la política conservadora será:

```text
DISCARD / REJECT
```

según se trate de conexión o resolución.

---

# 230. Arquitectura completa

```text
                       APPLICATION
                            │
                            ▼
                       TenantContext
                            │
                            ▼
                TenantConnectionResolver
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
      Placement         Operation         Existing
      Resolver           Intent           Affinity
          │                 │                 │
          └─────────────────┼─────────────────┘
                            ▼
                   Logical DB / Shard
                            │
                            ▼
                  Read / Write Routing
                            │
                            ▼
                   Topology Resolution
                            │
                ┌───────────┼───────────┐
                ▼           ▼           ▼
             Writer      Replica     Admin
                │           │           │
                └───────────┼───────────┘
                            ▼
                  Credential Resolver
                            │
                            ▼
                  Session Profile
                            │
                            ▼
                       Pool Key
                            │
                            ▼
              TenantConnectionResolution
                            │
════════════════════════════╪════════════════════════════
                            │
                    ConnectionManager
                            │
                            ▼
                     Connection Pool
                            │
                            ▼
                    Connection Lease
                            │
                            ▼
                Tenant Session Init
                            │
                            ▼
                      DB Operation
                            │
                            ▼
                 Tenant Session Reset
                            │
                            ▼
                       Verification
                            │
                    ┌───────┴────────┐
                    ▼                ▼
                  CLEAN            UNKNOWN
                    │                │
                    ▼                ▼
               Return Pool        DISCARD
```

---

# 231. Diseño evolutivo

Esta arquitectura permite que una aplicación empiece con:

```text
Tenant A ─┐
Tenant B ─┼── Shared Database
Tenant C ─┘
```

y posteriormente evolucione a:

```text
Tenant A → Shared Database
Tenant B → Dedicated Database
Tenant C → Dedicated Cluster
```

sin cambiar:

```php
Order::query()
    ->where('status', 'pending')
    ->get();
```

La diferencia se resuelve en:

```text
Tenant Placement
+
Connection Resolution
```

no en el código de negocio.

---

# 232. Principio final

El Tenant Connection Resolution System seguirá estas reglas:

```text
IDENTIFY TENANT
```

```text
RESOLVE PLACEMENT
```

```text
PRESERVE GENERATIONS
```

```text
PRESERVE TRANSACTION AFFINITY
```

```text
RESOLVE SHARD BEFORE CONNECTION
```

```text
RESOLVE ROLE BEFORE ENDPOINT
```

```text
RESOLVE CREDENTIALS SECURELY
```

```text
POOL BY PHYSICAL COMPATIBILITY
```

```text
LEASE WITH EXPLICIT OWNERSHIP
```

```text
INITIALIZE TENANT SESSION EXPLICITLY
```

```text
RESET BEFORE REUSE
```

```text
DISCARD WHEN CLEANLINESS IS UNKNOWN
```

```text
FAIL CLOSED WHEN PLACEMENT IS UNKNOWN
```

La regla arquitectónica definitiva será:

> **VoltStack nunca deberá encontrar una base de datos “adivinando” a partir del TenantId. El TenantConnectionResolver transformará identidad y contexto en una resolución explícita, versionada, segura y verificable; ConnectionManager será quien convierta posteriormente esa resolución en un ConnectionLease. Esta separación permitirá compartir infraestructura cuando sea seguro, dedicarla cuando sea necesario y mover tenants entre databases, shards, regiones o clusters sin romper el modelo de dominio ni las garantías de aislamiento.**

---

# 233. Estado del Bloque 26

```text
BLOCK 26 — MULTITENANCY INTEGRATION

✓ 261_DATABASE_MULTITENANCY_INTEGRATION_ARCHITECTURE.md
✓ 262_DATABASE_TENANT_CONNECTION_RESOLUTION_SYSTEM.md
│
├── 263_DATABASE_TENANT_DATABASE_ISOLATION_SYSTEM.md
├── 264_DATABASE_TENANT_SCHEMA_ISOLATION_SYSTEM.md
├── 265_DATABASE_TENANT_QUERY_CONTEXT_SYSTEM.md
└── 266_DATABASE_TENANT_MIGRATION_INTEGRATION_SYSTEM.md
```

---

# 234. Siguiente documento

```text
263_DATABASE_TENANT_DATABASE_ISOLATION_SYSTEM.md
```

El siguiente documento definirá específicamente cómo VoltStack garantizará aislamiento cuando uno o varios tenants utilicen:

```text
Shared Database
Database per Tenant
Dedicated Database
Dedicated Cluster
Shard-per-Tenant
Hybrid Placement
```

incluyendo:

```text
Isolation Domains
Tenant Persistence Domain
Database Boundaries
ORM Identity Isolation
Connection Isolation
Transaction Isolation
Relationship Isolation
Cache Isolation
Read/Write Isolation
Shard Isolation
Credential Isolation
Cross-Tenant Protection
Administrative Access
Data Movement
Tenant Mobility
Backup/Restore Boundaries
Failure Domains
Persistent Runtime Isolation
Security
Telemetry
Testing
```

bajo la regla:

> **El aislamiento de base de datos de un tenant no se considerará correcto simplemente porque las conexiones apunten a bases diferentes; VoltStack deberá preservar la identidad del tenant y su dominio de persistencia a través del ORM, las transacciones, las conexiones, el cache, el routing, los runtimes persistentes y cualquier operación administrativa que pueda cruzar fronteras de tenant.**