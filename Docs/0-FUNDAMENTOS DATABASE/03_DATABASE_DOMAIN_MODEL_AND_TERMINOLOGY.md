# 03_DATABASE_DOMAIN_MODEL_AND_TERMINOLOGY.md

# VoltStack Quantum Database
## Domain Model and Terminology

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 03 — Database Domain Model and Terminology  
**Estado:** Architecture Specification  
**Nivel:** Core Architecture  
**Versión:** 1.0

---

# 1. Propósito

Este documento define el modelo conceptual, vocabulario oficial, entidades arquitectónicas, límites y relaciones fundamentales del sistema de base de datos de VoltStack.

Su propósito es establecer un lenguaje común para todos los componentes posteriores de:

```text
VoltStack/Quantum/Database
```

incluyendo:

- Connection Management
- Query System
- Query AST
- Semantic Analysis
- Query Optimization
- Query Planning
- SQL Compilation
- Schema Management
- Migrations
- Transactions
- ORM
- Metadata
- Hydration
- Identity Map
- Unit of Work
- Repositories
- Relationships
- Persistence
- Caching
- Observability
- Multi-Tenancy
- Distributed Data
- Administration

Este documento es normativo.

Cuando otros documentos de `Quantum/Database` utilicen los términos definidos aquí, deberán conservar su significado salvo que indiquen explícitamente una especialización.

---

# 2. Objetivo arquitectónico

VoltStack Database no debe considerarse simplemente un ORM ni un Query Builder.

El dominio completo se modela como una plataforma de acceso, interpretación, planificación, ejecución y persistencia de datos.

Conceptualmente:

```text
Application
    │
    ▼
Database API
    │
    ├── Query API
    ├── ORM API
    ├── Repository API
    ├── Schema API
    └── Transaction API
    │
    ▼
Database Domain
    │
    ├── Connection
    ├── Query
    ├── Schema
    ├── Transaction
    ├── Persistence
    └── Metadata
    │
    ▼
Database Engine
    │
    ├── AST
    ├── Semantic Analysis
    ├── Optimization
    ├── Planning
    ├── Compilation
    └── Execution
    │
    ▼
Database Driver
    │
    ▼
Database Server
```

Los cuatro motores inicialmente soportados son:

```text
MySQL
PostgreSQL
MariaDB
SQLite
```

La arquitectura no deberá impedir la incorporación futura de nuevos dialectos o proveedores.

---

# 3. Principio fundamental del dominio

La arquitectura separa tres conceptos que frecuentemente aparecen mezclados en frameworks tradicionales:

```text
Database
Data Model
Persistence Model
```

No son equivalentes.

## 3.1 Database

Representa el sistema físico o lógico donde se almacenan los datos.

Ejemplos:

```text
MySQL database
PostgreSQL database
SQLite database
```

## 3.2 Data Model

Representa la estructura lógica de los datos.

Por ejemplo:

```text
User
 ├── id
 ├── name
 ├── email
 └── created_at
```

## 3.3 Persistence Model

Describe cómo el framework traduce objetos y operaciones del dominio hacia almacenamiento persistente.

Ejemplo:

```text
User Entity
     │
     ▼
ORM Metadata
     │
     ▼
Persistence Engine
     │
     ▼
users table
```

Esta separación permite que VoltStack soporte tanto modelos simples como arquitecturas empresariales complejas.

---

# 4. Capas conceptuales

El dominio Database se divide conceptualmente en seis capas.

```text
┌──────────────────────────────────────────────┐
│              Application Layer               │
├──────────────────────────────────────────────┤
│                 Data API                     │
├──────────────────────────────────────────────┤
│            Persistence Domain                │
├──────────────────────────────────────────────┤
│               Query Engine                   │
├──────────────────────────────────────────────┤
│         Connection / Driver Layer            │
├──────────────────────────────────────────────┤
│              Database Server                 │
└──────────────────────────────────────────────┘
```

Estas capas representan responsabilidades, no necesariamente namespaces PHP exactos.

---

# 5. Application Layer

La Application Layer contiene código perteneciente a la aplicación del desarrollador.

Ejemplos:

```php
$user = User::find(10);
```

o:

```php
$user = $users->findById(10);
```

o:

```php
$users = $database
    ->table('users')
    ->where('active', true)
    ->get();
```

VoltStack permitirá diferentes estilos de interacción sin crear motores de persistencia independientes.

---

# 6. Database API

`Database API` es el término genérico para las interfaces públicas mediante las cuales una aplicación interactúa con Quantum Database.

Incluye:

```text
Query API
ORM API
Repository API
Transaction API
Schema API
Migration API
Administration API
```

Estas APIs deben converger sobre infraestructura común.

No deberán existir dos motores de consulta o persistencia independientes para proporcionar distintas sintaxis públicas.

---

# 7. Database Manager

`DatabaseManager` representa el punto central de coordinación de conexiones y configuraciones de bases de datos.

Responsabilidades conceptuales:

```text
DatabaseManager
 ├── resolve connection
 ├── connection lifecycle
 ├── driver resolution
 ├── connection pools
 ├── read/write topology
 ├── tenant-aware resolution
 └── database context
```

Ejemplo conceptual:

```php
$db = $databaseManager->connection('primary');
```

No debe confundirse con `EntityManager`.

---

# 8. Database Connection

Una `Connection` representa una conexión lógica con una base de datos.

No necesariamente equivale permanentemente a una conexión física TCP individual.

Puede representar:

```text
logical connection
pooled connection
persistent connection
lazy connection
read connection
write connection
transaction-bound connection
```

Conceptualmente:

```text
Connection
    │
    ▼
Driver
    │
    ▼
Native Client
    │
    ▼
Database Server
```

---

# 9. Physical Connection

Una `Physical Connection` representa el recurso concreto utilizado para comunicarse con el servidor.

Ejemplos:

```text
PDO connection
native PostgreSQL connection
SQLite handle
```

VoltStack debe distinguir entre:

```text
Logical Connection

y

Physical Connection
```

para permitir pooling, reconexión y runtimes persistentes.

---

# 10. Connection Configuration

`ConnectionConfiguration` describe los parámetros necesarios para construir o resolver una conexión.

Puede contener:

```text
driver
host
port
database
username
credentials
charset
timezone
options
SSL configuration
read replicas
write endpoint
pool configuration
```

Las credenciales no deberán propagarse innecesariamente por otros objetos del dominio.

---

# 11. Driver

Un `Driver` implementa la comunicación específica con una tecnología de base de datos.

Ejemplos:

```text
MySQLDriver
PostgreSQLDriver
MariaDBDriver
SQLiteDriver
```

Responsabilidades:

```text
connection creation
native parameter binding
transaction primitives
capability detection
driver errors
low-level execution
```

El Driver no es responsable de implementar el ORM.

---

# 12. Dialect

`Dialect` representa las características sintácticas y semánticas específicas del lenguaje SQL soportado por un motor.

Ejemplos:

```text
MySQLDialect
PostgreSQLDialect
MariaDBDialect
SQLiteDialect
```

Driver y Dialect son conceptos diferentes.

```text
Driver
    = cómo comunicarse con la base de datos

Dialect
    = cómo expresar una operación para esa base de datos
```

Esta distinción es obligatoria.

---

# 13. Database Platform

`DatabasePlatform` representa las capacidades conocidas de una plataforma de base de datos.

Ejemplo:

```text
PostgreSQLPlatform
```

puede describir:

```text
RETURNING support
JSON capabilities
generated columns
index capabilities
locking features
transaction isolation
CTE support
window functions
schema namespaces
```

Por tanto:

```text
Driver
Dialect
Platform
```

son tres conceptos relacionados pero diferentes.

---

# 14. Query

Una `Query` representa una solicitud lógica de lectura o modificación de datos.

Ejemplos:

```text
SELECT
INSERT
UPDATE
DELETE
UPSERT
```

Una Query no deberá definirse internamente como una cadena SQL.

La cadena SQL es únicamente una posible representación compilada.

Conceptualmente:

```text
Query
    │
    ▼
Query AST
    │
    ▼
Semantic Query
    │
    ▼
Query Plan
    │
    ▼
Compiled Query
    │
    ▼
SQL
```

---

# 15. Query Builder

`QueryBuilder` es una API utilizada para construir una Query de forma programática.

Ejemplo:

```php
$db->table('users')
    ->where('status', 'active')
    ->orderBy('name')
    ->limit(100);
```

Internamente, el Query Builder debe producir estructuras del modelo de consulta.

No deberá depender fundamentalmente de concatenación incremental de SQL.

Modelo:

```text
Query Builder
     │
     ▼
Query AST
```

---

# 16. Query AST

`Query AST` significa:

```text
Query Abstract Syntax Tree
```

Es la representación estructurada de una consulta.

Ejemplo:

```text
SelectQuery
 ├── From
 │    └── users
 ├── Projection
 │    ├── id
 │    └── name
 ├── Predicate
 │    └── status = "active"
 ├── OrderBy
 │    └── name ASC
 └── Limit
      └── 100
```

El AST es independiente de un dialecto SQL concreto.

---

# 17. Query Node

Un `QueryNode` es cualquier elemento estructural perteneciente al Query AST.

Ejemplos:

```text
SelectNode
TableNode
ColumnNode
PredicateNode
JoinNode
OrderNode
LimitNode
ParameterNode
ExpressionNode
```

Los Query Nodes deberán ser estructuralmente analizables y transformables.

---

# 18. Expression

Una `Expression` representa una operación que produce o compara valores dentro de una Query.

Ejemplos:

```text
column = value
price > 100
COUNT(id)
LOWER(email)
a + b
EXISTS(...)
```

Las expresiones también deberán representarse estructuralmente.

---

# 19. Predicate

Un `Predicate` es una expresión cuyo resultado lógico determina inclusión, exclusión o condición.

Ejemplos:

```text
status = 'active'

age >= 18

deleted_at IS NULL
```

Puede componerse mediante:

```text
AND
OR
NOT
```

---

# 20. Parameter

Un `Parameter` representa un valor externo incorporado de forma segura a una Query.

Ejemplo:

```text
status = :status
```

Los Parameters son distintos de los Literals.

```text
Parameter
    → valor proporcionado durante ejecución

Literal
    → valor representado explícitamente por el AST
```

VoltStack deberá utilizar parameter binding por defecto cuando sea técnicamente aplicable.

---

# 21. Semantic Query Model

El `Semantic Query Model` representa una Query después de resolver su significado.

El AST puede indicar:

```text
users.email
```

El análisis semántico puede resolver:

```text
TableMetadata(users)
ColumnMetadata(email)
Type(string)
Nullable(false)
```

Pipeline:

```text
Query AST
    │
    ▼
Semantic Analyzer
    │
    ▼
Semantic Query Model
```

---

# 22. Semantic Graph

El `Semantic Graph` representa las relaciones semánticas detectadas entre los elementos involucrados en una operación de datos.

Puede modelar:

```text
entities
tables
columns
relationships
joins
dependencies
constraints
types
aliases
scopes
```

Ejemplo:

```text
User
 │
 ├── belongsTo ──► Organization
 │
 └── hasMany ────► Order
                     │
                     └── belongsTo ──► Product
```

El Semantic Graph proporciona información al optimizador y al planner.

---

# 23. Query Optimizer

`QueryOptimizer` transforma una representación lógica de consulta buscando una ejecución más eficiente sin modificar su semántica observable.

Puede realizar optimizaciones como:

```text
predicate simplification
duplicate predicate elimination
relation batching
query deduplication
projection reduction
eager-loading optimization
IN batching
pagination optimization
```

No intenta reemplazar al optimizador interno de MySQL, PostgreSQL u otros motores.

---

# 24. Query Planner

`QueryPlanner` convierte una consulta lógica validada en un plan ejecutable por la infraestructura de VoltStack.

Ejemplo:

```text
Semantic Query
       │
       ▼
Optimizer
       │
       ▼
Query Planner
       │
       ▼
Execution Plan
```

Puede decidir:

```text
connection
read/write target
relation loading strategy
query splitting
batching
execution ordering
compiler
```

---

# 25. Execution Plan

Un `ExecutionPlan` describe cómo VoltStack ejecutará una operación.

Puede contener:

```text
queries
dependencies
connection targets
transaction requirements
batch groups
hydration strategy
result transformations
```

Un plan puede contener una sola consulta o múltiples operaciones coordinadas.

---

# 26. Query Compiler

`QueryCompiler` transforma una representación lógica o planificada en una consulta ejecutable para un Dialect determinado.

Ejemplo:

```text
Query AST
     │
     ▼
PostgreSQL Compiler
     │
     ▼
SELECT ...
```

o:

```text
Query AST
     │
     ▼
MySQL Compiler
     │
     ▼
SELECT ...
```

---

# 27. Compiled Query

Una `CompiledQuery` representa el resultado listo para ser enviado al Driver.

Debe contener al menos conceptualmente:

```text
statement
parameters
parameter types
execution metadata
```

Ejemplo:

```text
statement:
SELECT id, name
FROM users
WHERE status = ?

parameters:
["active"]
```

---

# 28. Query Executor

`QueryExecutor` ejecuta una `CompiledQuery` mediante una Connection.

Conceptualmente:

```text
CompiledQuery
     │
     ▼
QueryExecutor
     │
     ▼
Connection
     │
     ▼
Driver
```

También participa en:

```text
telemetry
error translation
timing
result acquisition
resource cleanup
```

---

# 29. Result

`Result` representa la respuesta obtenida de una operación de base de datos.

Puede contener:

```text
rows
affected rows
generated identifiers
cursor
metadata
execution information
```

Result no equivale necesariamente a una Collection.

---

# 30. Result Set

`ResultSet` representa específicamente un conjunto tabular de resultados.

Ejemplo:

```text
ResultSet
 ├── Row
 ├── Row
 └── Row
```

Puede ser:

```text
buffered
streamed
cursor-based
lazy
```

---

# 31. Row

Una `Row` representa una fila lógica obtenida de la base de datos.

Antes de la hidratación ORM puede representarse como:

```php
[
    'id' => 10,
    'name' => 'Alice',
]
```

La representación concreta deberá abstraerse cuando sea necesario.

---

# 32. Schema

`Schema` representa la estructura lógica de una base de datos.

Puede contener:

```text
tables
columns
indexes
foreign keys
constraints
sequences
views
namespaces
```

---

# 33. Schema Metadata

`SchemaMetadata` representa información estructurada sobre un Schema existente o deseado.

Ejemplo:

```text
DatabaseMetadata
 └── TableMetadata
      ├── ColumnMetadata
      ├── IndexMetadata
      ├── ForeignKeyMetadata
      └── ConstraintMetadata
```

---

# 34. Schema Manager

`SchemaManager` permite consultar y manipular estructuras de base de datos mediante una API independiente del dialecto.

Ejemplo conceptual:

```php
$schema->table('users', function (Table $table) {
    $table->id();
    $table->string('name');
});
```

Esta definición deberá producir una representación estructurada antes de convertirse a SQL.

---

# 35. Schema AST

`Schema AST` representa operaciones de definición o modificación estructural.

Ejemplos:

```text
CreateTable
AddColumn
DropColumn
CreateIndex
AddForeignKey
RenameTable
```

Pipeline:

```text
Schema API
    │
    ▼
Schema AST
    │
    ▼
Schema Planner
    │
    ▼
Dialect Compiler
    │
    ▼
DDL
```

---

# 36. Migration

Una `Migration` representa una transición versionada del esquema o estado estructural de una aplicación.

Conceptualmente:

```text
Schema Version N
      │
      ▼
Migration
      │
      ▼
Schema Version N+1
```

Las migrations deberán utilizar el Schema System siempre que sea posible.

---

# 37. Transaction

Una `Transaction` representa una unidad atómica de operaciones coordinadas por la base de datos.

Propiedades clásicas:

```text
Atomicity
Consistency
Isolation
Durability
```

VoltStack deberá abstraer:

```text
begin
commit
rollback
savepoint
isolation level
read-only mode
retry
```

---

# 38. Transaction Context

`TransactionContext` representa el estado lógico de una transacción activa dentro de VoltStack.

Puede contener:

```text
connection
transaction depth
savepoints
isolation level
retry state
read-only state
lifecycle metadata
```

Este contexto debe ser request-safe y runtime-safe.

---

# 39. Entity

Una `Entity` representa un objeto del dominio con identidad persistente.

Ejemplo:

```php
class User
{
    public UserId $id;
    public string $name;
}
```

Una Entity no es necesariamente equivalente a una fila.

Puede mapear:

```text
one table
multiple tables
embedded values
relationships
computed state
```

---

# 40. Entity Identity

`Entity Identity` representa aquello que permite determinar si dos representaciones hacen referencia a la misma entidad persistente.

Ejemplo:

```text
User#42
```

Conceptualmente:

```text
Entity Type + Identifier
```

---

# 41. Entity Metadata

`EntityMetadata` describe cómo una Entity participa en el sistema de persistencia.

Puede contener:

```text
entity class
identifier
fields
types
table mapping
relationships
inheritance
lifecycle hooks
generation strategy
embedded objects
```

---

# 42. Model

En VoltStack, `Model` será un término de API de conveniencia y no el término fundamental del motor ORM.

Una clase Active Record puede denominarse Model:

```php
class User extends Model
{
}
```

Sin embargo, internamente el ORM trabajará fundamentalmente con:

```text
Entity
EntityMetadata
EntityManager
UnitOfWork
IdentityMap
```

Esto evita que la arquitectura interna dependa del patrón Active Record.

---

# 43. Active Record

`Active Record` es un patrón donde un objeto de dominio expone directamente operaciones de persistencia.

Ejemplo:

```php
$user = User::find(10);

$user->name = 'Alice';

$user->save();
```

VoltStack puede ofrecer esta API como capa de conveniencia.

Sin embargo:

```text
Active Record API
        │
        ▼
Shared ORM Engine
```

No deberá existir un segundo motor ORM dedicado exclusivamente a Active Record.

---

# 44. Data Mapper

`Data Mapper` separa los objetos del dominio de las operaciones responsables de persistirlos.

Ejemplo:

```php
$user = $users->find(10);

$user->rename('Alice');

$entityManager->flush();
```

VoltStack utilizará principios Data Mapper dentro de su infraestructura ORM.

---

# 45. Entity Manager

`EntityManager` coordina el ciclo de persistencia de Entities.

Responsabilidades:

```text
find
persist
remove
flush
clear
refresh
detach
metadata access
repository resolution
UnitOfWork coordination
IdentityMap coordination
```

Conceptualmente:

```text
EntityManager
 ├── Metadata
 ├── IdentityMap
 ├── UnitOfWork
 ├── Repository
 └── Persistence Engine
```

---

# 46. Identity Map

`IdentityMap` garantiza que una identidad persistente corresponda a una instancia administrada determinada dentro del contexto ORM.

Ejemplo:

```php
$a = $users->find(42);
$b = $users->find(42);
```

Idealmente:

```text
$a === $b
```

dentro del mismo contexto de persistencia.

---

# 47. Unit of Work

`UnitOfWork` registra y coordina cambios pendientes sobre Entities administradas.

Puede mantener estados como:

```text
NEW
MANAGED
DIRTY
REMOVED
DETACHED
```

Conceptualmente:

```text
Entities
   │
   ▼
UnitOfWork
   │
   ├── Inserts
   ├── Updates
   ├── Deletes
   └── Relationship Changes
   │
   ▼
Flush Plan
```

---

# 48. Change Set

Un `ChangeSet` representa las modificaciones detectadas sobre una Entity.

Ejemplo:

```text
User#42

email:
    old = old@example.com
    new = new@example.com

status:
    old = pending
    new = active
```

El UnitOfWork utiliza ChangeSets para planificar persistencia.

---

# 49. Flush

`Flush` es la operación mediante la cual los cambios pendientes registrados en el contexto ORM son convertidos en operaciones de persistencia.

Pipeline conceptual:

```text
UnitOfWork
    │
    ▼
Change Detection
    │
    ▼
Change Sets
    │
    ▼
Persistence Planner
    │
    ▼
Execution Plan
    │
    ▼
Database
```

`flush()` no debe interpretarse simplemente como ejecutar varios `UPDATE`.

---

# 50. Persistence Engine

`PersistenceEngine` transforma operaciones del dominio ORM en operaciones de almacenamiento.

Responsabilidades:

```text
insert entities
update entities
delete entities
relationship persistence
identifier handling
dependency ordering
```

Debe utilizar la infraestructura común de Query y Execution.

---

# 51. Persistence Plan

Un `PersistencePlan` representa el orden de operaciones necesarias para sincronizar el estado administrado con la base de datos.

Ejemplo:

```text
Insert Organization
        │
        ▼
Insert User
        │
        ▼
Insert Membership
        │
        ▼
Update Audit Reference
```

Esto permite respetar dependencias y constraints.

---

# 52. Repository

Un `Repository` proporciona acceso orientado al dominio a una colección lógica de Entities.

Ejemplo:

```php
$users->findByEmail($email);
```

No debe confundirse con QueryBuilder.

Un Repository puede utilizar QueryBuilder internamente.

```text
Repository
    │
    ▼
ORM Query Layer
    │
    ▼
Query Engine
```

---

# 53. Relationship

Una `Relationship` representa una asociación persistente entre entidades.

Tipos conceptuales:

```text
OneToOne
OneToMany
ManyToOne
ManyToMany
```

El sistema debe distinguir:

```text
domain relationship
mapping relationship
database foreign key
```

No siempre son equivalentes.

---

# 54. Relationship Metadata

`RelationshipMetadata` describe cómo debe interpretarse y persistirse una Relationship.

Puede incluir:

```text
source entity
target entity
cardinality
ownership
join columns
join table
cascade rules
fetch strategy
orphan handling
```

---

# 55. Relation Loading

`Relation Loading` representa el proceso mediante el cual VoltStack obtiene entidades relacionadas.

Estrategias posibles:

```text
eager
lazy
explicit
batched
joined
subselect
```

La estrategia elegida podrá ser modificada por el Relation Load Planner.

---

# 56. Relation Load Planner

`RelationLoadPlanner` determina cómo cargar relaciones de forma eficiente.

Puede prevenir problemas como:

```text
N+1 queries
excessive joins
duplicated relation queries
oversized IN clauses
```

Ejemplo:

```text
100 Users
   │
   └── Organization

naive:
101 queries

planned:
2 queries
```

---

# 57. Hydration

`Hydration` es el proceso de transformar resultados provenientes de la base de datos en representaciones de mayor nivel.

Ejemplo:

```text
Database Row
     │
     ▼
Hydrator
     │
     ▼
Entity
```

También pueden existir hidratadores para:

```text
arrays
DTOs
scalar values
projections
```

---

# 58. Hydrator

Un `Hydrator` implementa una estrategia concreta de Hydration.

Debe cooperar con:

```text
EntityMetadata
IdentityMap
Type System
Relationship System
```

cuando produzca Entities administradas.

---

# 59. Dehydration

`Dehydration` representa la conversión del estado de una Entity o Value Object hacia valores adecuados para persistencia.

No implica necesariamente serialización textual.

Ejemplo:

```text
Money Object
     │
     ▼
Type Conversion
     │
     ├── amount
     └── currency
```

---

# 60. Value Object

Un `Value Object` representa un valor del dominio definido por sus atributos y no por identidad persistente independiente.

Ejemplos:

```text
Money
Address
DateRange
Coordinates
```

Puede persistirse mediante:

```text
embedded mapping
custom type
multiple columns
JSON
```

---

# 61. Database Type

Un `DatabaseType` representa la interpretación lógica de un valor entre PHP y el sistema de almacenamiento.

Ejemplos:

```text
StringType
IntegerType
BooleanType
DateTimeType
UuidType
JsonType
DecimalType
```

No debe confundirse directamente con el tipo SQL concreto.

Ejemplo:

```text
BooleanType
    │
    ├── PostgreSQL → BOOLEAN
    ├── MySQL      → BOOLEAN/TINYINT semantics
    └── SQLite     → compatible storage representation
```

---

# 62. Type Conversion

`Type Conversion` representa la transformación entre:

```text
PHP value
    ↕
Logical Database Type
    ↕
Driver value
    ↕
Database representation
```

Esta capa permite mantener portabilidad.

---

# 63. Metadata

`Metadata` es información estructurada que describe otros elementos del sistema.

Principales categorías:

```text
DatabaseMetadata
SchemaMetadata
TableMetadata
ColumnMetadata
EntityMetadata
FieldMetadata
RelationshipMetadata
QueryMetadata
ExecutionMetadata
```

Metadata no debe utilizarse como término ambiguo sin contexto cuando exista una categoría específica.

---

# 64. Metadata Registry

`MetadataRegistry` proporciona acceso coordinado a metadata conocida por el framework.

Puede integrar:

```text
compiled metadata
runtime metadata
schema metadata
entity metadata
cached metadata
```

---

# 65. Metadata Compiler

`MetadataCompiler` transforma definiciones declarativas en representaciones optimizadas para runtime.

Fuentes posibles:

```text
PHP Attributes
configuration
conventions
generated metadata
package extensions
```

Ejemplo:

```text
PHP Attributes
     │
     ▼
Metadata Reader
     │
     ▼
Metadata Compiler
     │
     ▼
Compiled Metadata
```

---

# 66. Database Context

`DatabaseContext` representa el estado contextual de una operación o ciclo de aplicación relacionado con Database.

Puede contener referencias lógicas a:

```text
connection selection
transaction context
tenant context
query context
EntityManager
UnitOfWork
IdentityMap
telemetry context
```

Debe tener lifecycle explícito.

---

# 67. Request Database Context

En aplicaciones HTTP, un `RequestDatabaseContext` representa el contexto Database perteneciente exclusivamente a una petición.

Especialmente con FrankenPHP:

```text
Worker
 │
 ├── Request A
 │    └── DatabaseContext A
 │
 ├── RESET
 │
 └── Request B
      └── DatabaseContext B
```

Nunca deberá ocurrir:

```text
Request A
   │
   └── IdentityMap
          │
          ▼
Request B
```

---

# 68. Runtime-Safe State

`Runtime-Safe State` es estado que puede mantenerse correctamente bajo runtimes persistentes sin provocar contaminación entre ciclos de ejecución.

VoltStack deberá clasificar explícitamente el estado como:

```text
process-scoped
worker-scoped
application-scoped
request-scoped
transaction-scoped
operation-scoped
```

Esto afecta directamente a:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 69. Connection Pool

Un `ConnectionPool` administra un conjunto reutilizable de conexiones físicas.

Conceptualmente:

```text
ConnectionPool
 ├── Connection 1
 ├── Connection 2
 ├── Connection 3
 └── Connection N
```

La existencia de pooling dependerá de las capacidades del runtime, driver y configuración.

La API pública no deberá depender de su existencia.

---

# 70. Read/Write Connection

VoltStack distingue conceptualmente:

```text
Read Connection
Write Connection
```

Esto permite:

```text
primary
replicas
read/write splitting
```

Ejemplo:

```text
SELECT
   │
   ▼
Read Target

INSERT
   │
   ▼
Write Target
```

El planner puede participar en esta decisión.

---

# 71. Database Topology

`DatabaseTopology` representa la organización lógica de nodos de almacenamiento.

Ejemplos:

```text
Single Node

Primary + Replicas

Sharded Cluster

Tenant Database Pool
```

La topología es independiente de la API utilizada por la aplicación.

---

# 72. Database Capability

Una `DatabaseCapability` representa una funcionalidad disponible en una plataforma concreta.

Ejemplos:

```text
RETURNING
UPSERT
JSON operators
CTE
recursive CTE
window functions
savepoints
generated columns
partial indexes
```

El sistema debe consultar capacidades en lugar de dispersar comprobaciones como:

```php
if ($driver === 'pgsql') {
    ...
}
```

---

# 73. Database Feature

`DatabaseFeature` representa una funcionalidad de alto nivel proporcionada por VoltStack.

Ejemplos:

```text
ORM
Migrations
Schema Introspection
Read/Write Splitting
Query Caching
Temporal Data
Full-Text Search
```

Feature y Capability no son equivalentes.

```text
Capability
    = lo que la plataforma puede hacer

Feature
    = lo que VoltStack proporciona
```

---

# 74. Query Cache

`QueryCache` representa mecanismos de cache relacionados con consultas.

Debe distinguirse entre:

```text
Compiled Query Cache
Query Plan Cache
Result Cache
Metadata Cache
```

Cada uno tiene semántica y lifecycle diferente.

---

# 75. Compiled Query Cache

Almacena representaciones compiladas reutilizables.

Ejemplo:

```text
Query Shape
    │
    ▼
Compiled SQL Template
```

No contiene necesariamente resultados de la consulta.

---

# 76. Result Cache

`ResultCache` almacena resultados obtenidos previamente.

Ejemplo:

```text
Query + Parameters
       │
       ▼
Cached Result
```

Su utilización requiere políticas explícitas de:

```text
TTL
invalidation
consistency
scope
tenant isolation
```

---

# 77. Query Shape

`QueryShape` representa la estructura de una consulta independientemente de determinados valores runtime.

Ejemplo:

```text
WHERE id = ?
```

Las consultas:

```text
WHERE id = 10
WHERE id = 20
```

pueden compartir un QueryShape.

Este concepto resulta útil para:

```text
compilation caching
telemetry aggregation
query fingerprints
performance analysis
```

---

# 78. Query Fingerprint

Un `QueryFingerprint` es una identificación normalizada de una forma de consulta.

Puede utilizarse para detectar:

```text
repeated queries
slow query patterns
N+1
performance regressions
```

---

# 79. Database Event

Un `DatabaseEvent` representa un evento observable emitido por el subsistema.

Ejemplos:

```text
DatabaseQueryStarted
DatabaseQueryFinished
DatabaseQueryFailed

TransactionStarted
TransactionCommitted
TransactionRolledBack

ConnectionOpened
ConnectionClosed
ConnectionFailed

SlowQueryDetected
NPlusOneDetected
```

Estos eventos deberán integrarse con `VoltStack/Quantum/Telemetry`.

---

# 80. Database Telemetry

`DatabaseTelemetry` representa métricas, traces, logs y eventos generados por Database.

Puede registrar:

```text
query duration
connection duration
rows returned
rows affected
transaction duration
query fingerprint
database target
errors
retry count
```

Nunca deberá exponer credenciales y deberá aplicar políticas para evitar filtración de datos sensibles en parámetros SQL.

---

# 81. Database Exception

`DatabaseException` es la categoría general de errores normalizados por Quantum Database.

Ejemplos conceptuales:

```text
ConnectionException
QueryException
ConstraintViolationException
TransactionException
DeadlockException
TimeoutException
SchemaException
MappingException
HydrationException
```

Los errores específicos de drivers deberán traducirse a una taxonomía estable de VoltStack cuando sea posible.

---

# 82. Constraint

Una `Constraint` representa una regla que limita estados válidos de los datos.

Ejemplos:

```text
PRIMARY KEY
UNIQUE
FOREIGN KEY
CHECK
NOT NULL
```

Las constraints pertenecen principalmente al modelo Schema.

No deben confundirse con Validation de aplicación.

---

# 83. Database Validation vs Application Validation

VoltStack distingue:

```text
Application Validation

de

Database Constraints
```

Ejemplo:

```text
Application:
email must be syntactically valid

Database:
email column NOT NULL
email UNIQUE
```

Ambos mecanismos son complementarios.

---

# 84. Tenant Database Context

`TenantDatabaseContext` contiene la información necesaria para resolver recursos Database pertenecientes a un tenant.

Puede determinar:

```text
connection
database
schema
table prefix
shard
credentials reference
```

Quantum Database debe ofrecer puntos de integración para Multitenancy sin convertir Multitenancy en dependencia obligatoria del núcleo Database.

---

# 85. Tenant Isolation

`TenantIsolation` representa las garantías que evitan acceso cruzado entre tenants.

Estrategias posibles:

```text
shared database/shared tables
shared database/separate schemas
separate databases
separate clusters
```

Estas estrategias pertenecen a extensiones de arquitectura y no deberán contaminar las operaciones básicas del Query Engine.

---

# 86. Shard

Un `Shard` representa una partición horizontal de datos ubicada en un destino lógico o físico determinado.

Ejemplo:

```text
Users 1–1M
   → Shard A

Users 1M–2M
   → Shard B
```

---

# 87. Shard Key

`ShardKey` representa el valor utilizado para determinar la partición responsable de un conjunto de datos.

Ejemplos:

```text
tenant_id
user_id
region
```

---

# 88. Routing Decision

Una `RoutingDecision` determina a qué destino Database deberá enviarse una operación.

Puede considerar:

```text
read/write
tenant
shard
region
consistency
transaction
availability
```

Conceptualmente:

```text
Query
   │
   ▼
Database Router
   │
   ▼
Routing Decision
   │
   ├── Connection A
   └── Connection B
```

---

# 89. Consistency Model

`ConsistencyModel` describe las garantías esperadas al leer y escribir datos dentro de determinadas topologías.

Puede incluir conceptos como:

```text
strong consistency
eventual consistency
read-your-writes
replica lag awareness
```

No todas las configuraciones deberán soportar todos los modelos.

---

# 90. Database Session

`DatabaseSession` representa estado asociado por el motor a una conexión o sesión de base de datos.

Ejemplos:

```text
session variables
temporary tables
transaction state
locks
prepared statements
```

Este concepto es especialmente importante con pooling y runtimes persistentes.

El framework deberá evitar devolver conexiones contaminadas a un pool.

---

# 91. Connection Reset

`ConnectionReset` es el proceso de restaurar una conexión reutilizable a un estado seguro.

Puede requerir:

```text
rollback unfinished transaction
release locks
reset session variables
clear temporary state
restore configuration
```

---

# 92. ORM Context Reset

`OrmContextReset` elimina estado ORM perteneciente al ciclo anterior.

Como mínimo:

```text
IdentityMap
UnitOfWork
managed entities
pending operations
temporary hydration state
transaction-bound state
```

Esto es obligatorio para seguridad bajo persistent workers.

---

# 93. Database Lifecycle

`DatabaseLifecycle` representa las fases por las que atraviesa el subsistema durante ejecución.

Ejemplo HTTP:

```text
Worker Boot
    │
    ▼
Application Boot
    │
    ▼
Request Begin
    │
    ▼
Database Context Create
    │
    ▼
Database Operations
    │
    ▼
Flush / Commit
    │
    ▼
Context Reset
    │
    ▼
Request End
```

---

# 94. Scope

`Scope` define el periodo durante el cual un objeto o estado Database es válido.

Scopes fundamentales:

```text
Process Scope
Worker Scope
Application Scope
Request Scope
Transaction Scope
Operation Scope
```

La documentación de cada componente stateful deberá declarar su scope.

---

# 95. Core Domain Aggregates

Conceptualmente, Quantum Database puede organizarse alrededor de los siguientes grandes agregados arquitectónicos:

```text
Database Domain
│
├── Connection Domain
│
├── Query Domain
│
├── Execution Domain
│
├── Schema Domain
│
├── Transaction Domain
│
├── ORM Domain
│
├── Metadata Domain
│
├── Persistence Domain
│
├── Topology Domain
└── Observability Domain
```

---

# 96. Connection Domain

Incluye:

```text
DatabaseManager
Connection
PhysicalConnection
ConnectionConfiguration
Driver
Dialect
DatabasePlatform
ConnectionPool
ConnectionReset
```

---

# 97. Query Domain

Incluye:

```text
Query
QueryBuilder
QueryAST
QueryNode
Expression
Predicate
Parameter
SemanticQuery
SemanticGraph
QueryOptimizer
QueryPlanner
QueryShape
QueryFingerprint
```

---

# 98. Execution Domain

Incluye:

```text
ExecutionPlan
QueryCompiler
CompiledQuery
QueryExecutor
Result
ResultSet
Row
```

---

# 99. Schema Domain

Incluye:

```text
Schema
SchemaMetadata
SchemaManager
SchemaAST
SchemaPlanner
Migration
Constraint
```

---

# 100. Transaction Domain

Incluye:

```text
Transaction
TransactionContext
Savepoint
IsolationLevel
RetryPolicy
```

---

# 101. ORM Domain

Incluye:

```text
Entity
Model
EntityIdentity
EntityManager
Repository
IdentityMap
UnitOfWork
ChangeSet
Relationship
RelationLoadPlanner
Hydrator
```

---

# 102. Persistence Domain

Incluye:

```text
PersistenceEngine
PersistencePlan
ChangeSet
TypeConversion
Dehydration
Flush
```

---

# 103. Metadata Domain

Incluye:

```text
MetadataRegistry
MetadataCompiler
EntityMetadata
RelationshipMetadata
SchemaMetadata
DatabaseMetadata
```

---

# 104. Topology Domain

Incluye:

```text
DatabaseTopology
ReadTarget
WriteTarget
Shard
ShardKey
RoutingDecision
ConsistencyModel
TenantDatabaseContext
```

---

# 105. Observability Domain

Incluye:

```text
DatabaseEvent
DatabaseTelemetry
QueryFingerprint
SlowQueryDetection
NPlusOneDetection
DatabaseException
```

---

# 106. Relaciones principales del modelo

El modelo global puede representarse como:

```text
Application
    │
    ├───────────────┐
    ▼               ▼
Query API         ORM API
    │               │
    │          EntityManager
    │               │
    │       ┌───────┼────────┐
    │       ▼       ▼        ▼
    │ IdentityMap UnitOfWork Repository
    │               │        │
    │               ▼        │
    │        PersistenceEngine
    │               │        │
    └───────────────┴────────┘
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
               Optimizer
                    │
                    ▼
                 Planner
                    │
                    ▼
              Execution Plan
                    │
                    ▼
                 Compiler
                    │
                    ▼
              Compiled Query
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

---

# 107. Relación entre ORM y Query Engine

El ORM no debe implementar su propio motor SQL independiente.

La regla arquitectónica es:

```text
ORM
 │
 ▼
Query Engine
 │
 ▼
Execution Engine
 │
 ▼
Connection Layer
```

Por tanto:

```text
Active Record
Repository
EntityManager
Relationships
UnitOfWork
```

terminarán utilizando el mismo Query Engine.

---

# 108. Relación entre Query Builder y ORM

QueryBuilder y ORM son APIs diferentes, pero comparten infraestructura.

```text
QueryBuilder ────────┐
                    │
Repository ──────────┤
                    ▼
EntityManager ───► Query Model
                    │
                    ▼
                 Query AST
```

Esto evita duplicación y comportamientos inconsistentes.

---

# 109. Relación entre Schema y Query

Schema y Query deberán utilizar modelos estructurados independientes:

```text
Query AST
Schema AST
```

pero podrán compartir:

```text
Type System
Dialect
Platform Capabilities
Compiler Infrastructure
Metadata
Connection
Execution
```

---

# 110. Relación entre Transaction y Connection

Una Transaction siempre está asociada a una Connection lógica apropiada.

```text
Transaction
    │
    ▼
Connection
    │
    ▼
Physical Connection
```

Durante una transacción no deberán producirse cambios arbitrarios de conexión que rompan atomicidad.

---

# 111. Relación entre EntityManager y Transaction

`EntityManager` no es una Transaction.

Puede operar:

```text
sin transacción explícita
```

o dentro de:

```text
TransactionContext
```

Un `flush()` puede requerir una transacción dependiendo de la operación y política.

La coordinación será responsabilidad de la capa de persistencia/transacciones.

---

# 112. Relación entre Identity Map y Cache

IdentityMap no es un cache general.

```text
IdentityMap
    = identidad de objetos dentro del contexto ORM

ResultCache
    = cache reutilizable de resultados

MetadataCache
    = cache de metadata

QueryPlanCache
    = cache de planificación
```

Confundir estos conceptos puede provocar errores graves de consistencia.

---

# 113. Relación entre Metadata y Schema

`EntityMetadata` describe persistencia orientada al dominio.

`SchemaMetadata` describe estructura de base de datos.

No son equivalentes.

Ejemplo:

```text
EntityMetadata(User)
       │
       │ mapping
       ▼
SchemaMetadata(users)
```

Una Entity podría incluso involucrar múltiples estructuras Schema.

---

# 114. Terminología prohibida o desaconsejada

Para evitar ambigüedad, la documentación no deberá utilizar indiscriminadamente:

```text
DB object
database object
model data
query object
database model
manager
cache
context
metadata
```

cuando exista un término más preciso.

Por ejemplo:

En lugar de:

```text
manager
```

usar:

```text
DatabaseManager
EntityManager
TransactionManager
SchemaManager
```

según corresponda.

---

# 115. Convenciones terminológicas

Los nombres arquitectónicos en inglés se consideran canónicos.

La documentación española puede explicar:

```text
Query Planner → planificador de consultas
Identity Map → mapa de identidad
Unit of Work → unidad de trabajo
```

pero los identificadores técnicos deberán conservar:

```text
QueryPlanner
IdentityMap
UnitOfWork
```

Esto facilita correspondencia directa entre documentación y código.

---

# 116. Diferencia entre Manager, Engine, Planner y Compiler

VoltStack utilizará estas palabras con significados consistentes.

## Manager

Coordina recursos o componentes.

```text
DatabaseManager
EntityManager
SchemaManager
```

## Engine

Implementa un proceso funcional amplio.

```text
PersistenceEngine
ExecutionEngine
```

## Planner

Decide cómo realizar una operación.

```text
QueryPlanner
PersistencePlanner
RelationLoadPlanner
SchemaPlanner
```

## Compiler

Transforma una representación estructurada hacia otra representación ejecutable u optimizada.

```text
QueryCompiler
MetadataCompiler
SchemaCompiler
```

---

# 117. Resolver

`Resolver` selecciona una implementación o recurso a partir de contexto.

Ejemplos:

```text
ConnectionResolver
DriverResolver
DialectResolver
RepositoryResolver
TenantConnectionResolver
```

No debe realizar orquestación extensa propia de un Manager.

---

# 118. Registry

`Registry` mantiene o proporciona acceso a elementos registrados.

Ejemplos:

```text
DriverRegistry
DialectRegistry
MetadataRegistry
TypeRegistry
```

Un Registry no debería convertirse en un Service Locator universal.

---

# 119. Adapter

`Adapter` integra una API o tecnología externa con contratos de VoltStack.

Ejemplos futuros:

```text
PDOAdapter
TelemetryAdapter
CacheAdapter
```

Debe utilizarse cuando exista una verdadera frontera de adaptación.

---

# 120. Strategy

Una `Strategy` encapsula una política intercambiable de comportamiento.

Ejemplos:

```text
HydrationStrategy
RelationLoadingStrategy
IdentifierGenerationStrategy
RetryStrategy
```

---

# 121. Policy

Una `Policy` describe reglas configurables que determinan decisiones.

Ejemplos:

```text
RetryPolicy
ConnectionSelectionPolicy
QueryCachePolicy
```

Strategy y Policy pueden colaborar, pero no son necesariamente equivalentes.

---

# 122. Invariantes fundamentales

El dominio Database deberá respetar permanentemente las siguientes invariantes.

### DB-INV-001

Una Query lógica no deberá depender directamente de SQL específico de un dialecto hasta la fase apropiada de compilación.

### DB-INV-002

El ORM deberá utilizar el Query/Execution Engine común.

### DB-INV-003

Active Record no deberá constituir un segundo ORM independiente.

### DB-INV-004

IdentityMap tendrá un scope explícito y nunca podrá filtrarse accidentalmente entre requests.

### DB-INV-005

UnitOfWork deberá ser limpiado al terminar su lifecycle.

### DB-INV-006

Una Transaction no podrá migrar silenciosamente entre conexiones físicas incompatibles.

### DB-INV-007

Driver y Dialect deberán permanecer desacoplados conceptualmente.

### DB-INV-008

Las capacidades específicas de plataforma deberán modelarse explícitamente.

### DB-INV-009

Metadata deberá ser estructurada y potencialmente compilable/cacheable.

### DB-INV-010

Las credenciales nunca formarán parte de eventos o telemetría pública.

### DB-INV-011

El sistema deberá ser seguro bajo persistent workers.

### DB-INV-012

Multitenancy será una integración opcional y no una dependencia fundamental de Database Core.

---

# 123. Modelo de lifecycle bajo FrankenPHP

FrankenPHP es el runtime inicialmente recomendado para VoltStack.

Por ello, el modelo de dominio debe asumir que el proceso PHP puede sobrevivir a múltiples requests.

```text
Worker Start
     │
     ▼
Framework Boot
     │
     ├─────────────────────────────┐
     │                             │
     ▼                             │
Request 1                          │
     │                             │
DatabaseContext                    │
     │                             │
EntityManager                      │
     │                             │
UnitOfWork                         │
     │                             │
IdentityMap                        │
     │                             │
     ▼                             │
Request End                        │
     │                             │
     ▼                             │
RESET                              │
     │                             │
     ▼                             │
Request 2 ◄────────────────────────┘
```

El reset deberá impedir la persistencia accidental de estado request-scoped.

---

# 124. Compatibilidad con otros runtimes

La arquitectura no deberá codificar reglas exclusivas de FrankenPHP.

Debe modelar el lifecycle mediante abstracciones compatibles con:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

Conceptualmente:

```text
Runtime
   │
   ▼
Database Lifecycle Coordinator
   │
   ├── begin context
   ├── execute
   ├── finalize
   └── reset
```

---

# 125. Modelo de paquetes

La separación conceptual inicial será:

```text
VoltStack
│
├── Platform
│   └── Database Contracts
│
└── Quantum
    └── Database
```

`Platform` podrá contener únicamente contratos mínimos que otros subsistemas fundamentales necesiten conocer.

Ejemplos conceptuales:

```text
DatabaseManagerInterface
DatabaseConnectionInterface
TransactionInterface
```

La implementación completa deberá permanecer en:

```text
VoltStack/Quantum/Database
```

---

# 126. Dominios internos candidatos

Una organización conceptual posible es:

```text
Quantum/Database
│
├── Connection
├── Driver
├── Platform
├── Query
├── Semantic
├── Optimizer
├── Planner
├── Compiler
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
├── Type
├── Cache
├── Topology
├── Telemetry
├── Exception
└── Lifecycle
```

La estructura física definitiva se establecerá en documentos posteriores.

---

# 127. Modelo conceptual completo

```text
                        VOLTSTACK APPLICATION
                                 │
              ┌──────────────────┼───────────────────┐
              │                  │                   │
              ▼                  ▼                   ▼
        Active Record       Repository API       Query API
              │                  │                   │
              └──────────┬───────┘                   │
                         ▼                           │
                    EntityManager                    │
                         │                           │
             ┌───────────┼───────────┐               │
             ▼           ▼           ▼               │
        IdentityMap   UnitOfWork   Metadata           │
                         │                           │
                         ▼                           │
                 PersistenceEngine                   │
                         │                           │
                         └───────────┬───────────────┘
                                     ▼
                                  Query AST
                                     │
                                     ▼
                              Semantic Analyzer
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
                              Dialect Compiler
                                     │
                                     ▼
                               Compiled Query
                                     │
                                     ▼
                                Query Executor
                                     │
                                     ▼
                              Database Connection
                                     │
                      ┌──────────────┼───────────────┐
                      ▼              ▼               ▼
                    MySQL        PostgreSQL       MariaDB
                                                       │
                                                   SQLite
```

---

# 128. Glosario canónico resumido

| Término | Definición |
|---|---|
| DatabaseManager | Coordinador de conexiones y configuración Database |
| Connection | Conexión lógica a una base de datos |
| PhysicalConnection | Recurso físico real de conexión |
| Driver | Implementación de comunicación con una tecnología DB |
| Dialect | Reglas SQL específicas de una plataforma |
| DatabasePlatform | Capacidades conocidas del motor |
| Query | Solicitud lógica de datos |
| QueryBuilder | API programática para construir Query |
| QueryAST | Representación estructural de una Query |
| SemanticGraph | Relaciones semánticas de una operación |
| QueryOptimizer | Optimiza la representación lógica |
| QueryPlanner | Decide estrategia de ejecución |
| ExecutionPlan | Describe operaciones ejecutables |
| QueryCompiler | Convierte representación lógica a dialecto ejecutable |
| CompiledQuery | Statement y parámetros listos para ejecución |
| QueryExecutor | Ejecuta consultas compiladas |
| ResultSet | Conjunto tabular de resultados |
| Schema | Estructura lógica de base de datos |
| SchemaMetadata | Descripción estructurada del Schema |
| Migration | Transición versionada del Schema |
| Transaction | Unidad atómica de operaciones |
| Entity | Objeto de dominio con identidad |
| Model | API Active Record opcional |
| EntityManager | Coordinador del contexto ORM |
| IdentityMap | Garantiza identidad de instancias administradas |
| UnitOfWork | Coordina cambios pendientes |
| ChangeSet | Diferencias detectadas en una Entity |
| Repository | Acceso orientado al dominio a Entities |
| PersistenceEngine | Traduce cambios ORM a operaciones DB |
| Hydration | Conversión de resultados a objetos/estructuras |
| Relationship | Asociación entre entidades |
| RelationLoadPlanner | Planifica carga eficiente de relaciones |
| Metadata | Información estructurada sobre componentes DB |
| DatabaseContext | Estado Database perteneciente a un lifecycle |
| DatabaseTopology | Organización de nodos de almacenamiento |
| DatabaseCapability | Capacidad disponible en una plataforma |
| QueryFingerprint | Identidad normalizada de una consulta |
| DatabaseTelemetry | Observabilidad del subsistema Database |

---

# 129. Reglas de diseño derivadas

A partir de este modelo se establecen las siguientes reglas para documentos e implementación posteriores:

1. Toda API de consulta deberá converger eventualmente en el Query Engine.

2. Toda operación ORM deberá utilizar el mismo Execution Engine que las consultas directas.

3. SQL deberá considerarse una representación compilada, no el modelo interno principal.

4. Driver, Dialect y DatabasePlatform deberán permanecer separados.

5. Metadata deberá ser explícita, estructurada y compilable.

6. UnitOfWork e IdentityMap serán componentes centrales del ORM.

7. Active Record será una capa de ergonomía sobre el ORM común.

8. Repository será una API de dominio y no un sustituto del Query Engine.

9. Query optimization y database-native optimization deberán considerarse niveles diferentes.

10. Toda infraestructura stateful deberá declarar lifecycle y scope.

11. Persistent workers serán una condición arquitectónica de primer nivel.

12. Multitenancy, sharding y topologías distribuidas deberán extender el modelo sin romper el núcleo básico.

13. Telemetry deberá integrarse transversalmente sin contaminar la lógica del dominio.

14. Las APIs simples no deberán impedir capacidades empresariales avanzadas.

---

# 130. Resultado arquitectónico

El modelo de dominio resultante puede resumirse como:

```text
                    VoltStack Database
                           │
        ┌──────────────────┼───────────────────┐
        │                  │                   │
     Data API          Persistence          Schema
        │                  │                   │
        ▼                  ▼                   ▼
     Query AST          UnitOfWork         Schema AST
        │                  │                   │
        └──────────┬───────┴───────────────────┘
                   ▼
              Planning Layer
                   │
                   ▼
             Execution Layer
                   │
                   ▼
             Connection Layer
                   │
                   ▼
              Driver Layer
                   │
                   ▼
              Database
```

Este modelo permite que VoltStack proporcione simultáneamente:

```text
Laravel-like developer experience

+

Doctrine-like persistence architecture

+

VoltStack-native Query AST

+

Semantic Query Model

+

Query Optimization

+

Query Planning

+

Persistent Runtime Safety
```

sin mantener motores paralelos e incompatibles.

---

# 131. Conclusión

`VoltStack/Quantum/Database` se define como una plataforma completa de acceso y persistencia de datos, no simplemente como un ORM.

Su dominio se construye alrededor de seis ideas fundamentales:

```text
Structured Queries
Explicit Metadata
Planned Execution
Unified Persistence
Portable Database Infrastructure
Explicit Runtime Lifecycle
```

La API pública podrá ser sencilla:

```php
User::where('active', true)->get();
```

mientras que internamente la operación podrá recorrer:

```text
Model API
    ↓
ORM
    ↓
Query AST
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
    ↓
Hydration
```

La complejidad pertenece al framework, no al desarrollador.

Este vocabulario constituye la base terminológica para el resto de la arquitectura de `VoltStack/Quantum/Database`.

---

# 132. Siguiente documento

El siguiente documento deberá profundizar en la arquitectura estructural y límites internos definidos aquí.

```text
04_DATABASE_COMPONENT_AND_MODULE_ARCHITECTURE.md
```

Este documento deberá establecer cómo los dominios:

```text
Connection
Query
Semantic
Optimizer
Planner
Compiler
Execution
Schema
Transaction
ORM
Metadata
Persistence
Lifecycle
Telemetry
```

se convierten en módulos concretos, contratos, dependencias permitidas y namespaces dentro de `VoltStack/Quantum/Database`.