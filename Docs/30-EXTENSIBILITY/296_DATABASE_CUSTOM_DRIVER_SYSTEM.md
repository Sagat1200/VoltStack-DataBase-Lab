# 296_DATABASE_CUSTOM_DRIVER_SYSTEM.md

# VoltStack Quantum Database
## Custom Driver System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 296 — Custom Driver System  
**Bloque:** 30 — Extensibility  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `295_DATABASE_PLUGIN_SYSTEM.md`  
**Siguiente documento:** `297_DATABASE_CUSTOM_DIALECT_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura oficial para implementar, registrar, validar y ejecutar **drivers personalizados** dentro de:

```text
VoltStack/Quantum/Database
```

El sistema deberá permitir incorporar nuevos mecanismos de comunicación con motores de base de datos sin modificar el núcleo de Database y sin acoplar:

```text
Query Engine
ORM
Schema
Migration
Transaction orchestration
```

al protocolo específico de un proveedor.

La regla central será:

> **Un custom driver de VoltStack Database implementará el transporte y las operaciones de bajo nivel necesarias para comunicarse con un sistema de base de datos mediante los contratos del Driver Layer; no deberá convertirse en Query Builder, Dialect, SQL Compiler, ORM, Schema Engine ni autoridad de semántica perteneciente a capas superiores.**

Formalmente:

```text
Custom Driver
=
Driver Identity
+
Driver Descriptor
+
Configuration
+
Connection Factory
+
Physical Connection Adapter
+
Statement Adapter
+
Result Adapter
+
Transaction Primitives
+
Error Translation
+
Lifecycle
+
Conformance
```

y nunca:

```text
Custom Driver
=
Entire Database Stack
```

---

# 2. Objetivos

El Custom Driver System deberá permitir:

1. añadir drivers sin modificar Database Core;
2. registrar drivers mediante Extension Architecture;
3. distribuirlos mediante Plugin System;
4. mantener una API uniforme;
5. separar driver, dialecto y plataforma;
6. crear conexiones físicas;
7. preparar statements;
8. realizar binding;
9. ejecutar statements;
10. consumir resultados;
11. soportar cursores;
12. soportar streaming cuando sea posible;
13. exponer primitivas transaccionales;
14. soportar savepoints cuando corresponda;
15. soportar cancelación cuando sea posible;
16. clasificar errores;
17. resetear conexiones;
18. soportar runtimes persistentes;
19. declarar capacidades de transporte;
20. proporcionar diagnostics;
21. implementar conformance tests;
22. proteger credenciales;
23. mantener compatibilidad con pooling;
24. soportar implementaciones nativas o externas;
25. preservar las invariantes generales de Database.

---

# 3. Posición arquitectónica

El Driver se encuentra en la parte inferior de la arquitectura operacional:

```text
ORM
 ↓
Query Engine
 ↓
Compiler
 ↓
Execution Engine
 ↓
Connection
 ↓
Driver
 ↓
Database Protocol / Native Library
 ↓
Database Server
```

El flujo nunca deberá invertirse.

---

# 4. Driver ≠ Connection

El Driver representa la implementación/proveedor del protocolo.

La Connection representa una sesión/conexión concreta.

```text
Driver
   ↓ creates
Physical Connection
```

Por tanto:

```text
Driver
≠
Connection
```

---

# 5. Driver ≠ Dialect

El Driver sabe:

```text
how to communicate
```

El Dialect sabe:

```text
how SQL is represented syntactically
```

Por tanto:

```text
Driver
≠
Dialect
```

---

# 6. Driver ≠ Platform

La Platform representa conocimiento semántico/capabilities de una familia de DBMS.

El Driver representa transporte.

```text
Driver
≠
Platform
```

---

# 7. Driver ≠ Compiler

El Driver recibe una representación ya compilada.

No genera SQL a partir del AST.

```text
Query AST
   ↓
Compiler
   ↓
Compiled Query
   ↓
Driver
```

Nunca:

```text
Query AST
   ↓
Driver
   ↓
SQL
```

---

# 8. Driver ≠ Query Builder

El Driver no deberá conocer:

```text
select()
where()
join()
orderBy()
```

como API de construcción semántica.

---

# 9. Driver ≠ ORM

El Driver no deberá conocer:

```text
Entity
Repository
UnitOfWork
IdentityMap
Relationship
```

---

# 10. Driver ≠ Schema Engine

El Driver puede ejecutar DDL compilado.

Pero no deberá diseñar:

```text
SchemaDiff
MigrationPlan
SchemaAST
```

---

# 11. Driver ≠ Transaction Manager

El Driver proporciona primitivas:

```text
begin
commit
rollback
savepoint
```

El Transaction Manager gobierna la semántica de alto nivel.

---

# 12. Arquitectura general

```text
Custom Driver Package
        │
        ▼
Driver Plugin
        │
        ▼
Driver Extension
        │
        ▼
Driver Descriptor
        │
        ▼
Driver Registry
        │
        ▼
Driver Factory
        │
        ▼
Driver Instance
        │
        ▼
Physical Connection Factory
        │
        ▼
Physical Connection
        │
        ├── Statement
        ├── Result
        ├── Transaction Primitives
        ├── Cancellation
        └── Reset
                │
                ▼
          Database Protocol
                │
                ▼
          Database Server
```

---

# 13. Driver Identity

Todo driver tendrá:

```text
DriverId
```

estable.

Ejemplos:

```text
pdo.mysql
pdo.pgsql
pdo.sqlite

native.mysql
native.postgresql

acme.oracle
acme.clickhouse
acme.customdb
```

---

# 14. DriverId

Conceptualmente:

```php
final readonly class DriverId
{
    public function __construct(
        public string $value,
    ) {}
}
```

El ID deberá ser:

```text
stable
unique
normalized
case-safe
```

---

# 15. DriverId ≠ PHP Class

No deberá utilizarse el FQCN como identidad durable principal.

```text
DriverId
≠
ImplementationClass
```

---

# 16. Driver Descriptor

Todo driver deberá proporcionar metadata estructural.

Conceptualmente:

```php
final readonly class DriverDescriptor
{
    public function __construct(
        public DriverId $id,
        public DriverVersion $version,
        public DriverTransport $transport,
        public DriverCapabilitySet $capabilities,
        public DriverConfigurationSchema $configuration,
    ) {}
}
```

---

# 17. Descriptor ≠ Driver Instance

El descriptor deberá poder inspeccionarse sin abrir una conexión.

---

# 18. Driver Version

Se distinguirá:

```text
DriverVersion
```

de:

```text
DatabaseServerVersion
PlatformVersion
PluginVersion
PackageVersion
```

---

# 19. Driver Version ≠ Server Version

Ejemplo:

```text
Driver:
pdo_pgsql 8.x

Server:
PostgreSQL 18.x
```

Son dimensiones distintas.

---

# 20. Driver Transport

Podrá identificar el mecanismo base:

```text
PDO
NATIVE_EXTENSION
FFI
SOCKET
HTTP
CUSTOM
```

---

# 21. Transport ≠ Platform

Un transporte HTTP podría comunicarse con distintos productos.

No deberá utilizarse como identidad de plataforma.

---

# 22. Driver Contract

Contrato conceptual:

```php
interface DatabaseDriver
{
    public function id(): DriverId;

    public function descriptor(): DriverDescriptor;

    public function connect(
        DriverConnectionConfiguration $configuration
    ): DriverConnection;
}
```

---

# 23. Driver Factory

Preferentemente la construcción se realizará mediante:

```text
DriverFactory
```

para evitar que consumidores conozcan implementaciones concretas.

---

# 24. Driver Factory Contract

```php
interface DriverFactory
{
    public function driverId(): DriverId;

    public function create(
        DriverRuntimeContext $context
    ): DatabaseDriver;
}
```

---

# 25. Driver Registry

El registro resolverá:

```text
DriverId
→
DriverFactory
```

---

# 26. Registry Freeze

El Driver Registry será:

```text
mutable during bootstrap
frozen during runtime
```

---

# 27. Duplicate DriverId

Dos drivers no podrán registrar silenciosamente el mismo ID.

Resultado:

```text
DuplicateDriverException
```

---

# 28. Driver Registration

Un plugin podrá registrar:

```php
$registrar->drivers()->register(
    descriptor: $descriptor,
    factory: $factory,
);
```

La API final podrá variar.

---

# 29. Driver Plugin Integration

Arquitectura:

```text
Composer Package
      ↓
Database Plugin
      ↓
Driver Extension
      ↓
Driver Registry
```

---

# 30. Plugin ≠ Driver

Un plugin puede proporcionar varios drivers.

```text
Plugin
 ├── Driver A
 └── Driver B
```

---

# 31. Driver Configuration

El driver recibirá únicamente configuración relevante al transporte.

Ejemplo conceptual:

```php
new DriverConnectionConfiguration(
    host: 'db.internal',
    port: 5432,
    database: 'app',
    credential: $credentialReference,
    options: $options,
);
```

---

# 32. Typed Configuration

Se evitarán configuraciones ambiguas basadas exclusivamente en:

```php
array<string, mixed>
```

para APIs internas.

---

# 33. Configuration Schema

Cada driver podrá declarar:

```text
required fields
optional fields
supported options
validation rules
sensitive fields
```

---

# 34. Unknown Options

La política deberá ser explícita.

Preferentemente:

```text
unknown structural option
→
configuration error
```

en lugar de ignorarla silenciosamente.

---

# 35. Credentials

El driver no deberá exigir que passwords permanezcan como strings globales.

Podrá recibir:

```text
CredentialReference
CredentialHandle
ResolvedCredential
```

según el diseño final de seguridad.

---

# 36. Secret Lifetime

La credencial resuelta deberá existir durante el menor tiempo práctico.

---

# 37. Credential Logging

Nunca deberán registrarse:

```text
password
token
private key
full secret-bearing DSN
```

---

# 38. DSN

El driver podrá generar o consumir una representación de conexión.

Pero:

```text
DSN
≠
Database Configuration Model
```

El DSN es una representación de transporte.

---

# 39. Connection Factory

El Driver podrá delegar creación física a:

```text
DriverConnectionFactory
```

---

# 40. Physical Connection Contract

Conceptualmente:

```php
interface DriverConnection
{
    public function prepare(
        CompiledStatement $statement
    ): DriverStatement;

    public function close(): void;

    public function isAlive(): bool;
}
```

Las operaciones transaccionales podrán estar en interfaces especializadas.

---

# 41. Connection Wrapper

VoltStack podrá envolver la conexión física:

```text
DriverConnection
      ↓
VoltStack Connection
```

para añadir:

```text
state
lifecycle
telemetry
routing
pool integration
reset governance
```

---

# 42. Physical Connection ≠ Logical Connection

La conexión lógica utilizada por capas superiores puede encapsular una conexión física.

```text
Logical Connection
≠
Physical Driver Connection
```

---

# 43. Connection Identity

Cada conexión física deberá poder correlacionarse mediante una identidad interna segura.

Ejemplo:

```text
ConnectionId
```

sin exponer secretos.

---

# 44. Connection Lifecycle

Estados conceptuales:

```text
NEW
CONNECTING
OPEN
BUSY
RESETTING
CLOSING
CLOSED
BROKEN
UNKNOWN
```

---

# 45. Connection Establishment

El driver deberá distinguir fallos como:

```text
DNS
network
TLS
authentication
database selection
protocol negotiation
server rejection
timeout
```

cuando la infraestructura subyacente proporcione evidencia suficiente.

---

# 46. Connection Error ≠ Query Error

El error de conexión tendrá clasificación separada.

---

# 47. Native Exception Leakage

Las excepciones nativas del driver no deberán escapar indiscriminadamente hasta capas superiores.

---

# 48. Error Translation

El Driver Layer deberá traducir errores a una taxonomía estable.

Ejemplo:

```text
Native Driver Error
        ↓
Driver Error Classifier
        ↓
Database Exception
```

---

# 49. Preserve Native Evidence

La traducción no deberá destruir información útil.

Podrá conservar:

```text
native code
SQLSTATE
vendor code
safe message
phase
retry hints
```

de forma sanitizada.

---

# 50. Statement Preparation

El driver deberá poder recibir:

```text
CompiledStatement
```

y producir:

```text
DriverStatement
```

---

# 51. CompiledStatement

Conceptualmente contendrá:

```text
SQL/command representation
parameter descriptors
execution metadata
```

No deberá contener entidades ORM.

---

# 52. DriverStatement

Contrato conceptual:

```php
interface DriverStatement
{
    public function bind(
        DriverParameterBindings $bindings
    ): void;

    public function execute(): DriverExecutionResult;

    public function close(): void;
}
```

---

# 53. Prepare ≠ Execute

Regla:

```text
prepare()
≠
execute()
```

aunque una tecnología concreta no exponga ambas fases nativamente.

El adapter deberá preservar la semántica conceptual.

---

# 54. Parameter Binding

El driver recibirá valores ya clasificados mediante:

```text
Parameter
+
Database Type
+
Binding Metadata
```

---

# 55. Driver Binding ≠ ORM Conversion

La conversión:

```text
Value Object
→
Database logical value
```

pertenece al Type System.

El Driver realiza el binding final compatible con su API.

---

# 56. Binding Type

Ejemplos:

```text
NULL
BOOLEAN
INTEGER
FLOAT
STRING
BINARY
LOB
```

según soporte del transporte.

---

# 57. Value Parameterization

La regla general se mantiene:

```text
values
→
parameters
```

---

# 58. Identifier Binding

Los identificadores SQL normalmente no se resuelven como value parameters.

Por ello deberán haber sido:

```text
typed
validated
compiled
```

antes de llegar al driver.

---

# 59. Driver Must Not Quote Arbitrary Identifiers

La semántica de quoting de identifiers corresponde principalmente a Compiler/Dialect.

---

# 60. Statement Execution

El driver ejecutará la representación compilada.

Resultado conceptual:

```text
DriverExecutionResult
```

---

# 61. Execution Result Categories

Podrán existir:

```text
RowResult
AffectedRowsResult
GeneratedValueResult
CommandResult
MultiResult
```

según contratos soportados.

---

# 62. Driver Result ≠ ORM Entity

Nunca:

```text
Driver Result
→
Entity
```

directamente.

El flujo será:

```text
Driver Result
 ↓
Result System
 ↓
Hydration
 ↓
Entity / Scalar / DTO
```

---

# 63. Result Metadata

El driver podrá exponer:

```text
column count
column names
native type hints
affected rows
generated identifiers
```

cuando estén disponibles.

---

# 64. Result Metadata ≠ ORM Metadata

No confundir:

```text
Driver Result Metadata
```

con:

```text
Entity Metadata
```

---

# 65. Cursor Integration

Drivers capaces de cursor podrán implementar:

```text
DriverCursor
```

---

# 66. Cursor Contract

Conceptualmente:

```php
interface DriverCursor
{
    public function fetch(): ?DriverRow;

    public function close(): void;

    public function exhausted(): bool;
}
```

---

# 67. Cursor Ownership

El cursor deberá conocer qué recurso posee:

```text
statement
connection
server cursor
```

para garantizar cleanup correcto.

---

# 68. Cursor ≠ Array

El Result System no deberá asumir que todos los resultados están completamente materializados.

---

# 69. Streaming

Un driver podrá declarar soporte para:

```text
STREAMING_RESULTS
```

---

# 70. Streaming ≠ Cursor

Puede existir streaming sin un cursor server-side explícito.

---

# 71. Streaming Resource Ownership

Mientras exista un stream activo:

```text
connection availability
statement lifecycle
transaction lifecycle
```

deberán ser explícitos.

---

# 72. Multiple Active Results

El driver deberá declarar si soporta:

```text
MULTIPLE_ACTIVE_RESULTS
```

en una misma conexión.

---

# 73. Unknown Capability

Si no puede determinarse:

```text
UNKNOWN
```

no deberá interpretarse como soporte.

---

# 74. Transaction Primitives

El driver podrá implementar:

```text
begin
commit
rollback
```

como primitivas físicas.

---

# 75. Transaction Driver Contract

Conceptualmente:

```php
interface TransactionalDriverConnection extends DriverConnection
{
    public function begin(
        DriverTransactionOptions $options
    ): void;

    public function commit(): void;

    public function rollback(): void;
}
```

---

# 76. Transaction Primitive ≠ Transaction Policy

El driver no decide:

```text
retry policy
nested transaction policy
business transaction boundary
```

---

# 77. Isolation Level

El Driver puede transportar una solicitud de aislamiento.

Pero:

```text
Requested Isolation
≠
Effective Isolation
```

---

# 78. No Silent Isolation Downgrade

Si la plataforma/driver no soporta un nivel solicitado:

```text
reject
```

o reportar explícitamente la degradación cuando la política lo permita.

Nunca asumir equivalencia.

---

# 79. Savepoints

Podrá existir:

```php
interface SavepointDriverConnection
{
    public function createSavepoint(string $name): void;

    public function rollbackToSavepoint(string $name): void;

    public function releaseSavepoint(string $name): void;
}
```

---

# 80. Savepoint ≠ Nested Transaction

La existencia de savepoints permite implementar ciertas políticas de nested transactions.

Pero:

```text
Savepoint
≠
Independent Transaction
```

---

# 81. Generated Savepoint Names

Deberán ser:

```text
validated
collision-resistant within scope
dialect-compatible
```

---

# 82. Commit Outcome

El driver deberá conservar incertidumbre.

Ejemplo:

```text
COMMIT sent
connection lost
```

puede producir:

```text
UNKNOWN
```

---

# 83. Driver Must Not Invent Commit Success

Regla crítica:

> Si el transporte no puede demostrar si el servidor confirmó el COMMIT, el driver no deberá reportar éxito únicamente porque la operación fue enviada.

---

# 84. Statement Success ≠ Transaction Commit

Siempre:

```text
StatementSuccess
≠
TransactionCommit
```

---

# 85. Cancellation

Un driver podrá soportar:

```text
query cancellation
statement cancellation
connection cancellation
```

según tecnología.

---

# 86. Cancellation Capability

Podrá declararse:

```text
STATEMENT_CANCELLATION
CONNECTION_CANCELLATION
SERVER_SIDE_CANCELLATION
```

---

# 87. Cancellation Requested ≠ Cancellation Confirmed

```text
CancellationRequested
≠
CancellationSucceeded
```

---

# 88. Post-cancellation State

Después de cancelar deberá determinarse si la conexión está:

```text
REUSABLE
RESET_REQUIRED
BROKEN
UNKNOWN
```

---

# 89. Timeout

El driver podrá implementar timeouts nativos.

Pero el sistema general podrá imponer además:

```text
operation deadline
```

---

# 90. Timeout ≠ Cancellation

Una operación que excede un deadline puede requerir cancelación explícita.

---

# 91. Connection Reset

Un custom driver que permita reutilización deberá proporcionar estrategia de reset.

---

# 92. Reset Objective

Eliminar estado residual como:

```text
open transaction
session variables
temporary state
prepared resources
active cursor
changed isolation
locks
```

cuando sea aplicable.

---

# 93. Reset ≠ Health Check

```text
Reset
≠
Health Check
```

---

# 94. Reset ≠ Reconnect

Una implementación puede decidir reconectar como estrategia.

Pero conceptualmente:

```text
reset
```

significa volver a un estado conocido reutilizable.

---

# 95. Unknown Reset Outcome

Si no puede demostrarse:

```text
connection clean
```

la conexión no deberá volver al pool.

---

# 96. Connection Quarantine

Podrá marcarse:

```text
TAINTED
BROKEN
UNKNOWN
```

y cerrarse/descartarse.

---

# 97. Persistent Runtime Safety

Esto será crítico para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 98. Driver Object Sharing

Un driver stateless/inmutable podrá compartirse.

Una conexión física:

```text
must not be shared concurrently
```

salvo que el contrato específico demuestre seguridad.

---

# 99. Request Leakage

Nunca deberá sobrevivir accidentalmente:

```text
transaction
cursor
tenant session variable
temporary session configuration
```

entre requests.

---

# 100. FrankenPHP

Será runtime de referencia para probar:

```text
connection reuse
connection reset
worker lifecycle
request isolation
resource cleanup
```

---

# 101. RoadRunner

Aplicará las mismas invariantes.

---

# 102. OpenSwoole

El custom driver deberá declarar si su implementación es:

```text
coroutine-safe
```

cuando corresponda.

---

# 103. Coroutine Safety ≠ Thread Safety

Son propiedades distintas.

---

# 104. Driver Concurrency Model

El descriptor podrá declarar:

```text
SINGLE_OPERATION_PER_CONNECTION
MULTIPLEXED
COROUTINE_SAFE
THREAD_SAFE
UNKNOWN
```

---

# 105. UNKNOWN Concurrency

No deberá interpretarse como seguro.

---

# 106. Pool Compatibility

Un driver podrá declarar:

```text
POOLABLE
NOT_POOLABLE
CONDITIONALLY_POOLABLE
UNKNOWN
```

---

# 107. Poolable Requirements

Para ser poolable deberá existir estrategia segura de:

```text
liveness
reset
resource cleanup
state verification
```

---

# 108. Pool ≠ Driver

El driver no deberá convertirse en Connection Pool.

---

# 109. Driver Capability System

El driver podrá contribuir evidencia sobre capacidades de transporte.

Ejemplos:

```text
prepared statements
streaming
cancellation
binary binding
LOB streaming
multiple result sets
async I/O
```

---

# 110. Driver Capability ≠ Database Capability

Ejemplo:

```text
Driver supports cancellation API
```

no significa necesariamente:

```text
server guarantees cancellation semantics X
```

---

# 111. Capability Evidence

El driver será una fuente de evidencia para:

```text
DATABASE_DATABASE_FEATURE_CAPABILITY_SYSTEM
```

pero no autoridad absoluta.

---

# 112. Version ≠ Capability

No deberá asumirse:

```text
driver version >= X
→ capability Y
```

sin reglas/evidencia correspondientes.

---

# 113. Runtime Probes

Cuando sea necesario, un driver podrá contribuir probes.

---

# 114. Probe Safety

Un probe deberá ser:

```text
bounded
non-destructive by default
timeout-aware
observable
```

---

# 115. Probe Failure

```text
ProbeFailure
≠
CapabilityUnsupported
```

Podrá significar:

```text
UNKNOWN
```

---

# 116. Platform Association

Configuración podrá asociar:

```text
DriverId
+
PlatformId
```

pero una relación fija 1:1 no será obligatoria arquitectónicamente.

---

# 117. Driver Selection

El Connection System resolverá qué driver utilizar.

No el ORM.

---

# 118. Selection Model

Conceptualmente:

```text
Connection Configuration
       ↓
Driver Resolver
       ↓
DriverId
       ↓
Driver Registry
       ↓
Driver Factory
```

---

# 119. Explicit Driver Selection

Ejemplo:

```php
'driver' => 'acme.oracle',
```

---

# 120. Automatic Resolution

Podrá existir mediante:

```text
connection scheme
platform
configuration
```

si es determinista y no ambiguo.

---

# 121. Ambiguous Driver

Resultado:

```text
AmbiguousDriverException
```

No selección silenciosa.

---

# 122. Driver Aliases

Podrán existir:

```text
pgsql
postgres
pdo_pgsql
```

resolviendo a un ID canónico.

---

# 123. Alias Collision

Será error.

---

# 124. Driver Native Handle

Algunos casos avanzados pueden requerir acceso al handle nativo.

---

# 125. Native Handle Escape Hatch

Podrá existir una API explícita como:

```php
$connection->nativeHandle();
```

si se decide soportarla.

Pero deberá considerarse:

```text
unsafe / advanced
```

---

# 126. Native Handle Risks

Usarlo puede:

```text
bypass lifecycle
change session state
open transaction
change isolation
invalidate reset assumptions
```

---

# 127. Native Handle State Mutation

Si se permite mutación externa, la conexión podrá requerir:

```text
markTainted()
```

o quedar excluida del pooling.

---

# 128. No Hidden PDO Assumption

VoltStack no deberá diseñar Driver Contracts asumiendo que todos los drivers son PDO.

---

# 129. PDO as Adapter

PDO será:

```text
one driver implementation family
```

no la definición arquitectónica del Driver Layer.

---

# 130. Native Drivers

Podrán aprovechar características que PDO no exponga.

---

# 131. HTTP Database Drivers

También podrán existir drivers para sistemas con protocolo HTTP.

La arquitectura no deberá exigir sockets SQL tradicionales.

---

# 132. Async Drivers

La arquitectura deberá evitar bloquear futuras implementaciones:

```text
async
event-loop
coroutine
```

aunque V1 pueda ser predominantemente síncrona.

---

# 133. Sync Contract V1

V1 podrá establecer contratos síncronos como baseline.

Pero deberán evitarse decisiones que hagan imposible añadir:

```text
AsyncDriver
```

posteriormente.

---

# 134. Driver Error Taxonomy

Categorías recomendadas:

```text
DriverException
├── DriverConfigurationException
├── DriverConnectionException
├── DriverAuthenticationException
├── DriverTimeoutException
├── DriverProtocolException
├── DriverStatementException
├── DriverBindingException
├── DriverTransactionException
├── DriverCancellationException
├── DriverResetException
├── DriverCapabilityException
└── DriverResourceException
```

---

# 135. Database Error Classification

Errores del servidor deberán clasificarse además semánticamente cuando exista evidencia:

```text
constraint violation
unique violation
foreign key violation
deadlock
serialization failure
lock timeout
syntax error
permission denied
database unavailable
```

---

# 136. Native Code Mapping

Los mappings:

```text
SQLSTATE/vendor code
→
semantic error
```

podrán vivir en:

```text
Driver
Platform
Error Classifier
```

según la naturaleza de la evidencia.

---

# 137. Error Classifier Separation

Se recomienda:

```text
NativeError
   ↓
DriverErrorNormalizer
   ↓
PlatformErrorClassifier
   ↓
DatabaseException
```

cuando la distinción sea útil.

---

# 138. Retry Hint

Un driver puede aportar evidencia:

```text
possibly retryable
```

pero:

```text
Driver Retry Hint
≠
Retry Authorization
```

---

# 139. Retry Policy

Pertenece a:

```text
DATABASE_RETRY_POLICY_SYSTEM
```

y Transaction Retry System.

---

# 140. Diagnostics

El driver deberá exponer metadata segura para diagnóstico.

Ejemplo:

```text
Driver ID
Driver version
Transport
Native extension version
Platform association
Capabilities
Connection state
```

---

# 141. Diagnostics ≠ Secrets

Nunca:

```text
password
credential token
private key
full secret DSN
```

---

# 142. Driver Health

El Driver System podrá aportar pruebas de conectividad.

Pero:

```text
Driver Health
≠
Database Health
≠
Replica Eligibility
```

---

# 143. Driver Telemetry

Podrá emitir información como:

```text
connect duration
prepare duration
execute duration
fetch duration
reset duration
close duration
driver errors
```

---

# 144. Telemetry Cardinality

No deberá utilizar:

```text
raw SQL
raw parameters
secret DSN
```

como labels de métricas.

---

# 145. Query Fingerprint

La correlación deberá usar identificadores seguros como:

```text
QueryFingerprint
```

cuando corresponda.

---

# 146. Driver Events

Podrán existir eventos:

```text
DriverConnectionOpening
DriverConnectionOpened
DriverConnectionFailed
DriverStatementPrepared
DriverExecutionCompleted
DriverConnectionReset
DriverConnectionClosed
```

sin otorgarles autoridad para redefinir semántica.

---

# 147. Events ≠ Lifecycle Control

Un listener no deberá convertir:

```text
failed commit
```

en:

```text
success
```

---

# 148. Driver Security

El driver deberá respetar:

```text
credential protection
TLS configuration
certificate verification
safe logging
parameterization
least privilege
```

---

# 149. TLS

Un driver podrá declarar soporte para:

```text
TLS
certificate verification
client certificate
hostname verification
```

---

# 150. Secure Defaults

Cuando sea razonable:

```text
verify certificates
```

deberá ser el default para conexiones remotas seguras.

---

# 151. Insecure TLS

Deshabilitar verificación deberá requerir configuración explícita y generar diagnostics cuando corresponda.

---

# 152. Driver Authentication

Podrá soportar:

```text
username/password
certificate
token
IAM-like temporary credentials
socket/OS authentication
custom provider
```

---

# 153. Credential Rotation

La arquitectura deberá permitir que futuras conexiones utilicen credenciales rotadas sin reiniciar necesariamente todo el framework.

---

# 154. Connection Credential Snapshot

Una conexión existente puede haber sido creada con una generación anterior de credenciales.

Esto deberá ser explícito cuando sea relevante.

---

# 155. Driver Resource Management

Todo recurso deberá tener ownership definido:

```text
connection
statement
cursor
stream
LOB
native handle
```

---

# 156. Resource Owner

Ejemplo:

```text
Cursor
→ owns Statement
→ borrows Connection
```

o cualquier política que el contrato defina.

Nunca deberá quedar implícita.

---

# 157. Deterministic Cleanup

Siempre que sea posible:

```text
close()
```

deberá ser explícito.

No depender exclusivamente del GC de PHP.

---

# 158. Destructor ≠ Cleanup Guarantee

Los destructores pueden ser fallback.

No deberán ser la única garantía de liberación de recursos críticos.

---

# 159. Resource Leak Detection

Testing podrá detectar:

```text
unclosed statements
unclosed cursors
open transactions
unreleased connections
```

---

# 160. Driver Conformance Testing

Todo custom driver deberá ejecutar una suite común.

---

# 161. Conformance ≠ Platform Equivalence

La suite demostrará que el driver respeta contratos VoltStack.

No que todos los DBMS se comporten igual.

---

# 162. Driver Conformance Categories

```text
Identity
Configuration
Connection
Preparation
Binding
Execution
Results
Transactions
Errors
Cancellation
Reset
Resources
Persistent Runtime
Security
Diagnostics
```

---

# 163. Required Conformance

Un driver deberá demostrar como mínimo:

```text
connect
execute
bind
fetch
close
error translation
resource cleanup
```

para las capacidades que declare.

---

# 164. Capability-conditioned Conformance

Si declara:

```text
SAVEPOINTS
```

se ejecutarán tests de savepoints.

Si declara:

```text
STREAMING
```

se ejecutarán tests de streaming.

---

# 165. Capability Claim Must Be Tested

Regla:

```text
Declared Capability
→
Applicable Conformance Suite
```

---

# 166. Real Integration

Mocks no demostrarán comportamiento real del protocolo.

---

# 167. Fake Driver

Un Fake Driver podrá utilizarse para:

```text
unit testing
failure injection
contract testing of upper layers
```

pero:

```text
Fake Driver
≠
Real Driver Conformance Evidence
```

---

# 168. Driver Test Environment

Los tests reales utilizarán:

```text
286_DATABASE_DATABASE_TEST_ENVIRONMENT_SYSTEM.md
```

---

# 169. Driver Matrix

Para drivers oficiales:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

deberán probarse independientemente según aplique.

---

# 170. MySQL ≠ MariaDB

Aunque puedan compartir protocolos/implementaciones:

```text
MySQL behavior
≠
MariaDB behavior
```

---

# 171. SQLite

SQLite no deberá utilizarse como prueba universal de drivers cliente-servidor.

---

# 172. Failure Injection

El driver deberá probar cuando sea posible:

```text
connect timeout
server disconnect
query timeout
connection loss during statement
connection loss during commit
protocol error
authentication failure
```

---

# 173. Unknown Commit Testing

Drivers transaccionales deberán probar:

```text
COMMIT
→ connection failure at uncertain boundary
→ UNKNOWN
```

cuando el entorno permita inyección adecuada.

---

# 174. Reset Testing

Secuencia:

```text
Acquire Connection
      ↓
Change Session State
      ↓
Reset
      ↓
Reuse
      ↓
Verify Clean State
```

---

# 175. Persistent Runtime Testing

Secuencia:

```text
Request A
 ├── transaction
 ├── session state
 └── query

reset

Request B
 └── verify no leakage
```

---

# 176. Driver Performance Testing

Podrán medirse:

```text
connection establishment
prepared statement cost
execution overhead
fetch throughput
streaming throughput
reset cost
pool reuse
memory per connection
```

---

# 177. Performance ≠ Correctness

Un driver más rápido no podrá violar contratos.

---

# 178. Custom Driver Example

Estructura conceptual:

```text
acme/voltstack-oracle-driver/
├── composer.json
├── src/
│   ├── OracleDatabasePlugin.php
│   ├── OracleDriverExtension.php
│   ├── OracleDriver.php
│   ├── OracleDriverFactory.php
│   ├── OracleDriverDescriptor.php
│   ├── OracleConnection.php
│   ├── OracleStatement.php
│   ├── OracleResult.php
│   ├── OracleCursor.php
│   ├── OracleErrorClassifier.php
│   └── OracleConnectionResetter.php
└── tests/
    ├── Conformance/
    └── Integration/
```

---

# 179. Example Plugin

```php
final class OracleDatabasePlugin implements DatabasePlugin
{
    public function register(DatabasePluginRegistrar $registrar): void
    {
        $registrar->extension(
            new OracleDriverExtension()
        );
    }
}
```

---

# 180. Example Driver Extension

```php
final class OracleDriverExtension implements DatabaseExtension
{
    public function register(
        DatabaseExtensionRegistrar $registrar
    ): void {
        $registrar->drivers()->register(
            new OracleDriverDescriptor(),
            new OracleDriverFactory(),
        );
    }
}
```

---

# 181. Example Driver

```php
final class OracleDriver implements DatabaseDriver
{
    public function id(): DriverId
    {
        return new DriverId('acme.oracle');
    }

    public function connect(
        DriverConnectionConfiguration $configuration
    ): DriverConnection {
        // Transport-specific connection creation.
    }
}
```

---

# 182. What the Driver Does Not Implement

El ejemplo anterior no deberá contener:

```php
public function select(...)
public function where(...)
public function hydrateEntity(...)
public function persist(...)
public function schemaDiff(...)
```

porque pertenecen a otras capas.

---

# 183. Custom DBMS Integration

Para añadir un DBMS completamente nuevo normalmente se requerirán varias extensiones:

```text
Custom DBMS Plugin
│
├── Driver
├── Platform
├── Dialect
├── Compiler
├── Schema support
├── Type mappings
└── Capability providers
```

---

# 184. Driver Alone May Be Insufficient

Regla:

> **Poder conectarse a un DBMS no implica que VoltStack pueda utilizar todas sus capacidades.**

El Driver resuelve transporte.

Las demás capas resuelven semántica.

---

# 185. Driver Support Levels

Podrá declararse:

```text
EXPERIMENTAL
PARTIAL
STABLE
DEPRECATED
```

como maturity metadata.

---

# 186. Maturity ≠ Capability

Un driver `STABLE` puede no soportar streaming.

Un driver `EXPERIMENTAL` puede soportarlo.

---

# 187. Driver Deprecation

Drivers obsoletos seguirán la política general de:

```text
323_DATABASE_VERSIONING_SYSTEM.md
324_DATABASE_DEPRECATION_POLICY.md
```

---

# 188. Driver Replacement

Cambiar driver para una misma plataforma deberá considerarse una transición de infraestructura.

---

# 189. Driver Swap

Ejemplo:

```text
pdo.pgsql
→
native.postgresql
```

podría mantener:

```text
PostgreSQL Platform
PostgreSQL Dialect
```

pero cambiar:

```text
transport behavior
binding behavior
streaming
cancellation
performance
error evidence
```

---

# 190. Driver Swap Validation

Deberá ejecutar conformance/integration antes de considerarse equivalente.

---

# 191. No Driver Behavioral Equivalence Assumption

Dos drivers hacia el mismo servidor no son automáticamente equivalentes.

---

# 192. Driver Feature Negotiation

Podrá existir:

```text
Driver
+
Server
+
Configuration
→
Effective Driver Capability Snapshot
```

---

# 193. Capability Snapshot

Deberá estar asociado a:

```text
driver version
server endpoint
configuration generation
connection context
```

cuando corresponda.

---

# 194. Endpoint-specific Evidence

En clusters heterogéneos:

```text
Endpoint A
≠
Endpoint B
```

en capabilities observadas.

---

# 195. Driver Diagnostics Fingerprint

Podrá calcularse:

```text
DriverFingerprint
=
H(
  DriverId
  + DriverVersion
  + TransportVersion
  + StructuralConfiguration
)
```

sin secretos.

---

# 196. Cache Interaction

Cambiar driver podrá invalidar caches sensibles a:

```text
binding
compiled representation
capabilities
platform behavior
```

según diseño.

---

# 197. Compiled Query Cache

Si la representación compilada depende del driver:

```text
DriverFingerprint
```

deberá participar en su cache key.

Si no depende, no deberá añadirse innecesariamente.

---

# 198. Driver-specific Compilation

Deberá minimizarse.

Preferentemente:

```text
Platform/Dialect Compiler
```

maneja sintaxis.

Sólo características realmente ligadas al transporte deberán depender del driver.

---

# 199. Driver Boundary

La frontera ideal:

```text
Compiled Command
       ↓
Driver
       ↓
Protocol
```

---

# 200. Arquitectura de directorios propuesta

```text
src/Quantum/Database/Driver/
├── Contract/
│   ├── DatabaseDriver.php
│   ├── DriverFactory.php
│   ├── DriverConnection.php
│   ├── DriverStatement.php
│   ├── DriverCursor.php
│   └── TransactionalDriverConnection.php
│
├── Identity/
│   ├── DriverId.php
│   ├── DriverVersion.php
│   └── DriverFingerprint.php
│
├── Descriptor/
│   ├── DriverDescriptor.php
│   ├── DriverTransport.php
│   └── DriverMaturity.php
│
├── Configuration/
│   ├── DriverConfigurationSchema.php
│   ├── DriverConnectionConfiguration.php
│   └── DriverConfigurationValidator.php
│
├── Registry/
│   ├── DriverRegistry.php
│   ├── MutableDriverRegistry.php
│   └── FrozenDriverRegistry.php
│
├── Resolution/
│   ├── DriverResolver.php
│   ├── DriverAliasRegistry.php
│   └── DriverSelection.php
│
├── Connection/
│   ├── DriverConnectionFactory.php
│   ├── DriverConnectionState.php
│   ├── DriverConnectionResetter.php
│   └── DriverConnectionInspector.php
│
├── Statement/
│   ├── DriverStatement.php
│   ├── DriverParameterBindings.php
│   └── DriverExecutionResult.php
│
├── Result/
│   ├── DriverResult.php
│   ├── DriverRow.php
│   ├── DriverCursor.php
│   └── DriverResultMetadata.php
│
├── Transaction/
│   ├── DriverTransactionOptions.php
│   └── DriverSavepointSupport.php
│
├── Capability/
│   ├── DriverCapability.php
│   ├── DriverCapabilitySet.php
│   └── DriverCapabilityProvider.php
│
├── Error/
│   ├── DriverErrorNormalizer.php
│   ├── DriverErrorClassifier.php
│   └── NativeDriverError.php
│
├── Diagnostics/
│   ├── DriverDiagnostic.php
│   └── DriverInspector.php
│
└── Exception/
    ├── DriverException.php
    ├── DriverConfigurationException.php
    ├── DriverConnectionException.php
    ├── DriverStatementException.php
    ├── DriverBindingException.php
    ├── DriverTransactionException.php
    ├── DriverCancellationException.php
    └── DriverResetException.php
```

---

# 201. Custom Driver Package Structure

Para terceros:

```text
src/
├── Plugin/
├── Extension/
├── Driver/
├── Connection/
├── Statement/
├── Result/
├── Error/
├── Capability/
└── Testing/
```

---

# 202. Driver Conformance Package

VoltStack podrá publicar contratos reutilizables dentro de:

```text
VoltStack/Quantum/Database/Testing/Driver
```

---

# 203. Architectural Invariants

## DB-CUSTOM-DRV-001

Driver ≠ Connection.

## DB-CUSTOM-DRV-002

Driver ≠ Dialect.

## DB-CUSTOM-DRV-003

Driver ≠ Platform.

## DB-CUSTOM-DRV-004

Driver ≠ Compiler.

## DB-CUSTOM-DRV-005

Driver ≠ Query Builder.

## DB-CUSTOM-DRV-006

Driver ≠ ORM.

## DB-CUSTOM-DRV-007

Driver ≠ Schema Engine.

## DB-CUSTOM-DRV-008

Driver ≠ Transaction Manager.

## DB-CUSTOM-DRV-009

DriverId será estable.

## DB-CUSTOM-DRV-010

DriverId ≠ FQCN.

## DB-CUSTOM-DRV-011

DriverVersion ≠ ServerVersion.

## DB-CUSTOM-DRV-012

Descriptor ≠ Driver Instance.

## DB-CUSTOM-DRV-013

Transport ≠ Platform.

## DB-CUSTOM-DRV-014

Driver Registry será congelable.

## DB-CUSTOM-DRV-015

Duplicate DriverId será error.

## DB-CUSTOM-DRV-016

Plugin ≠ Driver.

## DB-CUSTOM-DRV-017

Configuración será validada.

## DB-CUSTOM-DRV-018

Credenciales no aparecerán en diagnostics.

## DB-CUSTOM-DRV-019

DSN ≠ Database Configuration Model.

## DB-CUSTOM-DRV-020

Physical Connection ≠ Logical Connection.

## DB-CUSTOM-DRV-021

Connection failure ≠ Query failure.

## DB-CUSTOM-DRV-022

Native exceptions no escaparán indiscriminadamente.

## DB-CUSTOM-DRV-023

Prepare ≠ Execute.

## DB-CUSTOM-DRV-024

Driver binding ≠ ORM conversion.

## DB-CUSTOM-DRV-025

Values continuarán parameterized.

## DB-CUSTOM-DRV-026

Driver no quoteará identifiers arbitrarios como responsabilidad general.

## DB-CUSTOM-DRV-027

Driver Result ≠ ORM Entity.

## DB-CUSTOM-DRV-028

Result Metadata ≠ ORM Metadata.

## DB-CUSTOM-DRV-029

Cursor ≠ Array.

## DB-CUSTOM-DRV-030

Streaming ≠ Cursor.

## DB-CUSTOM-DRV-031

Resource ownership será explícito.

## DB-CUSTOM-DRV-032

Transaction primitive ≠ Transaction policy.

## DB-CUSTOM-DRV-033

Requested isolation ≠ Effective isolation.

## DB-CUSTOM-DRV-034

No habrá silent isolation downgrade.

## DB-CUSTOM-DRV-035

Savepoint ≠ Nested Transaction.

## DB-CUSTOM-DRV-036

Unknown commit outcome permanecerá UNKNOWN.

## DB-CUSTOM-DRV-037

StatementSuccess ≠ TransactionCommit.

## DB-CUSTOM-DRV-038

CancellationRequested ≠ CancellationSucceeded.

## DB-CUSTOM-DRV-039

Timeout ≠ Cancellation.

## DB-CUSTOM-DRV-040

Reset ≠ Health Check.

## DB-CUSTOM-DRV-041

Reset ≠ Reconnect conceptualmente.

## DB-CUSTOM-DRV-042

Unknown reset state no volverá al pool.

## DB-CUSTOM-DRV-043

Request state no se filtrará entre workers/requests.

## DB-CUSTOM-DRV-044

Coroutine safety ≠ Thread safety.

## DB-CUSTOM-DRV-045

UNKNOWN concurrency ≠ safe.

## DB-CUSTOM-DRV-046

Pool ≠ Driver.

## DB-CUSTOM-DRV-047

Driver Capability ≠ Database Capability.

## DB-CUSTOM-DRV-048

Driver evidence ≠ final capability decision.

## DB-CUSTOM-DRV-049

Version ≠ Capability.

## DB-CUSTOM-DRV-050

Probe failure ≠ unsupported.

## DB-CUSTOM-DRV-051

Driver selection pertenece al Connection System.

## DB-CUSTOM-DRV-052

Ambiguous driver resolution será error.

## DB-CUSTOM-DRV-053

Alias collision será error.

## DB-CUSTOM-DRV-054

Native handle será escape hatch explícito.

## DB-CUSTOM-DRV-055

Native handle mutation podrá taint connection.

## DB-CUSTOM-DRV-056

PDO no definirá la arquitectura Driver.

## DB-CUSTOM-DRV-057

V1 síncrona no deberá impedir drivers async futuros.

## DB-CUSTOM-DRV-058

Error translation preservará evidencia segura.

## DB-CUSTOM-DRV-059

Retry hint ≠ Retry authorization.

## DB-CUSTOM-DRV-060

Diagnostics no expondrán secretos.

## DB-CUSTOM-DRV-061

Driver Health ≠ Database Health.

## DB-CUSTOM-DRV-062

Driver Health ≠ Replica Eligibility.

## DB-CUSTOM-DRV-063

Telemetry no utilizará raw parameters como labels.

## DB-CUSTOM-DRV-064

Events no redefinirán outcome.

## DB-CUSTOM-DRV-065

TLS insecure requerirá configuración explícita.

## DB-CUSTOM-DRV-066

Resource cleanup no dependerá únicamente del GC.

## DB-CUSTOM-DRV-067

Declared capability tendrá conformance test aplicable.

## DB-CUSTOM-DRV-068

Fake Driver ≠ Real Conformance Evidence.

## DB-CUSTOM-DRV-069

MySQL ≠ MariaDB.

## DB-CUSTOM-DRV-070

SQLite ≠ Universal Driver Proof.

## DB-CUSTOM-DRV-071

Unknown commit deberá probarse cuando sea técnicamente posible.

## DB-CUSTOM-DRV-072

Persistent runtime reset será parte de conformance.

## DB-CUSTOM-DRV-073

Performance no justificará romper contratos.

## DB-CUSTOM-DRV-074

Driver no implementará EntityManager.

## DB-CUSTOM-DRV-075

Driver no implementará Query AST.

## DB-CUSTOM-DRV-076

Driver no implementará SchemaDiff.

## DB-CUSTOM-DRV-077

Driver no decidirá ORM persistence.

## DB-CUSTOM-DRV-078

Driver alone no implica soporte completo del DBMS.

## DB-CUSTOM-DRV-079

Maturity ≠ Capability.

## DB-CUSTOM-DRV-080

Dos drivers para misma plataforma no se asumirán equivalentes.

## DB-CUSTOM-DRV-081

Capabilities podrán ser endpoint-specific.

## DB-CUSTOM-DRV-082

Driver fingerprint no incluirá secretos.

## DB-CUSTOM-DRV-083

Caches sensibles al driver incluirán fingerprint/generation cuando corresponda.

## DB-CUSTOM-DRV-084

Driver-specific compilation deberá minimizarse.

## DB-CUSTOM-DRV-085

Driver boundary ideal será Compiled Command → Protocol.

## DB-CUSTOM-DRV-086

Connection reset deberá eliminar estado transaccional residual.

## DB-CUSTOM-DRV-087

Active cursor deberá cerrarse antes de reutilizar connection.

## DB-CUSTOM-DRV-088

Open transaction impedirá retorno inseguro al pool.

## DB-CUSTOM-DRV-089

Driver deberá clasificar resource state después de cancellation.

## DB-CUSTOM-DRV-090

Driver no inventará server acknowledgement.

## DB-CUSTOM-DRV-091

Authentication errors serán distinguibles cuando exista evidencia.

## DB-CUSTOM-DRV-092

Protocol errors serán distinguibles de semantic query errors.

## DB-CUSTOM-DRV-093

Connection object no será compartido concurrentemente salvo soporte explícito.

## DB-CUSTOM-DRV-094

Connection configuration no será mutable arbitrariamente tras apertura.

## DB-CUSTOM-DRV-095

Driver extension utilizará Extension Architecture.

## DB-CUSTOM-DRV-096

Driver distribution podrá utilizar Plugin System.

## DB-CUSTOM-DRV-097

Driver contracts serán independientes de Composer.

## DB-CUSTOM-DRV-098

Driver contracts serán independientes de PDO.

## DB-CUSTOM-DRV-099

Custom Driver preservará Database architectural invariants.

## DB-CUSTOM-DRV-100

Custom Driver no invertirá dependencias hacia ORM/Query Engine.

---

# 204. Anti-patrones

## 204.1 Driver generando SQL desde Query AST

Incorrecto.

---

## 204.2 Driver hidratando entidades

Incorrecto.

---

## 204.3 Driver implementando UnitOfWork

Incorrecto.

---

## 204.4 Driver calculando SchemaDiff

Incorrecto.

---

## 204.5 Driver decidiendo retry de transacción

Incorrecto.

---

## 204.6 Driver devolviendo `COMMITTED` tras conexión perdida sin evidencia

Incorrecto.

---

## 204.7 Driver asumiendo que prepare = execute

Incorrecto conceptualmente.

---

## 204.8 Driver convirtiendo Value Objects ORM

Incorrecto.

---

## 204.9 Driver concatenando valores en SQL

Incorrecto.

---

## 204.10 Driver usando FQCN como identidad persistente

Incorrecto.

---

## 204.11 Driver registrándose durante una query

Incorrecto.

---

## 204.12 Driver guardando conexión actual en static

Incorrecto.

---

## 204.13 Reutilizar conexión después de reset incierto

Incorrecto.

---

## 204.14 Asumir que conexión viva está limpia

Incorrecto.

---

## 204.15 Tratar timeout como cancelación confirmada

Incorrecto.

---

## 204.16 Asumir que PDO representa todos los drivers

Incorrecto.

---

## 204.17 Asumir que mismo DBMS implica drivers equivalentes

Incorrecto.

---

## 204.18 Declarar capability sin conformance test

Incorrecto para drivers soportados oficialmente.

---

## 204.19 Usar SQLite fake para validar protocolo PostgreSQL

Incorrecto.

---

## 204.20 Devolver native exceptions sin normalización

Incorrecto como API pública.

---

# 205. Modelo formal

Sea:

```text
D
```

un driver.

Su responsabilidad deberá satisfacer:

```text
Responsibilities(D)
⊆
{
  Transport,
  PhysicalConnection,
  StatementPreparation,
  ParameterBinding,
  StatementExecution,
  ResultAccess,
  TransactionPrimitives,
  ResourceLifecycle,
  NativeErrorNormalization
}
```

y no:

```text
Responsibilities(D)
∩
{
  ORM,
  QueryPlanning,
  SemanticAnalysis,
  SQLGenerationFromAST,
  SchemaDiff,
  MigrationPlanning
}
≠ ∅
```

---

# 206. Driver Activation

Un driver podrá utilizarse si:

```text
Usable(D)
=
Registered(D)
∧
ConfigurationValid(D)
∧
Compatible(D)
∧
RequiredRuntimeAvailable(D)
∧
RequiredExtensionsAvailable(D)
```

---

# 207. Connection Reuse

Una conexión podrá volver al pool únicamente si:

```text
Reusable(C)
=
Open(C)
∧
NotBroken(C)
∧
NoActiveTransaction(C)
∧
NoActiveResult(C)
∧
ResetSucceeded(C)
∧
StateKnown(C)
```

---

# 208. Unknown State Rule

```text
State(C) = UNKNOWN
→
Reusable(C) = false
```

---

# 209. Capability Rule

Para capability `K`:

```text
EffectiveCapability(D, E, K)
=
Resolve(
  DriverEvidence(D, K),
  EndpointEvidence(E, K),
  PlatformEvidence(K),
  ConfigurationEvidence(K)
)
```

No:

```text
DriverClaims(K)
→
Supported(K)
```

automáticamente.

---

# 210. Transaction Outcome Rule

Después de un commit:

```text
Outcome
=
COMMITTED
```

sólo si existe evidencia suficiente.

Si la confirmación se pierde:

```text
Outcome
=
UNKNOWN
```

---

# 211. Arquitectura consolidada

```text
                    Application Configuration
                              │
                              ▼
                       Connection Manager
                              │
                              ▼
                        Driver Resolver
                              │
                              ▼
                         Driver Registry
                              │
                              ▼
                         Driver Factory
                              │
                              ▼
                        Database Driver
                              │
                              ▼
                    Physical Connection
                              │
             ┌────────────────┼─────────────────┐
             ▼                ▼                 ▼
         Statement        Transaction        Lifecycle
             │            Primitives           │
             ▼                │                 ▼
          Binding             │               Reset
             │                │                 │
             ▼                │                 │
          Execute             │                 │
             │                │                 │
             ▼                │                 │
           Result             │                 │
             │                │                 │
             └────────────────┼─────────────────┘
                              ▼
                       Native Protocol
                              │
                              ▼
                       Database Server
```

Mientras las capas superiores permanecen:

```text
ORM
 ↓
Query Model / AST
 ↓
Semantic Engine
 ↓
Optimizer
 ↓
Planner
 ↓
Compiler
 ↓
Execution Engine
 ↓
Connection
 ↓
Custom Driver
```

---

# 212. Estrategia V1

Para V1 deberán estabilizarse primero:

```text
DriverId
DriverDescriptor
DriverFactory
DriverRegistry
DriverConnection
DriverStatement
Parameter Binding
DriverResult
Transaction Primitives
Error Translation
Connection Reset
Capability Contribution
Driver Conformance Suite
```

---

# 213. Funciones avanzadas posteriores

Podrán evolucionar después:

```text
async drivers
multiplexing
native pipeline execution
advanced server cursors
driver-specific batching
zero-copy binary transfer
advanced cancellation
native observability
```

sin romper el contrato básico.

---

# 214. Regla final

> **El Custom Driver System será la frontera entre VoltStack Database y los mecanismos concretos utilizados para comunicarse con un DBMS. Su función será transportar comandos compilados, administrar recursos físicos y traducir evidencia del protocolo; nunca absorber las responsabilidades semánticas de las capas superiores.**

Por tanto:

```text
Driver
≠
Connection
```

```text
Driver
≠
Dialect
```

```text
Driver
≠
Platform
```

```text
Driver
≠
Compiler
```

```text
Driver
≠
Query Builder
```

```text
Driver
≠
ORM
```

```text
Driver
≠
Schema Engine
```

```text
Driver
≠
Transaction Manager
```

```text
DriverId
≠
FQCN
```

```text
DriverVersion
≠
ServerVersion
```

```text
Prepare
≠
Execute
```

```text
Driver Binding
≠
ORM Value Conversion
```

```text
Driver Result
≠
ORM Entity
```

```text
Savepoint
≠
Nested Transaction
```

```text
StatementSuccess
≠
TransactionCommit
```

```text
CancellationRequested
≠
CancellationSucceeded
```

```text
Timeout
≠
Cancellation
```

```text
Reset
≠
Health Check
```

```text
Driver Capability
≠
Database Capability
```

```text
Probe Failure
≠
Unsupported
```

```text
Retry Hint
≠
Retry Authorization
```

```text
Fake Driver
≠
Real Driver Evidence
```

```text
Same DBMS
≠
Equivalent Drivers
```

y finalmente:

```text
Safe Custom Driver
=
Stable Identity
+
Explicit Contracts
+
Typed Configuration
+
Secure Credentials
+
Physical Connection Abstraction
+
Prepared Statements
+
Safe Parameter Binding
+
Result Abstraction
+
Transaction Primitives
+
Error Translation
+
Explicit Resource Ownership
+
Connection Reset
+
Capability Evidence
+
Persistent Runtime Isolation
+
Conformance Testing
+
Diagnostics
```

---

# 215. Siguiente documento

```text
297_DATABASE_CUSTOM_DIALECT_SYSTEM.md
```

El siguiente documento deberá definir cómo VoltStack permitirá incorporar dialectos SQL personalizados sin mezclar la sintaxis de un DBMS con Driver, Platform, Query AST o Execution Engine.

La arquitectura deberá cubrir:

```text
Custom Dialect System
│
├── Dialect Identity
├── Dialect Descriptor
├── Dialect Registry
├── Dialect Factory
├── Dialect Resolution
├── Identifier Rules
├── Quoting Rules
├── Literal Representation
├── Parameter Placeholder Strategy
├── SQL Grammar Features
├── Function Syntax
├── Operator Syntax
├── DML Syntax
├── DDL Syntax Boundaries
├── Pagination Syntax
├── Returning Syntax
├── Lock Syntax
├── CTE Syntax
├── Window Syntax
├── Capability Requirements
├── Compiler Integration
├── Dialect Extensions
├── Validation
├── Conformance Testing
└── Plugin Integration
```

manteniendo como principio:

> **Un dialecto personalizado describirá las reglas sintácticas necesarias para representar operaciones SQL en una familia concreta de lenguaje, pero no ejecutará consultas, no administrará conexiones, no decidirá capacidades por sí solo y no sustituirá al modelo semántico de Query Engine.**