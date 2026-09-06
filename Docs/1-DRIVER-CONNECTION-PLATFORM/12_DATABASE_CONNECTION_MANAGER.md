# 12_DATABASE_CONNECTION_MANAGER.md

# VoltStack Quantum Database
## Connection Manager

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 12 — Database Connection Manager  
**Estado:** Architecture Specification  
**Nivel:** Infrastructure Architecture  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura oficial de:

```text
VoltStack\Quantum\Database\Connection\ConnectionManager
```

dentro de:

```text
VoltStack/Quantum/Database
```

`ConnectionManager` será el punto central de coordinación para resolver y proporcionar conexiones lógicas configuradas.

Su responsabilidad principal será responder:

> ¿Qué `Connection` lógica corresponde a una solicitud determinada y cómo se obtiene de manera segura?

No será responsable directamente de:

```text
SQL
Query Builder
ORM
PDO
pool implementation
transaction semantics
database topology
```

---

# 2. Posición arquitectónica

```text
Application / Database Services
             │
             ▼
      ConnectionManager
             │
    ┌────────┼─────────┐
    ▼        ▼         ▼
Definitions Factory  Resolver
    │        │         │
    └────────┼─────────┘
             ▼
      Logical Connection
             │
             ▼
   Resource Acquisition
             │
        ┌────┴────┐
        ▼         ▼
       Pool      Driver
        │         │
        └────┬────┘
             ▼
     Native Connection
```

---

# 3. Definición

> `ConnectionManager` es el servicio de infraestructura responsable de coordinar definiciones, resolución y creación de conexiones lógicas.

No representa una conexión.

Por tanto:

```text
ConnectionManager
≠
Connection
```

---

# 4. Responsabilidades principales

`ConnectionManager` deberá encargarse de:

```text
default connection resolution
named connection resolution
connection definition lookup
logical connection creation
logical connection reuse when safe
connection aliases
connection factory coordination
connection resolver coordination
scoped dynamic definitions
connection diagnostics
connection availability information
```

---

# 5. No responsabilidades

No deberá encargarse directamente de:

```text
SQL compilation
SQL execution
parameter binding
query optimization
entity hydration
UnitOfWork
IdentityMap
schema compilation
migration planning
retrying arbitrary writes
replica selection algorithms
physical pool implementation
native driver communication
```

---

# 6. Regla arquitectónica principal

```text
ConnectionManager
≠
DriverRegistry
≠
ConnectionPool
≠
TransactionManager
≠
QueryExecutor
≠
TopologyManager
```

Cada uno tiene un dominio independiente.

---

# 7. ConnectionManager como orquestador

El Manager coordina componentes.

No absorbe sus responsabilidades.

```text
ConnectionManager
     │
     ├── DefinitionRegistry
     ├── ConnectionFactory
     ├── ConnectionResolver
     ├── AliasRegistry
     └── ScopedDefinitionProvider
```

---

# 8. API conceptual

Una API inicial podría ser:

```php
interface ConnectionManagerInterface
{
    public function connection(
        ?string $name = null
    ): ConnectionInterface;

    public function has(string $name): bool;

    public function defaultConnectionName(): string;
}
```

La interfaz pública deberá permanecer deliberadamente pequeña.

---

# 9. API pública vs API interna

No todas las operaciones del Manager deberán formar parte del contrato público.

Ejemplo:

```text
Public:
connection()
has()
defaultConnectionName()

Internal:
resolveDefinition()
createConnection()
resolveAlias()
resolveScopedDefinition()
```

---

# 10. Default connection

Una aplicación podrá configurar:

```yaml
database:
  default: primary
```

Entonces:

```php
$manager->connection();
```

será equivalente conceptualmente a:

```php
$manager->connection('primary');
```

---

# 11. Default connection resolution

Flujo:

```text
connection(null)
      │
      ▼
DefaultConnectionResolver
      │
      ▼
"primary"
      │
      ▼
resolve("primary")
```

---

# 12. Default connection is configuration

El Manager no deberá tener:

```php
private string $default = 'mysql';
```

hardcoded.

El valor deberá provenir de configuración normalizada.

---

# 13. Named connections

VoltStack deberá soportar:

```php
$manager->connection('primary');
$manager->connection('analytics');
$manager->connection('legacy');
```

Cada nombre identifica una:

```text
ConnectionDefinition
```

---

# 14. Connection names

Los nombres serán identificadores lógicos.

Ejemplos:

```text
primary
replica
analytics
audit
reporting
legacy
```

No deberán utilizarse para inferir semántica.

---

# 15. No role inference from name

Incorrecto:

```php
if (str_contains($name, 'replica')) {
    // read-only
}
```

Correcto:

```text
ConnectionDefinition
    │
    └── ConnectionRole::REPLICA
```

---

# 16. ConnectionDefinitionRegistry

El Manager podrá depender de:

```text
ConnectionDefinitionRegistry
```

para almacenar definiciones estáticas compiladas durante bootstrap.

---

# 17. Registry contents

Ejemplo:

```text
ConnectionDefinitionRegistry
├── primary
├── analytics
├── reporting
└── legacy
```

Cada entrada contiene un objeto tipado.

---

# 18. Registry does not contain PDO

El Registry no deberá almacenar:

```text
PDO
NativeConnection
ConnectionLease
Transaction
```

Su responsabilidad son definiciones.

---

# 19. Definition immutability

Una `ConnectionDefinition` deberá ser:

```text
immutable
validated
normalized
```

después de bootstrap.

---

# 20. Registry freeze

Después del bootstrap:

```text
ConnectionDefinitionRegistry
       │
       ▼
     freeze()
```

Las definiciones globales no deberán modificarse durante requests ordinarios.

---

# 21. Razón del freeze

Evita:

```text
runtime configuration mutation
cross-request leakage
race conditions
unpredictable connection behavior
```

especialmente bajo workers persistentes.

---

# 22. ConnectionFactory

El Manager delegará creación de conexiones lógicas a:

```text
ConnectionFactory
```

---

# 23. Factory contract

Conceptualmente:

```php
interface ConnectionFactoryInterface
{
    public function create(
        ConnectionDefinition $definition
    ): ConnectionInterface;
}
```

---

# 24. Factory does not open physical connection

Regla:

```text
create logical connection
≠
connect to database
```

La adquisición física seguirá siendo lazy por defecto.

---

# 25. Connection creation

```text
ConnectionDefinition
       │
       ▼
ConnectionFactory
       │
       ▼
Logical Connection
```

---

# 26. Physical acquisition

Sólo posteriormente:

```text
Logical Connection
       │
       ▼
Resource Provider
       │
       ▼
Pool / Driver
       │
       ▼
Native Connection
```

---

# 27. ConnectionResolver

`ConnectionResolver` representa la lógica para convertir una solicitud lógica en una definición/conexión concreta.

---

# 28. Resolver contract

Conceptualmente:

```php
interface ConnectionResolverInterface
{
    public function resolve(
        ConnectionResolutionRequest $request
    ): ConnectionResolution;
}
```

---

# 29. Why a resolution request?

En escenarios simples basta un nombre.

En escenarios avanzados pueden intervenir:

```text
requested name
connection intent
execution scope
tenant context
topology decision
shard
role requirement
```

---

# 30. ConnectionResolutionRequest

Podrá modelarse como:

```php
final readonly class ConnectionResolutionRequest
{
    public function __construct(
        public ?ConnectionIdentity $identity = null,
        public ?ConnectionIntent $intent = null,
        public ?ExecutionScopeId $scope = null,
    ) {}
}
```

Sin convertirlo en una bolsa arbitraria de contexto.

---

# 31. ConnectionResolution

Resultado conceptual:

```php
final readonly class ConnectionResolution
{
    public function __construct(
        public ConnectionDefinition $definition,
        public ConnectionResolutionReason $reason,
    ) {}
}
```

---

# 32. Simple resolution

Caso básico:

```text
"analytics"
    │
    ▼
DefinitionRegistry
    │
    ▼
analytics definition
```

---

# 33. Advanced resolution

Caso futuro:

```text
READ request
    │
    ▼
Topology
    │
    ▼
Replica Target
    │
    ▼
ConnectionResolver
    │
    ▼
replica-eu-02
```

---

# 34. Topology boundary

El Manager no deberá implementar algoritmos completos de:

```text
replica balancing
sharding
partition selection
failover topology
```

Eso pertenece a Topology.

---

# 35. ConnectionManager receives decisions

Idealmente:

```text
Topology
   │
   ▼
Connection Target
   │
   ▼
ConnectionManager
```

No:

```text
ConnectionManager
   │
   ├── topology
   ├── sharding
   ├── load balancing
   └── ORM
```

---

# 36. Connection aliases

VoltStack podrá soportar aliases.

Ejemplo:

```text
default-write → primary
default-read  → replica
```

---

# 37. Alias registry

Los aliases deberán mantenerse separados de las definiciones.

```text
ConnectionAliasRegistry
```

---

# 38. Alias resolution

```text
requested alias
      │
      ▼
Alias Registry
      │
      ▼
Connection Identity
      │
      ▼
Definition Registry
```

---

# 39. Alias cycles

Deberán detectarse ciclos:

```text
a → b
b → c
c → a
```

y producir una excepción de configuración.

---

# 40. Alias normalization

Idealmente los aliases se resolverán durante bootstrap cuando sean completamente estáticos.

---

# 41. Runtime aliases

Sólo deberán existir cuando un caso de uso real lo requiera.

No se diseñará mutabilidad arbitraria por defecto.

---

# 42. Logical connection caching

El Manager podrá reutilizar objetos `Connection` lógicos si son:

```text
stateless
immutable
or safely scoped
```

---

# 43. Logical connection cache

Ejemplo:

```text
ConnectionManager
├── primary → LogicalConnection
└── analytics → LogicalConnection
```

Esto puede ser seguro si los objetos no almacenan estado request-specific.

---

# 44. Critical distinction

```text
Caching Logical Connection
≠
Caching Native Connection
```

---

# 45. Logical connection requirements

Una Logical Connection compartida deberá evitar propiedades como:

```text
current transaction
current tenant
current PDO
current statement
current query
current request
```

---

# 46. Stateful connection objects

Si la implementación de `ConnectionInterface` necesita estado mutable por execution scope, deberá ser:

```text
scoped
```

y no almacenarse globalmente en el Manager.

---

# 47. Recommended model

Preferencia:

```text
Persistent Logical Connection Descriptor
            │
            ▼
Scoped Connection Access / Lease
            │
            ▼
Native Resource
```

---

# 48. ConnectionManager lifetime

El Manager podrá ser:

```text
application/worker singleton
```

si permanece libre de estado mutable request-specific.

---

# 49. Singleton safety

Puede almacenar:

```text
immutable configuration
frozen registries
stateless factories
stateless resolvers
```

No puede almacenar:

```text
current tenant
current transaction
current request lease
current native connection
```

---

# 50. Persistent worker invariant

```text
ConnectionManager(Request A)
==
ConnectionManager(Request B)
```

puede ser la misma instancia.

Pero:

```text
DatabaseContext A
!=
DatabaseContext B
```

---

# 51. Request state

Todo estado contextual deberá permanecer en:

```text
DatabaseContext
ConnectionContext
TransactionContext
ExecutionContext
```

según ownership.

---

# 52. Incorrect Manager state

```php
final class ConnectionManager
{
    private ?ConnectionLease $currentLease = null;
}
```

si el Manager es singleton.

**Prohibido.**

---

# 53. Correct scoped state

```text
ConnectionManager
      │
      ▼
DatabaseContext
      │
      ▼
ConnectionLease
```

---

# 54. Dynamic connections

VoltStack deberá permitir conexiones dinámicas para casos como:

```text
multitenancy
temporary database targets
test databases
runtime credentials
ephemeral infrastructure
```

sin mutar necesariamente el registry global.

---

# 55. ScopedConnectionDefinitionProvider

Podrá existir:

```php
interface ScopedConnectionDefinitionProviderInterface
{
    public function resolve(
        ConnectionResolutionRequest $request
    ): ?ConnectionDefinition;
}
```

---

# 56. Resolution order

Ejemplo:

```text
Explicit Scoped Definition
          │
          ▼
Named Static Definition
          │
          ▼
Alias Resolution
          │
          ▼
Default Resolution
```

El orden definitivo deberá ser determinista.

---

# 57. Dynamic definition isolation

Una definición dinámica perteneciente a Tenant A no deberá quedar disponible accidentalmente para Tenant B.

---

# 58. No global tenant registration

Incorrecto:

```php
$globalRegistry->add(
    'tenant-' . $tenantId,
    $definition
);
```

para cada request.

Esto puede:

```text
grow indefinitely
leak tenant state
create concurrency problems
```

---

# 59. Correct tenant model

```text
TenantContext A
      │
      ▼
TenantConnectionResolver
      │
      ▼
Scoped Definition A
```

La definición desaparece con el scope.

---

# 60. Multitenancy boundary

`ConnectionManager` no dependerá directamente del paquete Multitenancy.

En su lugar:

```text
Multitenancy Package
        │
        ▼
Database Integration Port
        │
        ▼
Scoped Connection Resolver
```

---

# 61. Connection manager without multitenancy

Debe funcionar completamente sin:

```text
VoltStack/Quantum/Multitenancy
```

---

# 62. Connection Manager and Driver

El Manager no crea `PDO`.

Flujo correcto:

```text
ConnectionManager
       │
       ▼
ConnectionFactory
       │
       ▼
Logical Connection
       │
       ▼
Resource Provider
       │
       ▼
Driver
```

---

# 63. DriverRegistry boundary

El Manager puede depender indirectamente de Driver resolution durante creación/adquisición.

Pero no deberá convertirse en Driver Registry.

---

# 64. ConnectionManager and Pool

El Manager tampoco deberá almacenar directamente:

```text
idle native connections
pool wait queues
connection counters
```

Eso pertenece a:

```text
ConnectionPool
```

---

# 65. Pool interaction

```text
ConnectionManager
       │
       ▼
Logical Connection
       │
       ▼
ConnectionResourceProvider
       │
       ▼
ConnectionPool
```

---

# 66. No-pool compatibility

El mismo Manager deberá funcionar cuando:

```text
ConnectionResourceProvider
       │
       ▼
DirectDriverResourceProvider
```

---

# 67. Resource provider contract

Conceptualmente:

```php
interface ConnectionResourceProviderInterface
{
    public function acquire(
        ConnectionDefinition $definition,
        ConnectionAcquisitionContext $context
    ): ConnectionLease;
}
```

---

# 68. Why resource provider?

Permite ocultar si la conexión procede de:

```text
pool
direct driver connection
runtime-specific pool
specialized connector
```

---

# 69. Manager should not acquire prematurely

Solicitar:

```php
$manager->connection('primary');
```

no deberá necesariamente consumir un slot del pool.

---

# 70. Lazy acquisition preservation

```text
resolve connection
      │
      ▼
Logical Connection
      │
      ▼
no native resource yet
```

Sólo una operación que realmente necesita DB adquiere recurso.

---

# 71. ConnectionManager and Executor

Executor puede utilizar ConnectionManager para resolver acceso.

Pero el Manager no ejecuta SQL.

---

# 72. Example execution flow

```text
QueryExecutor
     │
     ▼
ConnectionRequirement
     │
     ▼
ConnectionManager
     │
     ▼
Logical Connection
     │
     ▼
Resource Provider
     │
     ▼
ConnectionLease
     │
     ▼
Driver
```

---

# 73. ConnectionRequirement

El Manager podrá aceptar una abstracción como:

```text
ConnectionRequirement
```

en APIs internas.

---

# 74. Requirement example

```php
new ConnectionRequirement(
    intent: ConnectionIntent::READ,
    consistency: ConsistencyRequirement::EVENTUAL,
);
```

---

# 75. Named override

Podrá existir:

```php
new ConnectionRequirement(
    identity: ConnectionIdentity::from('analytics'),
    intent: ConnectionIntent::READ,
);
```

---

# 76. Explicit connection wins carefully

Si el usuario solicita una conexión explícita, la resolución deberá respetarla salvo que viole:

```text
transaction affinity
security policy
capability requirement
```

---

# 77. TransactionManager boundary

El ConnectionManager no administra transactions.

---

# 78. Transaction context

Cuando existe una transacción:

```text
TransactionContext
      │
      ▼
Pinned ConnectionLease
```

---

# 79. Manager transaction awareness

El Manager podrá consultar un contrato contextual para respetar una afinidad existente.

Pero no deberá almacenar la transaction activa.

---

# 80. Correct interaction

```text
QueryExecutor
     │
     ▼
TransactionContext
     │
     ├── pinned lease exists
     │       │
     │       ▼
     │      use it
     │
     └── none
             │
             ▼
      ConnectionManager
```

---

# 81. Alternative orchestration

También podrá existir:

```text
ConnectionAcquisitionCoordinator
```

encima de ConnectionManager para combinar:

```text
transaction affinity
topology
pool acquisition
```

Esto puede mantener el Manager todavía más pequeño.

---

# 82. Manager design preference

Regla:

> Si una responsabilidad requiere conocer demasiado sobre transactions, topology o execution, probablemente no pertenece al ConnectionManager.

---

# 83. Transaction-pinned resolution

Nunca:

```text
Transaction on Connection A
      │
      ▼
manager resolves B
      │
      ▼
execute on B
```

para una operación que pertenece a esa transaction.

---

# 84. Connection identity inside transaction

El TransactionContext deberá conocer la identidad lógica y el lease físico utilizado.

---

# 85. Conflicting explicit connection

Ejemplo:

```text
transaction pinned to primary
```

y el código solicita:

```php
DB::connection('analytics')
```

dentro de la misma transaction.

El framework no deberá cambiar silenciosamente de conexión.

---

# 86. Conflict policy

Deberá:

```text
reject
```

o exigir una transaction independiente explícita.

Nunca hacer fallback silencioso.

---

# 87. Cross-database transactions

No forman parte del ConnectionManager básico.

---

# 88. Distributed transactions

Si se soportan en el futuro:

```text
DistributedTransactionCoordinator
```

será un subsystem separado.

---

# 89. Connection aliases and transaction affinity

Aliases deberán resolverse a identidades canónicas antes de comparar afinidad.

---

# 90. Canonical identity

Ejemplo:

```text
write → primary
```

Entonces:

```text
canonical(write)
=
primary
```

---

# 91. ConnectionDefinition fingerprint

Cada definición podrá poseer un fingerprint estable.

---

# 92. Fingerprint purpose

Puede utilizarse para:

```text
pool compatibility
configuration change detection
diagnostics
cache identity
resource invalidation
```

---

# 93. Fingerprint secrets

No deberá incluir secretos en forma reversible o visible.

---

# 94. Configuration generation

Podrá existir:

```text
ConfigurationGeneration
```

para detectar que recursos físicos fueron creados bajo una configuración anterior.

---

# 95. Credential rotation

Si las credenciales cambian:

```text
new generation
      │
      ▼
new acquisitions use new credentials
      │
      ▼
old pooled resources retire safely
```

---

# 96. Manager and secret resolution

El Manager idealmente trabaja con:

```text
CredentialReference
```

no con passwords plaintext.

---

# 97. Secret boundary

El secreto deberá resolverse lo más cerca posible de:

```text
physical connection creation
```

---

# 98. Connection Manager diagnostics

El Manager podrá proporcionar una vista segura de:

```text
configured connection names
default connection
roles
driver IDs
availability
configuration status
```

---

# 99. Diagnostics example

```text
Connections

primary
  role: PRIMARY
  driver: pdo.pgsql
  status: configured

analytics
  role: READ_ONLY
  driver: pdo.mysql
  status: configured
```

---

# 100. Diagnostics do not imply connectivity

```text
configured
≠
reachable
```

---

# 101. Health diagnostics

Reachability deberá consultarse mediante Health/Connection diagnostics específicos.

No durante una simple llamada de listado.

---

# 102. No accidental connections from debug tools

Mostrar la lista de conexiones no deberá abrirlas todas.

---

# 103. Manager telemetry

Operaciones observables:

```text
connection.manager.resolve
connection.manager.create_logical
connection.manager.alias_resolve
connection.manager.definition_resolve
connection.manager.resolve_failed
```

---

# 104. Physical telemetry belongs lower

Eventos como:

```text
physical connection opened
pool wait
native connect duration
```

pertenecen a Connection Resource/Pool/Driver.

---

# 105. Low-cardinality telemetry

Evitar etiquetas dinámicas sin control como:

```text
tenant ID
database name generated per customer
full endpoint
```

por defecto.

---

# 106. Events

Podrán existir eventos neutrales como:

```text
ConnectionResolutionStarted
ConnectionResolved
ConnectionResolutionFailed
LogicalConnectionCreated
```

---

# 107. Event payload security

Los eventos no deberán incluir secretos.

---

# 108. Resolver extensibility

Paquetes podrán añadir resolvers especializados mediante contratos.

Ejemplos:

```text
TenantConnectionResolver
ShardConnectionResolver
ReadWriteConnectionResolver
TestConnectionResolver
```

---

# 109. Resolver chain

Puede existir:

```text
ConnectionResolverPipeline
```

si múltiples resolvers necesitan colaborar.

---

# 110. Resolver pipeline caution

No deberá convertirse en:

```text
50 resolvers all trying random things
```

La precedencia deberá ser explícita.

---

# 111. Resolver result model

Cada resolver podrá responder:

```text
Resolved
NotApplicable
Rejected
```

en vez de usar `null` para todo.

---

# 112. Resolution rejection

Ejemplo:

```text
requested write
+
read-only connection
=
Rejected
```

---

# 113. Resolver precedence

Una posible jerarquía:

```text
1. Transaction affinity
2. Explicit connection requirement
3. Scoped integration resolver
4. Topology/routing decision
5. Named/default static resolution
```

La arquitectura definitiva podrá repartir estos pasos entre coordinadores.

---

# 114. Separation from routing

El Manager no deberá conocer semántica de query para decidir:

```text
SELECT
INSERT
UPDATE
DELETE
```

Debe recibir:

```text
ConnectionIntent
```

ya derivado por capas superiores.

---

# 115. Intent abstraction

```text
Query/Planner
     │
     ▼
ConnectionIntent::READ
     │
     ▼
Connection Resolution
```

---

# 116. Capability requirements

La resolución podrá incluir:

```text
RequiredCapabilitySet
```

---

# 117. Capability-aware resolution

Ejemplo:

```text
requires:
  platform.query.returning
```

Entonces una conexión incompatible no deberá seleccionarse.

---

# 118. Manager vs Capability System

El Manager consume resultados/capability providers.

No define qué significa `RETURNING`.

---

# 119. Connection availability

Puede distinguirse:

```text
DEFINED
AVAILABLE
UNAVAILABLE
DEGRADED
```

en diagnostics/resolution.

---

# 120. Availability source

Puede provenir de:

```text
driver availability
runtime requirements
health state
circuit breaker
pool state
topology state
```

No todo deberá calcularse dentro del Manager.

---

# 121. Connection selection failures

El Manager deberá producir errores claros.

Ejemplo:

```text
Requested connection:
analytics

Resolution:
failed

Reason:
connection is not configured
```

---

# 122. Exception hierarchy

Conceptualmente:

```text
ConnectionManagerException
├── ConnectionNotConfiguredException
├── DefaultConnectionNotConfiguredException
├── ConnectionAliasException
├── ConnectionAliasCycleException
├── ConnectionResolutionException
├── ConnectionResolutionRejectedException
├── ConnectionDefinitionException
└── ConnectionScopeException
```

---

# 123. No raw native exceptions

`ConnectionManager` no deberá exponer normalmente:

```text
PDOException
```

porque su función es resolución lógica.

---

# 124. Resolution trace

Para debugging podrá generarse:

```text
ConnectionResolutionTrace
```

---

# 125. Trace example

```text
Requested:
  intent = READ

Transaction affinity:
  none

Explicit identity:
  none

Scoped resolver:
  not applicable

Topology:
  selected replica-02

Final identity:
  replica-02
```

---

# 126. Resolution trace production policy

No deberá generarse con alto coste para todas las operaciones salvo:

```text
debug mode
sampling
diagnostic request
```

---

# 127. Manager cache

Podrá cachear:

```text
canonical aliases
validated definitions
stateless logical connection descriptors
```

---

# 128. Manager must not cache

No deberá cachear globalmente:

```text
current request resolution
tenant-specific definition
active transaction lease
NativeStatement
ResultCursor
```

---

# 129. Scoped cache

Si una resolución es costosa y request-specific podrá existir:

```text
ConnectionResolutionCache
```

dentro del `DatabaseContext`.

---

# 130. Tenant-scoped resolution cache

Ejemplo:

```text
DatabaseContext A
  tenant = A
  resolved connection = tenant-A-db
```

desaparece al finalizar A.

---

# 131. Connection manager in CLI

El mismo Manager deberá funcionar en:

```text
HTTP
CLI
Queue
Scheduler
Testing
```

---

# 132. ExecutionScope

El Manager no deberá asumir que todo scope es HTTP.

---

# 133. Testing overrides

Tests podrán proporcionar:

```text
ScopedConnectionOverride
```

sin mutar configuración global.

---

# 134. Example test override

```text
Production definition:
primary → PostgreSQL

Test scope:
primary → SQLite memory
```

El override deberá vivir en el scope de test.

---

# 135. Parallel tests

Los overrides deberán estar aislados para permitir:

```text
parallel test execution
```

---

# 136. No global test mutation

Evitar:

```php
Config::set('database.primary', ...);
```

como mecanismo interno principal cuando hay workers/tests concurrentes.

---

# 137. Manager extensibility

Extension points posibles:

```text
ConnectionResolverInterface
ConnectionDefinitionProviderInterface
ConnectionAliasProviderInterface
ConnectionFactoryInterface
ConnectionResolutionPolicyInterface
```

---

# 138. Extension stability

Los contratos públicos de extensión deberán permanecer:

```text
small
typed
documented
versioned
```

---

# 139. No service locator extension API

Incorrecto:

```php
public function resolve(Container $container, array $context);
```

Preferir dependencias explícitas.

---

# 140. No arbitrary arrays

Evitar:

```php
resolve(array $options)
```

para APIs arquitectónicas importantes.

Preferir:

```text
ConnectionResolutionRequest
ConnectionRequirement
ConnectionDefinition
```

---

# 141. Immutability

Preferir objetos inmutables para:

```text
ConnectionIdentity
ConnectionDefinition
ConnectionRequirement
ConnectionResolution
ConnectionAlias
```

---

# 142. Mutable state

Cuando sea necesario deberá estar en:

```text
ConnectionLease
ConnectionContext
DatabaseContext
ConnectionStateTracker
```

con ownership explícito.

---

# 143. ConnectionManager concurrency

Un Manager singleton deberá ser:

```text
reentrant
concurrency-safe
free of request-local mutable state
```

---

# 144. FrankenPHP

Modelo recomendado:

```text
FrankenPHP Worker
      │
      ├── ConnectionManager singleton
      ├── frozen definitions
      ├── resolvers
      ├── factories
      │
      ├── Request A → DatabaseContext A
      ├── Request B → DatabaseContext B
      └── Request C → DatabaseContext C
```

---

# 145. RoadRunner

Misma arquitectura:

```text
Worker-persistent manager
+
execution-scoped state
```

---

# 146. OpenSwoole

Especialmente importante:

```text
Coroutine A
Coroutine B
Coroutine C
```

pueden usar el mismo Manager.

Por tanto el Manager no deberá mantener:

```text
$currentConnection
$currentTenant
$currentTransaction
```

---

# 147. Fiber/coroutine context

VoltStack podrá proporcionar un `ExecutionScopeAccessor`.

Pero deberá evitar depender de globals tradicionales.

---

# 148. Context access

Idealmente las dependencias internas reciben explícitamente el contexto necesario.

El accessor deberá utilizarse sólo donde simplifique correctamente la integración.

---

# 149. Manager reset

Un Manager correctamente diseñado podría no necesitar reset por request.

---

# 150. Why?

Porque su estado persistente será:

```text
immutable/frozen
```

El estado request-specific vive fuera.

---

# 151. Important consequence

Si `ConnectionManager::reset()` necesita limpiar:

```text
current tenant
current transaction
current connection
```

es señal de que probablemente existe state ownership incorrecto.

---

# 152. Worker reset

Podría existir reset sólo para:

```text
diagnostic caches
dynamic application-level reload
shutdown
```

no como mecanismo para corregir leaks de diseño.

---

# 153. Manager lifecycle

```text
Application Bootstrap
       │
       ▼
Create ConnectionManager
       │
       ▼
Inject Frozen Definitions
       │
       ▼
Inject Factories/Resolvers
       │
       ▼
Use across execution scopes
       │
       ▼
Application/Worker Shutdown
```

---

# 154. Container registration

Conceptualmente:

```php
$container->singleton(
    ConnectionManagerInterface::class,
    ConnectionManager::class
);
```

sólo si cumple las invariantes de stateless/scoped separation.

---

# 155. Scoped dependencies

El singleton no deberá capturar accidentalmente:

```text
request-scoped DatabaseContext
```

durante construcción.

---

# 156. Scope inversion

Prohibido:

```text
Singleton
   │
   ▼
Request-scoped mutable object
```

como dependencia persistente directa.

---

# 157. Context provider

Si necesita acceder al contexto actual, deberá utilizar un boundary diseñado para resolución contextual segura.

---

# 158. Prefer explicit context

Para APIs internas críticas:

```php
$manager->resolve($requirement, $databaseContext);
```

puede ser más seguro que estado implícito.

---

# 159. DX facade

La API pública podrá ocultar ese detalle.

Ejemplo:

```php
DB::connection('analytics');
```

pero internamente el scope seguirá siendo explícito.

---

# 160. Manager and Facade

```text
DB Facade
   │
   ▼
Database API
   │
   ▼
ConnectionManager
```

Facade no almacenará la Connection actual estáticamente.

---

# 161. Fluent API

Puede existir:

```php
DB::connection('analytics')
    ->table('events')
    ->get();
```

mediante un:

```text
DatabaseConnectionContext
```

o Query API configurada con `ConnectionRequirement`.

---

# 162. Important fluent API rule

El objeto devuelto por:

```php
DB::connection('analytics')
```

no tiene que ser el low-level `ConnectionInterface`.

Puede ser:

```text
DatabaseConnectionSelection
```

---

# 163. DatabaseConnectionSelection

Conceptualmente:

```php
final readonly class DatabaseConnectionSelection
{
    public function __construct(
        public ConnectionIdentity $identity
    ) {}
}
```

y puede alimentar al Query API.

---

# 164. Why this matters

Evita contaminar `ConnectionInterface` con:

```text
table()
select()
transaction()
schema()
```

sólo para obtener una API Laravel-like.

---

# 165. Public DX architecture

```text
DB::connection('analytics')
        │
        ▼
DatabaseConnectionSelection
        │
        ▼
Query API
        │
        ▼
ConnectionRequirement
        │
        ▼
ConnectionManager
```

---

# 166. Laravel-like surface, VoltStack internals

La sintaxis pública puede ser familiar sin copiar las dependencias internas.

---

# 167. ConnectionManager and raw SQL

Raw SQL deberá seguir:

```text
Raw SQL API
    │
    ▼
Execution Engine
    │
    ▼
Connection Requirement
    │
    ▼
ConnectionManager
```

No:

```text
ConnectionManager->query($sql)
```

como responsabilidad arquitectónica primaria.

---

# 168. Administration operations

Operaciones administrativas podrán utilizar un:

```text
ConnectionIntent::ADMIN
```

y requerir una conexión con capabilities apropiadas.

---

# 169. Migration operations

Migraciones podrán solicitar:

```text
ConnectionIntent::MIGRATION
```

---

# 170. Schema operations

Schema podrá solicitar:

```text
ConnectionIntent::SCHEMA
```

---

# 171. Role validation

El Manager/Resolver podrá verificar compatibilidad básica:

```text
READ request
→ READ_ONLY acceptable

WRITE request
→ READ_ONLY rejected
```

---

# 172. Capability validation

La validación profunda podrá delegarse a un:

```text
ConnectionCapabilityMatcher
```

---

# 173. ConnectionCapabilityMatcher

Responsabilidad:

```text
Connection Requirement
+
Effective Capability Set
        │
        ▼
Compatible / Incompatible
```

---

# 174. Effective capabilities

Pueden resultar de:

```text
Driver Capabilities
+
Platform Capabilities
+
Connection Role Restrictions
+
Runtime Restrictions
```

---

# 175. Manager should consume effective capabilities

No deberá reconstruir manualmente estas reglas.

---

# 176. Availability vs capability

Una conexión puede estar:

```text
AVAILABLE
```

pero no ser:

```text
COMPATIBLE
```

con una operación determinada.

---

# 177. Example

```text
Replica PostgreSQL
Status: AVAILABLE
READ: supported
WRITE: prohibited
```

---

# 178. Resolution result detail

`ConnectionResolution` podrá incluir:

```text
selected identity
definition
resolution source
effective role
compatibility
diagnostic metadata
```

sin incluir secrets.

---

# 179. Resolution source

Ejemplos:

```text
DEFAULT
EXPLICIT
ALIAS
TOPOLOGY
TENANT
SHARD
TRANSACTION_AFFINITY
TEST_OVERRIDE
```

---

# 180. Useful diagnostics

Esto permitirá explicar:

> ¿Por qué VoltStack utilizó esta conexión?

---

# 181. Explainability

Especialmente importante para:

```text
replicas
multitenancy
sharding
testing
```

---

# 182. ConnectionManager performance

La resolución ocurre potencialmente en hot paths.

Debe minimizar:

```text
reflection
array normalization
configuration parsing
service-container lookups
string manipulation
```

---

# 183. Precompiled definitions

Las definiciones deberán estar listas antes del runtime normal.

---

# 184. Canonical IDs

Aliases y nombres podrán convertirse a:

```text
ConnectionIdentity
```

canónica.

---

# 185. Resolution cache

Resoluciones puramente estáticas podrán cachearse.

---

# 186. Scoped dynamic resolution

Resoluciones dependientes del contexto sólo deberán cachearse dentro del contexto apropiado.

---

# 187. No cross-tenant cache key omission

Incorrecto:

```text
cache key = "primary"
```

para una resolución tenant-dependent.

---

# 188. Correct scoped cache

Puede ser:

```text
DatabaseContext
└── primary → tenant-specific resolution
```

sin necesidad de tenant ID global en la cache.

---

# 189. Manager memory growth

Un Manager persistente no deberá acumular indefinidamente:

```text
tenant definitions
dynamic aliases
per-request traces
failed connection identities
```

---

# 190. Bounded caches

Si se introduce cache persistente, deberá ser:

```text
bounded
observable
evictable
```

---

# 191. Architecture testing

Tests deberán verificar:

```text
ConnectionManager does not depend on ORM
ConnectionManager does not depend on PDO
ConnectionManager does not own pool implementation
ConnectionManager does not own transaction state
ConnectionManager does not store tenant state
```

---

# 192. Unit tests

Deberán cubrir:

```text
default resolution
named resolution
unknown connection
alias resolution
alias cycles
definition creation
factory delegation
role rejection
capability rejection
```

---

# 193. Persistent runtime tests

```text
Request A
→ dynamic resolution A

Request ends

Request B
→ resolution B

assert no A state remains in manager
```

---

# 194. Concurrent resolution tests

```text
Context A → Tenant A
Context B → Tenant B

same ConnectionManager

assert:
A resolves A
B resolves B
```

---

# 195. Transaction affinity tests

```text
Transaction pinned to primary

request connection resolution

assert primary reused
```

---

# 196. Transaction conflict tests

```text
Transaction pinned to primary

explicit analytics requested

assert conflict
```

---

# 197. Test override isolation

```text
Test A → SQLite override
Test B → PostgreSQL override

parallel execution

assert no cross-test contamination
```

---

# 198. Manager conformance

Alternative implementations de `ConnectionManagerInterface` deberán pasar una suite común si el contrato se hace extensible.

---

# 199. Suggested namespaces

```text
VoltStack\Quantum\Database\Connection
│
├── Contract
│   ├── ConnectionManagerInterface.php
│   ├── ConnectionFactoryInterface.php
│   ├── ConnectionResolverInterface.php
│   ├── ConnectionDefinitionProviderInterface.php
│   └── ConnectionResourceProviderInterface.php
│
├── Manager
│   └── ConnectionManager.php
│
├── Definition
│   ├── ConnectionDefinition.php
│   ├── ConnectionDefinitionRegistry.php
│   └── ConnectionDefinitionFingerprint.php
│
├── Identity
│   └── ConnectionIdentity.php
│
├── Alias
│   ├── ConnectionAlias.php
│   └── ConnectionAliasRegistry.php
│
├── Factory
│   └── ConnectionFactory.php
│
├── Resolver
│   ├── ConnectionResolver.php
│   ├── ConnectionResolutionRequest.php
│   ├── ConnectionResolution.php
│   ├── ConnectionResolutionReason.php
│   └── ConnectionResolutionPipeline.php
│
├── Requirement
│   ├── ConnectionRequirement.php
│   └── ConnectionIntent.php
│
├── Capability
│   └── ConnectionCapabilityMatcher.php
│
├── Scope
│   ├── ScopedConnectionDefinitionProvider.php
│   └── ScopedConnectionOverride.php
│
├── Diagnostics
│   └── ConnectionResolutionTrace.php
│
└── Exception
```

---

# 200. Dependency matrix

| Componente | Puede depender de ConnectionManager | ConnectionManager puede depender de él |
|---|---:|---:|
| Query Executor | Sí | No |
| ORM | Indirectamente | No |
| Schema | Indirectamente | No |
| Migration | Indirectamente | No |
| Transaction Coordinator | Sí/coordina | Sólo contratos contextuales |
| ConnectionFactory | No | Sí |
| ConnectionResolver | No | Sí |
| DefinitionRegistry | No | Sí |
| DriverRegistry | No directo preferido | Indirecto |
| ConnectionPool | No | Sólo mediante Resource Provider |
| Multitenancy | Sí mediante integración | No directo |
| Telemetry | Adapter | Port opcional |
| HTTP | Sí indirectamente | No |

---

# 201. ConnectionManager invariants

## DB-CM-001

`ConnectionManager` no es una Connection.

## DB-CM-002

`ConnectionManager` no es Driver Registry.

## DB-CM-003

`ConnectionManager` no es Connection Pool.

## DB-CM-004

`ConnectionManager` no es Query Executor.

## DB-CM-005

`ConnectionManager` no es Transaction Manager.

## DB-CM-006

`ConnectionManager` no implementará ORM.

## DB-CM-007

El Manager no abrirá conexiones físicas al resolver una conexión lógica salvo petición explícita de infraestructura.

## DB-CM-008

La adquisición física será lazy por defecto.

## DB-CM-009

Las definiciones globales serán inmutables/frozen después de bootstrap.

## DB-CM-010

El Manager no almacenará el tenant actual.

## DB-CM-011

El Manager no almacenará la transaction actual.

## DB-CM-012

El Manager no almacenará leases request-scoped.

## DB-CM-013

El Manager podrá ser singleton sólo si es seguro para workers persistentes.

## DB-CM-014

Las definiciones dinámicas serán scoped.

## DB-CM-015

Multitenancy se integrará mediante ports/resolvers, no dependencia directa.

## DB-CM-016

Aliases deberán resolverse a identidades canónicas.

## DB-CM-017

Los ciclos de aliases deberán rechazarse.

## DB-CM-018

Los nombres de conexión no determinarán roles implícitamente.

## DB-CM-019

Los secretos no formarán parte de ConnectionIdentity.

## DB-CM-020

Los secretos no aparecerán en diagnostics.

## DB-CM-021

La resolución deberá ser determinista.

## DB-CM-022

La afinidad transaccional tendrá precedencia sobre routing incompatible.

## DB-CM-023

Los conflictos de afinidad no deberán resolverse silenciosamente.

## DB-CM-024

Una resolución tenant-specific no podrá cachearse globalmente sin aislamiento correcto.

## DB-CM-025

El Manager no acumulará estado dinámico ilimitado.

## DB-CM-026

El Manager consumirá capabilities; no reimplementará Platform semantics.

## DB-CM-027

El Manager consumirá decisiones de Topology; no será Topology Engine.

## DB-CM-028

La API pública deberá permanecer pequeña.

## DB-CM-029

Las APIs internas usarán value objects tipados en lugar de arrays arbitrarios.

## DB-CM-030

La API Laravel-like no obligará a contaminar el contrato low-level de Connection.

---

# 202. Anti-pattern — Manager como PDO registry

```php
final class ConnectionManager
{
    private array $pdoConnections = [];
}
```

sin lifecycle/pooling/scoping.

**Prohibido.**

---

# 203. Anti-pattern — current connection

```php
$manager->setCurrentConnection('analytics');
```

como estado global mutable.

**No deberá ser el modelo base.**

Preferir:

```text
ConnectionRequirement
```

o:

```text
DatabaseConnectionSelection
```

scoped/inmutable.

---

# 204. Anti-pattern — current tenant

```php
$manager->useTenant($tenant);
```

sobre un singleton persistente.

**Prohibido.**

---

# 205. Anti-pattern — Manager executes SQL

```php
$manager->query(
    'SELECT * FROM users'
);
```

como responsabilidad principal.

**Prohibido.**

---

# 206. Anti-pattern — Manager creates PDO

```php
new PDO(...)
```

dentro del Manager.

**Prohibido.**

Eso pertenece al Driver.

---

# 207. Anti-pattern — Manager contains pooling

```text
ConnectionManager
├── idle resources
├── active resources
├── wait queue
└── max connections
```

**Prohibido.**

Eso pertenece al Pool subsystem.

---

# 208. Anti-pattern — runtime-specific branches

```php
if ($runtime === 'frankenphp') {
    // ...
} elseif ($runtime === 'roadrunner') {
    // ...
}
```

dentro del Manager.

**Prohibido.**

---

# 209. Anti-pattern — dynamic global registry

```text
every request
→ register tenant DB globally
```

**Prohibido.**

---

# 210. Anti-pattern — hidden fallback

Si se solicita:

```text
analytics
```

y no existe, el Manager no deberá usar silenciosamente:

```text
default
```

salvo una policy explícita y visible.

---

# 211. Strict resolution

Por defecto:

```text
explicit unknown connection
→ exception
```

---

# 212. Default only when unspecified

```text
connection()
→ default

connection('unknown')
→ error
```

---

# 213. Anti-pattern — fallback from primary to replica for write

Nunca:

```text
primary unavailable
→ execute write on replica
```

sin topology/failover semantics explícitas.

---

# 214. Failure ownership

ConnectionManager reporta:

```text
resolution failure
```

Connection infrastructure reporta:

```text
acquisition failure
```

Driver reporta:

```text
native communication failure
```

Resilience decide:

```text
retry/failover/reject
```

---

# 215. Error boundary model

```text
Native Failure
     │
     ▼
Driver Failure
     │
     ▼
Connection Acquisition Failure
     │
     ▼
Resolution/Execution Context
     │
     ▼
Resilience Policy
```

---

# 216. Manager startup validation

Durante bootstrap podrá validarse:

```text
default exists
aliases valid
driver IDs registered
definitions structurally valid
roles valid
```

sin conectarse al servidor.

---

# 217. Offline validation

Debe ser posible ejecutar:

```text
config validation
container compilation
static architecture tests
```

sin Database disponible.

---

# 218. Runtime validation

Aspectos que requieren servidor:

```text
credentials valid
server reachable
server version
actual platform capabilities
```

se resolverán posteriormente.

---

# 219. Lazy server discovery

Server version/capabilities pueden descubrirse al abrir la primera conexión.

---

# 220. Platform binding

Una definición puede declarar:

```text
platform hint
```

pero la Platform efectiva puede validarse con información real del servidor.

---

# 221. Connection resolution without server access

Resolver:

```text
primary
```

no deberá necesitar conocer la versión PostgreSQL.

---

# 222. Separation benefit

Esto mantiene:

```text
ConnectionManager
```

rápido y predecible.

---

# 223. Public API example

```php
DB::connection('analytics')
    ->table('events')
    ->where('type', 'purchase')
    ->get();
```

Arquitectura interna:

```text
DB Facade
   │
   ▼
Connection Selection
   │
   ▼
Query Builder
   │
   ▼
Query Model / AST
   │
   ▼
Planner
   │
   ▼
ConnectionRequirement
   │
   ▼
ConnectionManager
   │
   ▼
Logical Connection
   │
   ▼
Executor
```

---

# 224. Transaction example

```php
DB::transaction(function () {
    // ...
});
```

Internamente:

```text
TransactionManager
       │
       ▼
ConnectionManager
       │
       ▼
Logical Connection
       │
       ▼
Acquire Lease A
       │
       ▼
TransactionContext
       │
       └── pin Lease A
```

Subsequent queries:

```text
TransactionContext
       │
       ▼
Lease A
```

sin nueva resolución incompatible.

---

# 225. Multitenancy example

Con paquete opcional:

```text
TenantContext
      │
      ▼
Tenant Database Adapter
      │
      ▼
ScopedConnectionDefinitionProvider
      │
      ▼
ConnectionManager
      │
      ▼
Tenant Logical Connection
```

El core Database no necesita conocer el modelo `Tenant`.

---

# 226. Sharding example

```text
Query/Domain determines ShardKey
        │
        ▼
Shard Router
        │
        ▼
Connection Target
        │
        ▼
ConnectionManager
```

---

# 227. Testing example

```text
Test ExecutionScope
      │
      ▼
Scoped Override
      │
      ▼
primary → sqlite-memory
      │
      ▼
ConnectionManager
```

Sin modificar el registry global.

---

# 228. Runtime architecture

```text
┌───────────────────────────────────────────────┐
│ Worker                                        │
│                                               │
│ ConnectionManager                            │
│ ├── Frozen Definition Registry               │
│ ├── Alias Registry                           │
│ ├── Connection Factory                       │
│ └── Resolver Infrastructure                  │
│                                               │
│ ┌────────────────┐    ┌────────────────┐      │
│ │ Execution A    │    │ Execution B    │      │
│ │ DatabaseCtx A  │    │ DatabaseCtx B  │      │
│ │ Scoped Def A   │    │ Scoped Def B   │      │
│ │ Lease A        │    │ Lease B        │      │
│ └────────────────┘    └────────────────┘      │
└───────────────────────────────────────────────┘
```

---

# 229. Manager architecture summary

```text
ConnectionManager
│
├── Static Configuration
│   ├── Definitions
│   ├── Aliases
│   └── Default Identity
│
├── Resolution
│   ├── Explicit
│   ├── Default
│   ├── Scoped
│   └── Integration
│
├── Creation
│   └── Logical Connection Factory
│
└── Diagnostics
```

No contiene:

```text
PDO
UnitOfWork
Query AST
Pool queue
current tenant
current transaction
```

---

# 230. Architectural equation

```text
ConnectionManager
=
Connection Definition Coordination
+
Logical Resolution
+
Logical Connection Creation
+
Scoped Integration Boundary
```

No:

```text
ConnectionManager
=
Everything related to databases
```

---

# 231. Master rule

> `ConnectionManager` decide qué conexión lógica debe utilizarse; no decide cómo ejecutar SQL ni mantiene el recurso físico como estado global.

---

# 232. Implementation priority

Orden recomendado:

```text
1. ConnectionIdentity
2. ConnectionDefinition
3. ConnectionDefinitionRegistry
4. ConnectionAliasRegistry
5. ConnectionFactoryInterface
6. ConnectionResolverInterface
7. ConnectionResolutionRequest
8. ConnectionResolution
9. ConnectionManagerInterface
10. ConnectionManager
11. Scoped Definition Provider
12. Capability Matcher
13. Resolution Diagnostics
14. Persistent Runtime Tests
15. Concurrency Tests
```

---

# 233. Architecture acceptance criteria

La implementación se considerará correcta cuando:

```text
named connections resolve deterministically
default connection resolves correctly
unknown explicit connections fail
aliases cannot cycle
logical resolution does not open PDO
dynamic definitions remain scoped
tenant state does not leak
transaction affinity cannot be violated
manager works without pooling
manager works without multitenancy
manager works without ORM
same manager survives multiple requests safely
parallel contexts remain isolated
```

---

# 234. Final architecture

```text
                     Consumers
                         │
                         ▼
               Connection Requirement
                         │
                         ▼
                 ConnectionManager
                         │
       ┌─────────────────┼──────────────────┐
       │                 │                  │
       ▼                 ▼                  ▼
 Definition Registry  Resolver       Scoped Providers
       │                 │                  │
       └─────────────────┼──────────────────┘
                         ▼
                 ConnectionDefinition
                         │
                         ▼
                  ConnectionFactory
                         │
                         ▼
                 Logical Connection
                         │
                         ▼
             Resource Acquisition Layer
                         │
                  ┌──────┴──────┐
                  ▼             ▼
                Pool          Driver
                  │             │
                  └──────┬──────┘
                         ▼
                  ConnectionLease
                         │
                         ▼
                 Native Connection
```

---

# 235. Conclusión

El `ConnectionManager` de VoltStack será un **orquestador de resolución**, no un contenedor global de conexiones físicas.

Esta decisión permite separar:

```text
configuration
resolution
logical connection
physical acquisition
pooling
transactions
topology
runtime state
```

y evita uno de los problemas más frecuentes de arquitecturas Database bajo runtimes persistentes:

```text
singleton manager
+
mutable request state
=
cross-request leakage
```

VoltStack utilizará en cambio:

```text
Persistent Manager
+
Immutable Definitions
+
Scoped Context
+
Explicit Leases
=
Persistent Runtime Safety
```

La misma arquitectura funcionará para:

```text
traditional PHP
FrankenPHP
RoadRunner
OpenSwoole
CLI
Queue workers
parallel tests
```

sin cambiar el modelo conceptual del Connection System.

---

# 236. Siguiente documento

El siguiente documento será:

```text
13_DATABASE_CONNECTION_CONFIGURATION_AND_RESOLUTION.md
```

y deberá profundizar en:

```text
connection configuration model
configuration schema
connection definitions
connection identities
connection aliases
default connection
configuration normalization
configuration validation
environment resolution
credential references
secret resolution
driver selection
platform hints
connection roles
timeouts
TLS
native options
session profiles
pool configuration references
read/write definitions
dynamic/scoped definitions
connection resolution requests
resolution precedence
resolution policies
capability requirements
resolution diagnostics
configuration fingerprints
configuration generations
persistent runtime safety
```

manteniendo la separación:

```text
Raw Configuration
       │
       ▼
Configuration Normalization
       │
       ▼
Validated ConnectionDefinition
       │
       ▼
Connection Resolution
       │
       ▼
Logical Connection
```

y evitando:

```text
env/config lookup
```

dentro de Driver, Connection o Query hot paths.
```

Con `12_DATABASE_CONNECTION_MANAGER.md` queda fijada una frontera importante: **el Manager puede vivir durante todo el worker, pero el tenant, la transacción y los leases no viven dentro de él**. Esto prepara directamente el diseño de `13_DATABASE_CONNECTION_CONFIGURATION_AND_RESOLUTION.md`.