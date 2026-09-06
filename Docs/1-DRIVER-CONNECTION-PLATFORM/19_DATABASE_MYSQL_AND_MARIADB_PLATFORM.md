# 19_DATABASE_MYSQL_AND_MARIADB_PLATFORM.md

# VoltStack Quantum Database
## MySQL and MariaDB Platform Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 19 — MySQL and MariaDB Platform  
**Estado:** Architecture Specification  
**Nivel:** Infrastructure / Platform / Capability  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura oficial de soporte para:

```text
MySQL
MariaDB
```

dentro de:

```text
VoltStack/Quantum/Database
```

El objetivo no es simplemente crear:

```text
MySqlDriver
MariaDbDriver
```

sino modelar correctamente las diferencias entre:

```text
transport
SQL syntax
server semantics
server version
capabilities
schema behavior
transaction behavior
type behavior
```

de ambos motores.

---

# 2. Decisión arquitectónica principal

VoltStack considerará:

```text
MySQL
≠
MariaDB
```

como plataformas.

Aunque puedan compartir:

```text
PDO MySQL transport
wire protocol
parts of SQL syntax
type families
schema concepts
```

no deberán tratarse como una única plataforma universal.

---

# 3. Regla fundamental

La arquitectura será:

```text
pdo.mysql
    │
    ├────────► MySqlPlatform
    │
    └────────► MariaDbPlatform
```

Por tanto:

```text
Driver identity
≠
Platform identity
```

---

# 4. Motivación

MariaDB nació con una fuerte compatibilidad con MySQL, pero ambas plataformas pueden divergir en:

- versiones;
- funciones;
- tipos;
- JSON;
- DDL;
- optimizador;
- SQL modes;
- generated columns;
- locking;
- replication;
- RETURNING;
- sequences;
- metadata;
- server variables;
- capabilities.

VoltStack no deberá asumir:

```text
PDO mysql driver
=
MySQL semantics
```

---

# 5. Separación completa

```text
Driver
    │
    ▼
pdo.mysql
    │
    ▼
Native Connection
    │
    ▼
Server Discovery
    │
    ├─────────────┐
    ▼             ▼
MySQL          MariaDB
    │             │
    ▼             ▼
Platform       Platform
    │             │
    ▼             ▼
Capabilities   Capabilities
```

---

# 6. Componentes principales

La integración se dividirá conceptualmente en:

```text
MySqlFamily/
├── Shared Infrastructure
├── Server Discovery
├── MySQL Platform
├── MariaDB Platform
├── MySQL Dialect
├── MariaDB Dialect
├── Capability Providers
├── Type Mapping
├── Schema Semantics
├── Transaction Semantics
├── Session Semantics
├── Error Classification
└── Conformance Testing
```

---

# 7. Shared family infrastructure

Podrá existir infraestructura común:

```text
MySqlFamily
```

pero únicamente para comportamiento realmente compartido.

---

# 8. Shared infrastructure no será una plataforma

No crear:

```text
MySqlCompatiblePlatform
```

como plataforma final universal.

Preferir:

```text
MySqlFamilySupport
        │
        ├── MySqlPlatform
        └── MariaDbPlatform
```

---

# 9. Composition over inheritance

Evitar una jerarquía profunda:

```text
AbstractDatabasePlatform
    ↓
AbstractMySqlPlatform
    ↓
MySqlPlatform
    ↓
MariaDbPlatform
```

especialmente si MariaDB termina heredando comportamiento MySQL que ya no es correcto.

---

# 10. Estrategia preferida

Preferir composición:

```text
MySqlPlatform
    ├── MySqlCapabilityProvider
    ├── MySqlTypeSemantics
    ├── MySqlSchemaSemantics
    └── SharedMySqlFamilyServices

MariaDbPlatform
    ├── MariaDbCapabilityProvider
    ├── MariaDbTypeSemantics
    ├── MariaDbSchemaSemantics
    └── SharedMySqlFamilyServices
```

---

# 11. Driver oficial inicial

La implementación inicial podrá utilizar:

```text
PDO_MYSQL
```

mediante:

```text
PdoMySqlDriver
```

---

# 12. Driver ID

ID conceptual:

```text
pdo.mysql
```

---

# 13. User-facing aliases

Podrán aceptarse:

```text
mysql
mariadb
```

como configuración de alto nivel.

Pero deberán resolverse por separado.

Ejemplo:

```text
mysql
    ├── driver   = pdo.mysql
    ├── platform = mysql
    └── dialect  = mysql

mariadb
    ├── driver   = pdo.mysql
    ├── platform = mariadb
    └── dialect  = mariadb
```

---

# 14. Configuración explícita

También será posible:

```yaml
database:
  connections:
    primary:
      driver: pdo.mysql
      platform: mariadb
```

cuando sea necesario.

---

# 15. Configuración simple

Para DX:

```yaml
database:
  connections:
    primary:
      driver: mariadb
```

podrá normalizarse internamente a:

```text
Driver:
pdo.mysql

Platform hint:
mariadb
```

---

# 16. No confundir alias con DriverId

`mariadb` podrá ser:

```text
configuration alias
```

sin significar que existe necesariamente:

```text
MariaDbNativeDriver
```

---

# 17. Futuro driver nativo

La arquitectura permitirá:

```text
native.mysql
native.mariadb
async.mysql
```

sin modificar la semántica de Platform.

---

# 18. Platform IDs

IDs oficiales:

```text
mysql
mariadb
```

---

# 19. Dialect IDs

Inicialmente:

```text
mysql
mariadb
```

aunque puedan compartir muchos componentes.

---

# 20. Family ID

Podrá existir:

```text
mysql-family
```

exclusivamente para:

- organización;
- discovery;
- shared services;
- testing;
- compatibility metadata.

No sustituirá el PlatformId.

---

# 21. Arquitectura resultante

```text
ConnectionDefinition
        │
        ▼
ConnectionManager
        │
        ▼
PdoMySqlDriver
        │
        ▼
NativeConnection
        │
        ▼
ServerMetadataDiscovery
        │
        ▼
PlatformResolver
        │
    ┌───┴────┐
    ▼        ▼
 MySQL    MariaDB
    │        │
    ▼        ▼
Dialect   Dialect
    │        │
    ▼        ▼
Capabilities
```

---

# 22. Platform resolution

VoltStack deberá poder resolver la plataforma mediante:

```text
configured platform hint
+
server identity
+
server version
+
server metadata
```

---

# 23. Offline resolution

Sin conexión activa:

```text
platform: mysql
```

producirá:

```text
PreliminaryPlatform(
    id: mysql
)
```

---

# 24. Online resolution

Al abrir la conexión:

```text
NativeConnection
    │
    ▼
Server Discovery
    │
    ▼
ResolvedPlatform
```

---

# 25. Server identity discovery

Deberá existir:

```text
MySqlFamilyServerDiscovery
```

o equivalente.

---

# 26. Discovery responsibilities

Podrá descubrir:

```text
server product
server version
version comment
server variables relevantes
compatibility indicators
```

---

# 27. Discovery no será ORM concern

Prohibido que:

```text
EntityManager
UnitOfWork
Repository
QueryBuilder
```

intenten determinar si el servidor es MySQL o MariaDB.

---

# 28. ServerMetadata

Modelo conceptual:

```php
final readonly class MySqlFamilyServerMetadata
{
    public function __construct(
        public DatabaseServerProduct $product,
        public ServerVersion $version,
        public ?string $versionComment,
        public ServerMetadataAttributes $attributes,
    ) {}
}
```

---

# 29. DatabaseServerProduct

Conceptualmente:

```text
MYSQL
MARIADB
UNKNOWN_MYSQL_COMPATIBLE
```

---

# 30. Unknown compatible server

Un servidor compatible con el protocolo MySQL pero no reconocido no deberá convertirse automáticamente en:

```text
MySqlPlatform
```

con todas sus capabilities.

---

# 31. Conservative compatibility

Preferir:

```text
UnknownMySqlCompatiblePlatform
```

o requerir:

```text
explicit platform configuration
```

antes de asumir capabilities avanzadas.

---

# 32. Platform hint validation

Si configuración declara:

```text
platform = mysql
```

pero discovery determina inequívocamente:

```text
MariaDB
```

VoltStack deberá detectar la discrepancia.

---

# 33. Mismatch policy

Opciones posibles:

```text
STRICT
WARN
TRUST_SERVER
```

---

# 34. Default production policy

Preferencia:

```text
STRICT
```

para incompatibilidades claras.

---

# 35. Server version

`ServerVersion` será value object.

Nunca:

```php
if ($version >= '8.0') {
}
```

mediante comparación de strings.

---

# 36. Version normalization

La versión deberá normalizarse a:

```text
major
minor
patch
vendor metadata
```

---

# 37. Vendor version parsers

Podrán existir:

```text
MySqlVersionParser
MariaDbVersionParser
```

---

# 38. Version semantics

La numeración de MySQL y MariaDB será interpretada por su propia plataforma.

---

# 39. No cross-vendor version assumptions

Nunca:

```text
MariaDB 10.x > MySQL 8.x
```

como inferencia de capabilities.

---

# 40. Capability resolution

Pipeline:

```text
Platform
   │
   ▼
Platform Baseline
   │
   ▼
Version Rules
   │
   ▼
Server Configuration
   │
   ▼
Driver Constraints
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

# 41. MySQL capability provider

Componente:

```text
MySqlPlatformCapabilityProvider
```

---

# 42. MariaDB capability provider

Componente:

```text
MariaDbPlatformCapabilityProvider
```

---

# 43. Shared capability rules

Sólo capabilities realmente comunes podrán vivir en:

```text
MySqlFamilyCapabilityRules
```

---

# 44. No optimistic sharing

Si existe duda sobre equivalencia:

```text
duplicate explicit rule
>
incorrect shared abstraction
```

---

# 45. Capability categories

Ambas plataformas deberán describir al menos:

```text
Connection
Query
Expression
CTE
Window
Returning
Upsert
JSON
Transactions
Locking
Schema
DDL
Indexes
Types
Generated Values
Bulk Operations
Session
Administration
```

---

# 46. Capability granularity

No usar:

```text
mysqlFeatures = true
```

ni:

```text
supportsAdvancedSql = true
```

---

# 47. Query capabilities

Ejemplos:

```text
query.cte
query.cte.recursive
query.window
query.union
query.intersect
query.except
query.upsert
query.returning.insert
query.returning.update
query.returning.delete
```

---

# 48. Capability values depend on version

Las capabilities concretas serán resueltas por:

```text
Platform
+
Version
+
Context
```

y no deberán asumirse universalmente en este documento.

---

# 49. Dialect architecture

Se mantendrán:

```text
MySqlDialect
MariaDbDialect
```

---

# 50. Shared dialect infrastructure

Podrá existir:

```text
MySqlFamilyDialectSupport
```

para:

- quoting compartido;
- placeholder mechanics;
- syntax helpers;
- common fragments.

---

# 51. Dialect specialization

Cada Dialect conservará control sobre:

```text
vendor syntax
version-sensitive syntax
feature syntax
DDL syntax
locking syntax
```

---

# 52. Compiler relationship

```text
ExecutionPlan
     │
     ▼
SQL Compiler
     │
     ├── CapabilitySnapshot
     └── Dialect
             │
             ▼
             SQL
```

---

# 53. Platform does not compile SQL

Prohibido:

```php
$platform->compileSelect(...);
```

como arquitectura general.

---

# 54. Dialect does not decide feature semantics

Dialect no deberá decidir:

```text
should this feature be emulated?
```

Eso pertenece al Planner.

---

# 55. Identifier quoting

La familia MySQL utiliza reglas específicas de quoting.

VoltStack representará esto mediante:

```text
IdentifierQuoter
```

asociado al Dialect.

---

# 56. Identifier abstraction

Query AST almacenará:

```text
Identifier
```

no:

```text
`users`
```

---

# 57. Quoting late

```text
Identifier
    │
    ▼
Compiler
    │
    ▼
Dialect
    │
    ▼
Quoted SQL
```

---

# 58. No prequoted identifiers

Evitar APIs internas que propaguen:

```text
`users`.`name`
```

antes del Compiler.

---

# 59. Identifier length

La longitud máxima y otras restricciones serán capabilities/Platform semantics.

---

# 60. Identifier case behavior

Platform deberá describir:

```text
case sensitivity
case folding
filesystem implications where applicable
```

sin contaminar Query Model.

---

# 61. Database/catalog semantics

La familia MySQL maneja conceptos que deberán mapearse cuidadosamente al modelo portable de VoltStack.

---

# 62. Schema terminology

VoltStack mantendrá su terminología canónica:

```text
DatabaseSchema
Table
Column
Index
Constraint
```

y Platform traducirá las diferencias conceptuales.

---

# 63. Type architecture

La plataforma deberá proporcionar:

```text
DatabaseTypeMapping
```

---

# 64. Type flow

```text
VoltStack Type
      │
      ▼
Platform Type Mapping
      │
      ▼
Dialect Type Declaration
      │
      ▼
SQL Type
```

---

# 65. Type mapping responsibilities

Debe cubrir:

```text
integer
bigint
decimal
float
boolean
string
text
binary
date
time
datetime
timestamp
json
uuid
enum
blob
```

según soporte efectivo.

---

# 66. Portable type vs native type

Ejemplo:

```text
VoltStack Boolean
```

podrá mapearse a una representación nativa compatible.

El dominio no deberá depender de cómo se represente físicamente.

---

# 67. Native type metadata

Schema introspection deberá preservar cuando sea necesario:

```text
native type
native modifiers
precision
scale
length
unsigned
charset
collation
```

---

# 68. Type round-trip

Objetivo:

```text
Database
   │ introspect
   ▼
Schema Model
   │ compile
   ▼
Database
```

con mínima pérdida semántica.

---

# 69. Unsigned numeric semantics

Si una plataforma ofrece:

```text
UNSIGNED
```

se modelará como metadata/capability específica.

No se asumirá portable a PostgreSQL/SQLite.

---

# 70. Boolean semantics

VoltStack mantendrá:

```text
BooleanType
```

independientemente de la representación nativa.

---

# 71. Enum semantics

MySQL/MariaDB podrán tener soporte nativo específico.

VoltStack deberá distinguir:

```text
Application Enum
Database Enum Type
String Constraint Emulation
```

---

# 72. Enum capability

Podrán existir:

```text
type.enum.native
type.enum.alter
```

---

# 73. JSON architecture

JSON merece tratamiento separado.

---

# 74. JSON semantic type

VoltStack tendrá:

```text
JsonType
```

portable.

---

# 75. Native JSON differences

MySQL y MariaDB no deberán asumirse idénticos internamente respecto a:

```text
storage
validation
functions
indexing
path behavior
DDL representation
```

---

# 76. JSON capabilities

Ejemplos:

```text
json.validation
json.extract
json.path
json.contains
json.mutation
json.index
json.aggregate
```

---

# 77. JSON query AST

Query Builder deberá producir:

```text
JsonExpression
```

en lugar de funciones vendor-specific cuando la operación sea portable.

---

# 78. JSON planner

Planner seleccionará:

```text
native strategy
emulated strategy
unsupported
```

---

# 79. JSON compiler

Dialect producirá la sintaxis apropiada.

---

# 80. JSON escape hatch

Operaciones vendor-specific seguirán siendo posibles mediante APIs explícitas.

---

# 81. Date/time semantics

Platform deberá describir:

```text
date types
time types
datetime types
timestamp behavior
fractional precision
timezone behavior
```

---

# 82. Timezone responsibility

VoltStack deberá evitar asumir que todos los tipos temporales conservan timezone de la misma manera.

---

# 83. DateTime mapping

El Type System decidirá conversiones PHP.

Platform describirá la semántica física.

---

# 84. Precision

Capabilities podrán indicar:

```text
datetime.fractional_precision
timestamp.fractional_precision
```

---

# 85. String semantics

Platform deberá manejar:

```text
charset
collation
length
binary/nonbinary behavior
```

---

# 86. Charset

La configuración de charset pertenecerá a:

```text
Connection Session Profile
+
Platform semantics
```

---

# 87. Collation

Collation podrá existir en:

```text
database
table
column
expression
```

según capacidad.

---

# 88. Portable collation

VoltStack no prometerá que una collation vendor-specific sea portable.

---

# 89. Binary types

El Type System deberá distinguir:

```text
binary data
text data
```

sin inferirlo sólo por PHP `string`.

---

# 90. Generated columns

Capabilities:

```text
schema.generated_column
schema.generated_column.virtual
schema.generated_column.stored
```

---

# 91. Generated column model

Schema AST:

```text
GeneratedColumnDefinition
├── expression
├── storage mode
└── type
```

---

# 92. Platform validation

Platform/Schema Planner determinarán si la definición puede representarse.

---

# 93. Generated values

No confundir:

```text
generated column
```

con:

```text
generated primary key
```

---

# 94. Identity generation

La familia MySQL puede proporcionar mecanismos específicos para IDs generados.

VoltStack lo modelará semánticamente.

---

# 95. Identity strategy

```text
GeneratedIdentifierStrategy
```

podrá resolverse a:

```text
native auto-generated value
sequence-like strategy
application-generated value
custom strategy
```

según Platform.

---

# 96. No ORM vendor check

UnitOfWork no preguntará:

```php
if ($platform === 'mysql') {
    // get last insert id
}
```

---

# 97. Persistence plan

```text
Entity ChangeSet
      │
      ▼
Persistence Planner
      │
      ├── CapabilitySnapshot
      └── GeneratedValueStrategy
              │
              ▼
          Query Model
```

---

# 98. Generated value retrieval

Driver y Platform deberán permanecer separados:

```text
Platform:
what generated-value semantics exist?

Driver:
what native API can retrieve generated values?
```

---

# 99. Effective generated-value strategy

Será intersección de ambas.

---

# 100. INSERT architecture

Query AST representará:

```text
InsertQuery
```

portable.

---

# 101. Multi-row INSERT

Capability:

```text
bulk.insert
```

podrá incluir límites:

```text
max parameters
max packet considerations
generated value limitations
```

---

# 102. Parameter limits

Planner podrá dividir batches según:

```text
parameter.max_count
```

y otras restricciones.

---

# 103. Packet limits

Límites dependientes de servidor/configuración podrán influir en batch planning cuando sean conocidos.

---

# 104. No hardcoded batch size

Evitar:

```php
$batchSize = 1000;
```

como supuesto universal.

---

# 105. UPSERT semantic model

VoltStack representará:

```text
UpsertQuery
```

como intención semántica.

---

# 106. Upsert AST

Conceptualmente:

```text
UpsertQuery
├── target table
├── insert values
├── conflict target
├── update assignments
└── returning
```

---

# 107. Platform upsert capabilities

Podrán incluir:

```text
query.upsert
query.upsert.conflict_columns
query.upsert.constraint_target
query.upsert.do_nothing
query.upsert.update
query.upsert.returning
```

---

# 108. No syntax in Builder

Builder no producirá:

```text
ON DUPLICATE KEY UPDATE
```

directamente.

---

# 109. Planner responsibility

Planner traducirá intención semántica a:

```text
MySQL-family upsert strategy
```

cuando sea compatible.

---

# 110. Compiler responsibility

Dialect producirá la sintaxis concreta.

---

# 111. Semantic mismatch

Si la semántica portable solicitada no puede representarse exactamente:

```text
CapabilityNotSupportedException
```

será preferible a una traducción aproximada.

---

# 112. RETURNING architecture

VoltStack no tendrá un simple:

```text
supportsReturning = true/false
```

---

# 113. Returning capability dimensions

```text
query.returning.insert
query.returning.update
query.returning.delete
query.returning.multi_row
query.returning.arbitrary_columns
```

---

# 114. MySQL/MariaDB divergence

Las plataformas resolverán estas capabilities independientemente.

---

# 115. Generated key API is not RETURNING

Muy importante:

```text
Driver generated-key retrieval
≠
SQL RETURNING capability
```

---

# 116. Persistence planner

Puede seleccionar:

```text
SQL RETURNING
```

o:

```text
native generated-key API
```

dependiendo de lo que realmente necesite.

---

# 117. CTE architecture

Capabilities:

```text
query.cte
query.cte.recursive
```

más cualquier variante necesaria.

---

# 118. CTE AST

El AST permanecerá portable:

```text
CommonTableExpression
```

---

# 119. Version-aware CTE support

Platform Capability Provider resolverá soporte según versión.

---

# 120. Window functions

Capabilities:

```text
query.window
query.window.frame.rows
query.window.frame.range
```

etc.

---

# 121. Window AST

Builder produce:

```text
WindowExpression
```

sin conocimiento de versión MySQL/MariaDB.

---

# 122. Set operations

Capabilities:

```text
query.union
query.intersect
query.except
```

deberán modelarse individualmente.

---

# 123. No SQL-standard assumption

Que una operación exista en SQL estándar no implica soporte uniforme.

---

# 124. Locking architecture

Platform deberá describir:

```text
row locking
shared locking
exclusive locking
nowait
skip locked
advisory locking
```

---

# 125. Lock semantic AST

Ejemplo:

```text
LockClause(
    mode: UPDATE,
    wait: SKIP_LOCKED
)
```

---

# 126. Dialect compilation

Dialect decide cómo expresarlo.

---

# 127. Capability validation

Planner verifica si la combinación:

```text
UPDATE + SKIP_LOCKED
```

es válida.

---

# 128. Advisory locks

Podrán exponerse como capability vendor-specific si su semántica no es portable.

---

# 129. Transaction architecture

Platform deberá describir:

```text
transaction support
savepoints
isolation levels
read-only transactions
locking semantics
DDL transaction behavior
implicit commits
```

---

# 130. TransactionManager remains generic

```text
TransactionManager
      │
      ▼
TransactionCapabilities
      │
      ▼
Connection
      │
      ▼
Driver primitives
```

---

# 131. Isolation level

VoltStack tendrá enum portable:

```text
READ_UNCOMMITTED
READ_COMMITTED
REPEATABLE_READ
SERIALIZABLE
```

---

# 132. Platform isolation mapping

Platform validará y mapeará niveles soportados.

---

# 133. Default isolation

No deberá asumirse en el framework.

Podrá descubrirse/configurarse.

---

# 134. Transactional DDL

Debe modelarse explícitamente.

No asumir:

```text
DDL inside transaction
=
fully rollbackable
```

---

# 135. Implicit commit semantics

La plataforma deberá proporcionar metadata/capabilities para operaciones DDL que afecten transacciones.

---

# 136. Migration Planner integration

Antes de ejecutar DDL:

```text
Migration Planner
      │
      ▼
Transaction/DDL Capabilities
```

---

# 137. Savepoints

Capabilities:

```text
transaction.savepoint
transaction.savepoint.release
```

cuando sea útil distinguirlas.

---

# 138. Nested transactions

VoltStack construirá nested transaction semantics sobre savepoints cuando corresponda.

---

# 139. Deadlocks

Platform/Error Classification deberá reconocer condiciones de:

```text
deadlock
lock timeout
serialization conflict
```

cuando puedan identificarse.

---

# 140. Error code handling

No dispersar:

```php
if ($errorCode === ...) {
}
```

por TransactionManager o ORM.

---

# 141. Failure classification

Pipeline:

```text
Native Error
    │
    ▼
Driver Error Adapter
    │
    ▼
Platform Failure Classifier
    │
    ▼
DatabaseFailure
```

---

# 142. DatabaseFailure categories

Ejemplos:

```text
DeadlockFailure
LockTimeoutFailure
UniqueConstraintFailure
ForeignKeyFailure
NotNullFailure
ConnectionFailure
AuthenticationFailure
SyntaxFailure
```

---

# 143. Retry policy separation

Platform clasifica.

Resilience System decide si reintenta.

---

# 144. Schema architecture

```text
Schema Model
     │
     ▼
Schema Planner
     │
     ├── Platform Capabilities
     └── Platform Schema Semantics
             │
             ▼
       Schema Operations
             │
             ▼
       Schema Compiler
             │
             ▼
           Dialect
```

---

# 145. Table semantics

Deberán contemplarse:

```text
engine
charset
collation
temporary
comment
platform options
```

sin convertir todas ellas en propiedades obligatorias del modelo portable.

---

# 146. Portable vs native table options

Separación:

```text
TableDefinition
├── portable options
└── PlatformOptionBag
```

---

# 147. PlatformOptionBag

Sólo para escape hatch/control avanzado.

---

# 148. Native options validation

Opciones deberán validarse mediante schema específico de Platform.

---

# 149. No arbitrary option leakage

No pasar arrays arbitrarios desde Application hasta SQL Compiler sin validación.

---

# 150. Storage engine

El concepto de storage engine podrá modelarse como opción MySQL-family específica.

---

# 151. Default storage engine

No deberá hardcodearse si el servidor puede determinarlo.

---

# 152. Engine capability effects

Algunas capabilities efectivas podrían depender del engine utilizado.

---

# 153. Table-specific capabilities

Cuando sea necesario, ciertas decisiones podrán requerir:

```text
TableCapabilityContext
```

---

# 154. Avoid globalizing table-specific behavior

No declarar:

```text
Platform supports X
```

si realmente depende del engine de una tabla concreta.

---

# 155. Column semantics

Debe soportarse metadata para:

```text
nullable
default
generated
auto-generated
unsigned
charset
collation
comment
```

según corresponda.

---

# 156. Defaults

Schema AST deberá distinguir:

```text
literal default
expression default
no default
NULL default
```

---

# 157. Default expressions

Platform capability determinará qué expresiones son válidas.

---

# 158. Index architecture

Schema Model:

```text
IndexDefinition
├── columns/expressions
├── uniqueness
├── name
├── predicate
└── options
```

---

# 159. Index capabilities

```text
schema.index
schema.index.unique
schema.index.expression
schema.index.prefix_length
schema.index.partial
schema.index.fulltext
schema.index.spatial
```

según plataforma.

---

# 160. Prefix indexes

Si son vendor/family-specific deberán modelarse como capability/opción específica.

---

# 161. Full-text indexes

No deberán considerarse automáticamente portables a otras plataformas.

---

# 162. Spatial indexes

Igualmente deberán ser capability-driven.

---

# 163. Foreign keys

Capabilities:

```text
schema.foreign_key
schema.foreign_key.on_delete
schema.foreign_key.on_update
schema.foreign_key.deferrable
```

---

# 164. Referential actions

VoltStack podrá modelar:

```text
CASCADE
RESTRICT
SET_NULL
NO_ACTION
SET_DEFAULT
```

pero Platform validará soporte real.

---

# 165. Constraint naming

Platform deberá manejar límites y reglas de nombres.

---

# 166. Check constraints

Capability específica:

```text
schema.check_constraint
```

y, si es necesario:

```text
schema.check_constraint.enforced
```

---

# 167. Syntax vs enforcement

Muy importante:

```text
syntax accepted
≠
constraint enforced
```

---

# 168. Schema introspection

Se necesitarán implementaciones específicas:

```text
MySqlSchemaIntrospector
MariaDbSchemaIntrospector
```

---

# 169. Shared introspection

Podrán compartir parsers/mappers cuando la metadata sea equivalente.

---

# 170. Introspection output

Siempre deberá producir:

```text
VoltStack Schema Model
```

---

# 171. Native metadata preservation

Metadata no portable podrá conservarse mediante:

```text
PlatformMetadata
```

---

# 172. Schema diff

```text
Existing Schema
      │
      ▼
Canonical Schema Model
      │
      ▼
Schema Diff
      │
      ▼
Desired Schema
```

---

# 173. Platform-aware diff

El Diff Engine podrá necesitar Platform semantics para evitar falsos cambios.

---

# 174. Example

Diferentes representaciones nativas que sean semánticamente equivalentes no deberán generar migraciones infinitas.

---

# 175. Canonicalization

Cada Platform deberá proporcionar reglas de:

```text
schema canonicalization
```

---

# 176. Type canonicalization

Ejemplo conceptual:

```text
native declaration
    │
    ▼
canonical platform type
    │
    ▼
VoltStack type metadata
```

---

# 177. Default canonicalization

También para:

```text
default expressions
```

---

# 178. Index canonicalization

También para:

```text
index definitions
```

---

# 179. Foreign key canonicalization

También para:

```text
referential actions
constraint names
```

---

# 180. Migration architecture

Migration Planner utilizará:

```text
MySqlPlatform
```

o:

```text
MariaDbPlatform
```

para planear operaciones seguras.

---

# 181. Migration target

Cada migration deberá ejecutarse contra un:

```text
ResolvedPlatform
+
CapabilitySnapshot
```

---

# 182. Offline migration compilation

Será posible con:

```text
configured platform
+
target version
```

cuando haya suficiente información.

---

# 183. Online verification

Antes de deployment podrá verificarse:

```text
target platform
=
actual platform
```

---

# 184. Migration drift

Si la migration fue compilada para:

```text
MySQL target profile
```

y se intenta ejecutar sobre:

```text
MariaDB
```

deberá detectarse cuando existan diferencias relevantes.

---

# 185. Migration capability fingerprint

Los planes precompilados podrán incluir:

```text
CapabilityFingerprint
```

---

# 186. DDL safety

Platform deberá describir características relevantes como:

```text
locking impact
implicit commit behavior
online DDL possibilities
algorithm hints
lock hints
```

cuando se soporten.

---

# 187. Zero-downtime migration

No será una simple opción:

```text
zeroDowntime = true
```

---

# 188. Migration Safety Planner

Deberá evaluar:

```text
operation
table characteristics
platform
version
capabilities
deployment policy
```

---

# 189. Online DDL

Podrá existir capability granular:

```text
migration.online_ddl
migration.online_index
migration.inplace_alter
```

---

# 190. Vendor-specific migration hints

Cuando sean necesarias podrán representarse como estrategias especializadas.

---

# 191. Session architecture

MySQL/MariaDB tienen estado de sesión que debe manejarse cuidadosamente en runtimes persistentes.

---

# 192. SessionProfile

Podrá incluir:

```text
charset
collation
timezone
sql mode
transaction defaults
application metadata
read-only settings
```

según soporte.

---

# 193. Session initialization

```text
Native Connection
      │
      ▼
Platform Session Initializer
      │
      ▼
Known Session State
```

---

# 194. Session state tracking

Connection State System deberá saber qué cambios pueden contaminar una conexión reutilizable.

---

# 195. Session reset

Antes de devolver una conexión al pool:

```text
Session State
      │
      ▼
Reset Planner
      │
      ▼
Platform Reset Strategy
      │
      ▼
Known Clean State
```

---

# 196. SQL mode

Debe tratarse como configuración semánticamente importante.

---

# 197. SQL mode effects

Puede afectar:

```text
query behavior
type coercion
validation
date handling
grouping behavior
```

por lo que no deberá cambiar silenciosamente entre requests.

---

# 198. Session fingerprint

Podrá existir:

```text
SessionProfileFingerprint
```

---

# 199. Pool compatibility

Una conexión física sólo podrá reutilizarse para un profile compatible.

---

# 200. Charset state

Debe conocerse y restaurarse según policy.

---

# 201. Collation state

Igualmente cuando sea relevante.

---

# 202. Timezone state

Una conexión del pool no deberá heredar timezone accidentalmente de una request anterior.

---

# 203. Transaction state

Nunca deberá regresar al pool una conexión con:

```text
active transaction
```

---

# 204. Temporary state

También considerar:

```text
temporary tables
user variables
session variables
locks
prepared resources
```

---

# 205. Advisory/user locks

Si se utilizan, deberán participar en:

```text
dirty-state tracking
```

o provocar descarte cuando no pueda garantizarse limpieza.

---

# 206. Native escape hatch

Uso directo de PDO puede alterar estado desconocido.

---

# 207. Unsafe native access policy

Después de acceso nativo:

```text
ConnectionState = TAINTED
```

podrá ser la estrategia conservadora.

---

# 208. Reset capability

La plataforma/driver deberán indicar:

```text
connection.reset
connection.session_reset
```

y nivel de garantía.

---

# 209. Reset strength

Podrá modelarse:

```text
NONE
PARTIAL
KNOWN_PROFILE
FULL
```

---

# 210. Pool reuse rule

```text
Reusable
=
No active transaction
+
No open cursor
+
No unreleased lock
+
Known session state
+
Compatible generation
+
Successful reset
```

---

# 211. Persistent runtime

Especialmente importante para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 212. FrankenPHP

Modelo:

```text
Worker
  │
  ├── Request A
  │    └── MySQL lease
  │
  ├── RESET
  │
  └── Request B
       └── MariaDB lease
```

sin contaminación.

---

# 213. Mixed-platform worker

El mismo worker podrá mantener pools para:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

sin estado global de plataforma.

---

# 214. No current platform singleton

Prohibido:

```php
MySqlPlatform::setCurrent(...);
```

---

# 215. Platform lifetime

Platform objects deberán ser preferentemente:

```text
immutable
stateless
application/worker safe
```

---

# 216. ResolvedPlatform lifetime

Un:

```text
ResolvedPlatform
```

podrá asociarse a una:

```text
connection generation
```

o servidor compatible.

---

# 217. CapabilitySnapshot lifetime

Será inmutable y reutilizable sólo mientras su contexto/fingerprint sea válido.

---

# 218. Connection failover

Si el endpoint cambia:

```text
Server A
→
Server B
```

deberán verificarse:

```text
platform
version
capabilities
session compatibility
```

---

# 219. Failover mismatch

Si el nuevo servidor no cumple requisitos:

```text
FailoverTargetIncompatibleException
```

o equivalente.

---

# 220. Read replicas

No asumir que todas las replicas tienen exactamente:

```text
same version
same configuration
same capabilities
```

---

# 221. Replica validation

Topology System podrá utilizar:

```text
CapabilityFingerprint
```

para verificar compatibilidad.

---

# 222. Query routing

Una query podrá requerir:

```text
ConnectionIntent::READ
+
CapabilityRequirementSet
```

---

# 223. Capability-aware replica selection

Sólo targets compatibles serán candidatos.

---

# 224. Platform health

Health Check podrá verificar:

```text
identity
version
readiness
capability profile
```

sin mezclarlo con ORM.

---

# 225. Server configuration discovery

Algunas variables podrán ser relevantes para capabilities.

---

# 226. Discovery whitelist

No leer indiscriminadamente todas las variables del servidor.

---

# 227. Minimal discovery

Sólo consultar:

```text
metadata required for correct semantics
```

---

# 228. Discovery caching

Los resultados podrán cachearse por:

```text
server identity
connection generation
capability fingerprint
```

---

# 229. Discovery security

Nunca registrar credenciales ni secretos.

---

# 230. Server variables

Deberán clasificarse:

```text
semantic
performance
diagnostic
irrelevant
```

---

# 231. Semantic variables

Son las únicas que pueden afectar decisiones de compilación/planning automáticamente.

---

# 232. Performance variables

Podrán alimentar Telemetry/Optimizer hints, pero no cambiar semántica sin una razón explícita.

---

# 233. Error architecture

Debe existir separación:

```text
PDOException
    │
    ▼
PdoMySqlErrorAdapter
    │
    ▼
NativeDatabaseError
    │
    ▼
MySql/MariaDb Failure Classifier
    │
    ▼
DatabaseFailure
```

---

# 234. Platform-specific classification

MySQL y MariaDB podrán tener classifiers distintos.

---

# 235. SQLSTATE

SQLSTATE será una fuente importante pero no la única.

---

# 236. Native error code

Podrá utilizarse para clasificación más precisa.

---

# 237. Error message parsing

Será último recurso.

No deberá ser la estrategia principal cuando exista código estructurado.

---

# 238. Constraint violation

Debe intentarse extraer:

```text
constraint name
table
column
```

cuando sea confiable.

---

# 239. Error normalization

ORM y Validation integration recibirán errores neutrales.

---

# 240. Unique violation

Ejemplo:

```text
Native Error
    │
    ▼
UniqueConstraintFailure
```

---

# 241. Foreign key violation

```text
Native Error
    │
    ▼
ForeignKeyConstraintFailure
```

---

# 242. Deadlock

```text
Native Error
    │
    ▼
DeadlockFailure
```

---

# 243. Retry integration

Resilience System podrá decidir:

```text
retry
backoff
fail
```

según:

```text
failure classification
transaction state
idempotency
policy
```

---

# 244. Query cancellation

Driver capabilities determinarán mecanismos disponibles.

Platform podrá aportar semántica adicional.

---

# 245. Timeout categories

Separar:

```text
connection timeout
pool acquisition timeout
statement timeout
lock timeout
transaction timeout
```

---

# 246. No generic timeout

Evitar:

```text
timeout = 30
```

aplicado ambiguamente a todo.

---

# 247. Statement timeout

Si la plataforma no tiene un mecanismo uniforme, deberá modelarse capability-driven.

---

# 248. Lock timeout

Separado de query timeout.

---

# 249. Connection timeout

Pertenece principalmente a Driver/Connection.

---

# 250. Platform-specific options

Configuración avanzada podrá usar:

```yaml
platform_options:
  mysql:
    ...
```

o equivalente tipado.

---

# 251. Option schemas

Cada Platform registrará:

```text
PlatformConfigurationSchema
```

---

# 252. Strict validation

Unknown options deberán rechazarse en strict mode.

---

# 253. No configuration bleed

Una opción MariaDB no deberá llegar accidentalmente a MySqlPlatform.

---

# 254. SQL hints

Los hints específicos de optimizador serán escape hatches avanzados.

---

# 255. Generic optimizer

El optimizador de VoltStack no deberá depender de hints MySQL/MariaDB por defecto.

---

# 256. Vendor hint AST

Podrá existir:

```text
PlatformQueryHint
```

con namespace explícito.

---

# 257. Hint portability

Un query con hint vendor-specific podrá marcarse:

```text
non-portable
```

---

# 258. Query metadata

Podrá incluir:

```text
portability level
required platform
required capabilities
```

---

# 259. Portable query

```text
requiredPlatform = null
```

---

# 260. MySQL-specific query

```text
requiredPlatform = mysql
```

cuando use una extensión exclusiva.

---

# 261. MariaDB-specific query

Igualmente:

```text
requiredPlatform = mariadb
```

---

# 262. Compiler selection

```text
ResolvedPlatform
+
ResolvedDialect
+
Query Plan
→
Compiler Strategy
```

---

# 263. Compiler classes

La arquitectura futura podrá incluir:

```text
MySqlSqlCompiler
MariaDbSqlCompiler
```

---

# 264. Shared compiler components

Podrán compartir:

```text
MySqlFamilyCompilerSupport
```

por composición.

---

# 265. No monolithic compiler switch

Evitar:

```php
switch ($platform) {
    case 'mysql':
    case 'mariadb':
}
```

en un compiler genérico gigante.

---

# 266. Schema compiler

Igualmente:

```text
MySqlSchemaCompiler
MariaDbSchemaCompiler
```

cuando las diferencias lo requieran.

---

# 267. Platform-specific planner strategies

Algunas estrategias podrán vivir en:

```text
Platform/MySql/Planning
Platform/MariaDb/Planning
```

---

# 268. Generic planner first

El Planner genérico deberá resolver la mayoría de decisiones mediante capabilities.

---

# 269. Vendor strategy only when necessary

Código específico sólo cuando la semántica no pueda expresarse suficientemente mediante capability/configuration.

---

# 270. Metadata architecture

Platform metadata deberá ser:

```text
immutable
typed
cacheable
```

cuando sea posible.

---

# 271. Metadata cache

Podrá almacenar:

```text
server identity
server version
capability profile
schema introspection metadata
```

con claves separadas.

---

# 272. No mixed cache

No crear:

```text
mysql_metadata_cache
```

que mezcle:

```text
server metadata
schema metadata
ORM metadata
query plans
```

---

# 273. Cache invalidation

Cada cache tendrá fingerprint/generation propios.

---

# 274. ORM metadata independence

Entity metadata no dependerá directamente de MySqlPlatform.

---

# 275. Platform-specific mapping

Sólo la fase de persistence/schema compilation utilizará Platform.

---

# 276. ORM portability

Una misma entidad deberá poder persistirse sobre MySQL o MariaDB cuando utilice tipos/features portables.

---

# 277. ORM platform requirements

Una entidad podrá declarar requisitos avanzados cuando utilice:

```text
generated columns
native enum
JSON-specific behavior
```

---

# 278. Mapping validation

Metadata Compiler podrá producir:

```text
PlatformRequirementSet
```

---

# 279. Deployment validation

Antes de ejecutar la aplicación podrá verificarse:

```text
ORM mapping requirements
vs
target capability profile
```

---

# 280. Repository independence

Repositories no deberán tener ramas:

```php
if ($db === 'mysql') {
}
```

para comportamiento portable.

---

# 281. Raw SQL

VoltStack seguirá permitiendo:

```php
DB::statement('...');
```

---

# 282. Raw SQL semantics

Raw SQL:

```text
bypasses
AST
Semantic Engine
Optimizer
Planner
```

pero conserva:

```text
connection resolution
binding
transaction context
telemetry
security policies where applicable
error normalization
```

---

# 283. Raw vendor SQL

El usuario será responsable de su portabilidad.

---

# 284. Raw SQL target metadata

Opcionalmente podrá declararse:

```text
required platform
```

para detectar ejecución accidental sobre otra plataforma.

---

# 285. Stored procedures

El soporte deberá modelarse como capability específica.

---

# 286. Stored procedure abstraction

No será requisito del Query Builder portable inicial.

---

# 287. Multiple result sets

Principalmente:

```text
Driver capability
+
Platform behavior
```

---

# 288. Functions

Funciones SQL comunes deberán modelarse semánticamente cuando sea práctico.

---

# 289. Function registry

Podrá existir posteriormente:

```text
SemanticFunctionRegistry
```

---

# 290. Platform function mapping

Ejemplo conceptual:

```text
SemanticFunction
    │
    ▼
Platform/Dialect Function Mapping
    │
    ▼
SQL Function
```

---

# 291. Vendor-specific functions

Disponibles mediante extension namespace.

---

# 292. Full-text search

Podrá modelarse como extensión/capability:

```text
search.fulltext
```

---

# 293. Full-text semantics

No asumir equivalencia exacta entre:

```text
MySQL full-text
MariaDB full-text
PostgreSQL full-text
```

---

# 294. Spatial data

Igual tratamiento:

```text
geo.*
```

mediante extensión futura.

---

# 295. Sequence support

Si una plataforma ofrece mecanismos de sequence distintos, deberán modelarse como capability.

---

# 296. Auto-increment ≠ sequence

No tratar ambos conceptos como idénticos.

---

# 297. Generated identifier abstraction

El ORM consumirá una abstracción superior.

---

# 298. Last inserted ID

Debe considerarse una capacidad del camino:

```text
Driver
+
Platform
+
Persistence Strategy
```

---

# 299. Connection initialization

Pipeline:

```text
Driver Connect
    │
    ▼
Native Connection
    │
    ▼
Platform Discovery
    │
    ▼
Platform Validation
    │
    ▼
Session Initialization
    │
    ▼
Capability Resolution
    │
    ▼
READY
```

---

# 300. Lazy discovery optimization

Discovery podrá diferirse hasta que una operación requiera información no conocida offline.

---

# 301. Minimum connection initialization

No ejecutar decenas de queries de discovery por cada nueva conexión.

---

# 302. Shared server metadata

Cuando varias conexiones apuntan al mismo target compatible, cierta metadata podrá compartirse de forma segura.

---

# 303. Sharing key

Debe utilizar:

```text
ServerIdentityFingerprint
```

no simplemente:

```text
host
```

---

# 304. ServerIdentity

Podrá considerar:

```text
endpoint
platform
server UUID/identity if available
version
configuration generation
```

sin secretos.

---

# 305. Connection pool interaction

PoolKey deberá considerar al menos:

```text
driver
endpoint
database
credential identity/generation
session profile
platform compatibility
configuration generation
```

---

# 306. MySQL/MariaDB pool separation

Nunca mezclar en un mismo pool recursos resueltos como plataformas incompatibles.

---

# 307. Platform mismatch during acquisition

Si un nuevo recurso del pool resuelve una plataforma distinta:

```text
discard
+
report configuration failure
```

---

# 308. Capability mismatch in pool

Si un recurso tiene capability fingerprint incompatible:

```text
do not lease as equivalent
```

---

# 309. Runtime server upgrades

Un rolling upgrade puede producir temporalmente:

```text
multiple server versions
```

---

# 310. Upgrade policy

Pool/Topology podrán:

```text
accept compatible capability profile
```

o:

```text
segregate generations
```

---

# 311. Capability-first compatibility

Dos versiones distintas pueden ser compatibles si satisfacen el mismo requirement set.

---

# 312. Exact-version mode

Migration/admin tooling podrá exigir versión exacta cuando sea necesario.

---

# 313. Production deployment

Recommended flow:

```text
Configuration
    │
    ▼
Offline Validation
    │
    ▼
Application Boot
    │
    ▼
Readiness Verification
    │
    ▼
Server Discovery
    │
    ▼
Platform Verification
    │
    ▼
Capability Verification
    │
    ▼
Application Ready
```

---

# 314. Readiness vs liveness

Separar:

```text
process alive
```

de:

```text
database target compatible
```

---

# 315. Health check levels

Podrán existir:

```text
BASIC
CONNECTIVITY
PLATFORM
CAPABILITY
TRANSACTIONAL
```

---

# 316. BASIC

No requiere necesariamente conexión.

---

# 317. CONNECTIVITY

Verifica que pueda abrirse una conexión.

---

# 318. PLATFORM

Verifica identidad/version.

---

# 319. CAPABILITY

Verifica requisitos declarados.

---

# 320. TRANSACTIONAL

Podrá ejecutar comprobaciones controladas en entornos apropiados.

---

# 321. Testing strategy

La integración deberá tener pruebas unitarias, integración y conformance.

---

# 322. Unit tests

Para:

```text
version parsers
platform resolver
capability rules
type mappings
schema canonicalization
error classification
session profile compilation
```

---

# 323. Integration tests

Contra instancias reales de:

```text
MySQL
MariaDB
```

---

# 324. Version matrix

CI deberá permitir probar múltiples versiones soportadas.

---

# 325. MySQL conformance suite

```text
MySqlPlatformConformanceSuite
```

---

# 326. MariaDB conformance suite

```text
MariaDbPlatformConformanceSuite
```

---

# 327. Shared family conformance

Podrá existir:

```text
MySqlFamilyConformanceSuite
```

para comportamiento genuinamente común.

---

# 328. Capability conformance

Cada capability NATIVE deberá tener prueba.

---

# 329. Unsupported conformance

También deberá probarse que features marcadas como:

```text
UNSUPPORTED
```

no se seleccionan accidentalmente.

---

# 330. Emulation conformance

Cada estrategia EMULATED deberá demostrar equivalencia.

---

# 331. Schema round-trip tests

```text
create
→ introspect
→ canonicalize
→ diff
```

deberá producir:

```text
no unintended diff
```

---

# 332. Transaction tests

Cubrir:

```text
commit
rollback
savepoints
nested behavior
deadlocks
lock timeout
cleanup
```

---

# 333. Pool tests

Cubrir:

```text
session contamination
transaction contamination
timezone contamination
charset contamination
tainted native access
reset failure
```

---

# 334. Persistent worker tests

Simular:

```text
Request A
RESET
Request B
```

repetidamente.

---

# 335. Concurrency tests

Especialmente relevantes para futuras integraciones:

```text
OpenSwoole
Fibers
coroutines
```

---

# 336. Platform switching tests

Mismo proceso:

```text
Connection A → MySQL
Connection B → MariaDB
```

sin contaminación.

---

# 337. Tenant switching tests

Cuando Multitenancy esté instalado:

```text
Tenant A → MySQL
Tenant B → MariaDB
```

con contexts aislados.

---

# 338. Error tests

Verificar clasificación de:

```text
unique constraint
foreign key
not null
deadlock
timeout
authentication
network
syntax
```

---

# 339. Security tests

Verificar:

```text
credential redaction
TLS configuration
DSN redaction
diagnostic redaction
error redaction
```

---

# 340. TLS architecture

TLS pertenece principalmente a:

```text
Connection Security
+
Driver Configuration
```

pero Platform podrá contribuir a validación semántica.

---

# 341. TLS policy

Producción podrá exigir:

```text
REQUIRED
VERIFY_CA
VERIFY_FULL
```

según el modelo genérico adoptado.

---

# 342. Certificate references

Nunca almacenar private keys directamente en:

```text
PlatformDescriptor
CapabilityFingerprint
Telemetry
```

---

# 343. Authentication plugins

Diferencias de mecanismos de autenticación deberán permanecer principalmente en Driver/Connection Security.

---

# 344. Platform capability interaction

Si un mecanismo depende del servidor, podrá expresarse mediante capabilities técnicas específicas.

---

# 345. Secret rotation

Una rotación deberá actualizar:

```text
CredentialGeneration
```

sin requerir cambiar:

```text
PlatformId
```

---

# 346. Pool rotation

Recursos con credential generation anterior deberán drenarse.

---

# 347. Platform fingerprint

Podrá existir:

```text
PlatformFingerprint
```

---

# 348. Platform fingerprint inputs

Ejemplo:

```text
PlatformId
ServerVersion
RelevantServerConfiguration
CompatibilityProfile
```

---

# 349. Capability fingerprint

Más específico:

```text
PlatformFingerprint
+
DriverCapabilityFingerprint
+
Policy
+
Extensions
+
Connection restrictions
```

---

# 350. Session fingerprint

Separado:

```text
SessionProfileFingerprint
```

---

# 351. Separation of fingerprints

No crear un único:

```text
DatabaseFingerprint
```

para todo.

---

# 352. Reason

Diferentes caches tienen diferentes criterios de invalidez.

---

# 353. Query compilation cache

Necesita:

```text
DialectFingerprint
CapabilityFingerprint
CompilerVersion
QueryFingerprint
```

---

# 354. Schema cache

Necesita:

```text
PlatformFingerprint
SchemaMetadataGeneration
```

---

# 355. Pool key

Necesita:

```text
ConnectionDefinitionFingerprint
CredentialGeneration
SessionProfileFingerprint
```

---

# 356. ORM metadata cache

Normalmente no necesita servidor exacto salvo metadata especializada.

---

# 357. Extension architecture

Paquetes podrán extender MySQL/MariaDB mediante extension points oficiales.

---

# 358. Extension examples

```text
custom type
custom function
custom capability
schema option
compiler strategy
metadata mapping
```

---

# 359. No monkey patching

Extensiones no deberán reemplazar internals arbitrariamente.

---

# 360. MySQL extension namespace

Ejemplo:

```text
VoltStack\Quantum\Database\Platform\MySql\Extension
```

---

# 361. MariaDB extension namespace

```text
VoltStack\Quantum\Database\Platform\MariaDb\Extension
```

---

# 362. Official extension registration

Podrá registrar:

```text
CapabilityProvider
TypeMapping
SemanticFunction
CompilerExtension
SchemaExtension
```

---

# 363. Capability requirements

Cada extensión podrá declarar:

```text
platform requirements
version range
driver requirements
capability requirements
```

---

# 364. Extension validation

Se realizará durante bootstrap cuando sea posible y al resolver el target cuando requiera metadata online.

---

# 365. Namespace propuesto

```text
VoltStack\Quantum\Database\Platform
```

---

# 366. Estructura propuesta

```text
Platform/
├── Contract/
├── Capability/
├── Resolution/
│
├── MySqlFamily/
│   ├── Discovery/
│   ├── Driver/
│   ├── Metadata/
│   ├── Session/
│   ├── Type/
│   ├── Schema/
│   ├── Error/
│   └── Support/
│
├── MySql/
│   ├── MySqlPlatform.php
│   ├── MySqlPlatformDescriptor.php
│   ├── MySqlPlatformResolver.php
│   ├── Capability/
│   ├── Dialect/
│   ├── Type/
│   ├── Schema/
│   ├── Transaction/
│   ├── Session/
│   ├── Error/
│   ├── Planning/
│   └── Testing/
│
└── MariaDb/
    ├── MariaDbPlatform.php
    ├── MariaDbPlatformDescriptor.php
    ├── MariaDbPlatformResolver.php
    ├── Capability/
    ├── Dialect/
    ├── Type/
    ├── Schema/
    ├── Transaction/
    ├── Session/
    ├── Error/
    ├── Planning/
    └── Testing/
```

---

# 367. Dialect location

Dependiendo de la estructura final, los Dialects podrán permanecer en:

```text
Dialect/MySql/
Dialect/MariaDb/
```

en vez de dentro de Platform.

La regla importante será:

```text
ownership conceptual
>
directory convenience
```

---

# 368. Driver location

El driver PDO seguirá en:

```text
Driver/Pdo/MySql/
```

y no dentro de Platform.

---

# 369. Dependency direction

```text
MySqlPlatform
MariaDbPlatform
      │
      ▼
Platform Contracts
Capability Contracts
Type Contracts
Schema Contracts
```

No deberán depender de:

```text
ORM EntityManager
Repository
UnitOfWork
Application
HTTP
Controller
```

---

# 370. Platform services

Preferentemente:

```text
immutable
stateless
reentrant
```

---

# 371. Request-specific state

Nunca dentro de:

```text
MySqlPlatform
MariaDbPlatform
MySqlDialect
MariaDbDialect
```

---

# 372. Connection-specific metadata

Deberá vivir en:

```text
ResolvedPlatform
CapabilitySnapshot
ConnectionContext
```

---

# 373. No mutable server version

Prohibido:

```php
$mySqlPlatform->setServerVersion(...);
```

si el mismo objeto puede ser compartido.

---

# 374. Correct model

```php
$resolvedPlatform = new ResolvedPlatform(
    platform: $mySqlPlatform,
    version: $serverVersion,
    capabilities: $snapshot,
);
```

conceptualmente.

---

# 375. Multiple versions simultaneously

Un worker podrá manejar:

```text
MySQL 8.x target A
MySQL future-version target B
MariaDB target C
```

sin crear estado global conflictivo.

---

# 376. Capability snapshot belongs to target context

No a la clase Platform singleton.

---

# 377. Compatibility policy

VoltStack deberá declarar formalmente qué versiones soporta.

---

# 378. Supported version range

No deberá codificarse dispersamente.

---

# 379. PlatformSupportPolicy

Podrá existir:

```text
PlatformSupportPolicy
```

con:

```text
minimum supported version
tested versions
deprecated versions
unsupported versions
```

---

# 380. Version support ≠ feature support

Una versión soportada puede no tener todas las features avanzadas.

---

# 381. Deprecated platform version

Podrá generar:

```text
PlatformVersionDeprecationWarning
```

sin alterar capabilities artificialmente.

---

# 382. Unsupported version

Podrá fallar durante readiness/connection resolution.

---

# 383. Future unknown version

Policy configurable:

```text
allow with warning
strict reject
compatibility mode
```

---

# 384. Conservative future handling

No asumir que una versión futura mantiene exactamente todos los comportamientos internos.

---

# 385. Platform compatibility tests

CI deberá validar:

```text
minimum supported
representative current
latest supported
```

según política de releases.

---

# 386. Documentation generation

Los capability descriptors podrán generar documentación específica:

```text
MySQL Support Matrix
MariaDB Support Matrix
```

---

# 387. Version-aware documentation

La documentación podrá indicar:

```text
capability available from version X
```

cuando corresponda.

---

# 388. No duplicated manual matrix

La fuente de verdad deberá ser:

```text
Capability Rules
+
Conformance Tests
```

---

# 389. Developer diagnostics

Ejemplo conceptual:

```text
$ php volt database:platform primary

Connection: primary
Driver: pdo.mysql
Platform: mysql
Version: <resolved>
Dialect: mysql
Capability Profile: <fingerprint>
Session Profile: <fingerprint>
```

---

# 390. MariaDB diagnostic

```text
Connection: reporting
Driver: pdo.mysql
Platform: mariadb
Version: <resolved>
Dialect: mariadb
Capability Profile: <fingerprint>
```

Esto deja visible la separación correcta.

---

# 391. Explain platform

Comando futuro:

```text
php volt database:platform:explain primary
```

podrá mostrar:

```text
Configured alias
Resolved driver
Configured platform hint
Detected server
Resolved platform
Version
Dialect
Capability source
Compatibility status
```

---

# 392. Explain feature

```text
php volt database:capability query.returning.insert --connection=primary
```

---

# 393. Security redaction

Nunca mostrar:

```text
password
secret DSN
private key
credential contents
```

---

# 394. Telemetry

Atributos seguros:

```text
db.system
db.platform
db.driver
db.operation
db.connection.role
```

según las convenciones de Telemetry adoptadas.

---

# 395. Version telemetry

Server version deberá exponerse sólo cuando la política de observabilidad lo permita.

---

# 396. Query telemetry

No necesita vendor branches.

Recibe:

```text
ResolvedPlatformDescriptor
```

---

# 397. Slow query telemetry

Será independiente de MySQL/MariaDB, aunque podrá adjuntar metadata segura de Platform.

---

# 398. Security architecture

Platform integration deberá respetar:

```text
prepared statements
safe identifier handling
credential isolation
TLS policy
secret redaction
session isolation
```

---

# 399. SQL injection

Platform/Dialect nunca concatenarán valores de usuario como literals sin pasar por mecanismos explícitos.

---

# 400. Identifier injection

Identifiers dinámicos deberán pasar por:

```text
Identifier
+
Dialect Quoter
```

---

# 401. Native options security

Opciones peligrosas deberán:

```text
validate
document
require explicit opt-in
```

cuando corresponda.

---

# 402. Production secure defaults

Configuración inicial deberá favorecer:

```text
prepared statements
strict binding
TLS where configured/required
strict reset
credential references
redacted telemetry
```

---

# 403. Resilience

Platform deberá proporcionar clasificación suficiente para:

```text
deadlock retry
connection retry
failover
lock timeout handling
```

sin implementar directamente la política.

---

# 404. Failover compatibility

Antes de aceptar target alternativo:

```text
required capabilities
⊆
target capabilities
```

---

# 405. Read/write safety

Una replica nunca deberá seleccionarse para write simplemente porque utiliza el mismo Driver.

---

# 406. Platform capability safety

Igualmente:

```text
same platform
≠
same effective capabilities
```

---

# 407. Architectural invariants

## DB-MYSQL-001

MySQL y MariaDB serán plataformas distintas.

## DB-MYSQL-002

Podrán compartir `pdo.mysql` como Driver.

## DB-MYSQL-003

Driver ID no determinará por sí solo Platform ID.

## DB-MYSQL-004

`mysql` y `mariadb` podrán funcionar como aliases de configuración de alto nivel.

## DB-MYSQL-005

Aliases de configuración se normalizarán a Driver + Platform + Dialect.

## DB-MYSQL-006

Server discovery validará el Platform configurado cuando exista conexión.

## DB-MYSQL-007

Platform mismatch claro no será ignorado silenciosamente.

## DB-MYSQL-008

Version parsing será vendor-specific y tipado.

## DB-MYSQL-009

No se compararán versiones MySQL y MariaDB como una secuencia común.

## DB-MYSQL-010

Capabilities serán version-aware.

## DB-MYSQL-011

Capabilities no se codificarán mediante vendor checks en capas superiores.

## DB-MYSQL-012

MySQL y MariaDB tendrán Capability Providers independientes.

## DB-MYSQL-013

Sólo comportamiento genuinamente común irá a `MySqlFamily`.

## DB-MYSQL-014

Se preferirá composición sobre herencia profunda.

## DB-MYSQL-015

MySqlPlatform y MariaDbPlatform no abrirán conexiones.

## DB-MYSQL-016

Platform no ejecutará SQL de aplicación.

## DB-MYSQL-017

Dialect será responsable de sintaxis, no de selección estratégica.

## DB-MYSQL-018

Planner será responsable de seleccionar estrategias.

## DB-MYSQL-019

Query Builder permanecerá vendor-neutral.

## DB-MYSQL-020

ORM permanecerá vendor-neutral para operaciones portables.

## DB-MYSQL-021

Generated-key retrieval no será confundido con SQL RETURNING.

## DB-MYSQL-022

JSON será modelado mediante capabilities granulares.

## DB-MYSQL-023

Schema introspection producirá el modelo canónico de VoltStack.

## DB-MYSQL-024

Metadata nativa podrá preservarse separadamente.

## DB-MYSQL-025

Schema canonicalization evitará diffs falsos.

## DB-MYSQL-026

Transactional DDL será capability-driven.

## DB-MYSQL-027

Implicit commits deberán ser conocidos por Migration/Transaction planning.

## DB-MYSQL-028

Session state será explícitamente administrado.

## DB-MYSQL-029

SQL mode no podrá contaminar requests posteriores.

## DB-MYSQL-030

Timezone no podrá contaminar requests posteriores.

## DB-MYSQL-031

Charset/collation state deberá permanecer conocido.

## DB-MYSQL-032

Una conexión con transacción activa no volverá al pool.

## DB-MYSQL-033

Una conexión tainted no será reutilizada sin reset verificable.

## DB-MYSQL-034

Platform objects no contendrán current request state.

## DB-MYSQL-035

Platform objects no contendrán current tenant state.

## DB-MYSQL-036

Platform objects no contendrán current transaction state.

## DB-MYSQL-037

ResolvedPlatform podrá variar por target dentro del mismo worker.

## DB-MYSQL-038

CapabilitySnapshot será target-specific e inmutable.

## DB-MYSQL-039

Un worker podrá manejar MySQL y MariaDB simultáneamente.

## DB-MYSQL-040

Failover deberá verificar compatibilidad de plataforma/capabilities.

## DB-MYSQL-041

Replicas no serán asumidas homogéneas.

## DB-MYSQL-042

Routing podrá ser capability-aware.

## DB-MYSQL-043

Platform discovery será lazy cuando sea posible.

## DB-MYSQL-044

Bootstrap no abrirá conexiones sólo para descubrir versiones.

## DB-MYSQL-045

Offline compilation podrá utilizar target platform/version declarados.

## DB-MYSQL-046

Runtime verification podrá detectar drift.

## DB-MYSQL-047

Error classification estará centralizada.

## DB-MYSQL-048

Retry policy no pertenecerá a Platform.

## DB-MYSQL-049

Platform-specific options serán tipadas/validadas.

## DB-MYSQL-050

Raw SQL seguirá siendo escape hatch explícito.

## DB-MYSQL-051

Vendor-specific queries podrán declarar target Platform.

## DB-MYSQL-052

Platform fingerprints no contendrán secretos.

## DB-MYSQL-053

Capability fingerprints no contendrán secretos.

## DB-MYSQL-054

Session fingerprints serán independientes de capability fingerprints.

## DB-MYSQL-055

Pool keys distinguirán configuraciones incompatibles.

## DB-MYSQL-056

MySQL y MariaDB tendrán conformance suites separadas.

## DB-MYSQL-057

Cada capability NATIVE deberá ser verificable.

## DB-MYSQL-058

Cada emulación deberá probar equivalencia semántica.

## DB-MYSQL-059

Schema round-trip deberá evitar migraciones espurias.

## DB-MYSQL-060

Persistent worker tests serán obligatorios.

---

# 408. Anti-pattern — MariaDB as MySQL version

Incorrecto:

```text
MariaDB
=
MySQL with different version number
```

---

# 409. Anti-pattern — Driver determines Platform

Incorrecto:

```php
if ($driver === 'pdo.mysql') {
    $platform = 'mysql';
}
```

---

# 410. Anti-pattern — MariaDB inheritance everywhere

Incorrecto:

```text
MariaDbPlatform extends MySqlPlatform
```

si eso provoca herencia accidental de semántica incorrecta.

---

# 411. Anti-pattern — Vendor branches in ORM

Incorrecto:

```php
if ($platform === 'mariadb') {
    $unitOfWork->...
}
```

---

# 412. Anti-pattern — Query Builder emits native UPSERT

Incorrecto:

```php
$query .= ' ON DUPLICATE KEY UPDATE ...';
```

desde Builder.

---

# 413. Anti-pattern — Global server version

Incorrecto:

```php
MySqlPlatform::$version = ...;
```

---

# 414. Anti-pattern — Global SQL mode

Incorrecto:

```php
Database::$sqlMode = ...;
```

---

# 415. Anti-pattern — Reuse without reset

Incorrecto:

```text
Request A
→ modifies session

Request B
→ receives same dirty session
```

---

# 416. Anti-pattern — One capability matrix forever

Incorrecto:

```text
MySQL:
    returning = false

MariaDB:
    returning = true
```

como tabla estática universal sin:

```text
version
operation
driver
context
```

---

# 417. Anti-pattern — Generated ID equals RETURNING

Incorrecto:

```text
lastInsertId available
=
RETURNING supported
```

---

# 418. Anti-pattern — Same protocol means same semantics

Incorrecto:

```text
MySQL-compatible protocol
=
MySQL Platform
```

---

# 419. Anti-pattern — Platform as service locator

Incorrecto:

```php
$platform->container()->get(...);
```

---

# 420. Anti-pattern — Platform owns current Connection

Incorrecto:

```php
$platform->setConnection($connection);
```

---

# 421. Anti-pattern — Platform owns pool

Incorrecto:

```php
$platform->pool()->acquire();
```

---

# 422. Anti-pattern — Platform owns ORM

Incorrecto:

```php
$platform->entityManager();
```

---

# 423. Anti-pattern — Error parsing everywhere

Incorrecto:

```text
ORM parses MySQL error text
Migration parses MySQL error text
Transaction parses MySQL error text
```

---

# 424. Correct error architecture

```text
Native Error
     │
     ▼
Central Adapter/Classifier
     │
     ▼
Neutral Database Failure
     │
     ├── ORM
     ├── Migration
     ├── Transaction
     └── Resilience
```

---

# 425. Reference architecture

```text
                       APPLICATION
                            │
                            ▼
                      DATABASE API
                            │
                ┌───────────┼───────────┐
                ▼           ▼           ▼
             Query         ORM        Schema
                │           │           │
                └──────┬────┴─────┬─────┘
                       ▼          ▼
                    Planner   Migration Planner
                       │          │
                       └────┬─────┘
                            ▼
                   CapabilitySnapshot
                            │
                  ┌─────────┴─────────┐
                  ▼                   ▼
           ResolvedPlatform      ResolvedDialect
                  │                   │
          ┌───────┴───────┐           │
          ▼               ▼           │
   MySqlPlatform     MariaDbPlatform   │
          │               │           │
          └───────┬───────┘           │
                  ▼                   ▼
            SQL Compilation ──────────┘
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
            PdoMySqlDriver
                  │
                  ▼
         Native PDO Connection
                  │
          ┌───────┴───────┐
          ▼               ▼
        MySQL           MariaDB
```

---

# 426. Runtime architecture

```text
Application
    │
    ▼
Worker
    │
    ├── Immutable MySqlPlatform
    ├── Immutable MariaDbPlatform
    ├── Immutable Dialects
    ├── Frozen Capability Rules
    │
    ├── Execution Scope A
    │      ├── Connection: primary
    │      ├── ResolvedPlatform: MySQL
    │      └── CapabilitySnapshot A
    │
    ├── RESET
    │
    └── Execution Scope B
           ├── Connection: tenant
           ├── ResolvedPlatform: MariaDB
           └── CapabilitySnapshot B
```

---

# 427. Semantic query architecture

```text
Laravel-like API
      │
      ▼
Query Builder
      │
      ▼
Semantic Query AST
      │
      ▼
Semantic Analysis
      │
      ▼
Capability Requirements
      │
      ▼
Planner
      │
      ├── MySQL capabilities
      │       or
      └── MariaDB capabilities
              │
              ▼
        Execution Strategy
              │
              ▼
            Compiler
              │
              ▼
       MySQL/MariaDB Dialect
              │
              ▼
              SQL
```

---

# 428. Schema architecture

```text
Schema Builder
      │
      ▼
Canonical Schema Model
      │
      ▼
Schema Planner
      │
      ├── Platform Semantics
      └── CapabilitySnapshot
              │
              ▼
        Schema Operations
              │
              ▼
        Schema Compiler
              │
              ▼
        Resolved Dialect
              │
              ▼
              SQL
```

---

# 429. Persistence architecture

```text
Entity
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
  ├── Generated Value Requirements
  ├── Bulk Requirements
  └── CapabilitySnapshot
          │
          ▼
      Query Model
          │
          ▼
      Query Engine
```

---

# 430. Platform family architecture

La relación final será:

```text
                  MySqlFamily
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
    Discovery      Shared        Shared
                  Utilities      Testing
        │
        ├─────────────────────────────┐
        ▼                             ▼
 MySqlPlatform                 MariaDbPlatform
        │                             │
        ├── Capabilities              ├── Capabilities
        ├── Semantics                 ├── Semantics
        ├── Schema                    ├── Schema
        ├── Types                     ├── Types
        ├── Transactions              ├── Transactions
        └── Errors                    └── Errors
```

---

# 431. Regla maestra MySQL-family

> **MySQL y MariaDB pueden compartir infraestructura, pero nunca compartirán semántica por simple suposición.**

---

# 432. Fórmula de resolución

```text
Connection Configuration
        │
        ▼
Driver Resolution
        │
        ▼
pdo.mysql
        │
        ▼
Server Discovery
        │
        ├──────────────┐
        ▼              ▼
      MySQL          MariaDB
        │              │
        ▼              ▼
Platform Semantics + Version Rules
        │              │
        └───────┬──────┘
                ▼
       CapabilitySnapshot
                │
                ▼
          Query/Schema/ORM
```

---

# 433. Fórmula de compatibilidad

```text
Target Compatibility
=
Platform Identity
+
Version Compatibility
+
Required Capabilities
+
Driver Compatibility
+
Session Compatibility
+
Policy
```

No:

```text
Target Compatibility
=
same PDO driver
```

---

# 434. Resultado arquitectónico

La arquitectura permitirá que VoltStack tenga una experiencia simple:

```php
DB::table('users')
    ->where('active', true)
    ->get();
```

mientras internamente mantiene:

```text
Query Intent
    │
    ▼
AST
    │
    ▼
Semantic Analysis
    │
    ▼
Planner
    │
    ▼
CapabilitySnapshot
    │
    ├──────────────┐
    ▼              ▼
 MySQL          MariaDB
    │              │
    ▼              ▼
Strategy        Strategy
    │              │
    ▼              ▼
Dialect         Dialect
    │              │
    └──────┬───────┘
           ▼
           SQL
```

sin introducir vendor coupling en la API pública.

---

# 435. Relación con documentos anteriores

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
```

---

# 436. Responsabilidades definitivas

```text
PdoMySqlDriver
=
native communication

MySqlDialect / MariaDbDialect
=
SQL syntax

MySqlPlatform / MariaDbPlatform
=
database semantics

Capability Providers
=
effective technical support

Connection
=
logical managed access

Planner
=
strategy selection

Compiler
=
SQL translation

Executor
=
execution

TransactionManager
=
transaction orchestration

SchemaPlanner
=
schema strategy

PersistencePlanner
=
ORM persistence strategy
```

---

# 437. Decisión final

VoltStack adoptará:

```text
Shared Transport
+
Separated Platforms
+
Separated Capability Profiles
+
Separated Dialects where semantics/syntax diverge
+
Shared Infrastructure by Composition
```

como arquitectura oficial para MySQL y MariaDB.

Esto evita dos extremos:

```text
Extreme A:
duplicate absolutely everything

Extreme B:
pretend MySQL and MariaDB are identical
```

La solución será:

```text
share only proven common behavior
specialize everything that can diverge
```

---

# 438. Siguiente documento

El siguiente documento será:

```text
20_DATABASE_POSTGRESQL_PLATFORM.md
```

y deberá aplicar los mismos principios a PostgreSQL, incluyendo:

```text
PostgreSqlPlatform
PostgreSqlDialect
pdo.pgsql integration
server/version discovery
capability model
schema/catalog semantics
search_path
schemas
sequences
identity
RETURNING
ON CONFLICT
CTEs
recursive CTEs
window functions
JSON/JSONB
arrays
UUID
enums
domains
range types
generated columns
indexes
partial indexes
expression indexes
concurrent indexes
constraints
deferrable constraints
transactions
savepoints
isolation
locking
NOWAIT
SKIP LOCKED
advisory locks
transactional DDL
session state
error classification
SQLSTATE
persistent runtime reset
connection pooling
schema introspection
migration planning
capability fingerprints
conformance testing
```

manteniendo la misma regla:

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

# 439. Conclusión

El soporte MySQL/MariaDB de VoltStack no se diseñará como un conjunto de condicionales especiales distribuidos por el framework.

Se diseñará como una plataforma formal:

```text
Transport
      │
      ▼
Server Discovery
      │
      ▼
Platform Resolution
      │
      ▼
Capability Resolution
      │
      ▼
Semantic Planning
      │
      ▼
Dialect Compilation
      │
      ▼
Execution
```

La regla final será:

> **Compartir protocolo no significa compartir plataforma; compartir sintaxis no significa compartir semántica; y compartir semántica en una versión no garantiza compartirla para siempre.**

Por ello:

```text
MySQL
    │
    ├── own Platform
    ├── own capability rules
    ├── own version semantics
    └── own conformance suite

MariaDB
    │
    ├── own Platform
    ├── own capability rules
    ├── own version semantics
    └── own conformance suite
```

mientras ambos podrán reutilizar de forma controlada:

```text
pdo.mysql
+
MySqlFamily shared infrastructure
```

sin comprometer la independencia arquitectónica del sistema.