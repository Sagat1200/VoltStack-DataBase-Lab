# 22_DATABASE_DRIVER_EXTENSION_SYSTEM.md

# VoltStack Quantum Database
## Driver Extension System Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 22 — Driver Extension System  
**Estado:** Architecture Specification  
**Nivel:** Infrastructure / Driver / Extension  
**Versión:** 1.0

---

# 1. Propósito

Este documento define el sistema oficial mediante el cual `VoltStack/Quantum/Database` podrá incorporar nuevos Drivers sin modificar el núcleo del framework.

El Driver Extension System permitirá registrar:

- nuevos Drivers;
- clientes nativos;
- implementaciones PDO;
- clientes síncronos;
- futuros clientes asíncronos;
- clientes coroutine-aware;
- conectores especializados;
- adaptadores cloud;
- factories;
- descriptors;
- configuration schemas;
- native connection adapters;
- native statement adapters;
- native result adapters;
- error translators;
- capability providers;
- aliases;
- availability probes;
- lifecycle hooks controlados;
- conformance suites.

La arquitectura deberá permitir que:

```text
Quantum/Database
```

permanezca cerrado respecto a modificaciones internas y abierto respecto a extensiones compatibles.

---

# 2. Principio fundamental

La regla principal será:

```text
Custom Driver
≠
Custom Dialect
≠
Custom Platform
```

Un paquete podrá registrar los tres:

```text
Driver
+
Dialect
+
Platform
```

pero seguirán siendo componentes independientes.

---

# 3. Ejemplo

Un paquete futuro podría proporcionar:

```text
voltstack/database-oracle
```

y registrar:

```text
Driver:
    pdo.oracle

Dialect:
    oracle

Platform:
    oracle
```

Sin embargo:

```text
pdo.oracle
```

no será automáticamente equivalente a:

```text
OraclePlatform
```

---

# 4. Razón de la separación

El mismo motor puede utilizar diferentes Drivers:

```text
PostgreSQL
├── pdo.pgsql
├── native.pgsql
├── async.pgsql
└── cloud.pgsql
```

mientras todos pueden compartir:

```text
PostgresqlDialect
PostgresqlPlatform
```

si sus semánticas son compatibles.

---

# 5. Caso inverso

Un Driver genérico podría potencialmente comunicarse con distintos perfiles compatibles:

```text
GenericWireDriver
        │
        ├── Platform A
        └── Platform B
```

Por ello el Driver no deberá codificar necesariamente toda la semántica del servidor.

---

# 6. Arquitectura general

```text
Composer Package
      │
      ▼
Driver Extension Discovery
      │
      ▼
Driver Extension Descriptor
      │
      ▼
Compatibility Validation
      │
      ▼
Driver Registration
      │
      ├── Descriptor
      ├── Factory
      ├── Configuration Schema
      ├── Availability Probe
      ├── Error Translator
      └── Capability Provider
             │
             ▼
       Driver Registry
             │
             ▼
        Registry Freeze
             │
             ▼
          Runtime
```

---

# 7. Arquitectura runtime

```text
ConnectionDefinition
       │
       ▼
DriverResolver
       │
       ▼
DriverRegistry
       │
       ▼
DriverDescriptor
       │
       ▼
DriverFactory
       │
       ▼
Driver
       │
       ▼
NativeConnection
       │
       ▼
NativeStatement
       │
       ▼
NativeResult
```

---

# 8. Objetivos

El sistema deberá:

1. permitir Drivers de terceros;
2. evitar modificaciones del núcleo;
3. preservar las fronteras arquitectónicas;
4. validar compatibilidad;
5. impedir registros ambiguos;
6. proporcionar configuración tipada;
7. soportar discovery;
8. soportar capabilities;
9. proporcionar error translation;
10. soportar persistent runtimes;
11. proporcionar conformance testing;
12. mantener seguridad;
13. permitir evolución versionada;
14. mantener bajo overhead runtime.

---

# 9. No objetivos

El Driver Extension System no será responsable de:

```text
Query Builder
AST
Semantic Analysis
Query Optimization
Query Planning
ORM
EntityManager
UnitOfWork
IdentityMap
Hydration
Repositories
Schema Model
Migration Model
Business Validation
Authorization
Multitenancy
```

---

# 10. Driver Extension

Una extensión de Driver será un paquete que añade una implementación compatible con el contrato de transporte de `Quantum/Database`.

Conceptualmente:

```php
interface DatabaseDriverExtensionInterface
{
    public function descriptor(): DriverExtensionDescriptor;

    public function register(
        DriverExtensionRegistrationContext $context
    ): void;
}
```

---

# 11. Contrato reducido

El contrato público deberá permanecer pequeño.

No deberá exponer internals innecesarios.

---

# 12. DriverExtensionDescriptor

Cada extensión tendrá metadata estructurada.

Conceptualmente:

```php
final readonly class DriverExtensionDescriptor
{
    public function __construct(
        public ExtensionId $id,
        public ExtensionVersion $version,
        public CompatibilityRequirement $compatibility,
        public array $drivers,
    ) {}
}
```

---

# 13. ExtensionId

Ejemplos:

```text
voltstack.driver.pdo.mysql
voltstack.driver.pdo.pgsql
voltstack.driver.pdo.sqlite
vendor.driver.async.pgsql
```

---

# 14. DriverId

Cada Driver tendrá un identificador independiente.

Ejemplos oficiales iniciales:

```text
pdo.mysql
pdo.pgsql
pdo.sqlite
```

---

# 15. Driver ID estable

`DriverId` será:

- estable;
- case-normalized;
- machine-readable;
- independiente de display name;
- independiente de aliases.

---

# 16. DriverDescriptor

Conceptualmente:

```php
final readonly class DriverDescriptor
{
    public function __construct(
        public DriverId $id,
        public DriverDisplayName $name,
        public DriverVersion $version,
        public DriverFamily $family,
        public DriverTransport $transport,
        public DriverExecutionModel $executionModel,
    ) {}
}
```

---

# 17. DriverFamily

Podrá describir:

```text
PDO
NATIVE
ASYNC
COROUTINE
PROXY
CLOUD
CUSTOM
```

---

# 18. DriverTransport

Podrá representar características del mecanismo utilizado.

Ejemplos conceptuales:

```text
PDO
Native PHP Extension
Socket Client
HTTP Gateway
Database Proxy
Embedded Engine
```

---

# 19. DriverExecutionModel

Conceptualmente:

```text
SYNCHRONOUS
ASYNCHRONOUS
COROUTINE_AWARE
```

---

# 20. No comportamiento implícito

El ExecutionModel no sustituirá capabilities específicas.

Por ejemplo:

```text
ASYNCHRONOUS
```

no implica automáticamente:

```text
query.cancel
stream.concurrent
connection.multiplex
```

---

# 21. DriverInterface

El contrato base deberá ser deliberadamente pequeño.

Ejemplo conceptual:

```php
interface DriverInterface
{
    public function descriptor(): DriverDescriptor;

    public function connect(
        DriverConnectionConfiguration $configuration
    ): NativeConnectionInterface;
}
```

---

# 22. No SQL semantics

`DriverInterface` no tendrá métodos como:

```php
supportsReturning();
supportsJson();
supportsGeneratedColumns();
quoteTableName();
compileUpsert();
```

porque pertenecen a otras abstracciones.

---

# 23. DriverFactory

La creación del Driver será responsabilidad de:

```text
DriverFactory
```

---

# 24. DriverFactoryInterface

Conceptualmente:

```php
interface DriverFactoryInterface
{
    public function create(
        DriverCreationContext $context
    ): DriverInterface;
}
```

---

# 25. Factory lifetime

La Factory deberá ser preferentemente:

```text
stateless
immutable
shareable
```

---

# 26. Factory no abre conexiones

La creación de:

```text
Driver
```

no deberá equivaler a:

```text
connect()
```

---

# 27. Bootstrap invariant

Registrar un Driver:

```text
≠
```

abrir una conexión.

---

# 28. Lazy physical connection

Las conexiones físicas seguirán siendo lazy.

---

# 29. Driver Registry

Componente:

```text
DriverRegistry
```

almacenará definiciones/factories de Drivers.

---

# 30. Registry contents

Podrá almacenar:

```text
DriverId
DriverDescriptor
DriverFactory
ConfigurationSchema
AvailabilityProvider
CapabilityProvider
ErrorAdapterFactory
```

---

# 31. Specialized Registry

No utilizar:

```text
UniversalExtensionRegistry
```

para todo Database.

---

# 32. Correct architecture

```text
DriverRegistry
DialectRegistry
PlatformRegistry
TypeRegistry
CompilerRegistry
OptimizerRuleRegistry
```

permanecerán especializados.

---

# 33. Registry lifecycle

```text
MUTABLE_BOOTSTRAP
      │
      ▼
VALIDATING
      │
      ▼
FROZEN
      │
      ▼
RUNTIME
```

---

# 34. Runtime registration

Por defecto no se permitirá:

```text
registerDriver()
```

durante una request.

---

# 35. Razón

La mutación runtime produciría riesgos para:

```text
concurrency
container compilation
capability caches
configuration caches
persistent workers
determinism
```

---

# 36. Registry freeze

Después del bootstrap:

```text
DriverRegistry::freeze()
```

conceptualmente bloqueará nuevas mutaciones.

---

# 37. Duplicate registration

Registrar dos veces:

```text
pdo.mysql
```

será error.

---

# 38. No last-wins

Prohibido:

```text
last registered driver wins
```

---

# 39. Explicit replacement

Si se permite reemplazo:

```text
replace driver pdo.mysql
```

deberá ser explícito y validado.

---

# 40. Replacement policy

Podrá requerir:

```text
configuration permission
compatible contract version
explicit target
extension priority policy
```

---

# 41. Decoration

Se preferirá decoration sobre replacement cuando el objetivo sea añadir comportamiento transversal.

---

# 42. Driver decoration

Ejemplo:

```text
Base Driver
    │
    ▼
Diagnostic Driver Decorator
    │
    ▼
Native Transport
```

aunque observabilidad de alto nivel seguirá perteneciendo preferentemente a Execution/Connection.

---

# 43. Decoration restrictions

Un decorator no deberá alterar silenciosamente:

```text
transaction semantics
parameter values
result semantics
security guarantees
```

---

# 44. Aliases

Aliases estarán separados de Driver IDs.

Ejemplo:

```text
mysql
    ↓
pdo.mysql
```

---

# 45. AliasRegistry

Podrá existir:

```text
DriverAliasRegistry
```

---

# 46. Alias cycle

Debe rechazarse:

```text
a → b
b → a
```

---

# 47. Alias collision

Un alias no deberá ocultar silenciosamente un Driver ID real.

---

# 48. Driver discovery

VoltStack deberá permitir discovery mediante mecanismos controlados.

---

# 49. Discovery sources

Podrán incluir:

```text
Composer package metadata
VoltStack package manifest
Service provider
compiled extension manifest
explicit configuration
```

---

# 50. Discovery ≠ activation

La existencia de un paquete no implica necesariamente que el Driver esté activo.

---

# 51. Extension lifecycle

```text
DISCOVERED
     │
     ▼
REGISTERED
     │
     ▼
VALIDATED
     │
     ▼
ENABLED
     │
     ▼
ACTIVE
```

---

# 52. Disabled extension

También:

```text
DISCOVERED
     │
     └── DISABLED
```

---

# 53. Discovery performance

Producción podrá utilizar:

```text
CompiledDriverExtensionManifest
```

para evitar scanning repetido.

---

# 54. No filesystem scan per request

Regla:

```text
Driver discovery
=
bootstrap operation
```

no:

```text
request operation
```

---

# 55. Configuration schema

Cada Driver podrá registrar su schema de configuración.

---

# 56. DriverConfigurationSchema

Ejemplo conceptual:

```php
interface DriverConfigurationSchemaInterface
{
    public function driverId(): DriverId;

    public function validate(
        DriverNativeOptions $options
    ): ValidationResult;
}
```

---

# 57. Portable vs native configuration

Mantener:

```text
ConnectionDefinition
├── portable configuration
└── native driver options
```

---

# 58. Portable configuration

Ejemplos:

```text
endpoint
database
credentials
connect timeout
role
TLS requirement
```

---

# 59. Driver-native options

Ejemplos:

```text
PDO attributes
native client flags
driver-specific protocol options
```

---

# 60. No arbitrary native options

Un Driver deberá poder validar:

```text
allowed options
value types
security restrictions
compatibility
```

---

# 61. Unknown options

Default:

```text
unknown driver option
→
configuration error
```

---

# 62. Strict configuration

La configuración de producción deberá ser estricta.

---

# 63. Extension schema registration order

```text
Discover Extension
      │
      ▼
Register Configuration Schema
      │
      ▼
Load/Normalize Config
      │
      ▼
Validate
      │
      ▼
Compile
      │
      ▼
Freeze
```

---

# 64. Driver availability

Un Driver registrado puede no estar disponible.

Ejemplo:

```text
pdo.pgsql registered
PDO pgsql extension missing
```

---

# 65. Registered ≠ Available

Formalmente:

```text
REGISTERED
≠
AVAILABLE
```

---

# 66. DriverAvailabilityProvider

Conceptualmente:

```php
interface DriverAvailabilityProviderInterface
{
    public function check(): DriverAvailability;
}
```

---

# 67. DriverAvailability

Podrá representar:

```text
AVAILABLE
UNAVAILABLE
DEGRADED
UNKNOWN
```

---

# 68. Availability reason

Ejemplos:

```text
missing PHP extension
unsupported PHP version
missing native library
unsupported operating system
invalid runtime
```

---

# 69. Availability checks

Deberán ser preferentemente:

```text
offline
cheap
deterministic
```

cuando sea posible.

---

# 70. Availability ≠ connectivity

Esto:

```text
PDO extension installed
```

no significa:

```text
database server reachable
```

---

# 71. Driver readiness

Distinguir:

```text
Driver Available
Connection Configured
Target Reachable
Target Compatible
```

---

# 72. Driver native dependencies

Un descriptor podrá declarar:

```text
PHP extensions
native libraries
runtime requirements
OS requirements
```

---

# 73. Dependency validation

El bootstrap podrá fallar tempranamente si una conexión configurada requiere un Driver no disponible.

---

# 74. Unused unavailable Driver

La política podrá permitir que un Driver opcional registrado pero no utilizado permanezca unavailable sin impedir el boot.

---

# 75. Required Driver

Si una conexión requerida utiliza:

```text
vendor.custom
```

y éste no está disponible:

```text
BOOT FAILURE
```

será apropiado.

---

# 76. NativeConnectionInterface

Un Driver deberá producir una abstracción neutral:

```text
NativeConnectionInterface
```

---

# 77. Native Connection responsibility

Encapsulará primitivas como:

```text
prepare
execute native operation
begin
commit
rollback
close
native status
```

según contratos segregados.

---

# 78. Interface segregation

No es obligatorio que exista una gigantesca:

```text
NativeConnectionInterface
```

con todas las features.

---

# 79. Capability interfaces

Podrán existir contratos especializados:

```text
NativeStatementPreparerInterface
NativeTransactionInterface
NativeCancellationInterface
NativeHealthInterface
NativeResetInterface
```

---

# 80. Avoid interface explosion

Sólo se crearán contratos cuando exista una frontera real y estable.

---

# 81. Native handle

El native handle concreto podría ser:

```text
PDO
PgSql\Connection
mysqli
SQLite3
custom client
```

pero no escapará normalmente al resto del framework.

---

# 82. NativeConnectionAdapter

Un Driver externo podrá implementar:

```text
VendorNativeConnectionAdapter
```

---

# 83. NativeStatementInterface

Abstraerá:

```text
parameter binding
execution
statement close
result access
```

sin semántica ORM.

---

# 84. NativeResultInterface

Abstraerá el resultado nativo antes de:

```text
Result
```

de VoltStack.

---

# 85. Result architecture

```text
Vendor Result
      │
      ▼
NativeResultInterface
      │
      ▼
Result Adapter
      │
      ▼
VoltStack Result
```

---

# 86. No entity hydration

Native Result nunca deberá conocer:

```text
Entity
Hydrator
Repository
EntityManager
```

---

# 87. Parameter binding

El Driver será responsable de traducir valores ya preparados por capas superiores al mecanismo nativo.

---

# 88. Correct binding pipeline

```text
Application Value
      │
      ▼
VoltStack Type System
      │
      ▼
Database Representation
      │
      ▼
Compiled Binding
      │
      ▼
Driver Binding Adapter
      │
      ▼
Native Statement
```

---

# 89. Driver does not perform domain casting

Incorrecto:

```text
Driver converts UserStatus enum
```

---

# 90. Correct

```text
Type System
→ database representation
→ Driver
```

---

# 91. Prepared statements

Un Driver podrá proporcionar:

```text
native prepared statements
emulated prepared statements
client-side prepared operations
```

---

# 92. Capability declaration

Ejemplos:

```text
driver.statement.prepare
driver.statement.prepare.native
driver.statement.prepare.emulated
```

---

# 93. Security policy

El framework podrá requerir determinadas capabilities de preparación según configuración.

---

# 94. No fake native prepare

Un Driver no deberá anunciar:

```text
native prepared statements
```

si internamente sólo interpola SQL.

---

# 95. Statement caching

Si un Driver implementa statement caching, deberá distinguirse de:

```text
CompiledQueryCache
```

---

# 96. Cache layers

```text
CompiledQueryCache
≠
PreparedStatementCache
≠
DatabaseServerPlanCache
```

---

# 97. Statement cache lifetime

Podrá ser:

```text
physical connection scoped
```

según el cliente.

---

# 98. Reset compatibility

Statement cache deberá integrarse con:

```text
Connection State & Reset System
```

---

# 99. Native reset

Un Driver podrá exponer:

```text
driver.connection.native_reset
```

---

# 100. Native reset contract

Conceptualmente:

```php
interface NativeConnectionResetInterface
{
    public function reset(
        NativeResetContext $context
    ): NativeResetResult;
}
```

---

# 101. Native reset semantics

No basta con que exista un método llamado:

```text
reset()
```

La extensión deberá declarar qué estado realmente limpia.

---

# 102. Reset capability descriptor

Podrá describir:

```text
transactions
session variables
roles
schemas
temporary objects
prepared statements
locks
driver buffers
```

---

# 103. Reset verification

El Connection Reset System decidirá si el native reset es suficiente.

---

# 104. Driver cannot self-declare reusable

El Driver no decidirá unilateralmente:

```text
this connection is safe for tenant B
```

---

# 105. Correct flow

```text
Driver Native Reset Capability
          │
          ▼
Platform Reset Semantics
          │
          ▼
ConnectionResetPlanner
          │
          ▼
Reuse / Additional Reset / Discard
```

---

# 106. Error translation

Cada Driver deberá adaptar errores nativos.

---

# 107. Error adapter

Conceptualmente:

```php
interface NativeErrorAdapterInterface
{
    public function adapt(
        Throwable $error
    ): NativeDatabaseError;
}
```

---

# 108. NativeDatabaseError

Podrá contener:

```text
SQLSTATE
native code
extended code
safe message
native category hints
connection status hints
```

---

# 109. No sensitive information

Nunca incluir:

```text
password
full DSN with credentials
access token
secret
raw sensitive bindings
```

---

# 110. Failure classification

Driver adapta errores.

Otra capa clasifica semántica.

```text
Native Error
     │
     ▼
Driver Error Adapter
     │
     ▼
NativeDatabaseError
     │
     ▼
Platform/Failure Classifier
     │
     ▼
DatabaseFailure
```

---

# 111. Driver does not own retry

Regla:

```text
Driver
≠
Retry Engine
```

---

# 112. Driver connection retry

Incluso connect retries deberán estar coordinados por:

```text
Resilience System
```

cuando impliquen política.

---

# 113. Capability provider

Un Driver Extension podrá registrar:

```text
DriverCapabilityProvider
```

---

# 114. Driver capability scope

Sólo capabilities propias del mecanismo de transporte/cliente.

Ejemplos:

```text
driver.async
driver.cancel
driver.streaming
driver.native_prepare
driver.multiple_results
driver.native_reset
driver.ping
```

---

# 115. No Platform capabilities in Driver

Incorrecto:

```text
driver.supportsReturning()
```

---

# 116. Correct

```text
SqlitePlatform:
    query.returning.insert

PdoSqliteDriver:
    driver.statement.prepare
```

---

# 117. Effective capability

Algunas features requieren ambas capas.

```text
Platform Capability
+
Driver Primitive
=
Effective Capability
```

---

# 118. Example

Una feature podría requerir:

```text
Platform:
    query.returning.insert = native

Driver:
    result.rows = supported
```

---

# 119. Capability provenance

El sistema deberá poder explicar:

```text
capability source
driver contribution
platform contribution
runtime restriction
configuration restriction
```

---

# 120. Capability honesty

Una extensión de terceros deberá declarar capabilities conservadoramente.

---

# 121. Capability conformance

VoltStack podrá ejecutar tests que validen que las capabilities anunciadas realmente funcionan.

---

# 122. Driver Conformance Suite

Todo Driver público deberá pasar:

```text
DatabaseDriverConformanceSuite
```

---

# 123. Conformance categories

Incluirá:

```text
registration
availability
connection
statement
binding
results
transactions
errors
cleanup
reset primitives
capabilities
persistent runtime
concurrency
resource ownership
security
```

---

# 124. Registration tests

Validarán:

```text
unique DriverId
valid descriptor
valid factory
valid aliases
valid dependencies
```

---

# 125. Connection tests

Validarán:

```text
connect
failed connect
close
double close behavior
resource cleanup
```

---

# 126. Statement tests

Validarán:

```text
prepare
bind
execute
close
failure handling
```

---

# 127. Binding tests

Incluirán:

```text
null
integer
float
string
binary
large values
unicode
```

---

# 128. Type tests

El Driver Conformance Suite no probará domain types directamente.

Eso corresponde al Type System.

---

# 129. Result tests

Validarán:

```text
row retrieval
empty results
column metadata where supported
buffered results
streaming results where declared
resource closure
```

---

# 130. Transaction primitive tests

Validarán:

```text
begin
commit
rollback
failure state
```

sin probar políticas avanzadas del TransactionManager.

---

# 131. Error tests

Validarán que errores nativos puedan convertirse en:

```text
NativeDatabaseError
```

sin perder códigos relevantes.

---

# 132. Capability tests

Cada capability declarada deberá tener un test de comportamiento cuando sea razonablemente verificable.

---

# 133. Persistent runtime tests

Deberán simular:

```text
Scope A
→ use
→ cleanup
→ Scope B
```

---

# 134. State leakage tests

Deberán verificar ausencia de fugas del Driver como:

```text
native buffers
statement state
pending results
current transaction primitive
```

---

# 135. Connection state tests

La limpieza completa de estado de sesión se probará junto al Platform/Reset conformance suite.

---

# 136. Concurrency tests

Cuando un Driver declare concurrency/async/coroutine capabilities deberá pasar suites adicionales.

---

# 137. Concurrency safety descriptor

Podrá declarar:

```text
NOT_THREAD_SAFE
CONNECTION_EXCLUSIVE
COROUTINE_SAFE
MULTIPLEX_SAFE
```

pero siempre respaldado por capabilities más concretas.

---

# 138. Default safety

Default:

```text
CONNECTION_EXCLUSIVE
```

---

# 139. Multiplexing

No se permitirá compartir una conexión física concurrentemente sólo porque el cliente lo permita técnicamente.

---

# 140. Multiplex capability

Requerirá:

```text
driver.connection.multiplex
```

más garantías de:

```text
transaction isolation
result ownership
statement isolation
session state
cancellation
```

---

# 141. Persistent runtime safety

Todo Driver oficial deberá funcionar correctamente bajo:

```text
FrankenPHP
```

como runtime de referencia.

---

# 142. FrankenPHP requirement

Un Driver no deberá almacenar en singleton:

```text
current transaction
current statement
current result
current tenant
current request
```

---

# 143. RoadRunner

El mismo contrato deberá funcionar sin cambios conceptuales.

---

# 144. OpenSwoole

Drivers coroutine-aware podrán añadir optimizaciones, pero deberán respetar:

```text
ExecutionScope isolation
ConnectionLease ownership
DatabaseContext isolation
```

---

# 145. Runtime-neutral core

Driver Extension System no deberá depender directamente de:

```text
FrankenPHP classes
RoadRunner classes
OpenSwoole classes
```

---

# 146. Runtime integration

Las optimizaciones runtime-specific deberán entrar mediante:

```text
Runtime Adapter
+
Driver Capability
```

---

# 147. Async Drivers

El sistema deberá permitir Drivers asíncronos futuros.

---

# 148. No premature async pollution

El contrato síncrono v1 no deberá deformarse para simular async.

---

# 149. Async extension contract

Podrá existir posteriormente:

```text
AsyncDriverInterface
AsyncNativeConnectionInterface
AsyncNativeStatementInterface
```

como capability/contract especializado.

---

# 150. Sync and async coexistence

```text
Driver Extension System
      │
      ├── Sync Driver
      └── Async Driver
```

sin duplicar:

```text
Query AST
Semantic Engine
ORM
Schema Model
```

---

# 151. Async execution

La diferencia deberá aparecer principalmente en:

```text
Execution
Connection Resource
Driver
Runtime
```

---

# 152. Coroutine Drivers

Un Driver OpenSwoole-aware podrá ser:

```text
COROUTINE_AWARE
```

pero no contaminará Query/ORM.

---

# 153. Cloud connectors

El sistema deberá permitir Drivers especializados para:

```text
database proxies
managed database gateways
cloud authentication
special transport layers
```

---

# 154. Cloud auth

La autenticación cloud podrá integrarse mediante:

```text
CredentialProvider
```

no mediante credenciales hardcodeadas en Driver config.

---

# 155. Credential lifecycle

```text
CredentialReference
      │
      ▼
CredentialResolver
      │
      ▼
Ephemeral Credential
      │
      ▼
Driver connect()
```

---

# 156. Credential rotation

El Driver deberá aceptar nuevas credenciales para nuevas conexiones sin requerir mutación global.

---

# 157. No credential cache by default

Un Driver singleton no deberá almacenar:

```text
current password
current token
tenant credential
```

---

# 158. Credential ownership

Las credenciales resueltas deberán tener lifetime mínimo necesario.

---

# 159. Driver package structure

Un paquete externo podrá utilizar:

```text
src/
├── DriverExtension.php
├── Driver/
│   ├── VendorDriver.php
│   ├── VendorDriverFactory.php
│   ├── VendorDriverDescriptor.php
│   └── VendorAvailabilityProvider.php
│
├── Native/
│   ├── VendorNativeConnection.php
│   ├── VendorNativeStatement.php
│   └── VendorNativeResult.php
│
├── Configuration/
│   └── VendorDriverConfigurationSchema.php
│
├── Capability/
│   └── VendorDriverCapabilityProvider.php
│
├── Error/
│   └── VendorNativeErrorAdapter.php
│
└── Testing/
    └── VendorDriverConformanceTest.php
```

---

# 160. Combined database package

Cuando sea necesario, un paquete podrá incluir:

```text
Driver/
Dialect/
Platform/
```

---

# 161. Example combined package

```text
voltstack/database-sqlserver

src/
├── Driver/
├── Dialect/
├── Platform/
├── Schema/
├── Capability/
└── Extension/
```

---

# 162. Registration independence

Aunque estén en el mismo paquete:

```text
registerDriver()
registerDialect()
registerPlatform()
```

serán operaciones independientes.

---

# 163. Compatibility declaration

El paquete deberá declarar:

```text
VoltStack Database API version
Driver Contract version
supported PHP versions
required extensions
optional integrations
```

---

# 164. Contract version

Podrá existir:

```text
DatabaseDriverContractVersion
```

---

# 165. Semantic versioning

Cambios incompatibles en contratos públicos deberán respetar versionado.

---

# 166. Internal APIs

Un Driver externo no deberá depender de namespaces marcados:

```text
Internal
Implementation
```

---

# 167. Public extension surface

Sólo deberá depender de:

```text
Contract
Extension
Public API
stable Value Objects
```

---

# 168. Internal dependency detection

Architecture tests podrán detectar imports prohibidos.

---

# 169. Extension compatibility validator

Componente:

```text
DriverExtensionCompatibilityValidator
```

---

# 170. Validation

Comprobará:

```text
contract version
PHP version
required extensions
Driver IDs
dependencies
conflicts
execution model
runtime requirements
```

---

# 171. Extension dependencies

Una extensión podrá requerir otra extensión.

---

# 172. Dependency graph

```text
Driver Extension A
       │
       ▼
Driver Extension B
```

deberá formar un DAG.

---

# 173. Cycles

Rechazar:

```text
A → B
B → A
```

---

# 174. Optional dependencies

También podrán declararse:

```text
optional dependency
```

para integración adicional.

---

# 175. Conflicts

Una extensión podrá declarar:

```text
conflictsWith
```

---

# 176. Conflict handling

Conflictos deberán fallar durante bootstrap, no producir comportamiento no determinista en runtime.

---

# 177. Extension ordering

Cuando sea necesario:

```text
requires
before
after
```

podrán determinar orden.

---

# 178. No arbitrary priority numbers

Preferir relaciones explícitas sobre:

```text
priority = 999999
```

cuando sea posible.

---

# 179. Driver providers

Un package service provider podrá registrar la extensión.

Conceptualmente:

```php
final class VendorDatabaseServiceProvider
{
    public function register(
        DatabaseExtensionRegistry $extensions
    ): void {
        // register descriptor/factory/schema...
    }
}
```

---

# 180. No container lookup inside Driver

Incorrecto:

```php
$container->get(Config::class);
```

desde `VendorDriver`.

---

# 181. Correct dependency injection

```text
Composition Root
      │
      ▼
DriverFactory
      │
      ▼
explicit dependencies
```

---

# 182. DriverFactory context

Podrá recibir sólo dependencias legítimas:

```text
Clock
native client factory
safe diagnostics
configuration
```

según necesidad.

---

# 183. No Application services

Driver Factory no deberá recibir:

```text
Controller
AuthManager
CurrentUser
EntityManager
Repository
```

---

# 184. Security boundary

Un Driver es código confiable con acceso potencial a:

```text
database credentials
database contents
network
filesystem
```

---

# 185. No security sandbox claim

El Extension System es:

```text
architectural isolation
```

no:

```text
security sandbox
```

---

# 186. Trusted packages

Drivers deberán considerarse paquetes privilegiados.

---

# 187. Package verification

Herramientas futuras podrán mostrar:

```text
package
publisher
version
permissions/capabilities
native dependencies
```

pero no sustituirán revisión de confianza.

---

# 188. Secrets

Drivers deberán cumplir:

```text
never log secrets
never include secrets in fingerprints
never expose credentials through diagnostics
```

---

# 189. DSN

Si un Driver construye un DSN nativo, éste no deberá ser mostrado sin redacción.

---

# 190. Sensitive native errors

Error adapters deberán sanitizar mensajes cuando el cliente incluya información sensible.

---

# 191. Native client escape hatch

El Driver podrá permitir acceso avanzado al cliente nativo mediante una API controlada.

---

# 192. Preferred API

Conceptualmente:

```php
$connection->withNativeConnection(
    function (object $native): void {
        // advanced operation
    }
);
```

---

# 193. Native access consequences

Acceso mutable podrá:

```text
mark state dirty
mark state unknown
require strict reset
force discard
```

---

# 194. Extension cannot bypass reset tracking

Un Driver externo deberá cooperar con:

```text
Connection State & Reset System
```

---

# 195. Native mutation declaration

Una futura API avanzada podrá permitir:

```text
NativeMutationDescriptor
```

para declarar qué estado fue modificado.

---

# 196. Unknown mutation

Default seguro:

```text
unknown native mutation
→
state UNKNOWN
```

---

# 197. Unknown state reuse

```text
UNKNOWN
→
STRICT RESET
or
DISCARD
```

nunca:

```text
UNKNOWN
→
IDLE
```

sin validación.

---

# 198. Driver lifecycle hooks

Las extensiones podrán registrar hooks limitados.

---

# 199. Valid hooks

Ejemplos:

```text
after native connection created
before native connection close
native error adaptation
availability discovery
capability discovery
```

---

# 200. Invalid hooks

No deberán existir hooks de Driver para:

```text
entity persisted
repository resolved
controller invoked
authorization decision
```

---

# 201. Connection initialization

Driver podrá realizar inicialización estrictamente necesaria para el cliente nativo.

---

# 202. Platform session initialization

Configuración semántica de sesión pertenece a:

```text
Platform Session Initializer
```

---

# 203. Separation

```text
Driver Initialization
    │
    └── native client setup

Platform Initialization
    │
    └── database session semantics
```

---

# 204. Example

Driver puede configurar:

```text
native client mode
transport flags
```

Platform puede configurar:

```text
timezone
schema/search_path
foreign-key enforcement
SQL mode
```

según motor.

---

# 205. Driver-specific TLS

La implementación concreta de TLS podrá pertenecer al Driver.

---

# 206. TLS semantic policy

La exigencia:

```text
TLS required
```

proviene de configuración/política superior.

---

# 207. Correct flow

```text
Security Policy
      │
      ▼
ConnectionConfiguration
      │
      ▼
Driver TLS Adapter
      │
      ▼
Native Client
```

---

# 208. No downgrade

Si:

```text
TLS = REQUIRED
```

el Driver no podrá conectarse sin TLS silenciosamente.

---

# 209. TLS capability

Podrá declarar:

```text
driver.transport.tls
```

---

# 210. TLS verification

Podrán existir capabilities más específicas:

```text
driver.transport.tls.verify_peer
driver.transport.tls.client_certificate
```

---

# 211. Cancellation

Un Driver podrá registrar:

```text
driver.query.cancel
```

---

# 212. Cancellation semantics

La capability deberá describir si:

```text
cancel affects statement
cancel affects connection
cancel leaves protocol usable
```

---

# 213. Cancellation state

Después de cancelación:

```text
ConnectionStateTracker
```

deberá saber si el recurso continúa siendo reutilizable.

---

# 214. Cancellation uncertainty

Si no puede garantizarse:

```text
DISCARD
```

---

# 215. Streaming

Capability:

```text
driver.result.streaming
```

---

# 216. Streaming ownership

Un streaming result deberá mantener:

```text
ConnectionLease
```

hasta cierre.

---

# 217. Driver streaming conformance

La extensión deberá demostrar:

```text
bounded memory behavior
correct close
failure cleanup
lease compatibility
```

---

# 218. Multiple result sets

Capability:

```text
driver.result.multiple_sets
```

---

# 219. Result cleanup

Todos los result sets deberán consumirse/cerrarse cuando el protocolo lo requiera antes de reutilizar la conexión.

---

# 220. Driver ping

Capability:

```text
driver.connection.ping
```

---

# 221. Ping semantics

`ping()` significa:

```text
transport/database reachability check
```

no:

```text
connection is clean
```

---

# 222. Health ≠ reset

Regla:

```text
Healthy Connection
≠
Clean Connection
```

---

# 223. Reset ≠ ping

Igualmente:

```text
Ping Success
≠
Session Baseline Verified
```

---

# 224. Generated key retrieval

Driver podrá proporcionar una primitive:

```text
driver.generated_key
```

---

# 225. Platform integration

El Persistence Planner determinará cuándo esa primitive es semánticamente válida.

---

# 226. Generated key ≠ RETURNING

Mantener separados.

---

# 227. Server metadata discovery

El Driver podrá proporcionar primitives para obtener:

```text
server version
client version
connection metadata
```

---

# 228. Discovery ownership

Platform Discovery utilizará esas primitives.

---

# 229. Driver does not interpret Platform version

El Driver puede reportar:

```text
native server version string
```

mientras Platform la interpreta.

---

# 230. Client version

Debe distinguirse:

```text
Driver Version
Native Client Version
Database Engine Version
```

---

# 231. Driver diagnostics

Podrá mostrar:

```text
Driver ID
Driver Version
Family
Execution Model
Availability
Native Client
Native Client Version
Declared Capabilities
```

---

# 232. No credentials in diagnostics

Obligatorio.

---

# 233. CLI

Podrá existir:

```text
php volt database:drivers
```

---

# 234. Example output

```text
DRIVER       STATUS       FAMILY   EXECUTION
pdo.mysql    available    PDO      sync
pdo.pgsql    available    PDO      sync
pdo.sqlite   available    PDO      sync
```

---

# 235. Driver details

```text
php volt database:driver pdo.pgsql
```

podrá mostrar:

```text
descriptor
availability
aliases
native dependencies
capabilities
extension provider
compatibility
```

---

# 236. Explain resolution

```text
php volt database:connection:explain default
```

podrá incluir:

```text
requested driver: pgsql
alias resolved: pdo.pgsql
extension: voltstack/database
availability: available
```

---

# 237. Extension diagnostics

Podrá existir:

```text
php volt database:extensions
```

---

# 238. Failure diagnostics

Si un Driver no está disponible:

```text
Connection "analytics" requires driver "pdo.pgsql".

Driver registered: yes
Driver available: no
Reason: required PHP extension is unavailable.
```

---

# 239. Developer experience

Instalar un Driver externo deberá ser simple.

Conceptualmente:

```text
composer require vendor/voltstack-database-driver
```

---

# 240. Auto-discovery

Cuando sea seguro:

```text
Composer
   │
   ▼
VoltStack Package Discovery
   │
   ▼
Driver Extension Registration
```

---

# 241. Explicit enable

Entornos estrictos podrán requerir:

```text
database.extensions.allow
```

o configuración equivalente.

---

# 242. Extension allowlist

Producción podrá utilizar:

```text
allowed database extensions
```

para evitar activaciones inesperadas.

---

# 243. Extension denylist

Podrá existir también una política de bloqueo, aunque allowlist será más determinista en entornos altamente controlados.

---

# 244. Package removal

Eliminar una extensión requerida deberá producir error de configuración claro durante boot.

---

# 245. No silent fallback

Si:

```text
driver = vendor.special
```

no está disponible, VoltStack no cambiará silenciosamente a:

```text
pdo.mysql
```

---

# 246. Explicit fallback

Cualquier fallback deberá ser declarado mediante una política superior.

---

# 247. Driver selection

DriverResolver resolverá:

```text
requested DriverId / alias
```

no seleccionará arbitrariamente “el mejor Driver” salvo política explícita.

---

# 248. Automatic selection

Una futura configuración:

```text
driver = auto
```

podrá existir, pero deberá tener resolución determinista y explicable.

---

# 249. Example auto policy

```text
Target Platform: PostgreSQL
       │
       ▼
Available Compatible Drivers
       │
       ├── native.pgsql
       └── pdo.pgsql
               │
               ▼
          Selection Policy
```

---

# 250. No hidden performance policy

VoltStack no deberá cambiar de Driver automáticamente entre deployments sin diagnóstico.

---

# 251. Driver preference policy

Podrá existir:

```text
prefer:
  - native.pgsql
  - pdo.pgsql
```

---

# 252. Resolution trace

El sistema podrá explicar:

```text
selected: native.pgsql
reason: first available compatible preferred driver
```

---

# 253. Driver-platform compatibility

No todo Driver será compatible con cualquier Platform.

---

# 254. Compatibility mapping

Podrá existir:

```text
DriverPlatformCompatibility
```

---

# 255. Example

```text
pdo.mysql
    ├── mysql
    └── mariadb
```

---

# 256. Compatibility does not merge Platforms

MySQL y MariaDB seguirán siendo plataformas distintas aunque compartan transporte.

---

# 257. Driver-dialect compatibility

También podrá validarse:

```text
Driver
+
Dialect
+
Platform
```

antes de construir el target efectivo.

---

# 258. Database target resolution

```text
ConnectionDefinition
      │
      ▼
Driver Resolution
      │
      ▼
Platform Resolution
      │
      ▼
Dialect Resolution
      │
      ▼
Compatibility Validation
      │
      ▼
ResolvedDatabaseTarget
```

---

# 259. ResolvedDatabaseTarget

Conceptualmente:

```php
final readonly class ResolvedDatabaseTarget
{
    public function __construct(
        public DriverDescriptor $driver,
        public PlatformDescriptor $platform,
        public DialectDescriptor $dialect,
        public EffectiveCapabilitySet $capabilities,
    ) {}
}
```

---

# 260. Target cache

Podrá cachearse usando fingerprints estables.

---

# 261. No connection object in target descriptor

`ResolvedDatabaseTarget` no deberá almacenar necesariamente una physical connection.

---

# 262. Runtime discovery

Información que requiera conexión podrá completar el target posteriormente.

---

# 263. Two-phase resolution

```text
Static Target Resolution
       │
       ▼
Partial Capabilities
       │
       ▼
Physical Connection
       │
       ▼
Runtime Discovery
       │
       ▼
Effective Target
```

---

# 264. Lazy runtime discovery

No abrir conexión durante bootstrap sólo para completar capabilities que no sean necesarias todavía.

---

# 265. Capability requirement trigger

La discovery podrá activarse cuando:

```text
query
schema operation
migration
runtime validation
```

requiera una capability desconocida.

---

# 266. Capability cache

El resultado podrá cachearse según:

```text
EngineFingerprint
DriverFingerprint
ConfigurationGeneration
```

---

# 267. Extension fingerprint

Podrá existir:

```text
DriverExtensionGraphFingerprint
```

---

# 268. Compiled cache compatibility

Caches relevantes deberán incluir la versión/fingerprint de extensiones que afecten su resultado.

---

# 269. Driver extension upgrade

Actualizar un Driver podrá invalidar:

```text
availability cache
driver capability cache
relevant compiled execution metadata
```

---

# 270. No unnecessary Query AST invalidation

AST semántico no deberá depender de la versión concreta del Driver.

---

# 271. Query portability

Esto permite:

```text
same AST
+
different Driver
=
different execution adapter
```

sin reconstruir intención de la aplicación.

---

# 272. Driver package testing

Todo paquete oficial deberá ejecutar:

```text
Unit Tests
Driver Conformance Tests
Integration Tests
Persistent Runtime Tests
Platform Compatibility Tests
```

---

# 273. Third-party certification

VoltStack podrá definir niveles como:

```text
COMMUNITY
CONFORMANT
OFFICIAL
```

para Drivers.

---

# 274. Conformant

Significará que una versión determinada pasó la suite correspondiente.

---

# 275. Official

Significará además mantenimiento/integración oficial de VoltStack.

---

# 276. No trust implication

`CONFORMANT` no implica auditoría completa de seguridad del paquete.

---

# 277. Reference implementation

Los Drivers:

```text
pdo.mysql
pdo.pgsql
pdo.sqlite
```

servirán como implementaciones de referencia.

---

# 278. Shared PDO infrastructure

Podrá existir:

```text
Driver/Pdo/
├── PdoDriverSupport
├── PdoNativeConnection
├── PdoNativeStatement
├── PdoNativeResult
└── PdoErrorAdapter
```

---

# 279. Avoid giant PdoDriver

No crear:

```text
PdoDriver
└── switch ($database)
```

con toda la lógica vendor-specific.

---

# 280. Correct PDO architecture

```text
Shared PDO Infrastructure
        │
        ├── MySQL Adapter
        ├── PostgreSQL Adapter
        └── SQLite Adapter
```

---

# 281. Shared code policy

Compartir únicamente:

```text
actual common transport behavior
```

no semántica aparentemente similar.

---

# 282. Inheritance policy

Preferir:

```text
composition
```

sobre jerarquías profundas.

---

# 283. Example

Preferir:

```text
PdoNativeConnection
+
MysqlPdoErrorAdapter
+
MysqlPdoConfigurationAdapter
```

sobre:

```text
AbstractPdoDriver
  ↓
AbstractRelationalPdoDriver
  ↓
AbstractMysqlCompatibleDriver
  ↓
MysqlDriver
```

---

# 284. Extension API stability

Los puntos de extensión deberán estar explícitamente clasificados.

---

# 285. Stability classes

```text
STABLE
EXTENSION
EXPERIMENTAL
INTERNAL
```

---

# 286. Driver contracts

Los contratos necesarios para Drivers externos deberán ser:

```text
EXTENSION
```

o:

```text
STABLE
```

según compromiso de compatibilidad.

---

# 287. Native implementation helpers

Helpers internos podrán ser:

```text
INTERNAL
```

---

# 288. Do not extend internals

Documentación deberá advertir:

```text
Internal classes are not extension points.
```

---

# 289. Extension point descriptor

Cada punto importante podrá documentar:

```text
contract
lifetime
thread/concurrency requirements
allowed dependencies
failure semantics
stability
```

---

# 290. Example

```text
Extension Point:
DriverFactoryInterface

Lifetime:
Application

State:
Stateless

Allowed dependencies:
Driver contracts
Configuration value objects
Native client factory

Forbidden:
EntityManager
Request
Current tenant
Repository

Stability:
EXTENSION
```

---

# 291. Resource ownership contract

Toda extensión deberá respetar:

```text
who creates
who owns
who closes
who resets
who discards
```

---

# 292. Ownership table

| Recurso | Creador | Owner principal | Cleanup |
|---|---|---|---|
| Driver | DriverFactory | Container | lifecycle |
| Native Connection | Driver | Pool/Lease infrastructure | close/reset |
| Native Statement | Native Connection | operation | close |
| Native Result | Native Statement | Result/operation | close |
| Connection Lease | Resource Provider | execution owner | release/discard |

---

# 293. Ownership violation

Un Driver no deberá mantener globalmente todas las conexiones que ha creado.

---

# 294. Resource leak

Conformance Suite deberá detectar recursos no liberados cuando sea posible.

---

# 295. Exception boundaries

Una extensión podrá lanzar excepciones propias internamente, pero deberán normalizarse antes de cruzar fronteras públicas cuando corresponda.

---

# 296. Driver exception hierarchy

Conceptualmente:

```text
DriverException
├── DriverUnavailableException
├── DriverConfigurationException
├── DriverConnectionException
├── DriverProtocolException
├── DriverOperationException
└── DriverExtensionException
```

---

# 297. Native exceptions

No deberán escapar indiscriminadamente:

```text
PDOException
VendorClientException
```

hasta Application Layer.

---

# 298. Original exception

Podrá conservarse como:

```text
previous exception
```

para diagnóstico controlado.

---

# 299. Debug mode

Podrá mostrar más detalles sanitizados.

---

# 300. Production mode

Deberá evitar leakage de:

```text
host internals
filesystem paths
credentials
tokens
raw SQL bindings
```

---

# 301. Failure policy

Fallos durante registration de un Driver requerido:

```text
fail bootstrap
```

---

# 302. Optional Driver failure

Una extensión opcional no utilizada podrá quedar:

```text
DISABLED / UNAVAILABLE
```

según policy.

---

# 303. Observer failure

Un observer diagnóstico no deberá normalmente inutilizar el Driver.

---

# 304. Security-critical hook failure

Un hook requerido para:

```text
TLS
credential handling
state reset
```

sí deberá fallar de forma segura.

---

# 305. Fail closed

Regla:

```text
uncertain security state
→
do not connect / do not reuse
```

---

# 306. Driver extension boot sequence

```text
Package Discovery
      │
      ▼
Extension Descriptor Load
      │
      ▼
Contract Compatibility
      │
      ▼
Dependency Graph
      │
      ▼
Configuration Schema Registration
      │
      ▼
Driver Registration
      │
      ▼
Alias Registration
      │
      ▼
Capability Provider Registration
      │
      ▼
Availability Validation
      │
      ▼
Registry Validation
      │
      ▼
Registry Freeze
      │
      ▼
Compiled Manifest
```

---

# 307. Runtime sequence

```text
Connection Requirement
      │
      ▼
ConnectionDefinition
      │
      ▼
DriverResolver
      │
      ▼
Frozen DriverRegistry
      │
      ▼
DriverFactory
      │
      ▼
Driver
      │
      ▼
connect()
      │
      ▼
NativeConnection
```

---

# 308. No extension boot during query

Prohibido:

```text
query execution
→ discover driver packages
```

---

# 309. No Composer access in hot path

Composer metadata deberá procesarse en bootstrap/cache compilation.

---

# 310. Production optimization

Producción podrá generar:

```text
CompiledDriverRegistry
```

---

# 311. CompiledDriverRegistry

Podrá contener:

```text
normalized Driver IDs
factory service references
aliases
configuration schema references
capability provider references
availability metadata
extension fingerprints
```

---

# 312. No live connection serialization

Nunca cachear:

```text
PDO
socket
native connection
statement
result
credential
```

en compiled registry.

---

# 313. Development mode

Podrá proporcionar:

```text
extension diagnostics
duplicate detection
compatibility warnings
architecture warnings
```

---

# 314. Production mode

Optimizará:

```text
discovery
validation
registry lookup
factory resolution
```

---

# 315. Registry lookup complexity

La resolución por DriverId deberá ser aproximadamente:

```text
O(1)
```

mediante mapas compilados.

---

# 316. No reflection hot path

Evitar reflection durante cada:

```text
connect
prepare
execute
```

---

# 317. Factory pre-resolution

Factories podrán precompilarse mediante Container.

---

# 318. No dynamic container lookup

DriverResolver no deberá utilizar service locator arbitrario.

---

# 319. Explicit service references

Composition Root deberá construir el mapa:

```text
DriverId
→
DriverFactory
```

---

# 320. Testing custom Driver

Un desarrollador deberá poder ejecutar conceptualmente:

```text
vendor/bin/phpunit \
    --testsuite voltstack-database-driver-conformance
```

---

# 321. Conformance fixture

VoltStack podrá proporcionar:

```text
AbstractDatabaseDriverConformanceTest
```

o composición equivalente.

---

# 322. Prefer composition in tests too

Evitar exigir herencia profunda para integrar una suite.

---

# 323. DriverTestHarness

Podrá existir:

```text
DriverTestHarness
```

configurado por el paquete.

---

# 324. Harness requirements

El paquete suministrará:

```text
DriverFactory
Test Connection Configuration
Capability Expectations
Cleanup Strategy
```

---

# 325. Capability expectation

Ejemplo:

```text
driver.statement.prepare = true
driver.result.streaming = false
driver.connection.native_reset = true
```

---

# 326. Behavioral verification

La suite comparará:

```text
declared capability
vs
observed behavior
```

---

# 327. Platform conformance

Si el paquete también proporciona Platform:

```text
DatabasePlatformConformanceSuite
```

se ejecutará por separado.

---

# 328. Dialect conformance

Igualmente:

```text
DatabaseDialectConformanceSuite
```

---

# 329. Combined conformance

Finalmente:

```text
Driver
+
Dialect
+
Platform
```

podrán someterse a:

```text
DatabaseTargetIntegrationSuite
```

---

# 330. Architecture conformance

También deberá verificar:

```text
Driver package does not import ORM internals
Driver does not compile SQL
Driver does not implement retry policy
Driver does not store request state
```

---

# 331. Custom database onboarding

Agregar un motor nuevo debería seguir:

```text
1. Determine transport
2. Implement Driver
3. Implement/choose Dialect
4. Implement/choose Platform
5. Declare capabilities
6. Add schema adapters
7. Add error classification
8. Add reset semantics
9. Run conformance suites
10. Package extension
```

---

# 332. Driver-only onboarding

Si el motor ya está soportado y sólo se añade un cliente nuevo:

```text
1. Implement Driver
2. Declare driver capabilities
3. Map existing Platform
4. Map existing Dialect
5. Add error/native adapters
6. Run driver conformance
7. Run target integration
```

---

# 333. Example future native PostgreSQL Driver

```text
native.pgsql
      │
      ├── NativePgsqlDriver
      ├── NativePgsqlConnection
      ├── NativePgsqlStatement
      ├── NativePgsqlResult
      ├── PgsqlNativeErrorAdapter
      └── PgsqlDriverCapabilityProvider
```

reutilizando:

```text
PostgresqlDialect
PostgresqlPlatform
```

---

# 334. Example async PostgreSQL Driver

```text
async.pgsql
      │
      ├── AsyncPgsqlDriver
      ├── AsyncNativeConnection
      ├── AsyncNativeStatement
      └── AsyncCapabilityProvider
```

sin modificar:

```text
ORM metadata
Query AST
Schema Model
```

---

# 335. Example SQLite3 Driver

Un futuro:

```text
native.sqlite3
```

podría utilizar la extensión:

```text
SQLite3
```

en vez de PDO.

Seguiría reutilizando:

```text
SqliteDialect
SqlitePlatform
```

cuando sea semánticamente compatible.

---

# 336. Example MySQLi Driver

Un futuro:

```text
native.mysqli
```

podría coexistir con:

```text
pdo.mysql
```

---

# 337. Driver selection example

```text
Platform: mysql

Available Drivers:
├── pdo.mysql
└── native.mysqli

Policy:
prefer native.mysqli

Resolved:
native.mysqli
+
MysqlDialect
+
MysqlPlatform
```

---

# 338. No ORM difference

La aplicación no deberá cambiar:

```php
User::query()->where('active', true)->get();
```

por cambiar de:

```text
pdo.mysql
```

a:

```text
native.mysqli
```

---

# 339. Escape hatch difference

Sólo APIs explícitamente nativas podrán variar.

---

# 340. Extension governance

Drivers oficiales deberán mantener:

```text
compatibility matrix
release policy
deprecation policy
security policy
conformance status
```

---

# 341. Deprecation

Un Driver podrá marcarse:

```text
DEPRECATED
```

sin desaparecer inmediatamente.

---

# 342. Deprecation diagnostics

El framework podrá advertir:

```text
Driver "pdo.legacy" is deprecated.
Recommended replacement: "native.vendor".
```

---

# 343. Driver removal

Requerirá una ventana de compatibilidad acorde a la política de versionado de VoltStack.

---

# 344. Alias migration

Aliases podrán ayudar durante migraciones, pero no ocultarán cambios semánticos incompatibles.

---

# 345. Security advisories

Una extensión vulnerable podrá ser marcada por tooling futuro como:

```text
UNSAFE_VERSION
```

pero la infraestructura base no dependerá de servicios externos para funcionar.

---

# 346. Extension metadata

Podrá contener:

```text
package
version
homepage
maintainer
license
support status
```

como metadata descriptiva.

---

# 347. Metadata not trusted for behavior

Las capabilities seguirán requiriendo contratos y validación.

---

# 348. Driver Extension API

La API de extensión deberá ser suficientemente pequeña para mantener compatibilidad a largo plazo.

---

# 349. Minimal extension surface

Idealmente un Driver externo sólo necesitará implementar/registrar:

```text
Descriptor
Factory
Native Connection Adapter
Native Statement Adapter
Native Result Adapter
Configuration Schema
Error Adapter
Capability Provider
Availability Provider
```

según necesidades.

---

# 350. Optional components

No todos serán obligatorios.

Por ejemplo:

```text
AvailabilityProvider
CapabilityProvider
```

podrán tener defaults conservadores.

---

# 351. Conservative defaults

Si una capability no está declarada:

```text
UNSUPPORTED / UNKNOWN
```

según naturaleza.

Nunca:

```text
probably supported
```

---

# 352. Extension correctness

La ausencia de información deberá favorecer seguridad y correctness.

---

# 353. Extension performance

Un Driver Extension no deberá introducir:

```text
package discovery
reflection
container lookup
config parsing
capability discovery
```

en cada query.

---

# 354. Runtime hot path

Ideal:

```text
Executor
   │
   ▼
Resolved Connection
   │
   ▼
Native Statement
   │
   ▼
Execute
```

con la mayor parte de metadata ya resuelta.

---

# 355. Driver reuse

El objeto Driver podrá ser singleton si:

```text
stateless
immutable
reentrant
```

---

# 356. Native connection reuse

Se gestiona separadamente por Pool/Lifecycle.

---

# 357. Driver singleton ≠ connection singleton

Regla:

```text
Singleton Driver
does not imply
Singleton Native Connection
```

---

# 358. Extension state

Cualquier estado mutable deberá tener ownership/lifetime explícito.

---

# 359. Forbidden extension state

No almacenar globalmente:

```text
current connection
current tenant
current transaction
current statement
current request
last query
current bindings
```

---

# 360. Diagnostics state

Counters/telemetry deberán utilizar integración apropiada y concurrency-safe.

---

# 361. Event integration

Driver-level events deberán ser mínimos.

---

# 362. Preferred telemetry layer

La mayoría de:

```text
QueryStarted
QueryCompleted
QueryFailed
```

deberán emitirse desde Execution Engine.

---

# 363. Driver telemetry

Puede aportar detalles de bajo nivel como:

```text
native connect duration
protocol failure
native cancellation
```

a través de ports.

---

# 364. EventSystem optional

Driver no dependerá directamente de:

```text
Quantum/EventSystem
```

---

# 365. Telemetry optional

Driver no dependerá directamente de:

```text
Quantum/Telemetry
```

---

# 366. Integration ports

Si se necesitan:

```text
DriverDiagnosticSinkInterface
```

o contratos equivalentes podrán utilizarse.

---

# 367. Null implementation

Sin Telemetry:

```text
NullDriverDiagnosticSink
```

---

# 368. No scattered checks

Evitar:

```php
if (class_exists(Telemetry::class)) {
}
```

dentro del Driver.

---

# 369. Extension system integration

Esto sigue la arquitectura definida en:

```text
09_DATABASE_EXTENSION_AND_CAPABILITY_MODEL.md
```

---

# 370. Driver-specific specialization

Este documento restringe ese modelo general al dominio:

```text
Driver
```

---

# 371. Relación con Connection Manager

ConnectionManager podrá resolver:

```text
ConnectionDefinition
```

que contiene `DriverId`.

No deberá saber cómo funciona la extensión concreta.

---

# 372. Relación con Pool

Pool administra:

```text
PhysicalConnection
```

creada por Driver.

No deberá conocer clases vendor-specific.

---

# 373. Relación con Lifecycle

Lifecycle coordina:

```text
initialize
lease
reset
retire
close
```

utilizando capabilities/primitives del Driver.

---

# 374. Relación con State & Reset

State & Reset decide si:

```text
Native Reset
```

es suficiente para volver al baseline.

---

# 375. Relación con Dialect

Dialect no depende del Driver concreto salvo que una capability específica lo requiera mediante una frontera neutral.

---

# 376. Relación con Platform

Platform describe el motor.

Driver describe comunicación.

---

# 377. Relación con Query Compiler

Compiler produce:

```text
CompiledQuery
```

sin llamar al Driver.

---

# 378. Relación con Executor

Executor es el principal consumidor runtime de las primitives del Driver a través de Connection.

---

# 379. Relación con ORM

No deberá existir dependencia directa.

```text
ORM
   │
   ▼
Query Engine
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

# 380. Invariantes arquitectónicos

## DB-DRV-EXT-001

Agregar un Driver no requerirá modificar el núcleo de Query Builder.

## DB-DRV-EXT-002

Agregar un Driver no requerirá modificar ORM.

## DB-DRV-EXT-003

Agregar un Driver no requerirá modificar Schema Model.

## DB-DRV-EXT-004

Driver, Dialect y Platform serán extensiones independientes.

## DB-DRV-EXT-005

Driver IDs serán estables y únicos.

## DB-DRV-EXT-006

Aliases estarán separados de IDs.

## DB-DRV-EXT-007

Aliases cíclicos serán rechazados.

## DB-DRV-EXT-008

Duplicate registration fallará.

## DB-DRV-EXT-009

No existirá last-registration-wins implícito.

## DB-DRV-EXT-010

Replacement deberá ser explícito.

## DB-DRV-EXT-011

Driver discovery ocurrirá durante bootstrap.

## DB-DRV-EXT-012

No habrá package discovery por query.

## DB-DRV-EXT-013

Registry será frozen antes del runtime normal.

## DB-DRV-EXT-014

Driver registration no abrirá conexiones.

## DB-DRV-EXT-015

Physical connections serán lazy.

## DB-DRV-EXT-016

Driver configuration será tipada y validada.

## DB-DRV-EXT-017

Native options estarán separados de portable configuration.

## DB-DRV-EXT-018

Unknown options fallarán por defecto.

## DB-DRV-EXT-019

Registered Driver no implicará available Driver.

## DB-DRV-EXT-020

Availability no implicará connectivity.

## DB-DRV-EXT-021

Un Driver requerido unavailable podrá fallar bootstrap.

## DB-DRV-EXT-022

Native handles no formarán parte de la API portable.

## DB-DRV-EXT-023

Native Results no hidratarán Entities.

## DB-DRV-EXT-024

Driver no realizará domain casting.

## DB-DRV-EXT-025

Type System preparará representaciones de base de datos.

## DB-DRV-EXT-026

Driver declarará honestamente prepared-statement capabilities.

## DB-DRV-EXT-027

Statement cache será distinto de CompiledQueryCache.

## DB-DRV-EXT-028

Native reset no implicará automáticamente reusable connection.

## DB-DRV-EXT-029

Reset safety será decidida por State/Platform/Lifecycle.

## DB-DRV-EXT-030

Driver adaptará errores nativos.

## DB-DRV-EXT-031

Driver no implementará retry policy.

## DB-DRV-EXT-032

Driver capabilities estarán limitadas a transporte/cliente.

## DB-DRV-EXT-033

Platform capabilities no se declararán como Driver capabilities.

## DB-DRV-EXT-034

Capabilities declaradas deberán ser verificables por conformance tests.

## DB-DRV-EXT-035

Todo Driver oficial pasará Driver Conformance Suite.

## DB-DRV-EXT-036

FrankenPHP será persistent-runtime reference.

## DB-DRV-EXT-037

Driver singleton no almacenará request state.

## DB-DRV-EXT-038

OpenSwoole optimizations respetarán scope isolation.

## DB-DRV-EXT-039

Core permanecerá runtime-neutral.

## DB-DRV-EXT-040

Async support utilizará contratos especializados.

## DB-DRV-EXT-041

Cloud credentials no se almacenarán globalmente.

## DB-DRV-EXT-042

Secrets no aparecerán en logs/fingerprints/diagnostics.

## DB-DRV-EXT-043

Drivers externos dependerán sólo de APIs de extensión públicas.

## DB-DRV-EXT-044

Internal APIs no serán extension points.

## DB-DRV-EXT-045

Extension dependency graph será acíclico.

## DB-DRV-EXT-046

Conflicts fallarán durante bootstrap.

## DB-DRV-EXT-047

Driver no utilizará Service Locator.

## DB-DRV-EXT-048

Driver no dependerá de EntityManager.

## DB-DRV-EXT-049

Driver no dependerá de Repository.

## DB-DRV-EXT-050

Driver no dependerá de HTTP.

## DB-DRV-EXT-051

Driver no dependerá de CurrentUser.

## DB-DRV-EXT-052

Drivers serán tratados como código privilegiado.

## DB-DRV-EXT-053

Native mutable access marcará el estado cuando corresponda.

## DB-DRV-EXT-054

UNKNOWN state nunca volverá al pool sin reset/validación.

## DB-DRV-EXT-055

Driver initialization y Platform session initialization estarán separados.

## DB-DRV-EXT-056

TLS REQUIRED nunca hará downgrade silencioso.

## DB-DRV-EXT-057

Cancellation deberá declarar sus efectos sobre Connection state.

## DB-DRV-EXT-058

Streaming Result mantendrá ownership apropiado del lease.

## DB-DRV-EXT-059

Ping no significará clean session.

## DB-DRV-EXT-060

Generated-key API será distinta de RETURNING.

## DB-DRV-EXT-061

Driver/Client/Engine versions serán conceptos distintos.

## DB-DRV-EXT-062

CLI diagnostics nunca expondrá secretos.

## DB-DRV-EXT-063

No existirá silent Driver fallback.

## DB-DRV-EXT-064

Automatic Driver selection deberá ser determinista y explicable.

## DB-DRV-EXT-065

Driver/Platform/Dialect compatibility será validada.

## DB-DRV-EXT-066

Runtime discovery será lazy cuando requiera conexión.

## DB-DRV-EXT-067

Driver extension upgrades invalidarán sólo caches relevantes.

## DB-DRV-EXT-068

Query AST permanecerá Driver-neutral.

## DB-DRV-EXT-069

Drivers oficiales PDO serán reference implementations.

## DB-DRV-EXT-070

Shared PDO code sólo contendrá comportamiento realmente común.

## DB-DRV-EXT-071

Se preferirá composición sobre herencia profunda.

## DB-DRV-EXT-072

Extension points tendrán stability classification.

## DB-DRV-EXT-073

Resource ownership será explícito.

## DB-DRV-EXT-074

Native exceptions no escaparán indiscriminadamente.

## DB-DRV-EXT-075

Security-critical extension failures serán fail-closed.

## DB-DRV-EXT-076

Compiled registries nunca serializarán conexiones/credenciales.

## DB-DRV-EXT-077

Registry lookup deberá ser de bajo costo.

## DB-DRV-EXT-078

No habrá reflection en hot path cuando pueda precompilarse.

## DB-DRV-EXT-079

Driver Conformance y Platform Conformance serán suites distintas.

## DB-DRV-EXT-080

Cambiar Driver no deberá cambiar el API ORM.

---

# 381. Anti-pattern — Driver switch central

Incorrecto:

```php
switch ($driver) {
    case 'mysql':
    case 'pgsql':
    case 'sqlite':
}
```

en el núcleo.

---

# 382. Correcto

```text
DriverId
   │
   ▼
Frozen DriverRegistry
   │
   ▼
DriverFactory
```

---

# 383. Anti-pattern — Driver compila SQL

Incorrecto:

```php
$driver->compileInsert($query);
```

---

# 384. Correcto

```text
Query
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
Driver
```

---

# 385. Anti-pattern — Driver como Platform

Incorrecto:

```php
$driver->supportsWindowFunctions();
```

---

# 386. Correcto

```text
Platform Capability System
```

---

# 387. Anti-pattern — Driver as Service Locator

Incorrecto:

```php
final class VendorDriver
{
    public function connect()
    {
        $config = app(Config::class);
        $telemetry = app(Telemetry::class);
        $tenant = app(CurrentTenant::class);
    }
}
```

---

# 388. Correcto

```text
Composition Root
      │
      ▼
Explicit Driver Dependencies
```

---

# 389. Anti-pattern — Driver owns current connection

Incorrecto:

```php
final class Driver
{
    private $currentConnection;
}
```

---

# 390. Correcto

```text
Stateless Driver
      │
      ▼
creates
      │
      ▼
NativeConnection
```

---

# 391. Anti-pattern — Native exception leaks

Incorrecto:

```text
PDOException
→
Controller
```

---

# 392. Correcto

```text
PDOException
      │
      ▼
NativeErrorAdapter
      │
      ▼
Database Failure System
```

---

# 393. Anti-pattern — Package discovery in hot path

Incorrecto:

```text
execute query
→ Composer scan
→ find driver
```

---

# 394. Correcto

```text
Bootstrap
→ Discover
→ Register
→ Validate
→ Freeze

Runtime
→ O(1) lookup
```

---

# 395. Anti-pattern — Capability optimism

Incorrecto:

```text
unknown capability
→ assume true
```

---

# 396. Correcto

```text
unknown capability
→ UNKNOWN / UNSUPPORTED
→ explicit resolution
```

---

# 397. Anti-pattern — Silent fallback

Incorrecto:

```text
native.pgsql unavailable
→ silently use pdo.pgsql
```

---

# 398. Correcto

```text
Explicit Selection Policy
      │
      ▼
Resolution Trace
```

---

# 399. Anti-pattern — Native reset equals clean

Incorrecto:

```text
driver.reset()
→ pool immediately
```

---

# 400. Correcto

```text
Native Reset
     │
     ▼
State Reset Semantics
     │
     ▼
Verification
     │
     ├── reusable
     └── discard
```

---

# 401. Anti-pattern — Async in every interface

Incorrecto:

```text
Promise<Result>
```

introducido en toda la arquitectura antes de necesitarlo.

---

# 402. Correcto

```text
Common Semantic Layers
        │
        ├── Sync Execution Adapter
        └── Async Execution Adapter
```

---

# 403. Anti-pattern — Platform package coupled to ORM

Incorrecto:

```text
VendorDriver
→ VendorEntityManager
```

---

# 404. Correcto

```text
ORM
  │
  ▼
Query Engine
  │
  ▼
Execution
  │
  ▼
Connection
  │
  ▼
Driver Extension
```

---

# 405. Arquitectura final

```text
                    VOLTSTACK DATABASE
                           │
             ┌─────────────┴─────────────┐
             │                           │
        Core Contracts              Extension API
                                         │
                                         ▼
                                Driver Extension
                                         │
                     ┌───────────────────┼───────────────────┐
                     ▼                   ▼                   ▼
                 Descriptor            Factory        Configuration
                     │                   │                 Schema
                     │                   │
                     │                   ├───────────────┐
                     │                   ▼               ▼
                     │                 Driver      Capability Provider
                     │                   │
                     │                   ▼
                     │            Native Connection
                     │                   │
                     │                   ▼
                     │            Native Statement
                     │                   │
                     │                   ▼
                     │             Native Result
                     │
                     └──────────────┬─────────────────────────┐
                                    ▼                         ▼
                              Error Adapter             Availability
                                    │
                                    ▼
                         Frozen Driver Registry
                                    │
                                    ▼
                             Connection System
                                    │
                                    ▼
                              Execution Engine
```

---

# 406. Arquitectura de extensibilidad completa

```text
Database Extension Package
           │
           ├── Driver Extension
           │       │
           │       └── Transport
           │
           ├── Dialect Extension
           │       │
           │       └── SQL Syntax
           │
           └── Platform Extension
                   │
                   └── Engine Semantics
```

---

# 407. Fórmula conceptual

```text
Database Target
=
Driver
+
Dialect
+
Platform
+
Effective Capabilities
```

---

# 408. Fórmula de extensión

```text
Custom Database Support
=
Driver Extension
+
Dialect Extension
+
Platform Extension
+
Schema Integration
+
Error Classification
+
Reset Semantics
+
Conformance Tests
```

No todos los términos serán necesarios cuando puedan reutilizarse componentes existentes.

---

# 409. Fórmula de seguridad

```text
Safe Driver Extension
=
Stable Contracts
+
Explicit Ownership
+
Validated Configuration
+
Secret Protection
+
Capability Honesty
+
State Tracking
+
Conformance Testing
+
Persistent Runtime Safety
```

---

# 410. Fórmula de runtime

```text
Persistent-Safe Driver
=
Stateless Driver
+
Scoped Native Resources
+
Exclusive Lease Ownership
+
Deterministic Cleanup
+
State Reset Cooperation
+
No Request Globals
```

---

# 411. Fórmula de rendimiento

```text
Low-Overhead Driver Resolution
=
Bootstrap Discovery
+
Compiled Registry
+
Frozen Metadata
+
O(1) Lookup
+
Pre-resolved Factory
+
No Hot-Path Reflection
```

---

# 412. Resultado

Con esta arquitectura, VoltStack podrá incorporar en el futuro:

```text
MySQL PDO
MySQLi
PostgreSQL PDO
Native PostgreSQL
SQLite PDO
SQLite3
SQL Server
Oracle
CockroachDB-compatible targets
specialized cloud connectors
async clients
coroutine clients
database proxies
```

sin convertir `Quantum/Database` en una colección de:

```text
if mysql
if postgres
if sqlite
if oracle
```

---

# 413. Cierre del bloque Infrastructure / Platform

Con este documento queda completo el primer bloque estructural de `Quantum/Database`:

```text
10_DATABASE_DRIVER_ARCHITECTURE.md
11_DATABASE_CONNECTION_SYSTEM.md
12_DATABASE_CONNECTION_MANAGER.md
13_DATABASE_CONNECTION_CONFIGURATION_AND_RESOLUTION.md
14_DATABASE_CONNECTION_POOLING_SYSTEM.md
15_DATABASE_CONNECTION_LIFECYCLE_SYSTEM.md
16_DATABASE_CONNECTION_STATE_AND_RESET_SYSTEM.md
17_DATABASE_DIALECT_SYSTEM.md
18_DATABASE_PLATFORM_CAPABILITY_SYSTEM.md
19_DATABASE_MYSQL_AND_MARIADB_PLATFORM.md
20_DATABASE_POSTGRESQL_PLATFORM.md
21_DATABASE_SQLITE_PLATFORM.md
22_DATABASE_DRIVER_EXTENSION_SYSTEM.md
```

El bloque establece definitivamente:

```text
Driver
    │
    ▼
Physical Communication

Connection
    │
    ▼
Managed Access

Dialect
    │
    ▼
SQL Language

Platform
    │
    ▼
Database Semantics

Capabilities
    │
    ▼
Effective Feature Availability

Extensions
    │
    ▼
Open Ecosystem
```

---

# 414. Regla maestra

> **Un Driver de VoltStack es un adaptador de comunicación con el motor de base de datos, no una implementación del ORM, del Query Builder, del SQL Compiler, de la Platform ni de la política de resiliencia.**

Por tanto:

```text
Driver Extension
      │
      ├── knows native client
      ├── knows native connection
      ├── knows native statement
      ├── knows native result
      ├── knows native errors
      └── declares transport capabilities

but does not know

      ├── Entity
      ├── Repository
      ├── UnitOfWork
      ├── Query Builder
      ├── Query AST semantics
      ├── Migration domain
      └── Application
```

---

# 415. Decisión final

VoltStack adoptará un Driver Extension System basado en:

```text
Stable Driver Contracts
+
Specialized Frozen Registries
+
Package Discovery
+
Typed Configuration
+
Explicit Factories
+
Native Adapters
+
Capability Providers
+
Error Adapters
+
Availability Providers
+
Compatibility Validation
+
Conformance Testing
+
Persistent Runtime Safety
```

Esto permitirá que `VoltStack/Quantum/Database` evolucione desde los Drivers iniciales:

```text
pdo.mysql
pdo.pgsql
pdo.sqlite
```

hacia un ecosistema extensible de clientes y motores sin comprometer la separación interna del framework.

---

# 416. Siguiente bloque

Con `22_DATABASE_DRIVER_EXTENSION_SYSTEM.md` finaliza el bloque:

```text
Driver / Connection / Dialect / Platform
```

El siguiente bloque inicia formalmente el:

```text
QUERY ENGINE
```

con:

```text
23_DATABASE_QUERY_ARCHITECTURE.md
```

---

# 417. Próximo documento

```text
23_DATABASE_QUERY_ARCHITECTURE.md
```

deberá establecer la arquitectura completa:

```text
Public Query API
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
Optimizer
      │
      ▼
Logical Plan
      │
      ▼
Physical Plan
      │
      ▼
Execution Plan
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
Driver
```

y fijará una de las reglas arquitectónicas más importantes de todo `Quantum/Database`:

> **El Query Builder describe intención; el AST representa estructura; el Semantic Engine determina significado; el Optimizer transforma; el Planner decide; el Compiler genera SQL; el Executor ejecuta.**

En consecuencia:

```text
Query Builder
≠
SQL Generator

AST
≠
SQL

Optimizer
≠
Database Optimizer

Planner
≠
Compiler

Compiler
≠
Executor
```

Esta separación será la base de todos los documentos `23`–`86` del Query Engine.