# 10_DATABASE_DRIVER_ARCHITECTURE.md

# VoltStack Quantum Database
## Driver Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 10 — Database Driver Architecture  
**Estado:** Architecture Specification  
**Nivel:** Infrastructure Architecture  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura oficial del sistema de Drivers de:

```text
VoltStack/Quantum/Database
```

El Driver representa la capa de adaptación entre VoltStack Database y la tecnología concreta utilizada para establecer comunicación con un servidor o motor Database.

Su responsabilidad principal será transformar operaciones de infraestructura de bajo nivel en llamadas compatibles con el cliente nativo utilizado por PHP.

Inicialmente deberá soportar:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

sin acoplar el resto del framework a:

```text
PDO
mysqli
pgsql extension
SQLite3
vendor-specific client libraries
```

---

# 2. Definición de Driver

Dentro de VoltStack Database:

> Un Driver es el componente que adapta primitivas Database internas a una API de comunicación nativa.

Conceptualmente:

```text
VoltStack Database
       │
       ▼
Driver Contract
       │
       ▼
Concrete Driver
       │
       ▼
Native PHP Client
       │
       ▼
Database Server
```

---

# 3. Driver no es Connection

La separación fundamental será:

```text
Driver
≠
Connection
```

El Driver conoce:

```text
how to connect
how to prepare
how to bind
how to execute
how to begin/commit/rollback
how to inspect native errors
```

La Connection conoce:

```text
logical connection identity
lifecycle
configuration
read/write role
leasing
pooling
runtime scope
```

---

# 4. Driver no es Dialect

También:

```text
Driver
≠
Dialect
```

El Driver responde:

> ¿Cómo me comunico con el motor?

El Dialect responde:

> ¿Cómo debe expresarse SQL para ese motor?

Ejemplo:

```text
PDO PostgreSQL
        │
        ├── Driver concern: connection + statement API
        └── Dialect concern: PostgreSQL SQL syntax
```

---

# 5. Driver no es Platform

También:

```text
Driver
≠
Platform
```

El Driver representa comunicación.

La Platform representa:

```text
server semantics
features
capabilities
version-dependent behavior
schema behavior
locking behavior
transaction behavior
```

---

# 6. Separación oficial

La relación será:

```text
Connection
    │
    ▼
Driver
    │
    ▼
Native Client

Dialect
    │
    ▼
SQL Syntax

Platform
    │
    ▼
Database Semantics
```

No deberán fusionarse.

---

# 7. Problema a evitar

No deberá existir una clase como:

```text
MySqlConnection
```

que al mismo tiempo:

```text
opens PDO
quotes identifiers
knows MySQL capabilities
compiles SQL
tracks transactions
handles ORM
```

Ese diseño sería demasiado acoplado.

---

# 8. Arquitectura general del Driver subsystem

```text
Driver
├── Contract
├── Descriptor
├── Registry
├── Factory
├── Resolver
├── NativeConnection
├── NativeStatement
├── NativeResult
├── Error
├── Capability
├── Lifecycle
└── Extension
```

---

# 9. Driver Contract

El contrato principal podrá ser:

```php
interface DriverInterface
{
    public function descriptor(): DriverDescriptor;

    public function connect(
        DriverConnectionConfiguration $configuration
    ): NativeConnectionInterface;
}
```

La API definitiva podrá evolucionar, pero deberá permanecer pequeña.

---

# 10. Driver minimalism

El Driver no deberá tener métodos como:

```php
findUser()
saveEntity()
table()
select()
insertModel()
migrate()
```

Su API debe permanecer cercana a primitivas de comunicación.

---

# 11. Driver responsibilities

El Driver podrá ser responsable de:

```text
native connection creation
native client configuration
native statement preparation
native parameter binding support
native execution primitives
native transaction primitives
native error translation
native cancellation support
native result adaptation
```

---

# 12. Driver non-responsibilities

No deberá ser responsable de:

```text
ORM
EntityManager
UnitOfWork
IdentityMap
Query Builder
AST
Semantic Analysis
Query Planning
Schema Model
Migration Model
business validation
authorization
tenant domain logic
```

---

# 13. DriverDescriptor

Cada Driver deberá exponer un descriptor.

Ejemplo conceptual:

```php
final readonly class DriverDescriptor
{
    public function __construct(
        public DriverId $id,
        public string $name,
        public Version $version,
        public DriverFamily $family,
    ) {}
}
```

---

# 14. DriverId

El Driver tendrá identificador estable.

Ejemplos:

```text
pdo.mysql
pdo.pgsql
pdo.sqlite
native.mysqli
native.pgsql
```

Esto permite que:

```text
Database Engine
```

y:

```text
PHP Transport Driver
```

sean conceptos distintos.

---

# 15. Motor vs Driver

Ejemplo:

```text
Database Engine:
MySQL

Possible Drivers:
PDO MySQL
mysqli
future async MySQL client
```

Por ello:

```text
mysql
```

no necesariamente deberá identificar para siempre una única implementación.

---

# 16. Driver aliases

Para DX podrá existir:

```text
driver: mysql
```

como alias de:

```text
pdo.mysql
```

si PDO es el Driver por defecto.

La resolución ocurrirá durante configuración/bootstrap.

---

# 17. Default initial strategy

Para VoltStack v1 se recomienda:

```text
PDO-based official drivers
```

para:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

debido a:

```text
availability
consistency
portability
maintainability
mature API
```

sin convertir PDO en parte de la arquitectura pública.

---

# 18. PDO is implementation detail

Regla:

```text
PDO
```

deberá permanecer detrás de:

```text
Driver
NativeConnection
Statement
Result
```

Los consumidores superiores no deberán depender directamente de `PDO`.

---

# 19. Future native drivers

La arquitectura deberá permitir posteriormente:

```text
mysqli
pgsql-native
SQLite3
async clients
coroutine-aware clients
specialized cloud connectors
```

sin cambiar Query/ORM.

---

# 20. NativeConnectionInterface

El Driver devolverá una abstracción de conexión física.

Conceptualmente:

```php
interface NativeConnectionInterface
{
    public function prepare(string $sql): NativeStatementInterface;

    public function execute(
        string $sql,
        ParameterSet $parameters
    ): NativeResultInterface;

    public function close(): void;
}
```

La API real podrá separar ejecución directa y prepared statement.

---

# 21. Native Connection meaning

`NativeConnectionInterface` representa:

```text
physical communication channel
```

No representa:

```text
logical VoltStack Connection
```

---

# 22. Layer model

```text
Application
    │
    ▼
VoltStack Connection
    │
    ▼
Driver
    │
    ▼
NativeConnection
    │
    ▼
PDO / native API
```

---

# 23. NativeConnection ownership

Una NativeConnection deberá estar controlada por:

```text
Connection Manager / Pool
```

no por ORM.

---

# 24. NativeConnection lifecycle

Estados conceptuales:

```text
Created
  │
  ▼
Connected
  │
  ▼
Ready
  │
  ▼
In Use
  │
  ▼
Resetting
  │
  ├── Ready
  └── Closed/Discarded
```

---

# 25. Driver connect()

El Driver deberá recibir una configuración ya:

```text
normalized
validated
typed
```

No deberá consultar:

```text
env()
config()
$_ENV
```

---

# 26. DriverConnectionConfiguration

Podrá incluir:

```text
endpoint
database
credentials
TLS
timeouts
native options
charset/session initialization hints
```

pero sólo aquello que corresponde realmente a conexión nativa.

---

# 27. Configuration projection

El Driver no deberá recibir:

```text
DatabaseConfiguration
```

completa.

Recibirá una proyección específica:

```text
DriverConnectionConfiguration
```

---

# 28. Credential handling

La resolución de credenciales deberá ocurrir cerca de:

```text
Driver/ConnectionFactory boundary
```

para minimizar exposición.

---

# 29. Driver and secrets

El Driver deberá evitar:

```text
logging credentials
embedding secrets in exception messages
serializing passwords
```

---

# 30. DSN generation

Si se utiliza PDO:

```text
Driver
```

puede construir la forma nativa del DSN a partir de objetos tipados.

Ejemplo conceptual:

```text
ConnectionConfiguration
      │
      ▼
PDO MySQL Driver
      │
      ▼
mysql:host=...;port=...;dbname=...
```

---

# 31. DSN ownership

El DSN nativo pertenece al Driver.

No al Query Engine.

---

# 32. Driver-specific configuration

Cada Driver podrá soportar opciones propias mediante:

```text
NativeDriverOptionSet
```

Estas opciones no deberán filtrarse hacia las capas superiores.

---

# 33. Portable vs native options

Separación:

```text
Portable Connection Options
        │
        ▼
Connection subsystem

Native Driver Options
        │
        ▼
Driver subsystem
```

---

# 34. Native option validation

El Driver deberá validar opciones nativas conocidas cuando sea posible.

Opciones inválidas no deberán ignorarse silenciosamente.

---

# 35. NativeStatementInterface

Contrato conceptual:

```php
interface NativeStatementInterface
{
    public function bind(
        ParameterSet $parameters
    ): void;

    public function execute(): NativeResultInterface;

    public function close(): void;
}
```

La API exacta dependerá del diseño del Execution System.

---

# 36. Statement responsibility

NativeStatement deberá encargarse únicamente de interacción con el statement del cliente nativo.

No deberá:

```text
hydrate entities
detect N+1
run ORM lifecycle events
compile SQL
```

---

# 37. Parameter binding

Driver deberá proporcionar binding seguro.

Flujo:

```text
CompiledQuery
     │
     ▼
Execution Engine
     │
     ▼
Driver Binding Adapter
     │
     ▼
Native Statement
```

---

# 38. Parameter types

El Driver podrá necesitar mapear tipos VoltStack a tipos del cliente nativo.

Ejemplo:

```text
BOOLEAN
INTEGER
STRING
BINARY
NULL
```

El Type System superior podrá realizar conversiones semánticas antes.

---

# 39. Driver-level type concern

El Driver sólo deberá resolver:

```text
binding mechanics
```

No convertir un:

```text
Money Value Object
```

a representation Database.

Eso pertenece al Type System.

---

# 40. Binding model

```text
Application Value
      │
      ▼
Database Type Conversion
      │
      ▼
Database Scalar/Native Value
      │
      ▼
Driver Binding
      │
      ▼
Native Statement
```

---

# 41. Prepared statements

Los Drivers oficiales deberán soportar prepared statements cuando el cliente subyacente lo permita.

Prepared statements serán el mecanismo normal para parámetros.

---

# 42. Native emulated prepares

PDO puede tener diferencias entre:

```text
native prepares
emulated prepares
```

Este comportamiento deberá ser explícito en Driver Configuration/Capabilities.

---

# 43. Prepare capability

Podrá existir capability:

```text
driver.statement.native_prepare
```

separada de:

```text
driver.statement.prepare
```

si la distinción es relevante.

---

# 44. Execution primitives

El Driver podrá exponer primitivas para:

```text
prepare
execute
fetch
row count
last insert/native generated key
cancel
```

según soporte.

No deberá exponer abstracciones de alto nivel.

---

# 45. NativeResultInterface

Podrá representar resultados del cliente.

Conceptualmente:

```php
interface NativeResultInterface
{
    public function fetch(): ?array;

    public function affectedRows(): int;

    public function close(): void;
}
```

El Execution/Result subsystem adaptará esto posteriormente a `ResultInterface`.

---

# 46. Native result is not public result

```text
NativeResultInterface
       !=
ResultInterface
```

El primero pertenece al Driver.

El segundo pertenece a la capa Result/Execution.

---

# 47. Result adaptation

```text
Native Result
     │
     ▼
Driver/Execution Adapter
     │
     ▼
VoltStack Result
```

ORM sólo verá el resultado VoltStack.

---

# 48. Streaming support

Un Driver podrá declarar:

```text
driver.result.streaming
```

y proporcionar primitives para lectura incremental.

---

# 49. Buffered vs unbuffered results

Algunos Drivers pueden permitir:

```text
buffered
unbuffered
server-side cursor
```

Estas diferencias deberán expresarse mediante capabilities/configuration.

---

# 50. Driver capabilities

El Driver tendrá su propio capability provider.

Ejemplos:

```text
prepared statements
native prepared statements
streaming
query cancellation
async execution
multiple result sets
generated key retrieval
connection ping
native reset
```

---

# 51. Driver capability namespace

Ejemplos:

```text
database.driver.statement.prepare
database.driver.statement.native_prepare
database.driver.result.streaming
database.driver.query.cancel
database.driver.connection.ping
database.driver.connection.reset
database.driver.execution.async
```

---

# 52. Driver capabilities vs platform capabilities

No confundir:

```text
driver.query.cancel
```

con:

```text
platform.query.returning
```

Uno depende del cliente.

El otro del motor SQL.

---

# 53. Capability composition example

```text
PostgreSQL server supports feature X
        +
PDO pgsql supports mechanism Y
        │
        ▼
Effective execution capability
```

Ambas capas pueden ser necesarias.

---

# 54. Transaction primitives

El Driver deberá poder representar primitivas:

```text
begin
commit
rollback
savepoint command execution support
```

pero la semántica de Transaction Manager pertenece a otra capa.

---

# 55. Driver transaction responsibility

Driver:

```text
send native BEGIN
send COMMIT
send ROLLBACK
```

Transaction Manager:

```text
nested semantics
retry
transaction context
rollback-only state
connection pinning
```

---

# 56. Savepoints

Driver puede ejecutar comandos de savepoint.

Dialect/Platform determinan:

```text
syntax
support
semantics
```

No debe hardcodearse toda la política dentro del Driver.

---

# 57. Autocommit

El Driver podrá controlar autocommit si el cliente lo requiere.

Pero el estado lógico será administrado por Connection/Transaction subsystem.

---

# 58. Error translation

Una de las responsabilidades centrales del Driver será traducir errores nativos.

```text
PDOException
native error code
SQLSTATE
        │
        ▼
Driver Error Translator
        │
        ▼
Normalized Database Failure
```

---

# 59. Native errors must not leak

APIs estables no deberán depender directamente de:

```text
PDOException
mysqli_sql_exception
native resource errors
```

---

# 60. Error information

El Driver podrá extraer:

```text
SQLSTATE
vendor code
message
connection status
statement state
constraint metadata if available
```

y entregarlo a una estructura neutral.

---

# 61. NativeErrorDescriptor

Conceptualmente:

```php
final readonly class NativeDatabaseError
{
    public function __construct(
        public ?string $sqlState,
        public string|int|null $vendorCode,
        public string $message,
        public NativeErrorSeverity $severity,
    ) {}
}
```

---

# 62. Failure classification

El Driver traduce información nativa.

`FailureClassifier` superior clasifica:

```text
deadlock
timeout
connection loss
constraint violation
authentication failure
```

No deberán fusionarse necesariamente.

---

# 63. Error message sanitization

Driver deberá evitar incluir:

```text
password
complete DSN
sensitive bindings
credentials
```

en errores normalizados.

---

# 64. Connection error distinction

Debe distinguir:

```text
connection creation failure
authentication failure
network failure
TLS failure
server rejection
```

cuando la información esté disponible.

---

# 65. Statement error distinction

También:

```text
syntax error
constraint violation
timeout
deadlock
cancelled
```

cuando pueda detectarse.

---

# 66. SQLSTATE

VoltStack podrá usar SQLSTATE como una señal de clasificación cuando el driver lo proporcione.

Pero no deberá asumir uniformidad perfecta entre vendors.

---

# 67. Driver error translator

Cada Driver podrá registrar:

```text
DriverErrorTranslatorInterface
```

o componente equivalente.

---

# 68. DriverFactory

La creación de Drivers podrá separarse de la creación de conexiones.

Conceptualmente:

```php
interface DriverFactoryInterface
{
    public function create(
        DriverDescriptor $descriptor
    ): DriverInterface;
}
```

En Drivers stateless una única instancia puede compartirse.

---

# 69. Driver lifetime

Preferencia:

```text
Driver
→ persistent/stateless singleton
```

si no almacena estado de conexión.

---

# 70. Driver must not own current connection

Incorrecto:

```php
final class PdoMySqlDriver
{
    private ?PDO $connection;
}
```

si esa propiedad representa la conexión activa actual.

---

# 71. Correct Driver

```text
PdoMySqlDriver
    │
    └── connect() → NativeConnection object
```

El estado vive en el objeto conexión.

---

# 72. Concurrency safety

Un Driver compartido deberá ser:

```text
stateless
immutable
reentrant
```

o documentar explícitamente sincronización.

---

# 73. Driver Registry

`DriverRegistry` relacionará IDs con factories/providers.

Ejemplo:

```text
mysql
    → pdo.mysql

pgsql
    → pdo.pgsql

sqlite
    → pdo.sqlite
```

---

# 74. Registry structure

Conceptualmente:

```php
$registry->register(
    DriverId::from('pdo.pgsql'),
    PostgreSqlPdoDriverFactory::class
);
```

---

# 75. Registry freeze

Después de bootstrap:

```text
DriverRegistry
     │
     ▼
freeze
```

No deberá modificarse durante requests ordinarios.

---

# 76. Duplicate Driver IDs

Un duplicate deberá producir error salvo replacement explícito.

No:

```text
last one wins
```

---

# 77. Driver aliases

Aliases deberán registrarse por separado.

Ejemplo:

```text
pgsql
→ pdo.pgsql
```

Esto permite reemplazar defaults en una futura major version sin cambiar conceptualmente el engine identifier.

---

# 78. DriverResolver

Resolverá:

```text
configuration driver ID
      │
      ▼
registered DriverDescriptor
      │
      ▼
DriverFactory
      │
      ▼
Driver
```

---

# 79. Driver resolution is bootstrap/infrastructure concern

Query Builder nunca deberá ejecutar:

```php
$drivers->resolve('pgsql');
```

---

# 80. Driver family

Podrá existir:

```text
PDO
NATIVE
ASYNC
COROUTINE
```

como descriptor, si aporta valor.

No deberá utilizarse como sustituto de capabilities.

---

# 81. PDO Driver architecture

Inicialmente:

```text
PdoDriver
├── PdoNativeConnection
├── PdoNativeStatement
├── PdoNativeResult
├── PdoErrorTranslator
└── vendor-specific specializations
```

---

# 82. Shared PDO infrastructure

Podrá existir infraestructura compartida:

```text
AbstractPdoDriver
```

o composition helpers.

Pero los consumidores dependerán de `DriverInterface`.

---

# 83. Avoid deep inheritance

No se recomienda:

```text
AbstractDriver
  ↓
AbstractPdoDriver
  ↓
AbstractSqlDriver
  ↓
AbstractMySqlFamilyDriver
  ↓
MySqlDriver
```

Preferir composición cuando sea posible.

---

# 84. PDO MySQL Driver

Responsabilidades específicas:

```text
build MySQL PDO DSN
configure PDO attributes
map native errors
create PdoNativeConnection
declare PDO MySQL driver capabilities
```

No:

```text
compile MySQL SQL
```

---

# 85. PDO MariaDB Driver

MariaDB podrá reutilizar gran parte de infraestructura MySQL, pero:

```text
MariaDB Platform
```

y:

```text
MariaDB Dialect/Capabilities
```

permanecerán conceptualmente separadas cuando existan diferencias.

---

# 86. PDO PostgreSQL Driver

Responsabilidades:

```text
pgsql PDO DSN
PDO options
native connection adapter
native error translation
driver capabilities
```

---

# 87. PDO SQLite Driver

Deberá soportar:

```text
file database
in-memory database
possibly URI forms
```

sin exigir host/port.

---

# 88. SQLite is special but not exceptional architecture

No deberá crearse una arquitectura alternativa.

SQLite seguirá:

```text
Connection
   │
   ▼
Driver
   │
   ▼
NativeConnection
   │
   ▼
SQLite database
```

---

# 89. Driver validation

Cada Driver podrá validar que su extensión PHP requerida está disponible.

Ejemplo:

```text
pdo_pgsql installed?
```

---

# 90. Missing PHP extension

Error claro:

```text
DatabaseDriverUnavailableException

Driver:
pdo.pgsql

Required PHP extension:
pdo_pgsql
```

---

# 91. Driver availability

Estados conceptuales:

```text
Registered
Available
Unavailable
```

Un Driver puede estar registrado pero no disponible en el entorno actual.

---

# 92. Availability detection

Deberá ocurrir durante bootstrap/first use según coste.

No necesariamente durante cada conexión.

---

# 93. DriverDescriptor availability requirements

Podrá declarar:

```text
required PHP extensions
minimum PHP version
native library requirements
```

---

# 94. Driver version

Debe distinguirse:

```text
VoltStack Driver Version
```

de:

```text
native client version
```

y:

```text
database server version
```

---

# 95. Client version capability

Algunas capabilities pueden depender de la versión del cliente nativo.

El Driver Capability Provider podrá considerarlo.

---

# 96. Driver initialization

Un Driver persistente podrá inicializar metadata estática.

No deberá:

```text
connect to database
select tenant
start transaction
```

durante su construcción.

---

# 97. Connection initialization

Después de crear NativeConnection puede ejecutarse un pipeline controlado de inicialización.

Ejemplos:

```text
set charset
configure session
verify TLS
set application name
```

Pero la ownership de estas policies deberá coordinarse con Connection subsystem.

---

# 98. Driver connection hook

El Driver puede proporcionar primitivas.

La Connection Initialization pipeline decide qué aplicar.

---

# 99. Driver reset primitive

Si el cliente proporciona reset nativo:

```text
reset session
```

el Driver podrá exponerlo mediante capability/contract.

---

# 100. Reset is not solely Driver responsibility

El sistema superior necesita saber:

```text
what state VoltStack changed
what state tenant changed
what transaction state exists
```

por lo que Connection Reset System coordina.

Driver ejecuta primitivas nativas.

---

# 101. Ping capability

Un Driver podrá soportar:

```text
database.driver.connection.ping
```

para health/pool validation.

No todos los clientes necesitan una primitive especial; puede existir fallback seguro.

---

# 102. Query cancellation

Si el cliente permite cancelación:

```text
Execution
    │
    ▼
Driver cancellation primitive
```

La capability deberá declararse.

---

# 103. Cancellation ownership

El Runtime/Executor decide:

```text
when cancel
```

El Driver sabe:

```text
how cancel
```

---

# 104. Async drivers

La arquitectura deberá permitir en el futuro:

```text
AsyncDriverInterface
```

o capability especializada sin romper el contrato síncrono inicial.

---

# 105. Do not pollute core sync contract prematurely

VoltStack v1 no deberá diseñar una API async artificial si no existe implementación real.

Extensibilidad futura deberá preservarse mediante boundaries correctos.

---

# 106. Multiple result sets

Si un cliente lo soporta:

```text
database.driver.result.multiple_sets
```

podrá declararse.

El Result System decidirá cómo exponerlo.

---

# 107. Generated keys

Driver puede exponer una primitive de:

```text
native generated identifier
```

cuando el cliente la ofrece.

Pero ORM/Planner decide cuándo utilizarla.

---

# 108. Generated key capability

Puede distinguir:

```text
driver.generated_key
```

de:

```text
platform.query.returning
```

Dos estrategias distintas para obtener IDs.

---

# 109. Last inserted ID

No deberá asumirse que:

```text
lastInsertId()
```

tiene la misma semántica en todos los motores.

Platform/ORM Strategy deberá decidir su validez.

---

# 110. Driver and schema

Driver no deberá inspeccionar schema por sí mismo salvo primitives nativas.

`SchemaIntrospector` utiliza:

```text
Query/Execution/Platform
```

según arquitectura.

---

# 111. Driver and migrations

Driver no conoce migrations.

Migration system termina ejecutando statements a través de Execution/Connection.

---

# 112. Driver and ORM

Prohibido:

```text
Driver
→ Entity
Driver
→ UnitOfWork
Driver
→ Repository
```

---

# 113. Driver and Query Builder

Prohibido:

```text
Driver
→ QueryBuilder
```

---

# 114. Driver and Compiler

La relación deberá ser indirecta.

```text
Compiler
   │
   ▼
CompiledQuery

Executor
   │
   ▼
Connection
   │
   ▼
Driver
```

Driver no recibe AST.

---

# 115. Driver and Dialect

Un Driver puede declarar un Dialect recomendado mediante descriptor/bootstrap metadata.

Pero no debe ser responsable de compilar SQL.

---

# 116. Driver and Platform

También puede declarar una Platform family por defecto.

La Platform efectiva puede depender de server discovery/version.

---

# 117. Driver resolution pipeline

```text
Connection Configuration
        │
        ▼
Driver ID
        │
        ▼
DriverResolver
        │
        ▼
DriverDescriptor
        │
        ▼
DriverFactory
        │
        ▼
Driver
```

---

# 118. Connection creation pipeline

```text
ConnectionManager
      │
      ▼
ConnectionFactory
      │
      ▼
Driver
      │
      ▼
Credential Resolution
      │
      ▼
DriverConnectionConfiguration
      │
      ▼
NativeConnection
      │
      ▼
Initialization
      │
      ▼
Ready Physical Connection
```

---

# 119. Statement pipeline

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
NativeConnection
     │
     ▼
prepare
     │
     ▼
NativeStatement
     │
     ▼
bind
     │
     ▼
execute
     │
     ▼
NativeResult
```

---

# 120. Driver failure pipeline

```text
Native Client Error
        │
        ▼
Driver Error Translator
        │
        ▼
NativeDatabaseError
        │
        ▼
Failure Classifier
        │
        ▼
Database Failure
        │
        ▼
Execution/Connection Exception
```

---

# 121. Driver state model

Preferred:

```text
Driver
├── immutable descriptor
├── immutable driver capabilities
└── stateless factories/helpers
```

No:

```text
current connection
current statement
current transaction
current tenant
```

---

# 122. Driver runtime safety

Un Driver singleton deberá funcionar correctamente con:

```text
Request A
Request B
Request C
```

simultáneamente o secuencialmente porque no almacena operación state.

---

# 123. NativeConnection state

En cambio:

```text
NativeConnection
```

sí es mutable y deberá tener ownership explícito.

---

# 124. Statement state

```text
NativeStatement
```

será operation/resource scoped.

---

# 125. NativeResult state

```text
NativeResult
```

también deberá cerrarse/liberarse según lifecycle.

---

# 126. Driver extension model

Paquetes externos podrán registrar Drivers mediante:

```text
Driver Extension Contract
```

sin modificar core.

---

# 127. Custom Driver requirements

Un custom Driver deberá proporcionar:

```text
DriverDescriptor
DriverFactory
NativeConnection adapter
NativeStatement adapter
NativeResult adapter
ErrorTranslator
DriverCapabilityProvider
```

según las features que soporte.

---

# 128. Driver extension isolation

No tendrá acceso privilegiado a ORM internals.

---

# 129. Driver conformance suite

Todo Driver deberá pasar:

```text
DriverConformanceSuite
```

---

# 130. Driver conformance categories

La suite deberá verificar:

```text
availability
connection
prepared statements
binding
result fetching
transaction primitives
error translation
resource cleanup
connection close
capability accuracy
persistent worker safety
```

---

# 131. Connection conformance test

Ejemplo conceptual:

```text
connect
→ execute SELECT 1
→ close
→ resource inaccessible
```

---

# 132. Prepared statement conformance

Debe comprobar:

```text
parameters are not interpolated unsafely
bindings preserve values
NULL works correctly
binary data works correctly
```

---

# 133. Transaction primitive conformance

```text
begin
write
rollback
verify write absent
```

y:

```text
begin
write
commit
verify write present
```

---

# 134. Error translation conformance

Cada Driver deberá normalizar errores comunes de forma consistente.

Ejemplos:

```text
invalid credentials
missing table
unique violation
foreign key violation
deadlock where reproducible
```

---

# 135. Cleanup conformance

Después de:

```text
failed query
cancelled query
statement error
```

el Driver deberá indicar correctamente si la conexión:

```text
remains usable
needs reset
must be discarded
```

---

# 136. Capability conformance

Cada capability declarada deberá probarse.

Ejemplo:

```text
driver.result.streaming = NATIVE
```

requiere una prueba que demuestre comportamiento incremental real.

---

# 137. Persistent runtime conformance

Test:

```text
same Driver instance
    │
    ├── Connection A
    └── Connection B
```

sin contaminación de estado.

---

# 138. Concurrent Driver conformance

Cuando el runtime soporte concurrencia:

```text
same Driver singleton
├── coroutine A
└── coroutine B
```

deberá permanecer seguro.

---

# 139. Architecture tests

Deberán impedir que namespaces Driver dependan de:

```text
ORM
Repository
Persistence
Hydration
Application
Controller
HTTP
```

---

# 140. Allowed dependencies

Driver podrá depender de:

```text
Platform contracts
Database low-level contracts
Configuration value objects
Error value objects
Capability contracts
Support primitives
```

---

# 141. Dependency graph

```text
Driver
   │
   ├── Configuration Value Objects
   ├── Driver Contracts
   ├── Capability Contracts
   ├── Native Resource Contracts
   └── Error Model
```

No dependencies upward.

---

# 142. Suggested namespace structure

```text
VoltStack\Quantum\Database\Driver
│
├── Contract
│   ├── DriverInterface.php
│   ├── DriverFactoryInterface.php
│   ├── NativeConnectionInterface.php
│   ├── NativeStatementInterface.php
│   ├── NativeResultInterface.php
│   └── DriverErrorTranslatorInterface.php
│
├── Descriptor
│   └── DriverDescriptor.php
│
├── Registry
│   └── DriverRegistry.php
│
├── Resolver
│   └── DriverResolver.php
│
├── Configuration
│   └── DriverConnectionConfiguration.php
│
├── Capability
├── Error
├── Pdo
│   ├── Connection
│   ├── Statement
│   ├── Result
│   ├── Error
│   ├── MySql
│   ├── MariaDb
│   ├── PostgreSql
│   └── SQLite
│
└── Exception
```

---

# 143. PDO shared layer

Conceptualmente:

```text
Driver/Pdo
├── PdoNativeConnection
├── PdoNativeStatement
├── PdoNativeResult
└── PdoErrorAdapter
```

mientras vendor-specific:

```text
Pdo/MySql
Pdo/MariaDb
Pdo/PostgreSql
Pdo/SQLite
```

aportan diferencias necesarias.

---

# 144. Avoid duplicated PDO wrappers

No deberá existir una copia completa de:

```text
PdoNativeConnection
```

para cada vendor si la diferencia puede componerse mediante strategies.

---

# 145. Avoid universal PDO class

Tampoco deberá existir una única clase gigante:

```text
PdoDriver
```

con:

```php
switch ($vendor) {
}
```

para toda diferencia.

---

# 146. Balanced specialization

Preferencia:

```text
Shared PDO primitives
        +
Vendor Driver Adapter
        +
Vendor Platform
        +
Vendor Dialect
```

---

# 147. Driver boot model

```text
Database Bootstrap
      │
      ▼
Register Driver Descriptors
      │
      ▼
Validate Runtime Availability
      │
      ▼
Compile Driver Registry
      │
      ▼
Freeze
```

No connections required.

---

# 148. Driver runtime model

```text
Query requires DB
      │
      ▼
Connection Manager
      │
      ▼
Driver Resolver
      │
      ▼
Driver
      │
      ▼
connect if no usable pooled resource
```

---

# 149. Driver metrics

Telemetry podrá observar:

```text
connection attempts
connection failures
prepare duration
execute duration
driver errors
cancellation
```

pero Driver Core no deberá depender directamente de Telemetry implementation.

---

# 150. Instrumentation placement

Preferencia:

```text
Execution/Connection Decorator
      │
      ▼
Driver
```

en vez de llenar cada Driver de lógica Telemetry.

---

# 151. Low-level driver diagnostics

El Driver puede proporcionar metadata técnica para diagnostics.

Ejemplo:

```text
driver ID
client version
native extension
capabilities
```

sin exponer secrets.

---

# 152. Driver health

Driver health no significa Database server health.

Separación:

```text
Driver Available
≠
Database Reachable
```

---

# 153. Driver availability diagnostic

Ejemplo:

```text
Driver: pdo.pgsql
PHP extension: pdo_pgsql
Status: available
```

---

# 154. Connection diagnostic

Separado:

```text
Connection: primary
Driver: pdo.pgsql
Server: reachable
Authentication: successful
```

---

# 155. Driver caching

Driver instances stateless pueden reutilizarse.

No deberá cachearse:

```text
NativeStatement
NativeResult
```

globalmente.

---

# 156. Prepared statement cache

Si existe, pertenece:

```text
per physical connection
```

no al Driver singleton.

---

# 157. Driver and pooling

Driver no administra pool.

Pool/Connection subsystem solicita:

```text
connect()
close()
reset primitives
```

al Driver/native connection.

---

# 158. Pool independence

Un Driver deberá funcionar:

```text
with pool
without pool
```

---

# 159. Runtime independence

Un Driver deberá ser neutral respecto a:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

salvo Drivers específicamente async/coroutine-aware en el futuro.

---

# 160. FrankenPHP behavior

Con FrankenPHP:

```text
Driver singleton
→ reusable

Native connections
→ pool/resource managed

Statements/results
→ operation scoped
```

---

# 161. RoadRunner behavior

La misma semántica deberá mantenerse.

---

# 162. OpenSwoole behavior

Drivers utilizados bajo concurrencia deberán declarar si:

```text
native client
```

es compatible con coroutines/concurrent use.

No deberá asumirse automáticamente.

---

# 163. Runtime capability interaction

Ejemplo:

```text
Driver supports async
+
Runtime supports async scheduling
=
async execution strategy available
```

---

# 164. Driver capability requirement

Un feature podrá requerir:

```text
Driver Capability
+
Platform Capability
```

al mismo tiempo.

---

# 165. Example — query cancellation

```text
Runtime timeout
      │
      ▼
Executor
      │
      ▼
requires driver.query.cancel
      │
      ├── supported
      │      └── Driver cancellation
      │
      └── unsupported
             └── fallback/discard according to policy
```

---

# 166. Example — streaming

```text
Application requests stream()
       │
       ▼
Execution Planner
       │
       ▼
driver.result.streaming
       │
       ├── supported
       │       ▼
       │   Native streaming
       │
       └── unsupported
               ▼
          buffered fallback
          or reject
```

según semántica/configuración.

---

# 167. Example — native prepared statements

```text
Security/Execution policy
       │
       ▼
requires native prepare?
       │
       ▼
Driver Capability
```

Si sólo existe prepare emulado, la policy podrá decidir si lo permite.

---

# 168. Driver replacement

Una aplicación avanzada podrá reemplazar:

```text
pdo.mysql
```

por:

```text
custom.mysql
```

sin cambiar:

```text
Query Builder
ORM
Schema
Migration
```

si ambos cumplen contracts/capabilities.

---

# 169. Driver abstraction success criterion

El diseño es correcto si:

> Cambiar el mecanismo de comunicación con la base de datos no requiere reescribir las capas superiores.

---

# 170. Driver public exposure

La mayoría de aplicaciones no deberán interactuar directamente con `DriverInterface`.

Su API será principalmente:

```text
Infrastructure / Extension API
```

---

# 171. Public native handle escape hatch

Si se permite acceso nativo:

```php
$connection->unwrap(PDO::class);
```

será parte de Connection API, no un método principal del Driver.

---

# 172. Escape hatch warning

Acceder al recurso nativo puede:

```text
bypass telemetry
modify session state
affect pooling
bypass safety abstractions
```

por lo que deberá considerarse advanced/unsafe API.

---

# 173. Driver lifecycle ownership table

| Componente | Lifetime | Owner |
|---|---|---|
| DriverDescriptor | Application | DriverRegistry |
| Driver | Application/Worker | Container/Registry |
| DriverFactory | Application/Worker | Container |
| DriverResolver | Application/Worker | Connection infrastructure |
| NativeConnection | Resource | Connection Manager/Pool |
| NativeStatement | Operation | NativeConnection/Executor |
| NativeResult | Operation/Stream | Executor/Result layer |
| NativeError | Value Object | Operation |
| DriverCapabilitySet | Application/Connection | Driver subsystem |

---

# 174. Driver state table

| Estado | ¿Permitido en Driver singleton? |
|---|---|
| Driver descriptor | Sí |
| Static capability definitions | Sí |
| Immutable helper | Sí |
| Current PDO connection | No |
| Current transaction | No |
| Current statement | No |
| Current tenant | No |
| Current query | No |
| Current request | No |

---

# 175. Driver architecture invariants

## DB-DRV-001

Driver no es Connection.

## DB-DRV-002

Driver no es Dialect.

## DB-DRV-003

Driver no es Platform.

## DB-DRV-004

Driver no conoce ORM.

## DB-DRV-005

Driver no conoce Query Builder.

## DB-DRV-006

Driver no recibe AST.

## DB-DRV-007

Driver no compila SQL de alto nivel.

## DB-DRV-008

Driver no administra UnitOfWork.

## DB-DRV-009

Driver no mantiene current connection en estado singleton.

## DB-DRV-010

Driver Configuration llegará normalizada y tipada.

## DB-DRV-011

Driver no leerá configuración global o environment directamente.

## DB-DRV-012

Los recursos nativos deberán permanecer ocultos detrás de contracts internos.

## DB-DRV-013

PDO será implementation detail.

## DB-DRV-014

Prepared statements y binding seguro serán el camino normal de ejecución parametrizada.

## DB-DRV-015

Los errores nativos deberán normalizarse antes de cruzar boundaries estables.

## DB-DRV-016

Las capabilities del Driver deberán ser explícitas.

## DB-DRV-017

Driver Registry deberá congelarse antes del runtime normal.

## DB-DRV-018

Drivers oficiales deberán pasar una suite de conformidad.

## DB-DRV-019

Un Driver persistente deberá ser stateless o concurrency-safe.

## DB-DRV-020

Driver no administrará connection pooling.

## DB-DRV-021

Driver no administrará request lifecycle.

## DB-DRV-022

Driver no administrará tenant lifecycle.

## DB-DRV-023

Un custom Driver no obtendrá acceso implícito a internals superiores.

## DB-DRV-024

Vendor-specific logic deberá concentrarse en Driver/Dialect/Platform especializados.

## DB-DRV-025

Cambiar Driver no deberá cambiar la semántica pública de Query/ORM mientras las capabilities requeridas se mantengan.

---

# 176. Anti-pattern — Driver God Object

```text
MySqlDriver
├── connect
├── query builder
├── compile SQL
├── migrate
├── save entity
├── cache
├── telemetry
└── transaction state
```

**Prohibido.**

---

# 177. Anti-pattern — PDO everywhere

```php
function execute(PDO $pdo)
```

distribuido por:

```text
ORM
Repository
Migration
Schema
```

**Prohibido.**

---

# 178. Anti-pattern — vendor checks above Driver boundary

```php
if ($connection->driver() === 'mysql') {
}
```

en ORM.

**Prohibido.**

---

# 179. Anti-pattern — Driver with scoped state

```php
final class Driver
{
    private ?string $tenantId;
}
```

**Prohibido para persistent singleton Driver.**

---

# 180. Anti-pattern — Driver returns entities

```php
$driver->fetchUser();
```

**Prohibido.**

---

# 181. Anti-pattern — Driver owns pool

```text
Driver
├── open connections
├── idle connections
├── leases
└── current request leases
```

**Prohibido en esta arquitectura.**

Pool pertenece a Connection infrastructure.

---

# 182. Anti-pattern — Driver performs retry policy

Driver no deberá decidir:

```text
retry query three times
```

basándose en semántica de operación que no conoce.

Puede reportar error; Resilience/Executor decide retry.

---

# 183. Anti-pattern — Driver performs transaction retry

Igualmente, no deberá repetir transactions completas.

---

# 184. Example — correct read execution

```text
Query Builder
      │
      ▼
Query AST
      │
      ▼
Planner
      │
      ▼
Compiler
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
Driver
      │
      ▼
Native Statement
      │
      ▼
Database
```

---

# 185. Example — correct ORM persistence

```text
Entity
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
Compiled Query
  │
  ▼
Execution
  │
  ▼
Connection
  │
  ▼
Driver
```

Driver nunca observa Entity.

---

# 186. Example — correct migration

```text
Migration
   │
   ▼
Schema Operations
   │
   ▼
Schema Planner
   │
   ▼
Schema Compiler
   │
   ▼
Compiled Statements
   │
   ▼
Execution
   │
   ▼
Connection
   │
   ▼
Driver
```

---

# 187. Reference architecture

```text
┌──────────────────────────────────────────────┐
│              Upper Database Layers           │
│ Query / Schema / ORM / Migration             │
└────────────────────┬─────────────────────────┘
                     │
                     ▼
                Execution
                     │
                     ▼
                Connection
                     │
                     ▼
               Driver Contract
                     │
         ┌───────────┼────────────┐
         ▼           ▼            ▼
      PDO MySQL   PDO PgSQL   PDO SQLite
         │           │            │
         ▼           ▼            ▼
      Native PHP Client Interfaces
                     │
                     ▼
                Database Server
```

En paralelo:

```text
Dialect
   │
   ▼
SQL Syntax

Platform
   │
   ▼
Server Semantics / Capabilities
```

---

# 188. Initial official drivers

La primera implementación deberá contemplar:

```text
PdoMySqlDriver
PdoMariaDbDriver
PdoPostgreSqlDriver
PdoSqliteDriver
```

La organización interna podrá compartir componentes PDO.

---

# 189. MariaDB decision

Aunque MariaDB y MySQL pueden compartir el mismo transporte PDO:

```text
PDO mysql
```

VoltStack deberá mantener:

```text
MariaDB Platform
```

separada de:

```text
MySQL Platform
```

cuando sus capabilities diverjan.

El Driver puede reutilizar transporte.

---

# 190. Transport reuse

Esto demuestra la separación:

```text
PdoMySqlFamilyTransport
      │
      ├── MySQL Platform
      └── MariaDB Platform
```

El transporte no determina completamente la semántica.

---

# 191. Driver implementation priority

Orden recomendado:

```text
1. Common Driver Contracts
2. Native Resource Contracts
3. Driver Registry
4. Driver Resolver
5. Shared PDO Infrastructure
6. PDO SQLite Driver
7. PDO PostgreSQL Driver
8. PDO MySQL Driver
9. MariaDB specialization
10. Driver Conformance Suite
```

SQLite puede facilitar tests iniciales, pero no deberá definir la arquitectura.

---

# 192. Testing against real engines

La conformidad final deberá probarse también contra:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

reales.

No bastará con mocks.

---

# 193. Test matrix

Conceptualmente:

```text
                    SQLite  PostgreSQL  MySQL  MariaDB

Connection             ✓        ✓         ✓       ✓
Prepare                ✓        ✓         ✓       ✓
Binding                ✓        ✓         ✓       ✓
Transaction            ✓        ✓         ✓       ✓
Rollback               ✓        ✓         ✓       ✓
Error translation      ✓        ✓         ✓       ✓
Streaming              ?        ?         ?       ?
Cancellation           ?        ?         ?       ?
Reset                  ?        ?         ?       ?
```

Los valores reales se documentarán en specs específicas.

---

# 194. Architecture review checklist

Para cada Driver nuevo:

```text
What native client does it use?
What PHP extension does it require?
Is Driver instance stateless?
What does connect() return?
How are statements represented?
How are results represented?
How are errors normalized?
Which capabilities are native?
Which capabilities are unavailable?
How is connection state reset?
How does cancellation behave?
Is it safe under persistent workers?
Is it safe under concurrent contexts?
What resource ownership rules exist?
Does it leak native types upward?
Does it depend on ORM/Query Builder?
```

---

# 195. Security review checklist

```text
Are credentials redacted?
Are TLS options honored?
Are prepared statements safe?
Can native errors expose secrets?
Can DSNs expose passwords?
Can unsafe native options weaken security?
How are binary values bound?
Can connection reuse leak session state?
```

---

# 196. Performance review checklist

```text
Does Driver allocate unnecessarily per row?
Does it wrap each scalar excessively?
Does it use reflection in hot paths?
Can prepared statements be reused safely?
Does it support streaming?
What is connection creation cost?
What reset cost exists?
Can capability metadata be cached?
```

---

# 197. Driver design objective

El Driver debe ser suficientemente pequeño para que un nuevo mecanismo de transporte pueda implementarse sin comprender:

```text
ORM internals
Schema planner
Migration lifecycle
Entity metadata
Application runtime
```

---

# 198. Architectural equation

```text
Driver
=
Native Communication Adapter
+
Error Normalization
+
Driver Capabilities
```

No:

```text
Driver
=
Complete Database Subsystem
```

---

# 199. Final principle

La regla maestra será:

> El Driver sabe cómo hablar con el cliente nativo, pero no sabe por qué la aplicación está hablando con la base de datos.

Esto significa que Driver conoce:

```text
connect
prepare
bind
execute
fetch
transaction primitive
cancel
close
```

pero no conoce:

```text
User
Order
Repository
Entity
QueryBuilder
Migration
Tenant business rule
```

---

# 200. Conclusión

`VoltStack/Quantum/Database` tendrá una capa Driver deliberadamente pequeña, especializada y reemplazable.

La arquitectura completa será:

```text
Application Intent
      │
      ▼
Query / ORM / Schema
      │
      ▼
Planning / Compilation
      │
      ▼
Execution
      │
      ▼
Logical Connection
      │
      ▼
Driver
      │
      ▼
Native Client
      │
      ▼
Database
```

mientras:

```text
Dialect
```

se ocupa de sintaxis y:

```text
Platform
```

de semántica/capabilities.

Esta separación permitirá en el futuro sustituir:

```text
PDO
```

por clientes:

```text
native
async
coroutine-aware
specialized
```

sin rediseñar el ORM, Query Builder o Migration System.

---

# 201. Siguiente documento

El siguiente documento es:

```text
11_DATABASE_CONNECTION_SYSTEM.md
```

Deberá definir formalmente:

```text
logical Connection
physical Connection
connection identities
ConnectionManager
ConnectionFactory
ConnectionResolver
ConnectionScope

connection creation
lazy connection acquisition
connection state
connection lifecycle
connection roles
connection leases
connection reset
connection release
connection errors

relationship with Driver
relationship with Platform
relationship with Dialect

persistent runtime behavior
connection reuse
request isolation
transaction affinity
read/write connection routing
```

manteniendo:

```text
Connection
    ≠
Driver
    ≠
Dialect
    ≠
Platform
```

y estableciendo la frontera:

```text
Application / Execution
        │
        ▼
Logical Connection
        │
        ▼
Connection Infrastructure
        │
        ▼
Driver
        │
        ▼
Physical Connection
```