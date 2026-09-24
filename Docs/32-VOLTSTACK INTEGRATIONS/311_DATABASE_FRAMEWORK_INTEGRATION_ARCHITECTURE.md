# 311_DATABASE_FRAMEWORK_INTEGRATION_ARCHITECTURE.md

# VoltStack Quantum Database
## Database Framework Integration Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 311 — Database Framework Integration Architecture  
**Bloque:** 32 — VoltStack Integration  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `310_DATABASE_CODE_GENERATION_SYSTEM.md`  
**Siguiente documento:** `312_DATABASE_CONTAINER_INTEGRATION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura maestra mediante la cual `VoltStack/Quantum/Database` se integra con el resto de VoltStack.

Database no es un sistema aislado. Durante la ejecución puede colaborar con:

- Service Container;
- Configuration;
- Cache;
- Event System;
- Telemetry;
- Validation;
- Authentication;
- Authorization;
- Jobs y Queues;
- HTTP Kernel;
- Runtime Manager;
- CLI;
- Testing;
- Security;
- Multitenancy;
- SaaS;
- extensiones oficiales y de terceros.

Sin embargo, estas integraciones no deberán convertir Database en un subsistema fuertemente acoplado al framework.

La regla central será:

> **VoltStack Database deberá integrarse profundamente con VoltStack mediante contratos, adapters, bridges y composition roots explícitos, manteniendo el núcleo Database independiente de los subsistemas de alto nivel que lo consumen.**

Formalmente:

```text
Framework Integration
=
Stable Contracts
+ Explicit Adapters
+ Controlled Bootstrap
+ Scoped Runtime Context
+ Lifecycle Coordination
```

y nunca:

```text
Framework Integration
=
Database knows everything about VoltStack
```

---

# 2. Regla fundamental de dependencias

La dirección general deberá ser:

```text
Application
    ↓
Framework Integration
    ↓
Quantum/Database
    ↓
Platform Contracts
```

y no:

```text
Quantum/Database
    ↓
HTTP Controllers
    ↓
Authentication
    ↓
Application
```

Database podrá utilizar contratos de infraestructura suficientemente generales.

No deberá depender de implementaciones de alto nivel.

---

# 3. Database Core ≠ Framework Glue

Se establece una separación formal:

```text
Database Core
≠
Database Framework Integration
```

El Core contiene:

```text
Driver
Connection
Query
Schema
Migration
ORM
Hydration
Persistence
Transaction
Cache abstractions
Events abstractions
Telemetry abstractions
Security primitives
Testing primitives
```

La capa de integración contiene:

```text
Container bindings
Config adapters
Cache adapters
Event bridges
Telemetry bridges
Validation bridges
Auth integration
Authorization integration
Queue integration
HTTP lifecycle integration
Runtime hooks
```

---

# 4. Objetivos

La arquitectura deberá:

1. mantener Database modular;
2. impedir dependencias circulares;
3. permitir uso standalone cuando sea razonable;
4. proporcionar integración zero/low configuration dentro de VoltStack;
5. soportar componentes opcionales;
6. preservar request/operation scope;
7. soportar runtimes persistentes;
8. permitir reemplazo de adapters;
9. evitar estado global;
10. centralizar bootstrap;
11. centralizar lifecycle;
12. integrar telemetry;
13. integrar cache;
14. integrar eventos;
15. integrar seguridad;
16. soportar jobs;
17. soportar HTTP;
18. soportar CLI;
19. soportar testing;
20. soportar extensiones.

---

# 5. Principio de integración

La arquitectura deberá seguir:

```text
Core defines need
       ↓
Contract defines boundary
       ↓
Framework provides adapter
       ↓
Composition Root connects both
```

Ejemplo:

```text
Database
needs cache
   ↓
CacheProvider contract
   ↓
VoltStack Cache Adapter
   ↓
Quantum/Cache
```

No:

```text
Database
→ instantiate VoltStackCache directly
```

---

# 6. Arquitectura global

```text
                         Application
                             │
                             ▼
                      VoltStack Framework
                             │
        ┌────────────────────┼─────────────────────┐
        │                    │                     │
        ▼                    ▼                     ▼
      HTTP                 Console              Workers
        │                    │                     │
        └────────────────────┼─────────────────────┘
                             ▼
                     Runtime / Lifecycle
                             │
                             ▼
                 Framework Integration Layer
                             │
        ┌──────────────┬─────┼─────┬───────────────┐
        ▼              ▼     ▼     ▼               ▼
    Container        Config Cache Events       Telemetry
        │              │     │     │               │
        └──────────────┴─────┼─────┴───────────────┘
                             ▼
                    Quantum/Database
                             │
        ┌────────────────────┼─────────────────────┐
        ▼                    ▼                     ▼
   Query / ORM          Transaction          Connection
                                                  │
                                                  ▼
                                               Driver
                                                  │
                                                  ▼
                                                 DBMS
```

---

# 7. Integration Layer

Se propone:

```text
VoltStack/Quantum/Database/Integration
```

como espacio conceptual para adapters específicos.

Ejemplo:

```text
Integration/
├── Container/
├── Config/
├── Cache/
├── Events/
├── Telemetry/
├── Validation/
├── Authentication/
├── Authorization/
├── Queue/
├── Http/
├── Console/
├── Runtime/
└── Testing/
```

---

# 8. Core contracts

Cuando Database necesite una capacidad externa deberá depender de una abstracción.

Ejemplo:

```php
interface DatabaseCacheProvider
{
    public function get(CacheKey $key): CacheLookupResult;

    public function put(
        CacheKey $key,
        mixed $value,
        CachePolicy $policy,
    ): void;
}
```

El adapter VoltStack implementará el contrato utilizando el Cache System oficial.

---

# 9. Contrato específico vs contrato genérico

No toda integración necesita crear:

```text
DatabaseSomethingInterface
```

Si `Platform` ya proporciona un contrato estable y suficientemente neutral, Database deberá reutilizarlo.

Ejemplo conceptual:

```text
Platform\Clock\Clock
Platform\Event\EventDispatcher
Platform\Telemetry\Tracer
```

---

# 10. Regla de ubicación

Un contrato deberá vivir en `Platform` cuando:

- sea útil para múltiples subsistemas;
- no dependa semánticamente de Database;
- represente infraestructura general.

Deberá vivir en Database cuando:

- exprese semántica específicamente Database;
- utilice conceptos Database;
- no sea una abstracción general del framework.

---

# 11. Ejemplo

General:

```text
Clock
Logger
EventDispatcher
Tracer
```

Database-specific:

```text
QueryCachePolicy
DatabaseAuditSink
TenantDatabaseResolver
ConnectionCredentialProvider
```

---

# 12. Composition Root

La integración se ensamblará desde un composition root.

Conceptualmente:

```text
DatabaseServiceProvider
```

o mecanismo equivalente.

Responsabilidades:

```text
register contracts
register factories
register adapters
load configuration
register lifecycle hooks
register extensions
freeze registries
validate configuration
```

---

# 13. Composition Root ≠ Runtime State

El service provider no deberá almacenar:

```text
current EntityManager
current Transaction
current tenant
current query
current connection
```

---

# 14. Bootstrap phases

Se propone:

```text
DISCOVERY
    ↓
REGISTRATION
    ↓
CONFIGURATION
    ↓
EXTENSION REGISTRATION
    ↓
VALIDATION
    ↓
COMPILATION
    ↓
FREEZE
    ↓
READY
```

---

# 15. Discovery

Detecta:

```text
installed database drivers
optional integrations
plugins
framework capabilities
```

sin ejecutar lógica operacional innecesaria.

---

# 16. Registration

Registra:

```text
contracts
factories
services
adapters
```

---

# 17. Configuration

Construye configuración normalizada.

No deberá conservar arrays arbitrarios como representación final del sistema.

---

# 18. Extension registration

Plugins podrán registrar:

```text
drivers
dialects
types
compilers
query extensions
ORM extensions
telemetry adapters
```

---

# 19. Validation

Antes de declarar Database READY:

```text
configuration valid?
required driver available?
required dialect available?
extension conflicts?
dependency graph valid?
```

---

# 20. Compilation

Cuando sea aplicable:

```text
ORM metadata
mapping metadata
container definitions
configuration
extension registries
```

podrán precompilarse.

---

# 21. Freeze

Después del bootstrap:

```text
DriverRegistry
TypeRegistry
DialectRegistry
CompilerRegistry
ExtensionRegistry
MetadataRegistry
```

deberán quedar inmutables cuando su diseño lo requiera.

---

# 22. Runtime mutable state

Lo mutable deberá vivir en scopes operacionales.

Ejemplo:

```text
RequestScope
OperationScope
JobScope
CommandScope
CoroutineScope
```

---

# 23. Shared vs scoped

Clasificación:

```text
Immutable / Shared
├── Metadata
├── Registries
├── Configuration
├── Platform definitions
├── Compiler definitions
└── Factories

Scoped / Mutable
├── DatabaseContext
├── EntityManager
├── UnitOfWork
├── IdentityMap
├── TransactionContext
├── QueryContext
├── TenantContext
└── connection leases
```

---

# 24. Regla de oro

> **Compartir definiciones inmutables entre requests es deseable; compartir estado mutable de ejecución entre requests es un error arquitectónico.**

---

# 25. Container Integration

El Service Container deberá resolver servicios Database.

Ejemplo:

```php
$entityManager = container()->get(EntityManager::class);
```

Pero el binding deberá respetar el scope.

---

# 26. Singleton danger

Incorrecto:

```text
EntityManager
→ process singleton
```

en FrankenPHP/RoadRunner/OpenSwoole.

Correcto:

```text
EntityManager
→ operation/request scoped
```

---

# 27. Singleton-safe services

Ejemplos potenciales:

```text
TypeRegistry
DialectRegistry
CompiledMetadataRegistry
QueryNormalizer
CompilerFactory
```

si son realmente inmutables/stateless.

---

# 28. Config Integration

Database deberá consumir configuración normalizada proveniente del Config System.

Ejemplo:

```text
config/database.php
        ↓
Config System
        ↓
DatabaseConfigurationResolver
        ↓
DatabaseConfiguration
```

---

# 29. Config ≠ runtime state

La configuración describe:

```text
connections
drivers
pooling
timeouts
cache policies
telemetry policies
```

No:

```text
current transaction
current connection lease
current tenant
```

---

# 30. Environment variables

El Config System podrá resolver:

```text
DB_HOST
DB_PORT
DB_DATABASE
```

pero Database no deberá implementar su propio `.env` parser.

---

# 31. Secrets

Configuración podrá contener:

```text
SecretReference
```

en lugar del secret materializado.

Flujo:

```text
Database Config
     ↓
Secret Reference
     ↓
Credential Provider
     ↓
Connection Factory
```

---

# 32. Cache Integration

Database podrá integrarse con el Cache System oficial para:

```text
Query Cache
Result Cache
Metadata Cache
Entity Cache
Compiled Query Cache
```

según las reglas de los documentos 186–192.

---

# 33. Database cache categories

Debe preservarse:

```text
Query Cache
≠
Result Cache
≠
Metadata Cache
≠
Entity Cache
≠
IdentityMap
≠
Hydration Cache
```

---

# 34. Cache bridge

```text
Database Cache Contract
        ↓
VoltStack Cache Bridge
        ↓
Quantum/Cache
```

---

# 35. Cache optionality

Si Cache no está instalado/configurado:

```text
Database
→ remains functional
```

cuando la característica sea opcional.

---

# 36. No-cache adapter

Podrá existir:

```text
NullDatabaseCache
```

si semánticamente apropiado.

Pero:

```text
Null Cache
≠
fake successful cache
```

---

# 37. Cache lifecycle

Invalidaciones relacionadas con transacciones deberán respetar:

```text
beforeCommit
afterCommit
rollback
unknown outcome
```

---

# 38. Event System Integration

Database podrá publicar eventos al Event System de VoltStack.

Ejemplos:

```text
QueryExecuting
QueryExecuted
TransactionStarted
TransactionCommitted
EntityPersisted
ConnectionOpened
```

---

# 39. Internal events vs framework events

Debe distinguirse:

```text
Database Internal Event
≠
Framework/Public Event
```

No todos los eventos internos deberán exponerse.

---

# 40. Event bridge

```text
Database Event
      ↓
Database Event Dispatcher
      ↓
Framework Event Bridge
      ↓
VoltStack Event System
```

---

# 41. Event semantics

La integración no podrá cambiar el resultado real de Database.

Ejemplo:

```text
COMMIT succeeded
→ afterCommit listener fails
```

El resultado sigue siendo:

```text
COMMITTED
```

aunque la integración de evento pueda fallar.

---

# 42. Domain Events

Los eventos de dominio de la aplicación son diferentes de eventos técnicos Database.

```text
OrderPlaced
≠
EntityInserted
```

---

# 43. Telemetry Integration

Database deberá integrarse profundamente con el sistema Telemetry de VoltStack.

Podrá producir:

```text
traces
spans
metrics
structured logs
profiling information
diagnostics
```

---

# 44. Telemetry bridge

```text
Database Telemetry
       ↓
Telemetry Contracts
       ↓
VoltStack Telemetry
       ↓
OpenTelemetry / Prometheus / etc.
```

---

# 45. Telemetry optionality

La ausencia de exporter externo no deberá impedir Database.

---

# 46. Telemetry ≠ semantics

Debe mantenerse:

```text
Telemetry
≠
Query Execution
```

y:

```text
Telemetry failure
≠
Database failure
```

salvo una política explícita excepcional.

---

# 47. Trace context

Database podrá recibir el contexto de tracing actual.

Ejemplo:

```text
HTTP Request Span
      ↓
Controller Span
      ↓
Database Query Span
```

---

# 48. Database does not own request trace

Database crea spans hijos.

No deberá asumir que controla el trace completo.

---

# 49. Cardinality

No deberá publicar como labels métricas de cardinalidad descontrolada:

```text
raw SQL
user id
tenant id
full URL
parameter values
```

---

# 50. Validation Integration

VoltStack Validation y Database podrán colaborar.

Pero:

```text
Validation
≠
Database Constraint
```

---

# 51. Application validation

Ejemplo:

```text
email must be valid
```

pertenece a Validation.

---

# 52. Database constraint

Ejemplo:

```text
UNIQUE(users.email)
```

pertenece al Database/Schema.

---

# 53. Unique validation

Una regla:

```text
unique:users,email
```

podrá utilizar Database para consulta.

Pero:

```text
validation query
≠
concurrency guarantee
```

---

# 54. Race condition

Esto:

```text
check email doesn't exist
↓
insert
```

puede competir con otro request.

La garantía final deberá provenir del constraint Database.

---

# 55. Exists validation

Podrá existir integración:

```text
exists
unique
database-backed custom rules
```

utilizando Query API pública.

---

# 56. Validation must not bypass Database

Incorrecto:

```text
Validation
→ raw PDO
```

Correcto:

```text
Validation
→ Database Public API
```

---

# 57. Authentication Integration

Authentication podrá usar Database para:

```text
user lookup
credential metadata
session persistence
token persistence
device records
MFA records
```

---

# 58. Dependency direction

Correcto:

```text
Authentication
→ Database
```

No:

```text
Database
→ Authentication User Provider
```

---

# 59. Authentication context

Database podrá recibir identidad autenticada únicamente a través de una abstracción cuando una feature Database realmente la requiera.

Ejemplo:

```text
audit actor
```

---

# 60. Actor Provider

Podrá definirse:

```php
interface DatabaseActorProvider
{
    public function currentActor(): ?DatabaseActor;
}
```

y Authentication proporcionar el bridge.

---

# 61. DatabaseActor

Debe ser una representación mínima.

Ejemplo:

```text
type
identifier
source
```

No un objeto `User` completo.

---

# 62. Authentication optionality

Database deberá funcionar sin Authentication.

Esto es esencial para:

```text
CLI
migrations
background jobs
standalone usage
testing
```

---

# 63. Authorization Integration

Authorization puede controlar:

```text
who may execute administrative DB operations
who may access certain entities
who may perform sensitive queries
```

según la capa.

---

# 64. Direction

Normalmente:

```text
Controller/Application
→ Authorization
→ Database
```

---

# 65. Database Data Access Security

Algunas reglas podrán aplicarse dentro de Database:

```text
tenant isolation
mandatory scopes
row access policy integration
sensitive field policy
```

pero deberán utilizar contratos neutrales.

---

# 66. Authorization ≠ Query Builder

Query Builder no deberá conocer:

```text
Gate
Policy
Voter
```

---

# 67. Security scope injection

En su lugar:

```text
Authorization
      ↓
Data Access Policy Adapter
      ↓
Database Query Context
```

---

# 68. Mandatory policy

Cuando una política sea obligatoria, deberá formar parte de la semántica del Query Context y no ser un simple listener opcional.

---

# 69. Jobs / Queue Integration

Jobs podrán utilizar Database.

Ejemplo:

```php
final class ProcessInvoice
{
    public function handle(EntityManager $em): void
    {
        // ...
    }
}
```

---

# 70. Job scope

Cada job deberá tener su propio:

```text
DatabaseContext
EntityManager
UnitOfWork
IdentityMap
TransactionContext
```

---

# 71. Worker reuse

En un queue worker:

```text
Job A
↓
reset
↓
Job B
```

Nunca:

```text
Job A EntityManager
→ reused as mutable state in Job B
```

---

# 72. Queue retry ≠ Transaction retry

Regla crítica:

```text
Queue Retry
≠
Database Transaction Retry
```

---

# 73. Transaction retry

Repite una unidad transaccional segura.

---

# 74. Job retry

Puede repetir una operación empresarial completa.

Esto puede producir efectos externos duplicados si no existe idempotencia.

---

# 75. Outbox integration

Para coordinación entre:

```text
Database commit
+
message publication
```

deberá utilizarse un patrón explícito como Transactional Outbox cuando corresponda.

---

# 76. Commit ≠ message delivered

Nunca asumir:

```text
DB commit
→ queue message guaranteed
```

sin protocolo adicional.

---

# 77. HTTP Integration

Database deberá integrarse con el lifecycle HTTP de VoltStack.

Flujo:

```text
HTTP Request
      ↓
Request Scope Begin
      ↓
Database Context Begin
      ↓
Controller / Components
      ↓
Database Operations
      ↓
Response
      ↓
Database Scope Finalization
      ↓
Reset
      ↓
Request Scope End
```

---

# 78. Request scope

En HTTP:

```text
Request
≈
default Database operation scope
```

pero no siempre.

---

# 79. Request ≠ Transaction

Regla:

```text
HTTP Request
≠
Database Transaction
```

No deberá iniciarse una transacción automáticamente para cada request salvo política explícita.

---

# 80. Request ≠ EntityManager lifetime universal

HTTP es un adapter.

Jobs y CLI tendrán otros scopes.

Por ello la abstracción general será:

```text
OperationScope
```

---

# 81. Operation scope

Ejemplos:

```text
HTTP Request
Queue Job
CLI Command
Scheduled Task
RPC Call
WebSocket Operation
SPA Action
```

---

# 82. SPA integration

VoltStack tendrá runtime SPA reactivo nativo.

Cada interacción SPA deberá recibir un scope Database claramente delimitado.

---

# 83. SPA action

Ejemplo:

```text
Browser
  ↓
VoltStack SPA Protocol
  ↓
Component Action
  ↓
Operation Scope
  ↓
DatabaseContext
```

---

# 84. Hydration ≠ frontend hydration

Debe evitarse confusión terminológica:

```text
Database Entity Hydration
≠
VoltStack Frontend Component Hydration
```

---

# 85. Reactive component state

No deberá contener automáticamente entidades ORM managed entre requests.

---

# 86. Critical rule

En runtimes persistentes:

```text
Serialized Component State
≠
Managed Entity State
```

---

# 87. Entity references in SPA

Preferible conservar:

```text
Entity identifier
DTO
projection
serialized value
```

según el caso, y re-resolver el contexto Database en una nueva operación.

---

# 88. HTTP error integration

Database exceptions deberán pasar por:

```text
Database Error Model
        ↓
Framework Exception Mapping
        ↓
HTTP Error Response
```

---

# 89. Database exceptions ≠ HTTP status codes

Database no deberá retornar:

```text
404
409
500
```

como conceptos propios.

---

# 90. Framework mapping

Ejemplo:

```text
EntityNotFound
→ application decides 404

OptimisticLockConflict
→ application/framework may decide 409

DatabaseUnavailable
→ framework may decide 503
```

según contexto.

---

# 91. CLI Integration

CLI tendrá su propio lifecycle:

```text
Command Start
      ↓
Command Scope
      ↓
Database Context
      ↓
Command
      ↓
Finalize
      ↓
Reset
```

---

# 92. Long-running CLI

Comandos como:

```text
import
export
migration
backup
maintenance
```

pueden durar mucho tiempo.

No deberán asumir semántica request HTTP.

---

# 93. Chunked CLI operations

Podrán crear sub-scopes:

```text
Command
├── Chunk 1 scope
├── Chunk 2 scope
├── Chunk 3 scope
└── ...
```

cuando convenga liberar memoria/IdentityMap.

---

# 94. Testing Integration

Testing deberá poder reemplazar integraciones mediante:

```text
test container
fake adapters
controlled clocks
controlled cache
test telemetry
test event dispatcher
```

sin modificar Database Core.

---

# 95. Test isolation

Cada test deberá poder crear:

```text
Test Operation Scope
```

independiente.

---

# 96. Framework testing ≠ DBMS conformance

Debe mantenerse:

```text
Framework Integration Test
≠
Driver Conformance Test
```

---

# 97. Runtime Integration

VoltStack deberá soportar:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

mediante Runtime adapters.

---

# 98. FrankenPHP

Será el runtime predeterminado.

La integración deberá estar optimizada primero para:

```text
FrankenPHP worker mode
```

sin comprometer arquitectura portable.

---

# 99. Runtime contract

Conceptualmente:

```php
interface DatabaseRuntimeAdapter
{
    public function beginOperation(
        RuntimeOperationContext $context,
    ): DatabaseOperationScope;

    public function endOperation(
        DatabaseOperationScope $scope,
    ): void;
}
```

---

# 100. Runtime adapter responsibility

Podrá coordinar:

```text
scope creation
context binding
resource release
connection return
state reset
leak detection
```

---

# 101. Runtime adapter ≠ Database semantics

No decide:

```text
SQL
ORM mapping
transaction semantics
query planning
```

---

# 102. Persistent Runtime Model

```text
Worker Start
    ↓
Shared immutable services boot
    ↓
┌─────────────────────────────┐
│ Operation A                 │
│ create mutable DB scope     │
│ execute                     │
│ finalize                    │
│ reset                       │
└─────────────────────────────┘
    ↓
┌─────────────────────────────┐
│ Operation B                 │
│ fresh mutable DB scope      │
│ execute                     │
│ finalize                    │
│ reset                       │
└─────────────────────────────┘
```

---

# 103. State leak prevention

Después de una operación no deberán sobrevivir accidentalmente:

```text
Entity instances
IdentityMap
UnitOfWork changes
transaction state
tenant context
query context
lazy-load references
temporary session state
```

---

# 104. Reset failure

Si reset falla:

```text
resource
→ DIRTY / TAINTED
```

y deberá evitarse su reutilización.

---

# 105. Worker quarantine

Si el framework no puede demostrar que el estado global Database está limpio:

```text
Worker
→ quarantine/recycle
```

según Runtime Manager.

---

# 106. Connection reuse

Las conexiones físicas podrán reutilizarse.

Pero:

```text
Physical Connection Reuse
≠
Logical Database State Reuse
```

---

# 107. Connection sanitization

Antes de volver al pool:

```text
rollback active tx
close cursors
clear temporary state
restore isolation
restore role
restore schema/search_path
restore timezone
clear tenant variables
```

según plataforma/capabilities.

---

# 108. Unknown reset

```text
Reset Outcome = UNKNOWN
```

implica:

```text
DO NOT REUSE
```

por defecto.

---

# 109. Multitenancy Integration

Multitenancy será paquete oficial opcional.

Por tanto:

```text
Quantum/Database
does not require
Quantum/Multitenancy
```

---

# 110. Direction

Correcto:

```text
Multitenancy
→ Database Integration Contracts
```

o:

```text
Database Multitenancy Bridge
→ optional package contract
```

sin dependencia obligatoria.

---

# 111. Tenant Context

Cuando Multitenancy esté instalado:

```text
Framework Tenant Resolution
        ↓
Tenant Context
        ↓
Database Tenant Adapter
        ↓
DatabaseContext
```

---

# 112. Tenant resolution timing

Deberá ocurrir antes de:

```text
connection resolution
query planning where tenant affects routing
cache key creation
shard routing
```

---

# 113. Tenant ≠ Authentication User

Regla:

```text
Tenant
≠
Authenticated User
```

aunque puedan estar relacionados.

---

# 114. SaaS Integration

`VoltStack/Quantum/SaaS` también será opcional.

Database no dependerá de SaaS.

---

# 115. SaaS direction

```text
SaaS
→ Multitenancy optional integration
→ Database
```

o directamente:

```text
SaaS
→ Database Public API
```

según la función.

---

# 116. SaaS metadata

Conceptos como:

```text
subscription
plan
billing
organization
quota
```

no pertenecen al Database Core.

---

# 117. Service boundaries

Database proporciona persistencia.

No deberá interpretar semántica SaaS.

---

# 118. Security Integration

VoltStack Security podrá aportar:

```text
secret providers
audit sinks
credential rotation
encryption services
access policies
```

mediante adapters.

---

# 119. Encryption

Database podrá utilizar servicios criptográficos para determinadas funciones.

Pero:

```text
Database
≠
Cryptography Provider
```

---

# 120. Hashing

Asimismo:

```text
Database
≠
Password Hashing System
```

Authentication deberá hash passwords antes de persistirlos.

---

# 121. Audit integration

Database Audit podrá publicar registros hacia el Audit/Security subsystem.

---

# 122. Audit failure policy

Deberá ser configurable según criticidad.

Ejemplo:

```text
best effort telemetry audit
```

no equivale a:

```text
mandatory compliance audit
```

---

# 123. Mandatory audit

Si una operación requiere auditoría obligatoria y no puede registrarse:

```text
operation may need rejection
```

pero esta política deberá estar definida explícitamente.

---

# 124. Filesystem Integration

Migrations, backup, restore, export e import pueden requerir filesystem.

Deberán utilizar contratos de filesystem cuando sea apropiado.

---

# 125. Database ≠ local disk assumption

El storage puede ser:

```text
local
network
object storage
temporary filesystem
```

según adapter.

---

# 126. Backup integration

El Backup System podrá integrarse con Storage.

Pero:

```text
Backup Artifact
≠
ordinary application file
```

por sus requisitos de seguridad, integridad y lifecycle.

---

# 127. Concurrency Integration

Si VoltStack dispone de un sistema general de Concurrency, Database podrá utilizarlo para coordinación externa.

Pero no deberá reemplazar:

```text
DB transactions
DB locks
optimistic locking
pessimistic locking
```

---

# 128. Process integration

Operaciones administrativas podrán utilizar el Process System para:

```text
pg_dump
pg_restore
mysqldump
native tools
```

mediante adapters controlados.

---

# 129. Database ≠ shell executor

Nunca:

```text
Database core
→ shell_exec()
```

arbitrariamente.

---

# 130. Native tool adapter

Flujo:

```text
Backup Planner
    ↓
Native Tool Requirement
    ↓
Process Adapter
    ↓
Controlled Process Execution
```

---

# 131. Dependency classification

Toda dependencia de integración deberá clasificarse.

Propuesta:

```text
REQUIRED_CORE
REQUIRED_RUNTIME
OPTIONAL
CONDITIONAL
DEVELOPMENT_ONLY
TEST_ONLY
```

---

# 132. REQUIRED_CORE

Sólo infraestructura indispensable para que Database exista.

Debe mantenerse mínima.

---

# 133. REQUIRED_RUNTIME

Necesaria para determinada configuración runtime.

Ejemplo:

```text
selected database driver
```

---

# 134. OPTIONAL

Ejemplo:

```text
distributed cache provider
telemetry exporter
```

---

# 135. CONDITIONAL

Necesaria sólo si se activa una feature.

Ejemplo:

```text
PostGIS extension adapter
```

---

# 136. DEVELOPMENT_ONLY

Ejemplo:

```text
debug toolbar
code generation tools
profiler UI
```

---

# 137. TEST_ONLY

Ejemplo:

```text
FakeConnection
TestClock
FailureInjector
```

---

# 138. Optional dependency rule

El Core no deberá fallar durante bootstrap simplemente porque una integración opcional no está instalada, salvo que la configuración la solicite.

---

# 139. Feature activation

Formalmente:

```text
Feature Enabled
∧
Required Integration Missing
→
Configuration Error
```

pero:

```text
Feature Disabled
∧
Integration Missing
→
Valid
```

---

# 140. Integration Capability Model

Además de Database Capabilities podrán existir capacidades de integración.

Ejemplo:

```text
CACHE_AVAILABLE
TELEMETRY_AVAILABLE
EVENT_BRIDGE_AVAILABLE
SECRET_PROVIDER_AVAILABLE
```

---

# 141. Integration Capability ≠ Database Capability

Debe mantenerse:

```text
PostgreSQL supports RETURNING
```

como Database Capability.

Mientras:

```text
shared cache provider installed
```

es Integration Capability.

---

# 142. Integration Registry

Podrá existir:

```php
interface DatabaseIntegrationRegistry
{
    public function has(IntegrationId $id): bool;

    public function get(IntegrationId $id): DatabaseIntegration;
}
```

---

# 143. Registry freeze

Tras bootstrap:

```text
IntegrationRegistry
→ FROZEN
```

---

# 144. Dynamic per-request integration

Lo dinámico no será el registro.

Será el contexto.

Ejemplo:

```text
current actor
current tenant
current trace
current request
```

---

# 145. Context providers

La integración podrá usar providers:

```text
ActorProvider
TenantProvider
TraceContextProvider
OperationContextProvider
```

---

# 146. Provider ≠ global mutable holder

Incorrecto:

```php
CurrentTenant::$tenant = $tenant;
```

Correcto:

```text
Scoped Tenant Context
```

---

# 147. Context propagation

En operaciones async/coroutines deberá propagarse explícitamente.

---

# 148. OpenSwoole

No se podrá depender de thread-local assumptions no válidas para coroutines.

---

# 149. Context isolation

Formalmente:

```text
Context(Operation A)
∩
Mutable Context(Operation B)
=
∅
```

para operaciones independientes.

---

# 150. FrankenPHP lifecycle

Flujo conceptual:

```text
Worker Boot
    ↓
Framework Bootstrap
    ↓
Database Shared Services Ready
    ↓
Request A
    ↓
Database Operation Scope A
    ↓
Finalize + Reset
    ↓
Request B
    ↓
Database Operation Scope B
```

---

# 151. Boot once ≠ state once

Regla:

> **El hecho de que un servicio sea construido una sola vez durante el boot del worker no autoriza que conserve estado mutable perteneciente a una operación.**

---

# 152. Lifecycle Coordinator

Podrá existir:

```text
DatabaseLifecycleCoordinator
```

para coordinar:

```text
begin operation
before execution
finalize operation
reset
resource release
leak verification
```

---

# 153. Coordinator ≠ God Object

Deberá delegar a:

```text
Context Manager
EntityManager Lifecycle
Transaction Manager
Connection Manager
Telemetry
Cache
```

---

# 154. Operation lifecycle

Estados propuestos:

```text
CREATED
   ↓
ACTIVE
   ↓
FINALIZING
   ↓
RESETTING
   ↓
CLOSED
```

Errores:

```text
TAINTED
FAILED
UNKNOWN
```

---

# 155. Finalization

Finalization podrá incluir:

```text
detect active transaction
detect pending UoW
close streams
release cursors
return connections
flush telemetry buffers
```

---

# 156. Automatic flush

No deberá asumirse:

```text
end request
→ EntityManager.flush()
```

por defecto.

---

# 157. Why

El framework no debe persistir cambios simplemente porque terminó una operación.

---

# 158. Pending UoW

Si al cerrar scope existen cambios pendientes:

```text
policy
→ ignore/detach with diagnostics
→ warning
→ fail in strict mode
```

según configuración.

Nunca auto-persistencia silenciosa.

---

# 159. Active transaction at scope end

Debe considerarse un problema de lifecycle.

Por defecto:

```text
rollback
+
diagnostic
+
connection sanitization
```

si el outcome es demostrable.

---

# 160. Rollback failure

Si falla:

```text
connection
→ tainted
→ discard
```

---

# 161. Stream/cursor leak

Streams/cursors abiertos deberán cerrarse o invalidar la reutilización segura del recurso.

---

# 162. Lazy loading after scope

Una entidad detached no deberá recuperar mágicamente un EntityManager global.

---

# 163. Lazy-loading rule

```text
scope closed
+
unloaded relation
+
lazy access
→
Detached/LazyLoadUnavailable error
```

según política.

---

# 164. Serialization

Serializar entidades no deberá disparar lazy loading por defecto.

Esto es particularmente importante para:

```text
HTTP JSON
SPA state
queue payloads
cache
logs
```

---

# 165. Queue serialization

Jobs no deberán transportar managed entities como estado ORM activo.

Preferible:

```text
EntityId
Value Object
DTO
Command payload
```

---

# 166. Rehydration

En el job:

```text
new Job Scope
↓
new EntityManager
↓
load entity
```

---

# 167. Transaction boundaries

Los subsistemas superiores podrán declarar transacciones.

Ejemplo:

```php
$transactions->run(function () use ($service) {
    $service->execute();
});
```

---

# 168. Controllers

Un controller no deberá gestionar detalles del driver.

Correcto:

```text
Controller
→ Application Service
→ Repository/EntityManager
```

---

# 169. Components

Un componente SPA tampoco deberá acceder al driver directamente.

---

# 170. Driver visibility

Idealmente:

```text
Driver APIs
```

serán infraestructura de bajo nivel y no la ruta normal para aplicación.

---

# 171. Public API boundary

La integración deberá favorecer:

```text
Database Facade
Repository
EntityManager
Model API
Query Builder
Schema API
Transaction API
```

según el caso.

---

# 172. Internal API protection

Framework integration podrá utilizar APIs internas cuando sea parte oficial del mismo paquete, pero deberán estar claramente marcadas.

---

# 173. Extension API

Plugins externos sólo deberán depender de:

```text
public
extension
SPI
```

estables.

---

# 174. SPI

Se recomienda distinguir:

```text
API
=
application-facing interface

SPI
=
extension/provider-facing interface
```

---

# 175. Example

```text
Database::query()
```

puede ser API.

Mientras:

```text
DriverProvider
DialectProvider
DatabaseIntegrationProvider
```

son SPI.

---

# 176. Framework internal contract

Puede existir una tercera categoría:

```text
INTERNAL
```

sin compatibilidad pública garantizada.

---

# 177. Dependency rules

Propuesta:

```text
Application
    ↓
Database API

Plugin
    ↓
Database SPI

VoltStack Framework
    ↓
Database Integration Contracts

Database Core
    ↓
Platform Contracts
```

---

# 178. Circular dependency prevention

Prohibido:

```text
Database
→ Authentication
→ ORM
→ Database
```

---

# 179. Solución

Extraer:

```text
ActorProvider contract
```

hacia una frontera neutral.

Entonces:

```text
Authentication
→ implements ActorProvider

Database
→ consumes ActorProvider
```

---

# 180. Otro ejemplo

Incorrecto:

```text
Database
→ HTTP Request
```

Correcto:

```text
HTTP Integration
→ constructs DatabaseOperationContext
→ Database
```

---

# 181. Integration adapters

Un adapter deberá ser pequeño.

Ejemplo:

```php
final class VoltStackCacheAdapter implements DatabaseCacheProvider
{
    public function __construct(
        private CacheManager $cache,
    ) {}
}
```

---

# 182. Adapter ≠ business logic

No deberá convertirse en una segunda implementación de Query Cache.

---

# 183. Bridge semantics

El bridge traduce:

```text
Database concepts
↔
Framework concepts
```

sin redefinirlos.

---

# 184. Error translation

Ejemplo:

```text
Cache Provider Timeout
→ DatabaseCacheUnavailable
```

cuando Database necesite abstraer el proveedor.

---

# 185. Error preservation

La causa original podrá conservarse para diagnostics.

---

# 186. Integration failure classification

Se propone:

```text
MISSING_DEPENDENCY
MISCONFIGURATION
ADAPTER_FAILURE
LIFECYCLE_FAILURE
CONTEXT_FAILURE
RESOURCE_FAILURE
SECURITY_FAILURE
CAPABILITY_MISMATCH
```

---

# 187. IntegrationResult

Cuando corresponda:

```text
SUCCESS
DEGRADED
FAILED
INCONCLUSIVE
UNKNOWN
```

---

# 188. Degraded mode

Ejemplo:

```text
shared result cache unavailable
```

puede permitir:

```text
Database executes without cache
```

si policy lo permite.

---

# 189. No degraded semantics

No deberá degradarse silenciosamente una garantía de correctness.

Ejemplo:

```text
mandatory tenant isolation unavailable
```

deberá fallar.

---

# 190. Required vs optimization integrations

Distinción:

```text
Correctness Integration
vs
Optimization Integration
```

---

# 191. Example correctness

```text
tenant routing
mandatory data access policy
credential provider
```

---

# 192. Example optimization

```text
result cache
compiled metadata cache
telemetry exporter
```

---

# 193. Failure policy

Para una integración `I`:

```text
FailurePolicy(I)
=
ABORT
| DEGRADE
| RETRY
| DISABLE
| QUARANTINE
```

según semántica.

---

# 194. Retry ownership

El adapter no deberá inventar retries si el subsistema propietario ya los administra.

---

# 195. Example

```text
Cache adapter
→ no hidden infinite retries
```

---

# 196. Observability

Cada integración importante podrá emitir:

```text
bootstrap duration
adapter failures
scope lifecycle
resource reset failures
cache bridge status
event bridge status
```

---

# 197. Integration diagnostics

Diagnostics deberán indicar:

```text
integration
operation
severity
cause
recommended action
```

---

# 198. Example

```text
DB-INT-FWK-001

Database result cache is enabled,
but no compatible cache provider is registered.

Action:
Install/enable Quantum Cache or disable database.result_cache.
```

---

# 199. Configuration validation

Debe ocurrir antes del tráfico cuando sea posible.

---

# 200. Fail fast

Ejemplo:

```text
production boot
+
required driver missing
→ fail boot
```

preferible a descubrirlo en el primer request.

---

# 201. Lazy optional initialization

Servicios opcionales costosos podrán inicializarse lazy.

Pero su configuración deberá poder validarse anticipadamente cuando sea razonable.

---

# 202. Framework integration testing

Debe existir una suite dedicada.

Propuesta:

```text
tests/Quantum/Database/Integration/Framework/
```

---

# 203. Test categories

```text
ContainerIntegrationTest
ConfigIntegrationTest
CacheIntegrationTest
EventIntegrationTest
TelemetryIntegrationTest
ValidationIntegrationTest
AuthenticationIntegrationTest
AuthorizationIntegrationTest
QueueIntegrationTest
HttpLifecycleIntegrationTest
RuntimeIntegrationTest
```

---

# 204. Container tests

Deberán comprobar:

```text
bindings
scope
shared services
circular dependencies
aliases
lazy services
```

---

# 205. Config tests

```text
normalization
defaults
invalid combinations
secret references
environment overrides
```

---

# 206. Cache tests

```text
hit
miss
provider failure
invalidation
transaction interaction
```

---

# 207. Event tests

```text
event translation
ordering
listener failure
afterCommit semantics
```

---

# 208. Telemetry tests

```text
trace propagation
redaction
bounded cardinality
exporter failure
```

---

# 209. Validation tests

```text
unique
exists
race condition behavior
constraint remains final authority
```

---

# 210. Authentication tests

```text
actor bridge
optional authentication
CLI without user
job actor context
```

---

# 211. Authorization tests

```text
mandatory policy
scope injection
denial
no query bypass
```

---

# 212. Queue tests

```text
job scope isolation
worker reuse
transaction cleanup
entity serialization policy
retry distinction
```

---

# 213. HTTP lifecycle tests

```text
request scope
response
exception path
streaming response
early termination
reset
```

---

# 214. Persistent runtime tests

Especialmente:

```text
1000 sequential operations
```

verificando:

```text
IdentityMap does not grow across requests
UnitOfWork does not leak
tenant does not leak
transaction does not leak
memory remains bounded
connections reset
```

---

# 215. Concurrent isolation tests

Para runtimes concurrentes:

```text
Operation A tenant = A
Operation B tenant = B
```

deberá demostrarse:

```text
A never observes B context
B never observes A context
```

---

# 216. FrankenPHP conformance

Al ser runtime predeterminado, deberá existir una suite oficial:

```text
DatabaseFrankenPhpIntegrationSuite
```

---

# 217. RoadRunner

Tendrá adapter/suite equivalente cuando se implemente.

---

# 218. OpenSwoole

Además deberá comprobar:

```text
coroutine isolation
concurrent connection leases
context propagation
```

---

# 219. Framework integration performance

Deberá medirse:

```text
container resolution overhead
scope creation cost
scope reset cost
adapter overhead
telemetry overhead
cache bridge overhead
```

sin confundirlo con performance del DBMS.

---

# 220. Fast path

Integración profunda no deberá implicar:

```text
hundreds of dynamic service lookups per query
```

---

# 221. Resolve once per scope

Cuando sea seguro:

```text
Operation Scope
→ resolves DatabaseContext once
```

y los componentes internos reciben referencias directas.

---

# 222. Service locator avoidance

Incorrecto:

```php
public function execute(): void
{
    $container = Container::global();
    $connection = $container->get(Connection::class);
}
```

---

# 223. Dependency injection

Preferir:

```php
public function __construct(
    private ConnectionManager $connections,
) {}
```

según lifecycle.

---

# 224. Static facades

Las Facades públicas pueden ofrecer sintaxis estática.

Pero:

```text
Facade static syntax
≠
static mutable Database state
```

---

# 225. Facade resolution

Debe resolver contra el scope actual.

---

# 226. No active scope

Si una facade scoped es utilizada fuera de un contexto válido:

```text
NoActiveDatabaseScopeException
```

o equivalente.

No deberá crear silenciosamente un scope global persistente.

---

# 227. Standalone Database usage

Aunque integrado con VoltStack, el paquete debería poder ser construido programáticamente.

Ejemplo conceptual:

```php
$database = DatabaseBuilder::create()
    ->connection(...)
    ->build();
```

cuando sea parte de los objetivos del package.

---

# 228. Standalone ≠ duplicate architecture

El builder deberá ensamblar los mismos componentes fundamentales.

---

# 229. Minimal standalone dependencies

Podrán utilizarse adapters:

```text
NullTelemetry
InMemoryEventDispatcher
NoCache
SystemClock
```

cuando semánticamente correctos.

---

# 230. Framework mode

VoltStack podrá proporcionar:

```text
DatabaseBuilder
+
Framework adapters
+
automatic bootstrap
```

sin cambiar el Core.

---

# 231. Deployment optimization

En producción podrá precompilarse:

```text
container
database config
ORM metadata
mapping
extension registries
```

---

# 232. Development mode

Podrá favorecer:

```text
discovery
hot reload
rich diagnostics
debug toolbar
profiling
```

---

# 233. Development ≠ production semantics

La correctness de queries/transacciones no deberá cambiar entre modos.

---

# 234. Debug integrations

Sólo añaden observabilidad.

No redefinen resultados.

---

# 235. Framework Integration Manifest

Podrá construirse un descriptor:

```text
DatabaseIntegrationManifest
```

con:

```text
installed integrations
versions
capabilities
adapters
configuration fingerprints
runtime adapter
```

---

# 236. Manifest security

No deberá incluir secrets.

---

# 237. Diagnostic use

Será útil para:

```text
php voltstack database:diagnose
```

---

# 238. Example

```text
Database Integration

Container       READY
Config          READY
Cache           READY
Events          READY
Telemetry       READY
Validation      READY
Authentication  OPTIONAL / READY
Authorization   OPTIONAL / READY
Multitenancy    NOT INSTALLED
Runtime         FrankenPHP
```

---

# 239. Health ≠ Integration readiness

Debe distinguirse:

```text
Integration READY
≠
Database server healthy
```

---

# 240. Boot readiness

Indica que componentes están correctamente ensamblados.

---

# 241. Runtime health

Indica evidencia operacional actual.

---

# 242. Capability discovery

Las integraciones podrán aportar evidencia al Capability System.

Ejemplo:

```text
PostGIS plugin
→ spatial capability evidence
```

pero la decisión final pertenece al Capability Resolver.

---

# 243. Evidence ≠ Decision

Se conserva:

```text
Integration provides evidence
Capability System decides
```

---

# 244. Extension discovery

Framework bootstrap podrá descubrir plugins.

Pero el Core no deberá escanear todo el filesystem arbitrariamente durante cada request.

---

# 245. Discovery caching

El resultado de discovery podrá compilarse.

---

# 246. Hot reload

En desarrollo podrá invalidarse cuando cambien paquetes/configuración.

---

# 247. Integration contracts versioning

Los contratos públicos deberán seguir versionado compatible.

---

# 248. Adapter compatibility

Un adapter deberá poder declarar:

```text
supported Database API version
supported Framework version
required capabilities
```

---

# 249. Version mismatch

Debe producir error explícito.

No comportamiento indefinido.

---

# 250. Security boundary

Plugins de integración deberán considerarse código privilegiado.

Pueden acceder a:

```text
queries
metadata
connection configuration
```

por lo que no deberán cargarse desde fuentes no confiables.

---

# 251. Framework privilege

Instalar un Database plugin equivale potencialmente a otorgarle acceso significativo al proceso.

---

# 252. No sandbox illusion

VoltStack no deberá afirmar que un plugin PHP ejecutado en el mismo proceso está completamente aislado si no lo está.

---

# 253. Package optionality model

Propuesta:

```text
voltstack/database
    │
    ├── required:
    │     voltstack/platform
    │
    ├── optional:
    │     voltstack/cache
    │     voltstack/events
    │     voltstack/telemetry
    │     voltstack/validation
    │     voltstack/authentication
    │     voltstack/authorization
    │     voltstack/queue
    │     voltstack/multitenancy
    │
    └── runtime adapters:
          frankenphp
          roadrunner
          openswoole
```

La distribución física final podrá diferir, pero la semántica de dependencia deberá conservarse.

---

# 254. Framework dependency matrix

| Integración | Dirección principal | Obligatoria | Scope |
|---|---|---:|---|
| Platform | Database → Platform | Sí | Shared |
| Container | Framework → Database | Framework mode | Shared + Scoped |
| Config | Framework → Database | Framework mode | Shared |
| Cache | Database ↔ Adapter | No | Shared/Scoped |
| Events | Database → Bridge | No | Scoped |
| Telemetry | Database → Bridge | No | Scoped |
| Validation | Validation → Database | No | Scoped |
| Authentication | Authentication → Database | No | Scoped |
| Authorization | Authorization → Database | No | Scoped |
| Queue | Queue → Database | No | Job |
| HTTP | HTTP → Database | No | Request |
| Multitenancy | Multitenancy → Database | No | Operation |
| SaaS | SaaS → Database | No | Application |
| Runtime | Runtime → Database lifecycle | Sí en persistent mode | Operation |

---

# 255. Architectural invariants

## DB-FWK-001

Database Core ≠ Framework Glue.

## DB-FWK-002

Framework Integration deberá respetar dependency direction.

## DB-FWK-003

Database no dependerá de HTTP.

## DB-FWK-004

Database no dependerá de Authentication como requisito core.

## DB-FWK-005

Database no dependerá de Authorization como requisito core.

## DB-FWK-006

Database no dependerá de Multitenancy.

## DB-FWK-007

Database no dependerá de SaaS.

## DB-FWK-008

Optional integration missing ≠ bootstrap failure si la feature está deshabilitada.

## DB-FWK-009

Enabled required feature + missing integration = configuration failure.

## DB-FWK-010

Adapters no redefinirán Database semantics.

---

# 256. Lifecycle invariants

## DB-FWK-011

Request ≠ Transaction.

## DB-FWK-012

Job ≠ Transaction.

## DB-FWK-013

Command ≠ Transaction.

## DB-FWK-014

OperationScope será la abstracción general.

## DB-FWK-015

EntityManager será scoped.

## DB-FWK-016

UnitOfWork será scoped.

## DB-FWK-017

IdentityMap será scoped.

## DB-FWK-018

TransactionContext será scoped.

## DB-FWK-019

DatabaseContext será scoped.

## DB-FWK-020

Mutable operation state no será process singleton.

---

# 257. Persistent runtime invariants

## DB-FWK-021

Worker reuse ≠ state reuse.

## DB-FWK-022

Connection reuse ≠ logical context reuse.

## DB-FWK-023

Reset deberá ejecutarse después de cada operation.

## DB-FWK-024

Failed reset impedirá reutilización segura.

## DB-FWK-025

UNKNOWN reset outcome impedirá reutilización por defecto.

## DB-FWK-026

Tenant state no sobrevivirá al scope.

## DB-FWK-027

Actor state no sobrevivirá al scope.

## DB-FWK-028

Trace context no sobrevivirá incorrectamente al scope.

## DB-FWK-029

Lazy loaders no recuperarán managers globales después del scope.

## DB-FWK-030

OpenSwoole deberá aislar coroutine contexts.

---

# 258. Container invariants

## DB-FWK-031

Container resolverá scoped services según lifecycle.

## DB-FWK-032

Static facade ≠ static mutable state.

## DB-FWK-033

Service Provider ≠ state holder.

## DB-FWK-034

Registries inmutables podrán ser shared.

## DB-FWK-035

Configuration normalizada podrá ser shared.

## DB-FWK-036

Circular dependencies serán detectadas.

## DB-FWK-037

Service locator global no será arquitectura interna por defecto.

## DB-FWK-038

Dependencies deberán ser explícitas.

## DB-FWK-039

Scoped service fuera de scope producirá error claro.

## DB-FWK-040

Container compilation no cambiará semántica.

---

# 259. Cache/Event/Telemetry invariants

## DB-FWK-041

Cache ≠ Database truth.

## DB-FWK-042

Event listener ≠ transaction outcome.

## DB-FWK-043

Telemetry ≠ Database semantics.

## DB-FWK-044

Telemetry failure no invalidará una query exitosa por defecto.

## DB-FWK-045

afterCommit failure no revertirá un commit ocurrido.

## DB-FWK-046

Uncommitted cache data no será publicado como committed.

## DB-FWK-047

UNKNOWN commit deberá producir invalidación conservadora cuando corresponda.

## DB-FWK-048

IdentityMap ≠ framework cache.

## DB-FWK-049

Domain event ≠ Database technical event.

## DB-FWK-050

Raw sensitive query values no serán telemetry labels.

---

# 260. Security invariants

## DB-FWK-051

Authentication User ≠ Tenant.

## DB-FWK-052

DatabaseActor será representación mínima.

## DB-FWK-053

Authorization Policy ≠ Query Builder.

## DB-FWK-054

Mandatory data policy no será listener best-effort.

## DB-FWK-055

Credentials no se resolverán desde globals arbitrarios.

## DB-FWK-056

Secret providers podrán ser reemplazables.

## DB-FWK-057

Audit y telemetry permanecerán diferenciados.

## DB-FWK-058

Password hashing no pertenece a Database.

## DB-FWK-059

Framework adapters deberán respetar redaction.

## DB-FWK-060

Plugins de integración serán considerados código privilegiado.

---

# 261. Queue/HTTP invariants

## DB-FWK-061

Queue Retry ≠ Transaction Retry.

## DB-FWK-062

Commit ≠ Message Delivered.

## DB-FWK-063

Managed Entity ≠ Queue Payload.

## DB-FWK-064

Serialized SPA State ≠ Managed ORM State.

## DB-FWK-065

HTTP status ≠ Database error type.

## DB-FWK-066

HTTP request end no implicará auto-flush.

## DB-FWK-067

HTTP request start no implicará auto-transaction.

## DB-FWK-068

Job end deberá resetear Database scope.

## DB-FWK-069

CLI command end deberá resetear Database scope.

## DB-FWK-070

Streaming HTTP deberá conservar recursos sólo mientras el stream los necesite.

---

# 262. Extension invariants

## DB-FWK-071

API ≠ SPI ≠ INTERNAL.

## DB-FWK-072

Plugins dependerán de SPI pública.

## DB-FWK-073

Registries serán frozen tras bootstrap.

## DB-FWK-074

Duplicate integration IDs no se resolverán silenciosamente.

## DB-FWK-075

Extension discovery no ocurrirá en cada query.

## DB-FWK-076

Integration capabilities ≠ Database capabilities.

## DB-FWK-077

Evidence ≠ Capability Decision.

## DB-FWK-078

Version mismatch será explícito.

## DB-FWK-079

Framework integration podrá ser testeada independientemente.

## DB-FWK-080

Standalone mode reutilizará el mismo Core.

---

# 263. Operational invariants

## DB-FWK-081

Bootstrap readiness ≠ runtime health.

## DB-FWK-082

Connection object existence ≠ connection health.

## DB-FWK-083

Degraded optimization integration podrá deshabilitarse.

## DB-FWK-084

Degraded correctness integration no será aceptada silenciosamente.

## DB-FWK-085

Retry ownership será explícito.

## DB-FWK-086

Adapter failure será clasificable.

## DB-FWK-087

Integration diagnostics serán estructurados.

## DB-FWK-088

Resource ownership será explícito.

## DB-FWK-089

Native tools se ejecutarán mediante adapters controlados.

## DB-FWK-090

Database Core no será shell executor.

---

# 264. Final invariants

## DB-FWK-091

Shared immutable state será favorecido.

## DB-FWK-092

Shared mutable request state será prohibido.

## DB-FWK-093

Composition root ensamblará dependencias.

## DB-FWK-094

Core no conocerá consumidores de alto nivel.

## DB-FWK-095

Framework simplicity no justificará hidden global state.

## DB-FWK-096

Optional package integration permanecerá realmente opcional.

## DB-FWK-097

Runtime adapter no definirá Database semantics.

## DB-FWK-098

Framework bridge no duplicará subsistemas Database.

## DB-FWK-099

Correctness tendrá prioridad sobre integración conveniente.

## DB-FWK-100

Todo lifecycle deberá terminar en un estado demostrablemente limpio o explícitamente tainted.

---

# 265. Anti-pattern: Database God Service

Incorrecto:

```text
DatabaseManager
├── Config
├── Cache
├── Auth
├── Events
├── HTTP
├── Queue
├── Telemetry
├── Multitenancy
├── ORM
├── Driver
└── Everything
```

Debe evitarse.

---

# 266. Anti-pattern: global current connection

```php
Database::$connection
```

como estado mutable global.

Especialmente peligroso en workers persistentes.

---

# 267. Anti-pattern: global current tenant

```php
Tenant::$current
```

sin aislamiento de scope.

---

# 268. Anti-pattern: HTTP-aware ORM

```php
if ($request->user()) {
    // ORM behavior
}
```

dentro del Core.

---

# 269. Anti-pattern: Authentication-aware Query Builder

Query Builder no deberá llamar:

```text
Auth::user()
```

---

# 270. Anti-pattern: hidden transaction

```text
HTTP middleware
→ transaction every request
```

como comportamiento obligatorio e invisible.

Podrá existir como política opt-in para casos específicos.

---

# 271. Anti-pattern: automatic flush

```text
response sent
→ flush all pending entities
```

Incorrecto como default.

---

# 272. Anti-pattern: worker-local EntityManager singleton

Esto produciría:

```text
identity leaks
tenant leaks
dirty entities
memory growth
cross-request contamination
```

---

# 273. Anti-pattern: retry everywhere

```text
driver retries
connection retries
transaction retries
repository retries
HTTP retries
queue retries
```

sobre la misma operación sin coordinación.

---

# 274. Anti-pattern: optional dependency becoming mandatory

Ejemplo:

```text
Database cannot boot because Prometheus is missing
```

cuando telemetry externa no está habilitada.

---

# 275. Anti-pattern: integration through static globals

Evitar:

```text
Auth::user()
Cache::get()
Config::get()
Request::current()
Tenant::current()
```

desde el Core Database.

Las Facades pueden existir en superficie, pero no definir la arquitectura interna.

---

# 276. Anti-pattern: listener for mandatory security

Incorrecto:

```text
QueryExecuting listener
→ add tenant predicate
```

si el listener puede no ejecutarse.

La seguridad obligatoria deberá integrarse en el pipeline semántico.

---

# 277. Anti-pattern: using entities as transport objects

Evitar transportar entidades managed entre:

```text
HTTP requests
jobs
workers
SPA interactions
```

---

# 278. Anti-pattern: framework exception swallowing

El framework no deberá convertir:

```text
commit outcome UNKNOWN
```

en un genérico:

```text
500 database error
```

perdiendo información operacional crítica.

---

# 279. Formalización de integración

Sea:

```text
D = Database Core
F = Framework
A = Integration Adapters
P = Platform Contracts
```

La arquitectura deseada es:

```text
F → A → D → P
```

No:

```text
D → F
```

---

# 280. Formalización de scope

Para cada operación `Oᵢ`:

```text
Sᵢ =
{
 DatabaseContext,
 EntityManager,
 UnitOfWork,
 IdentityMap,
 TransactionContext,
 QueryContext,
 TenantContext?
}
```

Para operaciones independientes:

```text
Mutable(Sᵢ) ∩ Mutable(Sⱼ) = ∅
```

cuando:

```text
i ≠ j
```

---

# 281. Formalización de lifecycle

Una operación sólo podrá considerarse cerrada limpiamente cuando:

```text
CleanClose(O)
=
NoActiveTransaction
∧
NoOwnedOpenCursor
∧
NoOwnedOpenStream
∧
ScopedStateReleased
∧
ConnectionStateSanitized
```

cuando cada condición sea aplicable.

---

# 282. Estado desconocido

Si alguna condición crítica no puede demostrarse:

```text
CleanClose(O) = UNKNOWN
```

entonces los recursos afectados no deberán considerarse automáticamente reutilizables.

---

# 283. Integration safety equation

Una integración `I` será utilizable cuando:

```text
Usable(I)
=
Registered(I)
∧
Configured(I)
∧
Compatible(I)
∧
RequiredCapabilitiesSatisfied(I)
```

---

# 284. Optional feature equation

Para una feature `F`:

```text
Enabled(F)
→
AllRequiredIntegrationsUsable(F)
```

Si no:

```text
ConfigurationFailure
```

---

# 285. Arquitectura propuesta de directorios

```text
src/Quantum/Database/
├── Integration/
│   ├── Contract/
│   │   ├── DatabaseIntegration.php
│   │   ├── DatabaseIntegrationProvider.php
│   │   └── DatabaseRuntimeAdapter.php
│   │
│   ├── Bootstrap/
│   │   ├── DatabaseBootstrapper.php
│   │   ├── DatabaseServiceProvider.php
│   │   ├── IntegrationRegistry.php
│   │   └── IntegrationManifest.php
│   │
│   ├── Container/
│   ├── Config/
│   ├── Cache/
│   ├── Events/
│   ├── Telemetry/
│   ├── Validation/
│   ├── Authentication/
│   ├── Authorization/
│   ├── Queue/
│   ├── Http/
│   ├── Console/
│   ├── Runtime/
│   │   ├── FrankenPhp/
│   │   ├── RoadRunner/
│   │   └── OpenSwoole/
│   ├── Security/
│   ├── Storage/
│   ├── Process/
│   └── Testing/
│
├── Runtime/
│   ├── Scope/
│   ├── Context/
│   ├── Lifecycle/
│   └── Reset/
│
└── ...
```

La ubicación definitiva deberá revisarse en:

```text
330_DATABASE_DIRECTORY_STRUCTURE.md
```

---

# 286. Arquitectura de paquetes

Conceptualmente:

```text
                    VoltStack Application
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
        HTTP              Queue              CLI
          │                 │                 │
          └─────────────────┼─────────────────┘
                            ▼
                   Framework Lifecycle
                            │
                            ▼
                Database Integration Layer
                            │
        ┌───────────────────┼────────────────────┐
        ▼                   ▼                    ▼
     Adapters          Composition Root      Scope Manager
        │                   │                    │
        └───────────────────┼────────────────────┘
                            ▼
                    Database Public API
                            │
                            ▼
                      Database Core
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
          Query            ORM          Transaction
            │               │               │
            └───────────────┼───────────────┘
                            ▼
                     Connection Layer
                            │
                            ▼
                         Driver
                            │
                            ▼
                          DBMS
```

---

# 287. Arquitectura de contexto

```text
Framework Operation
        │
        ▼
RuntimeOperationContext
        │
        ├── operation id
        ├── trace context
        ├── actor?
        ├── tenant?
        ├── runtime
        └── cancellation/deadline?
        │
        ▼
DatabaseOperationScope
        │
        ├── DatabaseContext
        ├── EntityManager
        ├── UnitOfWork
        ├── IdentityMap
        ├── TransactionContext
        └── QueryContext
```

---

# 288. Deadline propagation

Si HTTP/job/runtime posee deadline:

```text
Framework Deadline
       ↓
DatabaseOperationContext
       ↓
Query Timeout/Cancellation
```

podrá propagarse.

---

# 289. Deadline ≠ transaction timeout automáticamente

Los distintos timeouts deberán conservar semántica propia.

---

# 290. Cancellation propagation

Una cancelación externa podrá solicitar cancelación Database.

Pero:

```text
Cancellation requested
≠
statement definitely not executed
```

El outcome deberá conservarse correctamente.

---

# 291. Framework shutdown

Durante shutdown:

```text
stop accepting work
↓
finish/cancel operations according policy
↓
finalize Database scopes
↓
release/discard connections
↓
flush observability
↓
close shared resources
```

---

# 292. Forced shutdown

Si el proceso termina abruptamente no podrá afirmarse que todos los resources fueron limpiados.

---

# 293. Deployment modes

La misma arquitectura deberá funcionar en:

```text
PHP traditional request lifecycle
FrankenPHP worker mode
RoadRunner
OpenSwoole
CLI
queue workers
tests
```

---

# 294. Traditional PHP

Aunque el proceso muera tras cada request, VoltStack no deberá depender de ello para correctness.

---

# 295. Why

Porque el runtime principal será persistente.

Diseñar correctamente para persistent workers también mejora disciplina en runtimes tradicionales.

---

# 296. Integration roadmap principle

Toda nueva integración deberá responder:

```text
Who owns semantics?
Who owns lifecycle?
Who owns retries?
Who owns state?
Who owns cleanup?
Who owns failure classification?
```

Si estas preguntas no tienen respuesta clara, la integración no está suficientemente diseñada.

---

# 297. Regla arquitectónica definitiva

> **VoltStack Database será un subsistema profundamente integrado pero no arquitectónicamente absorbido por el framework. El framework ensamblará Database, proporcionará infraestructura, propagará contextos y coordinará lifecycle; Database conservará la autoridad sobre sus propias semánticas de consulta, persistencia, transacción, conexión y consistencia.**

---

# 298. Principio de propiedad

```text
Framework owns:
    composition
    application lifecycle
    operation context
    high-level integrations

Database owns:
    database semantics
    query semantics
    ORM semantics
    transaction semantics
    connection semantics
    persistence correctness

Driver owns:
    protocol interaction

DBMS owns:
    actual database state
```

---

# 299. Resultado

La arquitectura permite que una aplicación VoltStack experimente:

```php
$user = User::find(10);
```

o:

```php
$user = $users->find(10);
```

mientras internamente existe:

```text
HTTP / SPA / Job / CLI
        ↓
Operation Scope
        ↓
Framework Integration
        ↓
DatabaseContext
        ↓
ORM / Query
        ↓
Execution
        ↓
Connection
        ↓
Driver
        ↓
DBMS
```

sin depender de:

```text
static mutable state
global EntityManagers
global tenants
HTTP-aware ORM
Auth-aware Query Builders
framework-aware Drivers
```

---

# 300. Cierre

`DATABASE_FRAMEWORK_INTEGRATION_ARCHITECTURE` establece la frontera entre la arquitectura Database ya diseñada y el resto de VoltStack.

La idea central puede resumirse en:

```text
Deep Integration
without
Deep Coupling
```

La integración deberá proporcionar al desarrollador la sensación de un framework unificado:

```text
VoltStack
├── Database
├── Cache
├── Events
├── Telemetry
├── Validation
├── Authentication
├── Authorization
├── Queue
└── HTTP
```

mientras internamente se preservan:

```text
component boundaries
dependency direction
explicit contracts
scoped state
optional packages
replaceable adapters
testability
runtime isolation
```

Esto será especialmente importante para VoltStack porque su runtime predeterminado basado en FrankenPHP hará que errores de lifecycle que en PHP tradicional podrían permanecer ocultos se conviertan en problemas reales de:

```text
state leakage
memory growth
cross-request contamination
transaction leakage
tenant leakage
connection contamination
```

Por ello, la integración con el framework deberá diseñarse desde el inicio para procesos persistentes y no añadirse posteriormente como una optimización.

---

# 301. Siguiente documento

```text
312_DATABASE_CONTAINER_INTEGRATION_SYSTEM.md
```

Definirá específicamente cómo `Quantum/Database` se registra y resuelve mediante el Service Container de VoltStack, incluyendo:

```text
Database Service Provider
Container bindings
Shared vs scoped services
DatabaseContext scope
EntityManager scope
ConnectionManager lifecycle
factory bindings
aliases
lazy services
compiled container
extension registration
bootstrap ordering
scope creation
scope destruction
persistent worker isolation
circular dependency prevention
testing overrides
```

y formalizará una regla esencial:

```text
Container Lifetime
must match
Database Semantic Lifetime
```

de forma que servicios como:

```text
TypeRegistry
DialectRegistry
CompiledMetadata
```

puedan compartirse entre operaciones, mientras:

```text
EntityManager
UnitOfWork
IdentityMap
TransactionContext
DatabaseContext
```

permanezcan estrictamente aislados por scope.