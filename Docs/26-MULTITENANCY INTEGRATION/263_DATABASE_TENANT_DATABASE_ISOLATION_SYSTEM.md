# 263_DATABASE_TENANT_DATABASE_ISOLATION_SYSTEM.md

# VoltStack Quantum Database
## Tenant Database Isolation System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Integración:** `VoltStack/Quantum/Multitenancy`  
**Documento:** 263 — Tenant Database Isolation System  
**Bloque:** 26 — Multitenancy Integration  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `262_DATABASE_TENANT_CONNECTION_RESOLUTION_SYSTEM.md`  
**Siguiente documento:** `264_DATABASE_TENANT_SCHEMA_ISOLATION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura mediante la cual VoltStack Database preservará el aislamiento entre tenants cuando los datos estén distribuidos mediante diferentes estrategias físicas y lógicas:

```text
Shared Database / Shared Schema
Shared Database / Separate Schema
Database per Tenant
Shard per Tenant
Dedicated Database
Dedicated Cluster
Hybrid Placement
```

El aislamiento no se limitará a seleccionar una base de datos distinta.

La regla central será:

> **El aislamiento tenant de VoltStack será una propiedad end-to-end del dominio de persistencia: TenantContext, Query Context, ORM identity, Connection Resolution, Transaction Context, Cache, Relationships, Routing y Runtime State deberán coincidir con el mismo dominio tenant antes de que una operación pueda considerarse segura.**

Por tanto:

```text
Different Database
≠
Complete Tenant Isolation
```

y:

```text
Tenant Isolation
=
Identity Isolation
+
Query Isolation
+
Persistence Isolation
+
Connection Isolation
+
Transaction Isolation
+
Cache Isolation
+
Runtime Isolation
+
Security Isolation
```

---

# 2. Objetivo fundamental

VoltStack deberá impedir que:

```text
Tenant A
```

pueda accidentalmente:

```text
read Tenant B data
write Tenant B data
hydrate Tenant B entities
reuse Tenant B ORM state
reuse unsafe Tenant B connection state
commit inside Tenant B transaction
resolve Tenant B relationships
read Tenant B cache entries
execute Tenant B jobs
restore Tenant B backup
```

sin una operación cross-tenant explícita y autorizada.

---

# 3. Tenant isolation ≠ tenant filter

Una implementación básica podría considerar suficiente:

```sql
WHERE tenant_id = ?
```

VoltStack no adoptará esa simplificación como arquitectura general.

El filtro sólo cubre una estrategia concreta:

```text
Shared Database
+
Shared Schema
```

pero no resuelve por sí solo:

```text
IdentityMap
Transactions
Connections
Cache
Schemas
Shards
Persistent Workers
Jobs
Raw SQL
Relationships
Backups
Restores
Migrations
```

---

# 4. Isolation domains

Se introducirá el concepto:

```text
TenantIsolationDomain
```

como representación del dominio dentro del cual los recursos Database pueden considerarse compatibles.

Conceptualmente:

```php
final readonly class TenantIsolationDomain
{
    public function __construct(
        public TenantId $tenant,
        public TenantIsolationStrategy $strategy,
        public TenantPersistenceDomain $persistenceDomain,
        public TenantPlacementGeneration $placementGeneration,
    ) {}
}
```

---

# 5. TenantPersistenceDomain

Representará el dominio efectivo de persistencia.

Ejemplo conceptual:

```php
final readonly class TenantPersistenceDomain
{
    public function __construct(
        public LogicalDatabaseId $logicalDatabase,
        public ?DatabaseName $database,
        public ?SchemaName $schema,
        public ?ShardId $shard,
        public ?RegionId $region,
    ) {}
}
```

---

# 6. TenantId ≠ PersistenceDomain

Un tenant puede moverse:

```text
PersistenceDomain A
        ↓
PersistenceDomain B
```

sin cambiar:

```text
TenantId
```

---

# 7. PersistenceDomain ≠ Connection

Una conexión es un recurso temporal.

El dominio de persistencia es una propiedad lógica.

---

# 8. PersistenceDomain ≠ DatabaseName

Puede incluir:

```text
logical database
physical database
schema
shard
region
placement generation
```

dependiendo de la estrategia.

---

# 9. Isolation boundary

Una frontera tenant podrá existir en distintos niveles:

```text
Row
Schema
Database
Shard
Cluster
Region
```

---

# 10. Isolation strength

VoltStack no asumirá automáticamente que una estrategia es universalmente "mejor".

Cada una tiene:

```text
security properties
operational cost
resource cost
scalability properties
migration complexity
failure domains
```

---

# 11. Estrategias soportadas

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

# 12. Shared Database / Shared Schema

Arquitectura:

```text
Database
│
├── orders
│    ├── Tenant A rows
│    ├── Tenant B rows
│    └── Tenant C rows
│
└── customers
     ├── Tenant A rows
     ├── Tenant B rows
     └── Tenant C rows
```

La frontera principal será:

```text
tenant key
```

---

# 13. Row-level isolation

Ejemplo:

```text
orders
────────────────────
id
tenant_id
customer_id
status
```

Toda operación tenant-scoped deberá incorporar semánticamente:

```text
tenant_id = EffectiveTenant
```

cuando corresponda.

---

# 14. Shared-schema risk

Esta estrategia tiene alta sensibilidad a errores de:

```text
missing predicates
unsafe raw SQL
incorrect joins
incorrect relationships
cache key collisions
bulk operations
imports
exports
```

---

# 15. Shared Database / Separate Schema

Arquitectura:

```text
Database
│
├── tenant_a
│   ├── customers
│   └── orders
│
├── tenant_b
│   ├── customers
│   └── orders
│
└── tenant_c
    ├── customers
    └── orders
```

---

# 16. Schema isolation

La frontera primaria será:

```text
SchemaName
```

aunque el TenantContext seguirá siendo necesario.

---

# 17. Schema isolation ≠ complete isolation

Compartir:

```text
database process
credentials
connection pool
server resources
```

puede mantener riesgos de:

```text
session contamination
privilege misconfiguration
resource contention
```

---

# 18. Database per Tenant

```text
Tenant A
   ↓
Database A

Tenant B
   ↓
Database B
```

---

# 19. Database boundary

La separación física/lógica de bases reduce algunos riesgos de:

```text
missing tenant predicate
```

pero no elimina:

```text
wrong connection resolution
wrong cache key
wrong IdentityMap
wrong transaction context
wrong restore target
wrong administrative operation
```

---

# 20. Dedicated database

Un tenant enterprise podrá utilizar:

```text
Tenant A
   ↓
Dedicated Database
   ↓
Dedicated Credentials
```

---

# 21. Dedicated cluster

Nivel mayor:

```text
Tenant A
   ↓
Dedicated Cluster
   ↓
Writer
   ├── Replica 1
   └── Replica 2
```

---

# 22. Shard per tenant

```text
Shard 1
├── Tenant A
├── Tenant B
└── Tenant C

Shard 2
├── Tenant D
└── Tenant E
```

Aquí existen dos dimensiones:

```text
Tenant Isolation
+
Shard Routing
```

---

# 23. Tenant ≠ shard

Nunca:

```text
TenantId = ShardId
```

como regla general.

---

# 24. Hybrid isolation

VoltStack deberá permitir:

```text
Small Tenants
      ↓
Shared Database

Medium Tenants
      ↓
Separate Schemas

Large Tenants
      ↓
Dedicated Database

Enterprise Tenants
      ↓
Dedicated Cluster
```

sin crear ORM diferentes.

---

# 25. Isolation architecture

```text
                         TenantContext
                              │
                              ▼
                    TenantIsolationDomain
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
     Query Isolation      ORM Isolation      Connection
                                                Isolation
          │                   │                   │
          └───────────────────┼───────────────────┘
                              ▼
                    Transaction Isolation
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
       Cache              Relationship         Runtime
      Isolation             Isolation          Isolation
                              │
                              ▼
                      Persistence Domain
```

---

# 26. Isolation validator

Se propone:

```php
interface TenantIsolationValidator
{
    public function validate(
        TenantIsolationContext $context
    ): TenantIsolationValidationResult;
}
```

---

# 27. TenantIsolationContext

Podrá reunir:

```text
TenantContext
DatabaseContext
ORM Context
TransactionContext
Connection Resolution
Query Context
Persistence Domain
```

---

# 28. Validation result

Estados conceptuales:

```text
VALID
INVALID
UNKNOWN
NOT_APPLICABLE
```

---

# 29. UNKNOWN ≠ VALID

Regla crítica:

```text
UNKNOWN
≠
SAFE
```

---

# 30. Fail-closed rule

Para operaciones tenant-scoped:

```text
Isolation = UNKNOWN
```

deberá normalmente producir:

```text
REJECT
```

---

# 31. Query isolation

Toda query tenant-scoped deberá conocer el dominio tenant efectivo antes de ejecución.

---

# 32. QueryContext integration

El sistema utilizará posteriormente:

```text
265_DATABASE_TENANT_QUERY_CONTEXT_SYSTEM.md
```

para representar esta información.

---

# 33. Shared-schema query

Ejemplo lógico:

```php
Order::query()
    ->where('status', 'pending')
    ->get();
```

podrá convertirse semánticamente en:

```text
Entity: Order
Tenant: A
Predicate:
    status = pending
    AND tenant_id = A
```

---

# 34. No SQL patching

Nunca:

```php
$sql .= ' AND tenant_id = ' . $tenant;
```

---

# 35. Query AST

El TenantScope deberá incorporarse antes de:

```text
Optimization
Planning
Compilation
Execution
```

---

# 36. Query isolation proof

Una operación podrá considerarse query-isolated cuando:

```text
EffectiveTenant
=
QueryTenantScope
=
TargetPersistenceDomainTenant
```

según estrategia.

---

# 37. Database-per-tenant query

Puede no necesitar:

```text
tenant_id predicate
```

si la base completa pertenece al tenant.

Pero seguirá necesitando:

```text
TenantContext
```

para resolver correctamente la base.

---

# 38. Defense in depth

Una aplicación puede decidir conservar `tenant_id` incluso dentro de databases dedicadas.

VoltStack podrá soportarlo como capa adicional, pero no deberá imponerlo universalmente.

---

# 39. ORM identity isolation

El IdentityMap deberá incorporar el dominio tenant cuando corresponda.

---

# 40. Unsafe identity key

Incorrecto:

```text
User:10
```

si dos tenants pueden tener:

```text
User#10
```

---

# 41. Safe identity key

Conceptualmente:

```text
Tenant A + User + 10
```

y:

```text
Tenant B + User + 10
```

son identidades distintas.

---

# 42. Identity formula

```text
ORMIdentity
=
EntityType
+
EntityIdentifier
+
PersistenceDomainIdentity
```

donde `PersistenceDomainIdentity` incorpora tenant cuando sea necesario.

---

# 43. Same tenant after placement movement

Debe evitarse usar el physical database name como identidad lógica permanente.

Si Tenant A cambia:

```text
DB-X
→
DB-Y
```

su identidad de dominio puede seguir representando al mismo tenant lógico, pero el EntityManager antiguo no deberá mezclarse automáticamente con el nuevo placement.

---

# 44. Placement generation

Por ello:

```text
ManagedContext
```

deberá estar ligado a:

```text
TenantPlacementGeneration
```

cuando el cambio de placement pueda invalidar sus garantías.

---

# 45. EntityManager tenant affinity

Un EntityManager tenant-scoped tendrá afinidad con:

```text
TenantId
PersistenceDomain
PlacementGeneration
```

---

# 46. EntityManager switching

No:

```text
$entityManager->setTenant($tenantB);
```

sobre un manager que contiene entidades de A.

---

# 47. Safe switching

Debe crearse/resolverse otro scope:

```text
Tenant A Scope
   ↓
EntityManager A

Tenant B Scope
   ↓
EntityManager B
```

---

# 48. UnitOfWork isolation

El UnitOfWork no deberá contener simultáneamente entidades de dominios incompatibles salvo un modo administrativo explícito diseñado para ello.

---

# 49. ChangeSet isolation

Un ChangeSet deberá conservar:

```text
EntityTenantDomain
```

para validar persistencia.

---

# 50. Persistence planner

Antes de producir Query Models deberá validar que:

```text
Entity Domain
=
Persistence Context Domain
```

---

# 51. Cross-tenant persistence

Ejemplo:

```text
Context: Tenant B

persist(
    Customer belonging to Tenant A
)
```

debe fallar.

---

# 52. New entity assignment

Para una entidad nueva tenant-scoped:

```text
NEW
```

el tenant podrá asignarse desde el contexto de persistencia.

---

# 53. Tenant key source

Debe provenir de:

```text
trusted TenantContext
```

no de:

```text
untrusted request input
```

por default.

---

# 54. Tenant key mutation

Una entidad managed no deberá cambiar arbitrariamente:

```text
tenant_id A → B
```

---

# 55. Why

Eso conceptualmente puede significar:

```text
cross-tenant transfer
```

no una actualización ordinaria.

---

# 56. Tenant transfer

Deberá ser un workflow explícito.

Ejemplo:

```text
Authorize Transfer
      ↓
Validate Relationships
      ↓
Validate Destination
      ↓
Move Data
      ↓
Update Ownership
      ↓
Invalidate Cache
      ↓
Audit
```

---

# 57. Relationship isolation

Las relaciones deberán preservar tenant compatibility.

---

# 58. Tenant-local relationship

Ejemplo:

```text
Tenant A Order
      ↓
Tenant A Customer
```

válido.

---

# 59. Cross-tenant relationship

```text
Tenant A Order
      ↓
Tenant B Customer
```

rechazado por default.

---

# 60. Global relationship

```text
Tenant A Order
      ↓
Global Currency
```

puede permitirse.

---

# 61. Relationship metadata

Podrá clasificar:

```text
TENANT_LOCAL
TENANT_TO_GLOBAL
GLOBAL_TO_TENANT
CROSS_TENANT_PRIVILEGED
```

---

# 62. Foreign keys

Una FK física no demuestra tenant compatibility.

Ejemplo:

```text
orders.customer_id
→
customers.id
```

podría conectar incorrectamente tenants si los IDs no son globalmente únicos.

---

# 63. Composite FK option

En shared schema puede existir:

```text
(tenant_id, customer_id)
```

como frontera adicional.

VoltStack deberá poder modelarlo cuando la plataforma/esquema lo utilice.

---

# 64. Relationship loading

Lazy, eager y batch loading deberán conservar el TenantContext del owner.

---

# 65. Lazy loading

No deberá hacer:

```text
currentTenant()
```

desde un global mutable en el momento del lazy load.

Debe usar contexto asociado seguro.

---

# 66. Detached entities

Una entidad detached no podrá recuperar automáticamente un manager global para resolver relaciones tenant-scoped.

---

# 67. Hydration isolation

Hydration deberá validar que el resultado pertenece al dominio esperado cuando la estrategia lo requiera.

---

# 68. Shared-schema hydration

Si una query tenant-scoped devuelve inesperadamente una fila con:

```text
tenant_id = B
```

dentro de Context A:

```text
ISOLATION VIOLATION
```

---

# 69. Defense-in-depth hydration validation

Podrá ser configurable según:

```text
performance
risk
environment
mapping
```

pero debe existir como capability.

---

# 70. Connection isolation

Reutilizará las reglas del documento 262.

---

# 71. Connection target

Debe corresponder al:

```text
TenantPersistenceDomain
```

esperado.

---

# 72. Connection lease metadata

Un lease tenant-aware podrá registrar:

```text
TenantId
PersistenceDomain
PlacementGeneration
SessionProfile
```

---

# 73. Lease mismatch

Si:

```text
Lease Tenant Domain
≠
Operation Tenant Domain
```

la ejecución deberá rechazarse.

---

# 74. Shared connection pool

Dos tenants pueden compartir pool si son físicamente compatibles.

Esto no significa que compartan un lease activo.

---

# 75. Physical reuse

```text
Tenant A
   ↓
Connection C
   ↓
release + verified reset
   ↓
Tenant B
   ↓
Connection C
```

puede ser seguro.

---

# 76. Unverified reset

```text
reset = UNKNOWN
```

implica:

```text
discard connection
```

---

# 77. Session state isolation

Incluye potencialmente:

```text
current schema
database
role
session variables
temporary objects
advisory locks
transaction state
```

---

# 78. Connection state leak

Se considera:

```text
TenantIsolationViolation
```

aunque no se haya observado aún una fuga de datos.

---

# 79. Transaction isolation

Cada transaction tenant-aware tendrá afinidad con un dominio.

---

# 80. TransactionIsolationDomain

Conceptualmente:

```php
final readonly class TenantTransactionDomain
{
    public function __construct(
        public TenantId $tenant,
        public TenantPersistenceDomain $persistenceDomain,
        public TenantPlacementGeneration $placementGeneration,
    ) {}
}
```

---

# 81. Begin

Al ejecutar:

```text
BEGIN
```

el dominio queda fijado.

---

# 82. Tenant switching

Dentro de esa transaction:

```text
Tenant A → Tenant B
```

será rechazado.

---

# 83. Placement switching

Igualmente:

```text
PlacementGeneration 5
→
6
```

no cambiará una transaction ya iniciada.

---

# 84. Nested transactions

Nested transaction/savepoint conservará el mismo TenantTransactionDomain.

---

# 85. REQUIRES_NEW

Si VoltStack soporta una nueva transaction independiente, deberá adquirir un contexto compatible explícitamente.

---

# 86. Cross-tenant REQUIRES_NEW

Sólo en operaciones administrativas explícitas y con scopes separados.

---

# 87. Cross-tenant ACID

VoltStack no afirmará:

```text
Transaction A
+
Transaction B
=
Atomic Global Transaction
```

---

# 88. Failure semantics

Si A commit y B falla:

```text
PARTIAL
```

no:

```text
ROLLED_BACK
```

salvo protocolo distribuido real.

---

# 89. Cache isolation

Todos los caches cuyo valor dependa del tenant deberán incorporar el tenant domain en su identidad.

---

# 90. Query cache

Clave conceptual:

```text
QuerySemanticFingerprint
+
TenantPersistenceDomain
+
RelevantGenerations
```

---

# 91. Result cache

Igual.

---

# 92. Entity cache

Ejemplo:

```text
Tenant A / User / 10
```

no puede colisionar con:

```text
Tenant B / User / 10
```

---

# 93. Metadata cache

Metadata estructural tenant-independent puede compartirse.

---

# 94. Hydration cache

Planes/accessors podrán compartirse si no contienen estado tenant mutable.

---

# 95. Compiled query cache

Puede compartirse cuando la query compilada sea parametrizada y semánticamente compatible.

---

# 96. Cache sharing ≠ result sharing

Importante:

```text
Compiled Plan Sharing
```

puede ser seguro donde:

```text
Result Sharing
```

no lo es.

---

# 97. Cache invalidation

Un write de Tenant A deberá invalidar:

```text
Tenant A affected cache
```

sin destruir innecesariamente cache de B.

---

# 98. Placement movement

Mover Tenant A puede requerir invalidar:

```text
connection resolution cache
result cache
entity cache
routing state
compiled metadata if generation-sensitive
```

según arquitectura.

---

# 99. Cache generation

Puede utilizarse:

```text
TenantCacheGeneration
```

para invalidaciones masivas controladas.

---

# 100. Read/write isolation

Read/write routing deberá ocurrir dentro del placement del tenant.

---

# 101. Wrong replica group

Tenant A nunca deberá usar:

```text
Tenant B Replication Group
```

aunque ambas plataformas sean compatibles.

---

# 102. Sticky isolation

Sticky state será:

```text
Tenant + PersistenceDomain + Shard
```

aware.

---

# 103. Replica lag

La lag de un replication group no se aplicará arbitrariamente a otro.

---

# 104. Failover

Failover deberá conservar:

```text
tenant placement constraints
security constraints
region constraints
```

---

# 105. Shard isolation

Cuando tenants comparten shard:

```text
Tenant Query Isolation
```

sigue siendo obligatoria.

---

# 106. Shard-per-tenant

Cuando un tenant tiene shard dedicado, wrong-shard routing sigue siendo una violación de aislamiento.

---

# 107. Multi-shard tenant

Un tenant grande puede tener:

```text
Shard A
Shard B
Shard C
```

Entonces:

```text
Tenant Isolation
```

y:

```text
Partition Routing
```

son problemas separados.

---

# 108. Shard key ≠ tenant key

Puede ser:

```text
TenantId + CustomerId
```

o cualquier estrategia declarada.

---

# 109. Cross-shard query

No deberá convertirse accidentalmente en cross-tenant query.

---

# 110. Credential isolation

Las credenciales pueden ser:

```text
shared
role-specific
tenant-specific
cluster-specific
```

---

# 111. Least privilege

Cuando sea viable:

```text
CredentialPermissions
```

deberán limitar el alcance de datos accesibles.

---

# 112. Defense in depth

Ejemplo database-per-tenant:

```text
Tenant A Credentials
→
Database A only
```

reduce impacto de errores de routing.

---

# 113. Shared credentials

Son posibles, pero aumentan dependencia de aislamiento lógico.

---

# 114. Credential mismatch

Si:

```text
CredentialDomain
≠
TargetPersistenceDomain
```

la resolución debe fallar antes de uso.

---

# 115. Credential rotation

No deberá romper tenant domain identity.

---

# 116. Security architecture

El sistema de aislamiento no sustituye Authorization.

---

# 117. Isolation vs Authorization

```text
Isolation
=
where data may be accessed
```

mientras:

```text
Authorization
=
whether actor may perform operation
```

---

# 118. TenantContext ≠ permission

Tener:

```text
TenantContext(A)
```

no significa automáticamente que el actor esté autorizado para todas las operaciones de A.

---

# 119. Cross-tenant administrative access

Debe ser:

```text
explicit
authorized
scoped
auditable
time-bounded where appropriate
```

---

# 120. Global administrator

Incluso un administrador global deberá utilizar APIs explícitas.

---

# 121. No scope by omission

Eliminar TenantContext no deberá convertir una consulta en:

```text
all tenants
```

---

# 122. Privileged scope

Podrá existir:

```text
CrossTenantDatabaseScope
```

o capability equivalente.

---

# 123. CrossTenantCapability

Ejemplo conceptual:

```php
final readonly class CrossTenantCapability
{
    public function __construct(
        public ActorId $actor,
        public CrossTenantPermission $permission,
        public AuditReason $reason,
    ) {}
}
```

---

# 124. Capability ≠ boolean

Evitar APIs como:

```php
$query->ignoreTenant(true);
```

sin contexto de seguridad.

---

# 125. Better model

Conceptualmente:

```php
$adminContext->crossTenant(
    capability: $capability,
    operation: fn () => ...
);
```

---

# 126. Raw SQL isolation

Raw SQL es una frontera crítica.

---

# 127. Shared-schema raw SQL

```sql
SELECT *
FROM orders
```

no contiene evidencia suficiente de tenant isolation.

---

# 128. Raw SQL policies

```text
FORBID
REQUIRE_EXPLICIT_TENANT_BINDING
PRIVILEGED_ONLY
TRUSTED_INTERNAL_ONLY
```

---

# 129. Database-per-tenant raw SQL

Es menos vulnerable al missing row predicate, pero sigue dependiendo de:

```text
correct connection
correct database
correct credentials
correct transaction domain
```

---

# 130. Raw SQL cannot bypass domain

Una conexión tenant-scoped seguirá asociada a su domain.

---

# 131. Bulk operations

Los documentos:

```text
203_DATABASE_BULK_INSERT_SYSTEM.md
204_DATABASE_BULK_UPDATE_SYSTEM.md
205_DATABASE_BULK_DELETE_SYSTEM.md
```

deberán respetar TenantIsolationDomain.

---

# 132. Bulk insert

Un batch tenant-scoped no deberá contener entidades de tenants incompatibles.

---

# 133. Bulk update

Un update masivo shared-schema deberá conservar tenant predicate.

---

# 134. Bulk delete

Especialmente crítico:

```text
DELETE FROM orders
```

sin tenant scope deberá rechazarse en contexto tenant normal.

---

# 135. Import isolation

`206_DATABASE_IMPORT_SYSTEM.md` deberá validar:

```text
import target tenant
source tenant metadata
cross-tenant references
```

---

# 136. Import payload

Un payload no confiable no podrá seleccionar otro tenant mediante una columna `tenant_id`.

---

# 137. Export isolation

`207_DATABASE_EXPORT_SYSTEM.md` deberá garantizar que:

```text
Export Tenant A
```

no contenga registros de B.

---

# 138. Export metadata

El artefacto exportado podrá incluir tenant identity controlada para validar restore/import.

---

# 139. Large dataset processing

`208_DATABASE_LARGE_DATASET_PROCESSING_SYSTEM.md` deberá mantener el mismo TenantIsolationDomain durante la ejecución.

---

# 140. Chunk processing

Un checkpoint deberá estar ligado al tenant/domain.

---

# 141. Lazy collection

Una Lazy Collection creada bajo Tenant A no deberá continuar bajo Tenant B si cambia el contexto externo.

---

# 142. Cursor pagination

Los cursors externos deberán estar ligados al tenant cuando corresponda.

---

# 143. Pagination cursor security

Un cursor válido criptográficamente para A no será reutilizable para B.

---

# 144. Event isolation

Eventos Database tenant-aware deberán transportar identidad suficiente.

---

# 145. Event listener

No deberá consultar:

```text
global current tenant
```

para interpretar un evento histórico/asíncrono.

---

# 146. Transaction events

`afterCommit` deberá conservar el tenant domain de la transaction que produjo el evento.

---

# 147. Persistence events

`EntityPersisted` deberá poder asociarse con el dominio tenant sin exponer datos sensibles innecesarios.

---

# 148. Telemetry isolation

Telemetry deberá correlacionar operaciones tenant-aware sin crear riesgos de cardinalidad o privacidad.

---

# 149. Metric labels

No usar TenantId automáticamente.

---

# 150. Traces

Tenant identity puede ser:

```text
omitted
hashed
sampled
classified
```

según política.

---

# 151. Logs

No deberán mezclar contexto de tenant entre requests persistentes.

---

# 152. Query profiler

Los perfiles de una request tenant A no deberán aparecer en debug context de B.

---

# 153. Debug toolbar

Debe resetear:

```text
tenant query traces
connection resolution data
ORM diagnostics
transaction diagnostics
```

al finalizar request.

---

# 154. Audit isolation

Cross-tenant operations deberán registrar:

```text
actor
source context
target tenant
operation
reason
authorization
outcome
```

---

# 155. Audit ≠ telemetry

Audit tiene requisitos de:

```text
integrity
retention
accountability
```

distintos de observabilidad.

---

# 156. Persistent runtime isolation

Este punto será obligatorio para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 157. Request-scoped state

Deberá incluir:

```text
TenantContext
TenantIsolationDomain
TenantQueryContext
TenantConnectionResolution
TenantStickyState
Tenant ORM Context
Tenant Transaction Context
```

---

# 158. Worker-shared state

Sólo:

```text
immutable metadata
bounded topology cache
placement cache
compiled plans
platform capabilities
```

cuando sean seguros.

---

# 159. Forbidden worker state

Nunca:

```text
$currentTenant
$currentTenantEntityManager
$currentTenantTransaction
$currentTenantConnection
$currentTenantQuery
```

como estado global mutable del worker.

---

# 160. FrankenPHP

Request A:

```text
Tenant A
```

al terminar deberá ejecutar reset completo.

Request B no heredará A.

---

# 161. RoadRunner

Misma regla.

---

# 162. OpenSwoole

Además de sequential isolation deberá garantizar:

```text
concurrent coroutine isolation
```

---

# 163. Coroutine example

```text
Coroutine 1
Tenant A
│
├── EntityManager A
├── Transaction A
└── Connection Lease A

Coroutine 2
Tenant B
│
├── EntityManager B
├── Transaction B
└── Connection Lease B
```

---

# 164. Shared immutable infrastructure

Ambas pueden compartir:

```text
metadata
compiler
platform
topology cache
```

si son immutable/thread-safe/coroutine-safe.

---

# 165. State reset

Al cerrar un TenantExecutionScope:

```text
close result cursors
resolve transactions
clear ORM scope
release connections
reset sticky state
clear tenant query state
clear tenant telemetry buffers
clear tenant security scope
```

---

# 166. Reset failure

Si falla una limpieza crítica:

```text
do not reuse unsafe resource
```

---

# 167. TenantExecutionScope

Conceptualmente:

```php
interface TenantExecutionScope
{
    public function tenant(): TenantContext;

    public function isolationDomain(): TenantIsolationDomain;

    public function close(): void;
}
```

---

# 168. Scope lifecycle

```text
NEW
 ↓
ACTIVE
 ↓
CLOSING
 ↓
CLOSED
```

o:

```text
FAILED
TAINTED
```

---

# 169. Tainted scope

Si existe incertidumbre sobre:

```text
transaction
connection
ORM state
tenant switching
```

el scope puede marcarse:

```text
TAINTED
```

---

# 170. Tainted resources

No deberán regresar silenciosamente a pools compartidos.

---

# 171. Failure isolation

Un fallo de Tenant A no debería contaminar el estado lógico de Tenant B.

---

# 172. Shared infrastructure failure

Sin embargo, un fallo físico compartido puede afectar múltiples tenants.

Ejemplo:

```text
Shared DB Cluster failure
```

---

# 173. Logical isolation ≠ failure independence

Importante:

```text
Data Isolation
≠
Infrastructure Failure Isolation
```

---

# 174. Dedicated cluster

Mejora failure-domain separation, pero aumenta costo operacional.

---

# 175. Resource isolation

Shared DB implica posible:

```text
noisy neighbor
```

---

# 176. Resource governance

Integración con:

```text
249_DATABASE_RESOURCE_GOVERNANCE_SYSTEM.md
```

permitirá:

```text
tenant concurrency limits
query budgets
connection budgets
export limits
import limits
```

---

# 177. Resource isolation ≠ data isolation

Ambos problemas deben mantenerse separados.

---

# 178. Backup isolation

Backup deberá distinguir:

```text
Physical Database Backup
Tenant Logical Backup
Schema Backup
Shard Backup
Cluster Backup
```

---

# 179. Shared-schema backup

Un backup físico puede contener múltiples tenants.

Por tanto:

```text
Backup(Database)
≠
Backup(Tenant)
```

---

# 180. Tenant logical backup

Puede requerir:

```text
tenant-scoped export
relationship closure
consistency boundary
metadata
```

---

# 181. Database-per-tenant backup

Puede mapear más naturalmente a backup físico, pero deberá validar que la database pertenece al tenant esperado.

---

# 182. Restore isolation

Restaurar backup de A sobre B deberá rechazarse por default.

---

# 183. Restore target validation

```text
BackupTenant
=
TargetTenant
```

salvo workflow explícito de cloning/migration.

---

# 184. Tenant cloning

Clonar Tenant A para crear Tenant B es diferente de restore normal.

---

# 185. Clone workflow

Debe reescribir o reconstruir:

```text
tenant ownership
identifiers if required
secrets
external references
cache
audit context
```

---

# 186. Tenant deletion

La eliminación deberá respetar isolation strategy.

---

# 187. Shared-schema deletion

Puede requerir:

```text
DELETE tenant-owned rows
```

en orden seguro.

---

# 188. Schema-per-tenant deletion

Puede implicar:

```text
DROP SCHEMA
```

---

# 189. Database-per-tenant deletion

Puede implicar:

```text
DROP DATABASE
```

---

# 190. Dedicated cluster deletion

Puede involucrar infraestructura externa.

---

# 191. No automatic escalation

Eliminar una entidad `Tenant` nunca ejecutará automáticamente:

```text
DROP DATABASE
```

---

# 192. Destructive capability

Las operaciones destructivas deberán requerir capability específica.

---

# 193. Tenant mobility

VoltStack deberá permitir cambiar isolation strategy.

Ejemplo:

```text
Tenant A

Shared Database
      ↓
Dedicated Database
```

---

# 194. Mobility ≠ simple connection change

Requiere coordinar:

```text
data copy
writes
consistency
cutover
cache
connections
transactions
jobs
schema
verification
```

---

# 195. Migration state

Conceptualmente:

```text
STABLE_SOURCE
PREPARING
COPYING
CATCHING_UP
CUTOVER
VERIFYING
STABLE_TARGET
FAILED
```

---

# 196. Isolation during movement

Nunca deberá existir un período donde consultas normales puedan elegir arbitrariamente source o target.

---

# 197. Authoritative placement

En cada fase debe estar definido qué placement es authoritative para:

```text
reads
writes
```

---

# 198. Dual-write

Si se utiliza:

```text
DualWrite
```

será un protocolo explícito.

---

# 199. Dual-write ≠ isolation

Dos writes exitosos no prueban que ambos stores sean idénticos.

---

# 200. Reconciliation

Movimientos complejos pueden requerir:

```text
checksums
row counts
version comparison
change capture
reconciliation
```

---

# 201. Cutover

Después del cutover:

```text
TenantPlacementGeneration++
```

o token equivalente.

---

# 202. Stale workers

Workers con generation anterior deberán detectar incompatibilidad antes de nuevas operaciones.

---

# 203. In-flight operations

Podrán:

```text
finish on source
cancel
fail
```

según protocolo.

No migrarán silenciosamente.

---

# 204. Job isolation

Un job tenant-aware deberá transportar:

```text
TenantId
```

y opcionalmente generation constraints.

---

# 205. Long-delayed job

Si placement cambió:

```text
Job Tenant A generation 4
Current generation 7
```

el job deberá re-resolver conforme a su semántica.

---

# 206. Job ≠ connection snapshot

Nunca serializar una ConnectionResolution como garantía eterna.

---

# 207. Tenant deleted

Job deberá detectar:

```text
TenantStatus = DELETED
```

y fallar/cancelarse conforme a policy.

---

# 208. Testing isolation matrix

VoltStack deberá probar al menos:

| Scenario | A | B | Expected |
|---|---|---|---|
| Shared DB row read | Tenant A | Tenant B | isolated |
| Shared DB write | Tenant A | Tenant B | isolated |
| Schema reuse | schema_a | schema_b | reset |
| Database routing | db_a | db_b | isolated |
| ORM identity | User#1 | User#1 | distinct |
| Cache identity | User#1 | User#1 | distinct |
| Transaction | Tx A | Tenant B | reject |
| Relationship | A entity | B entity | reject |
| Raw SQL | Tenant A | unscoped | reject/policy |
| Worker reuse | A request | B request | no leakage |
| Coroutine | A | B | concurrent isolation |
| Restore | Backup A | Tenant B | reject |
| Job | A | changed placement | re-resolve |

---

# 209. Fault injection

Tests deberán introducir:

```text
connection reset failure
placement change
credential rotation
transaction failure
cache stale state
worker interruption
control-plane outage
replica failover
schema switch failure
```

---

# 210. Isolation proof tests

No basta probar happy path.

Se deberán crear pruebas cuyo objetivo sea intentar romper la frontera.

---

# 211. Adversarial tests

Ejemplos:

```text
malicious tenant_id
forged cursor
stale TenantContext
stale placement
wrong connection
wrong cache key
cross-tenant entity assignment
unsafe raw SQL
concurrent context switch
```

---

# 212. Proposed directory

```text
src/Quantum/Multitenancy/Database/Isolation/
│
├── Contract/
│   ├── TenantIsolationValidator.php
│   ├── TenantIsolationPolicy.php
│   └── TenantPersistenceDomainResolver.php
│
├── Domain/
│   ├── TenantIsolationDomain.php
│   ├── TenantPersistenceDomain.php
│   ├── TenantIsolationStrategy.php
│   └── TenantIsolationValidationResult.php
│
├── ORM/
│   ├── TenantEntityOwnershipValidator.php
│   ├── TenantIdentityDomainResolver.php
│   ├── TenantUnitOfWorkGuard.php
│   └── TenantRelationshipGuard.php
│
├── Query/
│   ├── TenantQueryIsolationValidator.php
│   └── TenantRawQueryPolicy.php
│
├── Connection/
│   ├── TenantConnectionIsolationValidator.php
│   ├── TenantSessionIsolationValidator.php
│   └── TenantConnectionContaminationDetector.php
│
├── Transaction/
│   ├── TenantTransactionDomain.php
│   └── TenantTransactionIsolationGuard.php
│
├── Cache/
│   ├── TenantCacheDomain.php
│   └── TenantCacheIsolationValidator.php
│
├── Runtime/
│   ├── TenantExecutionScope.php
│   ├── TenantRuntimeIsolationGuard.php
│   └── TenantScopeResetter.php
│
├── Security/
│   ├── CrossTenantCapability.php
│   └── CrossTenantOperationGuard.php
│
├── Diagnostics/
│   └── TenantIsolationDiagnostics.php
│
└── Exception/
    ├── TenantIsolationException.php
    ├── TenantIsolationViolationException.php
    ├── TenantDomainMismatchException.php
    ├── TenantEntityOwnershipException.php
    ├── CrossTenantRelationshipException.php
    ├── CrossTenantTransactionException.php
    ├── TenantConnectionContaminationException.php
    └── UnsafeTenantRawQueryException.php
```

---

# 213. Exception hierarchy

```text
TenantIsolationException
│
├── TenantIsolationViolationException
├── TenantDomainMismatchException
├── TenantEntityOwnershipException
├── TenantRelationshipIsolationException
├── TenantConnectionIsolationException
├── TenantTransactionIsolationException
├── TenantCacheIsolationException
├── TenantRuntimeIsolationException
├── TenantRestoreIsolationException
└── CrossTenantOperationException
```

---

# 214. Diagnostics

VoltStack podrá ofrecer:

```text
php volt database:tenant:isolation <tenant>
```

Ejemplo:

```text
Tenant Isolation Diagnostics
─────────────────────────────────────

Tenant:                 acme
Strategy:               DATABASE_PER_TENANT
Status:                 ACTIVE

Persistence Domain
  Logical Database:     customer-data
  Database:             [internal]
  Schema:               default
  Shard:                shard-07
  Region:               mx
  Placement Generation: 42

ORM
  Entity Manager:       isolated
  Identity Domain:      tenant-aware
  UoW:                  clean

Connection
  Target:               valid
  Session Isolation:    valid
  Reset Policy:         strict

Transaction
  Active:               no

Cache
  Tenant Namespace:     enabled
  Isolation:            valid

Runtime
  Scope:                request
  Worker Leakage:       none detected

Overall
  Isolation Status:     VALID
```

---

# 215. Explain isolation

```text
php volt database:tenant:isolation acme --explain
```

podrá mostrar:

```text
✓ TenantContext resolved
✓ Tenant authorized
✓ Persistence domain resolved
✓ Placement generation current
✓ ORM identity domain tenant-aware
✓ Connection target compatible
✓ Session profile isolated
✓ Transaction affinity compatible
✓ Cache namespace tenant-aware
✓ Runtime scope isolated
```

---

# 216. Isolation telemetry

Eventos conceptuales:

```text
TenantIsolationValidationStarted
TenantIsolationValidated
TenantIsolationViolationDetected
TenantDomainMismatchDetected
TenantConnectionContaminationDetected
CrossTenantRelationshipRejected
CrossTenantTransactionRejected
CrossTenantOperationAuthorized
TenantIsolationScopeReset
```

---

# 217. Violation severity

Podrá clasificarse:

```text
WARNING
ERROR
CRITICAL
SECURITY_CRITICAL
```

Una posible fuga cross-tenant deberá clasificarse como:

```text
SECURITY_CRITICAL
```

por default.

---

# 218. Telemetry data minimization

No registrar datos completos de las filas implicadas.

Preferir:

```text
tenant domain fingerprints
entity type
operation type
query fingerprint
connection resolution id
```

---

# 219. Core isolation invariants

## DB-TDI-001

Tenant isolation será end-to-end.

## DB-TDI-002

Tenant isolation no será equivalente a `tenant_id`.

## DB-TDI-003

Different database no implicará aislamiento completo.

## DB-TDI-004

TenantId será independiente del persistence domain.

## DB-TDI-005

Persistence domain será independiente de live connection.

## DB-TDI-006

Isolation strategy será explícita.

## DB-TDI-007

Shared-schema será soportado.

## DB-TDI-008

Separate-schema será soportado.

## DB-TDI-009

Database-per-tenant será soportado.

## DB-TDI-010

Shard-per-tenant será soportado.

## DB-TDI-011

Dedicated cluster será soportado.

## DB-TDI-012

Hybrid isolation será soportado.

## DB-TDI-013

UNKNOWN isolation no será SAFE.

## DB-TDI-014

Tenant-scoped UNKNOWN isolation fallará cerrado.

## DB-TDI-015

Query isolation será semántica.

## DB-TDI-016

Tenant predicates no se añadirán mediante SQL string patching.

## DB-TDI-017

Tenant scope existirá antes de optimization.

## DB-TDI-018

Tenant scope existirá antes de compilation.

## DB-TDI-019

Database-per-tenant no requerirá artificialmente tenant predicate.

## DB-TDI-020

Defense-in-depth tenant keys podrán soportarse.

## DB-TDI-021

IdentityMap será tenant-domain aware.

## DB-TDI-022

Mismo entity ID en tenants distintos será identidad distinta.

## DB-TDI-023

EntityManager será tenant-affine cuando esté tenant-scoped.

## DB-TDI-024

EntityManager con estado A no podrá convertirse en B.

## DB-TDI-025

UnitOfWork no mezclará dominios incompatibles.

## DB-TDI-026

ChangeSets conservarán domain identity suficiente.

## DB-TDI-027

Persistence validará entity ownership.

## DB-TDI-028

New entity tenant ownership vendrá de trusted context.

## DB-TDI-029

Tenant key no será mass-assignable por default.

## DB-TDI-030

Tenant key será immutable tras persistence por default.

## DB-TDI-031

Tenant transfer será workflow explícito.

## DB-TDI-032

Tenant-local relationships serán default.

## DB-TDI-033

Cross-tenant relationships serán rechazadas por default.

## DB-TDI-034

Tenant-to-global relationships podrán declararse.

## DB-TDI-035

Physical FK no probará tenant compatibility.

## DB-TDI-036

Lazy loading conservará tenant domain.

## DB-TDI-037

Eager loading conservará tenant domain.

## DB-TDI-038

Batch loading conservará tenant domain.

## DB-TDI-039

Detached entity no resolverá global tenant manager implícitamente.

## DB-TDI-040

Hydration podrá validar tenant ownership.

## DB-TDI-041

Unexpected row tenant será isolation violation.

## DB-TDI-042

Connection resolution deberá coincidir con persistence domain.

## DB-TDI-043

Lease mismatch será rechazado.

## DB-TDI-044

Shared pool no implicará shared lease.

## DB-TDI-045

Connection reuse requerirá verified reset.

## DB-TDI-046

UNKNOWN reset causará discard.

## DB-TDI-047

Residual tenant session state será isolation violation.

## DB-TDI-048

Transaction tendrá tenant domain fijo.

## DB-TDI-049

Transaction no cambiará de tenant.

## DB-TDI-050

Transaction no cambiará de placement silenciosamente.

## DB-TDI-051

Nested transaction conservará tenant domain.

## DB-TDI-052

Cross-tenant transactions no fingirán ACID global.

## DB-TDI-053

Partial distributed outcome permanecerá partial/unknown según evidencia.

## DB-TDI-054

Query cache será tenant-aware cuando corresponda.

## DB-TDI-055

Result cache será tenant-aware.

## DB-TDI-056

Entity cache será tenant-aware.

## DB-TDI-057

Metadata cache podrá compartirse cuando sea tenant-independent.

## DB-TDI-058

Compiled query cache podrá compartirse sólo cuando sea semánticamente seguro.

## DB-TDI-059

Cache invalidation será tenant-targeted cuando sea posible.

## DB-TDI-060

Placement movement invalidará tenant-sensitive cache state.

## DB-TDI-061

Read/write routing ocurrirá dentro del tenant placement.

## DB-TDI-062

Tenant no usará replication group incompatible.

## DB-TDI-063

Sticky state será tenant-domain aware.

## DB-TDI-064

Replica lag será evaluado en su propio replication group.

## DB-TDI-065

Failover preservará tenant constraints.

## DB-TDI-066

Tenant será distinto de shard.

## DB-TDI-067

Shard key será distinto de tenant key conceptualmente.

## DB-TDI-068

Multi-shard tenant será representable.

## DB-TDI-069

Cross-shard no implicará cross-tenant.

## DB-TDI-070

Credential scope deberá ser compatible con persistence domain.

## DB-TDI-071

Dedicated credentials podrán reforzar aislamiento.

## DB-TDI-072

Shared credentials no eliminarán logical isolation requirements.

## DB-TDI-073

Authorization será distinta de isolation.

## DB-TDI-074

TenantContext no será permission.

## DB-TDI-075

Cross-tenant access será explícito.

## DB-TDI-076

Cross-tenant access será autorizado.

## DB-TDI-077

Cross-tenant access será auditable.

## DB-TDI-078

No tenant context no significará all tenants.

## DB-TDI-079

Global administration requerirá explicit scope.

## DB-TDI-080

CrossTenantCapability no será boolean bypass trivial.

## DB-TDI-081

Raw SQL no se asumirá tenant-safe.

## DB-TDI-082

Shared-schema raw SQL tendrá política estricta.

## DB-TDI-083

Database-per-tenant raw SQL seguirá validando connection domain.

## DB-TDI-084

Bulk insert respetará tenant domain.

## DB-TDI-085

Bulk update respetará tenant domain.

## DB-TDI-086

Bulk delete respetará tenant domain.

## DB-TDI-087

Import respetará tenant target.

## DB-TDI-088

Export respetará tenant source.

## DB-TDI-089

Large dataset processing preservará tenant domain.

## DB-TDI-090

Chunk checkpoints serán tenant-bound.

## DB-TDI-091

Lazy collections serán tenant-bound.

## DB-TDI-092

Pagination cursors serán tenant-bound cuando corresponda.

## DB-TDI-093

Tenant A cursor no será válido para B.

## DB-TDI-094

Events transportarán tenant identity suficiente.

## DB-TDI-095

Async listeners no dependerán de global current tenant.

## DB-TDI-096

Transaction events conservarán originating tenant domain.

## DB-TDI-097

Telemetry state será request-scoped.

## DB-TDI-098

Debug toolbar no filtrará tenant diagnostics entre requests.

## DB-TDI-099

TenantId no será metric label automático.

## DB-TDI-100

Cross-tenant audit será reforzado.

## DB-TDI-101

Persistent workers no almacenarán current tenant global.

## DB-TDI-102

FrankenPHP tendrá tenant state reset.

## DB-TDI-103

RoadRunner tendrá tenant state reset.

## DB-TDI-104

OpenSwoole tendrá coroutine tenant isolation.

## DB-TDI-105

Immutable metadata podrá compartirse entre tenants.

## DB-TDI-106

Mutable execution state no se compartirá.

## DB-TDI-107

Scope close liberará tenant resources.

## DB-TDI-108

Tainted resources no volverán silenciosamente a shared pools.

## DB-TDI-109

Tenant failure no contaminará logical state de otro tenant.

## DB-TDI-110

Logical isolation no implicará infrastructure failure independence.

## DB-TDI-111

Resource isolation será distinto de data isolation.

## DB-TDI-112

Resource governance podrá ser tenant-aware.

## DB-TDI-113

Physical database backup no será necesariamente tenant backup.

## DB-TDI-114

Shared-schema backup podrá contener múltiples tenants.

## DB-TDI-115

Tenant logical backup será explícito.

## DB-TDI-116

Restore validará tenant target.

## DB-TDI-117

Backup A no restaurará sobre B por default.

## DB-TDI-118

Tenant clone será distinto de restore.

## DB-TDI-119

Deleting Tenant entity no ejecutará DROP DATABASE.

## DB-TDI-120

Destructive tenant operations requerirán explicit capability.

## DB-TDI-121

Tenant mobility será soportada.

## DB-TDI-122

Tenant mobility no será simple connection switch.

## DB-TDI-123

Mobility tendrá authoritative placement.

## DB-TDI-124

Dual-write nunca será implícito.

## DB-TDI-125

Cutover actualizará placement generation.

## DB-TDI-126

Stale workers deberán detectar generation mismatch.

## DB-TDI-127

In-flight transactions no migrarán silenciosamente.

## DB-TDI-128

Jobs transportarán tenant identity explícita.

## DB-TDI-129

Jobs no serializarán live connections.

## DB-TDI-130

Jobs re-resolverán current placement cuando corresponda.

## DB-TDI-131

Deleted tenant jobs fallarán de forma controlada.

## DB-TDI-132

Isolation tendrá adversarial tests.

## DB-TDI-133

Isolation tendrá fault-injection tests.

## DB-TDI-134

Connection contamination tendrá tests.

## DB-TDI-135

ORM identity collision tendrá tests.

## DB-TDI-136

Cache collision tendrá tests.

## DB-TDI-137

Cross-tenant relationship tendrá tests.

## DB-TDI-138

Raw SQL bypass tendrá tests.

## DB-TDI-139

Bulk mutation isolation tendrá tests.

## DB-TDI-140

Import/export isolation tendrá tests.

## DB-TDI-141

Backup/restore isolation tendrá tests.

## DB-TDI-142

Persistent worker isolation tendrá tests.

## DB-TDI-143

Concurrent coroutine isolation tendrá tests.

## DB-TDI-144

Placement mobility tendrá tests.

## DB-TDI-145

Stale context tendrá tests.

## DB-TDI-146

Wrong credential domain tendrá tests.

## DB-TDI-147

Wrong shard routing tendrá tests.

## DB-TDI-148

Wrong replica group tendrá tests.

## DB-TDI-149

No tenant context fallará cerrado para tenant-scoped resources.

## DB-TDI-150

Isolation validation será explainable.

## DB-TDI-151

Isolation validation será observable.

## DB-TDI-152

Isolation diagnostics no expondrán secrets.

## DB-TDI-153

Security tendrá prioridad sobre convenience.

## DB-TDI-154

Isolation tendrá prioridad sobre pool reuse.

## DB-TDI-155

Isolation tendrá prioridad sobre cache hit.

## DB-TDI-156

Isolation tendrá prioridad sobre transparent failover.

## DB-TDI-157

Isolation tendrá prioridad sobre implicit tenant switching.

## DB-TDI-158

UNKNOWN nunca será promovido a SAFE.

## DB-TDI-159

Tenant isolation podrá evolucionar entre estrategias sin crear un segundo ORM.

## DB-TDI-160

Todas las APIs ORM convergerán en el mismo TenantIsolationDomain.

## DB-TDI-161

Active Record no tendrá aislamiento tenant separado.

## DB-TDI-162

Repository API no tendrá aislamiento tenant separado.

## DB-TDI-163

EntityManager será la coordinación ORM común.

## DB-TDI-164

Query Engine continuará siendo responsable de query semantics.

## DB-TDI-165

Compiler no decidirá qué tenant está activo.

## DB-TDI-166

Driver no decidirá qué tenant está activo.

## DB-TDI-167

ConnectionManager no autorizará cross-tenant access.

## DB-TDI-168

Multitenancy no romperá las fronteras arquitectónicas de Database.

## DB-TDI-169

Una optimización nunca podrá eliminar una frontera tenant requerida.

## DB-TDI-170

Una operación sólo se considerará tenant-safe cuando todas sus fronteras relevantes sean compatibles con el mismo dominio de aislamiento.

---

# 220. Modelo formal

Sea:

```text
T = TenantContext
D = TenantPersistenceDomain
Q = QueryTenantDomain
O = ORMTenantDomain
C = ConnectionTenantDomain
X = TransactionTenantDomain
K = CacheTenantDomain
R = RuntimeTenantDomain
```

Una operación tenant-scoped será válida cuando:

```text
Compatible(T,D)
∧ Compatible(T,Q)
∧ Compatible(T,O)
∧ Compatible(T,C)
∧ Compatible(T,X)
∧ Compatible(T,K)
∧ Compatible(T,R)
```

para todas las dimensiones aplicables.

---

# 221. Strong equality vs compatibility

No todas las dimensiones necesitan igualdad textual.

Por ejemplo:

```text
Tenant A
```

puede usar:

```text
Shared Pool X
```

junto con Tenant B.

Lo importante es:

```text
Compatible(
    TenantIsolationDomain,
    ResourceIsolationDomain
)
```

---

# 222. Isolation compatibility

Formalmente:

```text
IsolationSafe(Operation)
=
∀ resource ∈ Resources(Operation):
    Compatible(
        TenantIsolationDomain(Operation),
        IsolationDomain(resource)
    )
```

---

# 223. Missing evidence

Si una frontera crítica no puede probar compatibilidad:

```text
Compatible = UNKNOWN
```

entonces:

```text
IsolationSafe = false
```

para operaciones tenant-scoped normales.

---

# 224. Arquitectura final

```text
                         APPLICATION
                              │
                              ▼
                         TenantContext
                              │
                              ▼
                    Authorization Context
                              │
                              ▼
                    TenantIsolationDomain
                              │
       ┌──────────────────────┼──────────────────────┐
       │                      │                      │
       ▼                      ▼                      ▼
 Query Context            ORM Context         Connection Context
       │                      │                      │
       ▼                      ▼                      ▼
 Tenant Scope          IdentityMap / UoW      Placement / Lease
       │                      │                      │
       └──────────────────────┼──────────────────────┘
                              ▼
                     Transaction Domain
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
            Cache        Relationships       Runtime
            Domain           Domain           Scope
              │               │               │
              └───────────────┼───────────────┘
                              ▼
                    Isolation Validation
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
                  VALID             INVALID/UNKNOWN
                    │                   │
                    ▼                   ▼
                 EXECUTE              REJECT
```

---

# 225. Evolución del aislamiento

Una ventaja central será poder migrar:

```text
Shared Database
      ↓
Separate Schema
      ↓
Dedicated Database
      ↓
Dedicated Cluster
```

manteniendo estable el código de dominio.

Ejemplo:

```php
$orders = Order::query()
    ->where('status', 'pending')
    ->get();
```

No deberá saber si el tenant reside en:

```text
row
schema
database
shard
cluster
```

---

# 226. Responsabilidad del paquete Multitenancy

El paquete:

```text
VoltStack/Quantum/Multitenancy
```

aportará:

```text
TenantContext
TenantIsolationDomain
TenantPlacement
Tenant Query Integration
Tenant ORM Integration
Tenant Connection Integration
Tenant Migration Integration
```

sin duplicar:

```text
ORM
Query Engine
Compiler
Executor
ConnectionManager
Transaction Engine
```

---

# 227. Principio de seguridad final

VoltStack seguirá:

```text
RESOLVE TENANT
      ↓
AUTHORIZE
      ↓
RESOLVE ISOLATION DOMAIN
      ↓
VALIDATE QUERY DOMAIN
      ↓
VALIDATE ORM DOMAIN
      ↓
VALIDATE CONNECTION DOMAIN
      ↓
VALIDATE TRANSACTION DOMAIN
      ↓
VALIDATE CACHE DOMAIN
      ↓
VALIDATE RUNTIME DOMAIN
      ↓
EXECUTE
```

---

# 228. Regla arquitectónica definitiva

> **Un tenant no estará aislado simplemente porque sus filas tengan un `tenant_id`, porque use otro schema o porque disponga de una base de datos dedicada. VoltStack considerará el aislamiento como una propiedad end-to-end que debe conservar la misma identidad tenant a través de Query Engine, ORM, UnitOfWork, IdentityMap, relaciones, conexiones, transacciones, cache, routing, jobs y runtimes persistentes. Si cualquiera de esas fronteras no puede demostrar compatibilidad con el dominio tenant efectivo, la operación deberá considerarse insegura y fallar de forma cerrada.**

---

# 229. Estado del Bloque 26

```text
BLOCK 26 — MULTITENANCY INTEGRATION

✓ 261_DATABASE_MULTITENANCY_INTEGRATION_ARCHITECTURE.md
✓ 262_DATABASE_TENANT_CONNECTION_RESOLUTION_SYSTEM.md
✓ 263_DATABASE_TENANT_DATABASE_ISOLATION_SYSTEM.md
│
├── 264_DATABASE_TENANT_SCHEMA_ISOLATION_SYSTEM.md
├── 265_DATABASE_TENANT_QUERY_CONTEXT_SYSTEM.md
└── 266_DATABASE_TENANT_MIGRATION_INTEGRATION_SYSTEM.md
```

---

# 230. Siguiente documento

```text
264_DATABASE_TENANT_SCHEMA_ISOLATION_SYSTEM.md
```

El siguiente documento deberá profundizar específicamente en:

```text
Schema-per-Tenant Architecture
Schema Identity
Tenant Schema Resolution
Schema Registry
Schema Naming
Logical Schema vs Physical Schema
Search Path
Qualified Table Names
Connection Session Schema State
Schema Switching
Schema Reset
Schema Pooling
Schema Compatibility
Schema Generations
Schema Version Skew
Schema Introspection
Schema Metadata
Schema Cache
ORM Mapping
Query Compilation
Cross-Schema Queries
Foreign Keys
Transactions
Migrations
Backup/Restore
Tenant Mobility
Security
Identifier Injection Protection
Persistent Runtime Safety
Telemetry
Diagnostics
Testing
```

bajo la regla:

> **Un schema tenant no será tratado como una cadena que pueda intercambiarse libremente en SQL; será una identidad estructural resuelta desde TenantContext, validada contra placement y metadata, propagada semánticamente hacia Query/ORM/Connection y eliminada de toda conexión antes de que ésta pueda reutilizarse para otro tenant.**