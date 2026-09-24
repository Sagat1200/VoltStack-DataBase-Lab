# 253_DATABASE_DATABASE_CONTEXT_SYSTEM.md

# VoltStack Quantum Database
## Database Context System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 253 — Database Context System  
**Bloque:** 25 — Persistent Runtime  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `252_DATABASE_REQUEST_SCOPE_SYSTEM.md`  
**Siguiente documento:** `254_DATABASE_STATE_ISOLATION_SYSTEM.md`

---

# 1. Propósito

Este documento define el **Database Context System** de `VoltStack/Quantum/Database`.

`DatabaseContext` será la representación tipada del **contexto efectivo bajo el cual debe ejecutarse una operación de base de datos**.

La regla central será:

> **Toda operación de VoltStack Database deberá ejecutarse bajo un contexto explícito, identificable y scope-safe que determine su dominio de persistencia, tenant, shard, intención de lectura/escritura, consistencia, transacción, seguridad, telemetría, cancelación y restricciones de recursos.**

Conceptualmente:

```text
Database Operation
        │
        ▼
 DatabaseContext
        │
        ├── Persistence Domain
        ├── Tenant
        ├── Shard
        ├── Read/Write Intent
        ├── Consistency
        ├── Transaction
        ├── Security
        ├── Telemetry
        ├── Cancellation
        ├── Resource Budget
        └── Runtime
        │
        ▼
 Query / ORM / Transaction / Connection
```

`DatabaseContext` será uno de los mecanismos fundamentales para impedir que los distintos subsistemas de Database dependan de estado global implícito.

---

# 2. Problema arquitectónico

Sin un contexto formal, distintos componentes podrían intentar obtener información desde lugares diferentes:

```text
Query Builder     → global config
ORM               → current tenant static
ConnectionManager → thread-local writer flag
Transaction       → singleton transaction
Telemetry         → current request global
Security          → HTTP request directly
```

Esto produciría:

```text
hidden dependencies
+
state leakage
+
runtime coupling
+
difficult testing
+
persistent-worker bugs
```

VoltStack deberá reemplazar ese modelo por:

```text
Execution Scope
      │
      ▼
DatabaseContext
      │
      ▼
Explicit Effective Context
```

---

# 3. DatabaseContext ≠ Configuration

Distinción fundamental:

```text
Database Configuration
≠
Database Context
```

Configuration describe:

```text
what can exist
```

Context describe:

```text
what applies now
```

Ejemplo:

```text
Configuration:
    writer = db-primary
    replicas = [db-r1, db-r2]
    consistency policies available
    tenant strategy = database-per-tenant

Context:
    tenant = acme
    intent = READ
    consistency = READ_YOUR_WRITES
    stickyWriter = true
```

---

# 4. DatabaseContext ≠ Request Scope

También:

```text
DatabaseExecutionScope
≠
DatabaseContext
```

El Scope representa:

```text
lifecycle
ownership
isolation boundary
```

El Context representa:

```text
effective database execution semantics
```

Relación:

```text
DatabaseExecutionScope
        │
        owns
        ▼
DatabaseContext
```

---

# 5. DatabaseContext ≠ QueryContext

`QueryContext` definido anteriormente pertenece específicamente al procesamiento de queries.

Por tanto:

```text
DatabaseContext
    ↓
QueryContext
```

pero:

```text
DatabaseContext
≠
QueryContext
```

---

# 6. DatabaseContext ≠ TransactionContext

Una transacción forma parte del contexto efectivo cuando existe.

Pero:

```text
DatabaseContext
≠
TransactionContext
```

---

# 7. DatabaseContext ≠ SecurityContext

Security será una dimensión contextual.

No será el contexto completo.

---

# 8. Objetivos

El sistema deberá proporcionar:

1. contexto explícito;
2. tipado fuerte;
3. composición controlada;
4. aislamiento por execution scope;
5. propagación segura;
6. derivación de contextos;
7. inmutabilidad donde sea posible;
8. validación de combinaciones;
9. integración con routing;
10. integración con transactions;
11. integración con ORM;
12. integración con tenant/shard;
13. integración con security;
14. integración con telemetry;
15. integración con cancellation;
16. resource governance;
17. runtime awareness;
18. diagnóstico;
19. fingerprinting;
20. persistent-runtime safety.

---

# 9. No objetivos

`DatabaseContext` no deberá:

```text
execute SQL
compile SQL
open connections
commit transactions
hydrate entities
authorize users by itself
resolve tenant business rules
implement telemetry exporters
```

Es un modelo contextual, no un motor de ejecución.

---

# 10. Principio de contexto explícito

El objetivo conceptual es transformar:

```php
Database::setTenant($tenant);
Database::useWriter();
Database::setTimeout(5000);

$result = User::all();
```

en una arquitectura interna equivalente a:

```text
Operation
   ↓
Current DatabaseExecutionScope
   ↓
DatabaseContext
   ↓
Effective Context
   ↓
Execution
```

La API pública puede seguir siendo ergonómica.

La arquitectura interna no dependerá de globals.

---

# 11. Modelo general

Se propone conceptualmente:

```php
final readonly class DatabaseContext
{
    public function __construct(
        public DatabaseScopeId $scopeId,
        public PersistenceDomainContext $persistence,
        public TenantContext $tenant,
        public ShardContext $shard,
        public AccessIntentContext $access,
        public ConsistencyContext $consistency,
        public TransactionBindingContext $transaction,
        public DatabaseSecurityContext $security,
        public DatabaseTelemetryContext $telemetry,
        public CancellationContext $cancellation,
        public ResourceBudgetContext $resources,
        public RuntimeContext $runtime,
    ) {}
}
```

La implementación final podrá modularizar los componentes opcionales.

---

# 12. Context Root

El `DatabaseExecutionScope` tendrá un:

```text
RootDatabaseContext
```

que representa el contexto base de la ejecución.

---

# 13. Context derivation

Operaciones específicas podrán derivar contextos:

```text
Root Context
    │
    ├── Query Context
    ├── Transaction Context
    ├── Chunk Context
    ├── Import Context
    └── Bulk Operation Context
```

---

# 14. Context derivation ≠ mutation

Regla:

```text
derive(parent, override)
→
child context
```

No:

```text
mutate(parent)
```

---

# 15. Ejemplo

```php
$readContext = $context->derive(
    access: AccessIntent::READ,
    consistency: ConsistencyRequirement::READ_YOUR_WRITES,
);
```

El contexto original permanece sin cambios.

---

# 16. Inmutabilidad

La preferencia será:

```text
immutable context values
```

especialmente para:

```text
tenant
shard
domain
security identity
runtime identity
```

---

# 17. Context state vs context values

Algunas dimensiones pueden depender de estado mutable.

Ejemplo:

```text
sticky writer activated after write
```

Ese estado no deberá convertir `DatabaseContext` en un God Object mutable.

---

# 18. Context handles

Podrán utilizarse referencias controladas:

```text
DatabaseContext
   │
   └── RoutingStateHandle
```

cuando una dimensión requiera estado scoped mutable.

---

# 19. Value vs State

Distinción:

```text
Context Value
=
immutable semantic fact

Context State
=
scoped mutable execution state
```

Ejemplo:

```text
TenantId
=
Context Value

StickyWriterState
=
Context State
```

---

# 20. Persistence Domain

El contexto deberá identificar el:

```text
PersistenceDomain
```

---

# 21. Persistence Domain definition

Un dominio de persistencia representa una frontera lógica dentro de la cual:

```text
entity identity
transaction affinity
routing
cache identity
metadata assumptions
```

tienen significado coherente.

---

# 22. PersistenceDomainId

```php
final readonly class PersistenceDomainId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 23. Persistence Domain ≠ Database name

No necesariamente:

```text
PersistenceDomain
=
physical database name
```

Puede representar:

```text
logical application database
tenant database family
shard topology
service database
```

---

# 24. Domain identity

El dominio deberá formar parte de identidades donde sea relevante:

```text
Entity Identity
Cache Key
Cursor Binding
Checkpoint Binding
Query Context
Transaction Context
```

---

# 25. Tenant Context

Se define:

```text
TenantContext
```

como la identidad efectiva del tenant para Database.

---

# 26. Tenant states

Conceptualmente:

```php
enum TenantContextState
{
    case NOT_APPLICABLE;
    case RESOLVED;
    case SYSTEM;
}
```

Evitar:

```text
null
```

con significados ambiguos.

---

# 27. Tenant unresolved

Si una operación requiere tenant y no está resuelto:

```text
TenantRequiredException
```

No se seleccionará un tenant arbitrario.

---

# 28. TenantContext example

```php
final readonly class TenantContext
{
    public function __construct(
        public TenantContextState $state,
        public ?TenantId $tenantId,
    ) {}
}
```

---

# 29. TenantContext ≠ Authentication

Un tenant puede derivarse de autenticación, hostname, job payload u otra fuente.

Database sólo recibe el contexto efectivo.

---

# 30. TenantContext ≠ Authorization

Conocer:

```text
tenant = acme
```

no demuestra que el actor esté autorizado para acceder a Acme.

---

# 31. Shard Context

El contexto podrá contener:

```text
ShardContext
```

---

# 32. Shard states

```php
enum ShardContextState
{
    case NOT_SHARDED;
    case UNRESOLVED;
    case RESOLVED;
    case MULTI_SHARD;
}
```

---

# 33. UNKNOWN ≠ all shards

Regla crítica:

```text
UNRESOLVED
≠
MULTI_SHARD
```

y:

```text
UNKNOWN
≠
GLOBAL
```

---

# 34. ShardContext

Conceptualmente:

```php
final readonly class ShardContext
{
    public function __construct(
        public ShardContextState $state,
        public ?ShardId $shard,
        public ?ShardMapGeneration $generation,
    ) {}
}
```

---

# 35. Shard map generation

Será relevante para:

```text
cursor pagination
chunk checkpoints
distributed queries
routing
failover
```

---

# 36. Access Intent

El contexto deberá expresar qué tipo de acceso solicita una operación.

---

# 37. AccessIntent

```php
enum DatabaseAccessIntent
{
    case READ;
    case WRITE;
    case LOCKING_READ;
    case SCHEMA_READ;
    case SCHEMA_WRITE;
    case ADMINISTRATIVE;
}
```

---

# 38. READ ≠ replica

Regla:

```text
READ
≠
must use replica
```

READ sólo expresa intención semántica.

Routing decide endpoint.

---

# 39. WRITE

Una operación WRITE requerirá autoridad compatible.

Normalmente:

```text
WRITE
→
writer
```

---

# 40. LOCKING_READ

Una lectura con:

```text
FOR UPDATE
```

conceptualmente será:

```text
LOCKING_READ
```

y no una lectura ordinaria.

---

# 41. Schema intent

Migraciones/introspection podrán utilizar intents separados para evitar mezclar tráfico ordinario con operaciones estructurales.

---

# 42. Consistency Context

El contexto deberá expresar requisitos de consistencia.

---

# 43. Consistency levels

Conceptualmente:

```php
enum DatabaseConsistencyRequirement
{
    case EVENTUAL;
    case BEST_EFFORT;
    case MONOTONIC;
    case READ_YOUR_WRITES;
    case SNAPSHOT;
    case STRONG;
}
```

No todos los modos serán implementables en todos los topologies.

---

# 44. Requested vs Effective Consistency

Distinción:

```text
RequestedConsistency
≠
EffectiveConsistency
```

---

# 45. No silent downgrade

Si se solicita:

```text
STRONG
```

y sólo puede ofrecerse:

```text
EVENTUAL
```

VoltStack no deberá degradarlo silenciosamente.

---

# 46. Consistency resolution

```text
Requested Requirement
        +
Transaction State
        +
Routing State
        +
Topology
        +
Platform Capabilities
        ↓
Effective Consistency Plan
```

---

# 47. Read-your-writes

Puede requerir:

```text
writer routing
```

o:

```text
replica >= minimum observed position
```

---

# 48. Minimum observed position

El contexto podrá incluir:

```text
ReplicationPosition
```

o abstracción equivalente.

---

# 49. Sticky writer state

Después de una escritura:

```text
RoutingState
    ↓
sticky writer
```

puede activarse dentro del execution scope.

---

# 50. Sticky ≠ Context mutation

El valor base del contexto puede permanecer inmutable mientras un state handle scoped registra la afinidad.

---

# 51. Transaction Binding

DatabaseContext deberá saber si la operación:

```text
is outside transaction
```

o está ligada a una transaction existente.

---

# 52. Transaction binding model

```php
enum TransactionBindingState
{
    case NONE;
    case ACTIVE;
    case SUSPENDED;
    case UNKNOWN;
}
```

---

# 53. Transaction ID

Cuando exista:

```text
TransactionId
```

formará parte del binding.

---

# 54. Transaction affinity

El contexto transaccional puede fijar:

```text
connection
writer
shard
persistence domain
```

---

# 55. Context override restrictions

Dentro de una transaction:

```text
transaction shard = shard-01
```

una operación hija no podrá simplemente derivar:

```text
shard = shard-02
```

---

# 56. Invalid derivation

Resultado:

```text
DatabaseContextConflictException
```

---

# 57. Transaction context precedence

Las restricciones transaccionales deberán tener precedencia sobre preferencias débiles de routing.

---

# 58. Security Context

Database tendrá una representación contextual de seguridad.

---

# 59. DatabaseSecurityContext

Podrá incluir:

```text
actor reference
authorization scope reference
data access policy
sensitive-data policy
audit correlation
privilege mode
```

---

# 60. Security context ≠ User object

No deberá ser necesario acoplar Database al modelo HTTP de usuario.

---

# 61. ActorReference

Ejemplo:

```php
final readonly class DatabaseActorReference
{
    public function __construct(
        public ActorType $type,
        public string $id,
    ) {}
}
```

---

# 62. Actor types

Podrán incluir:

```text
USER
SERVICE
JOB
SYSTEM
CLI
ANONYMOUS
```

---

# 63. System context

`SYSTEM` no deberá equivaler automáticamente a:

```text
unlimited database access
```

---

# 64. Privilege elevation

Cualquier elevación deberá ser:

```text
explicit
bounded
auditable
```

---

# 65. Security scope derivation

Un child context podrá reducir privilegios.

Por defecto no deberá poder aumentarlos arbitrariamente.

---

# 66. Monotonic restriction principle

Para dimensiones restrictivas:

```text
ChildPrivileges
⊆
ParentPrivileges
```

salvo mecanismo explícito autorizado.

---

# 67. Telemetry Context

Cada operación Database deberá poder correlacionarse con telemetry.

---

# 68. TelemetryContext

Puede contener:

```text
TraceId
SpanId
RequestCorrelationId
JobCorrelationId
DatabaseScopeId
OperationId
sampling policy
```

---

# 69. Telemetry context ≠ exporter

El contexto sólo transporta correlación/configuración efectiva.

No envía spans.

---

# 70. Telemetry child

Cada query podrá derivar:

```text
QuerySpan
```

desde el contexto de la operación.

---

# 71. Sensitive telemetry

El contexto deberá indicar políticas para:

```text
SQL visibility
parameter redaction
PII masking
query fingerprinting
```

cuando corresponda.

---

# 72. Cancellation Context

El contexto deberá incluir cancelación.

---

# 73. Cancellation tree

```text
Execution Cancellation
        │
        ├── Transaction Operation
        │
        ├── Query
        │
        └── Export
```

---

# 74. Parent cancellation

Si el root está cancelado:

```text
new child DB operation
```

deberá fallar antes de adquirir recursos cuando sea posible.

---

# 75. Cancellation ≠ Timeout

Distinción:

```text
Cancellation
≠
Timeout
≠
Deadline
```

---

# 76. Deadline Context

El contexto podrá contener:

```text
Deadline
```

absoluto.

---

# 77. Timeout derivation

Una query puede calcular:

```text
effective timeout
=
min(
    query timeout,
    operation deadline remaining,
    execution deadline remaining
)
```

---

# 78. Deadline monotonicity

Un child context no deberá extender un deadline heredado por default.

---

# 79. Resource Budget Context

El sistema de Resource Governance podrá proporcionar:

```text
ResourceBudgetContext
```

---

# 80. Resource dimensions

Ejemplos:

```text
max query duration
max rows
max memory
max open cursors
max connection leases
max concurrent operations
max result bytes
```

---

# 81. Budget inheritance

Un child context podrá consumir o restringir presupuesto.

No deberá crear recursos infinitos derivando un nuevo contexto.

---

# 82. Budget reservation

Podrá existir:

```text
ResourcePermit
```

obtenido desde el budget.

---

# 83. Context ≠ Permit

El contexto expresa política/presupuesto.

El permit representa consumo autorizado concreto.

---

# 84. Runtime Context

DatabaseContext podrá conocer características relevantes del runtime.

---

# 85. RuntimeContext

Ejemplo:

```php
final readonly class DatabaseRuntimeContext
{
    public function __construct(
        public RuntimeType $runtime,
        public WorkerId $worker,
        public ExecutionType $executionType,
        public bool $persistent,
        public ConcurrencyModel $concurrency,
    ) {}
}
```

---

# 86. Runtime types

```text
TRADITIONAL_PHP
FRANKENPHP
ROADRUNNER
OPENSWOOLE
TEST
CUSTOM
```

---

# 87. RuntimeContext ≠ vendor conditionals everywhere

Los componentes no deberán hacer:

```php
if ($runtime === 'frankenphp') {
    ...
}
```

por todo Database.

---

# 88. Runtime capabilities

Cuando sea necesario se utilizarán:

```text
RuntimeCapabilities
```

---

# 89. Capability example

```text
supportsRuntimeLocalStorage()
supportsConcurrentExecutions()
supportsWorkerRecycling()
supportsCancellationPropagation()
```

---

# 90. Version ≠ Capability

Igual que en Platform:

```text
Runtime Version
≠
Runtime Capability
```

---

# 91. Context composition

El contexto final se obtendrá mediante composición.

Conceptualmente:

```text
Framework defaults
       ↓
Application policy
       ↓
Execution context
       ↓
Tenant context
       ↓
Transaction constraints
       ↓
Operation requirements
       ↓
Effective DatabaseContext
```

---

# 92. Context precedence

No todas las capas tendrán la misma autoridad.

---

# 93. Strong constraints

Ejemplos:

```text
transaction shard affinity
security restrictions
tenant identity
deadline upper bound
```

no podrán ser anulados por preferencias débiles.

---

# 94. Preference vs Constraint

Se distinguirá:

```text
Preference
Constraint
Requirement
```

---

# 95. Preference

Ejemplo:

```text
prefer replica
```

---

# 96. Requirement

Ejemplo:

```text
must read own writes
```

---

# 97. Constraint

Ejemplo:

```text
must remain on shard-04
```

---

# 98. Context Resolver

Se propone:

```text
DatabaseContextResolver
```

---

# 99. Contract

```php
interface DatabaseContextResolver
{
    public function current(): DatabaseContext;

    public function hasContext(): bool;
}
```

---

# 100. Resolver flow

```text
DatabaseContextResolver
        ↓
DatabaseScopeResolver
        ↓
Current DatabaseExecutionScope
        ↓
DatabaseContext
```

---

# 101. No fallback global context

Fuera de execution scope:

```text
current()
```

deberá fallar.

---

# 102. Explicit contexts

Low-level APIs podrán aceptar explícitamente:

```php
$executor->execute(
    $plan,
    $context
);
```

Esto facilita testing y desacoplamiento.

---

# 103. Resolver at boundaries

Idealmente:

```text
Facade / Application Boundary
            ↓
       Context Resolver
            ↓
Explicit Context passed internally
```

---

# 104. Evitar Service Locator profundo

Los componentes internos no deberán llamar repetidamente:

```text
DatabaseContext::current()
```

como global implícito.

---

# 105. Query flow

```text
Developer API
     ↓
Query Builder
     ↓
Query Model
     ↓
Query Planner
     ↓
DatabaseContext
     ↓
Routing
     ↓
Executor
```

---

# 106. QueryContext derivation

El Query Engine podrá derivar:

```text
QueryContext
```

desde:

```text
DatabaseContext
+
Query Model
+
Operation Requirements
```

---

# 107. ORM flow

```text
Model API / Repository
        ↓
EntityManager
        ↓
DatabaseContext
        ↓
Query/Persistence Engine
```

---

# 108. Entity identity context

Para IdentityMap:

```text
EntityIdentity
=
EntityType
+
Identifier
+
PersistenceDomain
+
Tenant/Shard dimensions when required
```

---

# 109. Entity identity ≠ raw primary key

Dos filas:

```text
tenant A / users / id 10
tenant B / users / id 10
```

no son necesariamente la misma entidad.

---

# 110. Persistence flow

```text
UnitOfWork
    ↓
PersistencePlanner
    ↓
DatabaseContext
    ↓
Write routing
    ↓
Transaction
    ↓
Executor
```

---

# 111. Schema flow

Schema operations podrán derivar un contexto específico:

```text
SchemaContext
```

con intents:

```text
SCHEMA_READ
SCHEMA_WRITE
```

---

# 112. Migration context

Migration tendrá restricciones adicionales:

```text
migration mode
maintenance policy
DDL transaction capability
zero-downtime policy
```

sin sobrecargar el contexto root con todos esos detalles.

---

# 113. Specialized Context

Regla:

```text
DatabaseContext
=
common cross-cutting context

SpecializedOperationContext
=
operation-specific semantics
```

---

# 114. No God Context

No se añadirán cientos de campos específicos de:

```text
pagination
migration
hydration
seeding
backup
```

al root.

---

# 115. Context extensions

Podrá existir un mecanismo tipado de extensiones.

---

# 116. Extension example

```php
$context->extensions()->get(
    TemporalQueryContext::class
);
```

pero deberá evitarse convertirlo en:

```text
untyped associative bag
```

---

# 117. Typed context keys

Si se usa registry:

```php
ContextKey<T>
```

deberá mantener type safety.

---

# 118. Context Extension Registry

Las extensiones podrán registrarse durante bootstrap.

Después:

```text
freeze()
```

en persistent runtime.

---

# 119. Context fingerprint

Algunas operaciones necesitan identificar semánticamente el contexto.

Se propone:

```text
DatabaseContextFingerprint
```

---

# 120. Fingerprint use cases

```text
cache key
cursor binding
chunk checkpoint
audit correlation
diagnostics
compiled plan compatibility
```

---

# 121. Fingerprint ≠ serialization

No será necesario serializar el objeto completo.

---

# 122. Fingerprint dimensions

Sólo incluirá dimensiones relevantes al consumidor.

Ejemplo:

```text
QueryCacheContextFingerprint
```

puede incluir:

```text
persistence domain
tenant
shard
security data scope
consistency-sensitive dimensions
```

---

# 123. Context fingerprint profiles

No habrá necesariamente un único hash universal.

Podrán existir:

```text
CacheContextFingerprint
CursorContextFingerprint
CheckpointContextFingerprint
AuditContextFingerprint
```

---

# 124. Why profiles

Porque:

```text
Telemetry SpanId
```

no debe invalidar Query Cache.

---

# 125. Context equality

Dos contextos no deberán compararse simplemente mediante:

```php
$contextA == $contextB
```

para decisiones semánticas complejas.

---

# 126. Compatibility

Se utilizará:

```text
ContextCompatibilityChecker
```

cuando un recurso necesite verificar reutilización.

---

# 127. Example

Una prepared statement podría requerir compatibilidad en:

```text
platform
dialect
schema generation
parameter types
```

pero no necesariamente en:

```text
TraceId
```

---

# 128. Context compatibility ≠ identity

Regla:

```text
Context A ≠ Context B
```

pueden ser compatibles para una operación concreta.

---

# 129. Context validation

Antes de ejecutar operaciones críticas:

```text
DatabaseContextValidator
```

podrá verificar coherencia.

---

# 130. Validation examples

Invalid:

```text
access = WRITE
+
replicaOnly = true
```

Invalid:

```text
transaction shard = A
+
operation shard = B
```

Invalid:

```text
tenant required
+
tenant unresolved
```

Invalid:

```text
consistency = STRONG
+
topology cannot satisfy it
```

---

# 131. Validation stages

Podrán existir:

```text
STRUCTURAL
SEMANTIC
EXECUTION
```

---

# 132. Structural validation

Verifica que el objeto esté bien formado.

---

# 133. Semantic validation

Verifica compatibilidad entre dimensiones.

---

# 134. Execution validation

Verifica capacidades/topología/estado actual.

---

# 135. Context normalization

El sistema podrá normalizar valores equivalentes.

Ejemplo:

```text
READ + transaction active
```

puede derivar routing efectivo al writer.

Pero deberá conservar:

```text
requested intent
```

separado del:

```text
effective plan
```

---

# 136. Requested ≠ Effective

Principio general:

```text
Requested Context
≠
Effective Execution Decision
```

---

# 137. Context resolution pipeline

```text
Raw Requirements
      ↓
Context Composition
      ↓
Structural Validation
      ↓
Semantic Validation
      ↓
Capability Resolution
      ↓
Effective Context
      ↓
Execution Plan
```

---

# 138. Context conflict

Se propone:

```text
DatabaseContextConflict
```

con información estructurada.

---

# 139. Conflict example

```text
Dimension:
    shard

Parent:
    shard-01

Requested:
    shard-02

Reason:
    active transaction affinity
```

---

# 140. Error hierarchy

```text
DatabaseContextException
├── DatabaseContextNotAvailableException
├── DatabaseContextClosedException
├── DatabaseContextConflictException
├── DatabaseContextValidationException
├── PersistenceDomainMismatchException
├── TenantContextException
│   ├── TenantRequiredException
│   └── TenantMismatchException
├── ShardContextException
│   ├── ShardUnresolvedException
│   └── ShardAffinityException
├── ConsistencyRequirementException
├── TransactionContextConflictException
├── DatabaseSecurityContextException
├── DatabaseDeadlineExceededException
├── DatabaseResourceBudgetException
└── StaleDatabaseContextException
```

---

# 141. Context lifetime

El root context vive como máximo:

```text
DatabaseExecutionScope lifetime
```

---

# 142. Child context lifetime

Un child context no deberá sobrevivir a su parent scope.

---

# 143. Scope token

Todo contexto podrá contener o asociarse a:

```text
ScopeToken
```

---

# 144. Stale context detection

Si:

```text
context.scope = A
current scope = B
```

y se intenta usar como contextual resource:

```text
StaleDatabaseContextException
```

o cross-scope error.

---

# 145. Context sharing

Un `DatabaseContext` completamente inmutable puede técnicamente ser pasado entre funciones.

Pero su semántica sigue ligada a su scope cuando contiene referencias scoped.

---

# 146. Detached Context Descriptor

Para handoff se utilizará:

```text
DatabaseContextDescriptor
```

no el contexto vivo.

---

# 147. Descriptor contents

Puede contener:

```text
tenant id
persistence domain id
security actor reference
requested consistency
trace correlation
```

según política.

---

# 148. Descriptor forbidden contents

No:

```text
EntityManager
TransactionContext
ConnectionLease
ResultCursor
UnitOfWork
IdentityMap
```

---

# 149. Serialization

`DatabaseContext` vivo no será serializable por default.

---

# 150. Job propagation

Cuando un request despacha un job:

```text
Request Context
      ↓
Safe Context Propagator
      ↓
DatabaseContextDescriptor
      ↓
Queue
      ↓
Job Execution
      ↓
New DatabaseExecutionScope
      ↓
New DatabaseContext
```

---

# 151. Security propagation

No todo contexto de seguridad deberá propagarse automáticamente a jobs.

---

# 152. Explicit propagation policy

Se definirá:

```text
DatabaseContextPropagationPolicy
```

---

# 153. Propagation dimensions

Cada dimensión podrá ser:

```text
PROPAGATE
RECOMPUTE
DROP
FORBID
```

---

# 154. Example

```text
Tenant        → PROPAGATE
Trace         → PROPAGATE
Transaction   → FORBID
Connection    → FORBID
Deadline      → RECOMPUTE
Authorization → RECOMPUTE / policy-dependent
```

---

# 155. Context and retries

Cada retry attempt podrá derivar:

```text
AttemptContext
```

---

# 156. Retry attempt identity

```text
OperationId
AttemptNumber
```

deberán distinguirse.

---

# 157. Retry invariants

No deberán cambiar silenciosamente:

```text
tenant
shard
persistence domain
security scope
```

entre attempts.

---

# 158. Routing changes during retry

Un failover puede cambiar:

```text
physical endpoint
```

sin cambiar:

```text
logical persistence domain
```

cuando sea seguro.

---

# 159. Transaction retry

Un nuevo transaction attempt podrá obtener:

```text
new TransactionId
new ConnectionLease
```

manteniendo el contexto lógico.

---

# 160. Context and failover

Failover podrá modificar el plan efectivo.

Ejemplo:

```text
writer-A
   ↓ fail
writer-B
```

Pero el contexto lógico sigue solicitando:

```text
WRITE
```

---

# 161. Authority epoch

Podrá incluirse:

```text
AuthorityEpoch
```

en routing state cuando sea necesario.

---

# 162. Context and caching

Cache keys deberán utilizar únicamente dimensiones contextuales que afecten semántica/visibilidad.

---

# 163. Tenant cache isolation

Si tenant modifica visibilidad:

```text
TenantId
```

debe participar en key/fingerprint.

---

# 164. Security cache isolation

Si dos actores pueden observar datasets distintos:

```text
security data scope
```

deberá formar parte de la identidad apropiada.

---

# 165. Trace IDs and cache

No deberán formar parte de cache keys por defecto.

---

# 166. Context and cursor pagination

Cursor deberá ligarse al contexto semántico necesario.

Ejemplo:

```text
persistence domain
tenant
shard/topology
query fingerprint
ordering fingerprint
```

---

# 167. Cursor ≠ context token

El cursor no será una representación completa del DatabaseContext.

---

# 168. Context and chunk checkpoint

Checkpoint deberá validar dimensiones necesarias antes de resume.

---

# 169. Checkpoint context drift

Si:

```text
checkpoint tenant = A
current tenant = B
```

resume deberá rechazarse.

---

# 170. Context and lazy collection

Una Lazy Collection deberá capturar:

```text
safe context binding
```

para evitar que el tenant cambie entre creación e iteración.

---

# 171. Lazy execution caveat

Como la ejecución es diferida:

```text
creation context
```

y:

```text
iteration context
```

podrían diferir.

La política deberá ser explícita.

---

# 172. Preferred lazy policy

Para query-backed lazy collections:

```text
bind semantic context at creation
+
validate scope at iteration
```

cuando corresponda.

---

# 173. Context and streaming

Streaming mantiene recursos activos.

Por tanto:

```text
stream lifetime
≤
scope lifetime
```

---

# 174. Context and imports

Import podrá derivar:

```text
ImportDatabaseContext
```

con:

```text
resource budgets
transaction policy
tenant
target domain
```

---

# 175. Context and exports

Export podrá derivar:

```text
ExportDatabaseContext
```

con:

```text
consistency requirement
read routing
resource budgets
security filtering
```

---

# 176. Context and bulk operations

Bulk mutations deberán preservar:

```text
tenant
shard
authorization
transaction
```

igual que ORM mutations.

---

# 177. Raw SQL

Raw SQL no podrá utilizarse para evadir DatabaseContext.

---

# 178. Raw SQL routing

Si el sistema no puede inferir:

```text
tenant
shard
access intent
```

de raw SQL, deberá requerirse información explícita cuando sea necesaria.

---

# 179. Context and audit

Query Audit podrá recibir:

```text
ActorReference
TenantId
PersistenceDomain
OperationId
TransactionId
QueryFingerprint
```

sin depender del HTTP request.

---

# 180. Context and sensitive data

No todo contexto será seguro para logs.

---

# 181. Context redaction

Se utilizará:

```text
DatabaseContextRedactor
```

para diagnósticos.

---

# 182. Diagnostic representation

Ejemplo:

```text
DatabaseContext
├── Scope: dbscope_xxx
├── Domain: main
├── Tenant: acme
├── Shard: shard-02
├── Access: READ
├── Consistency: READ_YOUR_WRITES
├── Transaction: tx_***
├── Actor: user:***
├── Runtime: FrankenPHP
└── Deadline: 1.8s remaining
```

---

# 183. Credentials excluded

Nunca incluir por default:

```text
database password
raw DSN secrets
access tokens
encryption keys
```

---

# 184. Context telemetry

Eventos posibles:

```text
database.context.created
database.context.derived
database.context.conflict
database.context.validation_failed
database.context.stale_access
```

---

# 185. Cardinality control

No emitir automáticamente todos los context values como metric labels.

---

# 186. Context diagnostics

Developer tooling podrá mostrar:

```text
requested context
effective routing
transaction binding
tenant/shard
consistency
resource budget
```

---

# 187. Explain Context

Se podrá ofrecer:

```php
DB::explainContext();
```

conceptualmente.

---

# 188. Explain output

```text
Persistence Domain: main
Tenant: acme
Shard: shard-03
Requested Access: READ
Effective Route: WRITER
Reason:
  active transaction
  +
  read-your-writes requirement
```

---

# 189. Explainability

Una decisión contextual deberá poder responder:

```text
why writer?
why this shard?
why this consistency?
why query rejected?
why context conflict?
```

---

# 190. Context builder

Para construcción controlada podrá existir:

```text
DatabaseContextBuilder
```

---

# 191. Builder use

Principalmente:

```text
bootstrap
runtime integration
testing
specialized operations
```

---

# 192. Builder ≠ mutable runtime context

El builder sólo construye.

No se utilizará como objeto contextual vivo.

---

# 193. Test Context Factory

Testing tendrá:

```text
TestDatabaseContextFactory
```

---

# 194. Test example

```php
$context = TestDatabaseContext::make()
    ->tenant('acme')
    ->shard('shard-01')
    ->readOnly()
    ->build();
```

---

# 195. Deterministic testing

Tests podrán fijar:

```text
Clock
Deadline
ScopeId
Tenant
Shard
Consistency
Actor
```

cuando sea necesario.

---

# 196. Context fixtures

Podrán existir helpers.

Pero el production context resolver no dependerá de testing.

---

# 197. Persistent runtime

En workers persistentes:

```text
Worker
│
├── Scope A
│   └── DatabaseContext A
│
├── Scope B
│   └── DatabaseContext B
│
└── Scope C
    └── DatabaseContext C
```

---

# 198. No context reuse

No:

```text
Context A
clear()
setTenant(B)
reuse as Context B
```

---

# 199. New root context

Cada execution obtendrá un nuevo root context.

---

# 200. Shared immutable dependencies

El contexto sí podrá referenciar servicios inmutables compartidos indirectamente, pero no deberá convertirse en container.

---

# 201. Context ≠ Service Container

No:

```php
$context->get(QueryCompiler::class);
```

como mecanismo general.

---

# 202. Context data only principle

Preferentemente:

```text
Context
=
semantic execution data
```

no:

```text
service locator
```

---

# 203. Context size

El objeto deberá mantenerse razonablemente pequeño.

---

# 204. Avoid copying heavy objects

Derivar un contexto no deberá duplicar:

```text
metadata registry
connection pools
large security graphs
```

---

# 205. Structural sharing

Como los componentes son inmutables:

```text
Parent Context
      │
      ├── TenantContext ───────┐
      ├── SecurityContext ─────┤ shared immutable values
      └── RuntimeContext ──────┘
              │
              ▼
         Child Context
```

puede reutilizar referencias seguras.

---

# 206. Context derivation cost

Debe ser bajo.

Especialmente porque podrá ocurrir por:

```text
query
transaction
chunk
retry
```

---

# 207. Context stack

Podrá existir internamente:

```text
DatabaseContextStack
```

para APIs de scope lexical.

---

# 208. Example lexical API

```php
DB::withConsistency(
    Consistency::STRONG,
    fn () => $repository->find($id),
);
```

---

# 209. Internal behavior

```text
current context
      ↓
derive child
      ↓
push
      ↓
execute callback
      ↓
finally pop
```

---

# 210. Exception-safe restoration

El pop deberá ocurrir mediante:

```text
finally
```

---

# 211. Context stack ≠ global stack

Será execution-local.

---

# 212. Nested contexts

```text
Root
└── StrongConsistency
    └── Transaction
        └── Query
```

---

# 213. Context stack validation

Un pop fuera de orden deberá detectarse.

---

# 214. Context frame ID

Cada frame podrá tener:

```text
ContextFrameId
```

para detectar corrupción.

---

# 215. Async caveat

Stacks simples basados en arrays worker-global no son seguros bajo coroutines.

---

# 216. Runtime-local stack

Deberá utilizarse almacenamiento compatible con el runtime.

---

# 217. Explicit context preferred internally

Aunque exista stack para ergonomía:

```text
explicit context passing
```

será preferible dentro del core.

---

# 218. Context sealing

Un contexto efectivo podrá marcarse:

```text
SEALED
```

antes de ejecución.

---

# 219. Sealed context

No acepta nuevos overrides.

---

# 220. Why sealing

Evita:

```text
routing decided
then tenant changes
then query executes
```

---

# 221. Context lifecycle

Conceptualmente:

```text
DRAFT
  ↓
RESOLVED
  ↓
VALIDATED
  ↓
SEALED
  ↓
USED
```

El root context puede utilizar un modelo simplificado si permanece inmutable.

---

# 222. Operation Context lifecycle

Para operaciones críticas puede ser útil mantener ese lifecycle explícito.

---

# 223. Context generation

Se podrá asociar:

```text
ContextGeneration
```

para detectar cambios de topología/configuración relevantes.

---

# 224. Metadata generation

Context no deberá copiar toda metadata generation si no es relevante.

---

# 225. Topology generation

Sí puede ser necesaria para:

```text
distributed routing
cursor/checkpoint validation
```

---

# 226. Configuration reload

Si configuración cambia durante un worker:

```text
ConfigGeneration N
→
ConfigGeneration N+1
```

un execution activo no deberá quedar en un estado híbrido impredecible.

---

# 227. Snapshot configuration reference

El root context podrá capturar:

```text
DatabaseConfigurationGeneration
```

al inicio del execution.

---

# 228. Mid-request reload

Por default:

```text
Execution A
```

continuará con una generación coherente.

La nueva configuración aplicará a:

```text
Execution B
```

---

# 229. Same rule for metadata

Cuando metadata hot reload exista:

```text
request coherence
```

tendrá prioridad sobre mezcla de generaciones.

---

# 230. Context coherence

Regla:

> Un DatabaseContext debe representar una visión internamente coherente de las dimensiones necesarias para ejecutar una operación.

---

# 231. Partial context

Puede existir durante construcción.

Pero:

```text
PartialContext
```

no deberá llegar al Executor si faltan dimensiones requeridas.

---

# 232. Required dimensions

Dependen de la operación.

Ejemplo:

```text
non-tenant local SQLite test
```

no necesita TenantId.

Pero:

```text
tenant-isolated production query
```

sí.

---

# 233. Context Requirements

Cada operación podrá declarar:

```text
DatabaseContextRequirements
```

---

# 234. Example

```php
final readonly class DatabaseContextRequirements
{
    public bool $requiresTenant;
    public bool $requiresResolvedShard;
    public DatabaseAccessIntent $access;
    public DatabaseConsistencyRequirement $consistency;
}
```

---

# 235. Requirement validation

```text
Context
+
Requirements
    ↓
ContextValidator
    ↓
VALID / INVALID / UNSATISFIABLE
```

---

# 236. UNKNOWN requirement satisfaction

Si no puede determinarse:

```text
UNKNOWN
```

no deberá convertirse automáticamente en:

```text
VALID
```

para operaciones críticas.

---

# 237. Context resolution architecture

```text
Application / ORM / Query
          │
          ▼
DatabaseContextRequirements
          │
          ▼
DatabaseContextResolver
          │
          ▼
Root DatabaseContext
          │
          ▼
DatabaseContextComposer
          │
          ▼
DatabaseContextValidator
          │
          ▼
Effective DatabaseContext
          │
          ▼
Routing / Planner / Executor
```

---

# 238. Directory Structure

Propuesta:

```text
src/Quantum/Database/Context/
│
├── Contract/
│   ├── DatabaseContextResolver.php
│   ├── DatabaseContextComposer.php
│   ├── DatabaseContextValidator.php
│   ├── DatabaseContextPropagator.php
│   └── ContextCompatibilityChecker.php
│
├── Model/
│   ├── DatabaseContext.php
│   ├── DatabaseContextDescriptor.php
│   ├── DatabaseContextRequirements.php
│   ├── DatabaseContextFingerprint.php
│   ├── ContextGeneration.php
│   └── ContextFrameId.php
│
├── Persistence/
│   ├── PersistenceDomainContext.php
│   └── PersistenceDomainId.php
│
├── Tenant/
│   ├── TenantContext.php
│   ├── TenantContextState.php
│   └── TenantId.php
│
├── Shard/
│   ├── ShardContext.php
│   ├── ShardContextState.php
│   └── ShardId.php
│
├── Access/
│   ├── DatabaseAccessIntent.php
│   └── AccessIntentContext.php
│
├── Consistency/
│   ├── ConsistencyContext.php
│   ├── DatabaseConsistencyRequirement.php
│   └── EffectiveConsistency.php
│
├── Transaction/
│   ├── TransactionBindingContext.php
│   └── TransactionBindingState.php
│
├── Security/
│   ├── DatabaseSecurityContext.php
│   ├── DatabaseActorReference.php
│   └── DatabasePrivilegeMode.php
│
├── Telemetry/
│   └── DatabaseTelemetryContext.php
│
├── Cancellation/
│   └── DatabaseCancellationContext.php
│
├── Resource/
│   └── ResourceBudgetContext.php
│
├── Runtime/
│   └── DatabaseRuntimeContext.php
│
├── Stack/
│   ├── DatabaseContextStack.php
│   └── DatabaseContextFrame.php
│
├── Fingerprint/
│   ├── CacheContextFingerprint.php
│   ├── CursorContextFingerprint.php
│   ├── CheckpointContextFingerprint.php
│   └── AuditContextFingerprint.php
│
├── Diagnostics/
│   ├── DatabaseContextDiagnostics.php
│   ├── DatabaseContextExplainer.php
│   └── DatabaseContextRedactor.php
│
└── Exception/
    ├── DatabaseContextException.php
    ├── DatabaseContextNotAvailableException.php
    ├── DatabaseContextConflictException.php
    ├── DatabaseContextValidationException.php
    ├── PersistenceDomainMismatchException.php
    ├── TenantRequiredException.php
    ├── TenantMismatchException.php
    ├── ShardUnresolvedException.php
    ├── ShardAffinityException.php
    ├── ConsistencyRequirementException.php
    ├── TransactionContextConflictException.php
    └── StaleDatabaseContextException.php
```

---

# 239. Arquitectura de integración

```text
                       DatabaseExecutionScope
                               │
                               ▼
                        DatabaseContext
                               │
      ┌──────────────┬─────────┼─────────┬───────────────┐
      │              │         │         │               │
      ▼              ▼         ▼         ▼               ▼
 Persistence       Tenant     Shard    Security      Consistency
   Domain
      │              │         │         │               │
      └──────────────┴────┬────┴─────────┴───────────────┘
                          │
                          ▼
                 Context Requirements
                          │
                          ▼
                   Context Composer
                          │
                          ▼
                   Context Validator
                          │
                          ▼
                    Sealed Context
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
          Query          ORM       Transaction
             │            │            │
             └────────────┼────────────┘
                          ▼
                        Routing
                          │
                          ▼
                     Connection
                          │
                          ▼
                       Driver
```

---

# 240. Architectural Invariants

## DB-CTX-001

Toda operación Database deberá disponer de contexto efectivo.

## DB-CTX-002

DatabaseContext pertenecerá a un execution scope.

## DB-CTX-003

DatabaseContext no será configuration.

## DB-CTX-004

DatabaseContext no será QueryContext.

## DB-CTX-005

DatabaseContext no será TransactionContext.

## DB-CTX-006

DatabaseContext no será Service Container.

## DB-CTX-007

El root context será creado por execution.

## DB-CTX-008

Context derivation no mutará parent por default.

## DB-CTX-009

Context values serán inmutables cuando sea posible.

## DB-CTX-010

Scoped mutable state será separado de immutable context values.

## DB-CTX-011

Persistence Domain tendrá identidad explícita.

## DB-CTX-012

Persistence Domain no equivaldrá necesariamente a physical database.

## DB-CTX-013

Tenant será una dimensión contextual explícita.

## DB-CTX-014

Tenant unresolved no equivaldrá a system tenant.

## DB-CTX-015

TenantContext no equivaldrá a Authentication.

## DB-CTX-016

TenantContext no equivaldrá a Authorization.

## DB-CTX-017

Shard será una dimensión contextual explícita cuando aplique.

## DB-CTX-018

UNRESOLVED shard no equivaldrá a MULTI_SHARD.

## DB-CTX-019

UNKNOWN shard no equivaldrá a GLOBAL.

## DB-CTX-020

Shard map generation podrá formar parte del contexto.

## DB-CTX-021

Access intent será semántico.

## DB-CTX-022

READ no equivaldrá a replica.

## DB-CTX-023

WRITE no elegirá directamente SQL endpoint desde el Context.

## DB-CTX-024

LOCKING_READ será distinto de READ.

## DB-CTX-025

Schema access podrá distinguirse de data access.

## DB-CTX-026

Consistency requirement será explícito.

## DB-CTX-027

Requested consistency será distinta de effective consistency.

## DB-CTX-028

Consistency no se degradará silenciosamente.

## DB-CTX-029

Read-your-writes podrá influir routing.

## DB-CTX-030

Replication position podrá formar parte del state contextual.

## DB-CTX-031

Sticky writer state será execution-scoped.

## DB-CTX-032

Transaction binding será explícito.

## DB-CTX-033

Transaction affinity tendrá precedencia sobre routing preference.

## DB-CTX-034

Child context no violará transaction shard affinity.

## DB-CTX-035

Transaction context conflict será error.

## DB-CTX-036

Security será una dimensión contextual.

## DB-CTX-037

SecurityContext no requerirá HTTP User object.

## DB-CTX-038

ActorReference será transportable sin acoplar Database a HTTP.

## DB-CTX-039

SYSTEM actor no equivaldrá automáticamente a unrestricted.

## DB-CTX-040

Privilege elevation será explícita.

## DB-CTX-041

Privilege elevation será auditable.

## DB-CTX-042

Child context no aumentará privilegios arbitrariamente.

## DB-CTX-043

Telemetry correlation será contextual.

## DB-CTX-044

TelemetryContext no será exporter.

## DB-CTX-045

Sensitive telemetry policy podrá formar parte del contexto.

## DB-CTX-046

Cancellation será contextual.

## DB-CTX-047

Cancellation no equivaldrá a timeout.

## DB-CTX-048

Cancellation no equivaldrá a rollback.

## DB-CTX-049

Deadline será explícito cuando aplique.

## DB-CTX-050

Child deadline no extenderá parent deadline por default.

## DB-CTX-051

Effective timeout respetará deadlines superiores.

## DB-CTX-052

Resource budget será contextual.

## DB-CTX-053

Resource budget no será ResourcePermit.

## DB-CTX-054

Child context no fabricará presupuesto adicional.

## DB-CTX-055

Runtime information será contextual cuando sea relevante.

## DB-CTX-056

Runtime type no generará vendor conditionals dispersos.

## DB-CTX-057

Runtime capabilities serán preferidas sobre version checks.

## DB-CTX-058

Version no equivaldrá a capability.

## DB-CTX-059

Context composition tendrá precedencia definida.

## DB-CTX-060

Preference será distinta de Requirement.

## DB-CTX-061

Requirement será distinta de Constraint.

## DB-CTX-062

Strong constraint no será anulada por weak preference.

## DB-CTX-063

DatabaseContextResolver resolverá desde execution scope.

## DB-CTX-064

No existirá fallback global context.

## DB-CTX-065

Core podrá recibir DatabaseContext explícitamente.

## DB-CTX-066

Resolver se utilizará preferentemente en boundaries.

## DB-CTX-067

Core no dependerá de Service Locator contextual profundo.

## DB-CTX-068

QueryContext podrá derivarse de DatabaseContext.

## DB-CTX-069

ORM utilizará el mismo DatabaseContext base.

## DB-CTX-070

Entity identity incluirá domain context cuando sea necesario.

## DB-CTX-071

Raw primary key no definirá por sí solo identidad global.

## DB-CTX-072

Persistence Engine respetará DatabaseContext.

## DB-CTX-073

Schema operations podrán derivar specialized context.

## DB-CTX-074

Migration-specific semantics no inflarán root context.

## DB-CTX-075

DatabaseContext no será God Context.

## DB-CTX-076

Context extensions serán tipadas.

## DB-CTX-077

Context extension registry podrá congelarse.

## DB-CTX-078

Context fingerprint será semántico.

## DB-CTX-079

Context fingerprint no equivaldrá a object serialization.

## DB-CTX-080

No existirá necesariamente un único universal fingerprint.

## DB-CTX-081

Trace ID no formará parte de cache key por default.

## DB-CTX-082

Context compatibility será consumer-specific.

## DB-CTX-083

Context compatibility no equivaldrá a object identity.

## DB-CTX-084

Context será validado antes de operaciones críticas.

## DB-CTX-085

Structural validation será distinta de semantic validation.

## DB-CTX-086

Semantic validation será distinta de execution validation.

## DB-CTX-087

Requested context será distinto de effective execution decision.

## DB-CTX-088

Context conflict tendrá diagnóstico estructurado.

## DB-CTX-089

Root context no sobrevivirá a execution scope.

## DB-CTX-090

Child context no sobrevivirá al parent scope.

## DB-CTX-091

Stale context access será detectable.

## DB-CTX-092

DatabaseContext vivo no se propagará a queue.

## DB-CTX-093

Handoff utilizará descriptor seguro.

## DB-CTX-094

Descriptor no contendrá ConnectionLease.

## DB-CTX-095

Descriptor no contendrá EntityManager.

## DB-CTX-096

Descriptor no contendrá UnitOfWork.

## DB-CTX-097

Descriptor no contendrá TransactionContext vivo.

## DB-CTX-098

Descriptor no contendrá ResultCursor.

## DB-CTX-099

Context propagation tendrá política explícita.

## DB-CTX-100

Transaction será FORBID para async propagation por default.

## DB-CTX-101

Connection será FORBID para async propagation.

## DB-CTX-102

Retry attempts mantendrán domain identity.

## DB-CTX-103

Retry attempts mantendrán tenant identity.

## DB-CTX-104

Retry attempts mantendrán security scope salvo política explícita.

## DB-CTX-105

Retry podrá cambiar physical endpoint de forma segura.

## DB-CTX-106

Logical domain no cambiará por simple failover.

## DB-CTX-107

Cache keys incluirán dimensiones contextuales relevantes.

## DB-CTX-108

Tenant cache isolation será obligatoria cuando tenant afecte visibilidad.

## DB-CTX-109

Security data scope participará en cache identity cuando corresponda.

## DB-CTX-110

Cursor se ligará a dimensiones contextuales necesarias.

## DB-CTX-111

Cursor no será DatabaseContext serializado.

## DB-CTX-112

Checkpoint validará context binding.

## DB-CTX-113

Checkpoint tenant mismatch será rechazado.

## DB-CTX-114

Lazy collection deberá controlar context drift.

## DB-CTX-115

Stream lifetime no superará scope lifetime.

## DB-CTX-116

Import respetará target context.

## DB-CTX-117

Export respetará security context.

## DB-CTX-118

Bulk operations respetarán tenant context.

## DB-CTX-119

Bulk operations respetarán shard context.

## DB-CTX-120

Raw SQL no evadirá context requirements.

## DB-CTX-121

Audit utilizará actor/context references seguras.

## DB-CTX-122

Database credentials no aparecerán en diagnostics por default.

## DB-CTX-123

Context diagnostics serán redactados.

## DB-CTX-124

Context telemetry tendrá cardinalidad controlada.

## DB-CTX-125

Context decisions serán explicables.

## DB-CTX-126

Context builder no será mutable runtime context.

## DB-CTX-127

Testing podrá construir contextos deterministas.

## DB-CTX-128

Cada persistent-runtime execution tendrá nuevo root context.

## DB-CTX-129

Root context no será reciclado mediante clear-and-reconfigure.

## DB-CTX-130

DatabaseContext no será Service Locator.

## DB-CTX-131

Context derivation tendrá costo acotado.

## DB-CTX-132

Immutable values podrán usar structural sharing.

## DB-CTX-133

Context stack será execution-local.

## DB-CTX-134

Context stack será exception-safe.

## DB-CTX-135

Context stack será coroutine/fiber-safe.

## DB-CTX-136

Core preferirá explicit context passing.

## DB-CTX-137

Effective context podrá sellarse.

## DB-CTX-138

Sealed context no aceptará overrides incompatibles.

## DB-CTX-139

Context generation podrá utilizarse para detectar staleness.

## DB-CTX-140

Configuration generation podrá fijarse por execution.

## DB-CTX-141

Metadata generation podrá fijarse por execution cuando corresponda.

## DB-CTX-142

Mid-request reload no producirá contexto híbrido por default.

## DB-CTX-143

DatabaseContext deberá ser internamente coherente.

## DB-CTX-144

Partial context no llegará al Executor si faltan requisitos.

## DB-CTX-145

Context requirements serán operation-specific.

## DB-CTX-146

UNKNOWN satisfaction no equivaldrá automáticamente a VALID.

## DB-CTX-147

Context resolver no decidirá SQL.

## DB-CTX-148

Context composer no ejecutará queries.

## DB-CTX-149

Context validator no abrirá connections por default.

## DB-CTX-150

Routing consumirá contexto; no será parte del Context object.

## DB-CTX-151

Compiler no dependerá de actor HTTP.

## DB-CTX-152

Driver no resolverá tenant.

## DB-CTX-153

Connection no resolverá authorization.

## DB-CTX-154

ORM no mantendrá current tenant estático.

## DB-CTX-155

TransactionManager no mantendrá current transaction global.

## DB-CTX-156

QueryExecutor no mantendrá current context global.

## DB-CTX-157

Persistent runtime no modificará estas reglas.

## DB-CTX-158

FrankenPHP usará el mismo DatabaseContext model.

## DB-CTX-159

RoadRunner usará el mismo DatabaseContext model.

## DB-CTX-160

OpenSwoole usará el mismo DatabaseContext model.

## DB-CTX-161

Traditional PHP usará el mismo DatabaseContext model.

## DB-CTX-162

DatabaseContext será una frontera semántica central de Database.

## DB-CTX-163

DatabaseContext no sustituirá los specialized contexts.

## DB-CTX-164

Context state mutable tendrá ownership explícito.

## DB-CTX-165

Ninguna dimensión crítica dependerá de mutable process-global state.

---

# 241. Modelo formal

Sea un contexto:

```text
C
```

definido como:

```text
C =
(
 D,
 T,
 S,
 A,
 K,
 X,
 G,
 M,
 N,
 B,
 R
)
```

donde:

```text
D = Persistence Domain
T = Tenant Context
S = Shard Context
A = Access Intent
K = Consistency Requirement
X = Transaction Binding
G = Security Context
M = Telemetry Context
N = Cancellation/Deadline Context
B = Resource Budget
R = Runtime Context
```

---

# 242. Derivación formal

Un child context:

```text
C'
```

se obtiene mediante:

```text
C' = derive(C, Δ)
```

donde:

```text
Δ
```

representa overrides permitidos.

---

# 243. Constraint preservation

La derivación será válida sólo si:

```text
Constraints(C')
⊇
MandatoryConstraints(C)
```

cuando las restricciones heredadas no puedan relajarse.

---

# 244. Security monotonicity

Para contextos sin elevación explícita:

```text
Privileges(C')
⊆
Privileges(C)
```

---

# 245. Deadline monotonicity

```text
Deadline(C')
≤
Deadline(C)
```

cuando ambos existan.

---

# 246. Transaction affinity

Si:

```text
Transaction(C) = tx
```

y:

```text
Shard(tx) = s
```

entonces:

```text
Shard(C') = s
```

para operaciones pertenecientes a esa transaction.

---

# 247. Context validity

Definimos:

```text
Valid(C, O)
```

como:

```text
StructuralValid(C)
∧
SemanticValid(C)
∧
RequirementsSatisfied(C, O)
∧
CapabilitiesSatisfy(C, O)
```

---

# 248. UNKNOWN semantics

Si una dimensión requerida produce:

```text
UNKNOWN
```

entonces:

```text
Valid(C, O)
```

no deberá convertirse automáticamente en `true`.

---

# 249. Context flow completo

```text
                  EXECUTION
                      │
                      ▼
             DatabaseExecutionScope
                      │
                      ▼
              Root DatabaseContext
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
      Tenant        Security      Runtime
        │             │             │
        └─────────────┼─────────────┘
                      ▼
               Operation Request
                      │
                      ▼
              ContextRequirements
                      │
                      ▼
               ContextComposer
                      │
                      ▼
                Child Context
                      │
                      ▼
               ContextValidator
                      │
               ┌──────┴──────┐
               ▼             ▼
            INVALID        VALID
                              │
                              ▼
                           SEALED
                              │
                              ▼
                    Routing / Planning
                              │
                              ▼
                         Execution
                              │
                              ▼
                     Scoped Resources
                              │
                              ▼
                     Scope Finalization
```

---

# 250. Relación con los documentos anteriores

`251_DATABASE_PERSISTENT_RUNTIME_ARCHITECTURE.md` estableció:

```text
Worker
≠
Request
```

`252_DATABASE_REQUEST_SCOPE_SYSTEM.md` estableció:

```text
Execution
→
isolated DatabaseExecutionScope
```

Este documento establece:

```text
DatabaseExecutionScope
→
explicit DatabaseContext
```

La cadena completa queda:

```text
Persistent Worker
       │
       ▼
DatabaseExecutionScope
       │
       ▼
DatabaseContext
       │
       ▼
Database Operation Context
       │
       ▼
Query / ORM / Transaction
       │
       ▼
Routing
       │
       ▼
Connection Lease
       │
       ▼
Driver
```

---

# 251. Regla final

> **DatabaseContext será la fuente semántica explícita de las condiciones bajo las cuales una operación de VoltStack Database debe ejecutarse. Ningún subsistema deberá depender de variables globales para descubrir silenciosamente tenant, shard, transacción, consistencia, seguridad, routing o estado de ejecución.**

La separación fundamental será:

```text
Scope
=
Who owns the state and for how long?

Context
=
Under what database semantics is this operation running?

Operation
=
What is being performed?

Plan
=
How will it be performed?

Executor
=
Perform it.
```

Por tanto:

```text
DatabaseExecutionScope
        ↓
DatabaseContext
        ↓
Operation Requirements
        ↓
Effective Context
        ↓
Plan
        ↓
Execution
```

y nunca:

```text
random global state
        ↓
hidden behavior
```

---

# 252. Estado del Bloque 25

```text
BLOCK 25 — PERSISTENT RUNTIME

✓ 251_DATABASE_PERSISTENT_RUNTIME_ARCHITECTURE.md
✓ 252_DATABASE_REQUEST_SCOPE_SYSTEM.md
✓ 253_DATABASE_DATABASE_CONTEXT_SYSTEM.md
│
├── 254_DATABASE_STATE_ISOLATION_SYSTEM.md
├── 255_DATABASE_STATE_RESET_SYSTEM.md
├── 256_DATABASE_CONNECTION_REUSE_SYSTEM.md
├── 257_DATABASE_WORKER_LIFECYCLE_SYSTEM.md
├── 258_DATABASE_FRANKENPHP_INTEGRATION_SYSTEM.md
├── 259_DATABASE_ROADRUNNER_INTEGRATION_SYSTEM.md
└── 260_DATABASE_OPENSWOOLE_INTEGRATION_SYSTEM.md
```

---

# 253. Siguiente documento

```text
254_DATABASE_STATE_ISOLATION_SYSTEM.md
```

El siguiente documento definirá cómo VoltStack garantizará que:

```text
Request A
Job A
Transaction A
Tenant A
ORM State A
```

no puedan contaminar:

```text
Request B
Job B
Transaction B
Tenant B
ORM State B
```

dentro de procesos persistentes.

La arquitectura profundizará en:

```text
State Classification
        │
        ├── Immutable Shared State
        ├── Worker-local State
        ├── Execution-local State
        ├── Operation-local State
        ├── Transaction-local State
        └── Resource-local State

Isolation Boundaries
State Ownership
Runtime-local Storage
Cross-scope Detection
Tenant Isolation
ORM Isolation
Transaction Isolation
Connection State Isolation
Fiber/Coroutine Isolation
Static State Detection
Leak Detection
Isolation Verification
Persistent Worker Safety
```

estableciendo formalmente la regla:

> **En VoltStack Database, compartir un proceso nunca implicará compartir estado mutable de una ejecución.**