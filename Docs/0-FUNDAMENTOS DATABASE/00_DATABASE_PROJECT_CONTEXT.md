# 00_DATABASE_PROJECT_CONTEXT.md

# VoltStack Database — Project Context

## 1. Introducción

`VoltStack/Quantum/Database` es el subsistema oficial de acceso, modelado, consulta, persistencia y administración de bases de datos del framework VoltStack.

Su objetivo es proporcionar una infraestructura de datos moderna, extensible y de alto rendimiento que combine:

- la simplicidad de desarrollo de Laravel;
- la separación arquitectónica de Doctrine;
- un motor interno propio basado en Query AST;
- análisis semántico;
- planificación;
- compilación;
- ejecución desacoplada;
- persistencia mediante Unit of Work;
- Identity Map;
- Entity Manager;
- Repository;
- soporte opcional de Active Record;
- integración nativa con runtimes persistentes;
- extensibilidad mediante drivers, dialectos, compiladores y capacidades.

El sistema será diseñado desde cero como una arquitectura propia de VoltStack.

No será una adaptación de Eloquent, Doctrine ORM ni Doctrine DBAL.

---

# 2. Nombre del subsistema

El nombre conceptual oficial será:

```text
VoltStack/Quantum/Database
```

Posible paquete Composer:

```text
voltstack/database
```

Namespace principal:

```php
VoltStack\Quantum\Database
```

Ejemplos de namespaces internos:

```php
VoltStack\Quantum\Database\Connection
VoltStack\Quantum\Database\Driver
VoltStack\Quantum\Database\Query
VoltStack\Quantum\Database\Schema
VoltStack\Quantum\Database\Migration
VoltStack\Quantum\Database\ORM
VoltStack\Quantum\Database\Transaction
VoltStack\Quantum\Database\Persistence
VoltStack\Quantum\Database\Metadata
VoltStack\Quantum\Database\Compiler
VoltStack\Quantum\Database\Execution
```

---

# 3. Propósito

El sistema Database debe permitir que una aplicación VoltStack pueda trabajar con bases de datos desde diferentes niveles de abstracción.

Estos niveles incluyen:

```text
Raw SQL
   ↓
Connection API
   ↓
Query Builder
   ↓
Query Engine
   ↓
ORM
   ↓
Domain Model
```

El programador podrá elegir el nivel adecuado para cada problema.

No será obligatorio utilizar ORM para utilizar Database.

Tampoco será obligatorio utilizar Active Record.

---

# 4. Filosofía principal

La filosofía del sistema será:

> API sencilla en la superficie, arquitectura rigurosa internamente.

El desarrollador podrá escribir:

```php
$users = DB::table('users')
    ->where('active', true)
    ->orderBy('name')
    ->get();
```

o:

```php
$user = User::find(1);
```

o:

```php
$user = $userRepository->find(1);
```

sin que estas APIs tengan que conocer directamente:

```text
PDO
SQL generation
database dialects
query execution
connection state
transaction internals
```

Todas deberán utilizar componentes internos claramente separados.

---

# 5. Objetivo arquitectónico

El flujo principal de consultas será:

```text
Developer API
      │
      ▼
Query Builder
      │
      ▼
Query Model
      │
      ▼
Query AST
      │
      ▼
Semantic Analysis
      │
      ▼
Semantic Graph
      │
      ▼
Query Optimizer
      │
      ▼
Query Planner
      │
      ▼
Execution Plan
      │
      ▼
SQL Compiler
      │
      ▼
Statement
      │
      ▼
Executor
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

El ORM deberá utilizar el mismo motor.

```text
ORM
 │
 ▼
Persistence Engine
 │
 ▼
Query Engine
```

No deberá existir un generador SQL separado dentro del ORM.

---

# 6. Principio fundamental de separación

Una de las reglas principales del nuevo diseño será evitar que los componentes conozcan responsabilidades que no les pertenecen.

Se establece desde este documento la siguiente regla:

```text
ORM
≠
Query Builder
≠
Compiler
≠
Executor
≠
Connection
≠
Driver
```

Cada capa deberá mantener límites claramente definidos.

---

# 7. Reglas arquitectónicas fundamentales

Las siguientes reglas serán invariantes del sistema.

## 7.1 ORM no genera SQL

El ORM podrá producir operaciones de persistencia.

Ejemplo conceptual:

```text
InsertEntity
UpdateEntity
DeleteEntity
LoadEntity
```

Estas operaciones deberán convertirse posteriormente en consultas mediante el Query Engine.

---

## 7.2 Query Builder no genera SQL

El Query Builder deberá producir estructuras de consulta.

Ejemplo:

```text
Query Builder
      ↓
Query Model
      ↓
AST
```

Nunca:

```text
Query Builder
      ↓
SQL string
```

---

## 7.3 Compiler no ejecuta consultas

El compilador será responsable exclusivamente de transformar planes o AST compatibles en SQL.

```text
Query Plan
    ↓
Compiler
    ↓
SQL + Bindings
```

No deberá abrir conexiones ni ejecutar statements.

---

## 7.4 Executor no conoce entidades

El sistema de ejecución trabajará con:

```text
Statements
Parameters
Results
Cursors
Rows
```

No deberá conocer:

```text
Entity
Repository
UnitOfWork
Model
Relationship
```

La hidratación pertenece a otra capa.

---

## 7.5 Connection no conoce ORM

Una conexión solamente administra comunicación con el motor de base de datos.

Conceptualmente:

```text
Connection
 ├── connect
 ├── disconnect
 ├── prepare
 ├── execute
 ├── transaction
 └── metadata
```

No deberá conocer entidades ni repositories.

---

# 8. Experiencia de desarrollo

Uno de los objetivos principales será mantener una experiencia similar a Laravel para operaciones comunes.

Ejemplo:

```php
$users = DB::table('users')
    ->where('active', true)
    ->get();
```

También podrá existir:

```php
$user = User::query()
    ->where('email', $email)
    ->first();
```

Pero internamente ambas APIs deberán utilizar el mismo motor de consulta.

---

# 9. Dos modelos de ORM

VoltStack Database permitirá dos estilos de programación.

## 9.1 Data Mapper

Será el modelo arquitectónico principal.

Ejemplo:

```php
$user = $users->find($id);

$user->changeEmail($email);

$entityManager->flush();
```

Componentes principales:

```text
Entity
Repository
EntityManager
UnitOfWork
IdentityMap
PersistenceEngine
```

---

## 9.2 Active Record

Podrá existir como API de conveniencia.

Ejemplo:

```php
$user = User::find(1);

$user->name = 'Francisco';

$user->save();
```

Pero internamente:

```text
Model::save()
      │
      ▼
EntityManager
      │
      ▼
UnitOfWork
      │
      ▼
PersistenceEngine
```

Active Record no deberá implementar un segundo sistema de persistencia.

---

# 10. Query Engine

El Query Engine será uno de los principales componentes diferenciadores de VoltStack Database.

Estará compuesto por:

```text
Query Builder
Query Model
Query AST
Expression System
Semantic Analyzer
Semantic Graph
Optimizer
Planner
Compiler
Executor
```

El Query Engine deberá funcionar independientemente del ORM.

---

# 11. Query AST

Toda consulta estructurada deberá ser representada mediante un árbol de sintaxis abstracta.

Ejemplo:

```php
DB::table('users')
    ->where('active', true)
    ->where('age', '>', 18)
    ->orderBy('name')
    ->get();
```

Podrá representar internamente:

```text
SelectQuery
├── From
│   └── users
├── Predicate
│   ├── active = true
│   └── age > 18
└── OrderBy
    └── name ASC
```

El AST será independiente del SQL final.

---

# 12. Análisis semántico

Después de construir el AST, VoltStack podrá ejecutar una fase de análisis semántico.

Esta capa podrá resolver:

```text
tables
columns
aliases
relationships
types
expressions
joins
parameters
schema references
database capabilities
```

Ejemplo:

```text
AST
 ↓
Semantic Analyzer
 ↓
Semantic Graph
```

Esto permitirá detectar errores antes de enviar consultas al motor de base de datos.

---

# 13. Query Optimizer

VoltStack incluirá un optimizador a nivel de framework.

No intentará reemplazar el optimizer interno de:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

Su función será optimizar la representación de consultas antes de generar SQL.

Ejemplos:

```text
predicate simplification
duplicate condition elimination
relation batching
join normalization
subquery normalization
query deduplication
parameter normalization
pagination optimization
eager loading optimization
```

---

# 14. Query Planner

Después de optimizar una consulta podrá generarse un plan lógico.

```text
AST
 ↓
Semantic Graph
 ↓
Optimizer
 ↓
Logical Plan
 ↓
Physical Plan
 ↓
Execution Plan
```

El planner podrá tomar decisiones basadas en capacidades del motor.

Ejemplo:

```text
PostgreSQL supports RETURNING
MySQL has different RETURNING behavior
SQLite has different ALTER TABLE capabilities
```

El plan podrá adaptarse antes de llegar al compilador.

---

# 15. SQL Compiler

VoltStack utilizará compiladores específicos por plataforma.

Ejemplo:

```text
Query Plan
      │
      ├── MySqlCompiler
      ├── MariaDbCompiler
      ├── PostgreSqlCompiler
      └── SqliteCompiler
```

El compilador será responsable exclusivamente de producir:

```text
SQL
Bindings
Parameter metadata
Execution metadata
```

---

# 16. Driver System

El sistema deberá separar claramente:

```text
Driver
Dialect
Platform
Connection
```

## Driver

Gestiona la comunicación técnica.

Ejemplo:

```text
PDO
native extension
specialized runtime driver
```

## Dialect

Describe sintaxis SQL.

## Platform

Describe capacidades del motor.

## Connection

Representa una conexión concreta administrada por VoltStack.

---

# 17. Motores soportados inicialmente

La primera versión deberá soportar oficialmente:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

La arquitectura deberá permitir incorporar posteriormente otros motores mediante extensiones.

Ejemplos posibles:

```text
SQL Server
Oracle
CockroachDB
TiDB
YugabyteDB
```

Estos motores no formarán necesariamente parte del paquete base.

---

# 18. Sistema de capacidades

Cada plataforma deberá declarar sus capacidades.

Ejemplo conceptual:

```text
DatabaseCapabilities
├── supportsReturning
├── supportsCTE
├── supportsRecursiveCTE
├── supportsWindowFunctions
├── supportsJson
├── supportsSavepoints
├── supportsGeneratedColumns
├── supportsUpsert
└── supportsAdvisoryLocks
```

El resto del sistema deberá consultar capacidades en lugar de detectar motores mediante condicionales repetidos.

Evitar:

```php
if ($driver === 'pgsql') {
    // ...
}
```

Preferir:

```php
if ($platform->supportsReturning()) {
    // ...
}
```

---

# 19. Schema System

VoltStack incluirá un subsistema independiente para modelar el esquema de la base de datos.

Componentes conceptuales:

```text
Schema
Table
Column
Index
ForeignKey
Constraint
Sequence
View
```

Deberá existir representación estructurada del schema.

```text
Schema Model
     ↓
Schema AST
     ↓
Schema Compiler
```

---

# 20. Schema Introspection

El sistema podrá inspeccionar bases de datos existentes.

Ejemplo:

```text
Database
   ↓
Schema Introspector
   ↓
Schema Metadata
```

Podrá obtener:

```text
tables
columns
types
indexes
foreign keys
constraints
sequences
views
database metadata
```

Esta información será utilizada por:

```text
migrations
schema diff
ORM
validation
CLI
developer tools
diagnostics
```

---

# 21. Migraciones

VoltStack proporcionará un sistema de migraciones propio.

La experiencia podrá ser similar a:

```php
Schema::create('users', function (Table $table) {
    $table->id();
    $table->string('email')->unique();
    $table->timestamps();
});
```

Internamente:

```text
Migration
    ↓
Schema Model
    ↓
Schema AST
    ↓
Migration Planner
    ↓
Schema Compiler
    ↓
SQL
```

---

# 22. ORM

El ORM deberá ser un subsistema independiente construido encima del Query Engine.

Arquitectura general:

```text
Entity
   │
   ▼
EntityManager
   │
   ├── Metadata
   ├── IdentityMap
   ├── UnitOfWork
   └── Repository
          │
          ▼
PersistenceEngine
          │
          ▼
QueryEngine
```

---

# 23. Entity Manager

El Entity Manager será responsable de coordinar el contexto de persistencia.

Entre sus responsabilidades:

```text
find
persist
remove
refresh
detach
clear
flush
repository resolution
identity management
```

No deberá generar SQL.

---

# 24. Unit of Work

El Unit of Work será responsable del seguimiento de cambios.

Podrá clasificar entidades como:

```text
NEW
MANAGED
DIRTY
REMOVED
DETACHED
```

Ejemplo:

```text
UnitOfWork

NewEntities
DirtyEntities
RemovedEntities
RelationshipChanges
```

---

# 25. Identity Map

Durante un contexto de persistencia:

```php
$userA = $users->find(1);
$userB = $users->find(1);
```

se buscará mantener:

```php
$userA === $userB;
```

si ambas operaciones pertenecen al mismo contexto.

---

# 26. Change Tracking

VoltStack podrá soportar diferentes estrategias de seguimiento.

Ejemplos:

```text
snapshot
explicit
notification
property tracking
```

La estrategia exacta será definida en la documentación correspondiente.

---

# 27. Persistence Engine

El Persistence Engine será responsable de convertir cambios del dominio en operaciones persistibles.

Ejemplo:

```text
UnitOfWork
    ↓
Persistence Planner
    ↓
InsertOperation
UpdateOperation
DeleteOperation
RelationOperation
    ↓
Query Engine
```

---

# 28. Relationships

El ORM deberá soportar como mínimo:

```text
OneToOne
OneToMany
ManyToOne
ManyToMany
Polymorphic
```

También deberá soportar diferentes estrategias de carga:

```text
lazy
eager
batch
explicit
```

---

# 29. Prevención de N+1

VoltStack deberá incluir herramientas de detección de N+1.

Ejemplo:

```text
Potential N+1 detected

Relation:
User.posts

Parents:
100

Additional queries:
100
```

Podrá integrarse con Telemetry y herramientas de desarrollo.

---

# 30. Hydration

El ORM deberá separar explícitamente la ejecución de consultas y la construcción de objetos.

```text
Executor
   ↓
Result
   ↓
HydrationPlan
   ↓
Hydrator
   ↓
Entity
```

Esto permitirá diferentes estrategias:

```text
entity hydration
scalar hydration
array hydration
partial hydration
DTO hydration
stream hydration
```

---

# 31. Transactions

VoltStack deberá ofrecer una API sencilla:

```php
Database::transaction(function () {
    // ...
});
```

pero también APIs avanzadas.

Ejemplo:

```php
$transaction = $database->beginTransaction(
    isolation: IsolationLevel::Serializable
);
```

El sistema deberá considerar:

```text
isolation levels
nested transactions
savepoints
deadlock retries
timeouts
read-only transactions
transaction events
```

---

# 32. Connection Pooling

La arquitectura deberá estar preparada para reutilización eficiente de conexiones.

Especialmente en runtimes persistentes:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

El pool no deberá permitir que sobreviva accidentalmente estado perteneciente a una petición anterior.

---

# 33. Runtimes persistentes

VoltStack Database será diseñado desde el inicio para funcionar correctamente en procesos PHP persistentes.

El runtime inicial recomendado será:

```text
FrankenPHP
```

Posteriormente podrán existir adaptadores para:

```text
RoadRunner
OpenSwoole
```

La arquitectura deberá diferenciar:

```text
Process lifetime
Worker lifetime
Request lifetime
Transaction lifetime
Persistence context lifetime
Connection lifetime
```

---

# 34. Request Scope

Cada request deberá poseer su propio contexto de base de datos cuando corresponda.

Ejemplo:

```text
Worker
 │
 ├── Request A
 │     └── DatabaseContext A
 │
 ├── RESET
 │
 └── Request B
       └── DatabaseContext B
```

---

# 35. State Reset

Al finalizar una petición deberán poder limpiarse:

```text
IdentityMap
UnitOfWork
EntityManager state
Transaction state
Tenant context
Temporary query context
Hydration context
Request-scoped metadata
```

Esto será obligatorio para evitar contaminación entre requests.

---

# 36. Read/Write Splitting

El sistema podrá soportar múltiples conexiones según operación.

Ejemplo:

```text
SELECT
   ↓
Read Replica

INSERT
UPDATE
DELETE
   ↓
Primary
```

También podrá existir sticky routing después de una escritura.

---

# 37. Replicas

El sistema deberá estar preparado para:

```text
multiple replicas
load balancing
replica health
failover
replica lag awareness
read preference
```

Estas capacidades deberán permanecer opcionales.

---

# 38. Distribución y sharding

VoltStack Database podrá disponer de extensiones para:

```text
sharding
partition routing
distributed databases
database-per-region
database-per-customer
```

Estas capacidades no deberán complicar el Query Engine básico.

---

# 39. Multitenancy

Multitenancy no será una responsabilidad obligatoria del Database Core.

Será tratado como integración.

Conceptualmente:

```text
Quantum/Multitenancy
        │
        ▼
Database Tenant Adapter
        │
        ▼
Quantum/Database
```

Database deberá proporcionar los puntos de extensión necesarios para:

```text
tenant connection resolution
schema isolation
database isolation
tenant query context
tenant migration integration
```

---

# 40. Cache

Database podrá integrarse con:

```text
VoltStack/Quantum/Cache
```

pero no dependerá obligatoriamente de este paquete.

Podrán existir:

```text
query cache
result cache
metadata cache
entity cache
compiled query cache
```

---

# 41. Eventos

Database deberá publicar eventos mediante contratos desacoplados.

Ejemplos:

```text
QueryStarted
QueryCompleted
QueryFailed

ConnectionOpened
ConnectionClosed
ConnectionFailed

TransactionStarted
TransactionCommitted
TransactionRolledBack

EntityPersisted
EntityUpdated
EntityRemoved
```

La integración completa podrá realizarse con:

```text
VoltStack/Quantum/EventSystem
```

---

# 42. Telemetría

Database proporcionará puntos de observabilidad compatibles con:

```text
VoltStack/Quantum/Telemetry
```

Podrá generar:

```text
metrics
logs
traces
profiles
diagnostic events
```

Ejemplos:

```text
query duration
connection duration
transaction duration
rows returned
rows affected
slow queries
N+1 queries
connection failures
retry count
pool utilization
```

---

# 43. Seguridad

El sistema deberá considerar seguridad desde la arquitectura.

Principales áreas:

```text
prepared statements
parameter binding
SQL injection protection
identifier validation
credential protection
connection encryption
sensitive data masking
query audit
least privilege guidance
```

Raw SQL seguirá siendo posible, pero será considerado un escape hatch explícito.

---

# 44. Resiliencia

Database deberá proporcionar mecanismos para manejar fallos esperables.

Ejemplos:

```text
connection failure
temporary network failure
deadlock
timeout
replica failure
primary failure
resource exhaustion
```

Podrán existir políticas de:

```text
retry
backoff
failover
circuit breaking
```

sin esconder fallos permanentes.

---

# 45. Performance

El sistema deberá estar diseñado para minimizar:

```text
allocations
metadata parsing
reflection
query recompilation
connection overhead
unnecessary hydration
duplicate queries
N+1 operations
```

Se favorecerá la compilación y caché de metadatos cuando sea posible.

---

# 46. Metadata Compilation

Durante producción podrá precompilarse metadata.

Ejemplo:

```text
Attributes
   ↓
Metadata Scanner
   ↓
Metadata Compiler
   ↓
Compiled Metadata
```

Durante runtime:

```text
Compiled Metadata
       ↓
ORM
```

reduciendo reflexión repetitiva.

---

# 47. Testing

Database deberá incluir infraestructura para:

```text
unit tests
integration tests
driver compatibility tests
migration tests
ORM tests
query tests
transaction tests
performance tests
```

También deberá proporcionar herramientas de testing al desarrollador de aplicaciones.

Ejemplo:

```php
$this->assertDatabaseHas('users', [
    'email' => 'test@example.com',
]);
```

---

# 48. Factories y Seeders

VoltStack deberá proporcionar una experiencia sencilla para generación de datos.

Ejemplo:

```php
UserFactory::new()
    ->count(100)
    ->create();
```

Y:

```php
DatabaseSeeder::run();
```

Estos sistemas deberán trabajar sobre APIs públicas, no internals del ORM.

---

# 49. Developer Experience

El sistema deberá favorecer:

```text
clear errors
predictable APIs
good autocomplete
strong typing
IDE discoverability
minimal configuration
sensible defaults
```

El desarrollador no deberá comprender el AST o Planner para realizar operaciones comunes.

---

# 50. Escape Hatches

Una arquitectura avanzada no deberá impedir acceso directo al motor.

VoltStack deberá permitir:

```php
DB::select(...);

DB::statement(...);

$connection->execute(...);
```

cuando sea necesario.

Sin embargo, estos accesos deberán permanecer claramente identificados como APIs de bajo nivel.

---

# 51. Extensibilidad

Database deberá permitir extensiones para:

```text
drivers
platforms
dialects
query nodes
query functions
types
compilers
hydrators
metadata providers
ORM behaviors
schema types
migration operations
```

Las extensiones deberán registrarse mediante contratos públicos y registries definidos.

---

# 52. Dependencias

Database deberá reducir al mínimo las dependencias obligatorias sobre otros subsistemas VoltStack.

Dependencias conceptuales básicas:

```text
Platform
Container
Config
Contracts
Support
```

Integraciones opcionales:

```text
Cache
Telemetry
EventSystem
Multitenancy
Validation
Authentication
Authorization
Jobs
RuntimeManagerServer
```

---

# 53. No objetivos

Database no pretende convertirse en:

```text
a database server
a replacement for PostgreSQL optimizer
a distributed transaction coordinator by default
a mandatory ORM
a mandatory Active Record implementation
a mandatory multitenancy solution
```

Tampoco deberá implementar lógica perteneciente a otros módulos solo para evitar dependencias.

---

# 54. Arquitectura general

La arquitectura conceptual inicial será:

```text
                     Application
                         │
           ┌─────────────┴──────────────┐
           │                            │
      Active Record                Repository
           │                            │
           └─────────────┬──────────────┘
                         ▼
                    ORM Layer
                         │
              ┌──────────┼──────────┐
              │          │          │
          Metadata   IdentityMap  UnitOfWork
              │          │          │
              └──────────┼──────────┘
                         ▼
                 Persistence Engine
                         │
                         ▼
                    Query Engine
                         │
             ┌───────────┼────────────┐
             │           │            │
          Builder       AST       Semantic
             │           │            │
             └───────────┼────────────┘
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
                      Driver
                         │
                         ▼
                     Database
```

---

# 55. Sistemas transversales

Alrededor del núcleo existirán servicios transversales.

```text
                     Cache
                       │
                       │
Events ───────── Database ───────── Telemetry
                       │
                       │
                   Security
                       │
                       │
                    Runtime
```

Estos servicios no deberán romper los límites internos del Query Engine u ORM.

---

# 56. Principio de dependencia descendente

Las dependencias deberán fluir hacia capas inferiores.

Ejemplo permitido:

```text
ORM
 ↓
Query Engine
 ↓
Execution
 ↓
Connection
 ↓
Driver
```

Ejemplo prohibido:

```text
Driver
 ↓
ORM
```

También:

```text
Compiler
 ↓
EntityManager
```

estará prohibido.

---

# 57. Evitar dependencias circulares

El nuevo diseño deberá prestar especial atención a evitar ciclos.

Ejemplo prohibido:

```text
ORM
 ↓
Query
 ↓
Metadata
 ↓
ORM
```

Las abstracciones compartidas deberán colocarse en capas neutrales.

Ejemplo:

```text
Shared Contracts
       ▲
       │
 ┌─────┴─────┐
 ORM       Query
```

---

# 58. Diseño basado en contratos

Los componentes principales deberán depender de contratos.

Ejemplo:

```php
interface ConnectionInterface
{
}
```

```php
interface DriverInterface
{
}
```

```php
interface QueryCompilerInterface
{
}
```

```php
interface QueryExecutorInterface
{
}
```

```php
interface EntityManagerInterface
{
}
```

Las implementaciones concretas deberán poder sustituirse cuando sea razonable.

---

# 59. Modelo de capacidades opcionales

No todos los motores soportan las mismas características.

Por tanto, deberá evitarse asumir soporte universal.

Ejemplo:

```text
Feature
   ↓
Capability Resolution
   ↓
Supported?
   ├── yes → native strategy
   └── no  → fallback or explicit error
```

Esto aplicará a:

```text
RETURNING
CTE
recursive CTE
window functions
JSON
generated columns
partial indexes
sequences
savepoints
advisory locks
```

---

# 60. Compatibilidad

El sistema deberá mantener una política clara de compatibilidad.

La implementación deberá considerar:

```text
database version
driver version
PHP version
runtime capabilities
VoltStack version
```

Las características específicas de plataforma deberán estar documentadas explícitamente.

---

# 61. Filosofía de configuración

La configuración inicial deberá ser sencilla.

Ejemplo conceptual:

```php
return [

    'default' => 'primary',

    'connections' => [

        'primary' => [
            'driver' => 'pgsql',
            'host' => env('DB_HOST'),
            'database' => env('DB_DATABASE'),
            'username' => env('DB_USERNAME'),
            'password' => env('DB_PASSWORD'),
        ],

    ],

];
```

Las configuraciones avanzadas podrán añadirse progresivamente.

---

# 62. Secure Defaults

Las configuraciones predeterminadas deberán favorecer:

```text
prepared statements
safe parameter binding
secure credentials
transaction safety
strict error handling
production metadata cache
runtime state isolation
```

El comportamiento inseguro deberá requerir configuración explícita.

---

# 63. Zero Configuration

En aplicaciones sencillas deberá ser posible utilizar Database con una configuración mínima.

La complejidad arquitectónica interna no deberá trasladarse innecesariamente al desarrollador.

---

# 64. Escalabilidad conceptual

La misma arquitectura deberá ser válida para:

```text
small applications
traditional web applications
SPA applications
SaaS
enterprise systems
long-running workers
high concurrency systems
distributed infrastructures
```

No todas estas capacidades deberán instalarse o activarse por defecto.

---

# 65. Relación con VoltStack

Database será un componente fundamental del ecosistema VoltStack, pero seguirá principios de modularidad.

Conceptualmente:

```text
VoltStack Platform
       │
       ▼
Quantum/Database
       │
       ├── Query
       ├── ORM
       ├── Schema
       ├── Migration
       └── Transaction
```

Otros módulos podrán integrarse mediante contratos.

---

# 66. Objetivo de la documentación

La documentación de `VoltStack/Quantum/Database` deberá definir:

```text
architecture
contracts
responsibilities
component boundaries
data flows
state ownership
extension points
failure behavior
performance expectations
security expectations
testing requirements
integration rules
```

Los documentos no deberán limitarse a describir APIs.

Deberán funcionar como especificación arquitectónica para la implementación.

---

# 67. Orden de diseño

El sistema deberá diseñarse siguiendo aproximadamente este orden:

```text
Architecture
    ↓
Contracts
    ↓
Connection
    ↓
Driver
    ↓
Platform
    ↓
Query Model
    ↓
AST
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
Schema
    ↓
Migrations
    ↓
ORM
    ↓
UnitOfWork
    ↓
Persistence
    ↓
Relationships
    ↓
Transactions
    ↓
Advanced capabilities
    ↓
Integration
```

Esto evita implementar primero APIs de alto nivel sin tener definidos sus fundamentos.

---

# 68. Principio de estabilidad de capas

Las capas inferiores deberán cambiar con menor frecuencia que las superiores.

Conceptualmente:

```text
Developer API
      │
      │ high evolution
      ▼
ORM
Query Builder
      │
      ▼
Query Model
AST
      │
      ▼
Execution
Connection
Driver
      │
      │ high stability
      ▼
Database
```

Las APIs públicas deberán poder evolucionar sin requerir rediseñar los drivers.

---

# 69. Objetivo final

El objetivo de `VoltStack/Quantum/Database` será ofrecer una plataforma de datos capaz de proporcionar simultáneamente:

```text
Laravel-like developer experience
Doctrine-like architectural separation
VoltStack Query AST
VoltStack Semantic Engine
VoltStack Query Planner
VoltStack Compiler Pipeline
VoltStack Persistence Engine
VoltStack Runtime Safety
VoltStack Telemetry Integration
VoltStack extensibility
```

La complejidad deberá permanecer principalmente dentro del framework.

Para el desarrollador común:

```php
User::where('active', true)->get();
```

deberá seguir siendo suficiente.

Para arquitecturas complejas:

```php
$entityManager->transactional(function (
    EntityManagerInterface $em
) use ($command) {

    // Domain operation

});
```

también deberá existir una infraestructura sólida.

---

# 70. Principio rector

El principio rector del nuevo Database será:

> VoltStack Database deberá ser sencillo de utilizar, difícil de utilizar incorrectamente y arquitectónicamente preparado para crecer sin convertirse en un subsistema monolítico.

La arquitectura deberá priorizar:

```text
separation
predictability
performance
extensibility
security
runtime safety
developer experience
```

sobre soluciones rápidas que introduzcan dependencias circulares o responsabilidades ambiguas.

---

# 71. Estado del documento

Este documento establece el contexto general y las decisiones fundacionales del nuevo sistema Database.

Las especificaciones concretas serán desarrolladas en los siguientes documentos.

Próximo documento:

```text
01_DATABASE_ARCHITECTURE.md
```

---

# 72. Resumen arquitectónico

```text
VoltStack/Quantum/Database
│
├── Driver
├── Connection
├── Platform
├── Query
│   ├── Builder
│   ├── Model
│   ├── AST
│   ├── Semantic
│   ├── Optimizer
│   ├── Planner
│   ├── Compiler
│   └── Executor
│
├── Schema
├── Migration
│
├── ORM
│   ├── EntityManager
│   ├── Repository
│   ├── Metadata
│   ├── IdentityMap
│   ├── UnitOfWork
│   ├── Hydration
│   ├── Relationships
│   └── Persistence
│
├── Transaction
├── Cache Integration
├── Event Integration
├── Telemetry Integration
├── Runtime Integration
├── Security
├── Resilience
├── Testing
└── Extensions
```

Este mapa será refinado y formalizado en:

```text
01_DATABASE_ARCHITECTURE.md
```