# 265_DATABASE_TENANT_QUERY_CONTEXT_SYSTEM.md

# VoltStack Quantum Database
## Tenant Query Context System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Integración:** `VoltStack/Quantum/Multitenancy`  
**Documento:** 265 — Tenant Query Context System  
**Bloque:** 26 — Multitenancy Integration  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `264_DATABASE_TENANT_SCHEMA_ISOLATION_SYSTEM.md`  
**Siguiente documento:** `266_DATABASE_TENANT_MIGRATION_INTEGRATION_SYSTEM.md`

---

# 1. Propósito

Este documento define el sistema mediante el cual VoltStack transportará, validará y preservará el contexto tenant de una operación de consulta desde el momento en que nace hasta su ejecución final.

El sistema deberá garantizar que una query iniciada bajo:

```text
Tenant A
```

no pueda convertirse accidentalmente durante:

```text
Query Builder
      ↓
AST
      ↓
Semantic Analysis
      ↓
Optimization
      ↓
Planning
      ↓
Compilation
      ↓
Execution
```

en una operación perteneciente a:

```text
Tenant B
```

o en una query sin aislamiento tenant.

La regla central será:

> **Una query tenant-aware de VoltStack transportará explícitamente un TenantQueryContext inmutable desde su origen hasta su ejecución. Ninguna etapa del Query Engine deberá descubrir el tenant consultando estado global mutable.**

Por tanto:

```text
Query
+
TenantQueryContext
=
Tenant-Aware Query Operation
```

y nunca:

```text
Query
+
globalCurrentTenant()
```

---

# 2. Problema arquitectónico

Una arquitectura multitenant puede parecer segura si se limita a:

```sql
WHERE tenant_id = ?
```

pero la query puede atravesar múltiples subsistemas:

```text
Model API
Repository
EntityManager
Query Builder
AST
Semantic Engine
Optimizer
Planner
Compiler
Executor
Connection Manager
Replica Router
Shard Router
Result Cache
Hydrator
Relationship Loader
```

Si cualquiera de ellos obtiene el tenant desde un estado distinto, pueden aparecer inconsistencias.

Ejemplo:

```text
Request starts
Tenant = A

Query created
Tenant = A

Async operation
Tenant global changed = B

Query executes
Tenant = B
```

VoltStack deberá hacer este escenario estructuralmente imposible.

---

# 3. TenantQueryContext ≠ TenantContext

`TenantContext` representa el tenant activo dentro de un scope de aplicación.

`TenantQueryContext` representa las propiedades tenant que gobiernan una operación Database concreta.

Por tanto:

```text
TenantContext
≠
TenantQueryContext
```

El segundo podrá derivarse del primero.

---

# 4. TenantQueryContext ≠ QueryContext

VoltStack ya dispone conceptualmente de:

```text
DATABASE_QUERY_CONTEXT_SYSTEM
```

El contexto tenant será una especialización/integración dentro del contexto general de query.

Conceptualmente:

```text
QueryContext
│
├── Execution Context
├── Consistency Context
├── Security Context
├── Resource Context
├── Telemetry Context
└── Tenant Query Context
```

---

# 5. Modelo central

Se propone:

```php
final readonly class TenantQueryContext
{
    public function __construct(
        public TenantId $tenant,
        public TenantIsolationDomain $isolationDomain,
        public TenantPersistenceDomain $persistenceDomain,
        public TenantPlacementGeneration $placementGeneration,
        public TenantQueryScope $scope,
        public TenantQueryPolicy $policy,
        public ?TenantSchemaResolution $schema,
        public ?ShardRoutingContext $shard,
        public ?CrossTenantCapability $crossTenantCapability,
    ) {}
}
```

El objeto será:

```text
immutable
typed
explicit
scoped
serializable only through safe representation
```

---

# 6. Contexto inmutable

Una vez creada una operación:

```text
Query Q
TenantQueryContext A
```

no deberá permitirse:

```text
Q.setTenant(B)
```

sobre la misma operación mutable.

---

# 7. Cambio explícito

Si una operación administrativa necesita otro tenant:

```text
Query A
   ↓
derive/rebuild
   ↓
Query B
```

con un nuevo contexto validado.

---

# 8. TenantQueryScope

Clasificará el alcance de una query.

```php
enum TenantQueryScope
{
    case TENANT;
    case GLOBAL;
    case SYSTEM;
    case CROSS_TENANT;
}
```

---

# 9. TENANT

La operación pertenece exactamente a un tenant.

Ejemplo:

```php
Order::query()
    ->where('status', 'pending')
    ->get();
```

dentro de:

```text
Tenant A
```

---

# 10. GLOBAL

Accede a datos explícitamente globales.

Ejemplo:

```text
currencies
countries
system catalogs
```

No deberá interpretarse como:

```text
all tenants
```

---

# 11. SYSTEM

Reservado para infraestructura Database/framework.

---

# 12. CROSS_TENANT

Sólo para operaciones explícitamente autorizadas.

Ejemplo:

```text
administration
migration
audit
fleet operations
tenant analytics
```

---

# 13. No tenant context ≠ global

Regla crítica:

```text
TenantQueryContext = null
```

no significará:

```text
GLOBAL
```

ni:

```text
CROSS_TENANT
```

---

# 14. Fail closed

Una entidad marcada como tenant-scoped consultada sin contexto deberá producir:

```text
MissingTenantQueryContextException
```

por default.

---

# 15. Context creation

Pipeline:

```text
Application Scope
      ↓
TenantContext
      ↓
Authorization
      ↓
Tenant Placement
      ↓
Tenant Isolation Domain
      ↓
Schema / Shard Resolution
      ↓
TenantQueryContextFactory
      ↓
TenantQueryContext
```

---

# 16. Factory

```php
interface TenantQueryContextFactory
{
    public function create(
        TenantContext $tenant,
        DatabaseOperationIntent $intent
    ): TenantQueryContext;
}
```

---

# 17. Context creation ≠ query execution

La factory:

```text
resolves
validates
constructs context
```

pero no ejecuta queries de aplicación.

---

# 18. Query origin

El contexto puede originarse desde:

```text
Model API
Repository
EntityManager
Query Builder
Relationship Loader
Job
CLI
HTTP request
Console
Migration administration
```

---

# 19. Model API

Ejemplo:

```php
$tenantScope->run($tenant, function () {
    $orders = Order::query()
        ->where('status', 'pending')
        ->get();
});
```

Conceptualmente:

```text
Order::query()
      ↓
ModelContextResolver
      ↓
Scoped EntityManager
      ↓
TenantQueryContext
```

---

# 20. No static tenant state

`Order::query()` podrá seguir siendo una API estática conveniente.

Pero:

```text
static API
≠
static tenant state
```

---

# 21. Repository API

```php
$orders = $ordersRepository
    ->query()
    ->whereStatus(OrderStatus::PENDING)
    ->get();
```

El Repository obtendrá el contexto del:

```text
EntityManager / Repository Context
```

scoped.

---

# 22. Repository ≠ tenant resolver

El Repository no deberá decidir qué tenant está activo.

---

# 23. EntityManager integration

Un EntityManager tenant-scoped tendrá:

```text
TenantQueryContextProvider
```

compatible con su:

```text
TenantPersistenceDomain
```

---

# 24. EntityManager invariant

```text
EntityManager Tenant Domain
=
TenantQueryContext Domain
```

para operaciones tenant-scoped.

---

# 25. Query Builder integration

Al crear un builder:

```php
$query = Order::query();
```

éste deberá recibir o asociarse con un contexto inmutable.

Conceptualmente:

```php
final readonly class QueryDefinition
{
    public function __construct(
        public QueryAst $ast,
        public QueryContext $context,
    ) {}
}
```

---

# 26. Query Builder ≠ context owner

El builder construye la query.

El scope/database context gobierna el tenant.

---

# 27. Immutable derivation

Ejemplo:

```php
$q1 = Order::query();

$q2 = $q1->where('status', 'pending');
```

Conceptualmente:

```text
q1.context === q2.context
```

o una copia semánticamente equivalente.

---

# 28. Context replacement

No deberá existir un método público trivial como:

```php
$query->tenant('another-tenant');
```

para queries tenant-scoped ordinarias.

---

# 29. Administrative context switching

Podrá existir una API explícita:

```php
$adminScope->forTenant(
    $tenant,
    fn (TenantDatabaseContext $db) => ...
);
```

---

# 30. Query AST

El TenantQueryContext no deberá representarse únicamente como un predicate AST.

Esto es importante porque diferentes estrategias pueden ser:

```text
row-per-tenant
schema-per-tenant
database-per-tenant
shard-per-tenant
```

---

# 31. Row isolation

Puede producir:

```text
Predicate AST
tenant_id = :tenant
```

---

# 32. Schema isolation

Puede producir:

```text
Namespace Binding
tenant_schema.orders
```

sin predicate `tenant_id`.

---

# 33. Database isolation

Puede producir:

```text
Connection Resolution
Database A
```

sin cambiar Query AST lógico.

---

# 34. Shard isolation

Puede producir:

```text
Shard Routing Constraint
```

---

# 35. Hybrid isolation

Puede combinar:

```text
Database
+
Schema
+
Row Predicate
+
Shard
```

según tenant.

---

# 36. Context drives policy

Por tanto:

```text
TenantQueryContext
      ↓
Isolation Strategy
      ↓
Required Query Transformations
```

---

# 37. Tenant query transformation

Se propone una fase:

```text
TenantQueryIsolationPass
```

antes de optimization.

---

# 38. Pipeline

```text
Developer Query
      ↓
Query Builder
      ↓
Logical Query AST
      ↓
Tenant Query Isolation Pass
      ↓
Tenant-Aware AST / Namespace Binding
      ↓
Semantic Analysis
      ↓
Optimization
      ↓
Planning
      ↓
Compilation
      ↓
Execution
```

---

# 39. Why before optimization

El optimizer debe conocer restricciones tenant.

De lo contrario podría optimizar una query incompleta.

---

# 40. Tenant predicate injection

Para shared-schema:

```text
Original:
status = 'pending'
```

se transforma semánticamente en:

```text
status = 'pending'
AND
tenant_id = :tenant
```

---

# 41. Parameter binding

El TenantId deberá agregarse como parámetro typed.

No como literal SQL.

---

# 42. Trusted parameter

Su valor vendrá del:

```text
TenantQueryContext
```

no del request payload.

---

# 43. User tenant predicate

Si el usuario escribe:

```php
->where('tenant_id', $request->tenant)
```

eso no sustituye el scope interno.

---

# 44. Conflicting tenant predicate

Ejemplo:

```text
Context Tenant = A
Query says tenant_id = B
```

deberá producir:

```text
TenantQueryConflictException
```

o simplificarse a un resultado imposible bajo política explícita.

Preferencia:

```text
REJECT
```

para detectar bugs/security issues.

---

# 45. Same tenant predicate

Si:

```text
Context = A
Explicit predicate = A
```

puede normalizarse/deduplicarse.

---

# 46. Predicate deduplication

El optimizer podrá eliminar redundancia sólo si demuestra equivalencia.

---

# 47. OR predicates

Caso peligroso:

```text
tenant_id = A
OR
status = 'public'
```

El tenant guard no debe añadirse incorrectamente como una rama OR.

Debe envolver:

```text
tenant_id = A
AND
(
    tenant_id = A
    OR status = 'public'
)
```

o normalización equivalente.

---

# 48. Root tenant constraint

La frontera tenant debe aplicarse al conjunto lógico completo de la query.

---

# 49. Subqueries

Cada subquery deberá clasificarse.

Puede ser:

```text
TENANT
GLOBAL
CORRELATED_TENANT
PRIVILEGED_CROSS_TENANT
```

---

# 50. Tenant subquery

Ejemplo:

```sql
SELECT *
FROM orders
WHERE customer_id IN (
    SELECT id
    FROM customers
)
```

ambas tablas tenant-scoped deberán preservar Tenant A.

---

# 51. Global subquery

Puede existir:

```text
Tenant Orders
      ↓
Global Currencies
```

sin insertar tenant predicate en tabla global.

---

# 52. CTEs

Cada CTE conservará contexto semántico.

---

# 53. Recursive CTEs

No deberán escapar del domain tenant durante recursión.

---

# 54. UNION

```text
Tenant Query A
UNION
Tenant Query A
```

válido.

---

# 55. Incompatible union

```text
Tenant A Query
UNION
Tenant B Query
```

rechazado salvo operación cross-tenant autorizada.

---

# 56. JOINs

Cada relación participante deberá clasificarse por namespace/domain.

---

# 57. Tenant-local JOIN

```text
Tenant A Orders
JOIN
Tenant A Customers
```

válido.

---

# 58. Tenant-global JOIN

```text
Tenant A Orders
JOIN
Global Currencies
```

puede ser válido.

---

# 59. Cross-tenant JOIN

```text
Tenant A Orders
JOIN
Tenant B Customers
```

rechazado por default.

---

# 60. Join predicate

En shared schema puede ser necesario:

```text
orders.tenant_id = A
AND
customers.tenant_id = A
```

además de:

```text
orders.customer_id = customers.id
```

---

# 61. Composite tenant relationship

Si el modelo utiliza:

```text
(tenant_id, customer_id)
```

el Semantic Engine deberá reconocerlo.

---

# 62. UPDATE

Tenant isolation también aplica a:

```text
UPDATE
```

No sólo SELECT.

---

# 63. Shared-schema update

```php
Order::query()
    ->where('status', 'pending')
    ->update(['status' => 'processing']);
```

deberá significar:

```text
UPDATE logical Order
WHERE
    tenant = A
    AND status = pending
```

---

# 64. DELETE

Igual:

```php
Order::query()
    ->where('status', 'cancelled')
    ->delete();
```

debe conservar scope A.

---

# 65. INSERT

Insert también requiere contexto.

---

# 66. Tenant field ownership

En shared schema:

```text
tenant_id
```

deberá derivarse del contexto.

---

# 67. Explicit conflicting insert

Si payload contiene:

```text
tenant_id = B
```

bajo contexto A:

```text
REJECT
```

---

# 68. Bulk insert

Todas las filas deberán ser compatibles con el contexto.

---

# 69. Bulk update/delete

Nunca deberán perder tenant scope durante optimizaciones masivas.

---

# 70. Upsert

También requiere tenant semantics.

Un conflict target puede necesitar incluir:

```text
tenant_id
```

dependiendo del esquema.

---

# 71. Unique constraints

Una unicidad:

```text
email UNIQUE
```

puede significar global uniqueness.

Mientras:

```text
UNIQUE(tenant_id, email)
```

representa tenant-local uniqueness.

El Query Context no deberá asumir una u otra sin metadata.

---

# 72. Query metadata

La metadata deberá clasificar entidades/tablas como:

```text
TENANT_SCOPED
GLOBAL
SYSTEM
EXPLICIT_CROSS_TENANT
```

---

# 73. TenantScopeMetadata

Ejemplo:

```php
final readonly class TenantScopeMetadata
{
    public function __construct(
        public TenantScopeKind $kind,
        public ?ColumnName $tenantKey,
        public TenantIsolationStrategy $strategy,
    ) {}
}
```

---

# 74. Mapping example

```php
#[TenantScoped(
    strategy: TenantIsolationStrategy::SHARED_DATABASE_SHARED_SCHEMA,
    key: 'tenant_id',
)]
class Order
{
}
```

---

# 75. Strategy override

La estrategia física real puede provenir del tenant placement, no quedar hardcoded en la entidad.

Por tanto el mapping puede expresar:

```text
Entity is tenant-owned
```

sin decidir necesariamente:

```text
row/schema/database
```

---

# 76. Logical ownership metadata

Preferencia:

```php
#[TenantOwned]
class Order {}
```

y el placement decide el aislamiento físico.

---

# 77. Portable entity model

Esto permitirá que el mismo modelo pase de:

```text
Shared DB
```

a:

```text
Dedicated DB
```

sin reescribir la entidad.

---

# 78. Semantic graph

El sistema:

```text
DATABASE_QUERY_SEMANTIC_GRAPH_SYSTEM
```

podrá incluir nodos/edges de:

```text
tenant ownership
namespace scope
persistence domain
```

---

# 79. Semantic validation

Antes de planificar:

```text
QueryTenantDomain
```

deberá ser coherente.

---

# 80. Domain states

Conceptualmente:

```text
SINGLE_TENANT
GLOBAL
TENANT_PLUS_GLOBAL
CROSS_TENANT_AUTHORIZED
INVALID
UNKNOWN
```

---

# 81. UNKNOWN ≠ tenant safe

Regla:

```text
UNKNOWN
→
REJECT
```

para operaciones tenant-owned ordinarias.

---

# 82. Optimizer

El optimizer puede:

```text
deduplicate tenant predicates
push tenant predicates down
reorder joins
```

si preserva semántica.

---

# 83. Optimizer invariant

Nunca podrá:

```text
remove required tenant constraint
```

---

# 84. Predicate pushdown

Puede mejorar rendimiento:

```text
Tenant Filter
      ↓
scan
```

pero sólo cuando semánticamente seguro.

---

# 85. Join optimization

El join optimizer deberá conservar domain constraints aunque cambie orden físico.

---

# 86. Query rewrite

Toda rewrite deberá satisfacer:

```text
TenantSemantics(Q)
=
TenantSemantics(Rewrite(Q))
```

---

# 87. Query planner

El planner recibirá:

```text
TenantQueryContext
```

o una representación derivada e inmutable.

---

# 88. Planning decisions

Podrá decidir:

```text
database target
schema binding
shard
writer/replica
consistency route
```

sin autorizar nuevos tenants.

---

# 89. Planner ≠ authorization engine

No deberá convertir:

```text
Tenant A
```

en:

```text
Cross Tenant
```

porque resulte más eficiente.

---

# 90. Physical query plan

Deberá quedar ligado a:

```text
PersistenceDomain
PlacementGeneration
SchemaGeneration if applicable
ShardMapGeneration if applicable
```

cuando sean relevantes.

---

# 91. Stale physical plan

Si cambia placement antes de ejecución:

```text
Plan Generation != Current Generation
```

podrá requerir:

```text
REPLAN
```

o rechazo.

---

# 92. Logical plan reuse

Planes tenant-neutral pueden compartirse cuando no incorporan bindings físicos.

---

# 93. Physical plan reuse

Sólo cuando sus domain bindings sean compatibles.

---

# 94. Compiler

El SQL Compiler:

```text
does not choose tenant
does not authorize tenant
does not query global tenant state
```

---

# 95. Compiler input

Recibirá un plan ya resuelto.

---

# 96. Row isolation compilation

Puede producir:

```sql
SELECT ...
FROM orders
WHERE tenant_id = ?
  AND status = ?
```

---

# 97. Schema isolation compilation

Puede producir:

```sql
SELECT ...
FROM "tenant_x"."orders"
WHERE status = ?
```

---

# 98. Database isolation compilation

El SQL puede permanecer:

```sql
SELECT ...
FROM orders
WHERE status = ?
```

porque la conexión ya está ligada a la base correcta.

---

# 99. Compiler invariance

La ausencia de `tenant_id` en SQL no significa falta de aislamiento si la estrategia es schema/database.

---

# 100. Executor validation

Antes de ejecución deberá comprobarse compatibilidad entre:

```text
Execution Plan
Connection Lease
Tenant Persistence Domain
```

---

# 101. Execution guard

Se propone:

```php
interface TenantQueryExecutionGuard
{
    public function assertExecutable(
        ExecutionPlan $plan,
        ConnectionLease $connection,
        TenantQueryContext $tenant
    ): void;
}
```

---

# 102. Connection mismatch

```text
Query Tenant A
Connection Tenant B
```

debe fallar antes de enviar SQL.

---

# 103. Placement generation mismatch

Igualmente:

```text
Plan generation 10
Current generation 11
```

deberá detectarse cuando afecte seguridad/correctness.

---

# 104. Transaction integration

Si existe una transaction:

```text
TenantQueryContext Domain
=
Transaction Tenant Domain
```

---

# 105. Cross-tenant transaction mismatch

```text
Transaction A
Query B
```

se rechaza.

---

# 106. No implicit transaction switching

No:

```text
suspend Tx A
run B
resume A
```

como efecto oculto de una query.

---

# 107. Read/write routing

El contexto podrá contener:

```text
ReadIntent
WriteIntent
LockIntent
ConsistencyRequirement
```

pero el tenant domain permanece independiente.

---

# 108. Replica routing

Sólo dentro del replication group del tenant.

---

# 109. Sticky reads

Sticky state deberá estar keyeado por domain.

---

# 110. Locking query

Una query:

```php
->lockForUpdate()
```

mantiene el mismo tenant context y fuerza routing compatible con writer/transaction.

---

# 111. Sharding

El TenantQueryContext puede incluir:

```text
ShardRoutingContext
```

cuando el tenant placement lo requiera.

---

# 112. Tenant ≠ shard

Un tenant puede ocupar:

```text
one shard
multiple shards
```

---

# 113. Shard inference

Puede derivarse de:

```text
TenantPlacement
Query Predicates
Entity Identifier
Partition Key
```

---

# 114. Unknown shard

Si una write requiere un shard único:

```text
Shard = UNKNOWN
```

deberá fallar.

---

# 115. Multi-shard read

Podrá planificarse explícitamente si la política lo permite.

---

# 116. Cross-tenant vs cross-shard

Son dimensiones distintas.

```text
CrossShard
≠
CrossTenant
```

---

# 117. Relationship loading

El contexto original deberá propagarse a:

```text
Lazy Loading
Eager Loading
Batch Loading
```

---

# 118. Lazy loading

La entidad deberá conservar suficiente domain identity para que:

```text
$order->customer
```

no consulte el tenant actualmente activo de otra request.

---

# 119. Eager loading

```php
Order::query()
    ->with('customer')
    ->get();
```

deberá utilizar el mismo tenant domain.

---

# 120. Batch relation loading

Agrupará sólo cargas tenant-compatible.

---

# 121. Batch compatibility

Debe considerar:

```text
relationship
tenant domain
database
schema
shard
transaction
consistency
```

---

# 122. N+1 telemetry

El detector podrá correlacionar queries dentro del tenant query context sin utilizar TenantId como metric label de alta cardinalidad.

---

# 123. Pagination

Offset pagination conservará TenantQueryContext.

---

# 124. Cursor pagination

El cursor deberá quedar ligado al tenant domain cuando corresponda.

---

# 125. Cursor binding

Conceptualmente:

```text
Cursor
+
Query Fingerprint
+
Tenant Domain Fingerprint
```

---

# 126. Cursor A under B

Debe producir:

```text
CursorTenantBindingException
```

---

# 127. Chunk processing

El contexto se fija para el traversal.

---

# 128. Chunk checkpoint

Checkpoint deberá incluir/bindear:

```text
tenant domain
query fingerprint
placement generation
```

según estrategia.

---

# 129. Lazy collection

Crear:

```php
$lazy = Order::query()->lazy();
```

deberá capturar una definición/contexto seguro.

---

# 130. Deferred execution

Aunque la ejecución ocurra después:

```text
TenantQueryContext
```

no se reconstruirá desde un global mutable.

---

# 131. Streaming result

El stream permanecerá ligado al mismo connection/domain hasta cierre.

---

# 132. Bulk operations

Todos los bulk systems deberán aceptar únicamente contextos compatibles.

---

# 133. Import/export

El contexto de query deberá derivarse del target/source tenant explícito.

---

# 134. Query cache

El cache key deberá incluir:

```text
SemanticQueryFingerprint
+
TenantPersistenceDomain
+
RelevantGenerations
```

cuando el resultado sea tenant-specific.

---

# 135. Result cache

Nunca:

```text
same SQL
=
same tenant result
```

---

# 136. Compiled query cache

Puede compartir compilación tenant-neutral.

---

# 137. Cache domain

Debe distinguir:

```text
Logical Plan Cache
Physical Plan Cache
Compiled SQL Cache
Result Cache
```

porque cada uno necesita diferente grado de tenant binding.

---

# 138. Raw SQL

Es una frontera especial.

---

# 139. Raw SQL API

Evitar:

```php
DB::raw($request->sql);
```

como mecanismo tenant normal.

---

# 140. Tenant raw query

Podrá requerir:

```php
$database->raw(
    sql: '...',
    context: $tenantQueryContext,
    policy: RawQueryPolicy::TENANT_BOUND,
);
```

---

# 141. Shared-schema raw query

No puede probar automáticamente que:

```sql
SELECT * FROM orders
```

esté tenant-scoped.

---

# 142. Policy

Puede exigir:

```text
typed table/query APIs
explicit trusted tenant binding
privileged capability
```

---

# 143. Schema-per-tenant raw query

Debe ejecutarse sólo en schema correcto.

---

# 144. Database-per-tenant raw query

Debe ejecutarse sólo sobre connection target correcto.

---

# 145. Cross-tenant raw SQL

Reservado para APIs administrativas.

---

# 146. Native SQL bypass

Nunca deberá saltarse:

```text
Connection Domain Validation
Transaction Domain Validation
Security/Audit
```

aunque no pueda inspeccionarse semánticamente el SQL.

---

# 147. Query input security

El tenant no se derivará de:

```text
sort
filter
query string
JSON payload
header
```

sin pasar por TenantContext/Authorization.

---

# 148. Tenant IDs as filters

Una API administrativa puede aceptar TenantIds como filtros.

Eso no significa que esos valores se conviertan automáticamente en TenantQueryContext.

---

# 149. Authorization first

Pipeline:

```text
Requested Tenant
      ↓
Resolve Identity
      ↓
Authorize
      ↓
Create TenantContext
      ↓
Create TenantQueryContext
```

---

# 150. QueryContext ≠ Authorization

Un TenantQueryContext válido no sustituye autorización de operación.

---

# 151. Authorization snapshot

El contexto puede contener:

```text
AuthorizationScopeId
```

o fingerprint cuando sea necesario para cache/audit.

---

# 152. Long-running operations

La autorización puede cambiar mientras una operación larga está activa.

La política deberá decidir si:

```text
authorization at start
continuous revalidation
checkpoint revalidation
```

es necesaria.

---

# 153. CrossTenantCapability

Para `CROSS_TENANT` será obligatoria una capability explícita.

---

# 154. Capability model

```php
final readonly class CrossTenantQueryCapability
{
    public function __construct(
        public ActorId $actor,
        public CrossTenantPermission $permission,
        public AuditReason $reason,
        public CapabilityId $id,
    ) {}
}
```

---

# 155. Capability scope

Puede limitar:

```text
allowed tenants
allowed entities
allowed operations
read/write
expiration
```

---

# 156. Cross-tenant query

No deberá generarse simplemente mediante:

```php
withoutTenantScope()
```

---

# 157. Safer API

Conceptualmente:

```php
$admin->crossTenantQuery(
    capability: $capability,
    query: fn ($db) => ...
);
```

---

# 158. Scope removal

La API pública ordinaria no deberá ofrecer un bypass trivial.

---

# 159. Global models

Algunas entidades podrán declararse:

```text
GLOBAL
```

Ejemplo:

```text
Country
Currency
FrameworkMetadata
```

---

# 160. Global model query

No necesita tenant predicate.

---

# 161. Global ≠ cross-tenant

Muy importante:

```text
Global Data
≠
All Tenant Data
```

---

# 162. Mixed query

Una query puede contener:

```text
Tenant-owned root
+
Global reference
```

---

# 163. Mixed query domain

Se clasificará:

```text
TENANT_PLUS_GLOBAL
```

no CROSS_TENANT.

---

# 164. System models

Podrán estar reservados para framework/control plane.

---

# 165. User application access

No tendrán acceso automático a `SYSTEM` sólo porque exista TenantContext.

---

# 166. Context serialization

Un TenantQueryContext activo no deberá serializar conexiones, transactions o EntityManagers.

---

# 167. Safe serialized form

Para jobs podrá existir:

```php
final readonly class TenantQueryContextReference
{
    public function __construct(
        public TenantId $tenant,
        public ?TenantPlacementGeneration $expectedGeneration,
        public QueryScopeReference $scope,
    ) {}
}
```

---

# 168. Re-resolution

Al ejecutar el job:

```text
Reference
   ↓
Current Tenant Placement
   ↓
Fresh TenantQueryContext
```

---

# 169. Why

Placement puede cambiar entre:

```text
dispatch
```

y:

```text
execution
```

---

# 170. Generation policy

Un job podrá declarar:

```text
CURRENT_PLACEMENT
EXACT_GENERATION
AT_LEAST_GENERATION
CUSTOM
```

---

# 171. Persistent runtimes

El sistema deberá ser seguro en:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 172. No static current query tenant

Prohibido:

```php
TenantQueryContext::$current;
```

---

# 173. Scoped provider

Se utilizará un mecanismo scope-aware:

```php
interface TenantQueryContextProvider
{
    public function current(): TenantQueryContext;
}
```

cuya implementación dependa del runtime scope, no de un singleton mutable.

---

# 174. FrankenPHP

Cada request:

```text
RequestScope
   ↓
TenantContext
   ↓
DatabaseContext
   ↓
TenantQueryContext
```

---

# 175. RoadRunner

Al finalizar request:

```text
TenantQueryContext
```

deberá desaparecer/resetearse.

---

# 176. OpenSwoole

Cada coroutine/fiber deberá mantener su propio contexto.

---

# 177. Concurrent example

```text
Coroutine A
TenantQueryContext A
      │
      └── Query A

Coroutine B
TenantQueryContext B
      │
      └── Query B
```

---

# 178. Context capture

Callbacks asíncronos deberán recibir/capturar contexto de forma explícita y segura.

---

# 179. Context drift

Si:

```text
Operation created under A
Operation executed under runtime scope B
```

el sistema no deberá sustituir automáticamente A por B.

---

# 180. Context compatibility

Podrá:

```text
execute with captured A
```

si permitido y válido,

o:

```text
reject due to scope mismatch
```

según política.

---

# 181. Request termination

Debe limpiar:

```text
TenantQueryContext
Query diagnostics
Tenant query caches L0
Query telemetry buffers
```

---

# 182. Context leakage detection

En modo debug/testing podrá verificarse:

```text
no active TenantQueryContext after scope close
```

---

# 183. Query events

Los eventos deberán incluir un contexto seguro.

Ejemplo:

```text
QueryExecuting
```

puede contener:

```text
TenantDomainFingerprint
QueryScope
IsolationStrategy
```

sin exponer secretos.

---

# 184. Event listeners

No deberán llamar a:

```text
currentTenant()
```

para reinterpretar el evento.

---

# 185. Telemetry

Atributos útiles:

```text
tenant_query_scope
tenant_isolation_strategy
schema_bound
shard_bound
cross_tenant
```

---

# 186. High cardinality

No utilizar:

```text
tenant_id
tenant_slug
schema_name
```

como labels métricas por default.

---

# 187. Query profiler

Podrá mostrar tenant context de forma redactada.

Ejemplo:

```text
Query #12

Scope:              TENANT
Isolation:          SCHEMA
Tenant:             [redacted]
Schema Generation:  42
Read Route:         REPLICA
Transaction:        none
```

---

# 188. Slow query detection

Una slow query mantiene tenant domain para diagnóstico interno, pero la agregación debe evitar cardinalidad ilimitada.

---

# 189. Audit

Queries cross-tenant privilegiadas podrán registrar:

```text
actor
capability
scope
target set
operation
reason
outcome
```

---

# 190. Debug toolbar

Podrá indicar:

```text
Tenant Query Context
Isolation Strategy
Database Route
Schema Binding
Shard Binding
Query Count
```

sin revelar credenciales.

---

# 191. Error hierarchy

```text
TenantQueryException
│
├── MissingTenantQueryContextException
├── TenantQueryConflictException
├── TenantQueryScopeException
├── TenantQueryDomainMismatchException
├── TenantQueryConnectionMismatchException
├── TenantQueryTransactionMismatchException
├── TenantQuerySchemaMismatchException
├── TenantQueryShardMismatchException
├── TenantQueryGenerationMismatchException
├── UnauthorizedCrossTenantQueryException
├── UnsafeTenantRawQueryException
└── StaleTenantQueryContextException
```

---

# 192. Error messages

Ejemplo:

```text
Tenant query execution rejected.

Reason:
Query persistence domain is incompatible with the active connection lease.

Query scope:
TENANT

Expected domain:
[redacted fingerprint]

Actual domain:
[redacted fingerprint]
```

---

# 193. No sensitive leakage

Errores públicos no deberán mostrar:

```text
credentials
physical host
schema names if sensitive
tenant secrets
connection DSN
```

---

# 194. Diagnostics command

Conceptualmente:

```text
php volt database:tenant:query-context --tenant=acme
```

---

# 195. Example diagnostics

```text
Tenant Query Context
────────────────────────────────────

Tenant
  Status:               ACTIVE
  Query Scope:          TENANT

Isolation
  Strategy:             SCHEMA_PER_TENANT
  Persistence Domain:   valid
  Placement Generation: 42

Schema
  Bound:                yes
  Version:              120
  Generation:           42

Shard
  Required:             no

Security
  Cross Tenant:         no
  Raw Query Policy:     restricted

Routing
  Reads:                eligible replicas
  Writes:               writer

Runtime
  Scope:                request
  Mutable Global State: none

Result
  Status:               VALID
```

---

# 196. Query explain integration

`EXPLAIN` de VoltStack podrá mostrar:

```text
Tenant Analysis
  Scope: TENANT
  Isolation: SHARED_SCHEMA
  Tenant Predicate: injected
  Domain Validation: passed
  Cross-Tenant Access: no
```

---

# 197. Query fingerprint

El fingerprint semántico deberá considerar tenant semantics.

Pero podrá separar:

```text
QueryShapeFingerprint
```

de:

```text
QueryExecutionDomainFingerprint
```

---

# 198. Why

Así:

```text
Tenant A query shape
=
Tenant B query shape
```

puede compartir:

```text
logical optimization
```

sin compartir:

```text
result cache
```

---

# 199. QueryShapeFingerprint

Puede incluir:

```text
AST structure
entity metadata
logical tenant scope
parameter types
```

sin TenantId concreto.

---

# 200. ExecutionDomainFingerprint

Puede incluir:

```text
persistence domain
tenant identity fingerprint
schema generation
shard generation
placement generation
```

según necesidad.

---

# 201. Cache architecture benefit

Permite:

```text
shared logical plan
+
isolated physical/result state
```

---

# 202. Resource governance

El contexto tenant podrá asociarse con:

```text
QueryBudget
ConcurrencyBudget
Timeout
MemoryBudget
RowLimit
```

---

# 203. Tenant-specific budgets

Podrán definirse por plan/tenant tier.

Pero Database no deberá depender del sistema SaaS.

---

# 204. SaaS integration

Opcionalmente:

```text
VoltStack/Quantum/SaaS
```

puede aportar políticas de recursos.

La dependencia será:

```text
SaaS
   ↓
Database Resource Policy
```

no:

```text
Database
   ↓
SaaS
```

---

# 205. Query timeout

El timeout no deberá modificar tenant identity.

---

# 206. Cancellation

Una query cancelada deberá liberar:

```text
result resources
connection state
query scope resources
```

sin alterar otros tenant contexts.

---

# 207. Retry

Un retry deberá conservar el mismo tenant semantic context salvo re-resolution explícita permitida.

---

# 208. Retry after placement change

Si falla conexión y placement cambia:

```text
Tenant A / generation 42
→
generation 43
```

un retry podrá requerir:

```text
re-resolve
re-plan
re-compile
```

---

# 209. No blind retry

No deberá reutilizar automáticamente:

```text
stale physical plan
stale connection
stale schema binding
```

---

# 210. UNKNOWN write outcome

Si una write tiene outcome desconocido:

```text
UNKNOWN
```

no se reintentará simplemente porque pueda resolverse otro tenant placement.

---

# 211. Testing strategy

Se requerirán:

```text
Unit Tests
Semantic Tests
Query Builder Tests
AST Tests
Optimizer Tests
Planner Tests
Compiler Tests
Executor Tests
ORM Tests
Relationship Tests
Cache Tests
Transaction Tests
Runtime Tests
Security Tests
Concurrency Tests
Fault Injection Tests
```

---

# 212. Basic isolation test

```text
Context A
Query A
Connection A
→ success
```

---

# 213. Connection mismatch test

```text
Context A
Query A
Connection B
→ reject
```

---

# 214. Transaction mismatch test

```text
Transaction A
Query B
→ reject
```

---

# 215. Predicate injection test

Shared schema:

```text
status = pending
```

deberá resultar semánticamente en:

```text
tenant = A
AND status = pending
```

---

# 216. OR safety test

Probar:

```text
status = public OR amount > 100
```

sin permitir escapar del tenant root constraint.

---

# 217. Subquery test

Tenant scope deberá propagarse correctamente.

---

# 218. CTE test

Tenant scope deberá preservarse.

---

# 219. UNION test

A + B deberá rechazarse sin cross-tenant capability.

---

# 220. JOIN test

Tenant A entity + Tenant B entity deberá rechazarse.

---

# 221. Global join test

Tenant + Global deberá permitirse cuando metadata lo declare.

---

# 222. Update test

Mass update no deberá afectar otros tenants.

---

# 223. Delete test

Mass delete no deberá afectar otros tenants.

---

# 224. Insert conflict test

Payload tenant B bajo context A deberá rechazarse.

---

# 225. Upsert test

Conflict semantics deberán permanecer tenant-safe.

---

# 226. Raw SQL test

Un raw query inseguro deberá rechazarse según policy.

---

# 227. Cache test

Misma query shape en A/B no deberá compartir resultados.

---

# 228. Cursor test

Cursor A usado bajo B deberá rechazarse.

---

# 229. Chunk checkpoint test

Checkpoint A no deberá reanudarse bajo B.

---

# 230. Lazy collection test

Lazy collection creada bajo A no deberá cambiar a B al iterar.

---

# 231. Job test

Job deberá re-resolver current placement de A.

---

# 232. Placement change test

Plan generation obsoleta deberá detectarse.

---

# 233. Runtime reuse test

```text
Request A
Request B
Request A
Request C
```

sin leakage.

---

# 234. OpenSwoole concurrency test

Ejecutar simultáneamente:

```text
Coroutine A → Tenant A
Coroutine B → Tenant B
Coroutine C → Tenant C
```

con queries intercaladas.

---

# 235. Context capture test

Una callback creada bajo A y ejecutada después no deberá tomar B desde global mutable.

---

# 236. Security fuzzing

Probar:

```text
malformed tenant IDs
forged cursors
tenant predicates
raw schema names
raw table names
cross-tenant joins
nested subqueries
CTEs
unions
bulk mutations
```

---

# 237. Property-based invariant

Para toda query tenant-scoped:

```text
RowsAffected(Q)
⊆
RowsOwnedBy(Tenant(Q))
```

en estrategias row-based.

---

# 238. Schema invariant

Para schema isolation:

```text
NamespacesAccessed(Q)
⊆
AllowedNamespaces(Tenant(Q))
```

---

# 239. Database invariant

Para database isolation:

```text
DatabasesAccessed(Q)
⊆
AllowedDatabases(Tenant(Q))
```

---

# 240. Shard invariant

```text
ShardsAccessed(Q)
⊆
AllowedShards(Tenant(Q))
```

---

# 241. Proposed directory

```text
src/Quantum/Multitenancy/Database/Query/
│
├── Contract/
│   ├── TenantQueryContextProvider.php
│   ├── TenantQueryContextFactory.php
│   ├── TenantQueryIsolationPass.php
│   ├── TenantQueryExecutionGuard.php
│   └── TenantQueryPolicy.php
│
├── Context/
│   ├── TenantQueryContext.php
│   ├── TenantQueryContextReference.php
│   ├── TenantQueryScope.php
│   └── TenantQueryDomain.php
│
├── Metadata/
│   ├── TenantScopeMetadata.php
│   ├── TenantScopeKind.php
│   └── TenantOwnershipMetadata.php
│
├── Analysis/
│   ├── TenantQueryAnalyzer.php
│   ├── TenantQueryDomainAnalyzer.php
│   ├── TenantJoinAnalyzer.php
│   ├── TenantSubqueryAnalyzer.php
│   └── TenantQueryConflictDetector.php
│
├── Transformation/
│   ├── TenantPredicateInjector.php
│   ├── TenantNamespaceBinder.php
│   ├── TenantInsertBinder.php
│   └── TenantQueryTransformer.php
│
├── Planning/
│   ├── TenantQueryPlanningContext.php
│   ├── TenantPhysicalDomainBinder.php
│   └── TenantQueryGenerationValidator.php
│
├── Security/
│   ├── CrossTenantQueryCapability.php
│   ├── CrossTenantQueryGuard.php
│   ├── TenantRawQueryPolicy.php
│   └── TenantQuerySecurityPolicy.php
│
├── Runtime/
│   ├── ScopedTenantQueryContextProvider.php
│   └── TenantQueryContextResetter.php
│
├── Telemetry/
│   └── TenantQueryTelemetry.php
│
├── Diagnostics/
│   └── TenantQueryDiagnostics.php
│
└── Exception/
    ├── TenantQueryException.php
    ├── MissingTenantQueryContextException.php
    ├── TenantQueryConflictException.php
    ├── TenantQueryScopeException.php
    ├── TenantQueryDomainMismatchException.php
    ├── TenantQueryConnectionMismatchException.php
    ├── TenantQueryTransactionMismatchException.php
    ├── TenantQueryGenerationMismatchException.php
    ├── UnauthorizedCrossTenantQueryException.php
    └── UnsafeTenantRawQueryException.php
```

---

# 242. Dependency model

```text
Application
    ↓
TenantContext
    ↓
TenantQueryContext
    ↓
Query Engine
    ↓
Semantic Engine
    ↓
Optimizer
    ↓
Planner
    ↓
Compiler
    ↓
Executor
    ↓
Connection
```

---

# 243. Forbidden dependency

Nunca:

```text
Compiler
   ↓
globalCurrentTenant()
```

ni:

```text
Driver
   ↓
TenantResolver
```

---

# 244. Context propagation

Conceptualmente:

```text
Q0 = Query(AST, TenantContext=A)

Q1 = Normalize(Q0)
Q2 = TenantScope(Q1)
Q3 = SemanticAnalyze(Q2)
Q4 = Optimize(Q3)
Q5 = Plan(Q4)
Q6 = Compile(Q5)
Q7 = Execute(Q6)
```

deberá cumplirse:

```text
TenantDomain(Q0)
=
TenantDomain(Q1)
=
TenantDomain(Q2)
=
TenantDomain(Q3)
=
TenantDomain(Q4)
=
TenantDomain(Q5)
=
TenantDomain(Q6)
=
TenantDomain(Q7)
```

salvo una transición explícita, autorizada y validada.

---

# 245. Tenant query invariants

## DB-TQC-001

Toda query tenant-owned tendrá TenantQueryContext.

## DB-TQC-002

TenantQueryContext será typed.

## DB-TQC-003

TenantQueryContext será inmutable.

## DB-TQC-004

TenantQueryContext será distinto de TenantContext.

## DB-TQC-005

TenantQueryContext será parte del QueryContext general.

## DB-TQC-006

Missing context no equivaldrá a GLOBAL.

## DB-TQC-007

Missing context no equivaldrá a CROSS_TENANT.

## DB-TQC-008

Tenant-owned query sin contexto fallará cerrado.

## DB-TQC-009

Tenant scope será explícito.

## DB-TQC-010

GLOBAL será distinto de all-tenants.

## DB-TQC-011

SYSTEM será distinto de GLOBAL.

## DB-TQC-012

CROSS_TENANT requerirá capability.

## DB-TQC-013

Query Builder no resolverá tenant desde global mutable.

## DB-TQC-014

Model static API no implicará static tenant state.

## DB-TQC-015

Repository no decidirá tenant.

## DB-TQC-016

EntityManager tenant domain deberá coincidir con query domain.

## DB-TQC-017

Query derivation conservará context.

## DB-TQC-018

Ordinary query no podrá cambiar tenant mediante setter trivial.

## DB-TQC-019

Tenant isolation será aplicada antes de optimization.

## DB-TQC-020

Tenant isolation no será únicamente predicate injection.

## DB-TQC-021

Row isolation podrá generar tenant predicate.

## DB-TQC-022

Schema isolation podrá generar namespace binding.

## DB-TQC-023

Database isolation podrá generar connection binding.

## DB-TQC-024

Shard isolation podrá generar routing constraint.

## DB-TQC-025

Hybrid isolation podrá combinar mecanismos.

## DB-TQC-026

Tenant parameters serán bound parameters.

## DB-TQC-027

Tenant values no serán SQL literals concatenados.

## DB-TQC-028

Tenant binding vendrá de trusted context.

## DB-TQC-029

Explicit conflicting tenant predicate será detectado.

## DB-TQC-030

Matching redundant tenant predicate podrá normalizarse.

## DB-TQC-031

OR expressions no podrán escapar del root tenant constraint.

## DB-TQC-032

Subqueries tendrán tenant semantics explícitas.

## DB-TQC-033

CTEs conservarán tenant domain.

## DB-TQC-034

Recursive CTEs conservarán tenant domain.

## DB-TQC-035

UNION entre tenants será rechazado por default.

## DB-TQC-036

Tenant-local joins serán soportados.

## DB-TQC-037

Tenant-global joins podrán ser soportados.

## DB-TQC-038

Cross-tenant joins requerirán explicit capability.

## DB-TQC-039

UPDATE preservará tenant scope.

## DB-TQC-040

DELETE preservará tenant scope.

## DB-TQC-041

INSERT preservará tenant ownership.

## DB-TQC-042

Bulk insert preservará tenant ownership.

## DB-TQC-043

Bulk update preservará tenant scope.

## DB-TQC-044

Bulk delete preservará tenant scope.

## DB-TQC-045

Upsert preservará tenant semantics.

## DB-TQC-046

Tenant ownership metadata será distinta de physical isolation strategy.

## DB-TQC-047

Entity model podrá permanecer portable entre isolation strategies.

## DB-TQC-048

Semantic Graph representará tenant ownership cuando corresponda.

## DB-TQC-049

UNKNOWN semantic tenant domain no será SAFE.

## DB-TQC-050

Optimizer preservará tenant semantics.

## DB-TQC-051

Optimizer podrá deduplicar sólo con equivalencia demostrada.

## DB-TQC-052

Predicate pushdown preservará tenant semantics.

## DB-TQC-053

Join reordering preservará tenant domain.

## DB-TQC-054

Query rewrite no eliminará required tenant constraints.

## DB-TQC-055

Planner recibirá tenant domain explícito.

## DB-TQC-056

Planner no autorizará nuevos tenants.

## DB-TQC-057

Physical plan podrá quedar generation-bound.

## DB-TQC-058

Stale physical plan será detectable.

## DB-TQC-059

Logical plans podrán ser tenant-neutral cuando sea seguro.

## DB-TQC-060

Physical plan reuse requerirá domain compatibility.

## DB-TQC-061

Compiler no elegirá tenant.

## DB-TQC-062

Compiler no autorizará tenant.

## DB-TQC-063

Compiler no consultará global tenant state.

## DB-TQC-064

Executor validará plan/connection/domain.

## DB-TQC-065

Wrong tenant connection será rechazada.

## DB-TQC-066

Wrong placement generation será detectable.

## DB-TQC-067

Transaction tenant domain deberá coincidir.

## DB-TQC-068

Transaction A no ejecutará query B.

## DB-TQC-069

No habrá implicit transaction tenant switching.

## DB-TQC-070

Read routing permanecerá dentro del tenant domain.

## DB-TQC-071

Replica routing permanecerá dentro del tenant replication group.

## DB-TQC-072

Sticky state será tenant-domain aware.

## DB-TQC-073

Locking query conservará tenant context.

## DB-TQC-074

Tenant será distinto de shard.

## DB-TQC-075

Cross-shard será distinto de cross-tenant.

## DB-TQC-076

Unknown write shard será rechazado.

## DB-TQC-077

Multi-shard reads serán explícitos.

## DB-TQC-078

Relationship loading heredará tenant domain.

## DB-TQC-079

Lazy loading no consultará otro runtime tenant.

## DB-TQC-080

Eager loading preservará tenant domain.

## DB-TQC-081

Batch relation loading agrupará sólo domains compatibles.

## DB-TQC-082

Pagination conservará tenant context.

## DB-TQC-083

Cursor pagination bindeará tenant domain.

## DB-TQC-084

Cursor A no será válido bajo B.

## DB-TQC-085

Chunk processing conservará tenant context.

## DB-TQC-086

Chunk checkpoints serán tenant-bound.

## DB-TQC-087

Lazy collection conservará captured context.

## DB-TQC-088

Deferred execution no re-resolverá tenant desde global mutable.

## DB-TQC-089

Streaming conservará connection/domain affinity.

## DB-TQC-090

Import/export serán tenant-bound.

## DB-TQC-091

Query result cache será tenant-aware.

## DB-TQC-092

Same SQL no implicará same result domain.

## DB-TQC-093

Logical plan cache podrá compartirse cuando sea seguro.

## DB-TQC-094

Physical cache keys incorporarán relevant domain identity.

## DB-TQC-095

Raw SQL será una security boundary.

## DB-TQC-096

Raw SQL no evitará connection domain validation.

## DB-TQC-097

Raw SQL no evitará transaction domain validation.

## DB-TQC-098

Tenant no se derivará directamente de untrusted query input.

## DB-TQC-099

Authorization ocurrirá antes de crear privileged tenant context.

## DB-TQC-100

TenantQueryContext no sustituirá Authorization.

## DB-TQC-101

CrossTenantCapability será explícita.

## DB-TQC-102

CrossTenantCapability será scoped.

## DB-TQC-103

CrossTenantCapability podrá expirar.

## DB-TQC-104

No existirá bypass público trivial equivalente a `withoutTenantScope()`.

## DB-TQC-105

GLOBAL data será distinto de all tenant data.

## DB-TQC-106

TENANT_PLUS_GLOBAL será representable.

## DB-TQC-107

SYSTEM resources no serán accesibles automáticamente.

## DB-TQC-108

Active TenantQueryContext no serializará live connection.

## DB-TQC-109

Active TenantQueryContext no serializará transaction.

## DB-TQC-110

Jobs podrán serializar safe context references.

## DB-TQC-111

Jobs re-resolverán placement cuando corresponda.

## DB-TQC-112

Generation-sensitive jobs podrán exigir exact generation.

## DB-TQC-113

Persistent runtime no tendrá static current tenant query.

## DB-TQC-114

FrankenPHP context será request-scoped.

## DB-TQC-115

RoadRunner context será request-scoped.

## DB-TQC-116

OpenSwoole context será coroutine-scoped.

## DB-TQC-117

Concurrent tenants no compartirán mutable query context.

## DB-TQC-118

Async callbacks no dependerán de mutable global tenant.

## DB-TQC-119

Context drift será detectable.

## DB-TQC-120

Scope close limpiará tenant query state.

## DB-TQC-121

Debug mode podrá detectar leaked context.

## DB-TQC-122

Query events transportarán domain context suficiente.

## DB-TQC-123

Event listeners no reinterpretarán tenant desde global state.

## DB-TQC-124

Telemetry evitará TenantId como default metric label.

## DB-TQC-125

Profiler podrá mostrar tenant information redactada.

## DB-TQC-126

Cross-tenant queries serán auditables.

## DB-TQC-127

Errors no expondrán credentials.

## DB-TQC-128

Errors no expondrán sensitive schema/host information innecesariamente.

## DB-TQC-129

Query fingerprints distinguirán shape de execution domain.

## DB-TQC-130

Same query shape podrá compartir logical optimization.

## DB-TQC-131

Same query shape no implicará shared result cache.

## DB-TQC-132

Resource governance podrá usar tenant context.

## DB-TQC-133

Database no dependerá del SaaS package.

## DB-TQC-134

SaaS podrá aportar optional resource policies.

## DB-TQC-135

Cancellation preservará isolation.

## DB-TQC-136

Retry preservará tenant semantics.

## DB-TQC-137

Retry tras placement change podrá requerir replanning.

## DB-TQC-138

Retry no reutilizará stale connection silenciosamente.

## DB-TQC-139

UNKNOWN write outcome no se reintentará ciegamente.

## DB-TQC-140

Tenant query system tendrá semantic tests.

## DB-TQC-141

Tenant query system tendrá optimizer tests.

## DB-TQC-142

Tenant query system tendrá planner tests.

## DB-TQC-143

Tenant query system tendrá compiler tests.

## DB-TQC-144

Tenant query system tendrá executor tests.

## DB-TQC-145

Tenant query system tendrá ORM tests.

## DB-TQC-146

Tenant query system tendrá relationship tests.

## DB-TQC-147

Tenant query system tendrá raw SQL security tests.

## DB-TQC-148

Tenant query system tendrá cache isolation tests.

## DB-TQC-149

Tenant query system tendrá persistent runtime tests.

## DB-TQC-150

Tenant query system tendrá concurrent runtime tests.

## DB-TQC-151

Tenant query system tendrá fault-injection tests.

## DB-TQC-152

Rows affected en row isolation deberán pertenecer al tenant.

## DB-TQC-153

Namespaces accessed deberán estar autorizados.

## DB-TQC-154

Databases accessed deberán estar autorizadas.

## DB-TQC-155

Shards accessed deberán estar autorizados.

## DB-TQC-156

Tenant context será preservado end-to-end.

## DB-TQC-157

Una optimización de rendimiento nunca podrá debilitar tenant isolation.

## DB-TQC-158

Una cache hit nunca podrá debilitar tenant isolation.

## DB-TQC-159

Un failover nunca podrá debilitar tenant isolation.

## DB-TQC-160

Un retry nunca podrá debilitar tenant isolation.

## DB-TQC-161

Un runtime adapter nunca podrá redefinir tenant semantics.

## DB-TQC-162

Active Record y Repository usarán el mismo TenantQueryContext system.

## DB-TQC-163

No existirá un segundo query engine para multitenancy.

## DB-TQC-164

TenantQueryContext será una integración del Query Engine canónico.

## DB-TQC-165

Isolation strategy podrá cambiar sin reescribir business queries.

## DB-TQC-166

TenantQueryContext podrá representar row/schema/database/shard isolation.

## DB-TQC-167

UNKNOWN context compatibility fallará cerrado.

## DB-TQC-168

No se inferirá cross-tenant privilege por ausencia de contexto.

## DB-TQC-169

Toda query tenant-scoped deberá poder explicar su isolation domain.

## DB-TQC-170

Toda ejecución tenant-aware deberá demostrar compatibilidad entre query, persistence domain, transaction y connection antes de enviar una operación al driver.

---

# 246. Modelo formal

Sea:

```text
T = TenantQueryContext
Q = Logical Query
S = Semantic Query
P = Physical Plan
C = Compiled Query
L = Connection Lease
X = Transaction Context
```

El pipeline será:

```text
(Q,T)
   ↓
TenantIsolation
   ↓
(S,T)
   ↓
Optimization
   ↓
(S',T)
   ↓
Planning
   ↓
(P,T)
   ↓
Compilation
   ↓
(C,T)
   ↓
Execution Validation
   ↓
Execute(C,L)
```

---

# 247. Invariante de propagación

Para toda transformación segura:

```text
F(Q,T) → (Q',T')
```

deberá cumplirse:

```text
TenantSemantics(T')
=
TenantSemantics(T)
```

salvo transición privilegiada explícita.

---

# 248. Invariante de ejecución

Una query puede ejecutarse sólo si:

```text
Compatible(
    QueryPersistenceDomain,
    ConnectionPersistenceDomain
)
```

y, si existe transaction:

```text
Compatible(
    QueryPersistenceDomain,
    TransactionPersistenceDomain
)
```

---

# 249. Row isolation formal

Para tenant `t`:

```text
Result(Q,t)
=
{ r ∈ Result(Q) | Owner(r) = t }
```

para recursos tenant-owned row-scoped.

Para mutaciones:

```text
Affected(Q,t)
⊆
OwnedRows(t)
```

---

# 250. Schema isolation formal

```text
Namespaces(Q,t)
⊆
AllowedNamespaces(t)
```

---

# 251. Database isolation formal

```text
Database(Q,t)
∈
AllowedDatabases(t)
```

---

# 252. Hybrid isolation

En una arquitectura híbrida:

```text
TenantSafe(Q,t)
=
RowSafe(Q,t)
∧
SchemaSafe(Q,t)
∧
DatabaseSafe(Q,t)
∧
ShardSafe(Q,t)
```

para todas las dimensiones aplicables.

---

# 253. Arquitectura completa

```text
                         APPLICATION
                              │
                              ▼
                         TenantContext
                              │
                              ▼
                         Authorization
                              │
                              ▼
                    TenantQueryContextFactory
                              │
                              ▼
                     TenantQueryContext
                              │
             ┌────────────────┼────────────────┐
             │                │                │
             ▼                ▼                ▼
        Model API         Repository      EntityManager
             │                │                │
             └────────────────┼────────────────┘
                              ▼
                         Query Builder
                              │
                              ▼
                        Logical AST
                              │
                              ▼
                  Tenant Isolation Pass
                              │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                 ▼
      Row Predicate     Schema Binding     Shard/DB Domain
            │                 │                 │
            └─────────────────┼─────────────────┘
                              ▼
                     Semantic Analysis
                              │
                              ▼
                          Optimizer
                              │
                              ▼
                           Planner
                              │
                              ▼
                     Physical Query Plan
                              │
                              ▼
                          Compiler
                              │
                              ▼
                       Compiled Query
                              │
                              ▼
                  Tenant Execution Guard
                              │
                 ┌────────────┴────────────┐
                 ▼                         ▼
              VALID                  INVALID/UNKNOWN
                 │                         │
                 ▼                         ▼
             Executor                    REJECT
                 │
                 ▼
          Connection Manager
                 │
                 ▼
           Connection Lease
                 │
                 ▼
              DRIVER
                 │
                 ▼
             DATABASE
```

---

# 254. Flujo de ejemplo: shared schema

```php
$orders = Order::query()
    ->where('status', OrderStatus::PENDING)
    ->get();
```

Contexto:

```text
Tenant = ACME
Isolation = SHARED_DATABASE_SHARED_SCHEMA
Tenant Key = tenant_id
```

Flujo:

```text
Order Query
    ↓
TenantQueryContext(ACME)
    ↓
Logical Predicate
status = pending
    ↓
Tenant Isolation Pass
tenant_id = ACME
AND
status = pending
    ↓
Semantic Analysis
    ↓
Planner
    ↓
Compiler
    ↓
SELECT ...
FROM orders
WHERE tenant_id = ?
  AND status = ?
```

---

# 255. Flujo de ejemplo: schema-per-tenant

La misma aplicación:

```php
$orders = Order::query()
    ->where('status', OrderStatus::PENDING)
    ->get();
```

Contexto:

```text
Tenant = ACME
Isolation = SHARED_DATABASE_SEPARATE_SCHEMA
Schema = t_42
```

Flujo:

```text
Order Query
    ↓
TenantQueryContext(ACME)
    ↓
Logical Table = orders
    ↓
Tenant Namespace Binding
Schema = t_42
    ↓
Compiler
    ↓
SELECT ...
FROM "t_42"."orders"
WHERE status = ?
```

---

# 256. Flujo de ejemplo: database-per-tenant

Mismo código:

```php
$orders = Order::query()
    ->where('status', OrderStatus::PENDING)
    ->get();
```

Contexto:

```text
Tenant = ACME
Isolation = DATABASE_PER_TENANT
Database = customer_acme
```

Flujo:

```text
Order Query
      ↓
TenantQueryContext
      ↓
Connection Resolution
      ↓
Database customer_acme
      ↓
Compiler
      ↓
SELECT ...
FROM orders
WHERE status = ?
```

---

# 257. Resultado arquitectónico

Los tres casos conservan:

```php
Order::query()
    ->where('status', OrderStatus::PENDING)
    ->get();
```

porque:

```text
Business Query
≠
Tenant Physical Placement
```

Esta separación permitirá que VoltStack mueva tenants entre:

```text
Shared Rows
      ↓
Separate Schema
      ↓
Dedicated Database
      ↓
Dedicated Shard
      ↓
Dedicated Cluster
```

sin crear APIs ORM diferentes.

---

# 258. Regla arquitectónica definitiva

> **TenantQueryContext será la prueba de procedencia y aislamiento de una operación Database tenant-aware. Se construirá desde un TenantContext autorizado, será inmutable durante el pipeline y acompañará a la query desde Model/Repository/EntityManager hasta Execution. El Query Engine podrá transformar predicates, namespaces y planes físicos según la estrategia de aislamiento, pero ninguna etapa podrá cambiar, eliminar o reconstruir implícitamente la identidad tenant consultando estado global mutable.**

En consecuencia:

```text
Tenant Query Safety
=
Explicit Context
+
Semantic Isolation
+
Domain Validation
+
Connection Compatibility
+
Transaction Compatibility
+
Runtime Isolation
```

y:

```text
Missing Context
≠
Global Access
```

```text
Global Access
≠
Cross-Tenant Access
```

```text
Optimization
≠
Permission To Weaken Isolation
```

---

# 259. Estado del Bloque 26

```text
BLOCK 26 — MULTITENANCY INTEGRATION

✓ 261_DATABASE_MULTITENANCY_INTEGRATION_ARCHITECTURE.md
✓ 262_DATABASE_TENANT_CONNECTION_RESOLUTION_SYSTEM.md
✓ 263_DATABASE_TENANT_DATABASE_ISOLATION_SYSTEM.md
✓ 264_DATABASE_TENANT_SCHEMA_ISOLATION_SYSTEM.md
✓ 265_DATABASE_TENANT_QUERY_CONTEXT_SYSTEM.md
│
└── 266_DATABASE_TENANT_MIGRATION_INTEGRATION_SYSTEM.md
```

---

# 260. Siguiente documento

```text
266_DATABASE_TENANT_MIGRATION_INTEGRATION_SYSTEM.md
```

El siguiente documento deberá cerrar el bloque de Multitenancy definiendo:

```text
Tenant Migration Architecture
Migration Scope
Tenant Migration Context
Migration Discovery
Migration Versioning
Global vs Tenant Migrations
Shared-Schema Migrations
Schema-per-Tenant Migrations
Database-per-Tenant Migrations
Shard-aware Migrations
Migration Fleet Planning
Tenant Selection
Migration Batches
Concurrency Control
Migration Leases
Migration Locks
Per-Tenant Migration Repository
Migration State
Migration Version Skew
Compatibility Windows
Expand/Contract
Zero-Downtime Tenant Migrations
Migration Scheduling
Pause/Resume
Checkpointing
Retry
Failure Recovery
Partial Fleet Outcomes
Canary Tenants
Rollout Waves
Rollback
Forward Recovery
Schema Drift
Tenant Provisioning
Tenant Mobility
Read/Write Cutover
Persistent Runtime Coordination
Cache Invalidation
Jobs
Telemetry
Audit
Security
Resource Governance
CLI
Diagnostics
Testing
```

bajo la regla:

> **Una migración multitenant de VoltStack no será una única migración repetida ciegamente N veces. Será una operación de flota compuesta por unidades tenant-scoped independientes, versionadas, observables y recuperables, cuya ejecución deberá respetar isolation strategy, compatibility windows, placement generations y resultados parciales sin fingir atomicidad global.**