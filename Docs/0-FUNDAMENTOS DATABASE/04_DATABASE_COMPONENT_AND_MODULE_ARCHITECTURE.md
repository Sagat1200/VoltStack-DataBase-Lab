# 04_DATABASE_COMPONENT_AND_MODULE_ARCHITECTURE.md

# VoltStack Quantum Database
## Component and Module Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 04 — Database Component and Module Architecture  
**Estado:** Architecture Specification  
**Nivel:** Core Architecture  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura modular interna de:

```text
VoltStack/Quantum/Database
```

Su objetivo es transformar el modelo conceptual establecido en:

```text
03_DATABASE_DOMAIN_MODEL_AND_TERMINOLOGY.md
```

en una organización técnica concreta de:

- módulos;
- componentes;
- namespaces;
- contratos;
- dependencias;
- capas;
- puntos de extensión;
- boundaries;
- lifecycle;
- integración con otros subsistemas VoltStack.

La arquitectura definida aquí deberá utilizarse como referencia para la implementación física del paquete Database.

---

# 2. Objetivo principal

El principal objetivo es impedir que `Quantum/Database` evolucione hacia una arquitectura donde todos los componentes dependan de todos los demás.

El sistema deberá evitar estructuras como:

```text
ORM
 ↓
QueryBuilder
 ↓
Connection
 ↓
EntityManager
 ↓
Metadata
 ↓
QueryBuilder
```

Este tipo de relación produce:

```text
circular dependencies
hidden coupling
difficult testing
runtime state leakage
unpredictable lifecycle
hard-to-replace components
```

VoltStack utilizará una arquitectura dirigida por dependencias explícitas.

---

# 3. Principio fundamental

La regla estructural principal es:

> Las dependencias deben apuntar hacia abstracciones de menor nivel y nunca regresar hacia capas superiores.

Modelo simplificado:

```text
Public API
    │
    ▼
Domain Services
    │
    ▼
Planning / Persistence
    │
    ▼
Query / Schema Models
    │
    ▼
Compilation / Execution
    │
    ▼
Connection
    │
    ▼
Driver
```

Una capa inferior no deberá conocer las APIs de mayor nivel que la utilizan.

---

# 4. Arquitectura general

`Quantum/Database` se divide inicialmente en los siguientes dominios internos:

```text
VoltStack/Quantum/Database
│
├── Contract
├── Configuration
├── Connection
├── Driver
├── Platform
├── Type
├── Query
├── Semantic
├── Optimization
├── Planning
├── Compilation
├── Execution
├── Result
├── Schema
├── Migration
├── Transaction
├── ORM
├── Metadata
├── Hydration
├── Persistence
├── Relationship
├── Repository
├── Cache
├── Topology
├── Lifecycle
├── Telemetry
├── Security
├── Resilience
├── Extension
├── Exception
└── Support
```

Estos módulos representan boundaries arquitectónicos.

No todos deberán convertirse necesariamente en paquetes Composer independientes.

---

# 5. Vista por capas

La arquitectura puede visualizarse como:

```text
┌───────────────────────────────────────────────┐
│                 PUBLIC API                    │
│                                               │
│ DB │ Model │ Repository │ Schema │ Transaction│
├───────────────────────────────────────────────┤
│              APPLICATION SERVICES             │
│                                               │
│ ORM │ Repository │ Schema │ Migration         │
├───────────────────────────────────────────────┤
│             DOMAIN / PLANNING                  │
│                                               │
│ Persistence │ Relationships │ Query Planning  │
├───────────────────────────────────────────────┤
│               QUERY ENGINE                    │
│                                               │
│ AST │ Semantic │ Optimization │ Compilation   │
├───────────────────────────────────────────────┤
│             EXECUTION ENGINE                  │
│                                               │
│ Executor │ Result │ Transaction               │
├───────────────────────────────────────────────┤
│           DATABASE INFRASTRUCTURE             │
│                                               │
│ Connection │ Driver │ Platform │ Types        │
├───────────────────────────────────────────────┤
│               DATABASE                        │
│                                               │
│ MySQL │ MariaDB │ PostgreSQL │ SQLite         │
└───────────────────────────────────────────────┘
```

---

# 6. Clasificación de módulos

Los módulos se clasifican en cinco categorías.

## 6.1 Foundation Modules

Proporcionan primitivas básicas.

```text
Contract
Configuration
Platform
Type
Exception
Support
```

## 6.2 Infrastructure Modules

Gestionan comunicación con bases de datos.

```text
Connection
Driver
Transaction
Execution
Result
```

## 6.3 Query Modules

Implementan representación y procesamiento de consultas.

```text
Query
Semantic
Optimization
Planning
Compilation
```

## 6.4 Persistence Modules

Implementan persistencia orientada a objetos.

```text
ORM
Metadata
Hydration
Persistence
Relationship
Repository
```

## 6.5 Cross-Cutting Modules

Implementan capacidades transversales.

```text
Cache
Topology
Lifecycle
Telemetry
Security
Resilience
Extension
```

---

# 7. Grafo principal de dependencias

El grafo conceptual será:

```text
                    Public API
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
         ORM          Schema       Query API
          │             │             │
          ▼             ▼             ▼
    Persistence      Schema AST     Query AST
          │                           │
          │                           ▼
          │                       Semantic
          │                           │
          └──────────────┬────────────┘
                         ▼
                    Optimization
                         │
                         ▼
                      Planning
                         │
                         ▼
                    Compilation
                         │
                         ▼
                     Execution
                         │
                ┌────────┼────────┐
                ▼        ▼        ▼
          Transaction  Result  Connection
                                  │
                                  ▼
                                Driver
                                  │
                                  ▼
                              Database
```

---

# 8. Regla de dirección

Las dependencias deberán respetar aproximadamente:

```text
High-Level
    │
    ▼
Low-Level
```

Nunca:

```text
Low-Level
    │
    ▼
High-Level
```

Ejemplo permitido:

```text
ORM
 ↓
Query
```

Ejemplo prohibido:

```text
Query
 ↓
ORM
```

---

# 9. Contract Module

Namespace conceptual:

```php
VoltStack\Quantum\Database\Contract
```

Contendrá contratos internos fundamentales.

Ejemplos:

```text
ConnectionInterface
DriverInterface
DialectInterface
PlatformInterface
QueryCompilerInterface
QueryExecutorInterface
TransactionInterface
ResultInterface
EntityManagerInterface
RepositoryInterface
MetadataProviderInterface
```

Sin embargo, los contratos deberán mantenerse cerca de su dominio cuando ello mejore cohesión.

Por ejemplo:

```php
VoltStack\Quantum\Database\Connection\Contract\ConnectionInterface
```

podrá preferirse frente a un directorio global gigantesco.

---

# 10. Regla de contratos

Los contratos deberán existir cuando exista al menos una de estas razones:

```text
multiple implementations
extension boundary
test substitution
runtime adapter
external integration
architectural isolation
```

No se crearán interfaces automáticamente para cada clase.

Debe evitarse:

```text
UserFactoryInterface
UserFactory
```

si únicamente existe una implementación interna sin boundary real.

---

# 11. Platform Contracts

Algunos contratos mínimos podrán existir en:

```text
VoltStack/Platform
```

cuando otros subsistemas del framework necesiten depender de Database sin conocer `Quantum/Database`.

Ejemplos:

```php
VoltStack\Platform\Database\DatabaseManagerInterface

VoltStack\Platform\Database\ConnectionInterface

VoltStack\Platform\Database\TransactionInterface
```

La implementación será proporcionada por:

```text
VoltStack/Quantum/Database
```

---

# 12. Dependency Inversion con Platform

Modelo:

```text
Quantum/Database
       │
       ▼
Platform Contracts
       ▲
       │
Other Quantum Packages
```

Nunca:

```text
Platform
   │
   ▼
Quantum/Database
```

`Platform` no deberá depender de paquetes Quantum concretos.

---

# 13. Configuration Module

Namespace:

```php
VoltStack\Quantum\Database\Configuration
```

Responsabilidades:

```text
configuration models
configuration normalization
connection configuration
pool configuration
topology configuration
ORM configuration
migration configuration
runtime configuration
```

No deberá realizar conexiones.

Ejemplo:

```text
DatabaseConfiguration
ConnectionConfiguration
PoolConfiguration
OrmConfiguration
```

---

# 14. Driver Module

Namespace:

```php
VoltStack\Quantum\Database\Driver
```

Responsabilidad:

> Proporcionar primitivas de comunicación de bajo nivel con una tecnología Database.

Estructura conceptual:

```text
Driver
│
├── Contract
├── MySQL
├── MariaDB
├── PostgreSQL
└── SQLite
```

Ejemplo:

```php
DriverInterface
MySQLDriver
MariaDBDriver
PostgreSQLDriver
SQLiteDriver
```

---

# 15. Dependencias del Driver

Driver podrá depender de:

```text
Contract
Configuration
Platform primitives
Type primitives
Exception
Support
```

No podrá depender de:

```text
ORM
EntityManager
Repository
QueryBuilder
SchemaBuilder
Migration
Hydration
Persistence
```

---

# 16. Platform Module

Namespace:

```php
VoltStack\Quantum\Database\Platform
```

Responsabilidades:

```text
database capabilities
dialect identification
feature support
type mapping
SQL capability metadata
platform-specific behavior
```

Estructura:

```text
Platform
│
├── Capability
├── Dialect
├── MySQL
├── MariaDB
├── PostgreSQL
└── SQLite
```

---

# 17. Driver vs Platform vs Dialect

La separación obligatoria será:

```text
Driver
    → comunicación

Dialect
    → sintaxis

Platform
    → capacidades
```

Ejemplo:

```text
PostgreSQLDriver
PostgreSQLDialect
PostgreSQLPlatform
```

No deberán fusionarse en una única clase monolítica.

---

# 18. Type Module

Namespace:

```php
VoltStack\Quantum\Database\Type
```

Responsabilidades:

```text
logical database types
PHP conversion
driver conversion
platform mapping
custom type registration
```

Ejemplos:

```text
StringType
IntegerType
BooleanType
DecimalType
DateTimeType
UuidType
JsonType
EnumType
```

---

# 19. Connection Module

Namespace:

```php
VoltStack\Quantum\Database\Connection
```

Responsabilidades:

```text
logical connections
physical connections
connection resolution
connection lifecycle
pool interaction
read/write targets
connection reset
```

Estructura conceptual:

```text
Connection
│
├── Contract
├── Manager
├── Resolver
├── Factory
├── Pool
├── State
└── Reset
```

---

# 20. DatabaseManager

La implementación principal podrá residir en:

```php
VoltStack\Quantum\Database\Connection\DatabaseManager
```

o:

```php
VoltStack\Quantum\Database\DatabaseManager
```

dependiendo de la decisión final de Public API.

Será responsable de:

```text
resolve named connection
resolve default connection
coordinate connection factory
coordinate topology
expose connection API
```

No deberá convertirse en un objeto omnisciente.

---

# 21. Query Module

Namespace:

```php
VoltStack\Quantum\Database\Query
```

Será uno de los módulos fundamentales.

Submódulos:

```text
Query
│
├── Builder
├── Model
├── AST
├── Expression
├── Predicate
├── Parameter
├── Context
├── Validation
└── Normalization
```

---

# 22. Query Builder Boundary

`Query\Builder` podrá conocer:

```text
Query Model
Query AST
Expressions
Predicates
Parameters
```

No podrá conocer:

```text
PDO
Driver
PhysicalConnection
EntityManager
UnitOfWork
Hydrator
```

Regla:

```text
Builder
   │
   ▼
Query Model / AST
```

Nunca:

```text
Builder
   │
   ▼
SQL String
```

como arquitectura principal.

---

# 23. Query AST Module

Namespace:

```php
VoltStack\Quantum\Database\Query\AST
```

Contendrá representaciones estructurales.

Ejemplos:

```text
SelectQueryNode
InsertQueryNode
UpdateQueryNode
DeleteQueryNode

TableNode
ColumnNode
JoinNode
PredicateNode
OrderByNode
LimitNode
ParameterNode
```

Idealmente los nodos serán:

```text
immutable
side-effect free
serializable when useful
easy to inspect
easy to transform
```

---

# 24. Semantic Module

Namespace:

```php
VoltStack\Quantum\Database\Semantic
```

Responsabilidades:

```text
symbol resolution
table resolution
column resolution
alias resolution
type inference
relationship resolution
constraint analysis
semantic graph construction
```

Entrada:

```text
Query AST
```

Salida:

```text
Semantic Query Model
```

o:

```text
Semantic Graph
```

---

# 25. Semantic Dependency Rules

Semantic podrá depender de:

```text
Query AST
Schema Metadata
Type System
Metadata contracts
Platform capabilities
```

No deberá depender de:

```text
Driver
PhysicalConnection
EntityManager state
UnitOfWork
```

salvo que una operación explícita de introspección se encuentre abstraída detrás de Metadata.

---

# 26. Optimization Module

Namespace:

```php
VoltStack\Quantum\Database\Optimization
```

Responsabilidades:

```text
query rewrite
predicate simplification
projection reduction
query deduplication
relation query optimization
framework-level optimization
```

Submódulos:

```text
Optimization
│
├── Rule
├── Pipeline
├── Context
├── Cost
└── Analysis
```

---

# 27. Optimizer Rule Model

Cada optimización debería poder representarse como una regla.

Ejemplo:

```php
interface OptimizationRule
{
    public function supports(QueryPlan $plan): bool;

    public function optimize(
        QueryPlan $plan,
        OptimizationContext $context
    ): QueryPlan;
}
```

Conceptualmente:

```text
Plan
 │
 ▼
Rule 1
 │
 ▼
Rule 2
 │
 ▼
Rule N
 │
 ▼
Optimized Plan
```

---

# 28. Planning Module

Namespace:

```php
VoltStack\Quantum\Database\Planning
```

Responsabilidades:

```text
logical planning
physical planning
connection target selection
query ordering
batch planning
execution dependency planning
```

Submódulos:

```text
Planning
│
├── Logical
├── Physical
├── Execution
├── Cost
└── Strategy
```

---

# 29. Planning Boundary

Planner puede decidir:

```text
what to execute
where to execute
in what order
using what strategy
```

pero no deberá:

```text
execute SQL
hydrate entities
manage PDO
```

---

# 30. Compilation Module

Namespace:

```php
VoltStack\Quantum\Database\Compilation
```

Responsabilidad:

> Convertir una representación planificada en instrucciones compatibles con una plataforma concreta.

Submódulos:

```text
Compilation
│
├── Contract
├── SQL
├── MySQL
├── MariaDB
├── PostgreSQL
├── SQLite
└── Cache
```

---

# 31. Compiler Boundary

Compiler podrá depender de:

```text
Query AST
Execution Plan
Dialect
Platform
Type
```

No deberá depender de:

```text
EntityManager
Repository
UnitOfWork
ConnectionPool
Telemetry implementation
```

---

# 32. Execution Module

Namespace:

```php
VoltStack\Quantum\Database\Execution
```

Responsabilidades:

```text
compiled query execution
statement lifecycle
parameter binding
timeout
cancellation
retry coordination
result creation
```

Estructura:

```text
Execution
│
├── Executor
├── Statement
├── Parameter
├── Context
├── Timeout
├── Cancellation
└── Error
```

---

# 33. Executor Boundary

El Executor conoce:

```text
CompiledQuery
Connection
Driver execution primitives
Result
Transaction Context
```

No conoce:

```text
Entity
Model
Repository
Relationship
UnitOfWork
```

Esta es una de las invariantes más importantes.

---

# 34. Result Module

Namespace:

```php
VoltStack\Quantum\Database\Result
```

Contendrá:

```text
Result
ResultSet
Row
Cursor
Stream
ResultMetadata
```

El Result Module debe permanecer independiente del ORM.

---

# 35. ORM no debe contaminar Result

Esto es incorrecto:

```text
QueryExecutor
     │
     ▼
User Entity
```

Lo correcto:

```text
QueryExecutor
     │
     ▼
Result
     │
     ▼
Hydrator
     │
     ▼
User Entity
```

---

# 36. Schema Module

Namespace:

```php
VoltStack\Quantum\Database\Schema
```

Submódulos:

```text
Schema
│
├── Model
├── Metadata
├── Builder
├── AST
├── Introspection
├── Diff
├── Planning
└── Compilation
```

El Schema System tendrá su propio AST.

---

# 37. Schema Boundary

Schema podrá utilizar:

```text
Connection
Platform
Dialect
Type
Execution
```

pero no deberá depender de:

```text
ORM
EntityManager
Repository
UnitOfWork
```

El ORM sí podrá utilizar Schema Metadata cuando sea necesario.

---

# 38. Migration Module

Namespace:

```php
VoltStack\Quantum\Database\Migration
```

Responsabilidades:

```text
migration discovery
migration ordering
migration repository
migration planning
migration execution
rollback
batching
safety analysis
```

Flujo:

```text
Migration
   │
   ▼
Schema Operations
   │
   ▼
Schema AST
   │
   ▼
Schema Planner
   │
   ▼
Compiler
   │
   ▼
Execution
```

---

# 39. Transaction Module

Namespace:

```php
VoltStack\Quantum\Database\Transaction
```

Responsabilidades:

```text
transaction lifecycle
nested transactions
savepoints
isolation
retry
deadlock handling
transaction context
```

No deberá depender del ORM.

---

# 40. ORM Module

Namespace:

```php
VoltStack\Quantum\Database\ORM
```

Este módulo implementará el motor de persistencia orientado a objetos.

Submódulos conceptuales:

```text
ORM
│
├── Entity
├── Model
├── Manager
├── State
├── UnitOfWork
├── IdentityMap
├── Mapping
└── Lifecycle
```

---

# 41. ORM Dependency Direction

El ORM puede depender de:

```text
Metadata
Persistence
Hydration
Relationship
Repository
Query
Transaction
Type
```

Pero los módulos inferiores no deberán depender del ORM.

Especialmente:

```text
Connection  ✗→ ORM
Driver      ✗→ ORM
Execution   ✗→ ORM
Result      ✗→ ORM
Query AST   ✗→ ORM
```

---

# 42. Active Record Module

La API Active Record podrá existir en:

```php
VoltStack\Quantum\Database\ORM\Model
```

o mediante una capa pública equivalente.

Ejemplo:

```php
class User extends Model
{
}
```

Pero:

```text
Model
  │
  ▼
EntityManager
  │
  ▼
Shared ORM Engine
```

Nunca:

```text
Model
  │
  ▼
Independent SQL Engine
```

---

# 43. Metadata Module

Namespace:

```php
VoltStack\Quantum\Database\Metadata
```

Responsabilidades:

```text
entity metadata
field metadata
relationship metadata
metadata loading
metadata compilation
metadata caching
metadata registry
```

Submódulos:

```text
Metadata
│
├── Model
├── Reader
├── Loader
├── Compiler
├── Registry
└── Cache
```

---

# 44. Metadata Dependency Rule

Metadata debe ser relativamente independiente.

Podrá depender de:

```text
Type
Platform abstractions
Support
```

No deberá depender de:

```text
EntityManager runtime state
UnitOfWork
Connection
Executor
```

Esto permite compilar metadata durante bootstrap.

---

# 45. Hydration Module

Namespace:

```php
VoltStack\Quantum\Database\Hydration
```

Responsabilidades:

```text
entity hydration
scalar hydration
array hydration
DTO hydration
partial hydration
hydration planning
```

Hydration puede depender de:

```text
Result
Metadata
Type
IdentityMap contracts
Relationship metadata
```

---

# 46. Persistence Module

Namespace:

```php
VoltStack\Quantum\Database\Persistence
```

Responsabilidades:

```text
insert planning
update planning
delete planning
dependency ordering
change-set persistence
flush planning
batch persistence
```

Submódulos:

```text
Persistence
│
├── Engine
├── Planner
├── Insert
├── Update
├── Delete
├── Flush
└── Batch
```

---

# 47. Persistence Boundary

Persistence transforma:

```text
ORM State
   │
   ▼
Persistence Operations
   │
   ▼
Query Model
```

No deberá generar SQL directamente.

Correcto:

```text
Persistence
   ↓
Query AST
   ↓
Compiler
```

Incorrecto:

```text
Persistence
   ↓
"INSERT INTO users..."
```

---

# 48. Relationship Module

Namespace:

```php
VoltStack\Quantum\Database\Relationship
```

Responsabilidades:

```text
relationship metadata
relation loading
eager loading
lazy loading
batch loading
relation persistence
N+1 analysis
```

Submódulos:

```text
Relationship
│
├── Metadata
├── Loader
├── Planner
├── Eager
├── Lazy
├── Batch
└── Persistence
```

---

# 49. Repository Module

Namespace:

```php
VoltStack\Quantum\Database\Repository
```

Responsabilidades:

```text
repository resolution
base repositories
custom repositories
entity-oriented queries
repository metadata
```

Repository puede depender de:

```text
EntityManager
Query API
Metadata
Hydration
```

pero no deberá implementar ejecución SQL propia.

---

# 50. Cache Module

Namespace:

```php
VoltStack\Quantum\Database\Cache
```

Será un boundary de integración.

Tipos de cache:

```text
CompiledQueryCache
QueryPlanCache
ResultCache
MetadataCache
EntityCache
```

---

# 51. Quantum Cache Integration

Database no deberá depender obligatoriamente de:

```text
VoltStack/Quantum/Cache
```

En su lugar:

```text
Database
   │
   ▼
Cache Contract
   ▲
   │
Quantum/Cache Adapter
```

Sin `Quantum/Cache`, Database deberá poder operar mediante:

```text
NullCache
ArrayCache
InternalCache
```

según el caso.

---

# 52. Topology Module

Namespace:

```php
VoltStack\Quantum\Database\Topology
```

Responsabilidades:

```text
read/write routing
replicas
failover targets
shards
partition routing
tenant database routing
database topology
```

---

# 53. Topology Boundary

Topology determina:

```text
where
```

Query Planner determina:

```text
how
```

Connection ejecuta:

```text
through which connection
```

Esta distinción deberá mantenerse.

---

# 54. Lifecycle Module

Namespace:

```php
VoltStack\Quantum\Database\Lifecycle
```

Responsabilidades:

```text
database context lifecycle
request scope
worker scope
state reset
connection reset
ORM reset
resource cleanup
```

Este módulo es crítico para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 55. DatabaseContext

El Lifecycle Module administrará conceptualmente:

```text
DatabaseContext
│
├── QueryContext
├── TransactionContext
├── ORMContext
├── TenantContext reference
└── TelemetryContext reference
```

No necesariamente mediante un único objeto monolítico.

---

# 56. State Scope Registry

Todo componente stateful deberá declarar su scope.

Ejemplo:

| Componente | Scope |
|---|---|
| DriverRegistry | Application |
| MetadataRegistry | Application |
| Compiled Metadata | Application |
| Connection Pool | Worker/Application |
| DatabaseContext | Request |
| EntityManager | Request |
| IdentityMap | Request |
| UnitOfWork | Request |
| TransactionContext | Transaction |
| QueryContext | Operation |
| HydrationContext | Operation |

---

# 57. Persistent Runtime Rule

La arquitectura deberá funcionar bajo:

```text
Request A
   │
   ▼
DatabaseContext A
   │
   ▼
RESET
   │
   ▼
Request B
   │
   ▼
DatabaseContext B
```

Ningún estado mutable request-scoped deberá sobrevivir accidentalmente.

---

# 58. Telemetry Module

Namespace:

```php
VoltStack\Quantum\Database\Telemetry
```

Será un boundary hacia:

```text
VoltStack/Quantum/Telemetry
```

Database deberá emitir señales internas neutrales.

Ejemplo:

```text
QueryStarted
QueryFinished
QueryFailed
ConnectionOpened
TransactionCommitted
SlowQueryDetected
NPlusOneDetected
```

---

# 59. Telemetry Dependency Rule

El Query Engine no deberá depender directamente del paquete completo Telemetry.

Preferencia:

```text
QueryExecutor
     │
     ▼
DatabaseTelemetryContract
     ▲
     │
Telemetry Adapter
```

Esto mantiene Database funcional de forma independiente.

---

# 60. Security Module

Namespace:

```php
VoltStack\Quantum\Database\Security
```

Responsabilidades específicas de Database:

```text
safe parameter handling
credential protection
sensitive query metadata filtering
connection security
query auditing hooks
```

No sustituirá:

```text
Quantum/Authentication
Quantum/Authorization
Quantum/Validation
```

---

# 61. Resilience Module

Namespace:

```php
VoltStack\Quantum\Database\Resilience
```

Responsabilidades:

```text
retry policies
failure classification
connection recovery
failover coordination
resource exhaustion protection
circuit breaker integration
```

---

# 62. Extension Module

Namespace:

```php
VoltStack\Quantum\Database\Extension
```

Responsabilidades:

```text
driver registration
dialect registration
type registration
compiler extensions
query extensions
ORM extensions
capability discovery
```

---

# 63. Extension Points

Puntos de extensión oficiales podrán incluir:

```text
Driver
Dialect
Platform
DatabaseType
QueryExpression
OptimizationRule
PlannerStrategy
QueryCompiler
Hydrator
MetadataDriver
Repository
PersistenceStrategy
TelemetryAdapter
CacheAdapter
```

No todo componente interno deberá ser extensible.

---

# 64. Exception Module

Namespace:

```php
VoltStack\Quantum\Database\Exception
```

Taxonomía conceptual:

```text
DatabaseException
│
├── ConfigurationException
├── ConnectionException
├── DriverException
├── QueryException
├── CompilationException
├── ExecutionException
├── TransactionException
├── SchemaException
├── MigrationException
├── MappingException
├── HydrationException
└── PersistenceException
```

---

# 65. Exception Translation

Errores nativos:

```text
PDOException
native driver errors
database error codes
```

deberán traducirse cuando sea posible:

```text
Native Error
     │
     ▼
Driver Error Translator
     │
     ▼
VoltStack DatabaseException
```

---

# 66. Support Module

Namespace:

```php
VoltStack\Quantum\Database\Support
```

Contendrá únicamente utilidades realmente compartidas.

Ejemplos posibles:

```text
Identifier
Name
Hash
Collection primitives
Immutable helpers
```

No deberá convertirse en:

```text
Support/
   └── everything-that-does-not-fit-anywhere
```

---

# 67. Regla de cohesión

Cada módulo deberá responder a una pregunta principal.

Ejemplos:

```text
Connection:
¿Cómo obtenemos y administramos una conexión?

Query:
¿Cómo representamos una consulta?

Semantic:
¿Qué significa esa consulta?

Optimizer:
¿Cómo podemos mejorarla?

Planner:
¿Cómo debemos ejecutarla?

Compiler:
¿Cómo se expresa para esta plataforma?

Execution:
¿Cómo la ejecutamos?

Hydration:
¿Cómo convertimos el resultado?

Persistence:
¿Cómo sincronizamos cambios ORM?
```

Si un módulo responde demasiadas preguntas, probablemente requiere división.

---

# 68. Dependencias permitidas — Foundation

Matriz conceptual:

| Módulo | Puede depender de |
|---|---|
| Contract | Platform contracts mínimos |
| Configuration | Contract, Support |
| Type | Contract, Platform abstractions |
| Exception | Support mínimo |
| Support | Nada de alto nivel |

---

# 69. Dependencias permitidas — Infrastructure

| Módulo | Puede depender de |
|---|---|
| Driver | Contract, Config, Type, Platform, Exception |
| Connection | Driver, Config, Platform, Exception |
| Transaction | Connection, Driver primitives, Exception |
| Execution | Connection, Transaction, Result, Compilation |
| Result | Type, Support |

---

# 70. Dependencias permitidas — Query Engine

| Módulo | Puede depender de |
|---|---|
| Query | Type, Metadata abstractions |
| Semantic | Query, Schema Metadata, Type |
| Optimization | Semantic, Query |
| Planning | Optimization, Semantic, Topology abstractions |
| Compilation | Planning, Query, Dialect, Platform, Type |

---

# 71. Dependencias permitidas — ORM

| Módulo | Puede depender de |
|---|---|
| Metadata | Type, Support |
| Hydration | Metadata, Result, Type |
| Relationship | Metadata, Query, Hydration |
| Repository | Query, Metadata, ORM contracts |
| Persistence | Metadata, Query, Transaction |
| ORM | Metadata, Persistence, Hydration, Relationship, Repository |

---

# 72. Dependencias prohibidas

Las siguientes dependencias deberán considerarse violaciones arquitectónicas salvo ADR explícito.

```text
Driver          → ORM
Driver          → Repository
Driver          → QueryBuilder

Connection      → EntityManager
Connection      → UnitOfWork

Query AST       → Driver
Query AST       → Connection
Query AST       → ORM

Compiler        → EntityManager
Compiler        → UnitOfWork
Compiler        → Repository

Executor        → Entity
Executor        → Model
Executor        → Hydrator

Result          → Entity
Result          → Repository

Metadata        → EntityManager runtime
Metadata        → Connection

Platform        → ORM
Platform        → Migration

Transaction     → ORM

Support         → ORM
Support         → Query Engine
```

---

# 73. Anti-Circular Dependency Rule

El grafo completo deberá ser acíclico a nivel de módulos.

Conceptualmente:

```text
A → B → C
```

es válido.

```text
A → B → C → A
```

es inválido.

Se deberán implementar tests de arquitectura para verificar esta propiedad.

---

# 74. Arquitectura verificable

La suite de testing deberá poder detectar:

```text
forbidden namespace dependencies
circular module dependencies
layer violations
public/internal API violations
```

Ejemplo conceptual:

```php
architecture()
    ->module('Driver')
    ->mustNotDependOn('ORM');
```

---

# 75. Public vs Internal API

Cada componente deberá clasificarse como:

```text
Public API
Extension API
Internal API
```

## Public API

Estable para aplicaciones.

Ejemplos:

```text
DB
Model
Repository
Schema
Transaction API
```

## Extension API

Estable para paquetes/extensiones.

Ejemplos:

```text
DriverInterface
DatabaseType
OptimizationRule
CompilerExtension
```

## Internal API

Puede evolucionar entre versiones menores según política.

Ejemplos:

```text
internal planner nodes
optimizer state
hydration scratch buffers
```

---

# 76. Public API Boundary

La aplicación no deberá necesitar acceder normalmente a:

```text
Query AST internals
SemanticGraph internals
PhysicalQueryPlan
PDO wrappers
UnitOfWork internals
Compiler internals
```

La complejidad deberá quedar encapsulada.

---

# 77. Namespace inicial propuesto

```text
src/
└── Quantum/
    └── Database/
        ├── Contract/
        ├── Configuration/
        ├── Connection/
        ├── Driver/
        ├── Platform/
        ├── Type/
        ├── Query/
        ├── Semantic/
        ├── Optimization/
        ├── Planning/
        ├── Compilation/
        ├── Execution/
        ├── Result/
        ├── Schema/
        ├── Migration/
        ├── Transaction/
        ├── ORM/
        ├── Metadata/
        ├── Hydration/
        ├── Persistence/
        ├── Relationship/
        ├── Repository/
        ├── Cache/
        ├── Topology/
        ├── Lifecycle/
        ├── Telemetry/
        ├── Security/
        ├── Resilience/
        ├── Extension/
        ├── Exception/
        └── Support/
```

Esta estructura podrá refinarse en:

```text
330_DATABASE_DIRECTORY_STRUCTURE.md
```

---

# 78. Componentes Core

No todos los módulos tendrán el mismo nivel de importancia.

El núcleo operacional mínimo será:

```text
Configuration
      │
      ▼
Connection
      │
      ▼
Driver
      │
      ▼
Platform

Query
 │
 ▼
Compilation
 │
 ▼
Execution
 │
 ▼
Result
```

Esto permitirá utilizar Database sin ORM.

---

# 79. Database sin ORM

VoltStack deberá permitir:

```php
DB::table('users')
    ->where('active', true)
    ->get();
```

sin inicializar innecesariamente:

```text
EntityManager
UnitOfWork
IdentityMap
RelationshipManager
ORM Metadata
```

Esto reduce overhead.

---

# 80. Database con ORM

Cuando se utiliza:

```php
User::find(42);
```

se activa la capa ORM:

```text
Model API
    │
    ▼
EntityManager
    │
    ▼
Repository / Query
    │
    ▼
Query Engine
```

El motor inferior sigue siendo el mismo.

---

# 81. Lazy Module Activation

Los módulos costosos deberán poder activarse bajo demanda.

Ejemplo:

```text
Request
  │
  ├── DB::table(...)
  │      └── Query Engine
  │
  └── no ORM initialization
```

Mientras:

```text
Request
  │
  └── User::find(...)
         │
         └── ORM initialized
```

Esto es importante para performance.

---

# 82. Bootstrap Architecture

Durante bootstrap deberán registrarse principalmente componentes:

```text
stateless
immutable
compiled
shared-safe
```

Ejemplo:

```text
DriverRegistry
DialectRegistry
TypeRegistry
CompiledMetadataRegistry
CompilerRegistry
```

No deberá crearse durante bootstrap:

```text
request IdentityMap
request UnitOfWork
active TransactionContext
```

---

# 83. Request Initialization

Al comenzar un request podrán crearse:

```text
DatabaseContext
EntityManager
IdentityMap
UnitOfWork
QueryContext factory
```

según necesidad.

---

# 84. Request Termination

Al terminar un request:

```text
flush policy check
transaction cleanup
cursor cleanup
result stream cleanup
ORM clear
UnitOfWork reset
IdentityMap reset
connection reset/release
context destruction
```

---

# 85. Runtime Architecture

```text
Application Boot
      │
      ▼
Shared Safe Services
      │
      ├── DriverRegistry
      ├── TypeRegistry
      ├── MetadataCache
      └── Compilers
      │
      ▼
Worker
      │
      ├──────── Request A
      │            │
      │            ▼
      │      DatabaseContext A
      │            │
      │            ▼
      │          RESET
      │
      └──────── Request B
                   │
                   ▼
             DatabaseContext B
```

---

# 86. Integración con Container

`Quantum/Container` deberá registrar contratos y factories.

Ejemplo conceptual:

```php
$container->singleton(
    DriverRegistry::class,
    DriverRegistryFactory::class
);

$container->scoped(
    EntityManagerInterface::class,
    EntityManagerFactory::class
);
```

La distinción entre:

```text
singleton
scoped
transient
```

será crítica.

---

# 87. Regla de servicios Singleton

Podrán ser singleton únicamente servicios que sean:

```text
stateless
immutable
thread/worker safe
explicitly synchronized when needed
```

Ejemplos:

```text
Dialect
PlatformCapabilityRegistry
CompiledMetadata
TypeRegistry
```

---

# 88. Servicios Request-Scoped

Normalmente:

```text
DatabaseContext
EntityManager
IdentityMap
UnitOfWork
```

deberán ser scoped al request o lifecycle equivalente.

---

# 89. Servicios Operation-Scoped

Ejemplos:

```text
QueryContext
OptimizationContext
PlanningContext
CompilationContext
HydrationContext
```

podrán existir únicamente durante una operación.

---

# 90. Integración con Config

`Quantum/Database` consumirá configuración mediante contratos de:

```text
VoltStack/Quantum/Config
```

pero el modelo Database normalizará la configuración a objetos propios.

Ejemplo:

```text
Framework Config
      │
      ▼
Database Config Loader
      │
      ▼
DatabaseConfiguration
```

El dominio no deberá leer arrays globales constantemente.

---

# 91. Integración con EventSystem

Database deberá publicar eventos mediante un boundary.

```text
Database
    │
    ▼
Database Event Bridge
    │
    ▼
Quantum/EventSystem
```

No deberá existir dependencia fuerte desde cada componente hacia EventSystem.

---

# 92. Integración con Telemetry

Modelo:

```text
Database Instrumentation
       │
       ▼
Telemetry Contract
       │
       ▼
Database Telemetry Bridge
       │
       ▼
Quantum/Telemetry
```

Esto permitirá Database sin Telemetry completo.

---

# 93. Integración con Cache

Modelo:

```text
Database Cache Contract
        │
        ▼
Cache Bridge
        │
        ▼
Quantum/Cache
```

El Cache package será opcional.

---

# 94. Integración con Validation

ORM podrá utilizar `Quantum/Validation` mediante un integration layer.

No deberá producirse:

```text
EntityManager
   │
   ▼
Validation internals
```

Preferencia:

```text
ORM Lifecycle
   │
   ▼
Validation Bridge
   │
   ▼
Quantum/Validation
```

---

# 95. Integración con Authentication

Database no deberá conocer Authentication.

Correcto:

```text
Authentication
     │
     ▼
Repository
     │
     ▼
Database
```

Incorrecto:

```text
Database
   │
   ▼
Authentication
```

---

# 96. Integración con Authorization

Igualmente:

```text
Authorization
     │
     ▼
Database
```

o mediante adapters explícitos.

Database no deberá decidir permisos de aplicación.

---

# 97. Integración con Multitenancy

Multitenancy será opcional.

Modelo:

```text
Quantum/Multitenancy
        │
        ▼
Database Tenant Adapter
        │
        ▼
Topology / Connection Resolution
```

Database sólo expondrá contratos de contexto necesarios.

---

# 98. Integración con Jobs

El lifecycle Database deberá funcionar también fuera de HTTP.

Ejemplo:

```text
Queue Worker
    │
    ▼
Job Begin
    │
    ▼
DatabaseContext
    │
    ▼
Job Execute
    │
    ▼
Database Reset
```

Por ello, el término interno preferido no deberá depender exclusivamente de `Request`.

---

# 99. Execution Scope

Se define un concepto genérico:

```text
ExecutionScope
```

que podrá representar:

```text
HTTP Request
Queue Job
CLI Command
Scheduled Task
WebSocket Message
Worker Operation
```

DatabaseContext deberá poder vincularse a un ExecutionScope.

---

# 100. Integración con FrankenPHP

FrankenPHP será la implementación runtime primaria.

Database deberá soportar:

```text
worker reuse
connection reuse
safe request isolation
state reset
resource cleanup
persistent metadata
persistent compiled query cache
```

---

# 101. RoadRunner y OpenSwoole

La misma arquitectura deberá funcionar mediante adapters:

```text
Lifecycle
    │
    ├── FrankenPHP Adapter
    ├── RoadRunner Adapter
    └── OpenSwoole Adapter
```

La lógica Database no deberá contener:

```php
if ($runtime === 'frankenphp') {
    ...
}
```

disperso por múltiples módulos.

---

# 102. Arquitectura de adapters

Los adapters externos deberán permanecer en boundaries específicos.

Ejemplo:

```text
Integration/
├── Cache/
├── Telemetry/
├── Runtime/
├── Event/
├── Multitenancy/
└── Validation/
```

La ubicación física final se determinará posteriormente.

---

# 103. Ports and Adapters

Aunque VoltStack Database no utilizará obligatoriamente una implementación académica estricta de Hexagonal Architecture, sí adoptará el principio:

```text
Core
 │
 ▼
Port
 ▲
 │
Adapter
```

Ejemplo:

```text
Database Telemetry Core
        │
        ▼
TelemetryPort
        ▲
        │
QuantumTelemetryAdapter
```

---

# 104. Evitar Service Locator

Los módulos no deberán resolver servicios arbitrariamente desde el Container.

Incorrecto:

```php
$container->get(EntityManager::class);
```

desde cualquier clase interna.

Preferencia:

```php
public function __construct(
    private EntityManagerInterface $entityManager
) {}
```

Las dependencias deberán ser visibles.

---

# 105. Context Objects

Los Context Objects deberán utilizarse cuando una operación requiera información transversal.

Ejemplos:

```text
QueryContext
PlanningContext
ExecutionContext
HydrationContext
DatabaseContext
```

No deberán convertirse en bolsas genéricas de servicios.

---

# 106. Context Rule

Un Context contiene:

```text
state
metadata
operation-specific configuration
```

No deberá convertirse en:

```text
Service Locator
```

Ejemplo prohibido:

```php
$context->get(Container::class);
```

---

# 107. Immutable Models

Siempre que sea práctico deberán ser inmutables:

```text
Query AST
Schema AST
Execution Plans
Compiled Queries
Metadata
Configuration
```

Los componentes de lifecycle sí podrán contener estado controlado.

---

# 108. Stateful Components

Los principales componentes stateful serán:

```text
Connection
TransactionContext
DatabaseContext
EntityManager
IdentityMap
UnitOfWork
Cursor
Stream
```

Cada uno deberá tener lifecycle explícito.

---

# 109. Stateless Components

Idealmente:

```text
Compiler
Dialect
Platform
Optimizer Rules
Metadata Compiler
Type Definitions
```

deberán ser stateless o immutable.

Esto facilita:

```text
reuse
parallelism
persistent workers
testing
```

---

# 110. Concurrency

La arquitectura deberá asumir futuras operaciones concurrentes.

No se deberá almacenar estado de operación en servicios compartidos.

Incorrecto:

```php
class QueryCompiler
{
    private array $parameters = [];
}
```

si el compiler es singleton.

Preferencia:

```text
CompilationContext
```

creado por operación.

---

# 111. Module Communication

Los módulos podrán comunicarse mediante:

```text
method calls
contracts
immutable models
domain events
context objects
```

Se evitará comunicación mediante:

```text
global mutable state
static registries with runtime state
service locator
hidden singletons
```

---

# 112. Static API

VoltStack podrá ofrecer facades:

```php
DB::table(...)
```

pero la facade no representa el diseño interno.

Conceptualmente:

```text
DB Facade
   │
   ▼
Container Binding
   │
   ▼
DatabaseManager
```

---

# 113. Model Static API

Igualmente:

```php
User::query()
```

deberá resolverse sobre servicios scoped.

No deberá mantener globalmente:

```text
EntityManager
Connection
TenantContext
Transaction
```

en propiedades estáticas persistentes.

---

# 114. Dependency Matrix General

| Módulo | Query | Connection | ORM | Metadata | Execution | Platform |
|---|---:|---:|---:|---:|---:|---:|
| Driver | No | No | No | No | No | Sí |
| Connection | No | — | No | No | No | Sí |
| Query | — | No | No | Opcional | No | Mínimo |
| Semantic | Sí | No | No | Sí | No | Sí |
| Optimizer | Sí | No | No | Sí | No | Opcional |
| Planner | Sí | Abstracta | No | Sí | No | Sí |
| Compiler | Sí | No | No | No | No | Sí |
| Execution | No | Sí | No | No | — | Mínimo |
| Hydration | No | No | Contratos | Sí | Result | No |
| Persistence | Sí | No | Contratos | Sí | Indirecto | No |
| ORM | Sí | Indirecto | — | Sí | Indirecto | No |

La matriz definitiva será ampliada posteriormente.

---

# 115. Component Ownership

Cada responsabilidad tendrá un único propietario principal.

Ejemplos:

```text
SQL generation
    → Compilation

Connection lifecycle
    → Connection

Entity state
    → ORM / UnitOfWork

Relation loading
    → Relationship

Query optimization
    → Optimization

Database capability
    → Platform

Result iteration
    → Result

Runtime cleanup
    → Lifecycle
```

Esto evita duplicación.

---

# 116. Prohibición de lógica duplicada

No deberán existir dos implementaciones independientes de:

```text
SQL generation
parameter binding
transaction nesting
type conversion
entity identity
relationship mapping
connection selection
```

Las APIs superiores deberán delegar al componente propietario.

---

# 117. Architectural Invariants

### DB-MOD-001

El grafo de dependencias entre módulos deberá permanecer acíclico.

### DB-MOD-002

Driver nunca dependerá del ORM.

### DB-MOD-003

Connection nunca dependerá de EntityManager.

### DB-MOD-004

Query AST nunca dependerá de infraestructura de ejecución.

### DB-MOD-005

Compiler nunca ejecutará queries.

### DB-MOD-006

Executor nunca hidratará Entities.

### DB-MOD-007

Persistence nunca generará SQL directamente.

### DB-MOD-008

Result nunca dependerá del ORM.

### DB-MOD-009

Metadata compilable deberá permanecer independiente del runtime state.

### DB-MOD-010

Todo estado mutable deberá tener scope explícito.

### DB-MOD-011

Los paquetes opcionales deberán integrarse mediante boundaries.

### DB-MOD-012

Active Record y Repository deberán compartir el mismo ORM Engine.

### DB-MOD-013

Las facades no deberán contener estado Database real.

### DB-MOD-014

El Container no deberá utilizarse como Service Locator interno.

### DB-MOD-015

La infraestructura Database deberá poder funcionar sin ORM.

---

# 118. Dependency Flow Final

El flujo principal queda definido como:

```text
Application API
      │
      ▼
ORM / Query / Schema
      │
      ▼
Domain Models
      │
      ▼
Semantic Processing
      │
      ▼
Optimization
      │
      ▼
Planning
      │
      ▼
Compilation
      │
      ▼
Execution
      │
      ▼
Connection
      │
      ▼
Driver
      │
      ▼
Database
```

Y para persistencia:

```text
Entity
   │
   ▼
EntityManager
   │
   ▼
UnitOfWork
   │
   ▼
ChangeSet
   │
   ▼
Persistence Planner
   │
   ▼
Query Model
   │
   ▼
Query Engine
```

---

# 119. Cross-Cutting Architecture

Los módulos transversales se conectarán mediante ports:

```text
                     Cache
                       ▲
                       │
Telemetry ◄────── Database ──────► Events
                       │
                       ▼
                    Security
                       │
                       ▼
                   Resilience
```

Ninguno deberá introducir ciclos hacia el núcleo.

---

# 120. Arquitectura completa resumida

```text
┌─────────────────────────────────────────────────────────────┐
│                        PUBLIC API                           │
│ DB │ Model │ Repository │ Schema │ Transaction              │
└────────────────────────────┬────────────────────────────────┘
                             │
             ┌───────────────┼────────────────┐
             ▼               ▼                ▼
            ORM            Query            Schema
             │               │                │
      ┌──────┼──────┐        ▼                ▼
      ▼      ▼      ▼      Query AST       Schema AST
 Identity  UoW   Metadata      │                │
   Map       │                 ▼                │
             ▼              Semantic            │
       Persistence             │                │
             │                 ▼                │
             └──────────► Optimization          │
                               │                │
                               ▼                │
                            Planning ◄──────────┘
                               │
                               ▼
                          Compilation
                               │
                               ▼
                           Execution
                               │
                  ┌────────────┼────────────┐
                  ▼            ▼            ▼
               Result     Transaction   Connection
                                            │
                                            ▼
                                          Driver
                                            │
                                            ▼
                                         Database
```

---

# 121. Resultado arquitectónico

La división modular permite que VoltStack Database mantenga simultáneamente:

```text
simple public API
strong internal boundaries
testable components
replaceable drivers
portable query representation
advanced ORM
persistent runtime safety
optional integrations
future distributed capabilities
```

sin convertir el sistema en un monolito internamente acoplado.

---

# 122. Decisiones fundamentales

La arquitectura modular queda basada en las siguientes decisiones:

1. `Query` será independiente del ORM.

2. `Execution` será independiente de Hydration.

3. `Connection` será independiente de Query y ORM.

4. `Driver`, `Dialect` y `Platform` serán componentes separados.

5. `ORM` utilizará el Query Engine común.

6. `Persistence` producirá operaciones Query estructuradas, no SQL.

7. `Schema` tendrá su propio AST.

8. `Metadata` será estructurada y compilable.

9. `Lifecycle` será explícito.

10. Los módulos stateful tendrán scope declarado.

11. FrankenPHP será runtime prioritario, pero no estará codificado dentro del núcleo Database.

12. RoadRunner y OpenSwoole podrán integrarse mediante adapters.

13. Cache, Telemetry, Events y Multitenancy serán integraciones opcionales.

14. `Platform` contendrá contratos mínimos y nunca dependerá de `Quantum/Database`.

15. Las dependencias circulares estarán prohibidas y verificadas automáticamente.

---

# 123. Regla arquitectónica maestra

La arquitectura completa puede resumirse mediante una sola regla:

> Cada módulo debe conocer únicamente la información necesaria para cumplir su responsabilidad y nunca la implementación de las capas superiores que consumen sus servicios.

En forma estructural:

```text
API
 ↓
Domain
 ↓
Plan
 ↓
Compile
 ↓
Execute
 ↓
Connect
 ↓
Driver
```

y nunca:

```text
Driver
 ↓
Connection
 ↓
Executor
 ↓
ORM
 ↓
API
```

---

# 124. Conclusión

`VoltStack/Quantum/Database` se diseñará como un conjunto de módulos altamente cohesionados y débilmente acoplados.

La arquitectura evita deliberadamente que:

```text
ORM
Query Builder
SQL Compiler
Connection
Driver
Schema
Transactions
Metadata
```

se conviertan en una única infraestructura entrelazada.

El resultado esperado es un Database System donde:

```text
Query Builder
    ≠ SQL Compiler

SQL Compiler
    ≠ Executor

Executor
    ≠ Connection

Connection
    ≠ Driver

Driver
    ≠ Platform

ORM
    ≠ Query Engine

Entity
    ≠ Row

IdentityMap
    ≠ Cache

DatabaseContext
    ≠ Global State
```

Esta separación constituye una de las garantías estructurales más importantes del nuevo diseño de VoltStack Database.

---

# 125. Siguiente documento

El siguiente documento deberá formalizar los contratos y abstracciones que permiten mantener los boundaries establecidos aquí:

```text
05_DATABASE_CONTRACTS_AND_ABSTRACTIONS.md
```

Ese documento deberá definir, entre otros:

```text
contract taxonomy
public contracts
internal contracts
extension contracts
ports
adapters
manager contracts
connection contracts
driver contracts
query contracts
compiler contracts
executor contracts
transaction contracts
ORM contracts
metadata contracts
lifecycle contracts
telemetry/cache integration contracts
```

y establecer cuáles pertenecen a:

```text
VoltStack/Platform
```

y cuáles deberán permanecer exclusivamente dentro de:

```text
VoltStack/Quantum/Database
```.