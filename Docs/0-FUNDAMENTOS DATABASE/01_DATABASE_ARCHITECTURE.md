# 01_DATABASE_ARCHITECTURE.md

# VoltStack Database — Architecture

## 1. Propósito

Este documento define la arquitectura general de:

```text
VoltStack/Quantum/Database
```

Su objetivo es establecer:

```text
capas
subsistemas
responsabilidades
dependencias permitidas
dependencias prohibidas
flujos de datos
propiedad de estado
puntos de extensión
límites internos
```

Este documento debe ser considerado la referencia arquitectónica principal para todas las implementaciones posteriores del sistema Database.

---

# 2. Objetivo arquitectónico

La arquitectura debe permitir simultáneamente:

```text
simplicidad de uso
alta extensibilidad
bajo acoplamiento
alto rendimiento
compatibilidad multibase de datos
seguridad
observabilidad
runtime persistente
ORM avanzado
Query Engine independiente
```

La arquitectura se organizará alrededor de un principio fundamental:

> Las APIs de alto nivel nunca deberán depender directamente de detalles de infraestructura.

---

# 3. Vista general

La arquitectura completa se divide en las siguientes capas:

```text
Application API
      │
      ▼
ORM / Query API
      │
      ▼
Query Representation
      │
      ▼
Semantic Layer
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
Database Server
```

Alrededor de estas capas existirán subsistemas transversales.

```text
Metadata
Transactions
Schema
Migrations
Cache
Telemetry
Events
Runtime
Security
Testing
Extensions
```

---

# 4. Arquitectura por dominios

El sistema se divide conceptualmente en ocho grandes dominios internos.

```text
1. Infrastructure
2. Query
3. Schema
4. Migration
5. ORM
6. Transaction
7. Runtime
8. Integration
```

Cada dominio deberá mantener límites explícitos.

---

# 5. Dominio Infrastructure

El dominio Infrastructure contiene los componentes responsables de comunicarse con el motor de base de datos.

Incluye:

```text
Driver
Connection
Connection Manager
Connection Pool
Dialect
Platform
Capabilities
```

Arquitectura:

```text
Connection Manager
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

El dominio Infrastructure no deberá conocer:

```text
Entity
ORM
Repository
UnitOfWork
Model
Relationship
```

---

# 6. Driver Layer

El Driver representa la implementación técnica de comunicación con una base de datos.

Ejemplos:

```text
PDO Driver
PostgreSQL native driver
specialized runtime driver
```

Contrato conceptual:

```php
interface DriverInterface
{
    public function connect(
        ConnectionConfiguration $configuration
    ): DriverConnectionInterface;
}
```

El Driver será responsable de crear conexiones físicas.

No deberá contener lógica de ORM ni de construcción de queries.

---

# 7. Connection Layer

La Connection representa una conexión lógica administrada por VoltStack.

Responsabilidades:

```text
connection state
statement preparation
statement execution
transactions
driver delegation
connection metadata
```

Ejemplo conceptual:

```php
interface ConnectionInterface
{
    public function prepare(string $sql): StatementInterface;

    public function beginTransaction(): TransactionInterface;

    public function disconnect(): void;
}
```

La Connection no deberá conocer:

```text
Query AST
EntityManager
Repository
Hydrator
```

---

# 8. Connection Manager

El `ConnectionManager` será responsable de resolver conexiones configuradas.

Ejemplo:

```php
$connection = $connections->connection('primary');
```

Podrá administrar:

```text
default connection
named connections
read connections
write connections
tenant-aware adapters
replicas
connection pools
```

El manager deberá abstraer al resto del framework de la creación directa de conexiones.

---

# 9. Dialect Layer

El Dialect representa diferencias sintácticas del lenguaje SQL.

Ejemplos:

```text
identifier quoting
limit syntax
upsert syntax
returning syntax
json expressions
locking syntax
```

Ejemplo conceptual:

```text
PostgreSqlDialect
MySqlDialect
MariaDbDialect
SqliteDialect
```

El Dialect no ejecuta consultas.

---

# 10. Platform Layer

La Platform representa capacidades semánticas del motor.

Ejemplos:

```text
supportsReturning
supportsSequences
supportsWindowFunctions
supportsRecursiveCTE
supportsGeneratedColumns
supportsSavepoints
supportsPartialIndexes
```

Arquitectura:

```text
Database Platform
     │
     ├── Dialect
     └── Capabilities
```

El compilador utilizará esta información para generar SQL correcto.

---

# 11. Query Domain

El dominio Query será independiente del ORM.

Debe permitir:

```text
query building
query modeling
AST
semantic analysis
optimization
planning
compilation
execution
```

Flujo:

```text
Query API
   ↓
Query Model
   ↓
AST
   ↓
Semantic Layer
   ↓
Optimizer
   ↓
Planner
   ↓
Compiler
   ↓
Executor
```

---

# 12. Query Builder Layer

El Builder será una API de conveniencia.

Ejemplo:

```php
DB::table('users')
    ->where('active', true)
    ->orderBy('name')
    ->get();
```

Internamente:

```text
Builder
   ↓
Query Model
```

El Builder nunca deberá construir SQL mediante concatenación directa.

---

# 13. Query Model Layer

El Query Model representa consultas mediante objetos estructurados.

Ejemplos:

```text
SelectQuery
InsertQuery
UpdateQuery
DeleteQuery
```

El Query Model será independiente de una plataforma SQL específica.

Ejemplo:

```text
SelectQuery
├── source
├── projection
├── predicate
├── joins
├── grouping
├── ordering
└── limit
```

---

# 14. AST Layer

El AST representa la estructura sintáctica de una consulta.

Jerarquía conceptual:

```text
QueryNode
├── SelectNode
├── InsertNode
├── UpdateNode
├── DeleteNode
├── ExpressionNode
├── PredicateNode
├── JoinNode
└── OrderingNode
```

El AST debe ser:

```text
immutable cuando sea posible
serializable conceptualmente
validable
traversable
transformable
independiente del driver
```

---

# 15. Expression System

Las expresiones tendrán su propio modelo.

Ejemplos:

```text
ColumnExpression
LiteralExpression
ParameterExpression
BinaryExpression
FunctionExpression
AggregateExpression
SubqueryExpression
CaseExpression
```

Ejemplo:

```text
users.age > 18
```

se representará como:

```text
BinaryExpression
├── left: Column(users.age)
├── operator: >
└── right: Parameter(18)
```

---

# 16. Semantic Layer

El análisis semántico resolverá el significado de una consulta.

Responsabilidades:

```text
table resolution
column resolution
alias resolution
type resolution
relationship resolution
function resolution
capability validation
```

Flujo:

```text
AST
 ↓
Semantic Analyzer
 ↓
Semantic Graph
```

---

# 17. Semantic Graph

El Semantic Graph representará relaciones semánticas derivadas de la consulta.

Podrá contener:

```text
resolved tables
resolved columns
resolved aliases
resolved joins
type information
relationship information
platform constraints
```

Este grafo será utilizado por:

```text
optimizer
planner
diagnostics
IDE tooling
query validation
```

---

# 18. Query Optimizer

El Optimizer será responsable de transformar la consulta en una forma más eficiente.

Arquitectura:

```text
Semantic Query
      │
      ▼
Optimization Pipeline
      │
      ├── normalization rules
      ├── simplification rules
      ├── duplicate elimination
      ├── relation batching
      └── query rewrite rules
```

Cada regla deberá estar desacoplada.

Ejemplo conceptual:

```php
interface QueryOptimizationRuleInterface
{
    public function optimize(
        QueryPlan $plan,
        OptimizationContext $context
    ): QueryPlan;
}
```

---

# 19. Limitación del Optimizer

El Query Optimizer de VoltStack no sustituirá al optimizer del motor SQL.

No intentará determinar:

```text
physical index selection
disk page access
buffer management
low-level join algorithms
database internal execution costs
```

Eso seguirá siendo responsabilidad del motor.

---

# 20. Query Planner

El Planner convertirá una representación lógica en un plan ejecutable.

Pipeline:

```text
Semantic Query
     ↓
Logical Plan
     ↓
Optimized Plan
     ↓
Physical Plan
     ↓
Execution Plan
```

El planner podrá considerar capacidades de plataforma.

---

# 21. Logical Plan

El Logical Plan representa qué debe hacer la consulta.

Ejemplo:

```text
Project
  ↓
Filter
  ↓
Join
  ↓
Scan(users)
```

No deberá contener todavía detalles SQL concretos.

---

# 22. Physical Plan

El Physical Plan puede incorporar decisiones específicas de plataforma.

Ejemplos:

```text
RETURNING strategy
upsert strategy
pagination strategy
locking strategy
batch execution strategy
```

---

# 23. SQL Compiler

El compilador transforma planes en SQL.

Arquitectura:

```text
Execution Plan
      │
      ▼
Compiler
      │
      ├── PostgreSQL Compiler
      ├── MySQL Compiler
      ├── MariaDB Compiler
      └── SQLite Compiler
```

Resultado conceptual:

```php
CompiledQuery {
    sql
    bindings
    parameterTypes
    executionMetadata
}
```

---

# 24. Compiler invariants

El Compiler deberá cumplir:

```text
no connection access
no statement execution
no ORM dependency
no transaction ownership
no mutable global state
```

Entrada:

```text
Execution Plan
Platform
Compilation Context
```

Salida:

```text
CompiledQuery
```

---

# 25. Execution Layer

El Executor recibe una consulta compilada.

Flujo:

```text
CompiledQuery
      ↓
Executor
      ↓
Connection
      ↓
Statement
      ↓
Result
```

Responsabilidades:

```text
prepare
bind
execute
collect metadata
return result
```

---

# 26. Result Layer

El resultado deberá tener una representación independiente de PDO.

Ejemplos:

```text
ResultInterface
RowResult
CursorResult
AffectedRowsResult
ScalarResult
```

Esto permitirá soportar diferentes drivers.

---

# 27. Streaming

El sistema deberá poder procesar resultados sin cargarlos completamente en memoria.

Ejemplo:

```php
foreach ($query->stream() as $row) {
    // ...
}
```

Internamente:

```text
Statement
   ↓
Cursor
   ↓
Row stream
```

---

# 28. Schema Domain

Schema será un dominio separado del Query Engine.

Responsabilidades:

```text
schema model
schema builder
schema introspection
schema diff
schema compilation
```

Arquitectura:

```text
Schema API
   ↓
Schema Model
   ↓
Schema AST
   ↓
Schema Planner
   ↓
Schema Compiler
```

---

# 29. Schema Model

Representará elementos como:

```text
DatabaseSchema
Table
Column
Index
ForeignKey
Constraint
Sequence
View
```

El modelo deberá ser independiente de la sintaxis SQL.

---

# 30. Schema Introspection

La introspección convertirá estructura real de la base de datos en metadata interna.

```text
Database
   ↓
Schema Introspector
   ↓
Schema Metadata
```

Esto permitirá:

```text
migration generation
schema comparison
database diagnostics
ORM mapping validation
```

---

# 31. Migration Domain

Migration utilizará Schema.

Arquitectura:

```text
Migration
   ↓
Migration Planner
   ↓
Schema Operations
   ↓
Schema Compiler
   ↓
Executor
```

Migration no deberá generar SQL directamente.

---

# 32. ORM Domain

El ORM será una capa superior construida sobre el Query Engine.

Arquitectura:

```text
Entity
   │
   ▼
EntityManager
   │
   ├── Repository
   ├── Metadata
   ├── IdentityMap
   ├── UnitOfWork
   ├── ChangeTracker
   └── PersistencePlanner
          │
          ▼
      Query Engine
```

---

# 33. Entity Model

Las entidades deberán poder ser objetos de dominio independientes.

Ejemplo:

```php
final class User
{
    public function __construct(
        private UserId $id,
        private Email $email,
    ) {
    }
}
```

El objeto no deberá necesitar conocer SQL.

---

# 34. Entity Metadata

Metadata podrá definir:

```text
entity class
table
identifier
columns
types
relationships
indexes
lifecycle hooks
inheritance
generation strategy
```

Fuentes posibles:

```text
PHP attributes
compiled metadata
programmatic mapping
```

---

# 35. Metadata Architecture

El sistema deberá separar:

```text
Metadata Source
      ↓
Metadata Loader
      ↓
Metadata Validator
      ↓
Metadata Compiler
      ↓
Metadata Registry
```

En producción se favorecerá metadata precompilada.

---

# 36. Entity Manager

El `EntityManager` coordinará el contexto ORM.

No será responsable de:

```text
SQL generation
driver communication
query optimization
connection creation
```

Responsabilidades:

```text
find
persist
remove
refresh
detach
clear
flush
repository
context coordination
```

---

# 37. Repository Layer

Repository proporcionará acceso a entidades.

Ejemplo:

```php
$user = $users->find($id);
```

El repository deberá apoyarse en el ORM Query Layer.

Arquitectura:

```text
Repository
   ↓
Entity Query
   ↓
Query Engine
```

---

# 38. Active Record Adapter

Active Record será opcional.

Arquitectura:

```text
Model::save()
    ↓
ActiveRecordAdapter
    ↓
EntityManager
```

Nunca:

```text
Model::save()
    ↓
direct SQL
```

Esto evita tener dos ORM internos.

---

# 39. Identity Map

La Identity Map será propiedad del contexto ORM.

Ejemplo:

```text
PersistenceContext
└── IdentityMap
    ├── User:1
    ├── User:2
    └── Order:10
```

No deberá ser global al worker.

---

# 40. Unit of Work

El Unit of Work será responsable de registrar cambios.

Arquitectura:

```text
EntityManager
     │
     ▼
UnitOfWork
     │
     ├── new
     ├── managed
     ├── dirty
     ├── removed
     └── relation changes
```

---

# 41. Change Tracker

El `ChangeTracker` deberá permanecer desacoplado del Persistence Planner.

Responsabilidad:

```text
detectar diferencias
```

No:

```text
generar UPDATE SQL
```

---

# 42. Persistence Planner

El Persistence Planner transformará cambios de entidades en operaciones persistibles.

Ejemplo:

```text
UnitOfWork
   ↓
Persistence Planner
   ↓
Persistence Operations
```

Operaciones:

```text
InsertEntity
UpdateEntity
DeleteEntity
AttachRelation
DetachRelation
```

---

# 43. Persistence Engine

El Persistence Engine convertirá las operaciones anteriores en consultas.

```text
Persistence Operation
       ↓
Persistence Mapper
       ↓
Query Model
       ↓
Query Engine
```

El ORM no deberá saltarse esta capa para ejecutar SQL.

---

# 44. Hydration Domain

Hydration será un subsistema explícito.

```text
Database Result
      ↓
Hydration Plan
      ↓
Hydrator
      ↓
Entity / DTO / Scalar
```

Tipos de hidratación:

```text
entity
array
scalar
DTO
partial
stream
```

---

# 45. Relationship Layer

Las relaciones serán modeladas mediante metadata.

Tipos iniciales:

```text
OneToOne
OneToMany
ManyToOne
ManyToMany
Polymorphic
```

Cada relación deberá declarar:

```text
ownership
foreign keys
load strategy
cascade rules
persistence behavior
```

---

# 46. Relation Loading

El sistema deberá separar:

```text
relationship definition
relationship loading
relationship persistence
```

Estrategias:

```text
lazy
eager
batch
explicit
```

---

# 47. N+1 Detection

El ORM podrá registrar patrones de carga.

Arquitectura:

```text
Relation Loader
      ↓
Load Metrics
      ↓
N+1 Analyzer
      ↓
Diagnostic Event
```

El detector no deberá alterar silenciosamente la semántica de la aplicación.

---

# 48. Transaction Domain

Transaction será un dominio independiente.

Arquitectura:

```text
Transaction Manager
       │
       ▼
Transaction Context
       │
       ▼
Connection
```

El Transaction Manager podrá coordinar:

```text
begin
commit
rollback
savepoints
retries
isolation
```

---

# 49. Transaction ownership

Una transacción pertenecerá a una conexión o contexto claramente definido.

No deberá existir:

```text
global transaction state
```

especialmente bajo runtimes persistentes.

---

# 50. Transaction API

API sencilla:

```php
Database::transaction(function () {
    // ...
});
```

API avanzada:

```php
$transaction = $manager->begin(
    isolation: IsolationLevel::Serializable
);
```

---

# 51. Nested Transactions

Las transacciones anidadas podrán implementarse mediante:

```text
savepoints
```

cuando la plataforma lo soporte.

De lo contrario deberá existir comportamiento explícito.

Nunca se deberá simular soporte silenciosamente sin garantías.

---

# 52. Runtime Domain

Database será compatible con runtimes persistentes.

Arquitectura:

```text
Worker
  │
  ├── Request Scope
  │      └── Database Context
  │
  └── Persistent Services
```

Debe diferenciar claramente estado reusable y estado request-scoped.

---

# 53. Persistent Services

Podrán sobrevivir entre requests:

```text
compiled metadata
platform descriptors
dialect instances
immutable configuration
compiler registries
query optimization rules
```

---

# 54. Request Scoped State

No deberá sobrevivir:

```text
IdentityMap
UnitOfWork
EntityManager state
active transaction
tenant state
query execution state
hydration context
temporary cursor state
```

---

# 55. Runtime Reset

Al terminar un request:

```text
Request Finished
      ↓
Database Reset Pipeline
      ↓
clear UnitOfWork
clear IdentityMap
close cursors
rollback orphan transactions
release connections
reset tenant context
```

Este flujo será obligatorio.

---

# 56. FrankenPHP Integration

FrankenPHP será el runtime persistente de referencia inicial.

Database deberá integrarse con el lifecycle del worker.

Conceptualmente:

```text
FrankenPHP Worker
      ↓
Request Begin
      ↓
DatabaseContext::open()
      ↓
Application
      ↓
Request End
      ↓
DatabaseContext::reset()
```

---

# 57. Future Runtime Adapters

La arquitectura deberá permitir:

```text
RoadRunnerAdapter
OpenSwooleAdapter
```

sin modificar Query Engine u ORM.

---

# 58. Integration Domain

Database podrá integrarse opcionalmente con otros módulos.

Ejemplos:

```text
Quantum/Cache
Quantum/EventSystem
Quantum/Telemetry
Quantum/Multitenancy
Quantum/Validation
Quantum/Authorization
Quantum/Jobs
Quantum/RuntimeManagerServer
```

Estas dependencias serán opcionales salvo especificación posterior.

---

# 59. Integration Ports

La integración deberá realizarse mediante puertos.

Ejemplo:

```php
interface DatabaseTelemetryPortInterface
{
    public function queryCompleted(QueryTelemetry $event): void;
}
```

Database podrá proporcionar una implementación Null por defecto.

---

# 60. Cache Integration

Database podrá utilizar cache para:

```text
metadata
compiled queries
schema metadata
query results
entities
```

Pero el Query Engine deberá poder funcionar sin `Quantum/Cache`.

---

# 61. Event Integration

Los eventos internos deberán pasar por un contrato.

Ejemplo:

```text
Database Event Port
      │
      ├── Null implementation
      └── EventSystem adapter
```

Esto evita dependencia obligatoria.

---

# 62. Telemetry Integration

Telemetry consumirá información como:

```text
query duration
connection duration
rows
errors
transaction state
N+1 detection
slow queries
```

Database no deberá implementar un sistema de tracing completo por sí mismo.

---

# 63. Multitenancy Integration

Multitenancy será una integración externa.

Arquitectura:

```text
Multitenancy
     ↓
Tenant Database Resolver
     ↓
Connection Manager
```

Opcionalmente:

```text
Tenant
 ├── database-per-tenant
 ├── schema-per-tenant
 └── shared-database
```

Database deberá ofrecer puntos de extensión, no imponer una estrategia.

---

# 64. Dependency Direction

Regla principal:

```text
High-level
   ↓
Low-level
```

Permitido:

```text
ORM
 ↓
Query
 ↓
Execution
 ↓
Connection
 ↓
Driver
```

Prohibido:

```text
Driver
 ↓
ORM
```

---

# 65. Dependency Matrix

Matriz conceptual:

| Componente | Puede depender de |
|---|---|
| ORM | Query, Metadata, Transaction |
| Query Builder | Query Model |
| Semantic | Query AST, Schema Metadata |
| Optimizer | Semantic Model |
| Planner | Optimized Query, Platform |
| Compiler | Plan, Dialect, Platform |
| Executor | Compiled Query, Connection |
| Connection | Driver |
| Driver | Platform primitives |
| Schema | Platform, Compiler |
| Migration | Schema, Execution |
| Runtime | Database lifecycle contracts |
| Integrations | Public Database contracts |

---

# 66. Dependencias prohibidas

Se prohíben explícitamente:

```text
Driver → ORM
Driver → Query Builder
Connection → EntityManager
Compiler → Executor
Compiler → Connection
Executor → ORM
Executor → Entity
Query Builder → Driver
Schema → ORM
ORM → PDO
Entity → Connection
UnitOfWork → SQL string generation
```

---

# 67. Component Boundaries

Cada componente deberá poseer:

```text
single responsibility
public contract
clear input
clear output
defined state ownership
defined lifecycle
```

Un componente que mezcle varias responsabilidades deberá dividirse.

---

# 68. Shared Kernel

Existirá un conjunto mínimo de abstracciones compartidas.

Posible estructura:

```text
Database/Contracts
Database/Support
Database/Value
Database/Exception
```

El Shared Kernel no deberá convertirse en un depósito de lógica.

---

# 69. Contracts Layer

Los contratos fundamentales podrán incluir:

```text
ConnectionInterface
DriverInterface
StatementInterface
ResultInterface
QueryCompilerInterface
QueryExecutorInterface
EntityManagerInterface
RepositoryInterface
TransactionManagerInterface
SchemaManagerInterface
```

Las implementaciones concretas dependerán de ellos.

---

# 70. Immutability

Se favorecerá inmutabilidad en:

```text
Query AST
Query Plans
Metadata
Platform capabilities
Configuration values
Value Objects
Compiled queries
```

El estado mutable se limitará principalmente a:

```text
connections
transactions
UnitOfWork
IdentityMap
cursors
runtime context
```

---

# 71. State Ownership

Cada estado mutable deberá tener propietario.

Ejemplo:

```text
IdentityMap
owner:
PersistenceContext

TransactionState
owner:
TransactionContext

ConnectionState
owner:
Connection

CursorState
owner:
ResultCursor
```

No se permitirán estados ambiguos compartidos.

---

# 72. No Global Mutable State

Se prohíbe utilizar estado mutable global para:

```text
connections
entities
transactions
current tenant
active UnitOfWork
current query
```

Las facades solo podrán resolver servicios scoped.

---

# 73. Exception Architecture

Las excepciones deberán formar una jerarquía explícita.

Ejemplo:

```text
DatabaseException
├── ConnectionException
├── QueryException
├── CompilationException
├── ExecutionException
├── SchemaException
├── MigrationException
├── ORMException
├── TransactionException
└── MappingException
```

Los detalles del driver podrán preservarse como causa.

---

# 74. Error Boundaries

Cada capa deberá traducir errores cuando sea necesario.

Ejemplo:

```text
PDOException
    ↓
DriverException
    ↓
ConnectionException
```

El ORM no deberá exponer errores de PDO directamente.

---

# 75. Configuration Architecture

La configuración será resuelta antes de crear infraestructura mutable.

Flujo:

```text
Config
 ↓
Configuration Resolver
 ↓
Validated Database Configuration
 ↓
Connection Factory
```

La configuración validada deberá ser preferentemente inmutable.

---

# 76. Extension Architecture

El sistema deberá disponer de registries específicos.

Ejemplo:

```text
DriverRegistry
DialectRegistry
PlatformRegistry
TypeRegistry
CompilerRegistry
QueryFunctionRegistry
HydratorRegistry
OptimizationRuleRegistry
```

Evitar un registry universal.

---

# 77. Extension Isolation

Una extensión no deberá tener acceso irrestricto a internals.

Debe recibir contratos públicos.

Ejemplo:

```php
interface DatabaseExtensionInterface
{
    public function register(
        DatabaseExtensionContext $context
    ): void;
}
```

---

# 78. Capability Resolution

Las características específicas deberán resolverse por capacidad.

```text
Feature Request
     ↓
Platform Capability
     ↓
Native
Fallback
Unsupported
```

Esto reduce condicionales por driver.

---

# 79. Feature Fallbacks

Cuando una plataforma no soporte una característica se deberá elegir una de tres estrategias:

```text
native fallback
framework emulation
explicit exception
```

La estrategia deberá estar documentada.

---

# 80. Read/Write Routing

La resolución de conexión podrá incluir:

```text
Operation
   ↓
Connection Router
   ├── read
   └── write
```

El Query Engine no deberá conocer hosts o réplicas.

---

# 81. Replica Architecture

Las replicas estarán debajo del Connection Manager.

```text
Query Executor
     ↓
Connection Resolver
     ↓
Replica Selector
     ↓
Connection
```

Esto mantiene el Query Engine limpio.

---

# 82. Failover

Failover será responsabilidad de infraestructura.

No:

```text
ORM
```

Sí:

```text
Connection Manager
Resilience Layer
```

---

# 83. Resilience Architecture

La resiliencia deberá envolver operaciones de infraestructura.

Ejemplo:

```text
Execution Request
      ↓
Resilience Policy
      ↓
Connection / Statement
```

Policies posibles:

```text
retry
backoff
circuit breaker
failover
```

---

# 84. Retry Safety

No toda consulta será reintentable.

El sistema deberá distinguir:

```text
safe reads
idempotent writes
unsafe writes
transaction-sensitive operations
```

Los retries automáticos deberán ser conservadores.

---

# 85. Security Architecture

La seguridad será transversal.

Puntos principales:

```text
query parameters
identifier handling
credentials
connection encryption
sensitive logging
raw SQL
audit
```

---

# 86. Parameter Safety

Los valores deberán utilizar bindings.

```text
Query
 ↓
Parameter
 ↓
Binding
 ↓
Prepared Statement
```

No concatenación.

---

# 87. Identifier Safety

Los identificadores no se tratarán como parameters SQL tradicionales.

Deberán validarse y escaparse mediante Dialect.

Ejemplo:

```text
table name
column name
alias
```

---

# 88. Raw SQL Boundary

Raw SQL será una API de bajo nivel.

Ejemplo:

```php
DB::statement($sql, $bindings);
```

El uso deberá ser explícito y observable.

---

# 89. Telemetry Boundary

La telemetría nunca deberá alterar el resultado funcional de una query.

Si el sistema de telemetry falla:

```text
Database operation
```

deberá continuar salvo políticas explícitas.

---

# 90. Performance Architecture

El rendimiento se apoyará en:

```text
compiled metadata
compiled query cache
prepared statement reuse
connection reuse
batching
streaming
lazy materialization
minimal reflection
minimal allocations
```

---

# 91. Query Compilation Cache

Cuando sea seguro:

```text
Query Shape
   ↓
Query Cache Key
   ↓
Compiled Query
```

Los valores concretos permanecerán en bindings.

---

# 92. Metadata Cache

El ORM utilizará metadata preprocesada.

```text
Attributes
 ↓
Compile
 ↓
Metadata Cache
 ↓
Runtime
```

---

# 93. Request Lifetime

Durante request:

```text
Request
 ├── DatabaseContext
 ├── EntityManager
 ├── UnitOfWork
 ├── IdentityMap
 └── Transaction Context
```

Al finalizar:

```text
all request state
↓
disposed/reset
```

---

# 94. Worker Lifetime

Durante worker:

```text
Connection pools
Compiled metadata
Platform registry
Dialect registry
Compiler registry
Optimization rules
```

podrán mantenerse.

---

# 95. Process Lifetime

Estado a nivel proceso deberá ser mayormente:

```text
immutable
configuration-driven
safe for reuse
```

---

# 96. Concurrency Safety

La arquitectura no asumirá que todos los runtimes ejecutan exactamente un request por proceso de forma secuencial.

Por tanto, el estado de request deberá ser scoped.

Esto prepara el sistema para:

```text
FrankenPHP
RoadRunner
OpenSwoole
future runtimes
```

---

# 97. Testing Architecture

Cada dominio deberá poder probarse aisladamente.

Ejemplos:

```text
Query AST without database
Compiler without connection
ORM metadata without execution
Driver conformance tests
Schema tests
Transaction tests
```

---

# 98. Driver Conformance

Todo driver deberá superar una suite común.

Ejemplo:

```text
connect
prepare
bind
execute
transaction
savepoint
result
error mapping
disconnect
```

---

# 99. Platform Conformance

Cada plataforma deberá declarar y probar sus capacidades.

Ejemplo:

```text
PostgreSQLPlatformTest
MySQLPlatformTest
MariaDBPlatformTest
SQLitePlatformTest
```

---

# 100. Query Engine Testing

Las fases se probarán por separado:

```text
BuilderTest
ASTTest
SemanticTest
OptimizerTest
PlannerTest
CompilerTest
ExecutorTest
```

Esto ayudará a detectar errores por capa.

---

# 101. ORM Testing

El ORM tendrá pruebas para:

```text
mapping
identity
change tracking
flush
relationships
hydration
cascades
lifecycle
```

---

# 102. Architecture Tests

VoltStack deberá considerar pruebas automáticas de arquitectura.

Por ejemplo:

```text
Compiler namespace
must not depend on
ORM namespace
```

o:

```text
Driver namespace
must not depend on
Query namespace
```

Estas reglas podrían verificarse durante CI.

---

# 103. Namespace Architecture

Estructura inicial sugerida:

```text
src/
└── Quantum/
    └── Database/
        ├── Contracts/
        ├── Configuration/
        ├── Driver/
        ├── Connection/
        ├── Platform/
        ├── Dialect/
        ├── Query/
        ├── Semantic/
        ├── Optimizer/
        ├── Planner/
        ├── Compiler/
        ├── Execution/
        ├── Result/
        ├── Schema/
        ├── Migration/
        ├── ORM/
        ├── Metadata/
        ├── Hydration/
        ├── Persistence/
        ├── Transaction/
        ├── Runtime/
        ├── Integration/
        ├── Extension/
        ├── Exception/
        └── Support/
```

---

# 104. Internal vs Public API

No todas las clases serán públicas.

Se distinguirá:

```text
Public API
Internal API
Extension API
Implementation Detail
```

Solo contratos explícitos tendrán garantía fuerte de compatibilidad.

---

# 105. Public API

Ejemplos:

```text
DatabaseManager
ConnectionInterface
QueryBuilder
EntityManagerInterface
RepositoryInterface
Schema
Transaction API
```

---

# 106. Internal API

Ejemplos potenciales:

```text
AST transformation internals
query planner nodes
specific optimization rules
temporary execution contexts
hydration internals
```

Estas APIs podrán evolucionar más rápido.

---

# 107. Extension API

Las extensiones utilizarán contratos dedicados.

No deberán depender de internals arbitrarios.

---

# 108. Boot Architecture

Durante bootstrap:

```text
Configuration
      ↓
Database Service Provider
      ↓
Registries
      ↓
Connection Manager
      ↓
Query Services
      ↓
ORM Services
```

No se deberán abrir conexiones durante bootstrap salvo necesidad explícita.

---

# 109. Lazy Connection Creation

Las conexiones deberán crearse preferentemente bajo demanda.

Ejemplo:

```text
Application starts
    ↓
no database connection
    ↓
first query
    ↓
connection resolve
```

Esto reduce overhead.

---

# 110. Lifecycle Hooks

Database podrá exponer hooks como:

```text
beforeConnect
afterConnect
beforeQuery
afterQuery
beforeFlush
afterFlush
beforeCommit
afterCommit
```

Pero estos hooks deberán pasar por capas definidas.

---

# 111. Hook Safety

Los hooks no deberán romper invariantes internas.

Por ejemplo, un `beforeQuery` no deberá obtener acceso directo al UnitOfWork salvo que el contrato lo permita explícitamente.

---

# 112. Query Context

Toda consulta podrá transportar contexto.

Ejemplos:

```text
connection name
read/write intent
timeout
tenant context
trace context
query tags
retry policy
```

El contexto deberá ser un objeto explícito.

---

# 113. Execution Context

El `ExecutionContext` contendrá solo información necesaria para ejecución.

No deberá transportar entidades completas.

---

# 114. Persistence Context

El `PersistenceContext` pertenecerá al ORM y agrupará:

```text
EntityManager
IdentityMap
UnitOfWork
ChangeTracker
```

---

# 115. Context Separation

Se deberá mantener:

```text
QueryContext
≠
ExecutionContext
≠
TransactionContext
≠
PersistenceContext
≠
RuntimeContext
```

Pueden estar relacionados, pero no serán el mismo objeto.

---

# 116. Preventing God Objects

No se permitirá una clase central con responsabilidades como:

```text
DatabaseManager
 ├── query building
 ├── ORM
 ├── migrations
 ├── transactions
 ├── compiler
 ├── telemetry
 └── cache
```

El `DatabaseManager` deberá actuar principalmente como punto de acceso/orquestación.

---

# 117. Orchestration vs Implementation

Los managers podrán coordinar, pero no implementar toda la lógica.

Ejemplo:

```text
EntityManager
  delegates to
Metadata
UnitOfWork
RepositoryFactory
PersistenceEngine
```

---

# 118. Service Granularity

La arquitectura deberá evitar dos extremos:

```text
God objects
```

y:

```text
hundreds of trivial microservices
```

Cada servicio deberá representar una responsabilidad coherente.

---

# 119. Data Flow

Flujo completo de lectura ORM:

```text
Repository
   ↓
Entity Query
   ↓
Query Builder
   ↓
Query Model
   ↓
AST
   ↓
Semantic Analysis
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
   ↓
Database
   ↓
Result
   ↓
Hydrator
   ↓
IdentityMap
   ↓
Entity
```

---

# 120. Data Flow de escritura ORM

```text
Entity
   ↓
EntityManager::persist()
   ↓
UnitOfWork
   ↓
ChangeTracker
   ↓
EntityManager::flush()
   ↓
Persistence Planner
   ↓
Persistence Operations
   ↓
Query Engine
   ↓
Compiler
   ↓
Executor
   ↓
Transaction
   ↓
Database
```

---

# 121. Data Flow Query Builder

```text
DB::table()
    ↓
Builder
    ↓
Query Model
    ↓
AST
    ↓
Semantic
    ↓
Optimizer
    ↓
Planner
    ↓
Compiler
    ↓
Executor
```

---

# 122. Data Flow Raw SQL

Raw SQL podrá saltarse parte del pipeline.

```text
Raw SQL
   ↓
Raw Statement
   ↓
Executor
   ↓
Connection
```

Esto deberá estar claramente diferenciado.

---

# 123. Raw SQL Limitations

Raw SQL no obtendrá automáticamente todas las ventajas de:

```text
AST optimization
semantic validation
query portability
query rewriting
```

pero podrá seguir beneficiándose de:

```text
parameter binding
telemetry
transactions
connection management
```

---

# 124. Schema Data Flow

```text
Schema Builder
    ↓
Schema Model
    ↓
Schema AST
    ↓
Schema Planner
    ↓
Dialect Compiler
    ↓
Executor
```

---

# 125. Migration Data Flow

```text
Migration File
    ↓
Migration Loader
    ↓
Migration Plan
    ↓
Schema Operations
    ↓
Schema Compiler
    ↓
Transaction
    ↓
Executor
```

---

# 126. Compilation Boundary

Todos los componentes que produzcan instrucciones SQL estructuradas deberán pasar por compiladores.

Esto incluye:

```text
Query
Schema
Migration operations
```

pero cada dominio podrá tener compiladores especializados.

---

# 127. Shared Compiler Infrastructure

Podrá existir infraestructura compartida para:

```text
identifier quoting
parameter placeholders
platform capabilities
SQL fragments
```

sin mezclar ASTs de diferentes dominios.

---

# 128. Database API Surface

La API principal podrá ofrecer:

```php
DB::connection();

DB::table();

DB::transaction();

DB::select();

DB::statement();
```

ORM:

```php
$entityManager->find();

$entityManager->persist();

$entityManager->flush();
```

Schema:

```php
Schema::create();

Schema::table();

Schema::drop();
```

---

# 129. Facades

Las facades deberán resolver servicios desde Container.

Ejemplo:

```text
DB Facade
   ↓
DatabaseManager contract
```

La facade no deberá contener estado.

---

# 130. CLI Architecture

Los comandos CLI utilizarán APIs públicas.

Ejemplos:

```text
db:inspect
db:migrate
db:rollback
db:status
db:seed
db:diagnose
```

Los comandos no deberán acceder directamente a internals salvo tooling explícito.

---

# 131. Diagnostics Architecture

Database deberá poder exponer diagnósticos estructurados.

Ejemplo:

```text
ConnectionDiagnostic
QueryDiagnostic
SchemaDiagnostic
ORMDiagnostic
RuntimeDiagnostic
```

Esto permitirá tooling avanzado.

---

# 132. Debug Mode

En desarrollo podrá recopilarse información adicional.

Ejemplos:

```text
query origin
compiled SQL
bindings types
execution time
hydration time
N+1 warnings
transaction nesting
```

En producción deberá minimizarse overhead.

---

# 133. Production Mode

En producción se favorecerá:

```text
compiled metadata
cached plans
minimal reflection
minimal diagnostics
strict state reset
optimized connection reuse
```

---

# 134. Architectural Invariants

Las siguientes reglas serán obligatorias:

```text
1. Query Builder never emits SQL directly.
2. ORM never talks directly to PDO.
3. Compiler never executes.
4. Executor never hydrates entities.
5. Driver never knows ORM.
6. Connection never owns persistence state.
7. Request state never leaks across requests.
8. Platform differences use capabilities.
9. Integrations remain optional where possible.
10. Mutable state always has a clear owner.
```

---

# 135. Forbidden Shortcuts

Se consideran atajos arquitectónicos prohibidos:

```text
SQL generation inside Model
direct PDO access from Repository
global EntityManager
static transaction state
driver checks throughout codebase
ORM-specific logic inside Connection
connection-specific logic inside Query AST
```

---

# 136. Architectural Review Rule

Toda nueva funcionalidad deberá responder:

```text
What domain owns it?
What state does it own?
What does it depend on?
Who may depend on it?
What lifecycle does it have?
Is it public or internal?
```

Si estas respuestas no están claras, el componente no deberá incorporarse todavía.

---

# 137. Evolución futura

La arquitectura deberá poder incorporar posteriormente:

```text
additional SQL databases
distributed SQL
custom query languages
additional ORM strategies
async-capable drivers
specialized connection pools
cloud database integrations
advanced sharding
```

sin rediseñar las capas básicas.

---

# 138. Arquitectura resumida

```text
┌────────────────────────────────────┐
│            Application             │
└────────────────┬───────────────────┘
                 │
      ┌──────────┴──────────┐
      │                     │
      ▼                     ▼
 Query API                ORM API
      │                     │
      │              ┌──────┴───────┐
      │              │              │
      │          EntityManager   Repository
      │              │              │
      │          UnitOfWork      Metadata
      │              │              │
      │          Persistence        │
      │              │              │
      └──────────────┴──────────────┘
                     │
                     ▼
                Query Model
                     │
                     ▼
                    AST
                     │
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
                 Compiler
                     │
                     ▼
                 Executor
                     │
                     ▼
              Connection Manager
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

---

# 139. Sistemas laterales

```text
        ┌──────── Metadata
        │
        ├──────── Schema
        │
        ├──────── Migrations
        │
        ├──────── Transactions
        │
Database├──────── Runtime
        ├──────── Security
        ├──────── Telemetry
        ├──────── Cache
        ├──────── Events
        ├──────── Testing
        └──────── Extensions
```

---

# 140. Resultado esperado

Esta arquitectura debe permitir que VoltStack Database tenga una superficie sencilla:

```php
User::where('active', true)->get();
```

mientras internamente utiliza una infraestructura rigurosa:

```text
Model API
   ↓
ORM
   ↓
Query Model
   ↓
AST
   ↓
Semantic
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
   ↓
Driver
```

La complejidad interna deberá proporcionar capacidades avanzadas sin trasladarse innecesariamente al desarrollador.

---

# 141. Principio final de arquitectura

> Ningún componente de VoltStack Database deberá conocer más del sistema de lo estrictamente necesario para cumplir su responsabilidad.

La arquitectura priorizará:

```text
clear boundaries
unidirectional dependencies
explicit state ownership
runtime safety
testability
platform independence
performance
extensibility
developer experience
```

---

# 142. Estado del documento

Este documento define la arquitectura general del nuevo sistema de base de datos.

Los límites conceptuales definidos aquí deberán respetarse en todos los documentos posteriores.

El siguiente documento es:

```text
02_DATABASE_DESIGN_PRINCIPLES.md
```

Ese documento formalizará los principios de diseño, invariantes, reglas de acoplamiento, criterios de modularidad, estrategias de abstracción y decisiones que deberán utilizarse para evaluar cualquier componente nuevo dentro de `VoltStack/Quantum/Database`.