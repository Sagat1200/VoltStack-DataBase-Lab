# 261_DATABASE_MULTITENANCY_INTEGRATION_ARCHITECTURE.md

# VoltStack Quantum Database
## Database Multitenancy Integration Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Integración:** `VoltStack/Quantum/Multitenancy`  
**Documento:** 261 — Database Multitenancy Integration Architecture  
**Bloque:** 26 — Multitenancy Integration  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `260_DATABASE_OPENSWOOLE_INTEGRATION_SYSTEM.md`  
**Siguiente documento:** `262_DATABASE_TENANT_CONNECTION_RESOLUTION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura de integración entre:

```text
VoltStack/Quantum/Database
```

y el paquete oficial opcional:

```text
VoltStack/Quantum/Multitenancy
```

La integración permitirá que aplicaciones VoltStack operen con múltiples tenants mediante estrategias como:

```text
Shared Database / Shared Schema
Shared Database / Separate Schema
Database per Tenant
Cluster or Shard per Tenant
Hybrid Tenant Placement
```

sin convertir Multitenancy en una dependencia obligatoria del sistema Database.

La regla central será:

> **VoltStack Database será completamente funcional y tenant-agnostic por sí mismo; Multitenancy añadirá contexto, resolución, aislamiento, routing y políticas mediante contratos explícitos de integración.**

Formalmente:

```text
DatabaseCore
⊄
Multitenancy
```

mientras:

```text
Multitenancy
        ↓
Database Integration Contracts
        ↓
Database
```

---

# 2. Decisión arquitectónica fundamental

Multitenancy no formará parte obligatoria de:

```text
Database Core
ORM
Query Builder
Query Engine
SQL Compiler
Driver
Connection
Schema
Migration
Transaction
Hydration
```

En consecuencia, una aplicación sin multitenancy utilizará:

```text
Application
    ↓
Database
```

mientras una aplicación multitenant utilizará:

```text
Application
    ↓
Multitenancy
    ↓
Database Integration
    ↓
Database
```

---

# 3. Multitenancy como paquete oficial opcional

Arquitectura conceptual:

```text
VoltStack Framework
│
├── Quantum/Database
│
├── Quantum/Auth
│
├── Quantum/Authorization
│
├── Quantum/Cache
│
├── Quantum/Events
│
│
└── Optional Official Packages
    │
    ├── Quantum/Multitenancy
    └── Quantum/SaaS
```

Por tanto:

```text
Database ≠ Multitenancy
Multitenancy ≠ SaaS
SaaS ≠ Database
```

aunque puedan integrarse profundamente.

---

# 4. Multitenancy ≠ SaaS

Una aplicación puede ser:

```text
Multitenant
```

sin ser SaaS.

Ejemplo:

```text
sistema interno multiempresa
```

También puede existir:

```text
SaaS
```

sin aislamiento Database por tenant.

Ejemplo:

```text
single-database SaaS
+
tenant_id
```

Por tanto:

```text
Multitenancy
≠
Billing
≠
Subscriptions
≠
SaaS
```

---

# 5. Objetivos

La integración deberá proporcionar:

1. Tenant Context.
2. Tenant Resolution.
3. Tenant Connection Resolution.
4. Tenant Database Isolation.
5. Tenant Schema Isolation.
6. Tenant Query Context.
7. Tenant Query Scoping.
8. Tenant Migration Integration.
9. Tenant-aware transactions.
10. Tenant-aware connection pooling.
11. Tenant-aware read/write routing.
12. Tenant-aware caching.
13. Tenant-aware ORM identity.
14. Tenant-aware telemetry.
15. Tenant-aware auditing.
16. Tenant-aware security.
17. Tenant-aware persistent runtime isolation.
18. Tenant-aware sharding.
19. Tenant placement.
20. Tenant topology generations.
21. Tenant resource governance.
22. Tenant lifecycle integration.
23. Cross-tenant access controls.
24. privileged administration.
25. safe tenant switching.
26. testing and conformance.

---

# 6. No objetivos

Este bloque no definirá directamente:

```text
subscription billing
pricing
plans
invoices
payments
customer onboarding
SaaS product management
```

Esas responsabilidades pertenecerán al futuro:

```text
VoltStack/Quantum/SaaS
```

---

# 7. Terminología

## Tenant

Unidad lógica aislada de una aplicación.

Puede representar:

```text
Company
Organization
Workspace
Account
Customer
Institution
Business Unit
```

según dominio.

---

# 8. Tenant ID

Identificador estable del tenant.

Ejemplo conceptual:

```php
final readonly class TenantId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 9. Tenant ≠ Database

Un tenant puede estar ubicado en:

```text
one database
one schema
one shard
multiple databases
multiple regions
```

Por tanto:

```text
Tenant
≠
Database
```

---

# 10. Tenant ≠ Connection

Igualmente:

```text
Tenant
≠
Connection
```

Una conexión es un recurso físico/lógico temporal.

---

# 11. Tenant ≠ User

Un usuario puede:

```text
belong to one tenant
belong to multiple tenants
administer multiple tenants
```

Por tanto:

```text
TenantContext
≠
AuthenticationContext
```

---

# 12. TenantContext

Representará el tenant efectivo de una ejecución.

Ejemplo:

```php
final readonly class TenantContext
{
    public function __construct(
        public TenantId $tenantId,
        public TenantIsolationStrategy $strategy,
        public TenantPlacement $placement,
        public TenantContextGeneration $generation,
    ) {}
}
```

---

# 13. TenantContext ≠ Tenant Entity

`TenantContext` será información operacional.

No deberá requerir que:

```text
Tenant
```

sea una entidad ORM específica.

---

# 14. Tenant Resolution

Antes de ejecutar una operación tenant-aware deberá existir:

```text
Execution
    ↓
Tenant Resolver
    ↓
TenantContext
```

El tenant podría resolverse desde capas superiores mediante:

```text
domain
subdomain
route
authenticated account
API key
job metadata
CLI option
explicit application context
```

Database no deberá resolver HTTP domains directamente.

---

# 15. Boundary rule

Database recibirá:

```text
TenantContext
```

pero no conocerá necesariamente:

```text
HTTP Request
JWT
Session
Domain Name
Route
```

---

# 16. Tenant resolution ownership

Conceptualmente:

```text
HTTP Layer
    ↓
Multitenancy Package
    ↓
TenantContext
    ↓
Database Integration
```

---

# 17. DatabaseContext integration

El documento:

```text
253_DATABASE_DATABASE_CONTEXT_SYSTEM.md
```

definió `DatabaseContext`.

Multitenancy extenderá ese contexto conceptualmente:

```text
DatabaseContext
│
├── logical database
├── execution
├── consistency
├── routing
├── security
│
└── optional TenantContext
```

---

# 18. Optional tenant dimension

Una aplicación normal podrá tener:

```text
DatabaseContext
tenant = null
```

sin comportamiento especial.

---

# 19. Tenant-aware context

Aplicación multitenant:

```text
DatabaseContext
tenant = TenantContext(...)
```

---

# 20. Core neutrality

Database no deberá asumir que:

```text
tenant != null
```

siempre.

---

# 21. Tenant isolation strategies

VoltStack deberá poder representar al menos:

```php
enum TenantIsolationStrategy
{
    case SHARED_DATABASE_SHARED_SCHEMA;
    case SHARED_DATABASE_SEPARATE_SCHEMA;
    case DATABASE_PER_TENANT;
    case SHARD_PER_TENANT;
    case CLUSTER_PER_TENANT;
    case HYBRID;
}
```

---

# 22. Shared Database / Shared Schema

Modelo:

```text
Database
│
├── Tenant A rows
├── Tenant B rows
└── Tenant C rows
```

usualmente mediante:

```text
tenant_id
```

---

# 23. Shared Database / Separate Schema

Ejemplo:

```text
Database
│
├── schema_tenant_a
├── schema_tenant_b
└── schema_tenant_c
```

---

# 24. Database per Tenant

```text
Tenant A → Database A
Tenant B → Database B
Tenant C → Database C
```

---

# 25. Shard per Tenant

```text
Tenant A ─┐
Tenant B ─┼── Shard 1
Tenant C ─┘

Tenant D ─┐
Tenant E ─┼── Shard 2
Tenant F ─┘
```

---

# 26. Cluster per Tenant

Casos de aislamiento elevado:

```text
Tenant Enterprise A
        ↓
Dedicated Database Cluster
```

---

# 27. Hybrid

Ejemplo:

```text
Small Tenants
      ↓
Shared Database

Medium Tenants
      ↓
Dedicated Schema

Enterprise Tenants
      ↓
Dedicated Database/Cluster
```

---

# 28. Tenant Placement

La ubicación física/lógica deberá representarse independientemente.

Ejemplo:

```php
final readonly class TenantPlacement
{
    public function __construct(
        public LogicalDatabaseId $logicalDatabase,
        public ?ShardId $shard,
        public ?DatabaseName $database,
        public ?SchemaName $schema,
        public ?RegionId $region,
        public TenantPlacementGeneration $generation,
    ) {}
}
```

---

# 29. Isolation Strategy ≠ Placement

Importante:

```text
IsolationStrategy
```

describe **cómo** se aísla.

Mientras:

```text
TenantPlacement
```

describe **dónde** se encuentra.

---

# 30. Tenant placement resolver

```text
TenantId
   ↓
TenantPlacementResolver
   ↓
TenantPlacement
```

---

# 31. Placement resolution

Podrá provenir de:

```text
configuration
tenant registry
control plane
metadata database
cache
service discovery
custom resolver
```

---

# 32. No hardcoded database naming

Evitar:

```php
$database = 'tenant_' . $tenantId;
```

como arquitectura universal.

---

# 33. Why

Porque no soportaría correctamente:

```text
tenant migration
renaming
sharding
regions
dedicated clusters
hybrid isolation
credential rotation
```

---

# 34. Tenant registry

El paquete Multitenancy podrá mantener un:

```text
TenantRegistry
```

con información de:

```text
tenant identity
status
placement
isolation strategy
generation
capabilities
```

---

# 35. Registry ≠ ORM requirement

El TenantRegistry no deberá depender necesariamente de una entidad ORM `Tenant`.

---

# 36. TenantConnectionResolver

Arquitectura:

```text
TenantContext
      ↓
TenantConnectionResolver
      ↓
ConnectionIntent
      ↓
ConnectionManager
      ↓
ConnectionLease
```

---

# 37. Resolver does not open connections

Regla:

> **TenantConnectionResolver resuelve intención y destino; ConnectionManager adquiere la conexión.**

No:

```text
TenantResolver → PDO
```

---

# 38. TenantConnectionIntent

Ejemplo conceptual:

```php
final readonly class TenantConnectionIntent
{
    public function __construct(
        public TenantId $tenant,
        public LogicalDatabaseId $logicalDatabase,
        public ?ShardId $shard,
        public ?SchemaName $schema,
        public ReadWriteIntent $intent,
        public TenantPlacementGeneration $generation,
    ) {}
}
```

---

# 39. Tenant resolution pipeline

```text
TenantContext
      ↓
Placement Resolution
      ↓
Logical Database
      ↓
Shard Resolution
      ↓
Read/Write Routing
      ↓
Replica Eligibility
      ↓
Load Balancing
      ↓
Connection Acquisition
```

---

# 40. Ordering matters

No deberá ocurrir:

```text
choose random connection
        ↓
discover tenant
```

Debe ocurrir:

```text
resolve tenant
        ↓
resolve placement
        ↓
resolve endpoint
        ↓
acquire connection
```

---

# 41. Tenant Query Context

Toda query tenant-aware deberá poder incluir:

```text
TenantQueryContext
```

---

# 42. Shared-schema scoping

Para:

```text
SHARED_DATABASE_SHARED_SCHEMA
```

el sistema podrá aplicar:

```text
tenant predicate
```

a entidades/tablas declaradas tenant-scoped.

---

# 43. Example

Consulta lógica:

```php
Order::query()->where('status', 'pending')->get();
```

podrá representar semánticamente:

```text
status = 'pending'
AND tenant_id = CurrentTenant
```

sin concatenar SQL manualmente.

---

# 44. Tenant scope belongs before compilation

Debe integrarse:

```text
Query Model
    ↓
Tenant Semantic Scope
    ↓
Validation
    ↓
Optimizer
    ↓
Planner
    ↓
Compiler
```

No:

```text
compiled SQL
    ↓
string append "AND tenant_id = ..."
```

---

# 45. Tenant predicate is semantic

Por tanto:

```text
TenantScope
≠
SQL String Manipulation
```

---

# 46. Tenant metadata

ORM metadata podrá declarar:

```text
tenant scoped entity
global entity
tenant key
tenant relationship rules
cross-tenant relationship policy
```

---

# 47. Global entities

Algunas entidades podrán ser globales:

```text
Country
Currency
FrameworkMetadata
```

según aplicación.

---

# 48. Tenant-scoped entities

Ejemplos:

```text
Customer
Order
Invoice
Project
```

---

# 49. Tenant key metadata

Ejemplo:

```php
#[TenantScoped(key: 'tenant_id')]
final class Order
{
}
```

La sintaxis final se definirá en Multitenancy, no necesariamente en Database Core.

---

# 50. ORM IdentityMap

La identidad canonical deberá considerar tenant cuando sea necesario.

Formalmente:

```text
IdentityKey
=
EntityType
+
EntityIdentifier
+
PersistenceDomain
+
TenantContext
```

cuando el tenant forme parte del dominio de identidad.

---

# 51. Critical example

Tenant A:

```text
User#10
```

Tenant B:

```text
User#10
```

no deberán resolver al mismo objeto managed.

---

# 52. IdentityMap invariant

```text
Identity(A, User, 10)
≠
Identity(B, User, 10)
```

---

# 53. Entity cache

También deberá incluir dimensión tenant cuando corresponda.

---

# 54. Query cache

Igual.

---

# 55. Result cache

Igual.

---

# 56. Metadata cache

No siempre necesita tenant.

Metadata estructural compartida podrá reutilizarse.

---

# 57. Cache key rule

Si un valor depende del tenant:

```text
TenantDimension
```

deberá formar parte de su identidad de cache.

---

# 58. Tenant switching

Cambiar:

```text
Tenant A
→
Tenant B
```

dentro de un mismo execution scope será una operación sensible.

---

# 59. Default policy

El cambio implícito de tenant con un EntityManager activo deberá rechazarse.

---

# 60. Why

Podrían existir:

```text
managed entities
dirty entities
active transaction
connection lease
lazy relations
cached query context
```

pertenecientes al tenant anterior.

---

# 61. Safe tenant switching

Por defecto:

```text
close current tenant scope
        ↓
finalize resources
        ↓
new tenant scope
```

---

# 62. Explicit cross-tenant operations

Para administración podrán existir APIs explícitas.

Ejemplo conceptual:

```php
$tenants->forTenant($tenantId, function () {
    // isolated tenant execution
});
```

---

# 63. forTenant()

No deberá simplemente modificar:

```text
Tenant::$current
```

globalmente.

Deberá crear un contexto aislado.

---

# 64. Nested tenant scope

Si se soporta:

```text
Tenant A Scope
    ↓
Administrative Tenant B Scope
```

deberá preservar/restaurar el contexto padre explícitamente.

---

# 65. Cross-tenant operation ≠ normal query

Debe ser distinguible en:

```text
authorization
audit
telemetry
resource governance
```

---

# 66. Cross-tenant authorization

El hecho de conocer:

```text
TenantId
```

no autoriza acceso.

Formalmente:

```text
ResolvableTenant
≠
AuthorizedTenant
```

---

# 67. Tenant context ≠ authorization

También:

```text
TenantContext
≠
Permission
```

---

# 68. Authorization integration

Flujo:

```text
Principal
    ↓
Authorization
    ↓
Allowed Tenant Scope
    ↓
TenantContext
    ↓
Database Operation
```

---

# 69. Defense in depth

Database Multitenancy podrá validar:

```text
query tenant context
entity tenant ownership
connection placement
transaction tenant affinity
```

pero no reemplaza Authorization.

---

# 70. Tenant transaction affinity

Una transaction deberá estar asociada al tenant efectivo.

```text
Transaction
    ↓
Tenant A
```

No deberá cambiar a Tenant B.

---

# 71. Invariant

```text
TransactionTenant(T)
=
TenantAtBegin(T)
```

durante toda la transaction.

---

# 72. Tenant switching in transaction

Prohibido por default:

```text
BEGIN Tenant A
    ↓
switch Tenant B
    ↓
query
```

---

# 73. Cross-tenant transaction

No deberá fingirse una transacción ACID global.

---

# 74. Example

```text
Tenant A → Database A
Tenant B → Database B
```

Una operación sobre ambos:

```text
Transaction A + Transaction B
```

no equivale automáticamente a:

```text
GlobalTransaction
```

---

# 75. Distributed transaction rule

```text
Multiple Tenant Transactions
≠
Distributed ACID
```

---

# 76. Tenant-aware read/write routing

Dentro de Tenant A:

```text
Tenant Placement
      ↓
Replication Group
      ↓
Writer / Replicas
```

---

# 77. Sticky state

Read-your-writes sticky state deberá incluir:

```text
tenant
logical database
shard
```

cuando corresponda.

---

# 78. Replica health

Puede compartirse por endpoint.

---

# 79. Tenant read intent

Debe resolverse antes de replica selection.

---

# 80. Tenant failover

Un tenant puede cambiar de placement durante failover/migration.

---

# 81. Placement generation

Por ello:

```text
TenantPlacementGeneration
```

será importante.

---

# 82. Generation example

```text
Tenant A
Generation 41
Database Cluster X
```

migra a:

```text
Tenant A
Generation 42
Database Cluster Y
```

---

# 83. Existing transaction

Una transaction iniciada en generación 41 no migrará automáticamente.

---

# 84. New execution

Podrá utilizar generación 42.

---

# 85. Mid-execution topology change

Deberá seguir las reglas de consistency y runtime context.

---

# 86. Placement cache

Podrá cachearse.

Pero:

```text
PlacementCache
≠
PlacementTruth
```

---

# 87. Stale placement

Debe detectarse/invalidarse mediante:

```text
generation
TTL as freshness aid
events
control-plane updates
```

según implementación.

---

# 88. TTL ≠ correctness

Un TTL no garantiza placement correcto.

---

# 89. Tenant database isolation

Para `DATABASE_PER_TENANT`:

```text
TenantContext
      ↓
Placement
      ↓
Database
```

La selección será explícita.

---

# 90. Tenant schema isolation

Para `SEPARATE_SCHEMA`:

```text
TenantContext
      ↓
Schema Resolution
      ↓
Connection Session State
```

o SQL qualification según estrategia.

---

# 91. Schema switching

Si se modifica estado de conexión:

```text
search_path
current_schema
database session namespace
```

deberá restaurarse antes del reuse.

---

# 92. Connection pooling danger

Ejemplo:

```text
Connection C
used by Tenant A
schema = tenant_a
```

luego:

```text
Connection C
used by Tenant B
```

sin reset sería una violación crítica.

---

# 93. Connection cleanliness

Antes de reutilizar:

```text
tenant-specific database state = reset
```

deberá verificarse.

---

# 94. Unknown tenant state

Si no puede verificarse:

```text
DISCARD CONNECTION
```

---

# 95. Tenant-aware connection pools

Podrán existir diferentes modelos:

```text
shared physical pool
partitioned pools
pool per placement
pool per credential set
dedicated tenant pool
```

---

# 96. No automatic pool per tenant

Crear:

```text
1 pool × every tenant
```

podría causar explosión de recursos.

---

# 97. Example

```text
100,000 tenants
×
5 idle connections
=
500,000 connections
```

inaceptable.

---

# 98. Pool architecture

Preferir:

```text
Tenant
   ↓
Placement
   ↓
Pool Key
```

donde múltiples tenants puedan compartir infraestructura cuando sea seguro.

---

# 99. PoolKey

Puede incluir:

```text
endpoint
credentials
database
role
platform
generation
```

según estrategia.

---

# 100. Tenant ID in PoolKey

Sólo cuando el aislamiento físico lo requiera.

---

# 101. Credential isolation

Algunos tenants pueden usar credenciales dedicadas.

Entonces:

```text
TenantPlacement
+
CredentialIdentity
```

afectará el pool.

---

# 102. Credentials ≠ tenant ID

Nunca derivar contraseñas mediante concatenación simple del tenant ID.

---

# 103. Credential rotation

Debe ser generation-aware.

---

# 104. ORM relationships

Una relación entre entidades tenant-scoped deberá respetar compatibilidad tenant.

---

# 105. Default relation rule

```text
Entity A Tenant
=
Entity B Tenant
```

para relaciones tenant-local.

---

# 106. Cross-tenant relationships

Rechazadas por default.

---

# 107. Why

Pueden romper:

```text
isolation
foreign key assumptions
placement
sharding
security
```

---

# 108. Explicit global relationships

Una tenant entity sí podrá relacionarse con entidad global si metadata lo permite.

Ejemplo:

```text
Tenant Order
    ↓
Global Currency
```

---

# 109. Relationship loading

Lazy/eager/batch loading deberá conservar TenantContext.

---

# 110. Lazy loading

Un proxy/relationship lazy no podrá resolver usando el tenant actual arbitrariamente si la entidad pertenece a otro contexto.

---

# 111. Detached tenant entity

No deberá recuperar un global EntityManager y resolver relaciones fuera de contexto.

---

# 112. N+1 telemetry

Correlación N+1 será tenant-safe.

No mezclar queries de tenants distintos.

---

# 113. Hydration

Una entidad tenant-scoped hidratada deberá quedar asociada a su dominio tenant efectivo.

---

# 114. Persistence

Persistir entidad de Tenant A bajo Tenant B deberá producir error.

---

# 115. Tenant ownership validation

Antes de persistencia:

```text
EntityTenant
=
PersistenceTenant
```

cuando corresponda.

---

# 116. New entities

Para entidades nuevas tenant-scoped:

```text
TenantId
```

podrá asignarse mediante metadata/contexto controlado.

---

# 117. Mass assignment safety

El usuario no deberá poder cambiar arbitrariamente:

```text
tenant_id
```

mediante input no confiable.

---

# 118. Tenant key immutability

Por default:

```text
tenant_id
```

será immutable después de que la entidad entre en estado managed/persisted.

---

# 119. Moving entity between tenants

No será un simple:

```sql
UPDATE tenant_id = ...
```

conceptualmente.

---

# 120. Tenant transfer

Deberá tratarse como operación explícita de dominio/migración.

Puede requerir:

```text
authorization
relationship validation
data movement
cache invalidation
audit
```

---

# 121. Query input security

El cliente no podrá introducir libremente:

```text
tenant_id
```

para escapar del TenantContext.

---

# 122. Example unsafe API

Evitar:

```php
Order::query()
    ->withoutTenantScope()
    ->where('tenant_id', $_GET['tenant'])
    ->get();
```

como flujo normal.

---

# 123. Scope bypass

Cualquier bypass deberá ser:

```text
explicit
privileged
auditable
bounded
```

---

# 124. Administrative query

Ejemplo conceptual:

```php
$database->crossTenant()
    ->authorizedBy($capability)
    ->query(...);
```

La API final se definirá posteriormente.

---

# 125. Raw SQL

Raw SQL representa un caso crítico.

---

# 126. Shared-schema raw SQL

Database no siempre puede inferir que:

```sql
SELECT * FROM orders
```

necesita:

```text
tenant_id
```

---

# 127. Raw SQL policy

En contexto multitenant podrá existir:

```php
enum TenantRawSqlPolicy
{
    case FORBID;
    case REQUIRE_EXPLICIT_SCOPE;
    case PRIVILEGED_ONLY;
}
```

---

# 128. Default recommendation

Para tablas tenant-scoped:

```text
REQUIRE_EXPLICIT_SCOPE
```

o política aún más estricta según aplicación.

---

# 129. Raw SQL ≠ automatic safe query

El sistema nunca afirmará aislamiento que no pueda demostrar.

---

# 130. UNKNOWN tenant safety

Si no puede determinarse:

```text
TenantQuerySafety = UNKNOWN
```

No:

```text
SAFE
```

---

# 131. Schema Builder

Creación de tablas tenant-scoped podrá integrarse mediante metadata/extension.

---

# 132. Migration architecture

Multitenancy extenderá:

```text
Migration Discovery
Migration Planner
Migration Executor
Migration Repository
```

mediante integration contracts.

---

# 133. Tenant migrations

Podrán existir:

```text
global migrations
tenant migrations
placement-specific migrations
```

---

# 134. Global migration

Ejecutada una vez sobre infraestructura global.

---

# 135. Tenant migration

Ejecutada sobre cada tenant/placement correspondiente.

---

# 136. Migration fan-out

Ejemplo:

```text
Migration M42
   │
   ├── Tenant A
   ├── Tenant B
   ├── Tenant C
   └── ...
```

---

# 137. Large tenant fleets

Con miles de tenants no deberá asumirse:

```text
run everything in one process/transaction
```

---

# 138. Migration orchestration

Necesitará:

```text
batching
concurrency limits
checkpoints
retry
progress
failure isolation
rollout waves
```

---

# 139. Migration success

No deberá declararse globalmente:

```text
SUCCESS
```

si algunos tenants fallaron.

---

# 140. Fleet migration state

Podrá ser:

```text
PENDING
RUNNING
PARTIAL
COMPLETED
FAILED
PAUSED
UNKNOWN
```

---

# 141. Migration version skew

Durante rollout:

```text
Tenant A → Schema V42
Tenant B → Schema V41
```

puede existir temporalmente.

---

# 142. Application compatibility

Zero-downtime tenant migrations deberán soportar ventanas de compatibilidad.

---

# 143. Metadata generation

Schema version puede afectar:

```text
metadata generation
query capabilities
compiled plans
```

---

# 144. Tenant schema generation

Cuando tenants tengan diferentes versiones, el contexto deberá conocer la generación efectiva cuando sea necesario.

---

# 145. Query compiler

No deberá introducir vendor/tenant hacks directamente.

Recibirá información semántica/plataforma ya resuelta.

---

# 146. Events

Eventos tenant-aware deberán portar:

```text
TenantContextIdentity
```

cuando corresponda.

---

# 147. Event listener safety

Un listener global no deberá depender de:

```text
Tenant::$current
```

mutable.

---

# 148. Async events

Si un evento se procesa después:

```text
TenantContext
```

deberá reconstruirse explícitamente desde metadata segura.

---

# 149. Jobs

Un job tenant-aware deberá transportar:

```text
TenantId
```

o una referencia segura suficiente para reconstruir el contexto.

---

# 150. Job payload

No debería serializar:

```text
live EntityManager
live Connection
live TenantContext with secrets
```

---

# 151. Job execution

```text
Job
 ↓
TenantId
 ↓
TenantResolver
 ↓
TenantContext
 ↓
DatabaseExecutionScope
```

---

# 152. Tenant deletion

Un job antiguo para tenant eliminado deberá fallar de forma controlada.

---

# 153. Tenant suspension

TenantRegistry podrá indicar:

```text
ACTIVE
SUSPENDED
MIGRATING
DISABLED
DELETING
DELETED
```

---

# 154. Database access policy

Estado del tenant puede afectar admission.

---

# 155. Suspended tenant

No necesariamente significa:

```text
database physically inaccessible
```

pero la política puede impedir operaciones normales.

---

# 156. Administrative operations

Podrán seguir permitidas mediante capability privilegiada.

---

# 157. Cache invalidation

Cambios tenant-scoped deberán invalidar sólo entradas relacionadas cuando sea posible.

---

# 158. No FLUSHALL

Nunca usar:

```text
FLUSHALL
```

como estrategia normal de aislamiento tenant.

---

# 159. Cache namespace

Conceptualmente:

```text
database:
  tenant:{tenantIdentity}:
      ...
```

aunque la implementación de keys puede ser hash/canonical.

---

# 160. Tenant cache generation

Puede ayudar durante:

```text
tenant reset
migration
placement movement
```

---

# 161. Telemetry

Cada operación podrá asociarse con:

```text
tenant context
```

pero con políticas estrictas de cardinalidad y privacidad.

---

# 162. Metrics warning

No utilizar automáticamente:

```text
tenant_id
```

como metric label.

---

# 163. Why

Con:

```text
100,000 tenants
```

se produciría cardinalidad extrema.

---

# 164. Tracing

Tenant identity puede registrarse bajo políticas de observabilidad controladas.

---

# 165. Logs

Deberán respetar:

```text
sensitive data policy
PII policy
security policy
```

---

# 166. Audit

Cross-tenant access deberá ser especialmente auditable.

---

# 167. Audit fields

Podrán incluir:

```text
actor
source tenant
target tenant
operation
reason
authorization capability
timestamp
outcome
```

según política.

---

# 168. Query audit

No necesita registrar valores sensibles completos.

---

# 169. Persistent runtimes

Multitenancy deberá respetar:

```text
251–260 Persistent Runtime Architecture
```

---

# 170. Critical invariant

En:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

TenantContext será:

```text
execution scoped
```

no:

```text
worker scoped
```

---

# 171. Sequential contamination

Nunca:

```text
Request A → Tenant A
Request ends

Request B
accidentally still Tenant A
```

---

# 172. Concurrent contamination

Especialmente OpenSwoole:

```text
Coroutine A → Tenant A
Coroutine B → Tenant B
```

deben permanecer aisladas simultáneamente.

---

# 173. State reset

Al terminar execution:

```text
TenantContext
TenantQueryContext
TenantConnectionIntent
TenantStickyState
TenantSecurityScope
```

deberán liberarse.

---

# 174. Connection state reset

La conexión también deberá eliminar cualquier estado tenant-specific.

---

# 175. Worker infrastructure

Sí podrá conservar:

```text
TenantPlacementCache
TenantMetadataCache
TenantRegistryClient
```

si son:

```text
bounded
generation-aware
thread/coroutine safe where applicable
non-request-specific
```

---

# 176. Tenant resource governance

Podrán definirse límites por tenant:

```text
max concurrent queries
max connections
max query time
max rows
max export size
max import size
```

---

# 177. Resource governance ≠ billing

Los límites técnicos no implican planes SaaS.

---

# 178. SaaS integration

En el futuro:

```text
SaaS Plan
    ↓
Entitlements
    ↓
Resource Policy
    ↓
Multitenancy / Database
```

podrá integrarse.

Pero Database no conocerá:

```text
Gold Plan
Enterprise Plan
Monthly Subscription
```

---

# 179. Tenant noisy-neighbor protection

ResourceGovernor podrá evitar:

```text
Tenant A
consumes all DB capacity
```

afectando a B/C/D.

---

# 180. Global and tenant budgets

```text
Global Database Budget
          │
    ┌─────┼─────┐
    ▼     ▼     ▼
Tenant A Tenant B Tenant C
```

---

# 181. Dedicated tenant

Un tenant con DB dedicada puede tener políticas diferentes.

---

# 182. Fairness

El scheduler/routing podrá considerar fairness sin convertir Database en un sistema de billing.

---

# 183. Backup integration

Backup deberá conocer placement tenant cuando se solicite backup tenant-specific.

---

# 184. Restore integration

Restore tenant-specific deberá validar:

```text
target tenant
target placement
schema compatibility
security authorization
```

---

# 185. Tenant restore ≠ whole database restore

En shared-schema puede requerir restauración lógica selectiva.

---

# 186. Data retention

Políticas pueden variar por tenant mediante capas superiores.

---

# 187. Tenant deletion

La eliminación de un tenant deberá distinguir:

```text
logical disable
soft deletion
data purge
database deletion
schema deletion
archive
```

---

# 188. Data purge safety

Nunca inferir automáticamente:

```text
delete tenant record
⇒
drop database
```

---

# 189. Explicit destructive operations

Deberán requerir:

```text
authorization
intent
validation
audit
safety checks
```

---

# 190. Testing architecture

Se requerirán:

```text
Unit Tests
Tenant Context Tests
Connection Resolution Tests
Database Isolation Tests
Schema Isolation Tests
Query Scope Tests
ORM Identity Tests
Relationship Tests
Transaction Tests
Cache Isolation Tests
Migration Tests
Runtime Isolation Tests
Concurrency Tests
Security Tests
Cross-Tenant Tests
Failure Injection Tests
```

---

# 191. Shared-schema test

Tenant A y B contienen:

```text
Order#100
```

Query A sólo deberá observar A.

Query B sólo B.

---

# 192. Identity test

```text
Tenant A → User#10 → Object A
Tenant B → User#10 → Object B
```

y:

```text
Object A !== Object B
```

---

# 193. Persistence test

Intentar persistir:

```text
Entity(Tenant A)
```

en:

```text
Context(Tenant B)
```

debe fallar.

---

# 194. Relationship test

Intentar:

```text
Order(Tenant A)
    ↓
Customer(Tenant B)
```

debe fallar salvo relación cross-tenant explícita.

---

# 195. Connection reuse test

```text
Tenant A → Connection C
reset
Tenant B → Connection C
```

B no deberá observar estado A.

---

# 196. Schema test

```text
Tenant A → schema_a
Tenant B → schema_b
```

verificar aislamiento.

---

# 197. Transaction test

Transaction Tenant A no podrá cambiar a B.

---

# 198. Cache test

Misma query/id entre A y B no compartirá valor tenant-specific.

---

# 199. Raw SQL test

Query tenant-unsafe deberá rechazarse bajo política estricta.

---

# 200. Persistent runtime test

Secuencia:

```text
A → B → C → A → C → B
```

en mismo worker.

Sin contaminación.

---

# 201. Concurrent runtime test

OpenSwoole:

```text
100 concurrent tenant contexts
```

con consultas simultáneas.

Verificar aislamiento.

---

# 202. Migration test

Simular:

```text
1000 tenants
```

con:

```text
success
failure
retry
partial rollout
version skew
```

---

# 203. Placement migration test

Mover Tenant A:

```text
Database X
→
Database Y
```

y verificar generation handling.

---

# 204. Failover test

Cambiar writer del placement sin migrar transaction activa.

---

# 205. Security test

Usuario autorizado para A intenta usar TenantId B.

Debe fallar antes de exposición de datos.

---

# 206. Proposed integration contracts

```text
TenantContextProvider
TenantPlacementResolver
TenantConnectionResolver
TenantQueryScopeProvider
TenantEntityScopeResolver
TenantMigrationScopeProvider
TenantResourcePolicyProvider
TenantAuditContextProvider
```

---

# 207. Contract location

Los contratos mínimos que Database exponga para integración podrán vivir en:

```text
VoltStack/Quantum/Database/Integration/Multitenancy/
```

si son parte del extension surface.

La implementación pertenecerá a:

```text
VoltStack/Quantum/Multitenancy/Database/
```

---

# 208. Proposed Database directory

```text
src/Quantum/Database/Integration/Multitenancy/
│
├── Contract/
│   ├── TenantContextProvider.php
│   ├── TenantConnectionIntegration.php
│   ├── TenantQueryIntegration.php
│   └── TenantMigrationIntegration.php
│
├── Context/
│   ├── DatabaseTenantContext.php
│   └── TenantPersistenceDomain.php
│
├── Capability/
│   └── DatabaseMultitenancyCapability.php
│
└── Exception/
    └── DatabaseMultitenancyIntegrationException.php
```

---

# 209. Proposed Multitenancy package directory

```text
src/Quantum/Multitenancy/
│
├── Tenant/
│   ├── TenantId.php
│   ├── TenantContext.php
│   ├── TenantStatus.php
│   └── TenantRegistry.php
│
├── Resolution/
│   ├── TenantResolver.php
│   ├── TenantResolutionContext.php
│   └── TenantResolutionPipeline.php
│
├── Placement/
│   ├── TenantPlacement.php
│   ├── TenantPlacementResolver.php
│   ├── TenantPlacementGeneration.php
│   └── TenantPlacementRegistry.php
│
├── Isolation/
│   ├── TenantIsolationStrategy.php
│   ├── TenantIsolationPolicy.php
│   └── TenantIsolationValidator.php
│
├── Database/
│   ├── Connection/
│   ├── Query/
│   ├── ORM/
│   ├── Transaction/
│   ├── Migration/
│   ├── Cache/
│   └── Runtime/
│
├── Security/
├── Telemetry/
├── Audit/
├── Testing/
└── Exception/
```

---

# 210. Integration flow

```text
                    APPLICATION
                         │
                         ▼
                 Tenant Resolution
                         │
                         ▼
                    TenantContext
                         │
                         ▼
                DatabaseContext Bridge
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
     Query Scope     ORM Identity    Connection
          │              │           Resolution
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                    Query Engine
                         │
                         ▼
                  Execution Engine
                         │
                         ▼
                Connection Manager
                         │
                         ▼
                  Tenant Placement
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
          Shared      Schema      Dedicated
          Database    Tenant      Database
```

---

# 211. Shared-schema flow

```text
Tenant A
   ↓
TenantContext(A)
   ↓
Query Model
   ↓
Tenant Scope(A)
   ↓
Query AST
   ↓
Compiler
   ↓
Database
```

El tenant scope será parte de la semántica antes del SQL.

---

# 212. Database-per-tenant flow

```text
Tenant A
   ↓
TenantContext
   ↓
PlacementResolver
   ↓
Database A
   ↓
ConnectionManager
   ↓
ConnectionLease
   ↓
Query
```

---

# 213. Schema-per-tenant flow

```text
Tenant A
   ↓
Placement
   ↓
Shared Database
   ↓
Schema A
   ↓
Connection State / Qualified SQL
   ↓
Query
```

---

# 214. Hybrid flow

```text
                        Tenant
                          │
                          ▼
                  Placement Resolver
                          │
          ┌───────────────┼────────────────┐
          │               │                │
          ▼               ▼                ▼
     Shared DB        Dedicated DB    Dedicated Cluster
          │               │                │
          ▼               ▼                ▼
     tenant_id         database         topology
```

---

# 215. Failure model

Errores conceptuales:

```text
DatabaseMultitenancyException
├── TenantContextMissingException
├── TenantContextMismatchException
├── TenantPlacementException
├── TenantPlacementStaleException
├── TenantConnectionResolutionException
├── TenantIsolationViolationException
├── TenantQueryScopeException
├── TenantEntityOwnershipException
├── CrossTenantRelationshipException
├── CrossTenantTransactionException
├── TenantSchemaException
├── TenantMigrationException
├── TenantResourceLimitException
└── TenantSecurityException
```

---

# 216. Missing TenantContext

Para una entidad declarada tenant-scoped:

```text
TenantContext = null
```

deberá fallar por default.

---

# 217. Fail closed

Regla:

```text
TenantScopedResource
+
MissingTenantContext
=
REJECT
```

No:

```text
query all tenants
```

---

# 218. Unknown tenant

Igualmente:

```text
Unknown Tenant
≠
Global Context
```

---

# 219. Global context

Si se necesita deberá ser explícito:

```text
GlobalDatabaseContext
```

o capability equivalente.

---

# 220. Administrative global access

Nunca deberá obtenerse accidentalmente eliminando TenantContext.

---

# 221. Critical security distinction

```text
No Tenant Context
```

no significa:

```text
All Tenants
```

---

# 222. Multitenancy invariants

## DB-MT-001

Database funcionará sin Multitenancy.

## DB-MT-002

Multitenancy será paquete oficial opcional.

## DB-MT-003

Database Core no dependerá del paquete Multitenancy.

## DB-MT-004

Multitenancy se integrará mediante contratos explícitos.

## DB-MT-005

Multitenancy no será equivalente a SaaS.

## DB-MT-006

Tenant no será equivalente a database.

## DB-MT-007

Tenant no será equivalente a connection.

## DB-MT-008

Tenant no será equivalente a authenticated user.

## DB-MT-009

TenantContext será operacional, no una Entity obligatoria.

## DB-MT-010

Tenant resolution ocurrirá fuera del SQL Compiler.

## DB-MT-011

Database no resolverá HTTP hostnames directamente.

## DB-MT-012

TenantContext podrá integrarse en DatabaseContext.

## DB-MT-013

TenantContext será opcional para aplicaciones no multitenant.

## DB-MT-014

Tenant-scoped resources requerirán TenantContext.

## DB-MT-015

Missing tenant no significará all tenants.

## DB-MT-016

Unknown tenant no significará global access.

## DB-MT-017

Global access será explícito.

## DB-MT-018

Cross-tenant access será explícito.

## DB-MT-019

Cross-tenant access será autorizable.

## DB-MT-020

Cross-tenant access será auditable.

## DB-MT-021

Isolation strategy será explícita.

## DB-MT-022

Placement será independiente de isolation strategy.

## DB-MT-023

Tenant placement será resolvible dinámicamente.

## DB-MT-024

Tenant placement será generation-aware.

## DB-MT-025

Placement cache no será placement truth.

## DB-MT-026

TTL no será garantía de placement correctness.

## DB-MT-027

TenantConnectionResolver no abrirá conexiones directamente.

## DB-MT-028

ConnectionManager conservará ownership de acquisition.

## DB-MT-029

Tenant será resuelto antes de connection acquisition.

## DB-MT-030

Placement será resuelto antes de endpoint selection.

## DB-MT-031

Read/write routing ocurrirá después de tenant placement.

## DB-MT-032

Replica selection respetará tenant placement.

## DB-MT-033

Tenant query scope será semántico.

## DB-MT-034

Tenant scope no será concatenación SQL.

## DB-MT-035

Tenant scope ocurrirá antes de compilation.

## DB-MT-036

Shared-schema queries serán scoped.

## DB-MT-037

Tenant-scoped entities tendrán metadata explícita.

## DB-MT-038

Global entities podrán coexistir con tenant entities.

## DB-MT-039

IdentityMap incluirá tenant domain cuando sea necesario.

## DB-MT-040

Mismo entity ID en tenants distintos no compartirá managed identity.

## DB-MT-041

Entity Cache incluirá tenant cuando corresponda.

## DB-MT-042

Query Cache incluirá tenant cuando corresponda.

## DB-MT-043

Result Cache incluirá tenant cuando corresponda.

## DB-MT-044

Metadata Cache podrá compartirse cuando sea tenant-independent.

## DB-MT-045

Tenant switching con active mutable ORM state será rechazado por default.

## DB-MT-046

Tenant switching seguro utilizará scopes separados.

## DB-MT-047

forTenant no mutará globals.

## DB-MT-048

TenantContext no equivaldrá a authorization.

## DB-MT-049

Resolvable tenant no equivaldrá a authorized tenant.

## DB-MT-050

Transaction tendrá tenant affinity.

## DB-MT-051

Transaction no cambiará de tenant.

## DB-MT-052

Cross-tenant transactions no fingirán distributed ACID.

## DB-MT-053

Sticky state será tenant-aware.

## DB-MT-054

Replica health podrá compartirse por endpoint.

## DB-MT-055

Existing transaction no migrará con placement generation.

## DB-MT-056

New executions podrán adoptar new placement generation.

## DB-MT-057

Database-per-tenant será una estrategia soportada.

## DB-MT-058

Schema-per-tenant será una estrategia soportada.

## DB-MT-059

Shared-schema será una estrategia soportada.

## DB-MT-060

Shard-per-tenant será representable.

## DB-MT-061

Dedicated cluster será representable.

## DB-MT-062

Hybrid placement será representable.

## DB-MT-063

Connection tenant state será limpiado antes de reuse.

## DB-MT-064

Unknown connection tenant state causará discard.

## DB-MT-065

No existirá pool ilimitado por tenant.

## DB-MT-066

Pool partitioning dependerá de placement/credentials/capabilities.

## DB-MT-067

Credentials no se derivarán inseguramente del tenant ID.

## DB-MT-068

Credential rotation será generation-aware.

## DB-MT-069

Tenant relationships serán tenant-local por default.

## DB-MT-070

Cross-tenant relationships serán rechazadas por default.

## DB-MT-071

Tenant-to-global relationships podrán declararse.

## DB-MT-072

Lazy relationship loading conservará tenant identity.

## DB-MT-073

Eager loading conservará tenant identity.

## DB-MT-074

Batch loading conservará tenant identity.

## DB-MT-075

Hydration conservará persistence tenant domain.

## DB-MT-076

Persistence validará entity tenant ownership.

## DB-MT-077

New tenant entities recibirán tenant ownership mediante mecanismo controlado.

## DB-MT-078

Tenant key no será mass-assignable inseguramente.

## DB-MT-079

Tenant key será immutable por default tras persistence.

## DB-MT-080

Tenant transfer será operación explícita.

## DB-MT-081

Raw SQL no se asumirá tenant-safe.

## DB-MT-082

Raw SQL tenant-scoped tendrá política explícita.

## DB-MT-083

UNKNOWN query tenant safety no equivaldrá a SAFE.

## DB-MT-084

Global migrations se distinguirán de tenant migrations.

## DB-MT-085

Tenant migrations soportarán fleet execution.

## DB-MT-086

Fleet migration tendrá bounded concurrency.

## DB-MT-087

Fleet migration tendrá progress tracking.

## DB-MT-088

Fleet migration tendrá failure isolation.

## DB-MT-089

Fleet migration podrá tener partial state.

## DB-MT-090

Migration partial no será reported como global success.

## DB-MT-091

Schema version skew será representable.

## DB-MT-092

Zero-downtime migration considerará tenant fleet rollout.

## DB-MT-093

Async events reconstruirán tenant context explícitamente.

## DB-MT-094

Jobs no serializarán live Database resources.

## DB-MT-095

Tenant-aware jobs reconstruirán context.

## DB-MT-096

Deleted tenant job fallará controladamente.

## DB-MT-097

Tenant status podrá influir en admission.

## DB-MT-098

Administrative access podrá diferir de normal tenant access.

## DB-MT-099

Cache invalidation será tenant-targeted cuando sea posible.

## DB-MT-100

FLUSHALL no será estrategia normal de tenant invalidation.

## DB-MT-101

Tenant identity no será metric label automáticamente.

## DB-MT-102

Telemetry respetará cardinality policy.

## DB-MT-103

Audit respetará sensitive-data policy.

## DB-MT-104

Cross-tenant operations tendrán audit reforzado.

## DB-MT-105

TenantContext será execution-scoped en persistent runtimes.

## DB-MT-106

TenantContext no será worker-scoped.

## DB-MT-107

Sequential tenant contamination será imposible por diseño.

## DB-MT-108

Concurrent tenant contamination será imposible por diseño.

## DB-MT-109

Execution reset eliminará tenant-scoped runtime state.

## DB-MT-110

Connection reset eliminará tenant-specific session state.

## DB-MT-111

Tenant infrastructure caches podrán ser worker-shared sólo si son seguras.

## DB-MT-112

Tenant resource budgets serán soportables.

## DB-MT-113

Resource governance no será billing.

## DB-MT-114

SaaS plans no serán conceptos de Database Core.

## DB-MT-115

Noisy-neighbor protection será posible.

## DB-MT-116

Global resource budget tendrá prioridad sobre unlimited tenant concurrency.

## DB-MT-117

Tenant backup será placement-aware.

## DB-MT-118

Tenant restore será isolation-aware.

## DB-MT-119

Tenant deletion no implicará drop database automático.

## DB-MT-120

Destructive tenant operations serán explícitas.

## DB-MT-121

Destructive operations serán autorizadas.

## DB-MT-122

Destructive operations serán auditables.

## DB-MT-123

Shared-schema isolation tendrá tests.

## DB-MT-124

Schema isolation tendrá tests.

## DB-MT-125

Database isolation tendrá tests.

## DB-MT-126

ORM identity isolation tendrá tests.

## DB-MT-127

Relationship isolation tendrá tests.

## DB-MT-128

Transaction tenant affinity tendrá tests.

## DB-MT-129

Cache isolation tendrá tests.

## DB-MT-130

Raw SQL policies tendrán tests.

## DB-MT-131

Persistent worker isolation tendrá tests.

## DB-MT-132

Concurrent coroutine tenant isolation tendrá tests.

## DB-MT-133

Fleet migrations tendrán tests.

## DB-MT-134

Placement movement tendrá tests.

## DB-MT-135

Failover tendrá tests.

## DB-MT-136

Authorization bypass attempts tendrán tests.

## DB-MT-137

Tenant query scope no dependerá del runtime.

## DB-MT-138

Tenant connection resolution no dependerá del ORM API utilizada.

## DB-MT-139

Model API y Repository API compartirán la misma tenant semantics.

## DB-MT-140

Active Record convenience API no tendrá tenant engine separado.

## DB-MT-141

EntityManager seguirá siendo único ORM engine.

## DB-MT-142

Compiler seguirá siendo responsable sólo de SQL representation.

## DB-MT-143

Driver seguirá siendo tenant-agnostic cuando sea posible.

## DB-MT-144

ConnectionManager seguirá siendo owner de connection lifecycle.

## DB-MT-145

Multitenancy extenderá Database, no lo reemplazará.

## DB-MT-146

Database capabilities seguirán siendo válidas sin Multitenancy.

## DB-MT-147

Multitenancy podrá desinstalarse de aplicaciones que no lo necesiten.

## DB-MT-148

SaaS podrá integrarse posteriormente sin fusionarse con Multitenancy.

## DB-MT-149

Security tendrá prioridad sobre convenience.

## DB-MT-150

Isolation tendrá prioridad sobre connection reuse.

## DB-MT-151

Correctness tendrá prioridad sobre transparent tenant switching.

## DB-MT-152

UNKNOWN tenant isolation nunca será tratado como SAFE.

## DB-MT-153

Tenant identity deberá permanecer explícita en toda frontera donde afecte semántica.

## DB-MT-154

No Tenant Context nunca significará All Tenants.

## DB-MT-155

Cross-tenant access nunca será consecuencia accidental de ausencia de contexto.

## DB-MT-156

Tenant placement podrá cambiar sin cambiar TenantId.

## DB-MT-157

TenantId será identidad lógica estable.

## DB-MT-158

Physical placement será reemplazable.

## DB-MT-159

Multitenancy deberá soportar crecimiento desde shared DB hasta dedicated infrastructure.

## DB-MT-160

La arquitectura deberá permitir que una aplicación cambie estrategia de aislamiento sin rediseñar el Database Core.

---

# 223. Escalabilidad evolutiva

Una propiedad importante será permitir comenzar con:

```text
Phase 1
Shared Database
+
tenant_id
```

evolucionar hacia:

```text
Phase 2
Shared Database
+
Separate Schemas
```

y posteriormente:

```text
Phase 3
Database per Tenant
```

o:

```text
Phase 4
Hybrid Placement
```

sin cambiar la API principal de dominio.

---

# 224. Ejemplo de evolución

Aplicación:

```php
$orders = Order::query()
    ->where('status', 'pending')
    ->get();
```

podrá conservarse.

### Fase inicial

```text
Tenant A
    ↓
Shared Database
    ↓
tenant_id = A
```

### Fase posterior

```text
Tenant A
    ↓
Database A
```

### Fase enterprise

```text
Tenant A
    ↓
Dedicated Cluster
```

La aplicación no debería reescribir todas sus consultas.

---

# 225. Control plane y data plane

La arquitectura puede distinguir:

```text
CONTROL PLANE
```

de:

```text
DATA PLANE
```

---

# 226. Control Plane

Responsable potencial de:

```text
tenant registry
placement
tenant status
migration state
credentials references
resource policies
```

---

# 227. Data Plane

Responsable de:

```text
tenant queries
transactions
persistence
connection routing
```

---

# 228. Arquitectura

```text
                    CONTROL PLANE
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
          Registry   Placement   Policies
              │          │          │
              └──────────┼──────────┘
                         ▼
                   TenantContext
                         │
═════════════════════════╪═════════════════════════
                         │
                     DATA PLANE
                         │
                         ▼
                  DatabaseContext
                         │
          ┌──────────────┼───────────────┐
          ▼              ▼               ▼
       Query          ORM             Routing
          │              │               │
          └──────────────┼───────────────┘
                         ▼
                  Database Engine
```

---

# 229. Control-plane failure

Si el control plane no está disponible pero existe placement cache:

```text
Cached Placement
```

sólo podrá usarse conforme a política de freshness/consistency.

---

# 230. Unknown placement

Si no puede determinarse de forma segura dónde reside el tenant:

```text
FAIL CLOSED
```

---

# 231. No fallback tenant

Nunca:

```text
Tenant placement unknown
        ↓
use default database
```

si eso puede mezclar tenants.

---

# 232. Tenant mobility

La arquitectura deberá soportar conceptualmente:

```text
Tenant Mobility
```

es decir:

```text
Shared DB
   ↓
Dedicated DB
```

o:

```text
Region A
   ↓
Region B
```

---

# 233. Migration phases

Conceptualmente:

```text
PREPARE
   ↓
COPY
   ↓
CATCH_UP
   ↓
QUIESCE / DUAL-WRITE POLICY
   ↓
CUTOVER
   ↓
VERIFY
   ↓
RETIRE OLD PLACEMENT
```

La implementación específica corresponderá a sistemas posteriores.

---

# 234. No hidden dual writes

Database no habilitará dual-write automáticamente.

---

# 235. Why

Dual writes introducen:

```text
partial failure
ordering
idempotency
consistency
reconciliation
```

y requieren protocolo explícito.

---

# 236. Placement cutover

El cambio deberá incrementar:

```text
TenantPlacementGeneration
```

---

# 237. Stale execution

Una ejecución con generation antigua deberá:

```text
finish according to policy
```

o:

```text
be cancelled/rejected
```

pero nunca cambiar silenciosamente de database a mitad de transaction.

---

# 238. Principio de seguridad

La arquitectura completa se resume en:

```text
RESOLVE TENANT
      ↓
AUTHORIZE TENANT
      ↓
RESOLVE PLACEMENT
      ↓
BUILD DATABASE CONTEXT
      ↓
APPLY TENANT QUERY SEMANTICS
      ↓
RESOLVE ROUTING
      ↓
ACQUIRE SAFE CONNECTION
      ↓
EXECUTE
      ↓
PRESERVE TENANT IDENTITY
      ↓
RESET ALL TENANT STATE
```

---

# 239. Regla de fail-closed

En cualquier punto donde la seguridad dependa del tenant:

```text
UNKNOWN
```

deberá favorecer:

```text
REJECT
```

sobre:

```text
FALLBACK TO GLOBAL
```

---

# 240. Resultado arquitectónico

VoltStack podrá soportar desde:

```text
Small SaaS
10 tenants
shared database
```

hasta:

```text
Large Platform
100,000+ tenants
hybrid placement
multiple shards
multiple regions
dedicated enterprise databases
```

sin introducir un segundo ORM ni un segundo Database Engine.

---

# 241. Principio final

La integración Multitenancy seguirá:

```text
TENANT IS LOGICAL IDENTITY
```

```text
PLACEMENT IS PHYSICAL/LOGICAL LOCATION
```

```text
ISOLATION STRATEGY IS POLICY
```

```text
TENANT CONTEXT IS EXECUTION-SCOPED
```

```text
AUTHORIZATION IS EXPLICIT
```

```text
QUERY SCOPING IS SEMANTIC
```

```text
CONNECTION ROUTING IS CONTEXT-AWARE
```

```text
ORM IDENTITY IS TENANT-AWARE
```

```text
TRANSACTIONS ARE TENANT-AFFINE
```

```text
CACHES ARE TENANT-SAFE
```

```text
PERSISTENT WORKERS NEVER RETAIN CURRENT TENANT
```

```text
UNKNOWN ISOLATION FAILS CLOSED
```

La regla definitiva será:

> **Multitenancy en VoltStack no consistirá en añadir automáticamente un `tenant_id` a cada consulta. Será una arquitectura completa de identidad, contexto, aislamiento, placement, routing, ORM, conexiones, transacciones, cache, migraciones, seguridad y runtime capaz de mantener una frontera verificable entre tenants independientemente de si comparten una tabla, un schema, una base de datos, un shard o una infraestructura completa.**

---

# 242. Estado del Bloque 26

```text
BLOCK 26 — MULTITENANCY INTEGRATION

✓ 261_DATABASE_MULTITENANCY_INTEGRATION_ARCHITECTURE.md
│
├── 262_DATABASE_TENANT_CONNECTION_RESOLUTION_SYSTEM.md
├── 263_DATABASE_TENANT_DATABASE_ISOLATION_SYSTEM.md
├── 264_DATABASE_TENANT_SCHEMA_ISOLATION_SYSTEM.md
├── 265_DATABASE_TENANT_QUERY_CONTEXT_SYSTEM.md
└── 266_DATABASE_TENANT_MIGRATION_INTEGRATION_SYSTEM.md
```

---

# 243. Siguiente documento

```text
262_DATABASE_TENANT_CONNECTION_RESOLUTION_SYSTEM.md
```

El siguiente documento deberá profundizar específicamente en:

```text
TenantConnectionResolver
TenantPlacement
Logical Database Resolution
ConnectionIntent
Pool Resolution
Credential Resolution
Writer Resolution
Replica Resolution
Shard Resolution
Tenant Affinity
Connection Lease Ownership
Tenant Session State
Connection Reset
Placement Generations
Credential Generations
Failover
Tenant Mobility
Persistent Workers
Coroutine Isolation
Connection Pool Explosion Prevention
Diagnostics
Telemetry
Failure Handling
```

manteniendo la regla:

> **Resolver la conexión de un tenant significa determinar de forma segura el destino lógico y físico de una operación Database; no significa abrir directamente una conexión ni convertir TenantId en un nombre de base de datos mediante convenciones implícitas.**