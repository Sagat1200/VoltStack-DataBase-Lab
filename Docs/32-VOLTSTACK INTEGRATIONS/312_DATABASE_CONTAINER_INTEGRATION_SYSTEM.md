# 312_DATABASE_CONTAINER_INTEGRATION_SYSTEM.md

# VoltStack Quantum Database
## Database Container Integration System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 312 — Database Container Integration System  
**Bloque:** 32 — VoltStack Integration  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `311_DATABASE_FRAMEWORK_INTEGRATION_ARCHITECTURE.md`  
**Siguiente documento:** `313_DATABASE_CONFIG_INTEGRATION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura mediante la cual `VoltStack/Quantum/Database` se registra, construye, resuelve, comparte, aísla y destruye utilizando el **Service Container de VoltStack**.

El Container será responsable de ensamblar el grafo de dependencias de Database, pero no deberá convertirse en propietario de las semánticas internas del subsistema.

La regla central será:

> **El lifetime asignado por el Service Container a cada servicio Database deberá corresponder exactamente con su lifetime semántico; ningún servicio mutable perteneciente a una operación podrá convertirse accidentalmente en singleton de proceso.**

Formalmente:

```text
Container Lifetime
=
Semantic Lifetime
```

y nunca:

```text
Container Convenience
>
Database Isolation
```

Especialmente en:

```text
FrankenPHP
RoadRunner
OpenSwoole
queue workers
long-running CLI processes
```

un binding incorrecto puede provocar:

```text
cross-request state leakage
transaction leakage
tenant leakage
IdentityMap contamination
UnitOfWork contamination
connection state leakage
memory growth
security boundary violations
```

Por ello, Container Integration es parte de la arquitectura de correctness de Database y no simplemente configuración de Dependency Injection.

---

# 2. Objetivos

El sistema deberá proporcionar:

1. registro determinista de servicios;
2. resolución mediante contratos;
3. scopes explícitos;
4. servicios shared sólo cuando sean seguros;
5. servicios operation-scoped cuando mantengan estado mutable;
6. factories para objetos dinámicos;
7. bootstrap ordenado;
8. registries congelados;
9. integración con extensiones;
10. resolución lazy cuando sea apropiada;
11. detección de dependencias circulares;
12. soporte para compiled container;
13. aliases públicos controlados;
14. testing overrides;
15. integración con persistent workers;
16. destrucción determinista de scopes;
17. diagnostics;
18. validación de lifetimes;
19. aislamiento concurrente;
20. compatibilidad con modo standalone.

---

# 3. Principio arquitectónico

El Container deberá ensamblar:

```text
Contracts
    ↓
Implementations
    ↓
Factories
    ↓
Scoped Services
    ↓
Database Runtime
```

pero no deberá decidir:

```text
SQL semantics
ORM semantics
transaction semantics
query planning
persistence semantics
driver protocol behavior
```

---

# 4. Container ≠ Database Manager

Debe mantenerse:

```text
Service Container
≠
Database Manager
```

El Container conoce:

```text
how to construct
how to resolve
how long to retain
how to destroy
```

Database conoce:

```text
what the service means
what invariants it preserves
how database operations behave
```

---

# 5. Container ≠ Service Locator

Aunque el Container permita:

```php
$container->get(EntityManager::class);
```

esto no significa que todo el código interno deba consultar el Container.

La regla preferida será:

```text
Container
    ↓
Constructor Injection
    ↓
Object Graph
```

No:

```text
Every Object
    ↓
Container::get()
```

---

# 6. Composition Root

El Container será utilizado principalmente desde el **Composition Root**.

Conceptualmente:

```text
DatabaseServiceProvider
        ↓
DatabaseContainerRegistrar
        ↓
Container
```

---

# 7. DatabaseServiceProvider

Se propone:

```php
final class DatabaseServiceProvider
{
    public function register(Container $container): void;

    public function boot(DatabaseBootstrapContext $context): void;
}
```

La API concreta dependerá del Container definitivo de VoltStack.

---

# 8. Register ≠ Boot

Deberá distinguirse:

```text
register()
```

de:

```text
boot()
```

`register()` define el grafo.

`boot()` ejecuta inicialización que requiere que el grafo ya exista.

---

# 9. Fases

```text
Container Creation
      ↓
Core Service Registration
      ↓
Database Registration
      ↓
Extension Registration
      ↓
Optional Integration Registration
      ↓
Dependency Validation
      ↓
Container Compilation
      ↓
Container Freeze
      ↓
Database Boot
      ↓
Runtime Ready
```

---

# 10. Registration phase

Durante registration podrán declararse:

```text
contracts
implementations
factories
aliases
decorators
scopes
tags
extension points
```

No deberán ejecutarse queries.

---

# 11. Boot phase

Durante boot podrán:

```text
validate configuration
freeze registries
compile metadata
validate drivers
validate extension graph
prepare immutable runtime structures
```

según configuración.

---

# 12. Boot ≠ Connect to every database

Por defecto:

```text
Framework Boot
≠
Open every DB connection
```

salvo que una política explícita de:

```text
readiness
warm-up
health verification
```

lo requiera.

---

# 13. Lifetimes fundamentales

VoltStack deberá distinguir al menos:

```text
SINGLETON
SCOPED
TRANSIENT
FACTORY
```

conceptualmente.

Podrán existir nombres diferentes en el Container real.

---

# 14. Singleton

Un singleton pertenece al proceso/container principal.

Sólo será válido cuando el servicio sea:

```text
immutable
stateless
thread/coroutine safe where applicable
free from operation-specific state
```

---

# 15. Scoped

Un servicio scoped pertenece a:

```text
OperationScope
```

Ejemplos:

```text
HTTP request
SPA action
queue job
CLI operation
scheduled task
RPC operation
```

---

# 16. Transient

Se crea una instancia por resolución.

Adecuado para objetos ligeros sin identidad operacional compartida.

---

# 17. Factory

Construye recursos cuyo lifetime no coincide directamente con una resolución simple.

Ejemplos:

```text
ConnectionFactory
EntityHydratorFactory
CompilerFactory
RepositoryFactory
```

---

# 18. Lifetime taxonomy

Se propone:

| Lifetime | Significado |
|---|---|
| `SHARED` | una instancia segura para todo el container/worker |
| `OPERATION_SCOPED` | una instancia por operación |
| `TRANSIENT` | nueva instancia por resolución |
| `RESOURCE_LEASED` | recurso prestado temporalmente |
| `FACTORY_MANAGED` | lifetime controlado por una factory/manager |
| `EXTERNAL` | lifecycle perteneciente a otro subsistema |

---

# 19. Shared Database services

Potencialmente compartibles:

```text
DatabaseConfiguration
DriverRegistry
DialectRegistry
PlatformRegistry
TypeRegistry
CompilerRegistry
ExtensionRegistry
CompiledMetadataRegistry
QueryNormalizationRules
OptimizationRuleRegistry
CapabilityDefinitionRegistry
```

siempre que sean inmutables tras bootstrap.

---

# 20. Operation-scoped Database services

Deberán ser scoped:

```text
DatabaseContext
EntityManager
UnitOfWork
IdentityMap
TransactionContext
QueryContext
PersistenceContext
HydrationSession
TenantDatabaseContext
```

cuando correspondan.

---

# 21. Factory-managed services

Ejemplos:

```text
Connection
PreparedStatement
ResultCursor
Stream
TransactionHandle
```

Su lifecycle deberá ser administrado por sus respectivos subsistemas.

---

# 22. Shared ≠ global mutable

Un servicio compartido deberá satisfacer:

```text
Shared(Service)
→
NoOperationMutableState(Service)
```

---

# 23. Regla de seguridad

Si existe duda razonable entre:

```text
SHARED
```

y:

```text
SCOPED
```

el diseño deberá favorecer inicialmente:

```text
SCOPED
```

hasta demostrar que compartirlo es seguro.

---

# 24. Scope fundamental

El scope general será:

```text
DatabaseOperationScope
```

No:

```text
HttpRequestScope
```

porque Database también opera fuera de HTTP.

---

# 25. Operation Scope

Conceptualmente:

```php
interface DatabaseOperationScope
{
    public function id(): OperationId;

    public function context(): DatabaseContext;

    public function close(): void;
}
```

---

# 26. Framework scope mapping

```text
HTTP Request
      ↓
Operation Scope

SPA Action
      ↓
Operation Scope

Queue Job
      ↓
Operation Scope

CLI Command
      ↓
Operation Scope
```

---

# 27. Scope hierarchy

Podrá existir:

```text
Worker Scope
    ↓
Operation Scope
        ↓
Transaction Scope?
            ↓
Query Scope?
```

pero no todos deberán ser Container scopes reales.

---

# 28. Semantic scope ≠ DI scope obligatorio

No toda frontera semántica requiere crear un child container.

Por ejemplo:

```text
QueryContext
```

puede pertenecer al OperationScope y crear subcontextos internamente.

---

# 29. Scope creation

Flujo:

```text
Framework receives operation
        ↓
Container creates operation scope
        ↓
Database lifecycle begins
        ↓
Database scoped services become resolvable
        ↓
Application executes
```

---

# 30. Scope destruction

```text
Application finishes
        ↓
Database finalization
        ↓
Transaction verification
        ↓
Resource cleanup
        ↓
Connection release/reset
        ↓
Scoped services disposed
        ↓
Container scope destroyed
```

---

# 31. Destruction order

La destrucción deberá respetar dependencias.

Ejemplo:

```text
Repository
    ↓
EntityManager
    ↓
UnitOfWork
    ↓
Connection Lease
```

No deberá destruirse primero un recurso todavía requerido por otro finalizer.

---

# 32. Scope finalization ≠ garbage collection

No deberá confiarse en:

```text
PHP object destruction eventually
```

para liberar correctamente recursos críticos.

---

# 33. Explicit finalization

Recursos como:

```text
transactions
streams
cursors
connection leases
temporary session state
```

deberán tener lifecycle explícito.

---

# 34. EntityManager binding

El Container podrá registrar:

```php
EntityManager::class
```

como:

```text
OPERATION_SCOPED
```

---

# 35. EntityManager resolution

Dentro de una misma operación:

```php
$a = $container->get(EntityManager::class);
$b = $container->get(EntityManager::class);
```

deberá cumplirse normalmente:

```text
a === b
```

para el EntityManager predeterminado del mismo contexto.

---

# 36. Across operations

Para:

```text
Operation A
Operation B
```

deberá cumplirse:

```text
EntityManager(A) !== EntityManager(B)
```

---

# 37. UnitOfWork binding

`UnitOfWork` deberá pertenecer al mismo scope semántico que su `EntityManager`.

---

# 38. IdentityMap binding

Asimismo:

```text
IdentityMap
```

no podrá ser singleton.

---

# 39. Critical invariant

```text
IdentityMap(Operation A)
∩
IdentityMap(Operation B)
=
∅
```

en términos de estado mutable.

---

# 40. DatabaseContext binding

`DatabaseContext` deberá representar el contexto efectivo de la operación.

Puede contener:

```text
operation identity
connection intent
tenant context
shard context
actor context
deadline
read/write intent
consistency requirements
```

según la operación.

---

# 41. DatabaseContext immutability

Idealmente deberá favorecer una representación:

```text
immutable
```

o derivaciones inmutables.

Ejemplo:

```php
$tenantContext = $context->withTenant($tenant);
```

en vez de mutación global.

---

# 42. Context binding

El contexto inicial podrá construirse desde:

```text
RuntimeOperationContext
```

aportado por Framework Integration.

---

# 43. No active scope

Resolver:

```php
EntityManager::class
```

sin un OperationScope válido deberá producir un error explícito.

Ejemplo:

```text
NoActiveDatabaseScopeException
```

---

# 44. No implicit global fallback

Incorrecto:

```text
No scope?
→ create global EntityManager
```

Esto ocultaría errores graves en persistent workers.

---

# 45. Facade resolution

Una facade como:

```php
DB::query(...)
```

deberá resolver el servicio correspondiente desde el scope actual.

---

# 46. Facade ≠ singleton

Debe mantenerse:

```text
Static Facade Syntax
≠
Static Service Lifetime
```

---

# 47. Repository resolution

Repositorios podrán resolverse mediante Container.

Ejemplo:

```php
final class UserController
{
    public function __construct(
        private UserRepository $users,
    ) {}
}
```

---

# 48. Repository lifetime

Un Repository que mantenga referencia al EntityManager scoped deberá ser:

```text
OPERATION_SCOPED
```

o transient dentro del mismo scope.

Nunca process singleton con EntityManager capturado.

---

# 49. Repository factory

Podrá existir:

```text
RepositoryFactory
```

compartida si no retiene estado scoped.

Flujo:

```text
RepositoryFactory(shared)
        ↓
EntityManager(scoped)
        ↓
Repository(scoped/transient)
```

---

# 50. Model API integration

El API estilo Active Record:

```php
User::find(10);
```

deberá utilizar:

```text
ModelContextResolver
```

---

# 51. ModelContextResolver

Conceptualmente:

```php
interface ModelContextResolver
{
    public function resolve(): ModelRuntimeContext;
}
```

---

# 52. Resolver lifetime

El resolver puede ser shared si únicamente delega al proveedor de scope seguro.

Pero no deberá almacenar:

```text
current EntityManager
```

como mutable process state.

---

# 53. Model runtime resolution

```text
User::find(10)
      ↓
ModelContextResolver
      ↓
Current Operation Scope
      ↓
EntityManager
      ↓
Repository / Entity Query
```

---

# 54. Query Builder

Un Query Builder no deberá resolver dependencias desde el Container durante cada operación interna.

Preferible:

```text
Container
→ QueryBuilderFactory
→ QueryBuilder
```

---

# 55. QueryBuilderFactory

Podrá ser shared si:

```text
stateless
+
immutable dependencies
```

y recibe el contexto scoped cuando construye el builder.

---

# 56. Query Builder state

El Query Builder en sí suele contener:

```text
query construction state
```

por lo que deberá ser:

```text
TRANSIENT
```

o creado explícitamente.

---

# 57. Query Builder ≠ Container service singleton

Incorrecto:

```text
singleton SelectQueryBuilder
```

si el builder es mutable.

---

# 58. Compiler services

Los SQL Compilers podrán ser shared si son:

```text
stateless
immutable
platform-specific but context-independent
```

---

# 59. Compiler state

Información de una query concreta deberá viajar mediante:

```text
CompilationContext
```

no almacenarse en el Compiler singleton.

---

# 60. Optimizer

Igual principio:

```text
Optimizer(shared)
+
OptimizationContext(per operation/query)
```

---

# 61. Planner

```text
Planner(shared)
+
PlanningContext(scoped/query)
```

si la implementación lo permite.

---

# 62. Metadata

Metadata compilada deberá favorecer:

```text
shared immutable instances
```

---

# 63. Metadata mutation

No deberá permitirse:

```text
request modifies EntityMetadata
```

una vez congelado el registry.

---

# 64. DriverRegistry

Será típicamente:

```text
SHARED
FROZEN
```

tras bootstrap.

---

# 65. Driver instances

Debe distinguirse:

```text
Driver Definition
≠
Driver Runtime Resource
```

Un driver stateless podría ser shared.

Una conexión no.

---

# 66. Driver binding

Ejemplo:

```text
DriverRegistry
    ↓
DriverFactory
    ↓
Driver
    ↓
ConnectionFactory
```

---

# 67. ConnectionManager

El lifetime del `ConnectionManager` dependerá de su diseño.

Si administra infraestructura compartida:

```text
pools
driver definitions
endpoint registries
```

podrá tener una parte shared.

Pero el estado de leases de una operación deberá permanecer scoped.

---

# 68. Split recomendado

En lugar de un manager gigante:

```text
Shared Connection Infrastructure
├── ConnectionFactory
├── PoolRegistry
├── EndpointRegistry
└── DriverRegistry

Scoped Connection Context
├── leases
├── routing affinity
├── transaction pinning
└── sticky state
```

---

# 69. Connection pool

Un pool podrá ser:

```text
worker/process shared
```

si el runtime y driver lo permiten.

---

# 70. Connection lease

El lease será:

```text
operation/transaction bounded
```

según uso.

---

# 71. Pool ≠ connection

Debe mantenerse:

```text
ConnectionPool
≠
Connection
≠
ConnectionLease
```

---

# 72. TransactionManager

Si es un coordinador stateless:

```text
TransactionManager
```

podrá ser shared.

---

# 73. TransactionContext

Siempre será scoped.

```text
TransactionManager(shared)
+
TransactionContext(scoped)
```

---

# 74. Transaction binding error

Incorrecto:

```text
singleton TransactionContext
```

---

# 75. Cache integration binding

Database podrá registrar:

```text
DatabaseCacheProvider
```

contra:

```text
VoltStackCacheAdapter
```

si Quantum Cache está disponible.

---

# 76. Optional cache

Sin Cache:

```text
DatabaseCacheProvider
→ NullDatabaseCacheProvider
```

cuando la feature lo permita.

---

# 77. Mandatory cache feature

Si una feature configurada requiere shared cache:

```text
enabled
+
provider missing
→
Container/Configuration validation failure
```

---

# 78. Event dispatcher binding

Podrá existir:

```text
DatabaseEventDispatcher
```

implementado mediante bridge hacia Event System.

---

# 79. Event bridge lifetime

Podrá ser shared si no almacena contexto mutable y recibe contexto en cada dispatch.

---

# 80. Telemetry binding

Ejemplo:

```text
DatabaseTracer
→ VoltStackTelemetryAdapter
```

---

# 81. Current trace

El adapter shared no deberá almacenar:

```text
$currentSpan
```

como propiedad global.

Debe resolverlo desde contexto scoped o recibirlo como argumento.

---

# 82. Clock

`Clock` podrá ser shared.

En producción:

```text
SystemClock
```

En testing:

```text
TestClock
```

---

# 83. Actual performance timing

Para medición real de performance podrá requerirse:

```text
MonotonicClock
```

distinto de un reloj lógico controlado.

---

# 84. Secret provider

El Container podrá registrar:

```text
SecretProvider
```

contra el proveedor del framework.

---

# 85. Secret lifetime

Secrets deberán materializarse lo más tarde posible y conservarse el menor tiempo razonable.

---

# 86. Credential provider

Puede ser:

```text
shared coordinator
```

si no conserva credentials mutables inseguros.

---

# 87. Tenant provider

Cuando Multitenancy esté instalado:

```text
TenantContextProvider
```

deberá resolver el tenant del scope actual.

---

# 88. Tenant provider ≠ tenant singleton

Nunca:

```text
TenantContextProvider::$currentTenant
```

como estado global compartido.

---

# 89. Authentication actor provider

Igualmente:

```text
DatabaseActorProvider
```

deberá obtener el actor desde contexto scoped.

---

# 90. Optional bindings

El Container deberá permitir:

```text
conditional bindings
```

basados en capacidades instaladas.

Ejemplo:

```text
if Quantum/Telemetry installed
    bind DatabaseTracer → TelemetryAdapter
else
    bind DatabaseTracer → NullTracer
```

---

# 91. Conditional binding ≠ runtime branching everywhere

La decisión deberá resolverse preferentemente durante bootstrap.

No:

```php
if (class_exists(...))
```

en cada query.

---

# 92. Extension providers

Plugins Database podrán proporcionar:

```text
DatabaseExtensionProvider
```

---

# 93. Extension provider

Ejemplo conceptual:

```php
interface DatabaseExtensionProvider
{
    public function register(
        DatabaseExtensionRegistrationContext $context,
    ): void;
}
```

---

# 94. Extension registration

Podrá registrar:

```text
driver
dialect
compiler
type
query extension
ORM extension
capability provider
integration adapter
```

---

# 95. Extension provider ≠ arbitrary container mutation

Se recomienda proporcionar un API restringido.

No necesariamente entregar el Container completo.

---

# 96. Why

Esto evita que un plugin:

```text
replace unrelated framework services
mutate internal bindings
break lifecycle guarantees
```

accidentalmente.

---

# 97. DatabaseExtensionRegistrationContext

Podrá exponer:

```text
DriverRegistryBuilder
DialectRegistryBuilder
TypeRegistryBuilder
CompilerRegistryBuilder
CapabilityProviderRegistryBuilder
```

antes del freeze.

---

# 98. Registry builders

Durante bootstrap:

```text
MUTABLE
```

Después:

```text
FROZEN
```

---

# 99. Container freeze

Una vez compilado:

```text
core bindings
```

no deberán cambiar arbitrariamente durante requests.

---

# 100. Runtime mutation

Prohibido:

```text
request
→ replace EntityManager binding
```

en producción normal.

---

# 101. Testing exception

Testing podrá crear un Container independiente con overrides antes del freeze.

---

# 102. Testing override architecture

Ejemplo:

```text
Production Container Definition
        ↓
Test Container Builder
        ↓
Controlled Overrides
        ↓
Compiled Test Container
```

---

# 103. Allowed test overrides

Ejemplos:

```text
Clock
Telemetry sink
Event dispatcher
Cache provider
Failure injector
Credential provider
```

---

# 104. Dangerous fake replacement

Reemplazar:

```text
real Driver
```

por un FakeDriver convierte la evidencia en:

```text
unit/fake integration evidence
```

no DBMS integration evidence.

---

# 105. Binding metadata

Cada binding debería poder declarar:

```text
service id
contract
implementation
lifetime
source/provider
visibility
tags
dependencies
```

---

# 106. Database binding descriptor

Conceptualmente:

```php
final readonly class DatabaseBindingDescriptor
{
    public function __construct(
        public ServiceId $id,
        public ServiceLifetime $lifetime,
        public ServiceVisibility $visibility,
        public ProviderId $provider,
    ) {}
}
```

---

# 107. Lifetime validation

VoltStack deberá poder verificar reglas como:

```text
SHARED service
must not capture
OPERATION_SCOPED service
```

directamente.

---

# 108. Captive dependency

Problema clásico:

```text
Singleton
    ↓
Scoped Service
```

El singleton termina reteniendo una instancia scoped.

---

# 109. Database captive dependency

Ejemplo prohibido:

```text
Shared RepositoryRegistry
        ↓
EntityManager(scoped)
```

si lo captura directamente.

---

# 110. Correct alternative

```text
Shared RepositoryFactory
        ↓
Scoped EntityManager supplied per call/scope
```

---

# 111. Lifetime partial order

Podemos definir:

```text
SHARED > SCOPED > TRANSIENT
```

en duración.

Un servicio de duración mayor no deberá capturar directamente uno de duración menor.

Formalmente:

```text
Lifetime(A) > Lifetime(B)
∧
A captures B
→
Invalid
```

salvo mediante un resolver/provider diseñado específicamente para scope.

---

# 112. Scope-aware provider

Permitido:

```text
Shared ModelContextResolver
        ↓
CurrentScopeAccessor
        ↓
EntityManager(current operation)
```

si el resolver no retiene la instancia.

---

# 113. CurrentScopeAccessor

Este componente será infraestructura crítica.

---

# 114. Scope accessor contract

Conceptualmente:

```php
interface DatabaseScopeAccessor
{
    public function current(): DatabaseOperationScope;

    public function hasCurrent(): bool;
}
```

---

# 115. Scope accessor implementation

Debe ser compatible con:

```text
FrankenPHP worker requests
RoadRunner requests
OpenSwoole coroutines
CLI
queue workers
```

---

# 116. No universal static slot

No deberá implementarse simplemente como:

```php
private static ?DatabaseOperationScope $current;
```

porque no es seguro para todos los modelos concurrentes.

---

# 117. Runtime-specific context storage

Podrán existir adapters:

```text
SynchronousScopeStorage
FrankenPhpScopeStorage
CoroutineScopeStorage
```

según Runtime Manager.

---

# 118. Scope storage abstraction

```php
interface OperationScopeStorage
{
    public function bind(DatabaseOperationScope $scope): void;

    public function current(): DatabaseOperationScope;

    public function unbind(OperationId $id): void;
}
```

---

# 119. Binding collision

Intentar enlazar dos scopes incompatibles al mismo execution context deberá producir error.

---

# 120. Scope token

Podrá utilizarse:

```text
OperationScopeToken
```

para garantizar que quien cierra un scope es quien lo abrió.

---

# 121. Example

```php
$token = $scopes->enter($context);

try {
    // operation
} finally {
    $scopes->leave($token);
}
```

---

# 122. finally requirement

El framework deberá utilizar estructuras equivalentes a:

```text
try/finally
```

para garantizar cleanup incluso ante exceptions.

---

# 123. Exception path

```text
Operation
    ↓
Exception
    ↓
Database finalization
    ↓
Rollback if owned/appropriate
    ↓
Resource reset
    ↓
Scope destruction
    ↓
Exception propagation
```

---

# 124. Error during cleanup

Si cleanup también falla:

```text
PrimaryFailure
+
CleanupFailure
```

deberán preservarse ambos.

---

# 125. Cleanup failure ≠ replace original error

Diagnostics deberán conservar:

```text
primary cause
cleanup cause
resource state
```

---

# 126. Unknown cleanup

Si el estado final de un recurso es desconocido:

```text
quarantine/discard
```

por defecto.

---

# 127. Container disposal hooks

El Container podrá proporcionar:

```text
Disposable
ScopedDisposable
```

o mecanismo equivalente.

---

# 128. Database-specific lifecycle

No obstante, el Container destructor genérico no deberá ser el único mecanismo para cerrar transacciones.

Database Lifecycle Coordinator deberá intervenir antes.

---

# 129. Ordering

Correcto:

```text
Database finalize semantics
        ↓
resource cleanup
        ↓
container disposal
```

No depender únicamente de:

```text
container destroys objects
```

---

# 130. Lazy services

Servicios costosos podrán declararse lazy.

Ejemplos:

```text
backup provider
schema introspector
migration executor
administration services
```

---

# 131. Lazy ≠ unresolved configuration

Aunque un servicio sea lazy:

```text
configuration
```

deberá validarse anticipadamente cuando sea posible.

---

# 132. Lazy connections

Las conexiones Database deberían abrirse normalmente bajo demanda.

```text
resolve ConnectionManager
≠
connect immediately
```

---

# 133. Connection lazy creation

```text
Query requested
    ↓
Connection resolution
    ↓
Lease acquisition
    ↓
Physical connection if needed
```

---

# 134. Zero-query request

Un HTTP request que no use Database no debería abrir una conexión.

---

# 135. Container compilation

VoltStack podrá compilar definiciones del Container para producción.

---

# 136. Compiled definitions

Podrán incluir:

```text
service graph
constructor arguments
aliases
factory calls
lifetimes
decorators
tags
```

---

# 137. Compilation ≠ service serialization

No todos los objetos deben serializarse.

El compiled container puede generar código optimizado para construirlos.

---

# 138. Compiled container safety

La compilación deberá validar:

```text
missing dependencies
circular dependencies
invalid lifetimes
unknown service ids
duplicate aliases
invalid decorators
```

---

# 139. Database-specific compiler pass

Podrá existir:

```text
DatabaseContainerCompilerPass
```

---

# 140. Compiler pass responsibilities

Podría:

```text
collect drivers
collect dialects
collect custom types
collect extensions
validate lifetimes
generate registries
freeze descriptors
```

---

# 141. Compiler pass ≠ query compiler

Debe evitarse confusión:

```text
Container Compiler
≠
SQL Compiler
```

---

# 142. Tags

El Container podrá utilizar tags conceptuales:

```text
database.driver
database.dialect
database.type
database.compiler
database.extension
database.capability_provider
database.telemetry_adapter
```

---

# 143. Tag collection

Los tags se resolverán durante bootstrap/compilation.

No durante cada query.

---

# 144. Priority

Si existen múltiples providers:

```text
priority
```

podrá formar parte del registro.

---

# 145. Priority ≠ conflict resolution universal

Dos implementaciones incompatibles con el mismo ID deberán producir conflicto explícito.

---

# 146. Alias system

Podrán existir aliases:

```text
EntityManagerInterface
→ DefaultEntityManager

ConnectionResolver
→ DefaultConnectionManager
```

---

# 147. Alias stability

Aliases públicos deberán formar parte de la API documentada.

---

# 148. Internal aliases

Aliases internos podrán cambiar sin garantía pública.

---

# 149. Multiple connections

El Container no deberá registrar necesariamente:

```text
Connection::class
```

como una única conexión física.

Preferible:

```text
ConnectionManager
ConnectionResolver
```

---

# 150. Named connection

Podrá utilizarse:

```php
$connections->connection('analytics');
```

o un API equivalente.

---

# 151. Named DI binding

Opcionalmente podrían existir bindings calificados:

```text
Connection<default>
Connection<analytics>
```

si el Container soporta qualifiers.

---

# 152. Qualifier ≠ physical connection

El binding representa intención lógica.

La conexión física todavía puede provenir de un pool.

---

# 153. Read/write connections

No deberán modelarse ingenuamente como dos singletons:

```text
ReadConnection
WriteConnection
```

si existe routing dinámico.

Preferible:

```text
ConnectionRouter
```

---

# 154. Transaction pinning

Durante una transacción:

```text
ConnectionRouter
```

deberá respetar el `TransactionContext`.

El Container no deberá resolver una conexión independiente ignorando ese contexto.

---

# 155. Contextual binding

El Container podrá soportar contextual bindings, pero Database no deberá depender excesivamente de ellos para semántica dinámica.

---

# 156. Why

Routing depende de:

```text
transaction
tenant
shard
read/write intent
consistency
```

y esto pertenece al DatabaseContext, no sólo al tipo solicitado.

---

# 157. Decorators

El Container podrá decorar servicios para:

```text
telemetry
profiling
debugging
```

cuando la semántica lo permita.

---

# 158. Decorator rule

```text
Decorator
must preserve
Contract Semantics
```

---

# 159. Telemetry decorator

Ejemplo:

```text
QueryExecutor
    ↓
TelemetryQueryExecutor
    ↓
DefaultQueryExecutor
```

---

# 160. Decorator failure

Telemetry decorator no deberá transformar:

```text
successful query
```

en fallo simplemente porque un exporter opcional falló.

---

# 161. Security decorators

Algunas políticas de seguridad podrán usar decorators si son realmente obligatorias y el wiring garantiza su presencia.

---

# 162. Mandatory decorator validation

Si una política depende de un decorator:

```text
required decorator missing
→ boot failure
```

---

# 163. Decoration order

Cuando existan:

```text
security
telemetry
profiling
retry
```

el orden deberá definirse explícitamente.

---

# 164. Hidden order prohibited

No deberá dependerse del orden accidental de registro de Service Providers.

---

# 165. Dependency graph

Ejemplo:

```text
Application
     ↓
Repository
     ↓
EntityManager
     ↓
PersistenceEngine
     ↓
QueryExecutor
     ↓
ConnectionManager
     ↓
DriverRegistry
```

Cross-cutting:

```text
Telemetry
Cache
Events
Security
```

mediante contratos/adapters.

---

# 166. Circular dependency example

Problemático:

```text
EntityManager
→ EventDispatcher
→ Listener
→ Repository
→ EntityManager
```

si el listener se construye eager.

---

# 167. Solutions

Dependiendo del caso:

```text
lazy listener resolution
event subscriber factory
narrow contract extraction
domain redesign
```

No simplemente esconder el ciclo con Service Locator.

---

# 168. Circular dependency detection

El Container compiler deberá detectar ciclos estáticos cuando sea posible.

---

# 169. Runtime cycles

También deberán existir diagnostics para ciclos creados por factories dinámicas.

---

# 170. Bootstrap dependency ordering

Ejemplo:

```text
Platform
   ↓
Config
   ↓
Container Core
   ↓
Database Core Registration
   ↓
Database Extensions
   ↓
Optional Framework Bridges
   ↓
Registry Freeze
   ↓
Metadata Compilation
   ↓
Database Ready
```

---

# 171. Config before Database construction

Database necesita configuración normalizada para construir:

```text
connection definitions
pool policies
cache policies
driver selection
```

---

# 172. Secret materialization later

Pero:

```text
Config parsing
```

no necesariamente debe resolver inmediatamente los secretos.

---

# 173. Extension before freeze

Plugins deberán registrarse antes de:

```text
registry freeze
```

---

# 174. Late plugin registration

Después del freeze:

```text
LateDatabaseExtensionRegistrationException
```

o equivalente.

---

# 175. Hot reload

En desarrollo, cambios de extensiones deberán crear/recompilar una nueva generación de container/registry.

No mutar arbitrariamente la generación activa.

---

# 176. Generation model

Podrá existir:

```text
ContainerGeneration
DatabaseRegistryGeneration
MetadataGeneration
```

---

# 177. Generation consistency

Una operación iniciada con generación `G1` deberá mantener referencias coherentes durante su ejecución.

No mezclar:

```text
metadata G1
compiler registry G2
type registry G3
```

---

# 178. Atomic generation swap

Hot reload podrá hacer:

```text
build G2
validate G2
freeze G2
publish G2
```

Las nuevas operaciones usan G2.

Las existentes terminan con G1.

---

# 179. Production mode

Normalmente:

```text
single immutable generation
```

durante el lifetime del worker.

---

# 180. Worker restart

Cambios estructurales de container/config podrán requerir:

```text
worker recycle
```

en producción.

---

# 181. FrankenPHP integration

FrankenPHP será el runtime primario.

---

# 182. FrankenPHP boot

```text
Worker Start
    ↓
Build/Load Compiled Container
    ↓
Resolve Shared Database Infrastructure
    ↓
Freeze Registries
    ↓
READY
```

---

# 183. FrankenPHP request

```text
Request
   ↓
Create Operation Child Scope
   ↓
Bind Runtime Context
   ↓
Resolve Database scoped services lazily
   ↓
Execute
   ↓
Finalize Database
   ↓
Destroy child scope
```

---

# 184. No child scope reuse

Cada request deberá obtener:

```text
fresh mutable Database scope
```

---

# 185. Request A/B

```text
Worker
├── Shared Container
│
├── Scope A
│   ├── EntityManager A
│   ├── UnitOfWork A
│   └── IdentityMap A
│
└── Scope B
    ├── EntityManager B
    ├── UnitOfWork B
    └── IdentityMap B
```

---

# 186. Shared infrastructure

Ambos pueden utilizar:

```text
TypeRegistry
MetadataRegistry
CompilerRegistry
ConnectionPool
```

si sus contratos permiten compartirlos.

---

# 187. Connection contamination protection

El pool compartido deberá devolver sólo conexiones sanitizadas.

---

# 188. RoadRunner

El mismo modelo general:

```text
Worker
→ Shared Container
→ Per-request scope
```

---

# 189. OpenSwoole

El Container deberá soportar:

```text
concurrent scopes
```

en el mismo proceso.

---

# 190. OpenSwoole critical invariant

```text
CurrentScope
```

no podrá ser una única propiedad global del worker.

---

# 191. Coroutine scope

Conceptualmente:

```text
Coroutine A → Scope A
Coroutine B → Scope B
```

aunque ambas existan simultáneamente.

---

# 192. Concurrent resolution

Resolver:

```text
EntityManager::class
```

en A debe devolver A.

En B debe devolver B.

---

# 193. Scope propagation

Si una coroutine hija debe heredar contexto:

```text
inherit explicitly
```

según política.

No por accidente.

---

# 194. Queue workers

```text
Worker
├── Shared Container
├── Job Scope 1
├── Job Scope 2
└── ...
```

---

# 195. Job failure

Incluso cuando un job lanza excepción:

```text
scope cleanup
```

deberá ejecutarse.

---

# 196. CLI

Un comando corto puede usar:

```text
one operation scope
```

---

# 197. Long-running CLI

Podrá utilizar:

```text
parent command context
+
multiple operation scopes
```

para evitar crecimiento de:

```text
IdentityMap
UnitOfWork
memory
```

---

# 198. Example import

```text
Import Command
    ↓
Batch 1 Scope
    ↓
close/reset
    ↓
Batch 2 Scope
    ↓
close/reset
```

---

# 199. Scope nesting

Debe definirse cuidadosamente.

Por defecto:

```text
nested operation scope
```

no deberá crearse accidentalmente.

---

# 200. Nested scope policies

Posibles:

```text
JOIN_CURRENT
CREATE_CHILD
REJECT
```

según tipo de operación.

---

# 201. Scope nesting ≠ transaction nesting

Regla:

```text
Operation Scope Nesting
≠
Transaction Nesting
```

---

# 202. Child scope

Un child scope podría heredar:

```text
trace
actor
tenant
deadline
```

pero no necesariamente:

```text
EntityManager
UnitOfWork
TransactionContext
```

---

# 203. Inheritance policy

Cada componente de contexto deberá declarar:

```text
INHERIT
COPY
DERIVE
RESET
FORBID
```

---

# 204. Container scope API

Conceptualmente:

```php
$scope = $container->createScope(
    ScopeType::DATABASE_OPERATION,
    $context,
);

try {
    $scope->get(ApplicationOperation::class)->run();
} finally {
    $scope->close();
}
```

---

# 205. Scope ownership

Quien crea el scope es responsable de cerrarlo, salvo transferencia explícita.

---

# 206. Ownership token

Podrá utilizarse para impedir double-close o cierre por owner incorrecto.

---

# 207. Double close

`close()` deberá ser:

```text
idempotent where practical
```

pero registrar errores de lifecycle cuando exista uso incorrecto.

---

# 208. Container diagnostics

Debe ser posible inspeccionar:

```text
service
implementation
lifetime
scope
provider
dependencies
decorators
```

---

# 209. CLI diagnostics

Ejemplo futuro:

```bash
php voltstack database:container
```

Salida conceptual:

```text
EntityManager
  implementation: DefaultEntityManager
  lifetime: operation_scoped
  provider: database

TypeRegistry
  implementation: FrozenTypeRegistry
  lifetime: shared
```

---

# 210. Container graph diagnostics

Podrá existir:

```bash
php voltstack database:container --graph
```

---

# 211. Lifetime diagnostics

```bash
php voltstack database:container --validate-lifetimes
```

---

# 212. Example error

```text
DB-CONTAINER-004

Invalid captive dependency.

Shared service:
    UserRepositoryRegistry

captures scoped service:
    EntityManager

A shared Database service cannot retain an operation-scoped service.

Suggested resolution:
    inject RepositoryFactory or DatabaseScopeAccessor instead.
```

---

# 213. Diagnostics security

No mostrar:

```text
passwords
DSNs containing credentials
tokens
secret values
```

---

# 214. Container events

Podrán emitirse eventos internos:

```text
DatabaseServiceRegistered
DatabaseScopeOpened
DatabaseScopeClosing
DatabaseScopeClosed
DatabaseScopeCleanupFailed
```

---

# 215. Container events ≠ public application events

No todos deberán exponerse al Event System público.

---

# 216. Telemetry

Métricas posibles:

```text
database.scope.active
database.scope.duration
database.scope.cleanup.duration
database.container.resolve.duration
database.container.resolve.failures
database.scope.cleanup.failures
```

---

# 217. Cardinality

No utilizar:

```text
operation UUID
tenant ID
user ID
```

como metric labels sin control.

---

# 218. Resolution performance

El Container deberá minimizar:

```text
repeated dynamic reflection
repeated service graph analysis
repeated extension discovery
```

durante queries.

---

# 219. Compile hot paths

En producción deberá favorecerse:

```text
precomputed service graph
compiled factories
frozen registries
direct constructor wiring
```

---

# 220. No container lookup per AST node

Un SQL Compiler no deberá consultar el Container para cada nodo AST.

---

# 221. No container lookup per hydrated property

Hydration tampoco deberá hacer:

```text
container->get(...)
```

por cada columna.

---

# 222. Hot path rule

> **El Container deberá ensamblar el hot path, no formar parte del hot path innecesariamente.**

---

# 223. Resolution caching

Los servicios scoped podrán cachearse dentro del scope.

---

# 224. Transient object allocation

Objetos pequeños como:

```text
QueryBuilder
QueryContext child
AST node
```

podrán crearse directamente mediante factories optimizadas.

---

# 225. Container ≠ object factory universal

No todos los objetos PHP deben registrarse como servicios.

---

# 226. Value objects

No registrar como servicios:

```text
QueryId
EntityId
TableName
ColumnName
Timeout
CapabilityId
```

normalmente.

---

# 227. Entities

Entidades ORM tampoco serán servicios del Container.

---

# 228. Entity construction

Debe ocurrir mediante:

```text
ORM hydration
EntityFactory
application construction
```

no mediante DI container como regla general.

---

# 229. DTOs

DTOs tampoco requieren Container salvo que representen un servicio, no datos.

---

# 230. Migrations

Las clases Migration podrán ser construidas por Container si requieren servicios permitidos.

---

# 231. Migration dependency restrictions

Debe evitarse que migrations históricas dependan de servicios de aplicación volátiles.

---

# 232. Seeder resolution

Seeders podrán usar DI.

---

# 233. Seeder scope

Se ejecutarán dentro de un Database operation/command scope.

---

# 234. Event listener resolution

Listeners Database podrán resolverse lazy por Container.

---

# 235. Listener lifetime

Su lifetime deberá corresponder a sus dependencias.

Un listener shared no puede capturar Repository scoped.

---

# 236. Authorization integration

Un data-access policy adapter podrá ser scoped si depende del actor actual.

---

# 237. Authentication integration

Un actor provider shared puede consultar scope, pero no capturar actor durante bootstrap.

---

# 238. Config integration

`DatabaseConfiguration` deberá registrarse como:

```text
shared immutable
```

después de normalización.

---

# 239. Raw config arrays

No deberán circular ampliamente por Database.

---

# 240. Typed config

Preferir:

```text
DatabaseConfiguration
ConnectionConfiguration
PoolConfiguration
CacheConfiguration
TelemetryConfiguration
```

---

# 241. Configuration generation

Podrá existir:

```text
ConfigurationGeneration
```

para correlacionar container/config/metadata.

---

# 242. Configuration reload

No deberá mutarse una configuración shared utilizada por operaciones activas.

Preferible:

```text
build new generation
→ validate
→ swap for new operations
```

---

# 243. Container identity

Cada container/worker podrá tener:

```text
ContainerInstanceId
GenerationId
```

para diagnostics.

---

# 244. Scope identity

Cada scope:

```text
OperationId
ScopeId
```

sin exponerlos como high-cardinality metrics.

---

# 245. Dependency visibility

Servicios podrán clasificarse:

```text
PUBLIC
EXTENSION
INTERNAL
```

---

# 246. Public services

Ejemplos potenciales:

```text
EntityManager
TransactionManager
ConnectionManager
SchemaManager
MigrationManager
```

según Public API definitivo.

---

# 247. Extension services

Ejemplos:

```text
TypeRegistryBuilder
DriverRegistryBuilder
CompilerExtensionRegistry
```

sólo durante bootstrap.

---

# 248. Internal services

Ejemplos:

```text
HydrationAssembler
ChangeSetCalculator
PredicateNormalizer
```

no deberán convertirse accidentalmente en API pública sólo porque el Container pueda resolverlos.

---

# 249. Service ID stability

IDs públicos deberán tener política de compatibilidad.

---

# 250. Class name ≠ guaranteed service ID

VoltStack podrá usar clases como IDs internamente, pero deberá documentar qué bindings forman parte de API.

---

# 251. Multiple EntityManagers

La arquitectura deberá permitir potencialmente:

```text
default EntityManager
analytics EntityManager
legacy EntityManager
```

sin rediseñar el Container.

---

# 252. EntityManager key

Podrá existir:

```text
EntityManagerName
```

o:

```text
PersistenceContextId
```

---

# 253. Default manager

La facade/Model API podrá utilizar:

```text
default
```

salvo mapping explícito.

---

# 254. Entity manager routing

No deberá confundirse con:

```text
connection routing
shard routing
replica routing
```

---

# 255. Named manager factory

```text
EntityManagerRegistry
        ↓
get("default")
get("legacy")
```

podrá operar dentro del scope.

---

# 256. Registry lifetime

El descriptor de managers puede ser shared.

Las instancias de managers son scoped.

---

# 257. Formal model

Sea:

```text
C = Root Container
Sᵢ = Operation Scope i
```

Entonces:

```text
SharedService(C)
=
same instance for S₁...Sₙ
```

mientras:

```text
ScopedService(Sᵢ)
≠
ScopedService(Sⱼ)
```

para:

```text
i ≠ j
```

---

# 258. Captive dependency formalization

Sea:

```text
L(x)
```

la duración de un servicio.

Si:

```text
L(A) > L(B)
```

y A retiene directamente B:

```text
Capture(A,B)
```

entonces:

```text
InvalidLifetimeGraph(A,B)
```

salvo que B sea obtenido temporalmente mediante un mecanismo scope-aware sin retención.

---

# 259. Scope correctness

Un scope será correctamente cerrado si:

```text
Close(S)
=
DatabaseFinalized(S)
∧
ResourcesReleased(S)
∧
ScopedInstancesDisposed(S)
∧
ScopeBindingRemoved(S)
```

---

# 260. Safe reuse

Un recurso `R` podrá volver a infraestructura shared sólo si:

```text
Reusable(R)
=
OwnershipReleased
∧
StateSanitized
∧
NoActiveTransaction
∧
NoOpenCursor
∧
NoUnknownFailure
```

según sus capacidades.

---

# 261. Container readiness

```text
DatabaseContainerReady
=
BindingsValid
∧
LifetimesValid
∧
DependenciesResolvable
∧
RequiredExtensionsPresent
∧
RegistriesValid
```

---

# 262. Container readiness ≠ database readiness

No implica:

```text
DB server reachable
```

salvo que se ejecute un readiness probe explícito.

---

# 263. Architectural invariants

## DB-CONTAINER-001

Container Lifetime deberá coincidir con Semantic Lifetime.

## DB-CONTAINER-002

EntityManager no será process singleton.

## DB-CONTAINER-003

UnitOfWork no será process singleton.

## DB-CONTAINER-004

IdentityMap no será process singleton.

## DB-CONTAINER-005

TransactionContext no será process singleton.

## DB-CONTAINER-006

DatabaseContext no será process singleton.

## DB-CONTAINER-007

Shared service no capturará scoped mutable service.

## DB-CONTAINER-008

No active scope no producirá fallback global silencioso.

## DB-CONTAINER-009

Static facade no implicará static mutable state.

## DB-CONTAINER-010

Container no definirá Database semantics.

---

# 264. Registration invariants

## DB-CONTAINER-011

Registration ≠ Boot.

## DB-CONTAINER-012

Registration no ejecutará queries arbitrarias.

## DB-CONTAINER-013

Extensions deberán registrarse antes del freeze.

## DB-CONTAINER-014

Duplicate IDs deberán detectarse.

## DB-CONTAINER-015

Required missing dependency deberá fallar durante bootstrap cuando sea detectable.

## DB-CONTAINER-016

Optional missing dependency no deberá fallar si su feature está deshabilitada.

## DB-CONTAINER-017

Container compilation deberá preservar semántica.

## DB-CONTAINER-018

Circular dependencies deberán detectarse cuando sea posible.

## DB-CONTAINER-019

Binding visibility será explícita.

## DB-CONTAINER-020

Test overrides ocurrirán antes del freeze.

---

# 265. Scope invariants

## DB-CONTAINER-021

OperationScope será independiente de HTTP.

## DB-CONTAINER-022

Cada request obtendrá un scope mutable nuevo.

## DB-CONTAINER-023

Cada queue job obtendrá un scope mutable nuevo.

## DB-CONTAINER-024

Cada operación CLI obtendrá scope definido.

## DB-CONTAINER-025

Concurrent operations tendrán scopes aislados.

## DB-CONTAINER-026

Scope nesting no implicará transaction nesting.

## DB-CONTAINER-027

Scope ownership será explícito.

## DB-CONTAINER-028

Scope cleanup deberá ejecutarse en exception paths.

## DB-CONTAINER-029

Cleanup failure será observable.

## DB-CONTAINER-030

Unknown cleanup impedirá reutilización insegura.

---

# 266. Service invariants

## DB-CONTAINER-031

Mutable QueryBuilder no será singleton.

## DB-CONTAINER-032

Immutable metadata podrá ser shared.

## DB-CONTAINER-033

Frozen registries podrán ser shared.

## DB-CONTAINER-034

Repository con EntityManager capturado será scoped.

## DB-CONTAINER-035

RepositoryFactory podrá ser shared si es stateless.

## DB-CONTAINER-036

TransactionManager shared no almacenará TransactionContext.

## DB-CONTAINER-037

Compiler shared no almacenará query state.

## DB-CONTAINER-038

Optimizer shared no almacenará operation state.

## DB-CONTAINER-039

Planner shared no almacenará operation state.

## DB-CONTAINER-040

Entities no serán servicios DI por defecto.

---

# 267. Connection invariants

## DB-CONTAINER-041

Pool ≠ Connection.

## DB-CONTAINER-042

Connection ≠ ConnectionLease.

## DB-CONTAINER-043

Resolving Database services no abrirá conexión innecesariamente.

## DB-CONTAINER-044

Zero-query request podrá terminar sin conexión DB.

## DB-CONTAINER-045

Connection routing utilizará DatabaseContext.

## DB-CONTAINER-046

Transaction pinning no será ignorado por DI.

## DB-CONTAINER-047

Tainted connection no volverá al pool.

## DB-CONTAINER-048

UNKNOWN reset connection no será reusable por defecto.

## DB-CONTAINER-049

Named connection ≠ permanent physical connection.

## DB-CONTAINER-050

Connection ownership será explícito.

---

# 268. Runtime invariants

## DB-CONTAINER-051

FrankenPHP compartirá sólo servicios seguros.

## DB-CONTAINER-052

FrankenPHP requests no compartirán EntityManager.

## DB-CONTAINER-053

RoadRunner requests no compartirán mutable Database state.

## DB-CONTAINER-054

OpenSwoole coroutines tendrán scope isolation.

## DB-CONTAINER-055

CurrentScope no será un static global universal.

## DB-CONTAINER-056

Worker reuse ≠ scope reuse.

## DB-CONTAINER-057

Job retry utilizará un scope nuevo.

## DB-CONTAINER-058

Long-running CLI podrá rotar scopes.

## DB-CONTAINER-059

Worker shutdown finalizará scopes activos según policy.

## DB-CONTAINER-060

Worker recycle podrá ser requerido ante estado no recuperable.

---

# 269. Extension invariants

## DB-CONTAINER-061

Extension Provider ≠ unrestricted Container access por defecto.

## DB-CONTAINER-062

Extension registries serán frozen.

## DB-CONTAINER-063

Late extension registration fallará.

## DB-CONTAINER-064

Extension discovery no ocurrirá en cada query.

## DB-CONTAINER-065

Plugin service lifetimes serán validados.

## DB-CONTAINER-066

Extension API será distinta de internal API.

## DB-CONTAINER-067

Container tags no redefinirán IDs silenciosamente.

## DB-CONTAINER-068

Plugin conflicts serán explícitos.

## DB-CONTAINER-069

Registry generation será consistente por operación.

## DB-CONTAINER-070

Hot reload publicará generaciones completas.

---

# 270. Performance invariants

## DB-CONTAINER-071

Container no será consultado por cada AST node.

## DB-CONTAINER-072

Container no será consultado por cada hydrated field.

## DB-CONTAINER-073

Hot path deberá favorecer dependencias ya resueltas.

## DB-CONTAINER-074

Reflection repetitiva deberá minimizarse en producción.

## DB-CONTAINER-075

Compiled container no cambiará resultados.

## DB-CONTAINER-076

Lazy services no ocultarán errores de configuración críticos.

## DB-CONTAINER-077

Scope creation tendrá costo acotado.

## DB-CONTAINER-078

Scope destruction tendrá costo observable.

## DB-CONTAINER-079

Shared immutable infrastructure podrá amortizar bootstrap.

## DB-CONTAINER-080

Performance optimization no justificará shared mutable state.

---

# 271. Testing invariants

## DB-CONTAINER-081

Test Container podrá reemplazar adapters explícitamente.

## DB-CONTAINER-082

Fake Driver no demostrará DBMS conformance.

## DB-CONTAINER-083

Scope leakage tendrá pruebas dedicadas.

## DB-CONTAINER-084

Captive dependencies tendrán pruebas arquitectónicas.

## DB-CONTAINER-085

Container graph deberá ser validable sin ejecutar queries.

## DB-CONTAINER-086

Persistent worker tests ejecutarán múltiples operaciones secuenciales.

## DB-CONTAINER-087

Concurrent runtime tests comprobarán aislamiento.

## DB-CONTAINER-088

Cleanup failures podrán inyectarse.

## DB-CONTAINER-089

Test scopes no contaminarán otros tests.

## DB-CONTAINER-090

Container generation deberá formar parte de diagnostics reproducibles cuando sea relevante.

---

# 272. Security invariants

## DB-CONTAINER-091

Container diagnostics no expondrán secrets.

## DB-CONTAINER-092

Secret materialization será limitada.

## DB-CONTAINER-093

Tenant context no será singleton.

## DB-CONTAINER-094

Actor context no será singleton.

## DB-CONTAINER-095

Mandatory security adapters deberán validarse durante boot.

## DB-CONTAINER-096

Plugin registration será código privilegiado.

## DB-CONTAINER-097

Application code no podrá reemplazar servicios críticos después del freeze arbitrariamente.

## DB-CONTAINER-098

Service graph diagnostics respetarán redaction.

## DB-CONTAINER-099

Cross-operation service capture será considerado defecto de seguridad además de lifecycle.

## DB-CONTAINER-100

Conveniencia del Container nunca tendrá prioridad sobre aislamiento.

---

# 273. Anti-pattern: Singleton EntityManager

```php
$container->singleton(
    EntityManager::class,
    DefaultEntityManager::class,
);
```

Incorrecto en el modelo general de VoltStack.

---

# 274. Anti-pattern: Singleton Repository

```text
UserRepository(singleton)
    ↓
EntityManager(request A)
```

Después request B utiliza el mismo repository.

Resultado:

```text
cross-request EntityManager leakage
```

---

# 275. Anti-pattern: Global Current Tenant

```php
final class TenantContext
{
    public static ?Tenant $current = null;
}
```

No es un mecanismo válido para runtimes concurrentes/persistentes.

---

# 276. Anti-pattern: Container lookup everywhere

```php
public function hydrate(array $row): object
{
    $converter = Container::get(TypeConverter::class);
    ...
}
```

en cada row.

Debe evitarse.

---

# 277. Anti-pattern: Container as configuration store

No:

```text
container->get("database.connections.default.host")
```

en todo el sistema.

Usar objetos de configuración tipados.

---

# 278. Anti-pattern: registering entities

```text
User entity
→ singleton service
```

Incorrecto.

---

# 279. Anti-pattern: connection singleton

Una conexión física mutable no deberá convertirse automáticamente en singleton sólo porque el Container permita hacerlo.

---

# 280. Anti-pattern: hiding lifetime mismatch with lazy proxy

Un lazy proxy no corrige:

```text
singleton captures scoped dependency
```

si finalmente conserva la instancia incorrecta.

---

# 281. Anti-pattern: child container as transaction

No:

```text
new transaction
→ create DI child container
```

por defecto.

Son conceptos diferentes.

---

# 282. Anti-pattern: auto-resolve everything

El Container no deberá construir automáticamente cualquier clase interna mediante reflection sin control de visibilidad.

---

# 283. Anti-pattern: arbitrary runtime rebinding

```text
request A
→ container->bind(QueryExecutor::class, CustomExecutor::class)
```

en el container compartido.

Prohibido en operación normal.

---

# 284. Anti-pattern: Container-driven ORM identity

El Container no deberá decidir si dos rows representan la misma entidad.

Eso pertenece a:

```text
IdentityMap
```

---

# 285. Anti-pattern: container-managed transaction

El Container puede administrar lifetime del `TransactionContext`, pero no deberá interpretar:

```text
COMMIT
ROLLBACK
UNKNOWN
```

---

# 286. Anti-pattern: boot opens every connection

Esto:

```text
boot
→ connect MySQL
→ connect PostgreSQL
→ connect analytics
→ connect replicas
```

no deberá ser comportamiento predeterminado.

---

# 287. Anti-pattern: silent Null adapter for mandatory security

Si una política de seguridad requiere un provider:

```text
missing provider
```

no deberá sustituirse silenciosamente por:

```text
NullSecurityProvider
```

---

# 288. Testing architecture

Se propone:

```text
tests/Quantum/Database/Integration/Container/
├── DatabaseServiceProviderTest.php
├── DatabaseContainerCompilationTest.php
├── DatabaseScopeTest.php
├── DatabaseLifetimeTest.php
├── CaptiveDependencyTest.php
├── EntityManagerScopeTest.php
├── UnitOfWorkScopeTest.php
├── IdentityMapScopeTest.php
├── ConnectionLifecycleBindingTest.php
├── ExtensionRegistrationTest.php
├── OptionalIntegrationBindingTest.php
├── ContainerDiagnosticsTest.php
├── FrankenPhpScopeIsolationTest.php
├── RoadRunnerScopeIsolationTest.php
├── OpenSwooleCoroutineScopeTest.php
└── ContainerLeakTest.php
```

---

# 289. Architecture tests

Además de tests funcionales deberán existir pruebas que inspeccionen el grafo.

Ejemplo:

```text
assertSharedServiceDoesNotCaptureScopedService()
```

---

# 290. Scope identity test

```text
scope A:
    emA1 === emA2

scope B:
    emB1 === emB2

cross scope:
    emA1 !== emB1
```

---

# 291. IdentityMap isolation test

```text
scope A:
    User#10 → object A

scope B:
    User#10 → object B
```

Debe cumplirse:

```text
A !== B
```

aunque ambos representen la misma fila persistente.

---

# 292. Shared metadata test

```text
scope A metadata registry
===
scope B metadata registry
```

si es immutable/shared.

---

# 293. Leak test

Ejecutar:

```text
10,000 operation scopes
```

y comprobar:

```text
no retained EntityManagers
no retained UoWs
no retained IdentityMaps
no retained tenant contexts
bounded memory behavior
```

considerando GC y allocator behavior.

---

# 294. Failure cleanup test

Inyectar:

```text
query exception
transaction exception
rollback failure
cursor close failure
connection reset failure
```

y comprobar finalización correcta o quarantine.

---

# 295. Concurrent test

Ejemplo:

```text
Coroutine A
Tenant A
EntityManager A

Coroutine B
Tenant B
EntityManager B
```

Ninguna operación deberá observar contexto ajeno.

---

# 296. Proposed namespaces

```text
VoltStack\Quantum\Database\Integration\Container
VoltStack\Quantum\Database\Runtime\Scope
VoltStack\Quantum\Database\Runtime\Context
VoltStack\Quantum\Database\Runtime\Lifecycle
```

---

# 297. Componentes propuestos

```text
DatabaseServiceProvider
DatabaseContainerRegistrar
DatabaseContainerCompilerPass
DatabaseBindingDescriptor
DatabaseScopeManager
DatabaseScopeAccessor
DatabaseOperationScope
DatabaseLifecycleCoordinator
DatabaseScopedServiceFactory
DatabaseIntegrationRegistry
DatabaseContainerDiagnostics
```

---

# 298. Interfaces conceptuales

```php
interface DatabaseScopeManager
{
    public function enter(
        RuntimeOperationContext $context,
    ): DatabaseScopeToken;

    public function leave(
        DatabaseScopeToken $token,
    ): void;
}
```

```php
interface DatabaseScopeAccessor
{
    public function hasCurrent(): bool;

    public function current(): DatabaseOperationScope;
}
```

---

# 299. Container registration example

Conceptualmente:

```php
$container->shared(
    TypeRegistry::class,
    FrozenTypeRegistry::class,
);

$container->shared(
    DriverRegistry::class,
    FrozenDriverRegistry::class,
);

$container->scoped(
    DatabaseContext::class,
    DatabaseContextFactory::class,
);

$container->scoped(
    IdentityMap::class,
    DefaultIdentityMap::class,
);

$container->scoped(
    UnitOfWork::class,
    DefaultUnitOfWork::class,
);

$container->scoped(
    EntityManager::class,
    DefaultEntityManager::class,
);
```

La sintaxis es conceptual, no especificación definitiva del Container API.

---

# 300. Factory example

```php
final class EntityManagerFactory
{
    public function __construct(
        private MetadataRegistry $metadata,
        private ConnectionManager $connections,
        private EntityManagerConfiguration $configuration,
    ) {}

    public function create(
        DatabaseContext $context,
        IdentityMap $identityMap,
        UnitOfWork $unitOfWork,
    ): EntityManager {
        return new DefaultEntityManager(
            context: $context,
            metadata: $this->metadata,
            connections: $this->connections,
            identityMap: $identityMap,
            unitOfWork: $unitOfWork,
            configuration: $this->configuration,
        );
    }
}
```

---

# 301. Important observation

`EntityManagerFactory` puede ser shared.

El `EntityManager` producido no.

---

# 302. Service graph example

```text
Root Container
│
├── DatabaseConfiguration [SHARED]
├── DriverRegistry [SHARED]
├── TypeRegistry [SHARED]
├── MetadataRegistry [SHARED]
├── CompilerRegistry [SHARED]
├── ConnectionInfrastructure [SHARED]
├── EntityManagerFactory [SHARED]
│
└── Operation Scope
    │
    ├── DatabaseContext [SCOPED]
    ├── QueryContext [SCOPED]
    ├── TransactionContext [SCOPED]
    ├── IdentityMap [SCOPED]
    ├── UnitOfWork [SCOPED]
    ├── EntityManager [SCOPED]
    ├── Repository instances [SCOPED/TRANSIENT]
    └── Connection leases [RESOURCE]
```

---

# 303. Final rule

La arquitectura completa deberá obedecer:

```text
Shared Container
=
Definitions + Immutable Infrastructure

Operation Scope
=
Mutable Database Execution State

Resource Managers
=
Externally/Explicitly Managed Runtime Resources
```

---

# 304. Design consequence

Esto permite que VoltStack conserve una experiencia simple:

```php
final class CreateUser
{
    public function __construct(
        private EntityManager $entities,
    ) {}

    public function execute(User $user): void
    {
        $this->entities->persist($user);
        $this->entities->flush();
    }
}
```

sin que el desarrollador tenga que gestionar manualmente:

```text
EntityManager creation
IdentityMap creation
UnitOfWork creation
scope binding
scope reset
worker cleanup
```

---

# 305. Pero la simplicidad no oculta semántica

Aunque DI simplifique construcción:

```text
persist()
≠
INSERT
```

```text
flush()
≠
COMMIT
```

```text
request scope
≠
transaction
```

```text
EntityManager
≠
singleton
```

---

# 306. Resultado arquitectónico

La integración con el Container deberá conseguir simultáneamente:

```text
Developer Ergonomics
+
Explicit Architecture
+
Runtime Isolation
+
Persistent Worker Safety
+
Testability
+
Extensibility
+
Performance
```

---

# 307. Principio definitivo

> **El Service Container de VoltStack deberá conocer cómo construir y durante cuánto tiempo conservar un componente Database, pero nunca deberá convertirse en sustituto de los sistemas que determinan la identidad, consistencia, transacción, conexión o persistencia de Database.**

Por tanto:

```text
Container
manages object lifecycle

Database
manages database semantics
```

---

# 308. Checklist de implementación

Antes de considerar terminada la integración Container deberán verificarse:

- [ ] `DatabaseServiceProvider` definido.
- [ ] bootstrap registration separado de boot.
- [ ] shared/scoped/transient definidos formalmente.
- [ ] `DatabaseOperationScope` implementado.
- [ ] `DatabaseScopeAccessor` implementado.
- [ ] EntityManager scoped.
- [ ] UnitOfWork scoped.
- [ ] IdentityMap scoped.
- [ ] TransactionContext scoped.
- [ ] DatabaseContext scoped.
- [ ] registries immutable/shared.
- [ ] metadata immutable/shared.
- [ ] factories correctamente clasificadas.
- [ ] connection lifecycle separado de DI lifecycle.
- [ ] connection pool y lease diferenciados.
- [ ] captive dependency validation implementada.
- [ ] circular dependency detection implementada.
- [ ] optional integrations condicionadas.
- [ ] extension registration antes del freeze.
- [ ] registry freeze implementado.
- [ ] testing overrides soportados.
- [ ] scope cleanup determinista.
- [ ] cleanup failure observable.
- [ ] FrankenPHP isolation tests.
- [ ] RoadRunner adapter preparado.
- [ ] OpenSwoole coroutine isolation preparada.
- [ ] diagnostics con redaction.
- [ ] container compilation soportada.
- [ ] no Container lookups innecesarios en hot paths.
- [ ] leak tests para persistent workers.
- [ ] architecture tests para lifetimes.

---

# 309. Relación con documentos anteriores

Este documento implementa directamente principios establecidos en:

```text
07_DATABASE_BOOTSTRAP_AND_SERVICE_CONTAINER_INTEGRATION
08_DATABASE_LIFECYCLE_AND_RUNTIME_MODEL
16_DATABASE_CONNECTION_STATE_AND_RESET_SYSTEM
118_DATABASE_ENTITY_MANAGER_SYSTEM
123_DATABASE_IDENTITY_MAP_SYSTEM
124_DATABASE_UNIT_OF_WORK_ARCHITECTURE
164_DATABASE_TRANSACTION_ARCHITECTURE
165_DATABASE_TRANSACTION_MANAGER_SYSTEM
166_DATABASE_TRANSACTION_CONTEXT_SYSTEM
251_DATABASE_PERSISTENT_RUNTIME_ARCHITECTURE
252_DATABASE_REQUEST_SCOPE_SYSTEM
253_DATABASE_DATABASE_CONTEXT_SYSTEM
254_DATABASE_STATE_ISOLATION_SYSTEM
255_DATABASE_STATE_RESET_SYSTEM
256_DATABASE_CONNECTION_REUSE_SYSTEM
257_DATABASE_WORKER_LIFECYCLE_SYSTEM
258_DATABASE_FRANKENPHP_INTEGRATION_SYSTEM
259_DATABASE_ROADRUNNER_INTEGRATION_SYSTEM
260_DATABASE_OPENSWOOLE_INTEGRATION_SYSTEM
294_DATABASE_EXTENSION_ARCHITECTURE
295_DATABASE_PLUGIN_SYSTEM
302_DATABASE_PUBLIC_API_SYSTEM
303_DATABASE_FACADE_SYSTEM
311_DATABASE_FRAMEWORK_INTEGRATION_ARCHITECTURE
```

El objetivo del presente documento no es redefinir esos subsistemas, sino especificar cómo sus lifetimes y dependencias se materializan mediante el Container.

---

# 310. Cierre

La integración con el Service Container constituye una frontera crítica entre la arquitectura estática de VoltStack Database y su comportamiento real en ejecución.

Un diseño incorrecto podría funcionar aparentemente bajo el modelo tradicional:

```text
one PHP process
→ one request
→ process destroyed
```

pero fallar gravemente bajo:

```text
FrankenPHP
RoadRunner
OpenSwoole
Queue Workers
Long-running CLI
```

Por ello VoltStack diseñará Database desde el inicio bajo el modelo:

```text
Root Container
        │
        ├── Shared Immutable Infrastructure
        │
        └── Operation Scope
             ├── DatabaseContext
             ├── EntityManager
             ├── UnitOfWork
             ├── IdentityMap
             ├── TransactionContext
             └── Resource Leases
```

con una separación estricta:

```text
Process Lifetime
≠
Operation Lifetime
≠
Transaction Lifetime
≠
Connection Lifetime
≠
Entity Lifetime
```

Esta separación será una de las garantías fundamentales que permitirán a VoltStack ofrecer ergonomía similar a frameworks PHP tradicionales sin heredar supuestos incompatibles con su arquitectura persistente y reactiva.

---

# 311. Siguiente documento

```text
313_DATABASE_CONFIG_INTEGRATION_SYSTEM.md
```

Definirá cómo `Quantum/Database` se integra con el sistema de configuración de VoltStack, incluyendo:

```text
database configuration loading
typed configuration objects
connection profiles
environment variable resolution
secret references
configuration normalization
validation
defaults
configuration inheritance
environment overrides
runtime configuration
immutable configuration
configuration compilation
configuration cache
configuration generations
hot reload
persistent worker behavior
driver-specific configuration
pool configuration
timeout configuration
cache configuration
telemetry configuration
security configuration
multitenancy configuration bridges
```

y formalizará la siguiente regla:

```text
Raw Configuration
        ↓
Resolution
        ↓
Normalization
        ↓
Validation
        ↓
Typed Immutable Database Configuration
```

de modo que el resto de Database nunca dependa directamente de arrays arbitrarios, `.env`, variables globales o detalles del origen de configuración.