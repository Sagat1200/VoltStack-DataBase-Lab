# 20_DATABASE_POSTGRESQL_PLATFORM.md

# VoltStack Quantum Database
## PostgreSQL Platform Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 20 — PostgreSQL Platform  
**Estado:** Architecture Specification  
**Nivel:** Infrastructure / Platform / Capability  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura oficial de soporte para:

```text
PostgreSQL
```

dentro de:

```text
VoltStack/Quantum/Database
```

La integración deberá soportar desde operaciones SQL convencionales hasta capacidades avanzadas de PostgreSQL sin introducir dependencias específicas del motor en:

```text
Query Builder
ORM
EntityManager
UnitOfWork
Repository
Application
```

---

# 2. Principio arquitectónico

PostgreSQL será representado mediante componentes independientes:

```text
Driver
Dialect
Platform
Capabilities
```

Por tanto:

```text
PostgreSqlDriver
≠
PostgreSqlDialect
≠
PostgreSqlPlatform
≠
PostgreSqlCapabilityProvider
```

---

# 3. Arquitectura general

```text
Application
    │
    ▼
Database API
    │
    ▼
Query / ORM / Schema
    │
    ▼
Query Engine
    │
    ▼
Planner
    │
    ▼
PostgreSqlPlatform
    │
    ├── Capabilities
    ├── Type Semantics
    ├── Schema Semantics
    ├── Transaction Semantics
    └── Runtime Semantics
    │
    ▼
PostgreSqlDialect
    │
    ▼
SQL Compiler
    │
    ▼
Connection
    │
    ▼
PostgreSQL Driver
    │
    ▼
PostgreSQL Server
```

---

# 4. Driver oficial inicial

La implementación inicial podrá utilizar:

```text
PDO_PGSQL
```

mediante:

```text
PdoPostgreSqlDriver
```

---

# 5. Driver ID

ID conceptual:

```text
pdo.pgsql
```

---

# 6. Alias público

La configuración podrá aceptar:

```text
pgsql
postgres
postgresql
```

como aliases.

---

# 7. Normalización

Ejemplo:

```yaml
database:
  connections:
    primary:
      driver: postgresql
```

podrá normalizarse internamente a:

```text
Driver:
pdo.pgsql

Platform:
postgresql

Dialect:
postgresql
```

---

# 8. Platform ID

El identificador oficial será:

```text
postgresql
```

---

# 9. Dialect ID

```text
postgresql
```

---

# 10. Driver ≠ Platform

Nunca asumir:

```text
pdo.pgsql
=
PostgreSqlPlatform
```

como equivalencia arquitectónica absoluta.

El Driver describe:

```text
transport
native connection
statement preparation
parameter binding
result transport
native errors
```

Platform describe:

```text
server semantics
features
types
schema behavior
transaction behavior
capabilities
```

---

# 11. Driver alternativo futuro

La arquitectura deberá permitir:

```text
native.pgsql
async.pgsql
coroutine.pgsql
cloud.postgresql
```

sin modificar el Query Engine.

---

# 12. Componentes PostgreSQL

Conceptualmente:

```text
PostgreSql/
├── Platform
├── Dialect
├── Discovery
├── Capability
├── Type
├── Schema
├── Transaction
├── Session
├── Error
├── Planning
├── Introspection
├── Migration
└── Testing
```

---

# 13. PostgreSqlPlatform

Componente principal:

```php
final class PostgreSqlPlatform implements DatabasePlatformInterface
{
}
```

Preferentemente:

```text
immutable
stateless
reentrant
shareable
```

---

# 14. PostgreSqlPlatform no contiene Connection

Prohibido:

```php
$platform->setConnection($connection);
```

---

# 15. PostgreSqlPlatform no contiene versión mutable

Prohibido:

```php
$platform->setVersion($version);
```

---

# 16. ResolvedPlatform

La versión y capabilities específicas pertenecerán a:

```text
ResolvedPlatform
```

Conceptualmente:

```php
final readonly class ResolvedPlatform
{
    public function __construct(
        public DatabasePlatformInterface $platform,
        public ServerVersion $version,
        public CapabilitySnapshot $capabilities,
    ) {}
}
```

---

# 17. Múltiples servidores

Un mismo worker podrá manejar simultáneamente:

```text
PostgreSQL target A
PostgreSQL target B
MySQL target C
MariaDB target D
SQLite target E
```

sin estado global compartido.

---

# 18. Server discovery

Se implementará:

```text
PostgreSqlServerDiscovery
```

---

# 19. Discovery responsibilities

Podrá determinar:

```text
server product
server version
server identity
relevant server configuration
relevant session capabilities
```

---

# 20. Discovery flow

```text
NativeConnection
      │
      ▼
PostgreSqlServerDiscovery
      │
      ▼
PostgreSqlServerMetadata
      │
      ▼
PlatformResolver
      │
      ▼
ResolvedPlatform
```

---

# 21. Server metadata

Conceptualmente:

```php
final readonly class PostgreSqlServerMetadata
{
    public function __construct(
        public ServerVersion $version,
        public ServerIdentity $identity,
        public ServerMetadataAttributes $attributes,
    ) {}
}
```

---

# 22. ServerVersion

La versión será un value object.

Nunca:

```php
if ($version >= '15') {
}
```

mediante comparación textual arbitraria.

---

# 23. Version representation

Conceptualmente:

```text
major
minor
patch
vendor metadata
```

según la información disponible.

---

# 24. PostgreSqlVersionParser

Existirá:

```text
PostgreSqlVersionParser
```

responsable de normalizar la versión.

---

# 25. Platform support policy

Podrá existir:

```text
PostgreSqlPlatformSupportPolicy
```

para definir:

```text
minimum supported version
tested versions
deprecated versions
unsupported versions
future-version policy
```

---

# 26. Future versions

Una versión futura desconocida podrá manejarse mediante:

```text
STRICT
COMPATIBILITY
WARN
```

según configuración.

---

# 27. Capability architecture

PostgreSQL deberá integrarse completamente con:

```text
Database Platform Capability System
```

definido en el documento `18`.

---

# 28. Capability provider

```text
PostgreSqlPlatformCapabilityProvider
```

---

# 29. Capability resolution

```text
PostgreSqlPlatform
        │
        ▼
Baseline Capabilities
        │
        ▼
Version Rules
        │
        ▼
Server Configuration
        │
        ▼
Driver Capabilities
        │
        ▼
Connection Restrictions
        │
        ▼
Policy
        │
        ▼
EffectiveCapabilitySet
```

---

# 30. Capability categories

PostgreSQL deberá describir al menos:

```text
Query
Expression
CTE
Window
Set Operations
Returning
Upsert
Types
JSON
Arrays
UUID
Enums
Domains
Ranges
Schema
Indexes
Constraints
Sequences
Identity
Transactions
Locking
Session
DDL
Introspection
Bulk Operations
Administration
```

---

# 31. Capabilities granulares

Evitar:

```text
supportsPostgresFeatures = true
```

Preferir:

```text
query.returning.insert
query.returning.update
query.returning.delete

query.cte
query.cte.recursive

query.window

schema.index.partial
schema.index.expression

transaction.savepoint
```

---

# 32. PostgreSQL Query Builder

El Query Builder seguirá siendo portable.

Ejemplo:

```php
DB::table('users')
    ->where('active', true)
    ->orderBy('name')
    ->get();
```

no contendrá conocimiento de PostgreSQL.

---

# 33. Query pipeline

```text
Query Builder
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
     ├── PostgreSQL Capabilities
     └── PostgreSQL Semantics
             │
             ▼
        Execution Plan
             │
             ▼
          Compiler
             │
             ▼
      PostgreSqlDialect
             │
             ▼
             SQL
```

---

# 34. PostgreSqlDialect

Componente:

```text
PostgreSqlDialect
```

será responsable de sintaxis PostgreSQL.

---

# 35. Dialect responsibilities

Incluye:

```text
identifier quoting
SQL syntax
parameter syntax where applicable
RETURNING syntax
ON CONFLICT syntax
locking syntax
DDL fragments
type declarations
operator syntax
```

---

# 36. Dialect non-responsibilities

No decidirá:

```text
whether a feature should be used
whether a feature should be emulated
whether a query should use a replica
whether an entity should be persisted
whether a transaction should retry
```

---

# 37. Planner responsibility

Planner selecciona estrategia.

Dialect traduce estrategia a SQL.

---

# 38. Identifier quoting

Los identifiers deberán permanecer estructurados hasta Compiler.

```text
Identifier
    │
    ▼
Compiler
    │
    ▼
PostgreSqlDialect
    │
    ▼
Quoted Identifier
```

---

# 39. No prequoted identifiers

Evitar:

```text
"public"."users"
```

dentro del AST.

---

# 40. Schema-qualified identifiers

PostgreSQL requiere soporte natural para:

```text
schema.table
```

---

# 41. QualifiedIdentifier

Conceptualmente:

```text
QualifiedIdentifier
├── catalog?
├── schema?
└── object
```

---

# 42. PostgreSQL schemas

Los schemas serán una capacidad de primera clase.

---

# 43. Database ≠ Schema

Muy importante:

```text
PostgreSQL Database
≠
PostgreSQL Schema
```

---

# 44. Canonical schema model

VoltStack deberá distinguir:

```text
Database
Schema
Table
```

sin colapsarlos.

---

# 45. Schema namespace

Ejemplo:

```text
database
└── public
    └── users
```

---

# 46. Non-public schemas

Deberá soportarse:

```text
app
audit
billing
tenant_001
```

sin asumir `public`.

---

# 47. Search path

PostgreSQL `search_path` será tratado como:

```text
session state
```

---

# 48. Search path contamination

En runtime persistente:

```text
Request A
→ search_path = tenant_a

RESET

Request B
→ search_path = tenant_b
```

deberá garantizar aislamiento.

---

# 49. SearchPath value object

Podrá existir:

```text
SearchPath
```

dentro del Session Profile.

---

# 50. Explicit schema preference

Para operaciones sensibles será preferible usar:

```text
schema-qualified identifiers
```

en vez de depender implícitamente del `search_path`.

---

# 51. Multitenancy

El sistema opcional de Multitenancy podrá utilizar:

```text
schema-per-tenant
```

sobre PostgreSQL.

---

# 52. Database core independence

Sin embargo:

```text
PostgreSqlPlatform
```

no dependerá de:

```text
Quantum/Multitenancy
```

---

# 53. Tenant integration

Será:

```text
Multitenancy Package
       │
       ▼
TenantConnection/Schema Resolver
       │
       ▼
Database Context
       │
       ▼
PostgreSQL Schema Semantics
```

---

# 54. Type architecture

PostgreSQL posee un sistema de tipos especialmente rico.

VoltStack deberá preservarlo sin obligar al ORM portable a conocer todos sus detalles.

---

# 55. Type flow

```text
VoltStack Type
      │
      ▼
PostgreSqlTypeMapping
      │
      ▼
PostgreSQL Native Type
```

---

# 56. PostgreSqlTypeMapping

Componente:

```text
PostgreSqlTypeMapping
```

---

# 57. Core types

Debe contemplar:

```text
smallint
integer
bigint
numeric
decimal
real
double precision
boolean
char
varchar
text
bytea
date
time
timestamp
timestamp with time zone
interval
uuid
json
jsonb
```

---

# 58. Extended types

Arquitectura preparada para:

```text
array
enum
domain
range
multirange
network
bit string
geometric
xml
full-text types
custom extension types
```

---

# 59. Portable vs native types

Ejemplo:

```text
VoltStack JsonType
```

podrá mapearse a:

```text
json
```

o:

```text
jsonb
```

según:

```text
mapping policy
explicit metadata
platform strategy
```

---

# 60. JSON

PostgreSQL deberá exponer capabilities granulares:

```text
json.storage
json.validation
json.extract
json.path
json.contains
json.mutation
json.aggregate
json.index
```

---

# 61. JSONB

JSONB deberá modelarse como capacidad/tipo especializado.

---

# 62. JsonType vs JsonbType

VoltStack podrá ofrecer:

```text
JsonType
```

portable y:

```text
PostgreSqlJsonbType
```

como extensión nativa.

---

# 63. Preferencia configurable

También podrá existir una política:

```text
portable JSON
→ PostgreSQL JSONB
```

cuando el desarrollador la seleccione.

---

# 64. JSON expressions

Query AST deberá representar semántica:

```text
JsonExtractExpression
JsonContainsExpression
JsonPathExpression
```

cuando sean portables.

---

# 65. JSON compiler

```text
JsonExpression
      │
      ▼
Planner
      │
      ▼
PostgreSql Strategy
      │
      ▼
PostgreSqlDialect
```

---

# 66. Native JSON operators

Operadores específicos podrán exponerse mediante extensiones avanzadas sin contaminar el AST portable.

---

# 67. Arrays

PostgreSQL arrays serán una capability de plataforma.

---

# 68. Array type

Podrá existir:

```text
PostgreSqlArrayType<T>
```

o equivalente conceptual.

---

# 69. Array metadata

Debe preservar:

```text
element type
dimensions
nullable semantics
```

cuando corresponda.

---

# 70. Array query operations

Capabilities:

```text
array.contains
array.contained_by
array.overlap
array.length
array.unnest
```

según la abstracción final.

---

# 71. ORM arrays

El ORM podrá mapear:

```text
PHP array
```

a PostgreSQL array únicamente mediante mapping explícito.

---

# 72. PHP array ambiguity

No asumir:

```text
PHP array
=
PostgreSQL array
```

porque también puede representar:

```text
JSON
collection
relationship
serialized value
```

---

# 73. UUID

PostgreSQL tendrá soporte nativo para UUID.

---

# 74. Portable UUID type

VoltStack deberá tener:

```text
UuidType
```

portable.

---

# 75. PostgreSQL mapping

```text
UuidType
→
uuid
```

cuando corresponda.

---

# 76. UUID generation

No confundir:

```text
UUID storage
```

con:

```text
UUID generation
```

---

# 77. UUID generation capabilities

Podrán depender de:

```text
server version
installed extensions
server functions
application strategy
```

---

# 78. Application-generated UUID

Debe seguir siendo una estrategia válida y portable.

---

# 79. Enum types

PostgreSQL soporta enums nativos.

---

# 80. Enum model

VoltStack deberá distinguir:

```text
PHP Enum
Database Enum
String Mapping
Check Constraint Mapping
```

---

# 81. Native enum capability

```text
type.enum.native
```

---

# 82. Enum lifecycle

Schema System deberá contemplar:

```text
create enum
add value
rename value
drop enum
```

según capabilities.

---

# 83. Enum migrations

Cambios de enum requieren planificación específica.

---

# 84. No generic ALTER assumption

Migration Planner no deberá tratar un enum como simple VARCHAR.

---

# 85. Domains

PostgreSQL domains deberán poder representarse como extensión de Schema/Type System.

---

# 86. Domain model

Conceptualmente:

```text
DomainDefinition
├── name
├── baseType
├── default
├── nullable
└── constraints
```

---

# 87. Domain capability

```text
type.domain
schema.domain
```

---

# 88. ORM domain mapping

El ORM podrá mapear un Domain a:

```text
ValueObject
```

mediante Type System.

---

# 89. Range types

Arquitectura preparada para:

```text
int4range
int8range
numrange
tsrange
tstzrange
daterange
```

y tipos personalizados.

---

# 90. Range abstraction

Podrá existir:

```text
RangeValue<T>
```

---

# 91. Range capability

```text
type.range
query.range
```

---

# 92. Multirange

Se modelará como capability separada.

---

# 93. Date/time semantics

PostgreSQL distingue claramente tipos temporales.

---

# 94. Timestamp mapping

VoltStack deberá diferenciar:

```text
timestamp
timestamp with time zone
```

semánticamente.

---

# 95. PHP DateTime mapping

Type System manejará conversión.

Platform describe almacenamiento y semántica.

---

# 96. Timezone session state

La timezone de sesión deberá rastrearse.

---

# 97. Persistent runtime safety

Nunca:

```text
Request A timezone
→ leaks
→ Request B
```

---

# 98. Interval

PostgreSQL interval podrá exponerse mediante:

```text
IntervalType
```

o extensión específica.

---

# 99. Numeric precision

`numeric` deberá preservar:

```text
precision
scale
```

cuando estén declarados.

---

# 100. Arbitrary precision

El Type System no deberá convertir indiscriminadamente valores NUMERIC a `float`.

---

# 101. Decimal strategy

Preferir representación segura:

```text
string
Decimal Value Object
custom numeric type
```

según configuración.

---

# 102. Boolean

PostgreSQL boolean podrá mapearse directamente al tipo semántico:

```text
BooleanType
```

---

# 103. Binary data

```text
bytea
```

se mapeará a:

```text
BinaryType
```

---

# 104. Sequence architecture

Sequences serán una capacidad de primera clase.

---

# 105. Sequence model

```text
SequenceDefinition
├── name
├── schema
├── start
├── increment
├── min
├── max
├── cycle
└── cache
```

---

# 106. Sequence capabilities

```text
schema.sequence
schema.sequence.create
schema.sequence.alter
schema.sequence.drop
query.sequence.next_value
query.sequence.current_value
```

---

# 107. Sequence ≠ identity

Mantener:

```text
Sequence
≠
Identity Column
```

---

# 108. Identity columns

Schema Model deberá poder representar:

```text
GENERATED ALWAYS
GENERATED BY DEFAULT
```

semánticamente.

---

# 109. GeneratedIdentifierStrategy

ORM podrá utilizar:

```text
IDENTITY
SEQUENCE
APPLICATION
CUSTOM
```

sin depender directamente de PostgreSQL.

---

# 110. Persistence Planner

```text
Entity
   │
   ▼
UnitOfWork
   │
   ▼
Persistence Planner
   │
   ├── Platform Capabilities
   ├── Identifier Strategy
   └── Returning Capabilities
           │
           ▼
       Query Model
```

---

# 111. RETURNING

PostgreSQL deberá integrar `RETURNING` como capability semántica.

---

# 112. Granular returning capabilities

```text
query.returning.insert
query.returning.update
query.returning.delete
query.returning.multi_row
query.returning.arbitrary_columns
```

---

# 113. RETURNING AST

El AST podrá representar:

```text
ReturningClause
```

---

# 114. Query Builder

Ejemplo conceptual:

```php
DB::table('users')
    ->insert(...)
    ->returning(['id']);
```

sin que Builder genere SQL.

---

# 115. ORM generated values

Persistence Planner podrá utilizar RETURNING para recuperar:

```text
identity
generated columns
timestamps
database defaults
```

cuando sea semánticamente apropiado.

---

# 116. RETURNING ≠ generated key API

Mantener:

```text
SQL RETURNING
≠
Driver generated-key retrieval
```

---

# 117. ON CONFLICT

PostgreSQL UPSERT deberá modelarse mediante el AST semántico común.

---

# 118. UpsertQuery

```text
UpsertQuery
├── target
├── values
├── conflict target
├── action
└── returning
```

---

# 119. Conflict target

Podrá representar:

```text
columns
constraint
expression/index inference
```

según capacidades del modelo.

---

# 120. Conflict actions

```text
DO NOTHING
DO UPDATE
```

serán estrategias semánticas.

---

# 121. No native syntax in Builder

Nunca:

```php
$query .= ' ON CONFLICT ...';
```

desde Query Builder.

---

# 122. Upsert capabilities

```text
query.upsert
query.upsert.do_nothing
query.upsert.update
query.upsert.conflict_columns
query.upsert.constraint_target
query.upsert.returning
```

---

# 123. CTE

PostgreSQL CTE será capability de Query Engine.

---

# 124. CTE model

```text
CommonTableExpression
```

---

# 125. Recursive CTE

```text
query.cte.recursive
```

---

# 126. CTE materialization semantics

Cuando sea relevante, podrán existir capabilities para:

```text
materialized
not materialized
```

como hints/semántica avanzada.

---

# 127. Window functions

AST deberá soportar:

```text
WindowExpression
WindowDefinition
WindowFrame
```

---

# 128. Window capabilities

```text
query.window
query.window.partition
query.window.order
query.window.frame
```

---

# 129. Set operations

PostgreSQL Platform describirá:

```text
query.union
query.intersect
query.except
```

y variantes necesarias.

---

# 130. LATERAL

PostgreSQL deberá poder exponer:

```text
query.lateral
```

---

# 131. LateralJoin

Podrá modelarse semánticamente mediante:

```text
LateralSource
```

o:

```text
LateralJoin
```

---

# 132. DISTINCT ON

Podrá exponerse como capability PostgreSQL-specific:

```text
query.distinct_on
```

---

# 133. Portability marker

Queries usando `DISTINCT ON` podrán marcarse:

```text
PostgreSQL-specific
```

---

# 134. FILTER clause

Podrá representarse mediante:

```text
aggregate.filter
```

si el Query AST lo soporta.

---

# 135. Aggregate expressions

El modelo semántico podrá representar:

```text
AggregateExpression
├── function
├── arguments
├── distinct
├── filter
└── order
```

---

# 136. Locking architecture

PostgreSQL deberá describir capacidades de locking granularmente.

---

# 137. Lock modes

Modelo semántico preparado para:

```text
UPDATE
NO_KEY_UPDATE
SHARE
KEY_SHARE
```

cuando sea necesario.

---

# 138. Lock wait policy

```text
WAIT
NOWAIT
SKIP_LOCKED
```

---

# 139. LockClause

Conceptualmente:

```text
LockClause
├── mode
├── targets
└── waitPolicy
```

---

# 140. Capability validation

Planner verificará combinaciones válidas.

---

# 141. Lock target

PostgreSQL permite locks asociados a targets específicos.

El AST deberá poder representarlo si se expone públicamente.

---

# 142. Advisory locks

PostgreSQL advisory locks deberán tratarse como capability avanzada.

---

# 143. Advisory lock categories

Podrán distinguirse:

```text
session advisory lock
transaction advisory lock
shared advisory lock
exclusive advisory lock
```

---

# 144. Session lock contamination

Un advisory lock de sesión puede hacer una conexión:

```text
DIRTY
```

o:

```text
TAINTED
```

hasta liberación/reset verificable.

---

# 145. Transaction advisory lock

Podrá desaparecer al terminar la transacción según semántica del motor, pero el State System deberá conocerlo.

---

# 146. Transaction architecture

PostgreSQL deberá integrarse con el Transaction System genérico.

---

# 147. Transaction capabilities

```text
transaction
transaction.savepoint
transaction.read_only
transaction.deferrable
transaction.isolation.*
transaction.transactional_ddl
```

---

# 148. Isolation levels

El modelo portable:

```text
READ_UNCOMMITTED
READ_COMMITTED
REPEATABLE_READ
SERIALIZABLE
```

será interpretado mediante Platform semantics.

---

# 149. Semantic normalization

Si un nivel solicitado no tiene una distinción equivalente real en PostgreSQL, Platform deberá describirlo explícitamente.

---

# 150. Transaction characteristics

Podrán incluir:

```text
read/write
read only
deferrable
isolation
```

---

# 151. TransactionContext

Mantendrá estas decisiones por transacción.

---

# 152. No global transaction mode

Nunca:

```text
PostgreSqlPlatform::$readOnly = true
```

---

# 153. Savepoints

Nested transaction semantics podrán construirse sobre savepoints.

---

# 154. Savepoint naming

Los nombres deberán pasar por reglas seguras de identifier generation.

---

# 155. Failed transaction state

PostgreSQL tiene semántica importante cuando una sentencia falla dentro de una transacción.

---

# 156. Transaction state tracking

Connection State deberá distinguir:

```text
ACTIVE
FAILED
ROLLBACK_ONLY
COMMITTED
ROLLED_BACK
```

según la abstracción final.

---

# 157. Failed transaction handling

No continuar ejecutando operaciones como si la transacción estuviera limpia.

---

# 158. Recovery

Normalmente requerirá:

```text
rollback
```

o rollback a savepoint cuando sea válido.

---

# 159. TransactionManager responsibility

Debe conocer el estado abstracto.

Platform proporciona semántica.

Driver ejecuta primitives.

---

# 160. Serialization failures

Error classification deberá distinguir:

```text
SerializationFailure
```

---

# 161. Deadlocks

También:

```text
DeadlockFailure
```

---

# 162. Retry integration

Resilience System podrá utilizar:

```text
failure classification
+
transaction policy
+
idempotency
```

para decidir retry.

---

# 163. PostgreSQL SQLSTATE

PostgreSQL proporciona SQLSTATE estructurado que deberá aprovecharse.

---

# 164. Error pipeline

```text
PDOException
     │
     ▼
PdoPgSqlErrorAdapter
     │
     ▼
NativeDatabaseError
     │
     ▼
PostgreSqlFailureClassifier
     │
     ▼
DatabaseFailure
```

---

# 165. Failure categories

Al menos:

```text
UniqueConstraintFailure
ForeignKeyConstraintFailure
NotNullConstraintFailure
CheckConstraintFailure
ExclusionConstraintFailure
SerializationFailure
DeadlockFailure
LockTimeoutFailure
ConnectionFailure
AuthenticationFailure
SyntaxFailure
UndefinedTableFailure
UndefinedColumnFailure
```

---

# 166. Constraint metadata

Cuando esté disponible deberán extraerse:

```text
schema
table
column
constraint
datatype
```

de manera estructurada.

---

# 167. No regex-first architecture

No depender principalmente de parsear strings de error.

---

# 168. Error detail security

Los errores del servidor pueden contener valores sensibles.

---

# 169. Error redaction

Antes de:

```text
logs
telemetry
debug toolbar
production responses
```

deberán aplicarse políticas de redacción.

---

# 170. Schema architecture

PostgreSQL tendrá integración completa con:

```text
Schema Model
Schema AST
Schema Planner
Schema Compiler
Schema Introspection
Schema Diff
```

---

# 171. Schema object model

Debe contemplar:

```text
Schema
Table
Column
Index
ForeignKey
UniqueConstraint
CheckConstraint
ExclusionConstraint
Sequence
Enum
Domain
View
MaterializedView
```

según alcance y capacidades.

---

# 172. Views

Capabilities:

```text
schema.view
schema.view.create
schema.view.replace
schema.view.drop
```

---

# 173. Materialized views

Podrán exponerse mediante:

```text
schema.materialized_view
```

---

# 174. Refresh materialized view

Podrá ser operación administrativa/schema específica.

---

# 175. Tables

Table metadata deberá preservar:

```text
schema
name
persistence
partition metadata
tablespace
comment
platform options
```

cuando corresponda.

---

# 176. Temporary tables

Deberán modelarse cuidadosamente debido a su relación con:

```text
session state
pooling
persistent workers
```

---

# 177. Temporary object contamination

Una conexión con temporary objects no deberá volver al pool como limpia sin una estrategia segura.

---

# 178. Unlogged tables

Podrán exponerse como opción PostgreSQL-specific.

---

# 179. Portability

Una tabla `UNLOGGED` deberá marcar la definición como dependiente de PostgreSQL.

---

# 180. Columns

Column metadata deberá soportar:

```text
type
nullable
default
identity
generated
collation
comment
compression/options where applicable
```

---

# 181. Generated columns

Capability:

```text
schema.generated_column
```

---

# 182. Generated expression

Schema AST almacenará una expresión estructurada cuando sea posible.

---

# 183. Default expressions

Debe distinguirse:

```text
literal
expression
sequence-generated
identity-generated
NULL
none
```

---

# 184. Serial pseudo-types

La introspección no deberá confundir una convenience declaration con el modelo semántico final.

---

# 185. Canonicalization

Una columna creada mediante una convenience syntax podrá normalizarse a:

```text
integer/bigint
+
sequence/identity semantics
```

según corresponda.

---

# 186. Schema diff stability

Objetivo:

```text
create
→ introspect
→ canonicalize
→ diff
=
no change
```

---

# 187. Index architecture

PostgreSQL posee un sistema de índices avanzado.

---

# 188. IndexDefinition

Debe soportar conceptualmente:

```text
name
columns
expressions
unique
method
predicate
included columns
operator classes
collations
sort order
null ordering
storage parameters
tablespace
```

según el nivel de soporte.

---

# 189. Basic index capabilities

```text
schema.index
schema.index.unique
schema.index.expression
schema.index.partial
schema.index.include
```

---

# 190. Index methods

Arquitectura preparada para:

```text
btree
hash
gist
spgist
gin
brin
```

sin codificarlos en el modelo portable central.

---

# 191. Index method abstraction

Podrá existir:

```text
IndexMethod
```

como value object extensible.

---

# 192. Partial indexes

Schema AST deberá soportar:

```text
predicate
```

---

# 193. Expression indexes

Deberán usar:

```text
Expression AST
```

cuando sea posible.

---

# 194. Included columns

Deben distinguirse de key columns.

---

# 195. Operator classes

Serán metadata PostgreSQL-specific.

---

# 196. Concurrent index creation

Migration System deberá reconocer:

```text
CREATE INDEX CONCURRENTLY
```

como estrategia específica.

---

# 197. Concurrent index capability

```text
migration.index.concurrent
```

---

# 198. Transaction restriction

Migration Planner deberá conocer las restricciones transaccionales asociadas a operaciones concurrentes.

---

# 199. Migration execution segmentation

Una migration podrá dividirse:

```text
Transactional Segment
Non-Transactional Segment
Transactional Segment
```

cuando la plataforma lo requiera.

---

# 200. Migration Planner

No será simplemente:

```text
wrap everything in transaction
```

---

# 201. Constraints

PostgreSQL deberá soportar:

```text
Primary Key
Unique
Foreign Key
Check
Exclusion
Not Null
```

según el modelo.

---

# 202. Deferrable constraints

Capability:

```text
schema.constraint.deferrable
```

---

# 203. Initially deferred

Metadata:

```text
deferrable
initiallyDeferred
```

---

# 204. Constraint validation

PostgreSQL permite escenarios donde una constraint puede existir y validarse posteriormente.

---

# 205. Constraint validation state

Schema metadata podrá necesitar:

```text
validated
```

---

# 206. Migration safety

Esto permite estrategias de migration más seguras sobre tablas grandes.

---

# 207. Foreign keys

Debe soportarse:

```text
on delete
on update
match type
deferrable
initial state
validation state
```

cuando esté disponible.

---

# 208. Check constraints

Expresión deberá preservarse estructuralmente cuando sea posible.

---

# 209. Exclusion constraints

Podrán modelarse como extensión PostgreSQL-specific.

---

# 210. Constraint names

Los nombres deberán pasar por:

```text
Identifier
```

y reglas de longitud/normalización.

---

# 211. Identifier length

Platform capabilities deberán exponer límites relevantes.

---

# 212. Auto-generated names

Schema System deberá producir nombres:

```text
deterministic
stable
collision-resistant
platform-compatible
```

---

# 213. Identifier truncation

No usar truncación ingenua que produzca colisiones.

---

# 214. Name generator

Podrá utilizar:

```text
prefix
semantic tokens
stable hash suffix
```

---

# 215. DDL transactional semantics

PostgreSQL ofrece capacidades transaccionales importantes para DDL.

---

# 216. Capability

```text
transaction.transactional_ddl
```

---

# 217. Per-operation exceptions

No asumir que toda operación administrativa/DDL puede ejecutarse dentro de la misma transacción.

---

# 218. MigrationOperation capability requirements

Cada operación podrá declarar:

```text
requiresTransaction
allowsTransaction
forbidsTransaction
```

---

# 219. MigrationPlan

Podrá contener:

```text
MigrationPlan
├── TransactionalBatch
├── NonTransactionalOperation
└── TransactionalBatch
```

---

# 220. Zero-downtime migration

PostgreSQL capabilities avanzadas podrán aprovecharse para:

```text
concurrent indexes
deferred validation
multi-step constraint creation
online-safe transformations
```

cuando sea semánticamente correcto.

---

# 221. Migration safety planner

Evaluará:

```text
operation
platform
version
capabilities
table metadata
deployment policy
```

---

# 222. Schema introspection

Componente:

```text
PostgreSqlSchemaIntrospector
```

---

# 223. Introspection sources

Deberá utilizar interfaces/catalog metadata de PostgreSQL de forma encapsulada.

---

# 224. Introspection responsibilities

Descubrir:

```text
schemas
tables
columns
types
defaults
identity
generated columns
indexes
constraints
foreign keys
sequences
enums
domains
views
```

según alcance.

---

# 225. Introspection output

Siempre:

```text
Canonical VoltStack Schema Model
```

---

# 226. Native metadata

Detalles PostgreSQL-specific podrán vivir en:

```text
PostgreSqlSchemaMetadata
```

---

# 227. Schema canonicalizer

```text
PostgreSqlSchemaCanonicalizer
```

normalizará representaciones equivalentes.

---

# 228. Schema comparator

El comparator genérico utilizará metadata canonicalizada.

---

# 229. No endless migrations

El sistema deberá evitar:

```text
migration generated
→ applied
→ introspected
→ same migration generated again
```

por diferencias meramente sintácticas.

---

# 230. Sequence introspection

Sequences deberán asociarse correctamente cuando formen parte de generación de IDs.

---

# 231. Identity introspection

Identity columns deberán diferenciarse de sequence/default patterns.

---

# 232. Enum introspection

Deberá preservar:

```text
schema
name
values
order
```

---

# 233. Domain introspection

Deberá preservar:

```text
base type
constraints
default
nullable semantics
```

---

# 234. Search path and introspection

Schema introspection no deberá depender ciegamente del `search_path`.

---

# 235. Explicit catalog querying

La introspección deberá poder seleccionar schemas explícitamente.

---

# 236. System schemas

Por defecto podrán excluirse:

```text
pg_catalog
information_schema
```

y otros namespaces internos según policy.

---

# 237. Schema filters

Podrá configurarse:

```text
include
exclude
```

mediante patrones tipados.

---

# 238. Partitioning

La arquitectura deberá estar preparada para PostgreSQL partitioning.

---

# 239. Partition capability

```text
schema.partition
```

---

# 240. Partition model

Podrá evolucionar hacia:

```text
PartitionDefinition
├── strategy
├── key
├── bounds
└── options
```

---

# 241. Partition strategies

Conceptualmente:

```text
RANGE
LIST
HASH
```

según capabilities.

---

# 242. Partitioning scope

No será requisito obligatorio del ORM básico.

Será una capacidad avanzada de Schema/Platform.

---

# 243. Table inheritance

PostgreSQL-specific table inheritance podrá quedar como extensión avanzada separada de ORM inheritance.

---

# 244. Database inheritance ≠ ORM inheritance

Nunca confundir ambos conceptos.

---

# 245. Full-text search

PostgreSQL full-text search deberá tratarse como capability especializada.

---

# 246. Full-text types

Arquitectura preparada para:

```text
tsvector
tsquery
```

---

# 247. Full-text expressions

Podrán integrarse posteriormente mediante:

```text
Semantic Search AST
```

o extensión PostgreSQL-specific.

---

# 248. Search portability

No asumir equivalencia exacta con MySQL/MariaDB full-text search.

---

# 249. Extensions

PostgreSQL permite extensiones del servidor.

---

# 250. Server extension discovery

Podrá existir:

```text
PostgreSqlExtensionDiscovery
```

cuando una capability dependa de una extensión instalada.

---

# 251. Lazy extension discovery

No consultar todas las extensiones en cada conexión.

---

# 252. Extension-dependent capabilities

Ejemplo conceptual:

```text
server extension
      │
      ▼
Capability Provider
      │
      ▼
EffectiveCapabilitySet
```

---

# 253. VoltStack extension ≠ PostgreSQL extension

Muy importante:

```text
VoltStack Database Extension
≠
PostgreSQL Server Extension
```

---

# 254. Terminología

Internamente podrá distinguirse:

```text
FrameworkExtension
ServerExtension
```

---

# 255. Custom PostgreSQL types

El Type Registry deberá permitir registrar mappings para tipos provenientes de server extensions.

---

# 256. Unknown native type

Introspection no deberá fallar necesariamente ante un tipo desconocido.

---

# 257. UnknownTypeMetadata

Podrá representarse como:

```text
UnknownNativeType
```

con metadata preservada.

---

# 258. Strict mapping

ORM sí podrá exigir mapping explícito antes de hidratar/persistir ese tipo.

---

# 259. Session architecture

PostgreSQL posee abundante estado de sesión.

---

# 260. SessionProfile

Podrá incluir:

```text
timezone
search_path
application_name
role
statement settings
transaction defaults
locale-related settings
```

según configuración.

---

# 261. Session initialization

```text
Native Connection
      │
      ▼
PostgreSqlSessionInitializer
      │
      ▼
Known Baseline
```

---

# 262. Session state tracker

Deberá rastrear cambios relevantes.

---

# 263. Role

PostgreSQL permite cambiar role/session authorization en determinados contextos.

Esto tiene implicaciones de seguridad para pooling.

---

# 264. Role contamination

Nunca:

```text
Request A
SET ROLE tenant_a

Request B
inherits tenant_a
```

---

# 265. Reset strategy

Antes de devolver una conexión al pool deberá garantizarse:

```text
transaction clean
role clean
search_path clean
timezone clean
temporary state clean
cursor state clean
advisory locks clean
session settings clean
```

o descartarse la conexión.

---

# 266. Reset planner

```text
Connection State
      │
      ▼
PostgreSqlResetPlanner
      │
      ▼
Reset Operations
      │
      ▼
Known Baseline
```

---

# 267. Reset strength

Podrá clasificarse:

```text
NONE
PARTIAL
BASELINE
FULL
```

---

# 268. Unknown state

Si no puede demostrarse que el recurso está limpio:

```text
discard
```

será preferible a reutilización insegura.

---

# 269. Native access

Acceso directo al handle nativo podrá marcar:

```text
TAINTED
```

la conexión.

---

# 270. Temporary tables

Son especialmente relevantes para pool reuse.

---

# 271. Prepared resources

Recursos server-side o driver-side deberán tener ownership claro.

---

# 272. Cursors

Un cursor abierto puede mantener:

```text
connection lease
transaction
server resources
```

---

# 273. Streaming Result

```text
StreamingResult
      │
      ▼
ConnectionLease
```

hasta agotarse/cerrarse.

---

# 274. Buffered Result

Podrá liberar el lease antes cuando sea seguro.

---

# 275. Persistent runtime

PostgreSQL integration deberá ser segura para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 276. FrankenPHP model

```text
Worker
  │
  ├── Request A
  │     └── PostgreSQL lease
  │
  ├── Database RESET
  │
  └── Request B
        └── PostgreSQL lease
```

---

# 277. Worker-safe services

Podrán sobrevivir:

```text
PostgreSqlPlatform
PostgreSqlDialect
Capability rules
Type mapping descriptors
Compiler strategies
Compiled metadata
```

si son inmutables/stateless.

---

# 278. Request-scoped state

No sobrevivirá:

```text
current transaction
current tenant
current search_path override
current role
open cursor
current connection lease
temporary query state
```

---

# 279. OpenSwoole

No asumir que un worker procesa una sola operación simultáneamente.

---

# 280. Coroutine safety

Estado contextual deberá estar asociado a:

```text
ExecutionScope
```

no a variables estáticas.

---

# 281. Physical connection sharing

Por defecto:

```text
one active lease
=
exclusive physical connection ownership
```

---

# 282. Multiplexing

Sólo podrá habilitarse si un Driver futuro lo soporta explícitamente y las semánticas son seguras.

---

# 283. Connection pool

PoolKey deberá incluir elementos suficientes para impedir reutilización incompatible.

---

# 284. Pool compatibility inputs

Conceptualmente:

```text
DriverId
Endpoint
Database
CredentialGeneration
SessionProfileFingerprint
ConnectionDefinitionFingerprint
PlatformCompatibility
```

---

# 285. Platform version changes

Un rolling upgrade podrá producir conexiones hacia versiones diferentes.

---

# 286. Capability compatibility

Pool/Topology deberán utilizar:

```text
CapabilityFingerprint
```

o requisitos equivalentes cuando sea necesario.

---

# 287. Credential rotation

Nueva:

```text
CredentialGeneration
```

deberá provocar drenaje controlado de recursos antiguos.

---

# 288. Session profile changes

Cambio de:

```text
search_path
timezone
role baseline
```

puede requerir nueva generación de pool/configuración.

---

# 289. Read/write topology

PostgreSQL Platform no implementará routing.

---

# 290. Topology System

Será responsable de:

```text
primary selection
replica selection
lag awareness
failover
load balancing
```

---

# 291. Platform contribution

Platform aporta:

```text
target capabilities
server identity
read/write characteristics
transaction semantics
```

---

# 292. Replica compatibility

No asumir:

```text
same cluster
=
same capability profile
```

durante upgrades o configuraciones heterogéneas.

---

# 293. Capability-aware routing

Una operación podrá declarar:

```text
ConnectionRequirement
├── intent: READ
└── requiredCapabilities
```

---

# 294. Topology selection

Sólo targets compatibles podrán seleccionarse.

---

# 295. Transaction pinning

Una vez iniciada una transacción:

```text
Transaction
      │
      ▼
Pinned ConnectionLease
```

---

# 296. No replica switching

Dentro de una transacción no se cambiará silenciosamente de target.

---

# 297. Read-after-write

Sticky primary/consistency policies pertenecerán a Topology/Connection Routing.

---

# 298. Platform does not implement consistency routing

Mantener responsabilidades separadas.

---

# 299. Bulk operations

PostgreSQL Platform deberá describir capacidades para:

```text
multi-row insert
bulk update strategies
bulk delete
copy-like import/export
```

---

# 300. COPY

PostgreSQL COPY podrá modelarse como capability avanzada:

```text
bulk.copy
```

---

# 301. COPY ≠ generic INSERT

Debe tratarse como una estrategia especializada.

---

# 302. Import system

Podrá seleccionar:

```text
Prepared INSERT batches
```

o:

```text
PostgreSQL COPY
```

según capabilities/policy.

---

# 303. Driver support

Una estrategia COPY puede requerir capacidades adicionales del Driver.

---

# 304. Effective support

```text
Platform supports COPY
+
Driver supports required transport/API
=
usable COPY strategy
```

---

# 305. Query cancellation

Deberá modelarse mediante interacción:

```text
Driver Capability
+
Execution System
+
Platform semantics
```

---

# 306. Statement timeout

PostgreSQL ofrece mecanismos de sesión/transacción relevantes.

VoltStack deberá modelarlos sin contaminar conexiones posteriores.

---

# 307. Timeout scope

Distinguir:

```text
connect timeout
pool acquire timeout
statement timeout
lock timeout
transaction timeout
```

---

# 308. Scoped timeout

Un timeout temporal deberá restaurarse antes de devolver la conexión al pool.

---

# 309. Application name

`application_name` podrá configurarse mediante Session Profile.

---

# 310. Runtime metadata

Podrá incluir información controlada como:

```text
VoltStack
application identifier
worker type
```

sin PII innecesaria.

---

# 311. Security

La integración PostgreSQL deberá respetar:

```text
prepared statements
parameter binding
identifier safety
TLS
credential isolation
session role isolation
search_path safety
error redaction
```

---

# 312. Search path security

Objetos no calificados y `search_path` mal controlado pueden afectar resolución de objetos.

VoltStack deberá favorecer configuración explícita y segura.

---

# 313. Dynamic identifiers

Nunca concatenar directamente:

```php
$sql = 'SELECT * FROM ' . $userInput;
```

---

# 314. Safe identifiers

Usar:

```text
Identifier
→ validation
→ PostgreSqlDialect quoting
```

---

# 315. Credentials

`PostgreSqlPlatform` nunca recibirá passwords salvo que una operación de muy bajo nivel explícitamente lo requiera, lo cual deberá evitarse.

---

# 316. Credential ownership

```text
CredentialReference
      │
      ▼
CredentialResolver
      │
      ▼
Connection/Driver Boundary
```

---

# 317. TLS

TLS será responsabilidad principal de:

```text
Connection Security
+
Driver
```

---

# 318. TLS capabilities

Platform podrá aportar información adicional cuando sea necesaria, pero no almacenará secretos/certificados privados.

---

# 319. Native options

Opciones PDO/pgsql específicas deberán vivir en configuración de Driver, no Platform.

---

# 320. Platform options

Opciones semánticas PostgreSQL-specific pertenecerán a Platform configuration.

---

# 321. Strict option validation

Unknown options deberán fallar en modo estricto.

---

# 322. Query plan cache

CompiledQuery cache deberá considerar:

```text
QueryFingerprint
DialectFingerprint
CapabilityFingerprint
CompilerVersion
```

---

# 323. Platform fingerprint

Podrá incluir:

```text
PlatformId
ServerVersion
RelevantSemanticConfiguration
```

---

# 324. Capability fingerprint

Representará el conjunto efectivo de capacidades.

---

# 325. Schema metadata fingerprint

Separado del CapabilityFingerprint.

---

# 326. Session fingerprint

Separado igualmente.

---

# 327. No universal fingerprint

Evitar:

```text
PostgresFingerprint
```

utilizado indiscriminadamente para todos los caches.

---

# 328. Query preparation

Prepared statements seguirán siendo el camino normal.

---

# 329. Placeholder model

AST utilizará parámetros abstractos.

---

# 330. Compiler

Produce:

```text
CompiledQuery
├── sql
├── bindings
├── parameterTypes
└── executionMetadata
```

---

# 331. Driver

Transformará bindings al mecanismo nativo correspondiente.

---

# 332. PostgreSQL type binding

Algunos tipos avanzados podrán requerir:

```text
Type Binder
```

específico.

---

# 333. Type binder responsibility

Convertir:

```text
VoltStack Value
→
Driver-compatible representation
```

sin introducir ORM awareness.

---

# 334. Hydration

Result Hydrator utilizará Type System.

---

# 335. PostgreSqlPlatform does not hydrate entities

Nunca:

```text
PostgreSqlPlatform
→ Entity
```

---

# 336. Correct hydration flow

```text
Native Result
    │
    ▼
Result
    │
    ▼
Type Conversion
    │
    ▼
Hydration
    │
    ▼
Entity
```

---

# 337. Query optimizer

VoltStack Query Optimizer seguirá siendo vendor-neutral cuando sea posible.

---

# 338. PostgreSQL-specific optimization

Sólo se añadirá mediante reglas explícitas cuando:

```text
semantics
+
capabilities
```

lo justifiquen.

---

# 339. No database optimizer replacement

VoltStack no intentará reemplazar el optimizador interno de PostgreSQL.

---

# 340. Framework optimization scope

VoltStack puede optimizar:

```text
redundant predicates
duplicate framework queries
eager-loading patterns
batch planning
query shape
portable rewrites
N+1
```

---

# 341. EXPLAIN

PostgreSQL EXPLAIN podrá integrarse con Profiler/Telemetry.

---

# 342. Explain capability

```text
diagnostics.explain
```

---

# 343. Explain analyze

Separado:

```text
diagnostics.explain_analyze
```

debido a que ejecuta la operación.

---

# 344. Safety

`EXPLAIN ANALYZE` no deberá ejecutarse automáticamente sobre operaciones destructivas.

---

# 345. Telemetry integration

Podrá registrar:

```text
platform = postgresql
driver = pdo.pgsql
connection role
operation
duration
rows
failure class
```

según política.

---

# 346. High-cardinality metadata

Evitar labels métricas como:

```text
raw SQL
tenant ID
table names dinámicos ilimitados
user IDs
```

---

# 347. SQL redaction

Query telemetry deberá respetar la política de:

```text
SQL normalization
parameter redaction
sensitive-column redaction
```

---

# 348. PostgreSQL notices

Notices/warnings podrán integrarse mediante mecanismo específico del Driver cuando esté disponible.

---

# 349. Notice handling

No deberán imprimirse directamente.

---

# 350. Notice pipeline

```text
Native Notice
     │
     ▼
Driver Adapter
     │
     ▼
Database Diagnostic Event
```

---

# 351. Health checks

Niveles posibles:

```text
BASIC
CONNECTIVITY
PLATFORM
CAPABILITY
TRANSACTIONAL
```

---

# 352. Connectivity

Verifica acceso al servidor.

---

# 353. Platform health

Verifica:

```text
PostgreSQL identity
version
compatibility
```

---

# 354. Capability health

Verifica requisitos declarados por la aplicación.

---

# 355. Migration readiness

Podrá verificar:

```text
target version
required extensions
DDL capabilities
permissions
```

antes del deployment.

---

# 356. Permission awareness

Database Platform podrá detectar errores de permisos, pero no sustituirá el Authorization System de la aplicación.

---

# 357. Database privilege ≠ application authorization

Mantener:

```text
PostgreSQL privilege
≠
VoltStack Authorization Policy
```

---

# 358. Administration

Operaciones administrativas deberán mantenerse separadas del Query Builder de aplicación.

---

# 359. Admin API

Podrá existir posteriormente:

```text
DatabaseAdministration
```

---

# 360. PostgreSQL admin capabilities

Podrán incluir:

```text
database creation
schema administration
extension administration
maintenance
statistics
vacuum-related diagnostics
```

según alcance.

---

# 361. No admin privileges assumption

Una conexión de aplicación normal no deberá requerir privilegios administrativos.

---

# 362. Migration connection

Podrá utilizarse una conexión con:

```text
ConnectionIntent::MIGRATION
```

y credenciales diferentes.

---

# 363. Admin connection

Igualmente:

```text
ConnectionIntent::ADMIN
```

---

# 364. Least privilege

La arquitectura deberá facilitar:

```text
runtime credentials
migration credentials
administrative credentials
```

separadas.

---

# 365. Views and ORM

Una View podrá utilizarse como fuente de lectura.

---

# 366. Read-only entity mapping

ORM podrá permitir entidades/modelos:

```text
read-only
```

sobre views.

---

# 367. Materialized views

Podrán exponerse como:

```text
read model
```

sin asumir persistencia convencional.

---

# 368. Generated values

Persistence Planner deberá conocer:

```text
identity
sequence
default expression
generated column
trigger-generated value
```

cuando se requiera recuperar valores.

---

# 369. Trigger-generated values

No asumir que sólo las identity columns generan valores.

---

# 370. Returning strategy

Cuando sea necesario:

```text
INSERT/UPDATE
+
RETURNING
```

podrá recuperar valores generados.

---

# 371. Trigger awareness

ORM no necesita conocer la implementación del trigger.

Sólo necesita saber qué valores deben refrescarse.

---

# 372. GeneratedPropertyMetadata

Podrá describir:

```text
on insert
on update
always
```

---

# 373. Persistence refresh

Persistence Planner decide:

```text
RETURNING
refresh query
native mechanism
```

según capabilities.

---

# 374. Batch inserts

PostgreSQL Platform deberá exponer límites relevantes.

---

# 375. Parameter limits

Planner podrá utilizar:

```text
parameter.max_count
```

para dividir batches.

---

# 376. SQL size

También podrá considerar:

```text
statement size
memory policy
application batch policy
```

---

# 377. No fixed universal batch

Evitar:

```text
batch = 1000
```

como verdad universal.

---

# 378. COPY optimization

Para grandes datasets:

```text
Bulk API
   │
   ▼
Bulk Planner
   │
   ├── generic batch insert
   └── PostgreSQL COPY strategy
```

---

# 379. Portability

COPY será una optimización, no requisito del ORM básico.

---

# 380. Extension model

VoltStack extensions podrán añadir capacidades PostgreSQL.

---

# 381. Extension points

```text
TypeMapping
CapabilityProvider
SemanticFunction
QueryCompilerExtension
SchemaExtension
MigrationStrategy
FailureClassifierExtension
```

---

# 382. PostgreSQL server extensions

Una integración con una extensión de servidor podrá empaquetarse como:

```text
VoltStack Database Extension
```

que detecta:

```text
PostgreSQL Server Extension
```

---

# 383. Example architecture

```text
VoltStack Extension
      │
      ▼
Server Extension Detector
      │
      ▼
Capability Provider
      │
      ▼
Type / Query / Schema Extensions
```

---

# 384. No automatic trust

La presencia de una extensión no deberá inferirse sin discovery/configuración verificable.

---

# 385. Extension version

Podrá participar en:

```text
CapabilityFingerprint
```

si afecta semántica.

---

# 386. Namespaces propuestos

```text
VoltStack\Quantum\Database\Platform\PostgreSql
```

---

# 387. Estructura propuesta

```text
Platform/
└── PostgreSql/
    ├── PostgreSqlPlatform.php
    ├── PostgreSqlPlatformDescriptor.php
    ├── PostgreSqlPlatformSupportPolicy.php
    │
    ├── Discovery/
    │   ├── PostgreSqlServerDiscovery.php
    │   ├── PostgreSqlServerMetadata.php
    │   └── PostgreSqlVersionParser.php
    │
    ├── Capability/
    │   ├── PostgreSqlCapabilityProvider.php
    │   ├── PostgreSqlCapabilityRules.php
    │   └── PostgreSqlCapabilitySnapshotFactory.php
    │
    ├── Type/
    │   ├── PostgreSqlTypeMapping.php
    │   ├── PostgreSqlTypeCanonicalizer.php
    │   └── PostgreSqlTypeBinder.php
    │
    ├── Schema/
    │   ├── PostgreSqlSchemaSemantics.php
    │   ├── PostgreSqlSchemaIntrospector.php
    │   ├── PostgreSqlSchemaCanonicalizer.php
    │   └── PostgreSqlSchemaMetadata.php
    │
    ├── Transaction/
    │   └── PostgreSqlTransactionSemantics.php
    │
    ├── Session/
    │   ├── PostgreSqlSessionInitializer.php
    │   ├── PostgreSqlSessionStateTracker.php
    │   └── PostgreSqlResetPlanner.php
    │
    ├── Error/
    │   └── PostgreSqlFailureClassifier.php
    │
    ├── Planning/
    │   ├── PostgreSqlQueryStrategies.php
    │   └── PostgreSqlMigrationStrategies.php
    │
    ├── Extension/
    └── Testing/
```

---

# 388. Dialect namespace

Preferentemente separado:

```text
VoltStack\Quantum\Database\Dialect\PostgreSql
```

---

# 389. Driver namespace

```text
VoltStack\Quantum\Database\Driver\Pdo\PostgreSql
```

---

# 390. Separation visible

```text
Driver/Pdo/PostgreSql
      │
      └── transport

Dialect/PostgreSql
      │
      └── syntax

Platform/PostgreSql
      │
      └── semantics/capabilities
```

---

# 391. Dependency direction

```text
PostgreSqlPlatform
        │
        ▼
Platform Contracts
Capability Contracts
Schema Contracts
Type Contracts
```

Nunca hacia:

```text
EntityManager
Repository
UnitOfWork
HTTP
Controller
Application
```

---

# 392. Architecture tests

Deberán impedir dependencias prohibidas.

---

# 393. Static vendor checks

Capas superiores no deberán contener:

```php
if ($platform->name() === 'postgresql') {
}
```

salvo código explícitamente definido como extensión PostgreSQL-specific.

---

# 394. Capability checks

Preferir:

```php
if ($capabilities->supports($requirement)) {
}
```

---

# 395. Semantic strategy selection

Aún mejor:

```text
Feature Requirement
      │
      ▼
Planner Strategy Resolver
```

en vez de condicionales distribuidos.

---

# 396. Testing strategy

PostgreSQL deberá tener:

```text
Unit Tests
Integration Tests
Conformance Tests
Capability Tests
Schema Round-Trip Tests
Migration Tests
Transaction Tests
Pool/Reset Tests
Persistent Runtime Tests
Concurrency Tests
Security Tests
Performance Tests
```

---

# 397. PostgreSqlPlatformConformanceSuite

Componente conceptual:

```text
PostgreSqlPlatformConformanceSuite
```

---

# 398. Capability conformance

Toda capability:

```text
NATIVE
```

deberá poder demostrarse mediante test.

---

# 399. Emulation conformance

Toda capability:

```text
EMULATED
```

deberá demostrar equivalencia semántica.

---

# 400. Version matrix

CI deberá probar múltiples versiones soportadas.

---

# 401. Schema round-trip

Prueba fundamental:

```text
Schema Model
    │
    ▼
CREATE
    │
    ▼
PostgreSQL
    │
    ▼
INTROSPECT
    │
    ▼
Canonical Schema
    │
    ▼
DIFF
    │
    ▼
EMPTY
```

---

# 402. Complex schema tests

Cubrir:

```text
schemas
sequences
identity
enums
domains
indexes
partial indexes
expression indexes
foreign keys
check constraints
generated columns
views
```

según soporte.

---

# 403. Transaction tests

Cubrir:

```text
commit
rollback
savepoints
failed transaction state
serialization failures
deadlocks
read-only
isolation
cleanup
```

---

# 404. Session tests

Cubrir:

```text
search_path
timezone
role
statement settings
temporary objects
advisory locks
```

---

# 405. Persistent request test

```text
Request A
    │
    ├── SET ROLE
    ├── change search_path
    ├── change timezone
    └── execute
         │
         ▼
RESET
         │
         ▼
Request B
         │
         ▼
must receive baseline state
```

---

# 406. Connection pool tests

Verificar:

```text
clean release
tainted discard
failed reset discard
open transaction rejection
open cursor pinning
generation retirement
credential rotation
```

---

# 407. Concurrency tests

Especialmente:

```text
parallel execution scopes
Fibers
OpenSwoole-like coroutines
```

---

# 408. Multitenancy tests

Cuando el paquete esté instalado:

```text
Tenant A → schema_a
Tenant B → schema_b
```

sin contaminación.

---

# 409. Security tests

Verificar:

```text
identifier injection protection
credential redaction
TLS configuration
error detail redaction
role isolation
search_path isolation
```

---

# 410. Performance tests

Medir:

```text
platform discovery
capability resolution
schema introspection
query compilation
type conversion
session reset
pool reuse
```

---

# 411. Platform discovery caching

Evitar discovery completo por query.

---

# 412. Capability snapshot caching

Podrá reutilizarse mientras:

```text
target identity
+
version
+
relevant configuration
+
driver capabilities
```

no cambien.

---

# 413. Server upgrade invalidation

Un cambio de versión deberá invalidar snapshots incompatibles.

---

# 414. Extension change invalidation

Cambios en extensiones relevantes también.

---

# 415. Configuration generation

Cambios semánticos en conexión/session profile podrán crear nueva generación.

---

# 416. Observability

Podrán exponerse diagnósticos como:

```text
Connection: primary
Driver: pdo.pgsql
Platform: postgresql
Version: <resolved>
Schema: <configured/current>
Capability Profile: <fingerprint>
Session Profile: <fingerprint>
```

---

# 417. CLI

Comando futuro:

```text
php volt database:platform primary
```

---

# 418. Explain capabilities

```text
php volt database:capability --connection=primary
```

---

# 419. Explain specific capability

```text
php volt database:capability query.returning.insert --connection=primary
```

---

# 420. Explain schema

Podrá existir:

```text
php volt database:schema:inspect
```

---

# 421. Explain migration

```text
php volt database:migration:plan
```

podrá mostrar:

```text
transactional operations
non-transactional operations
locks
capability requirements
safety warnings
```

---

# 422. Secure diagnostics

Ningún comando deberá mostrar:

```text
password
credential value
private key
secret DSN
```

---

# 423. Architectural invariants

## DB-PG-001

PostgreSQL será una Platform independiente.

## DB-PG-002

El Driver inicial podrá ser `pdo.pgsql`.

## DB-PG-003

Driver y Platform permanecerán separados.

## DB-PG-004

Dialect y Platform permanecerán separados.

## DB-PG-005

PostgreSqlPlatform será preferentemente inmutable y stateless.

## DB-PG-006

PostgreSqlPlatform no almacenará current Connection.

## DB-PG-007

PostgreSqlPlatform no almacenará current Transaction.

## DB-PG-008

PostgreSqlPlatform no almacenará current Tenant.

## DB-PG-009

ServerVersion pertenecerá al contexto de plataforma resuelta.

## DB-PG-010

Capabilities serán version-aware.

## DB-PG-011

Query Builder permanecerá vendor-neutral.

## DB-PG-012

ORM permanecerá vendor-neutral para features portables.

## DB-PG-013

Planner seleccionará estrategias PostgreSQL.

## DB-PG-014

Dialect únicamente traducirá sintaxis.

## DB-PG-015

SQL Compiler no ejecutará SQL.

## DB-PG-016

Platform no ejecutará queries de aplicación.

## DB-PG-017

Schemas PostgreSQL serán first-class.

## DB-PG-018

Database y Schema no serán tratados como sinónimos.

## DB-PG-019

`search_path` será estado de sesión.

## DB-PG-020

`search_path` no podrá filtrarse entre execution scopes.

## DB-PG-021

Role de sesión no podrá filtrarse entre execution scopes.

## DB-PG-022

Timezone no podrá filtrarse entre execution scopes.

## DB-PG-023

Temporary state deberá limpiarse o provocar descarte.

## DB-PG-024

Advisory locks deberán participar en state tracking.

## DB-PG-025

Native access podrá marcar la conexión TAINTED.

## DB-PG-026

Una conexión con transacción activa no volverá al pool.

## DB-PG-027

Una conexión con cursor incompatible abierto no volverá al pool.

## DB-PG-028

Failed transaction state deberá ser reconocido.

## DB-PG-029

Retry policy no pertenecerá a Platform.

## DB-PG-030

SQLSTATE se normalizará centralmente.

## DB-PG-031

Error detail será tratado como potencialmente sensible.

## DB-PG-032

RETURNING será capability granular.

## DB-PG-033

RETURNING no será confundido con generated-key API.

## DB-PG-034

ON CONFLICT se representará semánticamente.

## DB-PG-035

CTE permanecerá en AST portable.

## DB-PG-036

Window functions permanecerán en AST portable.

## DB-PG-037

PostgreSQL-specific syntax será introducida en Compiler/Dialect.

## DB-PG-038

JSON y JSONB serán semánticamente distinguibles.

## DB-PG-039

PHP arrays no implicarán PostgreSQL arrays automáticamente.

## DB-PG-040

UUID storage y UUID generation serán conceptos distintos.

## DB-PG-041

Enum database type y PHP Enum serán conceptos distintos.

## DB-PG-042

Sequences e Identity serán conceptos distintos.

## DB-PG-043

Generated values serán resueltos por Persistence Planner.

## DB-PG-044

Schema introspection producirá Canonical Schema Model.

## DB-PG-045

Native metadata podrá preservarse separadamente.

## DB-PG-046

Schema canonicalization evitará migrations espurias.

## DB-PG-047

Partial indexes serán capability-driven.

## DB-PG-048

Expression indexes serán capability-driven.

## DB-PG-049

Concurrent index operations serán migration strategies.

## DB-PG-050

Migration Planner conocerá operaciones que prohíben transacción.

## DB-PG-051

Transactional DDL será capability-driven.

## DB-PG-052

Deferrable constraints serán representables.

## DB-PG-053

Constraint validation state podrá preservarse.

## DB-PG-054

Server extensions y VoltStack extensions serán conceptos diferentes.

## DB-PG-055

Tipos desconocidos podrán preservarse sin inventar mappings.

## DB-PG-056

ORM requerirá mapping conocido antes de hidratar tipos avanzados.

## DB-PG-057

CapabilitySnapshot será target-specific.

## DB-PG-058

Un worker podrá manejar múltiples versiones/targets.

## DB-PG-059

No existirá un current PostgreSQL platform global.

## DB-PG-060

Pooling utilizará compatibilidad explícita.

## DB-PG-061

Credential rotation producirá nueva generación cuando corresponda.

## DB-PG-062

Failover deberá verificar requirements/capabilities.

## DB-PG-063

Replicas no se asumirán homogéneas.

## DB-PG-064

Transaction affinity impedirá cambios silenciosos de target.

## DB-PG-065

COPY será una estrategia especializada, no semántica básica de INSERT.

## DB-PG-066

Driver capabilities podrán restringir features de Platform.

## DB-PG-067

Timeouts tendrán scopes diferenciados.

## DB-PG-068

Session-scoped settings temporales deberán restaurarse.

## DB-PG-069

Platform options y Driver options estarán separados.

## DB-PG-070

Secrets nunca formarán parte de fingerprints.

## DB-PG-071

Compiled Query Cache incluirá capability/dialect compatibility.

## DB-PG-072

Schema cache tendrá su propia invalidación.

## DB-PG-073

Session fingerprint será independiente.

## DB-PG-074

Cada capability NATIVE tendrá conformance testing.

## DB-PG-075

Cada capability EMULATED deberá probar equivalencia.

## DB-PG-076

Persistent-runtime isolation será obligatoria.

## DB-PG-077

FrankenPHP será el runtime persistente de referencia inicial.

## DB-PG-078

RoadRunner y OpenSwoole utilizarán la misma arquitectura neutral.

## DB-PG-079

Código PostgreSQL-specific no se dispersará por el framework.

## DB-PG-080

Capability checks sustituirán vendor checks siempre que sea posible.

---

# 424. Anti-pattern — PostgreSQL everywhere

Incorrecto:

```php
if ($database === 'postgresql') {
    // special behavior
}
```

distribuido por:

```text
ORM
Query Builder
Repository
Migration
Transaction
```

---

# 425. Correcto

```text
Semantic Requirement
      │
      ▼
Capability Resolver
      │
      ▼
Planner Strategy
```

---

# 426. Anti-pattern — Platform as mutable connection state

Incorrecto:

```php
$postgresPlatform->setSearchPath('tenant_a');
```

---

# 427. Correcto

```text
ExecutionScope
      │
      ▼
ConnectionContext
      │
      ▼
SessionState
      │
      ▼
SearchPath
```

---

# 428. Anti-pattern — PostgreSQL schema equals database

Incorrecto:

```text
Database = public
```

---

# 429. Correcto

```text
Database
└── Schema
    └── Table
```

---

# 430. Anti-pattern — SQL in Query Builder

Incorrecto:

```php
$query->append('RETURNING id');
```

---

# 431. Correcto

```text
Query Builder
    │
    ▼
ReturningClause AST
    │
    ▼
Planner
    │
    ▼
Compiler
    │
    ▼
PostgreSqlDialect
```

---

# 432. Anti-pattern — Enum as VARCHAR

Incorrecto:

```text
all enums
=
varchar
```

---

# 433. Correcto

```text
Application Enum
      │
      ▼
Mapping Strategy
      │
      ├── native PostgreSQL enum
      ├── string
      └── custom type
```

---

# 434. Anti-pattern — Array inference

Incorrecto:

```php
is_array($value)
→ PostgreSQL ARRAY
```

---

# 435. Correcto

El mapping deberá ser explícito.

---

# 436. Anti-pattern — Retry inside Platform

Incorrecto:

```php
$postgreSqlPlatform->retry($query);
```

---

# 437. Correcto

```text
Platform
→ classifies

Resilience
→ decides

Executor
→ retries when authorized
```

---

# 438. Anti-pattern — Always wrap migrations

Incorrecto:

```text
BEGIN
all migration SQL
COMMIT
```

sin analizar las operaciones.

---

# 439. Correcto

```text
Migration Operations
       │
       ▼
Platform Capability Analysis
       │
       ▼
Migration Plan
       │
       ├── Transactional Segment
       ├── Non-Transactional Operation
       └── Transactional Segment
```

---

# 440. Anti-pattern — Pool without session reset

Incorrecto:

```text
Request A
SET ROLE
SET search_path
SET timezone
     │
     ▼
Pool
     │
     ▼
Request B
```

---

# 441. Correcto

```text
Request A
     │
     ▼
Release
     │
     ▼
State Inspection
     │
     ▼
PostgreSQL Reset
     │
     ├── success → IDLE
     └── failure → DISCARD
```

---

# 442. Anti-pattern — Server extension equals framework extension

Incorrecto:

```text
PostGIS installed
=
VoltStack PostGIS integration automatically active
```

---

# 443. Correcto

```text
Server Extension Discovery
          │
          ▼
VoltStack Extension
          │
          ▼
Capability Validation
          │
          ▼
Integration Active
```

---

# 444. Reference architecture

```text
                         APPLICATION
                              │
                              ▼
                         DATABASE API
                              │
                ┌─────────────┼─────────────┐
                ▼             ▼             ▼
             Query           ORM          Schema
                │             │             │
                └────────┬────┴────┬────────┘
                         ▼         ▼
                      Planner   Schema Planner
                         │         │
                         └────┬────┘
                              ▼
                    CapabilitySnapshot
                              │
                              ▼
                    PostgreSqlPlatform
                              │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                 ▼
       Type Semantics    Schema Semantics  Tx Semantics
            │                 │                 │
            └─────────────────┼─────────────────┘
                              ▼
                     PostgreSqlDialect
                              │
                              ▼
                         SQL Compiler
                              │
                              ▼
                        CompiledQuery
                              │
                              ▼
                           Executor
                              │
                              ▼
                          Connection
                              │
                              ▼
                      PdoPgSqlDriver
                              │
                              ▼
                     Native Connection
                              │
                              ▼
                       PostgreSQL
```

---

# 445. Runtime reference architecture

```text
FrankenPHP Worker
       │
       ├── PostgreSqlPlatform
       ├── PostgreSqlDialect
       ├── Capability Rules
       ├── Type Mapping
       │
       ├── Request A
       │     │
       │     ├── DatabaseContext A
       │     ├── ConnectionLease A
       │     ├── TransactionContext A
       │     └── SessionState A
       │
       ├── RESET
       │
       └── Request B
             │
             ├── DatabaseContext B
             ├── ConnectionLease B
             ├── TransactionContext B
             └── SessionState B
```

---

# 446. Platform formula

La plataforma efectiva será conceptualmente:

```text
PostgreSqlPlatform
+
ServerVersion
+
RelevantServerConfiguration
+
DriverCapabilities
+
ConnectionRestrictions
+
Extensions
+
Policy
=
Effective PostgreSQL CapabilitySnapshot
```

---

# 447. Query formula

```text
Semantic Query
+
Capability Requirements
+
Effective PostgreSQL Capabilities
=
PostgreSQL Execution Strategy
```

---

# 448. Schema formula

```text
Canonical Schema Operation
+
PostgreSQL Schema Semantics
+
CapabilitySnapshot
=
PostgreSQL Schema Strategy
```

---

# 449. Migration formula

```text
Schema Diff
+
PostgreSQL Capabilities
+
Safety Policy
+
Deployment Context
=
Migration Plan
```

---

# 450. Persistence formula

```text
Entity Changes
+
Mapping Metadata
+
Generated Value Requirements
+
PostgreSQL Capabilities
=
Persistence Plan
```

---

# 451. Connection reuse formula

```text
Reusable PostgreSQL Connection
=
No Active Transaction
+
No Active Cursor
+
No Unsafe Advisory Lock
+
Known Role
+
Known SearchPath
+
Known Timezone
+
Known Session Settings
+
Compatible Generation
+
Successful Reset
```

---

# 452. Portability model

VoltStack deberá permitir tres niveles:

```text
PORTABLE
PLATFORM_AWARE
PLATFORM_SPECIFIC
```

---

# 453. PORTABLE

Ejemplo:

```php
DB::table('users')
    ->where('active', true)
    ->get();
```

---

# 454. PLATFORM_AWARE

Ejemplo:

```text
semantic UPSERT
semantic RETURNING
semantic locking
```

donde Planner selecciona implementación según capabilities.

---

# 455. PLATFORM_SPECIFIC

Ejemplo conceptual:

```text
DISTINCT ON
PostgreSQL arrays
PostgreSQL ranges
PostgreSQL-specific index operators
```

---

# 456. Explicit platform-specific API

Las capacidades específicas deberán ser explícitas.

No deberán aparecer accidentalmente en APIs aparentemente portables.

---

# 457. PostgreSQL as first-class platform

PostgreSQL no será tratado simplemente como:

```text
another PDO connection
```

sino como:

```text
fully modeled database platform
```

---

# 458. Resultado arquitectónico

La aplicación podrá escribir:

```php
$users = DB::table('users')
    ->where('active', true)
    ->orderBy('created_at', 'desc')
    ->get();
```

mientras internamente VoltStack ejecuta:

```text
Builder
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
  ├── PostgreSQL Platform
  └── CapabilitySnapshot
          │
          ▼
     Execution Plan
          │
          ▼
       Compiler
          │
          ▼
 PostgreSqlDialect
          │
          ▼
     Compiled SQL
          │
          ▼
       Executor
          │
          ▼
      Connection
          │
          ▼
    pdo.pgsql
          │
          ▼
     PostgreSQL
```

---

# 459. Integración con ORM

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
Persistence Planner
  │
  ├── Metadata
  ├── PostgreSQL Capabilities
  └── Generated Value Strategy
          │
          ▼
      Query Model
          │
          ▼
      Query Engine
```

El ORM nunca generará SQL PostgreSQL directamente.

---

# 460. Integración con Schema

```text
Schema API
    │
    ▼
Schema Model
    │
    ▼
Schema AST
    │
    ▼
Schema Planner
    │
    ├── PostgreSqlPlatform
    └── CapabilitySnapshot
            │
            ▼
      Schema Operations
            │
            ▼
      Schema Compiler
            │
            ▼
     PostgreSqlDialect
```

---

# 461. Integración con Transactions

```text
Application
     │
     ▼
TransactionManager
     │
     ▼
TransactionContext
     │
     ├── Isolation
     ├── ReadOnly
     ├── Deferrable
     └── Savepoints
            │
            ▼
 PostgreSqlTransactionSemantics
            │
            ▼
        Connection
            │
            ▼
          Driver
```

---

# 462. Integración con Runtime

```text
ExecutionScope
      │
      ▼
DatabaseContext
      │
      ▼
ConnectionLease
      │
      ▼
PostgreSQL Session State
      │
      ▼
Operation
      │
      ▼
Reset
      │
      ▼
Release / Discard
```

---

# 463. Relación con documentos anteriores

```text
10_DATABASE_DRIVER_ARCHITECTURE
        │
11_DATABASE_CONNECTION_SYSTEM
        │
12_DATABASE_CONNECTION_MANAGER
        │
13_DATABASE_CONNECTION_CONFIGURATION_AND_RESOLUTION
        │
14_DATABASE_CONNECTION_POOLING_SYSTEM
        │
15_DATABASE_CONNECTION_LIFECYCLE_SYSTEM
        │
16_DATABASE_CONNECTION_STATE_AND_RESET_SYSTEM
        │
17_DATABASE_DIALECT_SYSTEM
        │
18_DATABASE_PLATFORM_CAPABILITY_SYSTEM
        │
19_DATABASE_MYSQL_AND_MARIADB_PLATFORM
        │
20_DATABASE_POSTGRESQL_PLATFORM
```

---

# 464. Regla maestra

> PostgreSQL deberá integrarse mediante semántica y capabilities, no mediante condicionales de vendor distribuidos por el framework.

Por tanto:

```text
PostgreSQL
    │
    ├── Driver transport
    ├── Dialect syntax
    ├── Platform semantics
    ├── Capability profile
    ├── Type semantics
    ├── Schema semantics
    ├── Transaction semantics
    ├── Session semantics
    └── Error classification
```

permanecerán claramente separados.

---

# 465. Decisión final

VoltStack adoptará para PostgreSQL:

```text
Portable Public API
        │
        ▼
Semantic Database Model
        │
        ▼
Capability-Aware Planning
        │
        ▼
PostgreSQL Platform Semantics
        │
        ▼
PostgreSQL Dialect Compilation
        │
        ▼
Driver-Neutral Execution
        │
        ▼
pdo.pgsql initially
        │
        ▼
PostgreSQL
```

Esto permitirá aprovechar capacidades avanzadas de PostgreSQL sin convertir el resto del framework en código PostgreSQL-specific.

---

# 466. Siguiente documento

El siguiente documento será:

```text
21_DATABASE_SQLITE_PLATFORM.md
```

y deberá definir especialmente:

```text
SQLitePlatform
SQLiteDialect
pdo.sqlite
SQLite3 future driver
file databases
in-memory databases
shared-memory semantics
database identity
locking model
single-file concurrency
busy timeout
WAL
journal modes
foreign key enforcement
PRAGMA state
STRICT tables
WITHOUT ROWID
rowid semantics
generated columns
RETURNING
UPSERT
CTE
window functions
JSON capabilities
type affinity
dynamic typing
STRICT typing
schema limitations
ALTER TABLE capabilities
table rebuild migration strategy
indexes
partial indexes
expression indexes
transactions
savepoints
DDL behavior
connection state
PRAGMA reset
persistent runtime safety
testing
temporary databases
test isolation
capability/version discovery
SQLite library version
```

manteniendo la regla fundamental:

```text
Driver
≠
Dialect
≠
Platform
≠
Capability
```

---

# 467. Conclusión

El soporte PostgreSQL de VoltStack deberá combinar:

```text
simple developer experience
+
portable query semantics
+
rich PostgreSQL capabilities
+
strict architectural boundaries
+
persistent-runtime safety
```

La aplicación podrá permanecer sencilla mientras la infraestructura reconoce correctamente conceptos como:

```text
schemas
search_path
RETURNING
ON CONFLICT
sequences
identity
JSONB
arrays
UUID
enums
domains
ranges
partial indexes
expression indexes
concurrent indexes
deferrable constraints
transactional DDL
failed transaction state
advisory locks
COPY
```

sin introducirlos arbitrariamente en el núcleo portable.

La regla final será:

> **VoltStack no ocultará las capacidades de PostgreSQL, pero tampoco permitirá que PostgreSQL defina la arquitectura del framework.**

El núcleo expresará intención semántica; `PostgreSqlPlatform` describirá lo que el servidor puede hacer; el Planner decidirá cómo hacerlo; `PostgreSqlDialect` traducirá esa decisión a SQL; y el Driver será responsable únicamente de comunicarse con PostgreSQL.