# 21_DATABASE_SQLITE_PLATFORM.md

# VoltStack Quantum Database
## SQLite Platform Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 21 — SQLite Platform  
**Estado:** Architecture Specification  
**Nivel:** Infrastructure / Platform / Capability  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura oficial de soporte para:

```text
SQLite
```

dentro de:

```text
VoltStack/Quantum/Database
```

El objetivo es proporcionar soporte SQLite de primera clase manteniendo las mismas fronteras arquitectónicas establecidas para MySQL, MariaDB y PostgreSQL.

La integración deberá cubrir:

- archivos SQLite;
- bases de datos en memoria;
- conexiones URI cuando estén soportadas;
- type affinity;
- `rowid`;
- `INTEGER PRIMARY KEY`;
- `AUTOINCREMENT`;
- foreign keys;
- PRAGMAs;
- journal modes;
- WAL;
- locking;
- busy handling;
- transactions;
- savepoints;
- CTE;
- window functions;
- UPSERT;
- `RETURNING`;
- JSON;
- generated columns;
- STRICT tables;
- partial indexes;
- expression indexes;
- schema introspection;
- migrations;
- table rebuilds;
- persistent runtimes;
- testing;
- capability discovery.

---

# 2. Principio fundamental

SQLite deberá cumplir:

```text
Driver
≠
Connection
≠
Dialect
≠
Platform
≠
Capability
```

Por tanto:

```text
PdoSqliteDriver
≠
SqliteDialect
≠
SqlitePlatform
≠
SqliteCapabilityProvider
```

---

# 3. SQLite no es simplemente otro PDO Driver

La arquitectura incorrecta sería:

```text
PDO
└── SQLite
    └── done
```

La arquitectura correcta será:

```text
SQLite
├── Driver
├── Dialect
├── Platform
├── Capabilities
├── Type Semantics
├── Schema Semantics
├── Transaction Semantics
├── Session/PRAGMA State
├── Locking Semantics
├── Migration Strategies
└── Runtime Semantics
```

---

# 4. Arquitectura general

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
Semantic Engine
    │
    ▼
Optimizer
    │
    ▼
Planner
    │
    ├── SQLite Capabilities
    └── SQLite Semantics
            │
            ▼
      Execution Plan
            │
            ▼
         Compiler
            │
            ▼
       SqliteDialect
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
      PdoSqliteDriver
            │
            ▼
          SQLite
```

---

# 5. Driver inicial

El Driver oficial inicial será:

```text
pdo.sqlite
```

implementado conceptualmente por:

```php
final class PdoSqliteDriver implements DriverInterface
{
}
```

---

# 6. Driver futuro

La arquitectura deberá permitir posteriormente:

```text
native.sqlite3
async.sqlite
specialized.sqlite
```

sin modificar:

```text
Query Builder
ORM
Schema API
Migration API
```

---

# 7. Platform ID

Identificador oficial:

```text
sqlite
```

---

# 8. Dialect ID

```text
sqlite
```

---

# 9. Aliases

La configuración podrá aceptar:

```text
sqlite
sqlite3
```

normalizados al Driver/Platform correspondiente.

---

# 10. Separación de responsabilidades

## Driver

Responsable de:

```text
native connection
statement preparation
parameter binding
execution primitives
native results
native errors
```

## Dialect

Responsable de:

```text
SQLite SQL syntax
identifier quoting
DDL syntax
UPSERT syntax
RETURNING syntax
locking-related SQL syntax where applicable
```

## Platform

Responsable de:

```text
SQLite semantics
type affinity
schema semantics
transaction semantics
locking model
rowid semantics
PRAGMA semantics
capabilities
migration constraints
```

---

# 11. SqlitePlatform

Componente:

```php
final class SqlitePlatform implements DatabasePlatformInterface
{
}
```

deberá ser preferentemente:

```text
immutable
stateless
reentrant
shareable
```

---

# 12. No current connection

Prohibido:

```php
$sqlitePlatform->setConnection($connection);
```

---

# 13. No mutable current version

Prohibido:

```php
$sqlitePlatform->setVersion($version);
```

La versión efectiva pertenece al:

```text
ResolvedPlatform
```

---

# 14. SQLite library version

Las capacidades SQLite dependen significativamente de la versión de la biblioteca SQLite utilizada por el Driver.

Por tanto deberá distinguirse:

```text
PHP version
PDO_SQLITE version
SQLite library version
VoltStack Driver version
```

---

# 15. ServerVersion

Aunque SQLite no sea un servidor cliente-servidor tradicional, VoltStack reutilizará conceptualmente:

```text
ServerVersion
```

o una abstracción equivalente:

```text
DatabaseEngineVersion
```

para describir la versión efectiva del motor.

---

# 16. SqliteVersion

Podrá existir:

```php
final readonly class SqliteVersion
{
    public function __construct(
        public int $major,
        public int $minor,
        public int $patch,
    ) {}
}
```

---

# 17. Version discovery

```text
Native Connection
      │
      ▼
SqliteEngineDiscovery
      │
      ▼
SQLite Version
      │
      ▼
Capability Resolver
      │
      ▼
CapabilitySnapshot
```

---

# 18. No version assumptions

Nunca asumir capabilities únicamente porque:

```text
driver = pdo.sqlite
```

---

# 19. Capability architecture

SQLite deberá integrarse con:

```text
18_DATABASE_PLATFORM_CAPABILITY_SYSTEM.md
```

---

# 20. Capability provider

```text
SqlitePlatformCapabilityProvider
```

---

# 21. Effective capability resolution

```text
SQLite Baseline
      │
      ▼
SQLite Version
      │
      ▼
Compile-Time Features
      │
      ▼
Driver Capabilities
      │
      ▼
Connection Configuration
      │
      ▼
PRAGMA / Runtime State
      │
      ▼
Policy
      │
      ▼
EffectiveCapabilitySet
```

---

# 22. Compile-time capabilities

SQLite puede ser compilado con diferentes opciones.

Por tanto algunas capacidades pueden depender de:

```text
SQLite build
```

además de la versión.

---

# 23. Compile option discovery

Podrá existir:

```text
SqliteCompileOptionDiscovery
```

---

# 24. Discovery policy

La detección de compile options será:

```text
lazy
cacheable
target-specific
```

y no deberá ejecutarse por cada query.

---

# 25. Capability categories

SQLite deberá describir al menos:

```text
Query
CTE
Window
UPSERT
RETURNING
JSON
Type
Schema
Indexes
Foreign Keys
Generated Columns
STRICT Tables
Transactions
Savepoints
Locking
Journal
WAL
PRAGMA
Temporary Objects
Introspection
Migration
Bulk
Runtime
```

---

# 26. Capability granularity

Evitar:

```text
sqliteModern = true
```

Preferir:

```text
query.cte
query.cte.recursive
query.window
query.upsert
query.returning.insert

schema.index.partial
schema.index.expression
schema.generated_column
schema.strict_table
```

---

# 27. Query Builder

El Query Builder seguirá siendo portable.

Ejemplo:

```php
DB::table('users')
    ->where('active', true)
    ->orderBy('name')
    ->get();
```

no sabrá que el target es SQLite.

---

# 28. Query pipeline

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
     ├── SQLite Capabilities
     └── SQLite Semantics
             │
             ▼
        Execution Plan
             │
             ▼
          Compiler
             │
             ▼
        SqliteDialect
             │
             ▼
             SQL
```

---

# 29. SqliteDialect

Será responsable únicamente de traducir decisiones semánticas a sintaxis SQLite.

---

# 30. Dialect responsibilities

Incluye:

```text
identifier quoting
DDL syntax
LIMIT/OFFSET syntax
UPSERT syntax
RETURNING syntax
generated column syntax
STRICT syntax
index syntax
```

---

# 31. Dialect non-responsibilities

No deberá decidir:

```text
whether table rebuild is required
whether WAL should be enabled
whether foreign keys should be enabled
whether a transaction should retry
whether a query should use a capability
```

---

# 32. Identifier quoting

Identifiers permanecerán estructurados hasta Compiler.

```text
Identifier
   │
   ▼
Compiler
   │
   ▼
SqliteDialect
   │
   ▼
Quoted Identifier
```

---

# 33. No identifier concatenation

Nunca:

```php
$sql = 'SELECT * FROM "' . $table . '"';
```

con valores no validados.

---

# 34. Database identity

SQLite necesita un concepto de identidad distinto a motores cliente-servidor.

---

# 35. SQLite targets

Una base SQLite podrá ser:

```text
FileDatabase
MemoryDatabase
UriDatabase
TemporaryDatabase
```

---

# 36. SqliteEndpoint

Podrá existir:

```text
SqliteEndpoint
```

como variante de `DatabaseEndpoint`.

---

# 37. File database

Conceptualmente:

```php
final readonly class SqliteFileEndpoint
{
    public function __construct(
        public DatabaseFilePath $path,
    ) {}
}
```

---

# 38. Memory database

Deberá modelarse explícitamente:

```text
SqliteMemoryEndpoint
```

y no simplemente como un string mágico disperso.

---

# 39. `:memory:`

La sintaxis:

```text
:memory:
```

es un detalle de Driver/configuración.

Internamente deberá existir semántica explícita:

```text
MemoryDatabase
```

---

# 40. Critical memory rule

Una base:

```text
:memory:
```

normalmente está ligada a una conexión concreta.

Por tanto:

```text
Connection A :memory:
≠
Connection B :memory:
```

---

# 41. Pooling implication

Crear múltiples conexiones físicas a:

```text
:memory:
```

puede crear múltiples bases de datos independientes.

---

# 42. Memory pool policy

VoltStack deberá impedir configuraciones ambiguas como:

```text
memory database
+
pool max = 10
```

si eso produce semántica distinta a la esperada.

---

# 43. Memory database strategy

Podrán existir estrategias:

```text
PRIVATE_CONNECTION
SHARED_MEMORY_URI
PINNED_CONNECTION
TEST_SCOPE
```

según capacidades.

---

# 44. Shared memory

SQLite permite ciertos escenarios de memoria compartida mediante URI/configuración.

VoltStack deberá tratarlos explícitamente.

---

# 45. No hidden shared memory

Nunca convertir automáticamente:

```text
:memory:
```

en shared memory.

---

# 46. DatabaseFilePath

Las rutas deberán normalizarse.

---

# 47. Relative paths

Podrán resolverse durante Configuration System hacia una ruta canónica.

---

# 48. Runtime path independence

El Driver no deberá depender del:

```text
current working directory
```

para interpretar una ruta ya compilada.

---

# 49. Path security

Una ruta SQLite es información potencialmente sensible.

Diagnósticos podrán redaccionarla según política.

---

# 50. File lifecycle

Database core no deberá asumir ownership del archivo salvo que la operación lo declare explícitamente.

---

# 51. Existing database

Conectarse a un archivo existente no implica que VoltStack pueda:

```text
delete
move
replace
chmod
backup
```

el archivo.

---

# 52. File permissions

Errores de filesystem deberán clasificarse apropiadamente.

---

# 53. SQLite type model

Uno de los puntos más importantes es:

```text
SQLite declared type
≠
strict static type
```

en tablas tradicionales.

---

# 54. Type affinity

VoltStack deberá modelar:

```text
SQLite Type Affinity
```

como semántica de Platform.

---

# 55. Affinity categories

Conceptualmente:

```text
INTEGER
TEXT
BLOB
REAL
NUMERIC
```

según las reglas de SQLite.

---

# 56. DeclaredType

La introspección deberá preservar:

```text
declared type
```

además de la afinidad derivada.

---

# 57. Type metadata

Conceptualmente:

```text
SqliteColumnTypeMetadata
├── declaredType
├── affinity
├── strict
└── nativeDetails
```

---

# 58. Portable type mapping

```text
VoltStack Type
      │
      ▼
SqliteTypeMapping
      │
      ▼
Declared SQLite Type
      │
      ▼
SQLite Affinity
```

---

# 59. Type canonicalization

La introspección deberá evitar diferencias falsas entre tipos semánticamente equivalentes.

---

# 60. No false precision assumptions

VoltStack no deberá asumir que:

```text
VARCHAR(255)
```

impone necesariamente la misma semántica de longitud que otros motores.

---

# 61. String length metadata

La longitud declarada podrá preservarse como metadata, pero Platform deberá describir su enforcement real.

---

# 62. Boolean

SQLite no deberá obligar al dominio a pensar en:

```text
0 / 1
```

El Type System seguirá exponiendo:

```text
BooleanType
```

---

# 63. Boolean conversion

```text
BooleanType
      │
      ▼
SqliteTypeMapping
      │
      ▼
SQLite-compatible representation
```

---

# 64. Date/time

SQLite no posee el mismo modelo nativo temporal que PostgreSQL.

VoltStack deberá manejar fechas mediante Type System y mapping explícito.

---

# 65. DateTime storage strategies

Podrán existir:

```text
TEXT_ISO8601
INTEGER_UNIX
REAL_JULIAN
CUSTOM
```

según política.

---

# 66. Default datetime strategy

VoltStack deberá elegir una estrategia oficial determinista y documentada.

---

# 67. No environment-dependent dates

La conversión temporal no deberá depender implícitamente de:

```text
PHP default timezone
OS locale
worker state
```

---

# 68. Decimal values

SQLite affinity no elimina la necesidad de preservar semántica decimal en VoltStack.

---

# 69. Decimal strategy

El Type System podrá utilizar:

```text
string
Decimal ValueObject
custom converter
```

evitando pérdida de precisión accidental.

---

# 70. JSON

JSON en SQLite deberá modelarse como una combinación de:

```text
storage representation
+
JSON capabilities
```

---

# 71. JSON storage

Una columna JSON puede almacenarse utilizando una representación compatible, pero eso no implica automáticamente disponibilidad de funciones JSON.

---

# 72. JSON capability discovery

Capabilities podrán incluir:

```text
json.validation
json.extract
json.path
json.aggregate
json.modify
```

según la biblioteca disponible.

---

# 73. JSON semantic AST

Query AST seguirá utilizando expresiones semánticas cuando sean portables.

---

# 74. JSON-specific fallback

Si una operación JSON no está disponible:

```text
CapabilityNotSupportedException
```

o estrategia de emulación segura.

---

# 75. BLOB

VoltStack:

```text
BinaryType
```

se mapeará a almacenamiento BLOB apropiado.

---

# 76. UUID

SQLite no requiere un UUID type nativo para soportar:

```text
UuidType
```

---

# 77. UUID storage strategies

Podrán existir:

```text
TEXT
BINARY
CUSTOM
```

---

# 78. UUID portability

El ORM trabaja con:

```text
UuidType
```

no con la representación SQLite concreta.

---

# 79. Enum

PHP enums podrán mapearse mediante:

```text
TEXT
INTEGER
CHECK constraint
custom mapping
```

según metadata.

---

# 80. Native enum

No deberá declararse:

```text
type.enum.native
```

si SQLite no proporciona esa semántica.

---

# 81. STRICT tables

SQLite moderno soporta un modelo más estricto mediante:

```text
STRICT
```

cuando la versión lo permite.

---

# 82. STRICT capability

```text
schema.table.strict
```

---

# 83. StrictTableMetadata

Schema Model podrá contener una opción:

```text
strict = true
```

como metadata platform-aware.

---

# 84. Portable schema behavior

Una tabla portable no deberá volverse STRICT automáticamente salvo política explícita.

---

# 85. Strict schema policy

Podrá configurarse:

```text
DEFAULT
PREFER_STRICT
REQUIRE_STRICT
DISABLE_STRICT
```

---

# 86. Capability requirement

`REQUIRE_STRICT` deberá fallar si la versión efectiva no lo soporta.

---

# 87. `rowid`

SQLite posee semántica especial alrededor de:

```text
rowid
```

---

# 88. rowid abstraction

Platform deberá conocer:

```text
rowid tables
WITHOUT ROWID tables
INTEGER PRIMARY KEY aliases
```

---

# 89. ORM independence

ORM no deberá asumir que toda tabla SQLite posee un `rowid` utilizable.

---

# 90. INTEGER PRIMARY KEY

Deberá modelarse correctamente su relación con rowid.

---

# 91. AUTOINCREMENT

SQLite:

```text
AUTOINCREMENT
```

no deberá utilizarse automáticamente sólo porque una propiedad sea auto-generada.

---

# 92. Identifier strategy

ORM podrá expresar:

```text
GeneratedIdentifierStrategy::IDENTITY
```

---

# 93. Platform strategy

SQLite Platform determinará la representación apropiada.

---

# 94. AUTOINCREMENT policy

Podrá distinguirse:

```text
ROWID_IDENTITY
AUTOINCREMENT_IDENTITY
APPLICATION
CUSTOM
```

---

# 95. No automatic AUTOINCREMENT

Default recomendado:

```text
do not add AUTOINCREMENT unless semantics require it
```

---

# 96. WITHOUT ROWID

Capability:

```text
schema.table.without_rowid
```

---

# 97. Table option

Schema metadata podrá representar:

```text
withoutRowId
```

---

# 98. STRICT + WITHOUT ROWID

Las combinaciones deberán validarse mediante capabilities y reglas de versión.

---

# 99. Generated columns

Capability:

```text
schema.generated_column
```

---

# 100. GeneratedColumn

Schema Model podrá representar:

```text
expression
storage strategy
```

según capabilities.

---

# 101. Generated expression

Deberá utilizar AST de expresión cuando sea posible.

---

# 102. Schema architecture

SQLite se integrará con:

```text
Schema Model
Schema AST
Schema Planner
Schema Compiler
Schema Introspection
Schema Diff
Migration Planner
```

---

# 103. Schema object model

Deberá contemplar:

```text
Table
Column
Primary Key
Unique Constraint
Check Constraint
Foreign Key
Index
Trigger metadata when required
View
```

según alcance.

---

# 104. SQLite schemas

No deberá fingirse que SQLite posee exactamente el mismo modelo de schemas que PostgreSQL.

---

# 105. Attached databases

SQLite puede trabajar con múltiples databases adjuntas.

Si VoltStack soporta esa capacidad posteriormente, deberá modelarse explícitamente.

---

# 106. Database namespace

No confundir:

```text
SQLite attached database name
```

con:

```text
PostgreSQL schema
```

aunque ambos puedan aparecer sintácticamente como qualifiers.

---

# 107. Platform-specific namespace semantics

El modelo neutral podrá usar:

```text
QualifiedIdentifier
```

mientras Platform interpreta correctamente cada nivel.

---

# 108. Foreign keys

Foreign key support requiere atención especial.

---

# 109. Foreign key capability

Distinguir:

```text
schema.foreign_key.definition
schema.foreign_key.enforcement
```

---

# 110. Definition ≠ enforcement

Una FK presente en schema no garantiza por sí sola que la sesión actual esté aplicando enforcement.

---

# 111. Foreign key state

El State System deberá poder conocer la configuración relevante.

---

# 112. Secure default

VoltStack deberá favorecer:

```text
foreign key enforcement enabled
```

cuando la aplicación declara FKs.

---

# 113. Session initialization

```text
Connection acquired
      │
      ▼
SqliteSessionInitializer
      │
      ▼
Apply required PRAGMAs
      │
      ▼
Known Baseline
```

---

# 114. PRAGMA architecture

Los PRAGMAs no deberán dispersarse como SQL arbitrario por el framework.

---

# 115. SqlitePragma

Podrá existir una abstracción:

```text
SqlitePragma
```

---

# 116. PRAGMA categories

Distinguir:

```text
connection-scoped
database-file-scoped
transaction-sensitive
read-only
diagnostic
dangerous
```

---

# 117. PRAGMA configuration

Los PRAGMAs soportados por configuración deberán estar tipados.

---

# 118. No arbitrary production PRAGMA bag

Evitar:

```php
'pragmas' => [
    'anything' => 'anything',
]
```

sin validación.

---

# 119. Escape hatch

Podrá existir un mecanismo avanzado para PRAGMAs no modelados, marcado como:

```text
unsafe / platform-specific
```

---

# 120. Important PRAGMA state

La arquitectura deberá considerar, entre otros:

```text
foreign_keys
busy_timeout
journal_mode
synchronous
locking_mode
query_only
cache_size
temp_store
```

según soporte y política.

---

# 121. PRAGMA ownership

No todos los PRAGMAs pertenecen al mismo lifecycle.

---

# 122. Connection baseline

Algunos forman parte del:

```text
ConnectionSessionProfile
```

---

# 123. Database configuration

Otros pertenecen al archivo/base SQLite.

---

# 124. Operation overrides

Otros podrán ser temporales y requerir restauración.

---

# 125. PRAGMA state tracker

Podrá existir:

```text
SqlitePragmaStateTracker
```

---

# 126. Reset requirements

Una conexión reutilizada deberá regresar al baseline conocido.

---

# 127. Unknown PRAGMA mutation

Acceso nativo que modifique PRAGMAs desconocidos podrá marcar:

```text
DIRTY
```

o:

```text
TAINTED
```

el recurso.

---

# 128. Journal mode

Journal mode será una semántica SQLite importante.

---

# 129. JournalMode

Value object/enumeración conceptual:

```text
DELETE
TRUNCATE
PERSIST
MEMORY
WAL
OFF
```

según soporte.

---

# 130. No universal WAL default

VoltStack no deberá asumir:

```text
WAL is always best
```

---

# 131. WAL policy

La elección dependerá de:

```text
deployment
filesystem
concurrency
durability requirements
testing
runtime
```

---

# 132. WAL capability

```text
sqlite.journal.wal
```

---

# 133. WAL configuration

Podrá existir:

```text
SqliteJournalConfiguration
```

---

# 134. WAL state

Journal mode puede afectar al archivo/base, no sólo a una operación individual.

---

# 135. Configuration safety

Cambiar journal mode automáticamente en cada request sería incorrecto.

---

# 136. Initialization strategy

Cambios persistentes de database configuration deberán aplicarse en un lifecycle apropiado:

```text
database initialization
deployment
migration
explicit administration
```

---

# 137. Synchronous mode

Durability/performance settings deberán modelarse explícitamente.

---

# 138. Security/correctness

VoltStack no reducirá silenciosamente garantías de durabilidad para obtener benchmarks mejores.

---

# 139. Locking model

SQLite requiere un modelo de concurrencia diferente a motores cliente-servidor.

---

# 140. Platform responsibility

`SqlitePlatform` deberá describir:

```text
read concurrency
write serialization characteristics
transaction begin modes
busy behavior
journal implications
```

---

# 141. No fake row locks

No declarar capabilities como:

```text
lock.row.for_update
```

si SQLite no proporciona la semántica equivalente.

---

# 142. Pessimistic locking

ORM/Query API deberá recibir:

```text
CapabilityNotSupportedException
```

cuando solicite una semántica de lock no disponible y no exista emulación correcta.

---

# 143. No misleading emulation

No convertir:

```text
SELECT ... FOR UPDATE
```

en una operación diferente y llamarla equivalente si las garantías cambian.

---

# 144. Transaction begin modes

SQLite podrá requerir modelar:

```text
DEFERRED
IMMEDIATE
EXCLUSIVE
```

como estrategias platform-specific.

---

# 145. Transaction intent

Un:

```text
TransactionIntent
```

podrá permitir al Planner/TransactionManager seleccionar una estrategia apropiada.

---

# 146. Portable transaction API

Ejemplo:

```php
DB::transaction(function () {
    // ...
});
```

seguirá siendo portable.

---

# 147. Advanced SQLite transaction

Una API avanzada podrá permitir:

```text
SqliteTransactionMode::IMMEDIATE
```

de forma explícita.

---

# 148. TransactionManager

TransactionManager decide lifecycle.

Platform describe semántica.

Dialect genera sintaxis.

Driver ejecuta primitives.

---

# 149. Savepoints

SQLite deberá exponer:

```text
transaction.savepoint
```

cuando corresponda.

---

# 150. Nested transactions

Podrán implementarse sobre savepoints según el modelo general de VoltStack.

---

# 151. Transactional DDL

No asumir comportamiento idéntico a MySQL/MariaDB/PostgreSQL.

---

# 152. Capability

```text
transaction.transactional_ddl
```

deberá describir semántica efectiva.

---

# 153. Busy handling

SQLite puede devolver:

```text
database is busy
database is locked
```

en situaciones de contención.

---

# 154. Busy timeout

Distinguir:

```text
pool acquire timeout
connect timeout
query timeout
busy timeout
transaction timeout
```

---

# 155. BusyTimeout

Podrá ser un value object:

```text
BusyTimeout
```

---

# 156. Busy timeout ownership

Pertenece a configuración/session/driver-platform integration según mecanismo final.

---

# 157. Busy failure classification

Deberá existir:

```text
DatabaseBusyFailure
```

o una categoría equivalente.

---

# 158. Locked failure

Podrá distinguirse de otros errores de locking cuando sea útil.

---

# 159. Retry

El Platform no implementará retry.

---

# 160. Correct flow

```text
SQLite Failure
     │
     ▼
Failure Classifier
     │
     ▼
Resilience Policy
     │
     ├── retry
     ├── backoff
     └── fail
```

---

# 161. Retry safety

Un retry sólo será permitido cuando:

```text
operation semantics
transaction state
idempotency
failure class
policy
```

lo permitan.

---

# 162. No retry loops

Nunca:

```text
while locked:
    retry forever
```

---

# 163. Backoff

Resilience System podrá utilizar:

```text
bounded retry
backoff
jitter
```

---

# 164. Connection pooling

SQLite requiere especial cuidado con pooling.

---

# 165. Pool ≠ universal benefit

Un pool de múltiples conexiones físicas no siempre mejora SQLite.

---

# 166. Pool policy

Podrá depender de:

```text
file database
memory database
journal mode
runtime
workload
concurrency policy
```

---

# 167. Default conservative policy

La configuración inicial deberá favorecer una política segura y limitada.

---

# 168. File database pool

Para un archivo SQLite:

```text
multiple connections
```

podrán existir, pero deberán respetar la semántica de locking/concurrency.

---

# 169. Memory database pool

Para `:memory:`:

```text
multiple physical connections
```

no deberán habilitarse automáticamente.

---

# 170. Pool compatibility key

Conceptualmente:

```text
DriverId
EndpointIdentity
DatabaseIdentity
ConnectionDefinitionFingerprint
SessionProfileFingerprint
JournalCompatibility
ConfigurationGeneration
```

---

# 171. No credentials requirement

SQLite local puede no usar credenciales tradicionales.

Eso no elimina:

```text
filesystem security
path security
file permissions
encryption extension considerations
```

---

# 172. Physical resource exclusivity

Por defecto:

```text
one active lease
=
one exclusive physical SQLite connection
```

---

# 173. Concurrent sharing

No compartir simultáneamente un mismo native connection entre execution scopes.

---

# 174. Persistent runtimes

Esto es obligatorio para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 175. FrankenPHP

Modelo:

```text
Worker
  │
  ├── Request A
  │     └── SQLite Lease
  │
  ├── RESET
  │
  └── Request B
        └── SQLite Lease
```

---

# 176. State isolation

No deberán filtrarse:

```text
active transaction
temporary table state
temporary PRAGMA override
query_only state
busy configuration override
attached databases
open cursors
```

---

# 177. Attached databases

Una operación:

```text
ATTACH DATABASE
```

modifica estado de conexión.

---

# 178. State tracking

Si se soporta ATTACH/DETACH:

```text
AttachedDatabaseState
```

deberá rastrearse.

---

# 179. Pool safety

Una conexión con attachments inesperados no deberá volver al pool como limpia.

---

# 180. Temporary tables

También forman parte de estado relevante.

---

# 181. Temp state

Podrá requerir:

```text
cleanup
reset
discard
```

según la estrategia.

---

# 182. Cursor lifecycle

Un cursor/result streaming abierto mantiene ownership del recurso.

---

# 183. StreamingResult

```text
StreamingResult
      │
      ▼
ConnectionLease
```

hasta cierre o agotamiento.

---

# 184. Buffered results

Podrán liberar el lease antes cuando sea seguro.

---

# 185. Query cancellation

Sólo deberá anunciarse si Driver + SQLite integration proporcionan una semántica utilizable.

---

# 186. Capability composition

```text
Platform supports semantic feature
+
Driver exposes required primitive
=
effective capability
```

---

# 187. CTE

SQLite deberá soportar capability:

```text
query.cte
```

según versión.

---

# 188. Recursive CTE

```text
query.cte.recursive
```

---

# 189. Window functions

Capabilities:

```text
query.window
query.window.partition
query.window.order
query.window.frame
```

según versión.

---

# 190. UPSERT

SQLite deberá modelar UPSERT mediante el Query AST común.

---

# 191. UpsertQuery

```text
UpsertQuery
├── target
├── values
├── conflict target
├── action
└── returning
```

---

# 192. SQLite UPSERT compiler

```text
UpsertQuery
     │
     ▼
Planner
     │
     ▼
SQLite Upsert Strategy
     │
     ▼
SqliteDialect
```

---

# 193. No replace-as-upsert assumption

No tratar automáticamente:

```text
INSERT OR REPLACE
```

como equivalente semántico universal de UPSERT.

---

# 194. Replacement semantics

Las diferencias de delete/insert/constraint/trigger/identity deberán respetarse.

---

# 195. Conflict actions

Cuando estén soportadas:

```text
DO NOTHING
DO UPDATE
```

---

# 196. RETURNING

SQLite moderno puede proporcionar:

```text
RETURNING
```

según versión.

---

# 197. Capability granularity

```text
query.returning.insert
query.returning.update
query.returning.delete
```

---

# 198. ORM integration

Persistence Planner podrá utilizar RETURNING para recuperar:

```text
generated identifiers
generated columns
defaults
```

cuando la capability esté disponible.

---

# 199. Fallback strategy

Si RETURNING no está disponible:

```text
generated key API
follow-up query
other safe strategy
```

según semántica.

---

# 200. RETURNING ≠ generated key API

Mantener ambos conceptos separados.

---

# 201. LIMIT/OFFSET

Query Compiler deberá producir sintaxis SQLite apropiada sin que Query Builder la conozca.

---

# 202. Compound queries

Capabilities deberán describir:

```text
UNION
UNION ALL
INTERSECT
EXCEPT
```

según soporte.

---

# 203. Query planner limitations

El VoltStack Planner podrá considerar limitaciones SQLite cuando seleccione estrategias, sin intentar sustituir al query planner interno de SQLite.

---

# 204. Framework optimizer

Puede optimizar:

```text
duplicate queries
redundant predicates
N+1
batch sizing
portable rewrites
```

---

# 205. SQLite optimizer remains authoritative

VoltStack no intentará reemplazar el optimizador interno de SQLite.

---

# 206. Indexes

SQLite deberá soportar:

```text
normal indexes
unique indexes
partial indexes
expression indexes
```

según versión/capabilities.

---

# 207. Partial index capability

```text
schema.index.partial
```

---

# 208. Expression index capability

```text
schema.index.expression
```

---

# 209. IndexDefinition

Conceptualmente:

```text
IndexDefinition
├── name
├── columns
├── expressions
├── unique
└── predicate
```

---

# 210. Index expressions

Deberán representarse mediante Expression AST cuando sea posible.

---

# 211. Partial predicates

Igualmente:

```text
Predicate AST
```

---

# 212. Schema introspection

Componente:

```text
SqliteSchemaIntrospector
```

---

# 213. Introspection flow

```text
SQLite Database
      │
      ▼
SqliteSchemaIntrospector
      │
      ▼
Native Schema Metadata
      │
      ▼
SqliteSchemaCanonicalizer
      │
      ▼
Canonical Schema Model
```

---

# 214. Introspection responsibilities

Deberá descubrir:

```text
tables
columns
declared types
type affinity
nullability
defaults
primary keys
foreign keys
indexes
partial indexes
expression indexes
generated columns
STRICT metadata
WITHOUT ROWID metadata
views
```

según capabilities.

---

# 215. Introspection APIs

El uso de PRAGMAs/catalog metadata deberá quedar encapsulado dentro del adapter SQLite.

---

# 216. No PRAGMA leakage

Schema System genérico no deberá contener:

```text
PRAGMA table_info
```

ni otros detalles SQLite.

---

# 217. Canonicalization

`SqliteSchemaCanonicalizer` deberá normalizar diferencias sintácticas.

---

# 218. Round-trip invariant

```text
Schema Model
    │
    ▼
CREATE
    │
    ▼
SQLite
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

# 219. SQLite migration architecture

Este es uno de los aspectos más importantes de la plataforma.

---

# 220. ALTER TABLE limitations

Migration Planner no deberá asumir que toda transformación de schema puede expresarse mediante:

```text
ALTER TABLE ...
```

directo.

---

# 221. Migration strategy selection

```text
Schema Diff
    │
    ▼
Migration Planner
    │
    ├── Native SQLite ALTER
    ├── Direct DDL
    └── Table Rebuild Strategy
```

---

# 222. Table rebuild

Deberá existir una estrategia formal:

```text
SqliteTableRebuildStrategy
```

---

# 223. Rebuild pipeline

Conceptualmente:

```text
Original Table
      │
      ▼
Create Replacement Table
      │
      ▼
Copy / Transform Data
      │
      ▼
Recreate Indexes
      │
      ▼
Recreate Constraints
      │
      ▼
Recreate Relevant Objects
      │
      ▼
Swap / Rename
      │
      ▼
Validate
```

---

# 224. Rebuild is not simple copy

Debe preservar correctamente:

```text
columns
defaults
constraints
indexes
foreign keys
generated columns
STRICT
WITHOUT ROWID
platform metadata
```

---

# 225. Rebuild data mapping

Migration Planner deberá producir:

```text
ColumnMigrationMap
```

---

# 226. ColumnMigrationMap

Conceptualmente:

```text
old column
→
new column / expression / default / omitted
```

---

# 227. Data transformation

Cambios incompatibles podrán requerir:

```text
MigrationExpression
```

explícita.

---

# 228. No silent destructive conversion

VoltStack no deberá convertir datos silenciosamente si existe riesgo de pérdida.

---

# 229. Destructive migration classification

Operaciones podrán clasificarse:

```text
SAFE
POTENTIALLY_DESTRUCTIVE
DESTRUCTIVE
UNSUPPORTED
```

---

# 230. Migration safety

El usuario podrá exigir:

```text
safe migrations only
```

en producción.

---

# 231. Foreign keys during rebuild

El Migration Planner deberá manejar correctamente la interacción entre:

```text
foreign key enforcement
table replacement
data copy
transaction
```

---

# 232. No arbitrary FK disable

VoltStack no deberá deshabilitar foreign keys de manera global y olvidarse de restaurarlas.

---

# 233. State restoration

Cualquier cambio temporal de configuración deberá ser:

```text
tracked
scoped
restored
verified
```

---

# 234. Rebuild failure

Si una reconstrucción falla:

```text
transaction rollback
```

deberá utilizarse cuando la semántica efectiva lo permita.

---

# 235. Recovery

Migration System deberá evitar dejar:

```text
temporary replacement tables
partial indexes
half-migrated state
```

cuando sea razonablemente posible.

---

# 236. Migration plan

Una reconstrucción deberá aparecer explícitamente en:

```text
MigrationPlan
```

---

# 237. Explain migration

CLI podrá mostrar:

```text
ALTER users:
  strategy: TABLE_REBUILD
  risk: POTENTIALLY_DESTRUCTIVE
  copy rows: yes
  recreate indexes: yes
  recreate foreign keys: yes
```

---

# 238. No hidden expensive migrations

VoltStack no deberá ocultar que una operación aparentemente pequeña requiere reconstruir una tabla completa.

---

# 239. Migration telemetry

Podrá registrar:

```text
strategy
duration
rows copied
table rebuild
failure class
```

sin exponer datos sensibles.

---

# 240. Views

SQLite views deberán formar parte del Schema Model cuando estén soportadas.

---

# 241. View capability

```text
schema.view
```

---

# 242. Triggers

SQLite utiliza triggers en diversos escenarios.

El Schema System podrá preservar metadata de triggers aunque la API pública completa se implemente posteriormente.

---

# 243. Trigger ownership

Migration rebuild deberá considerar triggers asociados cuando sea necesario.

---

# 244. No trigger loss

Una table rebuild no deberá eliminar silenciosamente triggers conocidos por el Schema Model.

---

# 245. Unknown schema objects

Si VoltStack detecta objetos que no sabe preservar durante una reconstrucción, deberá:

```text
warn
reject
require explicit policy
```

en vez de destruirlos silenciosamente.

---

# 246. Schema fidelity

Este principio será especialmente importante para bases SQLite legacy.

---

# 247. Legacy databases

SQLite Platform deberá soportar introspección de bases existentes creadas fuera de VoltStack.

---

# 248. Unknown declared types

SQLite permite nombres de tipos diversos.

VoltStack deberá preservarlos aunque no exista mapping ORM directo.

---

# 249. UnknownSqliteType

Podrá existir:

```text
UnknownSqliteTypeMetadata
```

---

# 250. ORM strictness

ORM podrá rechazar la hidratación automática de un tipo desconocido hasta recibir mapping explícito.

---

# 251. Schema introspection must remain tolerant

Schema inspection no deberá fallar sólo porque exista un tipo custom.

---

# 252. Error architecture

SQLite deberá integrarse con el sistema neutral de errores.

---

# 253. Error flow

```text
PDOException
    │
    ▼
PdoSqliteErrorAdapter
    │
    ▼
NativeDatabaseError
    │
    ▼
SqliteFailureClassifier
    │
    ▼
DatabaseFailure
```

---

# 254. Failure categories

Podrán incluir:

```text
UniqueConstraintFailure
ForeignKeyConstraintFailure
NotNullConstraintFailure
CheckConstraintFailure
DatabaseBusyFailure
DatabaseLockedFailure
ReadOnlyFailure
CorruptDatabaseFailure
DiskFullFailure
IoFailure
PermissionFailure
SyntaxFailure
MissingTableFailure
MissingColumnFailure
```

según la información disponible.

---

# 255. Corruption

Errores que indiquen posible corrupción deberán tratarse como:

```text
high-severity database failure
```

---

# 256. No automatic retry on corruption

Nunca reintentar indiscriminadamente errores de corrupción.

---

# 257. Disk full

Un:

```text
DiskFullFailure
```

no deberá confundirse con un error temporal de lock.

---

# 258. I/O failures

Deberán clasificarse separadamente cuando sea posible.

---

# 259. Native error codes

La clasificación deberá utilizar códigos nativos/extendidos cuando el Driver los exponga.

---

# 260. No regex-first classification

Los mensajes de texto serán fallback, no arquitectura principal.

---

# 261. Security

SQLite es local en muchos deployments, pero sigue requiriendo seguridad.

---

# 262. Security concerns

Incluyen:

```text
file permissions
directory permissions
path traversal
symlink behavior
database replacement
backup exposure
temporary files
SQL injection
identifier injection
sensitive data
```

---

# 263. Path traversal

La configuración dinámica de rutas deberá estar controlada.

---

# 264. Tenant database files

Si Multitenancy utiliza:

```text
database-per-tenant
```

con SQLite, las rutas deberán provenir de resolución segura.

---

# 265. No raw tenant path

Incorrecto:

```php
$path = '/db/' . $tenantInput . '.sqlite';
```

sin validación.

---

# 266. Correct approach

```text
Tenant Identity
      │
      ▼
Tenant Database Resolver
      │
      ▼
Validated DatabaseFilePath
      │
      ▼
ConnectionDefinition
```

---

# 267. Multitenancy optional

`SqlitePlatform` no dependerá de:

```text
Quantum/Multitenancy
```

---

# 268. Encryption extensions

SQLite encryption no deberá asumirse como capability core.

---

# 269. Extension-specific encryption

Podrá añadirse posteriormente mediante:

```text
Driver Extension
Platform Extension
Capability Provider
```

---

# 270. Encryption capability

Ejemplo:

```text
storage.encryption.native
```

sólo cuando una integración concreta la proporcione.

---

# 271. Backup

SQLite permite estrategias de backup diferentes a motores servidor.

Sin embargo Backup System será un subsistema separado.

---

# 272. No naive file copy assumption

Copiar el archivo SQLite mientras está activo no deberá considerarse automáticamente un backup consistente.

---

# 273. Backup capability

Platform podrá describir mecanismos disponibles.

---

# 274. Backup System

Será quien seleccione:

```text
native backup API
safe snapshot
offline copy
other strategy
```

---

# 275. Testing

SQLite será especialmente útil para tests, pero VoltStack no deberá presentar SQLite como sustituto semántico perfecto de PostgreSQL/MySQL.

---

# 276. Testing principle

```text
SQLite test
≠
guaranteed production database compatibility
```

---

# 277. Test profiles

Podrán existir:

```text
SQLITE_MEMORY
SQLITE_FILE
PRODUCTION_PLATFORM
MULTI_PLATFORM_CONFORMANCE
```

---

# 278. Fast unit/integration tests

SQLite memory podrá proporcionar:

```text
fast isolated database tests
```

cuando la semántica requerida sea compatible.

---

# 279. Production parity tests

Aplicaciones que dependan de features específicas deberán probar también el motor real.

---

# 280. Test database lifecycle

Para memory database:

```text
Test Scope
    │
    ▼
Pinned Connection
    │
    ▼
Schema
    │
    ▼
Test
    │
    ▼
Dispose Connection
    │
    ▼
Database disappears
```

---

# 281. Test connection pinning

Será importante evitar que:

```text
schema setup
```

ocurra en una conexión y:

```text
test query
```

en otra `:memory:` distinta.

---

# 282. Test DatabaseContext

Podrá pinnear la conexión física durante todo el test cuando use memory database.

---

# 283. Parallel testing

Cada test/worker deberá obtener aislamiento explícito.

---

# 284. Parallel strategies

Podrán incluir:

```text
one memory database per test
one file per worker
one file per test
shared database with transaction isolation where safe
```

---

# 285. No shared mutable test DB by default

Evitar contaminación entre tests.

---

# 286. Factories and seeders

Funcionarán sobre la API portable.

---

# 287. Database assertions

Ejemplo:

```php
$this->assertDatabaseHas('users', [
    'email' => 'user@example.com',
]);
```

deberá funcionar con SQLite sin introducir lógica SQLite en Testing API.

---

# 288. Capability-aware tests

VoltStack Testing podrá permitir:

```text
requires capability
```

para omitir/fallar tests de forma explícita según plataforma.

---

# 289. SQLite as development database

Podrá utilizarse en desarrollo, pero la aplicación deberá recibir warnings cuando use schema/query semantics que divergen del target de producción declarado.

---

# 290. Portability diagnostics

Podrá existir:

```text
DatabasePortabilityAnalyzer
```

---

# 291. Example warning

```text
SQLite accepted this schema definition,
but production PostgreSQL enforces different type semantics.
```

---

# 292. Capability matrix

La documentación podrá generar una matriz:

```text
Feature                 SQLite
--------------------------------
CTE                     native
Recursive CTE           native/version-aware
Window Functions        version-aware
UPSERT                   version-aware
RETURNING                version-aware
Partial Indexes          version-aware
Expression Indexes       version-aware
STRICT Tables            version-aware
Row-level FOR UPDATE     unsupported
```

Los valores exactos deberán provenir del Capability System, no de tablas hardcodeadas en capas superiores.

---

# 293. Version-aware capabilities

Nunca:

```php
if ($sqliteVersion >= 'x.y.z')
```

distribuido por todo el código.

---

# 294. Centralized version rules

Preferir:

```text
SqliteCapabilityRules
```

---

# 295. Capability rule example

Conceptualmente:

```php
$rules->feature('query.returning.insert')
    ->nativeWhenVersionSatisfies(...);
```

---

# 296. No magic version checks

Los requisitos concretos de versión deberán vivir en un único lugar probado.

---

# 297. Capability provenance

Diagnostics podrá responder:

```text
query.returning.insert
status: NATIVE
source: SqlitePlatformCapabilityProvider
reason: SQLite engine version supports feature
```

---

# 298. Capability fingerprint

Podrá incluir:

```text
SQLite version
compile options
relevant driver capabilities
relevant configuration
```

---

# 299. Query cache compatibility

Compiled Query Cache deberá considerar:

```text
QueryFingerprint
DialectFingerprint
CapabilityFingerprint
CompilerVersion
```

---

# 300. Schema cache compatibility

Schema metadata cache deberá tener fingerprint/invalidation independiente.

---

# 301. Connection state fingerprint

PRAGMA/session state no deberá confundirse con Platform capability fingerprint.

---

# 302. Separate fingerprints

Mantener:

```text
CapabilityFingerprint
SchemaFingerprint
SessionProfileFingerprint
ConnectionDefinitionFingerprint
```

---

# 303. Query-only connections

SQLite podrá soportar una configuración semántica:

```text
READ_ONLY
```

---

# 304. Read-only enforcement

Podrá provenir de:

```text
file mode
URI mode
PRAGMA/query state
filesystem
driver capability
```

según estrategia.

---

# 305. Connection role

Connection System seguirá exponiendo:

```text
READ_ONLY
READ_WRITE
```

sin acoplarse a detalles SQLite.

---

# 306. Read/write topology

SQLite Platform no implementará routing.

---

# 307. Topology limitation

Un archivo SQLite local normalmente tendrá topología diferente a un cluster PostgreSQL/MySQL.

Eso no justifica contaminar Platform con TopologyManager.

---

# 308. Replication extensions

Si una solución externa ofrece replicación SQLite, deberá integrarse mediante:

```text
Topology Extension
Driver/Platform Capability Extension
```

según corresponda.

---

# 309. Bulk insert

SQLite deberá soportar planificación de inserts por lotes.

---

# 310. Batch limits

Planner deberá considerar:

```text
maximum bind parameters
statement size
transaction policy
SQLite version/build limits
```

---

# 311. No hardcoded batch size

Evitar:

```text
batch = 1000
```

universal.

---

# 312. Parameter capability

Podrá existir:

```text
parameter.max_count
```

---

# 313. Bulk planner

```text
Rows
 │
 ▼
Bulk Planner
 │
 ├── capability limits
 ├── transaction policy
 └── resource policy
       │
       ▼
   Batch Plans
```

---

# 314. Bulk transaction

Cuando sea apropiado, múltiples inserts podrán agruparse en una transacción para rendimiento.

---

# 315. Durability policy

VoltStack no cambiará silenciosamente:

```text
synchronous
journal mode
```

para acelerar bulk inserts.

---

# 316. Performance profiles

Optimizaciones agresivas deberán ser:

```text
explicit
documented
policy-driven
```

---

# 317. Vacuum

Operaciones como:

```text
VACUUM
```

pertenecerán a:

```text
Database Administration / Maintenance
```

no al Query Builder normal.

---

# 318. Analyze

Igualmente:

```text
ANALYZE
```

será operación administrativa/diagnóstica.

---

# 319. Maintenance capabilities

Podrán exponerse:

```text
maintenance.vacuum
maintenance.analyze
maintenance.optimize
```

según versión.

---

# 320. Migration vs maintenance

Migration System no deberá ejecutar maintenance automáticamente salvo política explícita.

---

# 321. Database size

Diagnostics podrá obtener:

```text
file size
page count
page size
```

cuando sea apropiado.

---

# 322. Sensitive paths

Las métricas no deberán usar la ruta completa del archivo como label.

---

# 323. Telemetry

Podrá incluir:

```text
platform = sqlite
driver = pdo.sqlite
database kind = file/memory
operation
duration
rows
failure class
busy events
```

---

# 324. High cardinality

No usar:

```text
absolute file path
tenant-specific filename
raw SQL
```

como metric labels.

---

# 325. Busy telemetry

Será útil medir:

```text
busy failures
lock wait duration
retry count
```

desde Resilience/Execution.

---

# 326. Pool telemetry

Podrá medir:

```text
active leases
idle resources
acquire duration
pool exhaustion
```

cuando pooling esté habilitado.

---

# 327. WAL telemetry

Podrá exponerse en diagnostics, pero no necesariamente como label de cada query.

---

# 328. Runtime diagnostics

Ejemplo:

```text
Connection: default
Driver: pdo.sqlite
Platform: sqlite
Engine Version: <resolved>
Database Kind: file
Journal Mode: WAL
Foreign Keys: enabled
Strict Capability: supported
Capability Profile: <fingerprint>
```

---

# 329. CLI

Podrá existir:

```text
php volt database:platform default
```

---

# 330. Capability inspection

```text
php volt database:capability --connection=default
```

---

# 331. SQLite state diagnostics

Podrá existir:

```text
php volt database:sqlite:state default
```

para diagnóstico avanzado.

---

# 332. State output

Podrá mostrar de forma segura:

```text
journal mode
foreign key enforcement
busy timeout
query-only state
attached database count
transaction state
```

sin datos sensibles.

---

# 333. Schema inspect

```text
php volt database:schema:inspect
```

---

# 334. Migration plan

```text
php volt database:migration:plan
```

deberá indicar claramente cuando una operación requiere:

```text
TABLE REBUILD
```

---

# 335. Explain rebuild

Ejemplo:

```text
users.email nullable → not-null

SQLite Strategy:
TABLE_REBUILD

Steps:
1. create replacement table
2. copy data
3. recreate indexes
4. recreate foreign keys
5. validate
6. swap tables

Risk:
POTENTIALLY_DESTRUCTIVE
```

---

# 336. State/reset architecture

SQLite tendrá integración con:

```text
16_DATABASE_CONNECTION_STATE_AND_RESET_SYSTEM.md
```

---

# 337. State categories

Deberán rastrearse al menos cuando corresponda:

```text
transaction
PRAGMA overrides
query-only
attached databases
temporary objects
open cursors
native access
```

---

# 338. SqliteResetPlanner

Componente:

```text
SqliteResetPlanner
```

---

# 339. Reset pipeline

```text
Connection State
      │
      ▼
SqliteResetPlanner
      │
      ▼
Reset Plan
      │
      ├── rollback transaction
      ├── close cursors
      ├── restore PRAGMAs
      ├── detach databases
      ├── clear temp state
      └── verify baseline
             │
             ▼
       REUSE / DISCARD
```

---

# 340. Reset uncertainty

Si no puede garantizarse el baseline:

```text
DISCARD
```

---

# 341. No unsafe reuse

Correctness tendrá prioridad sobre reutilización.

---

# 342. Pool interaction

```text
ConnectionLease
      │
      ▼
Release
      │
      ▼
State Inspection
      │
      ▼
SqliteResetPlanner
      │
      ├── clean → IDLE
      └── unsafe → DISCARD
```

---

# 343. Memory database discard

Descartar la única conexión de una base privada `:memory:` puede destruir la base.

---

# 344. Explicit memory ownership

Por ello una memory database deberá declarar claramente:

```text
scope
owner
connection lifetime
```

---

# 345. Memory lifetime models

Podrán existir:

```text
OPERATION
EXECUTION_SCOPE
TEST
WORKER
APPLICATION
```

sólo cuando la semántica/configuración lo permitan.

---

# 346. Default memory test lifetime

Para testing será común:

```text
TEST
```

---

# 347. Request memory database

Una base memory request-scoped podrá destruirse al finalizar la request.

---

# 348. Application memory database

Una base memory application-scoped requerirá estrategia de conexión persistente/pinneada explícita.

---

# 349. No accidental lifetime extension

No permitir que un cambio de pooling cambie accidentalmente la vida de una base en memoria.

---

# 350. Connection lifecycle integration

```text
Configured
    │
    ▼
Logical Connection
    │
    ▼
Acquire
    │
    ▼
Native SQLite Connection
    │
    ▼
Initialize Baseline
    │
    ▼
Use
    │
    ▼
Reset
    │
    ├── Release
    └── Discard
```

---

# 351. OpenSwoole

En un runtime coroutine-based:

```text
ExecutionScope A
ExecutionScope B
```

no deberán compartir simultáneamente el mismo native SQLite connection.

---

# 352. Concurrency guard

Pool/Resource Provider deberá aplicar exclusividad.

---

# 353. Async future drivers

Si en el futuro existe un Driver SQLite async especializado, deberá declarar capabilities explícitas.

---

# 354. No async assumption

La Platform no asumirá:

```text
async
```

por sí sola.

---

# 355. Platform-specific query features

SQLite-specific features podrán exponerse mediante extensiones.

---

# 356. Extension rule

Las extensiones no modificarán el AST central arbitrariamente.

Deberán registrar:

```text
semantic node
capability requirement
planner strategy
compiler strategy
```

cuando sea necesario.

---

# 357. Extension types

Ejemplos:

```text
custom SQLite function
virtual table integration
FTS integration
JSON extension
spatial extension
encryption integration
```

---

# 358. Virtual tables

SQLite virtual tables serán una capability avanzada.

---

# 359. VirtualTable capability

```text
schema.virtual_table
```

o namespace específico de extensión.

---

# 360. Core independence

El núcleo Database no dependerá de ninguna extensión concreta.

---

# 361. Full-text search

FTS deberá implementarse como integración/capability especializada.

---

# 362. No universal full-text abstraction assumption

No asumir que SQLite FTS, PostgreSQL FTS y MySQL FULLTEXT tienen semántica idéntica.

---

# 363. Portable search layer

Una futura API de búsqueda podrá tener:

```text
Semantic Search Model
        │
        ▼
Platform Strategy
```

---

# 364. Extension discovery

Compile options/server extensions disponibles podrán detectarse de manera centralizada.

---

# 365. No repeated discovery

No ejecutar discovery por query.

---

# 366. Discovery cache

Podrá utilizar:

```text
EngineFingerprint
```

---

# 367. EngineFingerprint

Conceptualmente:

```text
PlatformId
EngineVersion
CompileOptionsFingerprint
DriverCapabilityFingerprint
```

---

# 368. Config fingerprint separation

No mezclar con:

```text
ConnectionDefinitionFingerprint
```

---

# 369. Schema fidelity policy

Podrá configurarse:

```text
STRICT
WARN
BEST_EFFORT
```

para introspección/migraciones legacy.

---

# 370. STRICT fidelity

Si existe un objeto no preservable:

```text
migration rejected
```

---

# 371. WARN fidelity

Migration Plan podrá requerir confirmación explícita.

---

# 372. BEST_EFFORT

Sólo deberá utilizarse conscientemente.

---

# 373. Production default

Para migraciones destructivas/rebuild:

```text
STRICT
```

o equivalente seguro será preferible.

---

# 374. Zero-downtime implications

SQLite no deberá anunciar capacidades de zero-downtime migration equivalentes a motores distribuidos/cliente-servidor si no puede ofrecer las mismas garantías.

---

# 375. Migration capability honesty

Capability System deberá representar limitaciones reales.

---

# 376. Database portability

SQLite tendrá un rol importante en:

```text
327_DATABASE_DATABASE_PORTABILITY_SYSTEM.md
```

---

# 377. Portability levels

VoltStack podrá distinguir:

```text
PORTABLE
PORTABLE_WITH_DIFFERENCES
PLATFORM_AWARE
PLATFORM_SPECIFIC
```

---

# 378. Type portability

Ejemplo:

```text
VARCHAR(255)
```

puede compilar en SQLite, pero no necesariamente conservar idéntica enforcement semantics.

---

# 379. Semantic portability warning

El sistema deberá poder advertir sobre esas diferencias.

---

# 380. ORM behavior

ORM deberá trabajar sobre tipos semánticos VoltStack.

---

# 381. Correct ORM flow

```text
Entity
   │
   ▼
Metadata
   │
   ▼
UnitOfWork
   │
   ▼
Persistence Planner
   │
   ▼
Query Model
   │
   ▼
Query Engine
   │
   ▼
SQLite Strategy
```

---

# 382. ORM never generates SQLite SQL

Regla absoluta:

```text
ORM
≠
SQLite SQL Generator
```

---

# 383. Active Record

API Active Record seguirá siendo:

```text
Convenience API
      │
      ▼
EntityManager
      │
      ▼
same ORM Engine
```

---

# 384. Model does not know SQLite

Incorrecto:

```php
class User
{
    protected string $sqliteTable = 'users';
}
```

salvo metadata platform-specific explícita para un caso avanzado.

---

# 385. Schema API

Una definición portable:

```php
Schema::create('users', function ($table) {
    $table->id();
    $table->string('name');
});
```

deberá producir:

```text
Schema Model
```

antes de producir SQL.

---

# 386. Schema Planner

Será quien determine:

```text
SQLite-specific create strategy
```

---

# 387. Migration modifications

Una llamada conceptual:

```php
$table->dropColumn('legacy');
```

no significa necesariamente:

```text
ALTER TABLE DROP COLUMN
```

---

# 388. Semantic operation

Significa:

```text
DropColumnOperation
```

---

# 389. Planner decision

```text
DropColumnOperation
      │
      ▼
SQLite CapabilitySnapshot
      │
      ├── native supported → native strategy
      └── otherwise → rebuild strategy
```

---

# 390. Future-proofing

Esto permite aprovechar capacidades nuevas de SQLite sin cambiar la API pública.

---

# 391. Capability-driven evolution

```text
same Schema Operation
+
newer SQLite version
=
potentially better execution strategy
```

---

# 392. Migration reproducibility

La Migration Plan deberá registrar suficiente información para entender la estrategia seleccionada.

---

# 393. Platform upgrade

Actualizar SQLite puede cambiar:

```text
available capabilities
migration strategies
query compilation strategies
```

---

# 394. Cache invalidation

Un cambio de EngineVersion deberá invalidar:

```text
CapabilitySnapshot
relevant CompiledQuery cache
relevant migration plan cache
```

---

# 395. No unnecessary invalidation

Schema metadata y SessionProfile tendrán fingerprints separados.

---

# 396. Health checks

SQLite health checks podrán incluir:

```text
file accessible
database openable
engine compatible
schema readable
required capabilities available
```

---

# 397. Integrity checks

Operaciones profundas de integridad deberán pertenecer a Administration/Diagnostics, no al health check rápido normal.

---

# 398. Fast health

No ejecutar operaciones costosas en cada readiness probe.

---

# 399. Readiness

Podrá verificar:

```text
connection can be opened
required schema accessible
required capabilities available
```

según política.

---

# 400. Liveness

No deberá depender necesariamente de abrir SQLite repetidamente.

---

# 401. Administration

Podrá incluir en el futuro:

```text
vacuum
analyze
integrity check
optimize
backup
journal diagnostics
```

---

# 402. Administration separation

```text
DatabaseAdministration
≠
QueryBuilder
```

---

# 403. Namespace propuesto

```text
VoltStack\Quantum\Database\Platform\Sqlite
```

---

# 404. Estructura propuesta

```text
Platform/
└── Sqlite/
    ├── SqlitePlatform.php
    ├── SqlitePlatformDescriptor.php
    ├── SqlitePlatformSupportPolicy.php
    │
    ├── Discovery/
    │   ├── SqliteEngineDiscovery.php
    │   ├── SqliteVersion.php
    │   ├── SqliteVersionParser.php
    │   └── SqliteCompileOptionDiscovery.php
    │
    ├── Capability/
    │   ├── SqliteCapabilityProvider.php
    │   ├── SqliteCapabilityRules.php
    │   └── SqliteCapabilitySnapshotFactory.php
    │
    ├── Type/
    │   ├── SqliteTypeAffinity.php
    │   ├── SqliteTypeMapping.php
    │   ├── SqliteTypeCanonicalizer.php
    │   └── SqliteTypeMetadata.php
    │
    ├── Schema/
    │   ├── SqliteSchemaSemantics.php
    │   ├── SqliteSchemaIntrospector.php
    │   ├── SqliteSchemaCanonicalizer.php
    │   └── SqliteSchemaMetadata.php
    │
    ├── Migration/
    │   ├── SqliteMigrationStrategyResolver.php
    │   ├── SqliteTableRebuildStrategy.php
    │   ├── SqliteColumnMigrationMap.php
    │   └── SqliteMigrationSafetyAnalyzer.php
    │
    ├── Transaction/
    │   ├── SqliteTransactionSemantics.php
    │   └── SqliteTransactionMode.php
    │
    ├── Session/
    │   ├── SqliteSessionInitializer.php
    │   ├── SqlitePragmaStateTracker.php
    │   └── SqliteResetPlanner.php
    │
    ├── Journal/
    │   ├── SqliteJournalMode.php
    │   └── SqliteJournalConfiguration.php
    │
    ├── Error/
    │   └── SqliteFailureClassifier.php
    │
    ├── Testing/
    └── Extension/
```

---

# 405. Dialect namespace

```text
VoltStack\Quantum\Database\Dialect\Sqlite
```

---

# 406. Driver namespace

```text
VoltStack\Quantum\Database\Driver\Pdo\Sqlite
```

---

# 407. Separation

```text
Driver/Pdo/Sqlite
     │
     └── native communication

Dialect/Sqlite
     │
     └── SQL syntax

Platform/Sqlite
     │
     └── SQLite semantics
```

---

# 408. Dependency restrictions

`SqlitePlatform` no dependerá de:

```text
ORM\EntityManager
ORM\UnitOfWork
Repository
Controller
HTTP
Application
Multitenancy
Telemetry implementation
Cache implementation
```

---

# 409. Architecture tests

Deberán verificar estas fronteras automáticamente.

---

# 410. Vendor checks

Capas superiores no deberán contener:

```php
if ($platform->name() === 'sqlite') {
}
```

para seleccionar comportamiento portable.

---

# 411. Correct approach

```text
Semantic Requirement
      │
      ▼
Capability Requirement
      │
      ▼
Planner Strategy Resolver
```

---

# 412. Explicit SQLite extension

Los checks SQLite sólo serán aceptables dentro de:

```text
SQLite-specific adapters
SQLite-specific extensions
SQLite-specific diagnostics
```

---

# 413. Architectural invariants

## DB-SQLITE-001

SQLite será una Platform de primera clase.

## DB-SQLITE-002

El Driver inicial será `pdo.sqlite`.

## DB-SQLITE-003

Driver, Dialect y Platform permanecerán separados.

## DB-SQLITE-004

SqlitePlatform será preferentemente inmutable y stateless.

## DB-SQLITE-005

SqlitePlatform no almacenará current Connection.

## DB-SQLITE-006

SqlitePlatform no almacenará current Transaction.

## DB-SQLITE-007

SqlitePlatform no almacenará current Tenant.

## DB-SQLITE-008

La versión SQLite será descubierta explícitamente.

## DB-SQLITE-009

Capabilities serán version-aware.

## DB-SQLITE-010

Capabilities podrán considerar compile options.

## DB-SQLITE-011

`:memory:` se modelará semánticamente, no sólo como string.

## DB-SQLITE-012

Una base privada `:memory:` estará asociada a su conexión física.

## DB-SQLITE-013

Pooling no cambiará silenciosamente la semántica de una base en memoria.

## DB-SQLITE-014

File paths serán normalizados antes del runtime.

## DB-SQLITE-015

Type affinity será modelada por Platform.

## DB-SQLITE-016

Declared type y affinity serán conceptos distintos.

## DB-SQLITE-017

El ORM utilizará tipos VoltStack, no affinity directamente.

## DB-SQLITE-018

Boolean no será expuesto al dominio simplemente como integer.

## DB-SQLITE-019

Date/time tendrá estrategia explícita.

## DB-SQLITE-020

Decimal conversion evitará pérdida silenciosa de precisión.

## DB-SQLITE-021

JSON storage y JSON capability serán conceptos distintos.

## DB-SQLITE-022

STRICT será capability-driven.

## DB-SQLITE-023

rowid no será asumido universalmente.

## DB-SQLITE-024

INTEGER PRIMARY KEY y AUTOINCREMENT serán semánticamente distintos.

## DB-SQLITE-025

AUTOINCREMENT no será añadido automáticamente sin necesidad.

## DB-SQLITE-026

WITHOUT ROWID será representable.

## DB-SQLITE-027

Foreign-key definition y enforcement serán conceptos distintos.

## DB-SQLITE-028

Foreign-key state deberá inicializarse/verificarse según policy.

## DB-SQLITE-029

PRAGMA state será tratado como estado de conexión/base según su semántica.

## DB-SQLITE-030

PRAGMAs no se dispersarán como SQL arbitrario.

## DB-SQLITE-031

Cambios temporales de PRAGMA deberán restaurarse.

## DB-SQLITE-032

Journal mode no cambiará por request.

## DB-SQLITE-033

WAL no será habilitado universalmente.

## DB-SQLITE-034

Durability settings no se reducirán silenciosamente.

## DB-SQLITE-035

SQLite locking semantics no se fingirán como row locks.

## DB-SQLITE-036

FOR UPDATE no será emulado con garantías diferentes.

## DB-SQLITE-037

Transaction begin modes podrán modelarse explícitamente.

## DB-SQLITE-038

Savepoints serán capability-driven.

## DB-SQLITE-039

Busy timeout será distinto de query/acquire/connect timeout.

## DB-SQLITE-040

Platform no ejecutará retry.

## DB-SQLITE-041

Retries serán bounded y policy-driven.

## DB-SQLITE-042

Pooling será conservador y workload-aware.

## DB-SQLITE-043

Una physical connection tendrá un único lease activo por defecto.

## DB-SQLITE-044

Execution scopes no compartirán simultáneamente native connection.

## DB-SQLITE-045

ATTACH/DETACH será stateful.

## DB-SQLITE-046

Temporary objects participarán en reset.

## DB-SQLITE-047

Open cursors impedirán liberación insegura.

## DB-SQLITE-048

UPSERT se representará semánticamente.

## DB-SQLITE-049

INSERT OR REPLACE no será considerado UPSERT universal.

## DB-SQLITE-050

RETURNING será capability version-aware.

## DB-SQLITE-051

RETURNING y generated-key API serán distintos.

## DB-SQLITE-052

Schema introspection producirá Canonical Schema Model.

## DB-SQLITE-053

SQLite PRAGMAs/catalog details no escaparán del adapter.

## DB-SQLITE-054

Schema round-trip deberá ser estable.

## DB-SQLITE-055

Migration Planner seleccionará native ALTER o rebuild.

## DB-SQLITE-056

Table rebuild será una estrategia explícita.

## DB-SQLITE-057

Table rebuild preservará objetos conocidos.

## DB-SQLITE-058

Objetos desconocidos no serán destruidos silenciosamente.

## DB-SQLITE-059

Destructive migrations serán clasificadas.

## DB-SQLITE-060

Foreign-key state temporal será restaurado/verificado.

## DB-SQLITE-061

Migration plans mostrarán rebuilds costosos.

## DB-SQLITE-062

Unknown declared types serán preservados.

## DB-SQLITE-063

ORM podrá exigir mapping explícito para unknown types.

## DB-SQLITE-064

Native/extended error codes serán preferidos para clasificación.

## DB-SQLITE-065

Corruption no será reintentada automáticamente.

## DB-SQLITE-066

Disk-full no será clasificado como lock temporal.

## DB-SQLITE-067

SQLite file security será parte del modelo de seguridad.

## DB-SQLITE-068

Multitenancy no será dependencia del Platform.

## DB-SQLITE-069

Tenant file paths deberán resolverse de forma segura.

## DB-SQLITE-070

Encryption no será asumida como capability core.

## DB-SQLITE-071

SQLite memory tests no implicarán producción-equivalence.

## DB-SQLITE-072

Tests `:memory:` podrán pinnear conexión.

## DB-SQLITE-073

Parallel tests tendrán aislamiento explícito.

## DB-SQLITE-074

Capability checks sustituirán version/vendor checks distribuidos.

## DB-SQLITE-075

Capability discovery no ocurrirá por query.

## DB-SQLITE-076

Fingerprints de capability/schema/session/config permanecerán separados.

## DB-SQLITE-077

Bulk batch sizing será capability-aware.

## DB-SQLITE-078

Performance tuning no reducirá durability silenciosamente.

## DB-SQLITE-079

Maintenance API permanecerá separada de Query Builder.

## DB-SQLITE-080

Reset incierto provocará discard.

## DB-SQLITE-081

FrankenPHP será el runtime persistente de referencia inicial.

## DB-SQLITE-082

RoadRunner utilizará el mismo lifecycle neutral.

## DB-SQLITE-083

OpenSwoole requerirá aislamiento coroutine-safe.

## DB-SQLITE-084

ORM nunca generará SQLite SQL.

## DB-SQLITE-085

Schema API nunca decidirá directamente la sintaxis ALTER SQLite.

## DB-SQLITE-086

Actualizar SQLite podrá cambiar estrategias sin cambiar API pública.

## DB-SQLITE-087

Query Compiler Cache incluirá capability compatibility.

## DB-SQLITE-088

Schema fidelity será explícita.

## DB-SQLITE-089

Zero-downtime capabilities no serán exageradas.

## DB-SQLITE-090

SQLite-specific behavior permanecerá dentro de adapters/extensiones específicas.

---

# 414. Anti-pattern — SQLite como MySQL pequeño

Incorrecto:

```text
if SQLite:
    use almost the same SQL
```

---

# 415. Correcto

```text
Semantic Operation
      │
      ▼
Capability Analysis
      │
      ▼
SQLite Strategy
      │
      ▼
SqliteDialect
```

---

# 416. Anti-pattern — Memory database pool

Incorrecto:

```yaml
database: ":memory:"
pool:
  max: 20
```

sin definir semántica de sharing.

---

# 417. Correcto

```text
MemoryDatabase
      │
      ▼
MemoryLifetimePolicy
      │
      ▼
Connection Ownership Strategy
```

---

# 418. Anti-pattern — Foreign keys assumed

Incorrecto:

```text
schema contains FK
→ enforcement guaranteed
```

---

# 419. Correcto

```text
ForeignKey Definition
      │
      +
Connection FK Enforcement State
      │
      ▼
Effective ForeignKey Semantics
```

---

# 420. Anti-pattern — PRAGMA everywhere

Incorrecto:

```php
DB::statement('PRAGMA foreign_keys = ON');
```

disperso por application code.

---

# 421. Correcto

```text
ConnectionSessionProfile
      │
      ▼
SqliteSessionInitializer
      │
      ▼
Known PRAGMA Baseline
```

---

# 422. Anti-pattern — Fake FOR UPDATE

Incorrecto:

```text
Application asks FOR UPDATE
SQLite ignores it
VoltStack reports success
```

---

# 423. Correcto

```text
Lock Requirement
      │
      ▼
Capability Validation
      │
      └── unsupported
             │
             ▼
CapabilityNotSupportedException
```

---

# 424. Anti-pattern — Always rebuild

Incorrecto:

```text
any ALTER
→ table rebuild
```

---

# 425. Correcto

```text
Schema Operation
      │
      ▼
SQLite CapabilitySnapshot
      │
      ├── native strategy
      └── rebuild strategy
```

Esto permite aprovechar mejoras de versiones futuras.

---

# 426. Anti-pattern — Never rebuild

También incorrecto:

```text
SQLite cannot alter this
→ migration impossible
```

cuando una reconstrucción segura puede implementarlo.

---

# 427. Correct migration abstraction

```text
Desired Schema Change
      │
      ▼
Migration Planner
      │
      ▼
Equivalent Safe Strategy
```

---

# 428. Anti-pattern — Silent rebuild

Incorrecto:

```text
drop column
→ hidden full table copy
```

sin informar al usuario.

---

# 429. Correcto

Migration Plan deberá mostrar:

```text
strategy
cost
risk
locks
data copy
transaction behavior
```

---

# 430. Anti-pattern — AUTOINCREMENT everywhere

Incorrecto:

```text
auto ID
=
AUTOINCREMENT
```

---

# 431. Correcto

```text
Generated Identifier Requirement
      │
      ▼
SQLite Identifier Strategy
      │
      ├── rowid identity
      ├── autoincrement identity
      └── application generated
```

---

# 432. Anti-pattern — VARCHAR semantics assumption

Incorrecto:

```text
VARCHAR(255)
=
same enforcement on every DB
```

---

# 433. Correcto

```text
Portable String Type
      │
      ▼
Platform Mapping
      │
      ▼
Platform Semantic Diagnostics
```

---

# 434. Anti-pattern — test portability assumption

Incorrecto:

```text
tests pass on SQLite
=
application works identically on PostgreSQL
```

---

# 435. Correcto

```text
Fast SQLite Tests
+
Production Platform Integration Tests
+
Cross-Platform Conformance Tests
```

---

# 436. Anti-pattern — unsafe path

Incorrecto:

```php
DB::connection('/data/' . $tenant . '.sqlite');
```

---

# 437. Correcto

```text
Tenant/Database Identity
      │
      ▼
Validated Resolver
      │
      ▼
DatabaseFilePath
      │
      ▼
ConnectionDefinition
```

---

# 438. Anti-pattern — unsafe pooled state

Incorrecto:

```text
Request A
ATTACH database
PRAGMA query_only
temp table
      │
      ▼
Pool
      │
      ▼
Request B
```

---

# 439. Correcto

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
Reset Plan
      │
      ├── known clean → reuse
      └── unknown/unsafe → discard
```

---

# 440. Reference architecture

```text
                        APPLICATION
                             │
                             ▼
                        DATABASE API
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
            Query           ORM           Schema
              │              │              │
              └──────────────┼──────────────┘
                             ▼
                       Semantic Model
                             │
                             ▼
                           Planner
                             │
                ┌────────────┼────────────┐
                ▼            ▼            ▼
           Capabilities   Platform    Migration
                          Semantics    Strategies
                │            │            │
                └────────────┼────────────┘
                             ▼
                       SqlitePlatform
                             │
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
    Type Affinity      Schema Semantics   Tx/Lock Semantics
          │                  │                  │
          └──────────────────┼──────────────────┘
                             ▼
                        SqliteDialect
                             │
                             ▼
                          Compiler
                             │
                             ▼
                         Executor
                             │
                             ▼
                         Connection
                             │
                             ▼
                       PdoSqliteDriver
                             │
                             ▼
                           SQLite
```

---

# 441. Migration reference architecture

```text
Desired Schema
      │
      ▼
Current Canonical Schema
      │
      ▼
Schema Diff
      │
      ▼
Migration Planner
      │
      ├──────────────┐
      ▼              ▼
Native ALTER    Table Rebuild
      │              │
      └──────┬───────┘
             ▼
       Migration Plan
             │
             ▼
      Safety Analysis
             │
             ▼
          Executor
```

---

# 442. Runtime reference architecture

```text
FrankenPHP Worker
       │
       ├── SqlitePlatform
       ├── SqliteDialect
       ├── Capability Rules
       ├── Type Mapping
       │
       ├── Request A
       │     │
       │     ├── DatabaseContext A
       │     ├── ConnectionLease A
       │     ├── Transaction A
       │     └── SQLite State A
       │
       ├── RESET
       │
       └── Request B
             │
             ├── DatabaseContext B
             ├── ConnectionLease B
             ├── Transaction B
             └── SQLite State B
```

---

# 443. SQLite effective platform formula

```text
SqlitePlatform
+
SQLite Engine Version
+
Compile-Time Capabilities
+
Driver Capabilities
+
Connection Configuration
+
Relevant PRAGMA State
+
Extensions
+
Policy
=
Effective SQLite CapabilitySnapshot
```

---

# 444. Migration formula

```text
Schema Diff
+
SQLite Version
+
Capabilities
+
Current Schema Metadata
+
Safety Policy
=
SQLite Migration Plan
```

---

# 445. Memory database formula

```text
Memory Database Semantics
=
Endpoint Type
+
Connection Identity
+
Sharing Strategy
+
Connection Lifetime
+
Pool Policy
```

---

# 446. Reusable connection formula

```text
Reusable SQLite Connection
=
No Active Transaction
+
No Active Cursor
+
Known PRAGMA State
+
Known Attachments
+
Known Temporary State
+
Compatible Configuration Generation
+
Successful Reset
```

---

# 447. ORM formula

```text
Entity Changes
+
Mapping Metadata
+
SQLite Capabilities
+
Identifier Strategy
=
Persistence Plan
```

---

# 448. Portability model

SQLite deberá integrarse con tres niveles principales:

```text
PORTABLE
PLATFORM_AWARE
PLATFORM_SPECIFIC
```

---

# 449. Portable

Ejemplo:

```php
DB::table('users')
    ->where('active', true)
    ->get();
```

---

# 450. Platform-aware

Ejemplos:

```text
UPSERT
RETURNING
generated columns
schema alterations
```

donde el Planner selecciona la estrategia.

---

# 451. Platform-specific

Ejemplos:

```text
PRAGMA
WITHOUT ROWID
STRICT
journal mode
ATTACH
virtual tables
SQLite transaction begin modes
```

---

# 452. Explicit platform-specific API

Estas características deberán aparecer como SQLite-specific cuando no sean portables.

---

# 453. SQLite as first-class platform

SQLite no será tratado como:

```text
temporary test database only
```

ni como:

```text
simplified PDO backend
```

Será:

```text
fully modeled VoltStack database platform
```

---

# 454. Objetivo de Developer Experience

El desarrollador podrá escribir:

```php
$user = DB::table('users')
    ->where('email', $email)
    ->first();
```

sin preocuparse por:

```text
PDO_SQLITE
type affinity
PRAGMAs
rowid
journal modes
busy states
```

para operaciones portables.

---

# 455. Advanced Developer Experience

Cuando requiera SQLite-specific features podrá utilizarlas explícitamente sin romper la arquitectura portable.

---

# 456. Example architecture for schema change

La aplicación podrá expresar:

```text
Drop Column
```

mientras internamente:

```text
DropColumnOperation
       │
       ▼
SQLite Capability Analysis
       │
       ├── native supported
       │        │
       │        ▼
       │   Native ALTER
       │
       └── native unavailable
                │
                ▼
          Table Rebuild
```

---

# 457. Evolución automática

La misma migration semántica podrá utilizar una estrategia mejor al ejecutarse sobre una versión SQLite con mayores capabilities.

---

# 458. Determinismo

La selección deberá depender de:

```text
migration definition
capability snapshot
schema state
policy
```

y no de condicionales dispersos.

---

# 459. Relación con documentos anteriores

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
        │
21_DATABASE_SQLITE_PLATFORM
```

---

# 460. Regla maestra

> SQLite deberá integrarse según sus semánticas reales y no mediante la simulación de comportamientos de MySQL o PostgreSQL.

Por tanto:

```text
SQLite
├── Driver Transport
├── Dialect Syntax
├── Platform Semantics
├── Capability Profile
├── Type Affinity
├── Schema Semantics
├── Migration Strategies
├── Transaction Semantics
├── Locking Semantics
├── PRAGMA State
├── Journal Semantics
└── Runtime Lifecycle
```

permanecerán modelados explícitamente.

---

# 461. Decisión final

VoltStack adoptará:

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
SQLite Platform Semantics
        │
        ▼
SQLite Strategy Selection
        │
        ▼
SqliteDialect Compilation
        │
        ▼
Driver-Neutral Execution
        │
        ▼
pdo.sqlite initially
        │
        ▼
SQLite Engine
```

---

# 462. Principios especiales de SQLite

La implementación deberá recordar permanentemente cinco principios:

### 1. Type affinity is not static typing

```text
Declared Type
≠
Guaranteed Runtime Type
```

salvo semánticas adicionales como STRICT.

### 2. Memory database identity depends on connection semantics

```text
:memory:
+
new connection
=
potentially new database
```

### 3. PRAGMAs are state

```text
PRAGMA
≠
random configuration string
```

### 4. Schema migration may require physical reconstruction

```text
Semantic ALTER
≠
necessarily SQL ALTER TABLE
```

### 5. SQLite concurrency is its own model

```text
SQLite Locking
≠
PostgreSQL Row Locking
≠
MySQL/InnoDB Locking
```

---

# 463. Resultado arquitectónico

Con esta arquitectura VoltStack podrá utilizar SQLite para:

```text
development
testing
desktop/local applications
embedded applications
small services
edge scenarios
persistent local storage
specialized production deployments
```

sin comprometer la arquitectura general de `Quantum/Database`.

---

# 464. Conclusión

El soporte SQLite deberá equilibrar:

```text
Laravel-like simplicity
+
SQLite-native correctness
+
Doctrine-like separation
+
VoltStack capability-aware planning
+
persistent-runtime safety
```

La aplicación continuará viendo una API sencilla:

```php
DB::table('users')->get();
```

mientras internamente VoltStack podrá comprender correctamente:

```text
file vs memory database
SQLite engine version
compile-time capabilities
type affinity
STRICT tables
rowid
AUTOINCREMENT
WITHOUT ROWID
foreign-key enforcement
PRAGMAs
journal modes
WAL
busy handling
locking
transaction modes
UPSERT
RETURNING
CTE
window functions
JSON
partial indexes
expression indexes
generated columns
schema introspection
table rebuild migrations
```

sin permitir que estos conceptos contaminen el Query Builder, ORM o Application Layer.

La regla final será:

> **VoltStack no intentará convertir SQLite en PostgreSQL o MySQL. Modelará SQLite como SQLite, mientras conserva una API semántica común para aquello que realmente puede ser portable.**

---

# 465. Siguiente documento

El siguiente documento será:

```text
22_DATABASE_DRIVER_EXTENSION_SYSTEM.md
```

y cerrará el bloque inicial de:

```text
Driver
Connection
Dialect
Platform
Capabilities
Vendor Platforms
```

estableciendo formalmente cómo terceros y paquetes oficiales podrán incorporar nuevos Drivers sin modificar `Quantum/Database`.

Deberá cubrir especialmente:

```text
Driver Extension Contracts
Driver Descriptors
Driver Factories
Driver Registration
Driver Discovery
Driver Availability
Driver Configuration Schema
Native Connection Adapters
Native Statement Adapters
Native Result Adapters
Error Translators
Driver Capability Providers
Driver Versioning
Driver Compatibility
Driver Aliases
Extension Boot Lifecycle
Registry Freeze
Driver Replacement
Driver Decoration
Driver Conformance Testing
Persistent Runtime Safety
Custom Native Clients
Async Drivers
Coroutine Drivers
Cloud Database Connectors
Package Discovery
Security Boundaries
Extension Stability
```

manteniendo la regla:

```text
Custom Driver
≠
Custom Dialect
≠
Custom Platform
```

aunque un mismo paquete pueda registrar los tres componentes cuando sea necesario.