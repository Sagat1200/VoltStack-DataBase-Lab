# 07_DATABASE_BOOTSTRAP_AND_SERVICE_CONTAINER_INTEGRATION.md

# VoltStack Quantum Database
## Bootstrap and Service Container Integration

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 07 — Database Bootstrap and Service Container Integration  
**Estado:** Architecture Specification  
**Nivel:** Core Architecture  
**Versión:** 1.0

---

# 1. Propósito

Este documento define cómo:

```text
VoltStack/Quantum/Database
```

se registra, construye, configura, inicializa y conecta con el Service Container de VoltStack.

El Bootstrap System deberá transformar:

```text
CompiledDatabaseConfiguration
```

en un grafo funcional de servicios Database sin:

- abrir conexiones innecesariamente;
- capturar estado request-scoped dentro de singletons;
- introducir dependencias circulares;
- convertir el Container en Service Locator;
- inicializar ORM cuando no se utiliza;
- obligar a instalar integraciones opcionales;
- introducir estado global mutable;
- comprometer runtimes persistentes.

---

# 2. Principio fundamental

La regla central será:

> El Container construye el grafo de Database; Database no utiliza el Container como mecanismo de resolución durante su lógica ordinaria.

Por tanto:

```text
Container
   │
   ▼
construct services
   │
   ▼
explicit dependency graph
```

y no:

```text
Database Service
      │
      ▼
Container::get(...)
      │
      ▼
hidden dependency
```

---

# 3. Objetivos

El sistema deberá proporcionar:

```text
deterministic bootstrap
explicit service lifetimes
lazy resource creation
optional integrations
extension registration
compiled configuration
runtime-safe scoping
container compilation compatibility
architecture validation
```

---

# 4. Flujo general

```text
Application Boot
      │
      ▼
Platform Boot
      │
      ▼
Quantum/Config
      │
      ▼
Database Package Registration
      │
      ▼
Database Extension Discovery
      │
      ▼
Configuration Compilation
      │
      ▼
Database Composition Root
      │
      ▼
Service Definitions
      │
      ▼
Container Compilation
      │
      ▼
Persistent Services Ready
      │
      ▼
Execution Scope Begins
      │
      ▼
Scoped Database Services
```

---

# 5. Dos fases principales

Database distinguirá:

```text
Registration Phase
```

de:

```text
Runtime Phase
```

## Registration Phase

Ocurre durante bootstrap.

Construye:

- registries;
- factories;
- compiler services;
- metadata infrastructure;
- integration adapters;
- service definitions;
- lifecycle infrastructure.

## Runtime Phase

Ocurre durante cada:

```text
HTTP Request
Queue Job
CLI Operation
Scheduled Task
WebSocket Operation
```

y crea estado scoped.

---

# 6. Composition Root

La composición completa de Database deberá concentrarse conceptualmente en:

```text
DatabaseCompositionRoot
```

Su responsabilidad será conectar:

```text
configuration
contracts
implementations
registries
factories
adapters
lifetimes
extensions
```

No ejecutará lógica de dominio Database.

---

# 7. DatabaseServiceProvider

VoltStack podrá exponer:

```php
final class DatabaseServiceProvider
{
    public function register(...): void
    {
    }

    public function boot(...): void
    {
    }
}
```

según el sistema final de Bootstrap de VoltStack.

La API concreta dependerá de `Quantum/Bootstrap`.

---

# 8. Responsabilidad del Service Provider

Será responsable de iniciar la composición.

No deberá contener directamente toda la arquitectura.

Preferencia:

```text
DatabaseServiceProvider
        │
        ▼
DatabaseBootstrapper
        │
        ▼
DatabaseCompositionRoot
```

Esto evita un Service Provider gigantesco.

---

# 9. Bootstrapper

Conceptualmente:

```php
final class DatabaseBootstrapper
{
    public function bootstrap(
        CompiledDatabaseConfiguration $configuration,
        ServiceRegistry $services,
    ): void {
    }
}
```

Su función será coordinar las fases de registro.

---

# 10. Fases oficiales de Bootstrap

Se recomienda:

```text
Phase 1  Core Contracts
Phase 2  Configuration
Phase 3  Extension Discovery
Phase 4  Driver Infrastructure
Phase 5  Platform/Dialect Infrastructure
Phase 6  Type System
Phase 7  Connection Infrastructure
Phase 8  Query Engine
Phase 9  Schema/Migrations
Phase 10 ORM
Phase 11 Transactions
Phase 12 Integrations
Phase 13 Runtime Lifecycle
Phase 14 Public APIs
Phase 15 Validation
Phase 16 Freeze/Compile
```

El orden deberá ser determinista.

---

# 11. Service Lifetimes

Database reconocerá al menos:

```text
Singleton
Scoped
Transient
Resource
```

---

# 12. Singleton

Un Singleton puede sobrevivir durante todo el worker/process.

Adecuado para servicios:

```text
immutable
stateless
concurrency-safe
configuration-derived
```

Ejemplos:

```text
Dialect
DatabasePlatform descriptor
TypeRegistry
CompilerRegistry
OptimizationRuleRegistry
CompiledMetadataRegistry
Configuration
```

---

# 13. Scoped

Un servicio Scoped pertenece a un Execution Scope.

Ejemplos:

```text
DatabaseContext
EntityManager
UnitOfWork
IdentityMap
PersistenceContext
TransactionContext
Tenant Database Context
HydrationContext
```

Debe destruirse o resetearse al terminar el scope.

---

# 14. Transient

Un servicio Transient se crea según necesidad.

Ejemplos potenciales:

```text
QueryBuilder
SchemaBuilder
temporary planner
operation context
```

si no requieren identidad compartida.

---

# 15. Resource

Los recursos externos requieren lifecycle especial.

Ejemplos:

```text
physical connection
statement
cursor
stream
transaction
```

No son simples singletons/scoped services.

Necesitan ownership y cleanup explícitos.

---

# 16. Regla crítica de lifetimes

Está prohibido:

```text
Singleton
   │
   ▼
Scoped Dependency
```

cuando el Singleton captura la instancia.

Ejemplo incorrecto:

```php
final class QueryCompiler
{
    public function __construct(
        private EntityManagerInterface $entityManager,
    ) {}
}
```

si `QueryCompiler` es singleton y `EntityManager` scoped.

---

# 17. Lifetime Dependency Direction

Permitido:

```text
Scoped
  │
  ▼
Singleton
```

Ejemplo:

```text
EntityManager (scoped)
      │
      ▼
MetadataRegistry (singleton)
```

No permitido:

```text
MetadataRegistry (singleton)
      │
      ▼
EntityManager (scoped)
```

---

# 18. Lifetime Matrix

| Consumer | Dependency | Estado |
|---|---|---|
| Singleton | Singleton | Permitido |
| Singleton | Scoped | Prohibido por captura directa |
| Singleton | Transient stateless | Condicional |
| Scoped | Singleton | Permitido |
| Scoped | Scoped | Permitido |
| Scoped | Transient | Permitido |
| Transient | Singleton | Permitido |
| Transient | Scoped | Sólo dentro del scope correcto |

---

# 19. Scoped Provider Pattern

Cuando un singleton necesite obtener un recurso scoped dinámicamente, deberá utilizarse una abstracción explícita y limitada.

Ejemplo:

```text
DatabaseContextProvider
```

en lugar de:

```text
Container
```

Pero deberá evitarse incluso este patrón cuando una dependencia explícita pueda resolver el problema.

---

# 20. Container no es Runtime Context

Nunca deberá utilizarse:

```text
Container
```

como almacenamiento de:

```text
current transaction
current tenant
current EntityManager
current query
current connection
```

Ese estado pertenece a contexts específicos.

---

# 21. Database Context

Cada Execution Scope dispondrá de:

```text
DatabaseContext
```

Conceptualmente:

```text
ExecutionScope
     │
     ▼
DatabaseContext
     │
     ├── PersistenceContext
     ├── TransactionContext
     ├── ConnectionScope
     └── optional TenantDatabaseContext
```

---

# 22. Request Scope

En HTTP:

```text
HTTP Request
    │
    ▼
ExecutionScope
    │
    ▼
DatabaseContext
```

Cuando termina el request:

```text
DatabaseContext
    │
    ▼
cleanup
    │
    ▼
destroy/reset
```

---

# 23. Queue Job Scope

El mismo modelo deberá aplicarse a:

```text
Queue Worker
   │
   ├── Job A
   │      └── DatabaseContext A
   │
   └── Job B
          └── DatabaseContext B
```

Nunca:

```text
Job A EntityManager
       │
       ▼
reused by Job B
```

---

# 24. CLI Scope

Un comando CLI podrá representar un Execution Scope.

Ejemplo:

```text
php voltstack migrate
        │
        ▼
CLI Execution Scope
        │
        ▼
DatabaseContext
```

Comandos de larga duración podrán crear subscopes cuando sea necesario.

---

# 25. Persistent Worker Model

Con FrankenPHP:

```text
Worker
│
├── Persistent Database Services
│
├── Request A
│   └── DatabaseContext A
│
├── RESET
│
├── Request B
│   └── DatabaseContext B
│
├── RESET
│
└── Request C
    └── DatabaseContext C
```

---

# 26. Servicios persistentes

Podrán sobrevivir:

```text
CompiledDatabaseConfiguration
DriverRegistry
DialectRegistry
PlatformRegistry
TypeRegistry
CompilerRegistry
OptimizationRuleRegistry
Compiled Metadata
immutable descriptors
stateless factories
```

---

# 27. Estado que nunca debe sobrevivir

No deberá persistir entre requests:

```text
EntityManager state
IdentityMap
UnitOfWork
ChangeSets
TransactionContext
active transaction
tenant selection
query operation state
HydrationContext
open cursor
request-specific connection state
```

---

# 28. Connection Manager Lifetime

`ConnectionManager` requiere especial atención.

Puede existir una parte persistente:

```text
ConnectionManager
```

responsable de:

```text
definitions
factories
pool coordination
topology
```

pero el estado de uso actual deberá vivir en:

```text
ConnectionScope
```

---

# 29. Separación Manager/Scope

Preferencia:

```text
Persistent ConnectionManager
          │
          ▼
Scoped ConnectionScope
          │
          ▼
leased connections
```

Esto evita almacenar:

```text
$currentConnection
```

en un singleton.

---

# 30. Connection Lease

En pooling:

```text
Pool
 │
 ▼
Connection Lease
 │
 ▼
Execution Scope
```

El scope no se convierte necesariamente en propietario de la conexión física.

Es propietario del lease.

---

# 31. Connection Factory

`ConnectionFactory` podrá ser singleton si:

```text
stateless
configuration-safe
thread-safe
```

Su salida será un recurso con lifecycle independiente.

---

# 32. Lazy Connection Creation

Bootstrap no deberá ejecutar:

```text
connect()
```

por defecto.

Flujo:

```text
Application Boot
     │
     ▼
Connection Definitions
     │
     ▼
No DB Connection
     │
     ▼
First Query
     │
     ▼
Connection Acquire/Create
```

---

# 33. Excepciones al Lazy Boot

Algunos comandos podrán solicitar conexión durante boot operativo:

```text
db:health
migrate
schema:inspect
```

pero será una decisión de la operación, no del Service Provider.

---

# 34. Driver Registry

Durante bootstrap se construirá:

```text
DriverRegistry
```

Inicialmente:

```text
mysql
mariadb
pgsql
sqlite
```

Los paquetes podrán registrar drivers adicionales.

---

# 35. Driver Registry Lifetime

Deberá ser:

```text
persistent
immutable after freeze
```

durante runtime normal.

---

# 36. Driver Registration

Conceptualmente:

```php
$drivers->register(
    'pgsql',
    PostgreSqlDriverFactory::class,
);
```

La API definitiva deberá evitar instanciación innecesaria.

---

# 37. Driver Factory

Un registro podrá apuntar a:

```text
DriverFactory
```

en vez de mantener instancias cuando el driver tenga construcción contextual.

---

# 38. Dialect Registry

Separado de Driver Registry:

```text
DialectRegistry
```

Ejemplos:

```text
mysql
mariadb
postgresql
sqlite
```

La separación mantiene:

```text
Driver != Dialect
```

---

# 39. Platform Registry

Separado:

```text
PlatformRegistry
```

Responsable de descriptors/capabilities.

Esto mantiene:

```text
Driver != Dialect != Platform
```

---

# 40. Type Registry

`TypeRegistry` contendrá tipos Database registrados.

Ejemplos:

```text
integer
string
boolean
decimal
datetime
json
uuid
```

y extensiones.

---

# 41. Type Registry Freeze

Después del bootstrap:

```text
TypeRegistry
     │
     ▼
freeze
```

para garantizar compilación determinista.

Hot registration durante requests estará prohibido por defecto.

---

# 42. Query Engine Bootstrap

El Query Engine deberá componerse por capas.

```text
Query API
   │
   ▼
Normalizer
   │
   ▼
Validator
   │
   ▼
Semantic Analyzer
   │
   ▼
Optimizer
   │
   ▼
Planner
   │
   ▼
Compiler
   │
   ▼
Executor
```

---

# 43. Query Engine Lifetimes

Servicios preferentemente persistentes:

```text
normalizers
validators
semantic rules
optimization rules
planner strategies
compiler registry
stateless compiler services
```

Estado de una query:

```text
QueryContext
SemanticContext
PlanningContext
ExecutionContext
```

será operation-scoped.

---

# 44. Query Builder

No deberá registrarse como singleton.

Normalmente:

```text
QueryBuilder
→ transient
```

porque representa una consulta en construcción.

---

# 45. AST Nodes

No son servicios del Container.

Son objetos de dominio creados durante construcción de queries.

Nunca:

```text
container->get(ColumnExpression::class)
```

---

# 46. Query Models

Tampoco deberán registrarse individualmente.

El Container construye servicios, no cada Value Object del sistema.

---

# 47. Compiler Registry

Podrá mapear:

```text
platform/query type
        │
        ▼
compiler
```

Ejemplo conceptual:

```text
PostgreSQL + SELECT
→ PostgreSqlSelectCompiler
```

La implementación concreta podrá usar compiler composition en lugar de un compiler por query type.

---

# 48. Optimizer Bootstrap

`OptimizationRuleRegistry` será construido durante bootstrap.

Ejemplo:

```text
OptimizationRuleRegistry
├── PredicateSimplificationRule
├── RedundantOrderRule
├── ConstantExpressionRule
└── Extension Rules
```

El orden deberá ser determinista.

---

# 49. Rule Priorities

La prioridad se resolverá durante bootstrap.

No ordenar rules en cada query.

```text
Registered Rules
      │
      ▼
Compile Pipeline
      │
      ▼
Immutable Ordered Rule Set
```

---

# 50. Schema Bootstrap

Schema services podrán registrarse aunque no se utilicen inmediatamente.

Preferentemente serán stateless/lazy.

Ejemplos:

```text
SchemaManagerFactory
SchemaIntrospectorRegistry
SchemaDiffer
SchemaPlanner
SchemaCompilerRegistry
```

---

# 51. Migration Bootstrap

Migration infrastructure puede ser parcialmente lazy.

Servicios:

```text
MigrationDiscovery
MigrationRepositoryFactory
MigrationPlanner
MigrationExecutor
MigrationLockManager
```

No deberá escanear necesariamente todos los archivos en cada HTTP worker boot.

---

# 52. Migration Discovery

En producción normal HTTP:

```text
MigrationDiscovery
```

no debería ejecutarse salvo que sea necesario.

Los comandos CLI podrán activarlo.

---

# 53. ORM Bootstrap

Si:

```text
orm.enabled = false
```

el bootstrap deberá omitir servicios ORM costosos.

---

# 54. ORM Lazy Initialization

Incluso con ORM habilitado:

```text
EntityManager
```

no deberá construirse hasta que el scope lo solicite.

---

# 55. EntityManager Lifetime

Regla:

```text
EntityManager
→ scoped
```

Nunca:

```text
EntityManager
→ process singleton
```

---

# 56. UnitOfWork Lifetime

```text
UnitOfWork
→ scoped
```

y normalmente vinculado al mismo PersistenceContext que EntityManager.

---

# 57. IdentityMap Lifetime

```text
IdentityMap
→ scoped
```

Debe comenzar vacío en cada scope.

---

# 58. Metadata Registry Lifetime

En contraste:

```text
CompiledMetadataRegistry
→ persistent
```

porque metadata compilada es:

```text
immutable
application-level
shareable
```

---

# 59. Metadata Source Lifecycle

En development:

```text
Attribute Metadata Loader
```

podrá permanecer disponible.

En producción:

```text
CompiledMetadataRegistry
```

deberá ser preferido.

---

# 60. Repository Factory

Puede ser persistent/stateless.

Los repositories concretos podrán ser:

```text
scoped
```

si capturan `EntityManager`.

---

# 61. Repository Lifetime

Si:

```php
final class UserRepository
{
    public function __construct(
        private EntityManagerInterface $entityManager
    ) {}
}
```

entonces:

```text
UserRepository
→ scoped
```

Nunca singleton.

---

# 62. Static Model API

Una API tipo:

```php
User::query()
```

no deberá guardar un EntityManager global estático.

Deberá resolver el contexto actual mediante infraestructura controlada.

---

# 63. Active Record Context Bridge

Podrá existir:

```text
ModelRuntimeContext
```

o un port equivalente para conectar APIs estáticas ergonómicas con el scope actual.

Debe evitarse convertirlo en Service Locator universal.

---

# 64. Active Record Restriction

El bridge sólo podrá resolver capacidades ORM estrictamente necesarias.

No deberá proporcionar:

```text
container()
getAnything()
```

---

# 65. Transaction Manager Bootstrap

La infraestructura transaccional podrá ser persistent/stateless.

Pero:

```text
TransactionContext
```

será scoped.

---

# 66. Transaction State

Nunca almacenar:

```text
$currentTransaction
```

en:

```text
TransactionManager singleton
```

La información activa pertenece al scope/context.

---

# 67. Nested Transactions

La composición deberá proporcionar:

```text
SavepointStrategy
TransactionCapabilityResolver
RetryPolicy
```

sin hardcodear decisiones dentro del Container.

---

# 68. Integration Bootstrap

Integraciones opcionales:

```text
Quantum/Cache
Quantum/EventSystem
Quantum/Telemetry
Multitenancy
Validation
Authorization
Jobs
RuntimeManagerServer
```

deberán conectarse mediante adapters.

---

# 69. Integration Detection

La detección deberá ocurrir durante bootstrap.

No durante cada query.

Incorrecto:

```php
if (class_exists(Telemetry::class)) {
}
```

en `QueryExecutor`.

Correcto:

```text
Bootstrap
   │
   ├── Telemetry installed
   │      └── QuantumTelemetryAdapter
   │
   └── not installed
          └── NullDatabaseTelemetry
```

---

# 70. Cache Adapter

Si `Quantum/Cache` está instalado:

```text
DatabaseCacheInterface
       │
       ▼
QuantumCacheAdapter
```

Si no:

```text
DatabaseCacheInterface
       │
       ▼
Null/Internal Adapter
```

según el tipo de cache.

---

# 71. Event Adapter

```text
DatabaseEventDispatcherInterface
        │
        ├── QuantumEventAdapter
        └── NullDatabaseEventDispatcher
```

---

# 72. Telemetry Adapter

```text
DatabaseTelemetryInterface
        │
        ├── QuantumTelemetryAdapter
        └── NullDatabaseTelemetry
```

---

# 73. Multitenancy Adapter

Cuando el paquete Multitenancy esté instalado:

```text
Multitenancy
     │
     ▼
TenantDatabaseAdapter
     │
     ▼
Database Topology/Connection Resolution
```

Database Core no dependerá del paquete Multitenancy.

---

# 74. Validation Integration

La validación de entidades o DTOs puede integrarse mediante port.

Pero Database Mapping Validation seguirá siendo responsabilidad propia.

No confundir:

```text
ORM mapping validation
```

con:

```text
application data validation
```

---

# 75. Authorization Integration

Database no deberá preguntar directamente:

```text
can user update entity?
```

durante persistence ordinaria.

Authorization pertenece a capas superiores salvo mecanismos específicos de data access expresamente diseñados.

---

# 76. RuntimeManagerServer Integration

La integración con servidores persistentes deberá conectarse mediante:

```text
DatabaseLifecycleInterface
```

o contratos equivalentes.

---

# 77. FrankenPHP Adapter

Modelo:

```text
FrankenPHP Worker
       │
       ▼
VoltStack Runtime
       │
       ▼
Database Runtime Adapter
       │
       ▼
DatabaseLifecycle
```

---

# 78. Request Start

En inicio:

```text
beginExecutionScope()
```

deberá crear:

```text
DatabaseContext
PersistenceContext
TransactionContext
ConnectionScope
```

de forma lazy cuando sea posible.

---

# 79. Request End

Al terminar:

```text
endExecutionScope()
```

deberá garantizar:

```text
flush policy check
open transaction detection
cursor cleanup
connection release/reset
UnitOfWork disposal
IdentityMap disposal
tenant state disposal
context disposal
```

---

# 80. Auto Flush

Por defecto, terminar un request no deberá significar automáticamente:

```text
EntityManager::flush()
```

porque puede persistir cambios accidentalmente.

El flush deberá ser explícito salvo API de alto nivel con semántica claramente definida.

---

# 81. Orphan Transaction

Si al terminar scope existe una transacción activa:

```text
active transaction
       │
       ▼
ROLLBACK
```

y deberá generarse diagnóstico apropiado.

Nunca commit implícito.

---

# 82. Dirty UnitOfWork

Si existe un UnitOfWork con cambios no flushed:

```text
development
→ warning/diagnostic

production
→ discard scoped state
```

según policy.

Nunca auto-flush silencioso.

---

# 83. Cursor Cleanup

Cursors abiertos deberán cerrarse antes de liberar la conexión.

```text
Cursor
  │
  ▼
close
  │
  ▼
connection reset/release
```

---

# 84. Connection Cleanup

Una conexión deberá ser:

```text
reset
validate if necessary
release to pool
```

o:

```text
discard
```

si no puede garantizarse estado limpio.

---

# 85. Poisoned Connection

Una conexión deberá marcarse como no reutilizable si ocurre:

```text
protocol error
failed reset
broken transaction state
connection loss
unknown session state
```

---

# 86. Worker Recycle Signal

Errores graves de cleanup podrán escalar a:

```text
worker recycle requested
```

mediante Runtime Adapter.

Database no deberá matar procesos directamente.

---

# 87. RoadRunner Adapter

En el futuro:

```text
RoadRunner
    │
    ▼
Runtime Adapter
    │
    ▼
DatabaseLifecycle
```

utilizará la misma semántica de scope.

---

# 88. OpenSwoole Adapter

Igualmente:

```text
OpenSwoole Coroutine/Request
        │
        ▼
Execution Scope
        │
        ▼
DatabaseContext
```

con aislamiento compatible con concurrencia.

---

# 89. Coroutine Safety

En runtimes concurrentes:

```text
current context
```

no podrá almacenarse en una variable estática global.

Deberá utilizar infraestructura scope-aware.

---

# 90. Context Propagation

El Runtime System será responsable de propagar:

```text
ExecutionScope
```

correctamente hacia Database.

Database no deberá inferirlo mediante globals.

---

# 91. Service Container Scope Support

El Container de VoltStack deberá soportar:

```text
scope begin
scope resolve
scope end
```

o una abstracción equivalente.

Database dependerá de esa capacidad de Platform/Container, no de FrankenPHP directamente.

---

# 92. Scope Hierarchy

Podrá existir:

```text
Process
   │
   ▼
Worker
   │
   ▼
Execution Scope
   │
   ▼
Transaction/Operation
```

No todos estos niveles necesitan ser Container scopes formales.

---

# 93. Operation Context

Query operations deberán usar objetos creados explícitamente.

Ejemplo:

```text
ExecutionContext
```

No será necesario abrir un Container scope por query.

---

# 94. Transaction Context

Una transacción podrá ser un sub-lifecycle lógico dentro del Execution Scope.

```text
Execution Scope
    │
    └── Transaction Context
```

---

# 95. Container Compilation

VoltStack podrá compilar definiciones del Container en producción.

Database deberá ser compatible con:

```text
compiled service graph
```

y evitar resolución dinámica innecesaria.

---

# 96. Compile-Time Resolution

Cuando sea posible:

```text
interface
   │
   ▼
implementation
```

deberá resolverse durante container compilation.

---

# 97. Dynamic Resolution

Se reservará para casos legítimos:

```text
named connections
tenant connection definitions
driver extensions
runtime topology
repositories by entity
```

y se realizará mediante registries/factories específicos.

---

# 98. Named Services

El Container no deberá registrar necesariamente:

```text
connection.primary
connection.analytics
connection.audit
```

como conexiones físicas ya creadas.

Preferencia:

```text
ConnectionManager
       │
       ▼
named definitions
       │
       ▼
lazy resource
```

---

# 99. Container Definition vs Runtime Instance

Diferencia:

```text
Service Definition
→ how to create a service

Connection Definition
→ how to create/acquire DB resource
```

No son lo mismo.

---

# 100. Service Aliases

Public contracts podrán tener aliases:

```text
ConnectionManagerInterface
→ ConnectionManager

EntityManagerInterface
→ ScopedEntityManager

DatabaseTelemetryInterface
→ configured adapter
```

---

# 101. Conditional Services

ORM services podrán registrarse condicionalmente según configuración.

Pero la condición deberá evaluarse durante bootstrap.

No en cada resolución.

---

# 102. Feature Modules

Database podrá organizar bootstrap mediante módulos:

```text
ConnectionModule
QueryModule
SchemaModule
MigrationModule
OrmModule
TransactionModule
RuntimeModule
IntegrationModule
```

Cada uno registra sólo su dominio.

---

# 103. Module Bootstrap Contract

Conceptualmente:

```php
interface DatabaseBootstrapModuleInterface
{
    public function register(
        DatabaseServiceRegistry $services,
        DatabaseBootstrapContext $context
    ): void;
}
```

Podrá ser Internal API.

---

# 104. Bootstrap Context

Deberá contener únicamente información necesaria:

```text
compiled configuration
registries
extension descriptors
environment profile
```

No deberá convertirse en Service Locator.

---

# 105. Module Ordering

La dependencia entre bootstrap modules deberá declararse.

Ejemplo:

```text
Driver
   ↓
Connection
   ↓
Execution
   ↓
ORM
```

No depender del orden de archivos.

---

# 106. Bootstrap Dependency Graph

```text
Configuration
     │
     ▼
Driver ─────► Platform/Dialect
     │              │
     └──────┬───────┘
            ▼
       Connection
            │
            ▼
        Execution
            ▲
            │
Query Model → Semantic → Optimizer → Planner → Compiler
            │
            ▼
           ORM
```

---

# 107. Cycle Detection

El Bootstrap System deberá poder detectar ciclos entre módulos/servicios cuando sea posible.

Ejemplo inválido:

```text
EntityManager
    ↓
RepositoryFactory
    ↓
ConnectionManager
    ↓
EntityManager
```

---

# 108. Lazy Dependencies

Una dependencia lazy no deberá utilizarse para ocultar ciclos arquitectónicos.

Incorrecto:

```text
A → Lazy<B>
B → Lazy<A>
```

si A/B realmente están acoplados circularmente.

---

# 109. Provider Injection

Providers específicos podrán resolver recursos dinámicos.

Ejemplo:

```text
ConnectionProvider
RepositoryProvider
```

Pero deberán tener una API limitada.

---

# 110. Container Injection Prohibition

Database Core no deberá declarar:

```php
public function __construct(
    ContainerInterface $container
)
```

salvo componentes explícitos de bootstrap/integration donde sea arquitectónicamente necesario.

---

# 111. Service Locator Detection

Architecture Tests deberán detectar dependencias de:

```text
ContainerInterface
```

fuera de namespaces autorizados como:

```text
Bootstrap
Integration
Container
```

---

# 112. Extension Bootstrap

Paquetes externos podrán registrar:

```text
drivers
dialects
types
optimization rules
compilers
metadata loaders
hydrators
configuration schemas
integration adapters
```

antes del freeze.

---

# 113. Extension Descriptor

Una extensión podrá declarar:

```text
name
version
capabilities
bootstrap module
configuration schema
dependencies
priority
```

mediante un descriptor.

---

# 114. Extension Discovery

La discovery puede ocurrir mediante:

```text
Composer metadata
VoltStack package manifest
compiled package registry
explicit configuration
```

La estrategia exacta pertenecerá al Package System.

---

# 115. No Runtime Package Discovery

En producción:

```text
scan Composer packages on every request
```

deberá evitarse.

La lista podrá compilarse durante application build/bootstrap.

---

# 116. Extension Dependency Validation

Ejemplo:

```text
PostGIS Extension
requires:
PostgreSQL Platform
```

La validación deberá ocurrir antes del runtime.

---

# 117. Extension Ordering

Cuando varias extensiones actúen sobre el mismo registry:

```text
priority
phase
dependency
```

deberán producir un orden determinista.

---

# 118. Registry Freeze

Registries estructurales deberán congelarse:

```text
DriverRegistry
DialectRegistry
PlatformRegistry
TypeRegistry
CompilerRegistry
OptimizationRuleRegistry
MetadataLoaderRegistry
```

antes del runtime ordinario.

---

# 119. Runtime Registries

Si existe necesidad de registros dinámicos, deberán separarse explícitamente de los compilados.

No hacer mutable todo el registry por una excepción.

---

# 120. Public Facade Bootstrap

La facade:

```php
DB
```

deberá apuntar a una API estable como:

```text
DatabaseManagerInterface
```

No directamente al Container.

---

# 121. Facade Resolution

Si VoltStack utiliza Facades, el mecanismo global de Facades podrá resolver el servicio actual.

Pero `Quantum/Database` no deberá implementar un Service Locator paralelo.

---

# 122. DB Facade

Conceptualmente:

```text
DB
 │
 ▼
DatabaseManagerInterface
 │
 ├── connection()
 ├── table()
 ├── transaction()
 └── ...
```

Las operaciones delegarán a servicios especializados.

---

# 123. Facade State

`DB` no deberá almacenar:

```text
current connection
current transaction
current tenant
current EntityManager
```

como propiedades globales.

---

# 124. Helper Bootstrap

Helpers como:

```php
db()
```

podrán ser wrappers sobre Public API.

No deberán introducir otra arquitectura de resolución.

---

# 125. EntityManager Public Resolution

Dependency Injection recomendado:

```php
final class UserService
{
    public function __construct(
        private EntityManagerInterface $entities
    ) {}
}
```

El Container resolverá la instancia scoped correcta.

---

# 126. Repository Autowiring

VoltStack podrá soportar:

```php
public function __construct(
    UserRepository $users
) {}
```

si existe metadata suficiente.

La implementación deberá construirse dentro del scope correcto.

---

# 127. Repository Compilation

En producción, mappings:

```text
UserRepository
→ User entity
→ repository factory
```

podrán compilarse para evitar reflexión repetida.

---

# 128. Model Autowiring

Las entidades no deberán registrarse como servicios por defecto.

```text
User Entity
```

es dato de dominio, no service.

---

# 129. Migration Autowiring

Las Migration classes podrán recibir servicios limitados si la arquitectura final lo permite.

Pero deberá evitarse darles el Container completo.

---

# 130. Migration Execution Scope

Una ejecución de migrations deberá crear un DatabaseContext propio.

```text
Migration Command
      │
      ▼
Execution Scope
      │
      ▼
Migration Context
```

---

# 131. Seeder Scope

Seeders también deberán ejecutarse dentro de un scope explícito.

Esto evita estado ORM retenido en procesos largos.

---

# 132. Long-Running Commands

Ejemplo:

```text
import 10 million rows
```

No deberá mantener indefinidamente:

```text
IdentityMap
UnitOfWork
EntityManager state
```

Podrá crear ciclos:

```text
process batch
   │
   ▼
flush
   │
   ▼
clear scoped persistence state
```

---

# 133. Scoped Reset API

Para procesos largos podrá existir una API controlada:

```text
PersistenceContext::clear()
```

sin destruir necesariamente todo el Execution Scope.

---

# 134. Bootstrap Error Model

Errores de composición deberán ser distintos de errores runtime.

Ejemplos:

```text
DatabaseBootstrapException
MissingDatabaseServiceException
CircularDatabaseDependencyException
InvalidServiceLifetimeException
ExtensionRegistrationException
RegistryFrozenException
```

---

# 135. Fail Fast

Errores como:

```text
unknown driver
missing required compiler
invalid service lifetime
duplicate extension ID
```

deberán fallar durante bootstrap cuando sea posible.

---

# 136. Deferred Errors

Errores que requieren conexión:

```text
invalid credentials
server unavailable
database missing
```

podrán ocurrir en first-use debido al lazy connection model.

---

# 137. Bootstrap Diagnostics

En development podrá mostrarse:

```text
Database Bootstrap
------------------
Default connection: primary
Driver: PostgreSQL
ORM: enabled
Metadata: compiled
Telemetry: QuantumTelemetryAdapter
Cache: QuantumCacheAdapter
Runtime: FrankenPHP
Strict reset: enabled
```

sin secretos.

---

# 138. Service Graph Diagnostics

CLI podrá inspeccionar:

```text
db:services
```

o equivalente.

Podrá mostrar:

```text
contract
implementation
lifetime
module
source
```

---

# 139. Lifetime Diagnostics

Especialmente útil:

```text
EntityManagerInterface
implementation: EntityManager
lifetime: scoped

MetadataRegistry
lifetime: singleton

QueryBuilder
lifetime: transient
```

---

# 140. Dependency Graph Inspection

Tooling podrá mostrar:

```text
EntityManager
├── MetadataProvider
├── UnitOfWork
├── IdentityMap
└── PersistenceEngine
```

para debugging arquitectónico.

---

# 141. Production Optimization

El bootstrap de producción podrá compilar:

```text
service definitions
registry maps
extension ordering
metadata registry
compiler registry
configuration
```

reduciendo trabajo por worker boot.

---

# 142. Preloading Compatibility

Servicios/clases inmutables podrán ser compatibles con:

```text
PHP OPcache preload
```

si el Runtime Manager lo utiliza.

No deberá asumirse que preload implica instancias globales mutables.

---

# 143. FrankenPHP Worker Boot

Modelo:

```text
Process Start
     │
     ▼
VoltStack Boot
     │
     ▼
Database Bootstrap
     │
     ▼
Persistent Service Graph
     │
     ▼
FrankenPHP Worker Loop
```

---

# 144. FrankenPHP Request Lifecycle

```text
Request Arrives
      │
      ▼
Execution Scope Begin
      │
      ▼
DatabaseContext
      │
      ▼
Application
      │
      ▼
Scope Termination
      │
      ▼
Database Cleanup
      │
      ▼
Connection Release
      │
      ▼
Scoped Services Destroyed
```

---

# 145. Scope Termination Ordering

Orden recomendado:

```text
1. stop new Database operations
2. close streams/cursors
3. detect active transaction
4. rollback orphan transaction
5. inspect dirty persistence state
6. clear UnitOfWork
7. clear IdentityMap
8. clear hydration/operation contexts
9. reset/release connections
10. clear tenant context
11. destroy DatabaseContext
```

El orden exacto podrá ajustarse según dependencias.

---

# 146. Cleanup Must Be Defensive

Si falla una etapa:

```text
cursor close failed
```

las etapas posteriores críticas deberán intentarse cuando sea seguro.

No abandonar cleanup inmediatamente dejando recursos vivos.

---

# 147. Cleanup Failure Aggregation

Múltiples fallos podrán agregarse en:

```text
DatabaseCleanupReport
```

para diagnostics.

El error original de aplicación no deberá perderse.

---

# 148. Exception Preservation

Si la aplicación ya está terminando por:

```text
ApplicationException
```

y cleanup genera:

```text
RollbackException
```

VoltStack deberá conservar ambos contextos.

No reemplazar silenciosamente la causa original.

---

# 149. Shutdown Fallback

Podrá existir cleanup defensivo de último recurso.

Pero:

```text
shutdown handler
```

no deberá ser el mecanismo principal de lifecycle.

---

# 150. Connection Pool Bootstrap

Si PoolMode es:

```text
Internal
```

se registrará infraestructura de pool.

Si:

```text
External
```

se registrará el adapter correspondiente.

Si:

```text
Disabled
```

no se inicializará pool innecesariamente.

---

# 151. Pool Lifetime

El Pool puede ser:

```text
worker/process scoped
```

si está diseñado para ello.

Los leases son:

```text
execution/operation scoped
```

---

# 152. Pool State Safety

El pool sólo podrá recibir conexiones:

```text
clean
validated according to policy
not in transaction
not carrying tenant/session leakage
```

---

# 153. Pool Warmup

Database no deberá abrir `min` conexiones automáticamente en todo escenario.

Podrá existir:

```text
lazy pool warmup
explicit warmup
```

según runtime/deployment.

---

# 154. Health Check Bootstrap

Health-check services podrán registrarse persistentemente.

Las verificaciones reales serán on-demand o coordinadas externamente.

---

# 155. Telemetry Bootstrap Safety

Telemetry no deberá crear dependencia circular:

```text
Database
  ↓
Telemetry
  ↓
Database
```

Si Telemetry almacena datos usando Database, deberá existir aislamiento arquitectónico.

---

# 156. Telemetry Recursion Guard

Database Telemetry deberá contemplar:

```text
telemetry export query
```

que no debe generar recursión infinita de telemetría Database.

---

# 157. Event Bootstrap Safety

Igualmente:

```text
Database Event
    ↓
Listener
    ↓
Database
```

es válido como comportamiento de aplicación, pero deberá respetar transacciones/reentrancy.

El Event Adapter no deberá crear ciclo de construcción.

---

# 158. Cache Bootstrap Safety

Si `Quantum/Cache` utiliza Database como backend, Database no podrá requerir ese cache para poder arrancar.

Ejemplo:

```text
Database Metadata Cache
      ↓
Quantum Cache
      ↓
Database Driver
```

puede crear ciclo.

---

# 159. Cache Dependency Cycle Prevention

Database deberá proporcionar:

```text
internal/null/bootstrap-safe cache
```

para caches críticos de bootstrap cuando el cache externo dependa de Database.

---

# 160. Bootstrap Dependency Tiers

Se recomienda:

```text
Tier 0
Platform / Container / Config

Tier 1
Database Core Infrastructure

Tier 2
Query / Schema / Transaction

Tier 3
ORM

Tier 4
Optional Integrations

Tier 5
Application Extensions
```

---

# 161. Tier Rule

Un tier inferior no deberá requerir un tier superior para inicializarse.

Especialmente:

```text
Driver
```

no deberá depender de:

```text
ORM
Telemetry adapter
Multitenancy package
```

---

# 162. Optional Integration Failure

Si una integración configurada explícitamente no puede inicializarse:

```text
telemetry.adapter = quantum
```

pero Quantum Telemetry no existe:

```text
bootstrap error
```

No fallback silencioso.

Si la integración no fue solicitada:

```text
Null Adapter
```

es válido.

---

# 163. Explicit vs Automatic Integration

Podrán existir modos:

```text
auto
enabled
disabled
```

Ejemplo:

```text
telemetry: auto
```

significa:

```text
use adapter if subsystem exists
otherwise Null Adapter
```

---

# 164. Database Module Manifest

Database podrá declarar al Package System:

```text
package name
service provider
bootstrap modules
extension points
configuration schema
public contracts
```

para discovery eficiente.

---

# 165. Boot Idempotency

Registrar Database dos veces accidentalmente no deberá duplicar:

```text
drivers
rules
listeners
types
compilers
```

El sistema deberá detectar:

```text
duplicate bootstrap
```

o mantener idempotencia donde sea apropiado.

---

# 166. Duplicate Registration

Una extensión intentando registrar dos veces el mismo ID deberá producir:

```text
DuplicateRegistrationException
```

salvo que la API permita override explícito.

---

# 167. Override Policy

Overrides deberán ser explícitos.

No:

```text
last registration wins
```

silenciosamente.

Preferencia:

```text
register
replace
decorate
```

como operaciones semánticamente distintas.

---

# 168. Service Decoration

Una extensión podrá decorar servicios autorizados.

Ejemplo:

```text
QueryExecutorInterface
       │
       ▼
TelemetryExecutorDecorator
       │
       ▼
NativeQueryExecutor
```

Sólo extension points documentados deberán garantizar esta capacidad.

---

# 169. Internal Service Replacement

No deberá prometerse compatibilidad al reemplazar servicios `INTERNAL`.

Las extensiones deberán depender de `EXTENSION` contracts.

---

# 170. Bootstrap API Stability

Las APIs utilizadas por paquetes externos deberán clasificarse:

```text
Public Bootstrap API
Extension Bootstrap API
Internal Bootstrap API
```

---

# 171. DatabaseBootstrapContext

No deberá exponer internals mutables arbitrarios.

Ejemplo conceptual:

```php
final readonly class DatabaseBootstrapContext
{
    public function __construct(
        public CompiledDatabaseConfiguration $configuration,
        public DatabaseEnvironment $environment,
        public DatabaseExtensionSet $extensions,
    ) {}
}
```

---

# 172. DatabaseServiceRegistry

Si se crea una abstracción propia, será un wrapper restringido del Container durante bootstrap.

Podrá permitir:

```text
singleton()
scoped()
transient()
alias()
decorate()
```

No necesariamente:

```text
resolve()
```

durante registration.

---

# 173. Registration vs Resolution

Regla:

```text
Registration phase
→ define services

Runtime phase
→ resolve services
```

Resolver servicios durante registration deberá evitarse salvo infraestructura explícita.

---

# 174. Premature Service Resolution

Problema:

```text
register()
   │
   ▼
resolve EntityManager
   │
   ▼
creates scoped service before scope exists
```

Debe producir error o impedirse.

---

# 175. Bootstrap Scope

El bootstrap puede tener su propio contexto, pero no deberá confundirse con un Application Execution Scope.

---

# 176. Compile-Time Services

Algunos servicios pueden existir sólo durante build:

```text
ConfigurationCompiler
ContainerCompiler
MetadataCompiler
PackageScanner
```

No necesitan permanecer cargados durante runtime production.

---

# 177. Runtime Service Graph

Producción podrá contener únicamente servicios necesarios para ejecución.

Esto reduce:

```text
memory
startup work
attack surface
```

---

# 178. Development Service Graph

Development podrá incluir:

```text
diagnostics
metadata scanners
configuration provenance
architecture inspectors
debug decorators
```

---

# 179. Environment-Neutral Semantics

Aunque el grafo pueda optimizarse distinto:

```text
development
production
```

la semántica funcional de Database deberá permanecer consistente.

---

# 180. Public Service Contracts

Candidatos principales para Container:

```text
DatabaseManagerInterface
ConnectionManagerInterface
TransactionManagerInterface
EntityManagerInterface
SchemaManagerInterface
```

No todos necesitan pertenecer a `Platform`.

---

# 181. Internal Service Contracts

Ejemplos:

```text
SemanticAnalyzerInterface
QueryOptimizerInterface
QueryPlannerInterface
QueryExecutorInterface
PersistencePlannerInterface
ConnectionResetterInterface
```

pueden estar registrados internamente.

---

# 182. Value Objects Are Not Services

No registrar:

```text
CompiledQuery
ConnectionName
EntityMetadata
ChangeSet
Query AST Node
DatabaseTarget
```

como servicios individuales.

---

# 183. Entity Metadata Exception

El `MetadataRegistry` sí es service.

`EntityMetadata` son valores almacenados por el registry.

---

# 184. Contexts Are Not Global Services

Aunque el Container pueda resolver:

```text
DatabaseContextInterface
```

su implementación será scoped.

Nunca singleton.

---

# 185. Transaction Callback Scope

Una llamada:

```php
DB::transaction(function () {
});
```

no necesariamente crea un nuevo Container scope.

Crea un:

```text
TransactionContext
```

dentro del scope actual.

---

# 186. Nested Execution Scopes

Sólo deberán utilizarse cuando exista un lifecycle real independiente.

No por cada método Database.

---

# 187. Async/Future Compatibility

El diseño deberá permitir en el futuro:

```text
async driver
promise/future based execution
coroutine connection acquisition
```

sin cambiar los principios de lifecycle.

---

# 188. Context Locality

Para concurrencia:

```text
DatabaseContext
```

deberá asociarse al execution context correcto.

No a:

```text
process global
thread global assumption
```

---

# 189. Container Context Storage

Si el Container implementa context-local storage, Database podrá usarlo a través de contracts de Platform.

No deberá implementar otro sistema incompatible.

---

# 190. FrankenPHP First-Class Support

FrankenPHP será el runtime de referencia inicial.

Por ello Architecture Tests e Integration Tests deberán cubrir:

```text
multiple sequential requests
connection reuse
scope reset
transaction leakage
IdentityMap leakage
tenant leakage
cursor leakage
exception cleanup
```

---

# 191. Runtime Adapter Conformance

RoadRunner y OpenSwoole deberán superar la misma suite conceptual:

```text
DatabaseRuntimeAdapterConformanceSuite
```

para garantizar semántica equivalente.

---

# 192. Container Conformance Requirements

El Container de VoltStack deberá ofrecer a Database:

```text
singleton lifetime
scoped lifetime
transient creation
aliases
decorators or equivalent composition
scope lifecycle
cycle detection
lazy factories
compiled definitions
```

o mecanismos equivalentes.

---

# 193. Scope Destruction Hooks

Database deberá poder registrar cleanup al finalizar un scope.

Preferencia:

```text
Lifecycle Manager
```

sobre destructores PHP como mecanismo principal.

---

# 194. Destructor Policy

`__destruct()` puede utilizarse como defensa adicional para recursos.

No como garantía primaria de:

```text
rollback
cursor cleanup
connection release
```

---

# 195. Bootstrap Performance

El bootstrap deberá minimizar:

```text
filesystem scanning
reflection
database connections
metadata parsing
extension discovery
dynamic class inspection
```

en producción.

---

# 196. Bootstrap Compilation

Flujo recomendado de deployment:

```text
Application Source
      │
      ▼
Discover Packages
      │
      ▼
Compile Configuration
      │
      ▼
Compile Metadata
      │
      ▼
Compile Container
      │
      ▼
Generate Registries
      │
      ▼
Production Artifact
```

---

# 197. Worker Startup

Entonces:

```text
Production Artifact
      │
      ▼
Load Compiled Container
      │
      ▼
Load Database Registries
      │
      ▼
Ready
```

sin reconstruir toda la arquitectura.

---

# 198. Cold Start vs Warm Runtime

VoltStack deberá optimizar ambos:

```text
cold start
```

y:

```text
persistent worker throughput
```

sin sacrificar correctness.

---

# 199. Bootstrap Security

La fase bootstrap deberá evitar:

```text
secret logging
arbitrary extension execution from untrusted sources
unsafe generated cache permissions
silent service replacement
```

---

# 200. Extension Trust Boundary

Instalar un paquete PHP implica código ejecutable.

Aun así, VoltStack deberá limitar qué internals son considerados API soportada para extensiones.

---

# 201. Service Definition Validation

Antes del freeze deberá comprobarse:

```text
missing dependencies
invalid lifetime dependency
circular dependencies
duplicate IDs
unknown contracts
invalid decorators
```

---

# 202. Lifetime Validation

Especialmente:

```text
Singleton → Scoped
```

deberá detectarse automáticamente cuando el Container pueda analizar el grafo.

---

# 203. Scope-Aware Factory Validation

Si un singleton utiliza una factory para obtener servicios scoped, deberá declararse explícitamente como:

```text
scope-aware
```

para que architecture tooling pueda auditarlo.

---

# 204. Container Freeze

Después de compilation/bootstrap:

```text
Service Definitions
       │
       ▼
FREEZE
```

No deberán registrarse servicios arbitrariamente durante requests.

---

# 205. Development Dynamic Registration

Podrá permitirse en development tooling bajo condiciones controladas.

No será semántica estándar de producción.

---

# 206. Hot Reload

Si development hot reload modifica servicios Database:

```text
terminate safe scope
rebuild graph
dispose affected persistent resources
restart/recycle worker if required
```

No parchear instancias stateful vivas.

---

# 207. Service Graph Version

El grafo compilado podrá tener:

```text
DatabaseServiceGraphVersion
```

para invalidación de caches y diagnostics.

---

# 208. Architecture Fingerprint

Podrá generarse un fingerprint de:

```text
configuration
extensions
driver registry
type registry
compiler registry
service graph
```

para comparar workers.

Sin incluir secrets en claro.

---

# 209. Worker Consistency

Workers de la misma deployment deberían utilizar el mismo:

```text
architecture fingerprint
```

salvo casos explícitos de rollout progresivo.

---

# 210. Bootstrap Observability

Podrán emitirse eventos:

```text
DatabaseBootstrapStarted
DatabaseBootstrapCompleted
DatabaseBootstrapFailed
DatabaseExtensionRegistered
DatabaseServiceGraphCompiled
```

pero Database no deberá depender de Telemetry para poder bootstrappear.

---

# 211. Bootstrap Telemetry Recursion

El adapter de Telemetry deberá estar disponible sólo en una fase donde no provoque ciclos.

Early bootstrap diagnostics podrán usar mecanismos mínimos de Platform.

---

# 212. Service Resolution Telemetry

No se recomienda instrumentar cada:

```text
container resolve
```

desde Database.

Eso pertenece al Container/Telemetry system.

---

# 213. Database Boot Event

Un evento de alto nivel podrá indicar:

```text
DatabaseReady
```

después de:

```text
configuration validated
registries frozen
service graph ready
```

No significa que exista una conexión abierta.

---

# 214. Database Ready Semantics

```text
DatabaseReady
```

significa:

> El subsistema está correctamente compuesto y puede crear/adquirir recursos cuando sean solicitados.

No:

> El servidor Database fue contactado exitosamente.

---

# 215. Online Readiness

Una verificación distinta podrá representar:

```text
DatabaseConnectivityReady
```

mediante Health System.

---

# 216. Bootstrap and Testing

Tests unitarios deberán poder construir módulos aislados.

Ejemplo:

```text
Query Engine
```

sin bootstrappear todo VoltStack.

---

# 217. Database Test Container

Testing podrá proporcionar un composition helper:

```text
DatabaseTestContainer
```

o fixture equivalente.

No deberá ser requisito de arquitectura productiva.

---

# 218. Minimal Query Engine Test Graph

```text
Test Dialect
   │
   ▼
Compiler
   │
   ▼
Fake Executor
```

sin:

```text
ORM
Telemetry
Multitenancy
HTTP
```

---

# 219. Integration Test Graph

Para tests reales:

```text
Real Driver
Real Connection
Real Database
```

podrán componerse con el mismo bootstrap modular.

---

# 220. Runtime Leak Test

Prueba obligatoria conceptual:

```text
Scope A
├── load User #1
├── begin transaction
└── terminate

Scope B
├── IdentityMap empty
├── no active transaction
├── no tenant A
└── clean connection state
```

---

# 221. Container Architecture Tests

Deberán verificar:

```text
no ORM dependency from Driver
no scoped dependency captured by singleton
no Container injection in forbidden namespaces
no mutable request state in persistent services
no optional package hard dependency
```

---

# 222. Bootstrap Module Tests

Cada módulo deberá poder comprobar:

```text
registered services
lifetimes
required dependencies
extension hooks
freeze behavior
```

---

# 223. Reference Lifetime Table

| Componente | Lifetime recomendado |
|---|---|
| CompiledDatabaseConfiguration | Singleton |
| DriverRegistry | Singleton |
| DialectRegistry | Singleton |
| PlatformRegistry | Singleton |
| TypeRegistry | Singleton |
| CompilerRegistry | Singleton |
| OptimizationRuleRegistry | Singleton |
| MetadataRegistry | Singleton |
| ConnectionFactory | Singleton |
| ConnectionManager | Singleton/Coordinator |
| ConnectionScope | Scoped |
| DatabaseContext | Scoped |
| EntityManager | Scoped |
| UnitOfWork | Scoped |
| IdentityMap | Scoped |
| PersistenceContext | Scoped |
| TransactionContext | Scoped |
| Repository con EntityManager | Scoped |
| QueryBuilder | Transient |
| SchemaBuilder | Transient |
| QueryContext | Operation |
| ExecutionContext | Operation |
| Cursor | Resource |
| Physical Connection | Resource/Pool-managed |
| Transaction | Resource/Scoped operation |

---

# 224. Reference Dependency Model

```text
                     PLATFORM
                        │
                        ▼
              CONFIG / CONTAINER
                        │
                        ▼
                 DATABASE BOOT
                        │
       ┌────────────────┼─────────────────┐
       ▼                ▼                 ▼
    Driver           Platform           Types
       │                │                 │
       └──────────┬─────┴─────────────────┘
                  ▼
              Connection
                  │
                  ▼
              Execution
                  ▲
                  │
       Query Engine / Compiler
                  │
                  ▼
          Schema / Transaction
                  │
                  ▼
                 ORM
                  │
                  ▼
         Optional Integrations
```

---

# 225. Persistent Runtime Model

```text
┌──────────────────────────────────────────────┐
│ Worker                                       │
│                                              │
│  Compiled Configuration                     │
│  Registries                                  │
│  Metadata                                    │
│  Compiler Services                           │
│  Connection Pool                             │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │ Request A                              │  │
│  │ DatabaseContext A                      │  │
│  │ EntityManager A                        │  │
│  │ UnitOfWork A                           │  │
│  │ IdentityMap A                          │  │
│  └────────────────────────────────────────┘  │
│                     │                        │
│                   RESET                      │
│                     │                        │
│  ┌────────────────────────────────────────┐  │
│  │ Request B                              │  │
│  │ DatabaseContext B                      │  │
│  │ EntityManager B                        │  │
│  │ UnitOfWork B                           │  │
│  │ IdentityMap B                          │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

---

# 226. Architectural Invariants

### DB-BOOT-001

Database Core no utilizará el Container como Service Locator.

### DB-BOOT-002

El Composition Root será responsable de conectar contratos e implementaciones.

### DB-BOOT-003

Bootstrap no abrirá conexiones por defecto.

### DB-BOOT-004

EntityManager será scoped.

### DB-BOOT-005

UnitOfWork será scoped.

### DB-BOOT-006

IdentityMap será scoped.

### DB-BOOT-007

TransactionContext será scoped.

### DB-BOOT-008

Los registries estructurales serán persistentes e inmutables después del freeze.

### DB-BOOT-009

Un singleton no capturará directamente servicios scoped.

### DB-BOOT-010

Los Value Objects no serán servicios del Container.

### DB-BOOT-011

Las integraciones opcionales se resolverán durante bootstrap.

### DB-BOOT-012

El ORM podrá omitirse completamente cuando esté deshabilitado.

### DB-BOOT-013

La resolución de conexiones será lazy por defecto.

### DB-BOOT-014

Los runtimes persistentes deberán crear un DatabaseContext nuevo por Execution Scope.

### DB-BOOT-015

El final del scope nunca hará commit implícito de una transacción huérfana.

### DB-BOOT-016

El final del scope nunca hará flush implícito de un UnitOfWork sucio.

### DB-BOOT-017

Una conexión no regresará al pool si su estado limpio no puede garantizarse.

### DB-BOOT-018

Las extensiones se registrarán antes del freeze.

### DB-BOOT-019

Los registros duplicados no se resolverán mediante "last registration wins" silencioso.

### DB-BOOT-020

Driver, Dialect, Platform y Type registries permanecerán separados.

### DB-BOOT-021

La lógica específica de FrankenPHP permanecerá en un Runtime Adapter.

### DB-BOOT-022

RoadRunner y OpenSwoole reutilizarán el mismo lifecycle Database.

### DB-BOOT-023

Los servicios persistentes no conservarán estado específico del request.

### DB-BOOT-024

El Container deberá validar dependencias incompatibles entre lifetimes cuando sea posible.

### DB-BOOT-025

El grafo de servicios deberá ser determinista para una configuración y conjunto de extensiones determinados.

---

# 227. Anti-Patterns prohibidos

## 227.1 Global EntityManager

```php
EntityManager::instance();
```

si devuelve una instancia global persistente.

**Prohibido.**

---

## 227.2 Container dentro del Query Engine

```php
$this->container->get(QueryCompiler::class);
```

**Prohibido en runtime ordinario.**

---

## 227.3 Connection durante Service Provider boot

```php
$pdo = new PDO(...);
```

**Prohibido por defecto.**

---

## 227.4 Mutable Singleton Context

```php
$this->currentTenant = $tenant;
```

en un servicio singleton.

**Prohibido.**

---

## 227.5 Static Transaction

```php
TransactionManager::$current;
```

**Prohibido.**

---

## 227.6 Static Identity Map

```php
IdentityMap::$entities;
```

**Prohibido.**

---

## 227.7 Runtime extension discovery

```text
scan Composer packages on every request
```

**Prohibido en producción.**

---

## 227.8 Hidden service replacement

```text
last registration wins
```

**Prohibido.**

---

## 227.9 Circular integration

```text
Database
  ↓
Cache
  ↓
Database
```

sin bootstrap-safe boundary.

**Prohibido.**

---

## 227.10 Worker-global tenant state

```text
TenantContext singleton
```

con tenant mutable.

**Prohibido.**

---

# 228. Ejemplo de composición conceptual

```php
$services->singleton(
    DriverRegistry::class,
    fn () => $driverRegistry
);

$services->singleton(
    QueryCompilerInterface::class,
    DefaultQueryCompiler::class
);

$services->scoped(
    DatabaseContextInterface::class,
    DatabaseContext::class
);

$services->scoped(
    IdentityMapInterface::class,
    IdentityMap::class
);

$services->scoped(
    UnitOfWorkInterface::class,
    UnitOfWork::class
);

$services->scoped(
    EntityManagerInterface::class,
    EntityManager::class
);
```

Este código es conceptual.

La API real dependerá del Container de VoltStack.

---

# 229. Ejemplo de error arquitectónico

Supongamos:

```text
QueryCompiler
lifetime: singleton
```

y se registra:

```php
new QueryCompiler(
    $container->get(EntityManagerInterface::class)
);
```

Esto crea:

```text
Singleton QueryCompiler
        │
        ▼
Scoped EntityManager
```

La instancia del primer request podría sobrevivir.

Resultado potencial:

```text
Request A EntityManager
        │
        ▼
captured by QueryCompiler
        │
        ▼
Request B accesses A state
```

Este tipo de error deberá ser detectado por Container/Architecture Tests.

---

# 230. Modelo correcto

```text
QueryCompiler
    │
    ├── Dialect
    ├── Platform
    └── Compiler Registry
```

Todos:

```text
persistent/stateless
```

Mientras:

```text
EntityManager
    │
    ├── UnitOfWork
    ├── IdentityMap
    └── PersistenceContext
```

son:

```text
scoped
```

Los dos mundos sólo se conectan mediante operaciones estructuradas:

```text
ORM
 │
 ▼
Query Model
 │
 ▼
Query Engine
```

---

# 231. Modelo final de bootstrap

```text
                    APPLICATION START
                           │
                           ▼
                  VOLTSTACK PLATFORM
                           │
                           ▼
                DATABASE CONFIGURATION
                           │
                           ▼
                 EXTENSION DISCOVERY
                           │
                           ▼
                DATABASE COMPOSITION
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
          Drivers       Query Core       ORM
             │             │             │
             └─────────────┼─────────────┘
                           ▼
                    Integrations
                           │
                           ▼
                     FREEZE GRAPH
                           │
                           ▼
                  PERSISTENT SERVICES
                           │
                           ▼
                  ┌────────────────┐
                  │ EXECUTION SCOPE│
                  └───────┬────────┘
                          ▼
                   DatabaseContext
                          │
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
      ConnectionScope   UoW       TransactionContext
                          │
                          ▼
                    IdentityMap
                          │
                          ▼
                   Application Work
                          │
                          ▼
                     Scope End
                          │
                          ▼
                  CLEANUP / RESET
```

---

# 232. Criterio de éxito

La integración con Bootstrap y Container será considerada correcta cuando sea posible ejecutar indefinidamente:

```text
Request A
RESET
Request B
RESET
Request C
RESET
...
Request N
```

en un mismo worker sin que exista transferencia accidental de:

```text
entities
transactions
tenant context
connection session state
query state
cursors
UnitOfWork state
```

entre scopes.

---

# 233. Resultado esperado

El sistema permitirá combinar:

```text
Laravel-like Developer Experience
        +
Dependency Injection
        +
Compiled Service Graph
        +
Doctrine-like Component Boundaries
        +
Persistent Runtime Safety
        +
Optional Quantum Integrations
```

sin hacer que el Container se convierta en el centro lógico de Database.

El Container será:

> el constructor de la arquitectura.

No:

> la arquitectura misma.

---

# 234. Principio final

La regla maestra de Bootstrap será:

> Construir una vez lo que puede compartirse; crear por scope todo lo que contiene estado; adquirir bajo demanda todo lo que representa un recurso externo.

En forma resumida:

```text
Immutable / Stateless
        │
        ▼
Persistent

Request State
        │
        ▼
Scoped

Operation State
        │
        ▼
Transient

External Resource
        │
        ▼
Acquire → Use → Cleanup → Release
```

---

# 235. Conclusión

`VoltStack/Quantum/Database` utilizará el Service Container para realizar composición explícita, validada y optimizable.

La arquitectura evitará uno de los errores más peligrosos en frameworks PHP modernos con servidores persistentes:

```text
request-scoped state
        │
        ▼
captured by persistent service
        │
        ▼
cross-request state leakage
```

La separación entre:

```text
Persistent Services
Scoped State
Transient Operations
External Resources
```

será una propiedad fundamental del Database System desde su primera implementación.

Esto permitirá que FrankenPHP sea soportado como runtime principal sin diseñar primero una arquitectura request-per-process y tratar de corregirla posteriormente.

---

# 236. Siguiente documento

El siguiente documento es:

```text
08_DATABASE_LIFECYCLE_AND_RUNTIME_MODEL.md
```

Deberá profundizar específicamente en:

```text
Database lifecycle
ExecutionScope
DatabaseContext
request lifecycle
worker lifecycle
connection lifecycle
EntityManager lifecycle
UnitOfWork lifecycle
IdentityMap lifecycle
transaction lifecycle
cursor lifecycle
cleanup orchestration
reset guarantees
failure during cleanup
persistent worker isolation
concurrent execution
context propagation
FrankenPHP lifecycle
RoadRunner lifecycle
OpenSwoole lifecycle
CLI lifecycle
queue worker lifecycle
long-running processes
resource ownership
state leak detection
worker recycle policies
```

y establecer formalmente la transición:

```text
Worker
  │
  ├── Scope A
  │      │
  │      ▼
  │    Cleanup
  │
  ├── Scope B
  │      │
  │      ▼
  │    Cleanup
  │
  └── Scope N
```

como uno de los invariantes centrales de toda la arquitectura Database.