# 06_DATABASE_CONFIGURATION_SYSTEM.md

# VoltStack Quantum Database
## Configuration System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 06 — Database Configuration System  
**Estado:** Architecture Specification  
**Nivel:** Core Architecture  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura completa del sistema de configuración de:

```text
VoltStack/Quantum/Database
```

Su objetivo es establecer cómo Database obtiene, interpreta, valida, normaliza, compila y utiliza configuración relacionada con:

```text
connections
drivers
platforms
dialects
credentials
read/write topology
replicas
pooling
transactions
query engine
ORM
metadata
schema
migrations
cache
telemetry
security
resilience
runtime
extensions
```

La configuración deberá convertirse en un modelo tipado antes de ser consumida por los subsistemas internos.

---

# 2. Objetivo arquitectónico

El resto de `Quantum/Database` no deberá depender directamente de:

```text
$_ENV
$_SERVER
getenv()
global config arrays
dotenv files
framework config repository
runtime request data
```

El flujo será:

```text
External Configuration
        │
        ▼
Configuration Source
        │
        ▼
Configuration Loader
        │
        ▼
Normalization
        │
        ▼
Validation
        │
        ▼
Typed Configuration Model
        │
        ▼
Compilation
        │
        ▼
Runtime Configuration
```

---

# 3. Principio fundamental

La regla central será:

> La configuración es entrada de bootstrap, no dependencia dinámica distribuida por todo Database.

Por tanto, un componente interno no deberá hacer:

```php
$config = config('database.connections.mysql.host');
```

ni:

```php
$host = getenv('DB_HOST');
```

En su lugar recibirá:

```php
ConnectionConfiguration $configuration
```

o una abstracción más específica.

---

# 4. Responsabilidades del Configuration System

El sistema será responsable de:

```text
loading
merging
normalization
validation
typing
default resolution
environment overrides
secret references
connection definitions
topology definitions
capability hints
runtime settings
configuration compilation
cache generation
diagnostics
```

No será responsable de:

```text
opening database connections
running migrations
executing SQL
detecting entities
hydrating models
```

---

# 5. Arquitectura general

```text
Application Config
Environment
Secrets
Package Defaults
        │
        ▼
Configuration Sources
        │
        ▼
Database Configuration Loader
        │
        ▼
Merge Pipeline
        │
        ▼
Normalization
        │
        ▼
Validation
        │
        ▼
DatabaseConfiguration
        │
        ▼
Configuration Compiler
        │
        ▼
CompiledDatabaseConfiguration
        │
        ▼
Database Composition Root
```

---

# 6. Fuentes de configuración

Database podrá recibir información desde:

```text
framework configuration files
environment variables
application overrides
package defaults
runtime deployment configuration
secret providers
programmatic configuration
```

Estas fuentes deberán converger hacia un modelo común.

---

# 7. Integración con Quantum/Config

`Quantum/Database` podrá integrarse con:

```text
VoltStack/Quantum/Config
```

mediante un adapter.

Conceptualmente:

```text
Quantum/Config
     │
     ▼
DatabaseConfigSourceAdapter
     │
     ▼
Database Configuration Loader
```

El core Database no deberá depender de APIs concretas del Config package en sus subsistemas internos.

---

# 8. Configuration Source Contract

Podrá existir un contrato interno o de integración:

```php
interface DatabaseConfigurationSourceInterface
{
    public function load(): array;
}
```

Implementaciones:

```text
FrameworkConfigSource
EnvironmentConfigSource
ProgrammaticConfigSource
CompiledConfigSource
```

La API definitiva podrá utilizar objetos en lugar de arrays para determinadas fuentes.

---

# 9. Orden de precedencia

La precedencia deberá ser explícita.

Ejemplo recomendado:

```text
1. Database defaults
2. Package configuration
3. Application configuration
4. Environment overrides
5. Deployment/runtime overrides
6. Explicit programmatic overrides
```

Conceptualmente:

```text
Defaults
   ↓
Application
   ↓
Environment
   ↓
Deployment
   ↓
Explicit Override
```

El orden no deberá depender del orden accidental de carga de Service Providers.

---

# 10. Configuración inicial sugerida

Una aplicación podría disponer de:

```php
return [
    'default' => 'primary',

    'connections' => [
        'primary' => [
            'driver' => 'mysql',
            'host' => env('DB_HOST', '127.0.0.1'),
            'port' => env('DB_PORT', 3306),
            'database' => env('DB_DATABASE'),
            'username' => env('DB_USERNAME'),
            'password' => env('DB_PASSWORD'),
        ],
    ],
];
```

Esta representación pertenece a la capa de entrada.

No deberá viajar por los internals.

---

# 11. Typed Configuration Model

Después de normalizar y validar:

```text
Raw array
   │
   ▼
DatabaseConfiguration
```

Ejemplo conceptual:

```php
final readonly class DatabaseConfiguration
{
    public function __construct(
        public ConnectionName $defaultConnection,
        public ConnectionConfigurationCollection $connections,
        public OrmConfiguration $orm,
        public MigrationConfiguration $migrations,
        public RuntimeDatabaseConfiguration $runtime,
    ) {}
}
```

---

# 12. DatabaseConfiguration

Será la raíz del modelo de configuración.

Podrá contener:

```text
default connection
connection definitions
query settings
transaction defaults
schema settings
migration settings
ORM settings
metadata settings
cache settings
telemetry settings
security settings
runtime settings
extension settings
```

No deberá convertirse en una bolsa genérica arbitraria.

---

# 13. Configuration Decomposition

Preferir:

```text
DatabaseConfiguration
├── ConnectionConfigurationSet
├── QueryConfiguration
├── TransactionConfiguration
├── SchemaConfiguration
├── MigrationConfiguration
├── OrmConfiguration
├── CacheConfiguration
├── TelemetryConfiguration
├── RuntimeDatabaseConfiguration
└── SecurityConfiguration
```

sobre:

```php
$config->get('anything.deep.anywhere');
```

---

# 14. ConnectionConfiguration

Representará una conexión lógica.

Ejemplo conceptual:

```php
final readonly class ConnectionConfiguration
{
    public function __construct(
        public ConnectionName $name,
        public DriverName $driver,
        public DatabaseEndpoint $endpoint,
        public DatabaseName $database,
        public CredentialReference $credentials,
        public ConnectionOptionSet $options,
    ) {}
}
```

---

# 15. DatabaseEndpoint

Un endpoint deberá poder modelar:

```text
host
port
socket
DSN fragment
embedded file path
```

según driver.

Ejemplos:

```text
TCP endpoint
Unix socket
SQLite file
in-memory SQLite
```

No todos los drivers requieren host/port.

---

# 16. DriverConfiguration

Configuración específica del driver podrá encapsularse.

Ejemplo:

```text
DriverConfiguration
├── driver name
├── client options
├── connection flags
└── native driver settings
```

La parte genérica deberá mantenerse separada de opciones nativas.

---

# 17. Native Driver Options

VoltStack deberá permitir escape hatches controlados.

Ejemplo:

```php
'native_options' => [
    // driver-specific options
],
```

Después de normalizar:

```text
NativeDriverOptionSet
```

Estos valores deberán permanecer en la capa Driver/Connection.

No viajar al ORM.

---

# 18. Dialect Configuration

Normalmente el Dialect se inferirá del driver/platform.

Sin embargo, podrá existir override explícito para:

```text
custom dialect
compatible database service
proxy
vendor fork
extension package
```

Ejemplo:

```text
driver: mysql
dialect: custom-mysql-compatible
```

No deberá ser necesario para aplicaciones ordinarias.

---

# 19. Platform Configuration

La plataforma podrá resolverse desde:

```text
configured platform hint
driver
server version
capability discovery
```

Preferencia:

```text
Configuration
   +
Driver
   +
Server Discovery
       │
       ▼
Effective DatabasePlatform
```

No se deberán hardcodear capacidades únicamente desde nombre del driver.

---

# 20. Platform Version Hint

Podrá permitirse:

```php
'server_version' => '17.2',
```

o equivalente.

Esto puede ser útil para:

```text
offline compilation
build-time metadata
CI
proxy environments
avoiding runtime probing
```

El hint deberá validarse.

---

# 21. Capability Overrides

Para casos avanzados podrán existir overrides explícitos.

Ejemplo conceptual:

```text
capabilities:
    returning: true
    json: true
```

Deberán utilizarse con cautela.

Una declaración incorrecta puede provocar SQL inválido.

Por ello serán APIs avanzadas.

---

# 22. Credential Configuration

Las credenciales deberán representarse mediante:

```text
CredentialReference
```

cuando sea posible.

No mediante passwords propagados por todo el sistema.

Ejemplos:

```text
EnvironmentCredentialReference
StaticCredentialReference
SecretManagerReference
CloudIdentityReference
```

---

# 23. Credential Resolution

Flujo:

```text
ConnectionConfiguration
        │
        ▼
CredentialReference
        │
        ▼
CredentialProvider
        │
        ▼
DatabaseCredentials
        │
        ▼
Connection Factory
```

La resolución deberá ocurrir cerca de la creación de conexión.

---

# 24. DatabaseCredentials

Ejemplo conceptual:

```php
final readonly class DatabaseCredentials
{
    public function __construct(
        public ?string $username,
        public ?SensitiveString $password,
    ) {}
}
```

El objeto deberá aplicar reglas seguras para:

```text
debugging
serialization
stringification
logging
```

---

# 25. Secret Redaction

Nunca deberá ocurrir:

```text
DatabaseCredentials::__toString()
→ username/password exposed
```

Los dumps deberán producir algo como:

```text
DatabaseCredentials(
    username: "voltstack_app",
    password: "[REDACTED]"
)
```

---

# 26. Environment Variables

Variables de entorno podrán utilizarse en la capa Config.

Ejemplo:

```text
DB_CONNECTION
DB_HOST
DB_PORT
DB_DATABASE
DB_USERNAME
DB_PASSWORD
```

Pero después del bootstrap:

```text
Database internals
```

no deberán consultar estas variables directamente.

---

# 27. Zero Configuration

VoltStack deberá proporcionar defaults razonables.

Ejemplo para desarrollo:

```text
default connection: sqlite
database: storage/database.sqlite
```

podría ser un perfil opcional.

Sin embargo, los defaults exactos dependerán de la filosofía global de VoltStack.

---

# 28. Sensible Defaults

Defaults seguros deberán incluir cuando sea apropiado:

```text
parameter binding enabled
lazy connection creation
strict identifier handling
no automatic destructive migration
telemetry sensitive binding redaction
runtime state reset enabled
```

---

# 29. Default Connection

La configuración deberá definir:

```text
default connection
```

Ejemplo:

```php
'default' => 'primary',
```

Internamente:

```text
ConnectionName('primary')
```

La existencia deberá validarse.

---

# 30. Named Connections

Debe soportarse:

```text
primary
analytics
reporting
legacy
audit
```

Ejemplo:

```php
DB::connection('analytics');
```

La configuración de cada una será independiente.

---

# 31. Connection Aliases

Podrán existir aliases controlados.

Ejemplo:

```text
default → primary
read → primary-read
write → primary
```

No deberán permitir ciclos:

```text
A → B
B → A
```

---

# 32. Read/Write Configuration

Ejemplo conceptual:

```php
'connections' => [
    'primary' => [
        'driver' => 'pgsql',

        'write' => [
            'host' => 'primary-db',
        ],

        'read' => [
            ['host' => 'replica-1'],
            ['host' => 'replica-2'],
        ],
    ],
],
```

La entrada será normalizada a una topología estructurada.

---

# 33. ReadWriteConfiguration

Modelo:

```text
Logical Connection
├── Write Target
└── Read Targets
    ├── Replica A
    ├── Replica B
    └── Replica C
```

La resolución concreta pertenecerá a Topology/Connection Manager.

---

# 34. Replica Configuration

Cada replica podrá declarar:

```text
endpoint
weight
priority
region
tags
health policy
lag limits
```

Las opciones avanzadas serán opcionales.

---

# 35. Sticky Reads

Configuración posible:

```text
sticky_reads: true
```

Semántica:

```text
Write in current scope
      │
      ▼
subsequent compatible reads
      │
      ▼
primary/write connection
```

La configuración no implementa la política; únicamente la declara.

---

# 36. Connection Role Configuration

Los targets podrán etiquetarse:

```text
primary
replica
analytics
migration
read-only
```

El significado deberá resolverse mediante policies, no condicionales dispersos.

---

# 37. Pool Configuration

Ejemplo:

```php
'pool' => [
    'enabled' => true,
    'min' => 1,
    'max' => 20,
    'idle_timeout' => 60,
    'acquire_timeout' => 5,
],
```

Se normalizará a:

```text
ConnectionPoolConfiguration
```

---

# 38. Pool Defaults

El pooling no deberá asumirse universalmente.

Posibles estados:

```text
disabled
framework-managed
runtime-managed
external
```

La configuración deberá poder representar estas diferencias.

---

# 39. PoolMode

Podrá modelarse mediante enum:

```php
enum PoolMode
{
    case Disabled;
    case Internal;
    case External;
}
```

Esto evita flags ambiguos.

---

# 40. Pool Configuration Validation

Validaciones:

```text
min >= 0
max > 0
min <= max
acquire timeout >= 0
idle timeout >= 0
```

Si:

```text
pool disabled
```

ciertas opciones podrán ignorarse o producir warning según strict mode.

---

# 41. Connection Lifetime Configuration

Podrán configurarse políticas:

```text
max lifetime
idle lifetime
max queries per connection
health validation interval
```

para escenarios persistentes.

---

# 42. Lazy Connection Configuration

Default recomendado:

```text
lazy: true
```

La aplicación podrá arrancar sin abrir DB hasta primera operación.

---

# 43. Persistent Connection Configuration

No deberá confundirse:

```text
persistent worker
```

con:

```text
persistent database connection
```

Son conceptos distintos.

La configuración deberá tratarlos separadamente.

---

# 44. Connection Reset Configuration

Podrá existir:

```text
reset_on_scope_end: true
```

pero en runtimes persistentes el reset de estado inseguro será una garantía arquitectónica, no una feature opcional trivial.

El usuario podrá configurar estrategia, no desactivar garantías críticas sin una API unsafe explícita.

---

# 45. Session State Configuration

Podrán configurarse defaults de sesión:

```text
timezone
charset
collation
SQL mode
search path
application name
statement timeout
```

Estas opciones deberán aplicarse cerca del Connection Initialization pipeline.

---

# 46. Session State Ownership

Una opción de sesión puede clasificarse como:

```text
connection default
transaction local
operation local
tenant local
```

Esta clasificación es importante para reset.

---

# 47. Character Set Configuration

Para motores que lo requieran:

```text
charset
collation
```

deberán ser opciones tipadas/validadas.

No deberán interpolarse inseguramente en SQL.

---

# 48. Timezone Configuration

Deberá distinguirse:

```text
database session timezone
application timezone
PHP DateTime timezone
stored temporal semantics
```

La configuración Database sólo gobernará lo correspondiente a Database.

---

# 49. Query Configuration

Podrá incluir:

```text
default timeout
optimizer enabled
optimizer profile
query compilation cache
query tags
strict semantic validation
raw query policies
```

Ejemplo conceptual:

```text
QueryConfiguration
├── timeout
├── optimizer
├── compilation
└── diagnostics
```

---

# 50. Query Timeout Default

Un timeout por defecto podrá definirse:

```text
query.default_timeout
```

Pero deberá existir posibilidad de override por operación.

Prioridad conceptual:

```text
Global Query Default
       ↓
Connection Default
       ↓
Operation Override
```

---

# 51. Optimizer Configuration

Ejemplo:

```php
'query' => [
    'optimizer' => [
        'enabled' => true,
        'profile' => 'default',
    ],
],
```

No deberá exponer decenas de rules internas al usuario común.

Configuración avanzada podrá hacerlo.

---

# 52. Optimization Rule Configuration

Para debugging/extensibilidad podrá permitirse:

```text
enable rule
disable rule
rule-specific configuration
```

mediante IDs estables.

No por nombre de clase interna cuando sea evitable.

---

# 53. Compilation Configuration

Podrá incluir:

```text
compiled query cache
cache size
cache backend
debug SQL metadata
deterministic aliases
```

La mayor parte tendrá defaults internos.

---

# 54. Raw SQL Configuration

Opciones avanzadas:

```text
allow raw expressions
raw query telemetry
raw query auditing
strict intent requirement
```

No se deberá desactivar la API Raw arbitrariamente si es parte de la API pública, salvo políticas de aplicación.

---

# 55. Transaction Configuration

Podrá definir:

```text
default isolation level
automatic transaction around flush
deadlock retry policy
serialization retry policy
nested transaction behavior
savepoint policy
```

---

# 56. Isolation Level

Ejemplo:

```php
'transactions' => [
    'isolation' => 'read_committed',
],
```

Normalizado a:

```text
TransactionIsolationLevel::READ_COMMITTED
```

La configuración deberá validarse contra capabilities cuando se conozca la plataforma.

---

# 57. Transaction Retry Configuration

Ejemplo:

```php
'retry' => [
    'deadlock' => 3,
    'serialization_failure' => 3,
],
```

No significa que toda operación se reintentará automáticamente.

Transaction Manager seguirá evaluando retry safety.

---

# 58. Nested Transaction Configuration

Podrá definir comportamiento:

```text
savepoint
reject
join existing
```

dependiendo de API y capabilities.

No deberá simular atomicidad inexistente.

---

# 59. ORM Configuration

`OrmConfiguration` podrá contener:

```text
enabled
entity paths
metadata drivers
naming strategy
lazy loading policy
change tracking policy
auto transaction on flush
default repository
strict mapping
proxy strategy
```

---

# 60. ORM Optionality

Database deberá funcionar con:

```text
orm.enabled = false
```

sin inicializar innecesariamente:

```text
EntityManager
Metadata scanners
IdentityMap
UnitOfWork
relationship services
```

---

# 61. Entity Discovery Configuration

Podrá aceptar:

```text
paths
namespaces
explicit entities
package sources
```

Ejemplo:

```php
'orm' => [
    'entities' => [
        app_path('Domain'),
    ],
],
```

Discovery deberá compilarse/cachearse en producción.

---

# 62. Metadata Driver Configuration

Ejemplo:

```text
attributes
programmatic
compiled
convention
```

Podrá existir una chain.

Ejemplo:

```text
Compiled
  ↓ fallback development
Attributes
```

La precedencia deberá ser determinista.

---

# 63. Naming Strategy Configuration

Ejemplo:

```php
'orm' => [
    'naming_strategy' => 'snake_case',
],
```

Normalizado a una estrategia registrada.

No a una clase instanciada arbitrariamente desde string sin validación.

---

# 64. Lazy Loading Configuration

Podrán existir modos:

```text
allow
warn
forbid
```

Ejemplo:

```php
'lazy_loading' => 'warn',
```

Especialmente útil en desarrollo.

---

# 65. Change Tracking Configuration

La estrategia podrá ser:

```text
snapshot
explicit
notification
```

si el ORM soporta varias.

No se expondrán estrategias no implementadas.

---

# 66. ORM Flush Configuration

Podrán configurarse políticas como:

```text
transactional_flush
batch size
cascade warning threshold
```

Pero `flush()` seguirá teniendo semántica estable.

---

# 67. ORM Strict Mode

Podrá activar:

```text
mapping validation
detached entity checks
lazy loading restrictions
unsafe cascade diagnostics
partial entity checks
```

Sin alterar persistencia válida.

---

# 68. Metadata Configuration

Puede contener:

```text
compile
cache
strict mode
discovery
warmup
validation
```

---

# 69. Metadata Compilation

Producción debería favorecer:

```text
metadata.compile = true
```

o equivalente.

Flujo:

```text
Metadata Sources
       │
       ▼
Compiler
       │
       ▼
Compiled Metadata
       │
       ▼
Runtime Registry
```

---

# 70. Metadata Cache Configuration

Debe distinguir:

```text
metadata cache
query cache
result cache
```

No utilizar un único flag genérico:

```text
cache = true
```

para todo.

---

# 71. Schema Configuration

Podrá incluir:

```text
default schema
introspection cache
strict portability
identifier naming
schema diff behavior
native metadata retention
```

---

# 72. Default Schema

Para PostgreSQL u otros motores con namespaces:

```text
default_schema
```

podrá configurarse.

No deberá asumirse como equivalente a database name.

---

# 73. Schema Introspection Configuration

Podrá controlar:

```text
cache
refresh
native metadata
online/offline
```

No deberá provocar introspection en cada request.

---

# 74. Migration Configuration

Podrá contener:

```text
paths
table/repository
auto discovery
transaction mode
safety policy
lock policy
batching
zero-downtime analysis
```

---

# 75. Migration Paths

Ejemplo:

```php
'migrations' => [
    'paths' => [
        database_path('migrations'),
    ],
],
```

Se normalizará a rutas verificadas.

---

# 76. Migration Repository Configuration

Podrá definir:

```text
table name
connection
schema
```

Ejemplo:

```text
voltstack_migrations
```

No deberá acoplar Migration internals a una única tabla hardcoded sin posibilidad de configuración.

---

# 77. Migration Connection

Podrá utilizar:

```text
default runtime connection
```

o una conexión dedicada:

```text
migration
```

Esto permite separación de privilegios.

---

# 78. Migration Safety Configuration

Posibles políticas:

```text
allow
warn
strict
```

para operaciones:

```text
drop table
drop column
type narrowing
non-null conversion
blocking operation
```

---

# 79. Migration Transaction Configuration

Podrá ser:

```text
automatic
always
never
platform_default
```

pero deberá respetar:

```text
transactional DDL capability
```

---

# 80. Migration Lock Configuration

Deberá poder configurar:

```text
migration lock enabled
timeout
lock strategy
```

para evitar ejecuciones simultáneas.

---

# 81. Cache Configuration

Database deberá declarar sus necesidades sin exigir `Quantum/Cache`.

Configuración conceptual:

```text
database.cache
├── metadata
├── compiled_query
├── query_plan
├── result
└── entity
```

---

# 82. Cache Backend Resolution

Ejemplo:

```text
metadata → internal
compiled query → internal
result → quantum-cache
entity → disabled
```

La resolución ocurrirá durante bootstrap.

---

# 83. Cache Namespaces

Cada cache deberá utilizar namespace/prefix separado.

Ejemplo:

```text
voltstack.database.metadata
voltstack.database.compiled_query
voltstack.database.result
```

para evitar colisiones.

---

# 84. Cache Versioning

Configuración compilada deberá incorporar:

```text
framework version
database package version
compiler version
metadata version
```

donde corresponda.

---

# 85. Result Cache Defaults

Result Cache debería estar:

```text
disabled by default
```

porque cambia características de consistencia y freshness.

Debe activarse explícitamente.

---

# 86. Entity Cache Defaults

Second-level entity cache también deberá ser:

```text
disabled by default
```

en la primera arquitectura salvo decisión posterior.

---

# 87. Telemetry Configuration

Podrá incluir:

```text
enabled
query timing
slow query threshold
capture SQL template
capture bindings
query origin
N+1 detection
transaction tracing
connection metrics
```

---

# 88. Telemetry Optionality

Si `Quantum/Telemetry` no está instalado:

```text
telemetry adapter
→ NullDatabaseTelemetry
```

sin romper Database.

---

# 89. Binding Capture

Default recomendado en producción:

```text
capture_bindings: false
```

o:

```text
redacted
```

Nunca deberán almacenarse secretos indiscriminadamente.

---

# 90. Slow Query Threshold

Ejemplo:

```text
slow_query_threshold: 500ms
```

Debe normalizarse a un Duration Value Object.

No utilizar strings arbitrarios internamente.

---

# 91. Query Origin Capture

Puede ser:

```text
disabled
sampled
debug-only
```

porque obtener stack traces puede ser costoso.

---

# 92. N+1 Detection Configuration

Posibles modos:

```text
off
observe
warn
strict
```

Su implementación pertenece a Relationship/Telemetry.

Configuration sólo expresa política.

---

# 93. Security Configuration

Podrá incluir:

```text
credential handling
TLS policy
sensitive binding redaction
raw SQL audit
identifier strictness
administrative operation guards
```

---

# 94. TLS Configuration

Modelo conceptual:

```text
TlsConfiguration
├── enabled
├── required
├── verifyPeer
├── verifyHost
├── caReference
├── certificateReference
└── keyReference
```

No deberán mezclarse certificados como strings inseguros en diagnostics.

---

# 95. TLS Required Semantics

Si:

```text
required = true
```

y TLS no puede establecerse:

```text
connection fails
```

Nunca:

```text
fallback to plaintext
```

---

# 96. Identifier Strictness

Una opción podrá determinar cómo tratar identificadores dinámicos.

Ejemplo:

```text
strict identifiers: true
```

pero las APIs estructuradas deberán ser seguras por defecto independientemente de este flag.

---

# 97. Raw SQL Audit Configuration

Podrá activar eventos para:

```text
raw statement
native handle usage
unsafe expression
```

sin impedir su uso legítimo.

---

# 98. Resilience Configuration

Podrá contener:

```text
connect retry
query retry
transaction retry
backoff
circuit breaker port
failover
failure thresholds
```

---

# 99. Retry Profiles

Podrán existir perfiles:

```text
none
conservative
custom
```

Default:

```text
conservative
```

siempre respetando idempotency.

---

# 100. Connect Retry

Errores transitorios de conexión pueden ser reintentables.

Pero:

```text
authentication failure
invalid database
invalid credentials
```

no deberán reintentarse agresivamente.

---

# 101. Query Retry

No deberá configurarse simplemente:

```text
retry all queries = 3
```

porque puede duplicar writes.

La configuración deberá trabajar con FailureClassifier + QueryIntent.

---

# 102. Transaction Retry

La política de transacciones deberá estar separada de Query Retry.

Ejemplo:

```text
serialization failure
→ retry complete transaction callback
```

no únicamente último statement.

---

# 103. Backoff Configuration

Podrá incluir:

```text
base delay
max delay
multiplier
jitter
```

y deberá normalizarse a un Policy Object.

---

# 104. Circuit Breaker Configuration

Database podrá configurar un Port de resiliencia.

Pero el Circuit Breaker concreto puede pertenecer a otro subsystem.

Database no deberá implementar necesariamente una plataforma general de circuit breaking.

---

# 105. Runtime Configuration

Especialmente importante para persistent workers.

Podrá incluir:

```text
scope model
connection reuse
reset strategy
worker lifecycle
context cleanup
leak detection
strict reset
```

---

# 106. Runtime Profile

Podrá existir resolución automática por adapter:

```text
frankenphp
roadrunner
openswoole
classic
cli
```

Pero el Core recibirá un modelo neutral.

---

# 107. Classic Runtime

Un modo clásico puede representar:

```text
one request
one process lifecycle
```

Aun así, Database utilizará las mismas abstracciones de scope.

Esto evita tener dos arquitecturas.

---

# 108. FrankenPHP Configuration

Las opciones específicas de integración deberán vivir en:

```text
Runtime Adapter Configuration
```

no en el Query/ORM config.

Ejemplos:

```text
reuse connections
strict reset
worker health escalation
```

---

# 109. RoadRunner/OpenSwoole Configuration

En el futuro cada adapter podrá tener opciones específicas.

Pero:

```text
DatabaseConfiguration
```

deberá contener una sección neutral o extension-defined para ello.

---

# 110. Strict Reset

En persistent runtimes, default recomendado:

```text
strict_reset: true
```

Puede detectar:

```text
open transaction
open cursor
dirty UnitOfWork
tenant state
poisoned connection
```

al terminar scope.

---

# 111. Reset Failure Policy

Configuración conceptual:

```text
reset_failure:
    discard_connection
    invalidate_context
    signal_worker_recycle
```

El comportamiento concreto dependerá de gravedad.

---

# 112. State Leak Diagnostics

En development podrá activarse:

```text
runtime.detect_state_leaks = true
```

para identificar componentes scoped retenidos accidentalmente.

---

# 113. Extension Configuration

Cada extensión Database podrá registrar un schema de configuración propio.

Ejemplo:

```text
extensions:
    postgis:
        ...
```

No deberá usar keys globales arbitrarias.

---

# 114. Extension Namespacing

Las extensiones deberán declarar un namespace único:

```text
database.extensions.vendor.package
```

o equivalente estructural.

Esto evita colisiones.

---

# 115. Extension Configuration Contract

Podrá existir:

```php
interface DatabaseConfigurationExtensionInterface
{
    public function namespace(): string;

    public function normalize(mixed $value): mixed;

    public function validate(mixed $value): void;
}
```

La API definitiva podrá refinarse posteriormente.

---

# 116. Extension Configuration Freeze

Una vez terminado bootstrap:

```text
configuration
→ frozen
```

Las extensiones no deberán modificar silenciosamente configuración compartida durante runtime.

---

# 117. Configuration Normalization

La normalización convierte múltiples sintaxis equivalentes a un modelo común.

Ejemplo:

```php
'port' => '5432'
```

se convierte a:

```text
PortNumber(5432)
```

Otro ejemplo:

```text
'pool' => false
```

puede convertirse a:

```text
PoolConfiguration(mode: Disabled)
```

---

# 118. Normalization Responsibilities

Incluye:

```text
type coercion
default application
alias resolution
enum conversion
duration parsing
path normalization
connection inheritance
profile expansion
```

No deberá realizar I/O externo innecesario.

---

# 119. Validation

Después de normalizar:

```text
Normalized Configuration
          │
          ▼
Validation
          │
          ▼
Valid Typed Configuration
```

Si es inválida:

```text
fail before Database runtime
```

cuando sea posible.

---

# 120. Validation Categories

Se distinguen:

```text
structural validation
semantic validation
cross-reference validation
platform validation
online validation
```

---

# 121. Structural Validation

Comprueba:

```text
required fields
types
allowed values
format
range
```

Ejemplo:

```text
port between 1 and 65535
```

---

# 122. Semantic Validation

Comprueba relaciones lógicas.

Ejemplo:

```text
pool.min <= pool.max
```

o:

```text
default connection exists
```

---

# 123. Cross-Reference Validation

Ejemplos:

```text
cache backend exists
driver exists
dialect exists
connection alias target exists
migration connection exists
```

---

# 124. Platform Validation

Una vez conocida la plataforma podrá comprobar:

```text
configured isolation supported?
savepoints supported?
requested TLS mode available?
session option supported?
```

Puede ocurrir después de bootstrap base.

---

# 125. Online Validation

Requiere conexión al server.

Ejemplos:

```text
server version
database exists
credential validity
server capability
schema availability
```

No deberá ejecutarse necesariamente durante cada application boot.

---

# 126. Offline Validation

Debe poder ejecutarse sin DB cuando exista suficiente información.

Ejemplos:

```text
configuration structure
driver registration
known server version
metadata configuration
migration paths
```

---

# 127. Validation Modes

Podrán existir:

```text
basic
strict
online
```

según CLI/build/runtime.

No todas las aplicaciones requerirán conexión durante bootstrap.

---

# 128. Configuration Diagnostics

Los errores deberán indicar:

```text
configuration path
invalid value classification
expected form
connection name
recommended correction
```

sin exponer secrets.

Ejemplo:

```text
database.connections.primary.port:
Expected integer between 1 and 65535.
Received "postgres".
```

---

# 129. Unknown Configuration Keys

En strict mode:

```text
unknown keys
→ error
```

o warning configurable.

Esto ayuda a detectar typos:

```text
passwrod
```

en vez de:

```text
password
```

---

# 130. Secret-Key Special Handling

Un typo en campos sensibles deberá tratarse cuidadosamente.

Ejemplo:

```text
database.connections.primary.passwrod
```

no deberá ser ignorado silenciosamente dejando conexión sin password y generando diagnóstico ambiguo.

---

# 131. Configuration Inheritance

Podrá permitirse reutilizar definiciones.

Ejemplo:

```php
'connections' => [
    'base_pgsql' => [
        'driver' => 'pgsql',
        'port' => 5432,
    ],

    'primary' => [
        'extends' => 'base_pgsql',
        'host' => 'db-primary',
    ],
],
```

La herencia deberá resolverse durante normalization.

---

# 132. Inheritance Cycle Detection

Prohibido:

```text
A extends B
B extends C
C extends A
```

Debe detectarse antes de runtime.

---

# 133. Profiles

Podrán existir perfiles predefinidos.

Ejemplo:

```text
development
production
testing
```

pero no deberán cambiar semántica crítica automáticamente sin claridad.

---

# 134. Production Profile

Podrá favorecer:

```text
compiled metadata
strict reset
debug disabled
binding redaction
configuration cache
lazy connection
safe retry
```

---

# 135. Development Profile

Podrá favorecer:

```text
mapping diagnostics
query origin
N+1 warnings
uncompiled fallback
configuration explanations
```

---

# 136. Testing Profile

Podrá configurar:

```text
test connection
transactional test strategy
strict cleanup
diagnostics
```

sin asumir SQLite automáticamente.

---

# 137. Environment Configuration

El environment puede seleccionar valores.

Pero Database no deberá tener lógica como:

```php
if (APP_ENV === 'testing') {
    use SQLite;
}
```

hardcoded.

La aplicación/config define la conexión.

---

# 138. Configuration Compilation

Una vez validada:

```text
DatabaseConfiguration
       │
       ▼
ConfigurationCompiler
       │
       ▼
CompiledDatabaseConfiguration
```

La representación compilada estará optimizada para bootstrap/runtime.

---

# 139. Compiled Configuration

Podrá contener:

```text
resolved driver identifiers
resolved connection definitions
normalized value objects
compiled topologies
resolved integration IDs
compiled policies
prevalidated extension config
```

No deberá contener recursos vivos.

---

# 140. Compiled Configuration Safety

No deberá contener:

```text
PDO handles
open connections
active transactions
EntityManager
UnitOfWork
request context
```

---

# 141. Secrets in Compiled Configuration

Preferencia:

```text
CredentialReference
```

y no:

```text
plain password
```

cuando el sistema de secretos lo permita.

Si se requiere cachear credenciales estáticas, deberá aplicarse una política segura y explícita.

---

# 142. Configuration Cache

Producción podrá guardar:

```text
CompiledDatabaseConfiguration
```

para evitar:

```text
parsing
normalization
repeated validation
extension discovery
```

en cada worker boot.

---

# 143. Cache Format

Podrá ser:

```text
generated PHP
safe serialized structure
compiled array
other versioned representation
```

No deberá convertirse en API pública.

---

# 144. Cache Signature

La configuración compilada deberá poder invalidarse con una firma basada en:

```text
VoltStack version
Database package version
configuration schema version
extension set
relevant environment inputs
```

---

# 145. Atomic Cache Writes

La generación deberá utilizar una estrategia segura:

```text
temporary file
   │
   ▼
complete write
   │
   ▼
atomic rename
```

cuando el backend sea filesystem.

---

# 146. Runtime Configuration Immutability

Una vez iniciado el scope runtime:

```text
CompiledDatabaseConfiguration
```

deberá considerarse inmutable.

No:

```php
$config->connections['primary']['host'] = 'other';
```

---

# 147. Dynamic Connection Definitions

Las aplicaciones avanzadas podrán necesitar conexiones dinámicas.

Ejemplo:

```text
database-per-tenant
```

Esto deberá utilizar:

```text
DynamicConnectionDefinition
ConnectionFactory
ConnectionResolver
```

no mutar configuración global.

---

# 148. Ephemeral Connection Configuration

Una definición dinámica podrá ser:

```text
request-scoped
tenant-scoped
operation-scoped
```

y deberá tener lifecycle explícito.

---

# 149. Dynamic Definition Validation

Toda definición dinámica deberá pasar por un subset apropiado de:

```text
normalization
validation
credential resolution
driver resolution
```

No deberá saltarse seguridad.

---

# 150. Dynamic Connection Cache

Si se cachean conexiones/definiciones dinámicas deberá existir:

```text
TTL
max size
eviction
scope
```

para evitar crecimiento sin límite.

---

# 151. Multitenancy Configuration

El core Database sólo deberá incluir puntos genéricos.

El paquete Multitenancy podrá registrar configuración como:

```text
tenant connection resolver
tenant schema resolver
tenant database strategy
```

sin convertir estas opciones en requisitos del core.

---

# 152. Tenant Configuration Ownership

Configuración de:

```text
tenant lifecycle
tenant provisioning
tenant domains
tenant users
```

pertenece a Multitenancy.

Database únicamente consume configuración de resolución DB cuando el adapter existe.

---

# 153. Sharding Configuration

Las capacidades distribuidas podrán definir:

```text
shards
partition keys
routing rules
regions
fallback
```

en módulos especializados.

No deberá contaminar el config mínimo de una aplicación simple.

---

# 154. Configuration Capability Discovery

Tooling podrá mostrar:

```text
available drivers
available platforms
available types
available cache adapters
available runtime adapters
```

para ayudar a construir configuración válida.

---

# 155. Driver Registry Integration

Durante validation:

```text
driver: pgsql
```

deberá verificarse contra:

```text
DriverRegistry
```

No mediante:

```text
class_exists
```

disperso.

---

# 156. Extension Registration Order

Las extensiones deberán registrarse antes de la fase final de validation/compilation cuando aporten:

```text
driver
dialect
type
config schema
capability
```

---

# 157. Bootstrap Phases

Configuración seguirá fases:

```text
1. Core defaults registered
2. Extension schemas registered
3. Sources loaded
4. Merge
5. Normalize
6. Validate
7. Compile
8. Freeze
9. Compose services
```

El orden será determinista.

---

# 158. Configuration Composition Root

La composición podrá verse como:

```text
Raw Config
    │
    ▼
Configuration System
    │
    ▼
Compiled Configuration
    │
    ▼
Database Composition Root
    │
    ├── DriverRegistry
    ├── ConnectionManager
    ├── Query Engine
    ├── ORM
    └── Integrations
```

---

# 159. No Connection During Basic Configuration Compilation

La fase básica de configuración no deberá abrir conexiones.

Esto permite:

```text
offline build
container compilation
CLI help
cache warmup partial
```

---

# 160. Deferred Online Resolution

Información como:

```text
actual server version
effective server capabilities
```

podrá resolverse al establecer primera conexión.

---

# 161. Platform Descriptor Cache

Una vez determinada una plataforma efectiva podrá conservarse de forma segura según:

```text
logical connection
server/version signature
```

si la topología es homogénea.

---

# 162. Heterogeneous Topologies

Si replicas poseen versiones diferentes:

```text
Connection Configuration
```

no deberá asumir una única capability set cuando no sea cierto.

La Topology/Platform layer resolverá capacidades efectivas.

---

# 163. Connection URL/DSN Support

Para DX podrá soportarse:

```text
DATABASE_URL
```

o connection URI.

Ejemplo conceptual:

```text
postgresql://user:secret@host:5432/app
```

Debe parsearse inmediatamente a objetos estructurados.

---

# 164. DSN Security

Una URI con password no deberá aparecer completa en:

```text
exceptions
logs
debug output
compiled diagnostics
```

Se representará como:

```text
postgresql://user:[REDACTED]@host:5432/app
```

---

# 165. DSN vs Structured Configuration

Ambos podrán ser aceptados como input.

Pero internamente convergerán:

```text
DSN ─────────┐
             ▼
        ConnectionConfiguration
             ▲
Structured ──┘
```

---

# 166. Conflicting Inputs

Si se define:

```text
DATABASE_URL
```

y al mismo tiempo:

```text
host
port
database
```

deberá existir una regla clara de precedencia o error.

No mezclar silenciosamente partes incompatibles.

---

# 167. Connection Option Categories

Las opciones deberán clasificarse.

```text
Portable Options
├── timeout
├── charset
├── timezone
└── TLS

Native Options
└── driver-specific
```

Esto mantiene portabilidad.

---

# 168. Portable Option Validation

El framework podrá validar opciones comunes.

Las native options deberán validarse por el driver cuando sea posible.

---

# 169. Unsupported Option Policy

Si una opción portable no está soportada:

```text
native fallback
emulation
warning
error
```

según semántica.

No ignorarla silenciosamente.

---

# 170. Configuration Schema

Database deberá mantener un schema formal conceptual para sus opciones.

Ejemplo:

```text
database
├── default: ConnectionName
├── connections: map<ConnectionName, ConnectionConfiguration>
├── query: QueryConfiguration
├── transactions: TransactionConfiguration
├── orm: OrmConfiguration
└── ...
```

Esto permitirá tooling y validación.

---

# 171. Tooling Support

El Configuration Schema podrá ser utilizado para:

```text
IDE completion
CLI validation
documentation generation
config inspection
migration between versions
```

---

# 172. Configuration Introspection

CLI futuro podrá ofrecer:

```text
db:config
```

o equivalente para mostrar:

```text
default connection
resolved driver
platform hint
pool mode
ORM state
cache adapters
runtime strategy
```

con secrets redactados.

---

# 173. Configuration Explain

Podrá existir un modo:

```text
config explain
```

para mostrar:

```text
value
source
default/override
normalization
```

Ejemplo:

```text
database.connections.primary.port
value: 5432
source: DB_PORT
normalized as: PortNumber
```

---

# 174. Configuration Provenance

Cada valor podrá opcionalmente mantener información de origen durante diagnostics.

Ejemplos:

```text
default
config file
environment
runtime override
extension
```

No es necesario conservar provenance en el hot path.

---

# 175. Configuration Error Codes

Errores relevantes podrán tener códigos estructurados.

Ejemplos:

```text
DB-CONFIG-001 unknown driver
DB-CONFIG-002 missing default connection
DB-CONFIG-003 invalid pool limits
DB-CONFIG-004 circular connection inheritance
DB-CONFIG-005 invalid TLS policy
```

La nomenclatura final podrá definirse posteriormente.

---

# 176. Configuration Exceptions

Jerarquía conceptual:

```text
DatabaseConfigurationException
├── InvalidConfigurationException
├── MissingConfigurationException
├── UnknownDriverConfigurationException
├── ConfigurationConflictException
├── ConfigurationCycleException
└── ConfigurationCompilationException
```

---

# 177. Configuration Warnings

No todo problema requiere detener bootstrap.

Podrán existir warnings para:

```text
deprecated option
ignored advanced option
potentially unsafe setup
unnecessary option
unsupported diagnostic feature
```

Pero seguridad/correctness crítica deberá producir error.

---

# 178. Deprecated Configuration

Las opciones deprecadas deberán indicar:

```text
old key
replacement
version deprecated
target removal
```

Ejemplo:

```text
database.pool.size
deprecated
use:
database.pool.max
```

---

# 179. Configuration Migration

VoltStack podrá proporcionar tooling para transformar configuraciones antiguas.

Especialmente cuando existan cambios de schema entre major versions.

---

# 180. Backward Compatibility

Alias temporales de configuración podrán mantenerse durante periodo de deprecación.

No deberán permanecer indefinidamente.

---

# 181. Programmatic Configuration

Paquetes o aplicaciones podrán registrar configuración mediante objetos.

Ejemplo conceptual:

```php
$database->configure(
    ConnectionConfiguration::postgres(...)
);
```

Esta API deberá pasar por la misma validación que configuración declarativa.

---

# 182. Programmatic Builders

Para DX podrán existir builders:

```php
ConnectionConfiguration::builder()
    ->driver('pgsql')
    ->host('localhost')
    ->database('app')
    ->build();
```

El resultado será un objeto tipado.

---

# 183. Configuration Builders vs Runtime Builders

No confundir:

```text
ConnectionConfigurationBuilder
```

con:

```text
QueryBuilder
```

Son dominios distintos.

---

# 184. Testing Configuration

Testing podrá construir:

```text
DatabaseConfiguration
```

programáticamente sin depender del sistema global Config.

Esto mejora unit/integration tests.

---

# 185. Minimal Test Configuration

Ejemplo conceptual:

```php
$config = DatabaseConfiguration::testing(
    connection: ConnectionConfiguration::sqliteMemory(),
);
```

si se decide proporcionar factories de conveniencia.

---

# 186. Config Fake

No deberá ser necesario mockear:

```text
env()
```

para probar Database.

Los componentes reciben objetos tipados.

---

# 187. Deterministic Configuration

Dadas las mismas:

```text
sources
environment values
extension set
```

la compilación deberá producir la misma configuración efectiva.

---

# 188. Configuration Hash

Podrá generarse:

```text
ConfigurationSignature
```

para:

```text
cache validation
diagnostics
worker comparison
deployment checks
```

sin incluir secrets directamente.

---

# 189. Secret-Safe Signature

Si secretos deben influir en invalidez, deberán utilizarse referencias o hashes seguros y no valores en claro almacenados.

---

# 190. Configuration and Hot Reload

VoltStack no asumirá que Database configuration cambia durante un request.

El hot reload, si se soporta en development, deberá:

```text
rebuild config
recompose affected services
dispose old scoped state
```

No mutar objetos existentes arbitrariamente.

---

# 191. Production Reconfiguration

Cambios importantes como:

```text
driver
host
pool
topology
credentials
```

normalmente requerirán:

```text
new connection lifecycle
```

y posiblemente worker recycle.

---

# 192. Credential Rotation

Es una excepción importante.

Una `CredentialReference` puede resolver nuevos secretos sin recompilar toda configuración.

Esto permite rotación segura.

---

# 193. Configuration and Dependency Injection

Los servicios deberán recibir secciones específicas.

Ejemplo:

```php
final class ConnectionFactory
{
    public function __construct(
        private ConnectionConfigurationSet $connections,
    ) {}
}
```

No:

```php
final class ConnectionFactory
{
    public function __construct(
        private DatabaseConfiguration $everything,
    ) {}
}
```

si sólo necesita connections.

---

# 194. Least Configuration Knowledge

Cada componente deberá recibir la menor configuración necesaria.

Ejemplo:

```text
QueryOptimizer
→ OptimizationConfiguration

MigrationRunner
→ MigrationConfiguration

EntityManagerFactory
→ OrmConfiguration
```

No todo `DatabaseConfiguration`.

---

# 195. Configuration Projection

La raíz podrá proyectarse en:

```text
ConnectionRuntimeConfig
QueryRuntimeConfig
OrmRuntimeConfig
```

durante composición.

Esto reduce acoplamiento.

---

# 196. Configuration Objects as Contracts

Los objetos de configuración son Value Objects, no service contracts.

No deberán adquirir comportamientos como:

```text
connect()
execute()
migrate()
```

---

# 197. No Config Service Locator

Debe evitarse:

```php
$config->get('database.anything');
```

dentro de Database.

La configuración tipada reemplaza este patrón.

---

# 198. Performance

El runtime no deberá pagar repetidamente por:

```text
string key lookup
environment parsing
duration parsing
schema validation
connection inheritance resolution
```

Estas operaciones deberán realizarse durante bootstrap/compilation.

---

# 199. Memory

La configuración compilada deberá ser suficientemente compacta para workers persistentes.

No deberá conservar:

```text
raw duplicated arrays
source parser state
full provenance
validation AST
```

salvo debug mode.

---

# 200. Development vs Production Representation

Development puede conservar más diagnostics:

```text
source provenance
warnings
schema paths
```

Production podrá utilizar una representación más compacta.

La semántica debe ser igual.

---

# 201. Security

El Configuration System deberá proteger especialmente:

```text
passwords
TLS private keys
secret references
cloud credentials
connection URLs
database usernames when sensitive
```

---

# 202. Configuration Serialization

Los objetos que contengan secrets resueltos no deberán serializarse por defecto.

Preferir cachear referencias.

---

# 203. Debug Information

Los Value Objects sensibles deberán implementar representación segura.

Conceptualmente:

```php
public function __debugInfo(): array
{
    return [
        'username' => $this->username,
        'password' => '[REDACTED]',
    ];
}
```

---

# 204. Configuration File Permissions

Si VoltStack genera archivos compilados que contengan información sensible, deberá:

```text
apply restrictive permissions where supported
avoid world-readable files
```

y documentar responsabilidades de deployment.

---

# 205. No Secrets in Generated Documentation

Comandos de diagnostics nunca deberán imprimir passwords reales por defecto.

---

# 206. Configuration and Audit

Cambios de configuración runtime relevantes podrán generar eventos auditables en tooling administrativo.

No es necesario registrar cada lectura de config.

---

# 207. Configuration Change Detection

En entornos persistentes, si un worker detecta que su configuración compilada es obsoleta, la política podrá ser:

```text
continue current worker until recycle
signal worker recycle
reload at safe boundary
```

No mutar a mitad de operación.

---

# 208. Worker Configuration Snapshot

Cada worker deberá operar sobre una snapshot consistente de configuración.

Esto evita:

```text
query 1 uses old host
query 2 uses new topology
same transaction
```

---

# 209. Scope Configuration Snapshot

Una operación transaccional deberá conservar las decisiones relevantes durante toda la transacción.

No cambiar:

```text
primary target
tenant target
isolation policy
```

en mitad del lifecycle.

---

# 210. Configuration of Connection Timeouts

Debe distinguir:

```text
connect timeout
acquire timeout
statement timeout
lock timeout
```

No agrupar todo bajo:

```text
timeout
```

---

# 211. Duration Type

Internamente:

```text
Duration
```

deberá representar tiempos.

No strings como:

```text
"5 seconds"
```

en hot paths.

---

# 212. Size Types

Pool limits, batch sizes y cache capacities podrán usar tipos validados.

No necesariamente Value Object para cada entero, pero sí configuraciones semánticamente claras.

---

# 213. Batch Configuration

Podrán existir defaults separados:

```text
persistence batch size
relation batch size
bulk insert size
```

No una única variable global:

```text
batch_size
```

para todo Database.

---

# 214. Parameter Limit Awareness

El usuario podrá definir:

```text
batch.max_parameters
```

como override.

Pero el Planner deberá considerar también:

```text
Platform capability
```

y seleccionar el mínimo seguro.

---

# 215. Query Cache Configuration vs Result Cache

Separación obligatoria:

```text
Compiled Query Cache
→ compilation optimization

Result Cache
→ data caching
```

Sus configuraciones no deberán compartir semántica accidental.

---

# 216. Debug Toolbar Configuration

La integración con Developer Debug Toolbar podrá configurarse en Telemetry/Integration.

No será responsabilidad del Query Engine.

---

# 217. Profiler Configuration

Podrá incluir:

```text
max queries stored
capture stack traces
capture plans
capture bindings
memory limits
```

para evitar leaks.

---

# 218. Logging Configuration

Database no deberá implementar un logging system paralelo.

Podrá configurar qué eventos envía a Telemetry/Logging adapter.

---

# 219. Audit Configuration

Audit podrá tener requisitos de:

```text
enabled operations
durability
sink
sensitive-data policy
```

separados de ordinary logs.

---

# 220. Administrative Configuration

Podrá incluir:

```text
health checks
diagnostics
backup integration
maintenance tools
```

pero estos módulos se documentarán posteriormente.

No deben afectar Query hot path.

---

# 221. Health Check Configuration

Podrá definir:

```text
timeout
connection
read-only probe
frequency for external monitor
```

La frecuencia normalmente pertenece al sistema que ejecuta health checks, no a Database Core.

---

# 222. Configuration Registry

No deberá existir un registry universal de toda la configuración.

El modelo root y sus subobjetos son suficientes.

Registries se utilizarán para extensiones/implementaciones, no como sustituto de config.

---

# 223. Configuration Extensions and Container

Una extensión podrá:

```text
register schema
register defaults
consume validated config
register services
```

No deberá mutar arbitrariamente configuración después del freeze.

---

# 224. Configuration Freeze

Punto conceptual:

```text
Configuration Compilation Complete
              │
              ▼
            FREEZE
              │
              ▼
        Service Composition
```

Después de `FREEZE`, sólo los dynamic/scoped definitions explícitos podrán variar.

---

# 225. Configuration Lifecycle

```text
Framework Boot
    │
    ▼
Register Schemas
    │
    ▼
Load Sources
    │
    ▼
Merge
    │
    ▼
Normalize
    │
    ▼
Validate
    │
    ▼
Compile
    │
    ▼
Freeze
    │
    ▼
Compose Database Services
    │
    ▼
Runtime
```

---

# 226. Configuration Module Structure

Propuesta conceptual:

```text
Configuration/
├── Contract/
├── Source/
├── Loader/
├── Merge/
├── Normalize/
├── Validation/
├── Compilation/
├── Model/
├── Connection/
├── Query/
├── Transaction/
├── ORM/
├── Schema/
├── Migration/
├── Cache/
├── Telemetry/
├── Security/
├── Runtime/
├── Extension/
└── Exception/
```

La estructura física final se refinará posteriormente.

---

# 227. Configuration Invariants

### DB-CONFIG-001

Los internals Database nunca leerán variables de entorno directamente.

### DB-CONFIG-002

La configuración raw no deberá circular por subsistemas internos.

### DB-CONFIG-003

Toda configuración runtime deberá estar normalizada y validada.

### DB-CONFIG-004

Los objetos de configuración serán preferentemente inmutables.

### DB-CONFIG-005

Los secrets deberán redactarse en diagnostics.

### DB-CONFIG-006

La configuración básica no abrirá conexiones.

### DB-CONFIG-007

La resolución online deberá estar separada de validation offline.

### DB-CONFIG-008

El default connection deberá existir.

### DB-CONFIG-009

Las extensiones deberán namespacear su configuración.

### DB-CONFIG-010

Después del freeze, configuración global no será mutable.

### DB-CONFIG-011

Los componentes recibirán sólo la configuración que necesitan.

### DB-CONFIG-012

La configuración no será utilizada como Service Locator.

### DB-CONFIG-013

Pool configuration deberá ser independiente de persistent runtime configuration.

### DB-CONFIG-014

Result Cache estará desactivado por defecto salvo decisión posterior explícita.

### DB-CONFIG-015

Second-level Entity Cache estará desactivado por defecto salvo configuración explícita.

### DB-CONFIG-016

Runtime state reset será una garantía arquitectónica en workers persistentes.

### DB-CONFIG-017

Database deberá poder funcionar sin ORM habilitado.

### DB-CONFIG-018

Database deberá poder funcionar sin Cache, Telemetry, EventSystem o Multitenancy instalados.

### DB-CONFIG-019

Driver, Dialect y Platform deberán resolverse como conceptos separados.

### DB-CONFIG-020

Credenciales deberán resolverse cerca del Connection boundary.

---

# 228. Ejemplo conceptual completo

Configuración de entrada:

```php
return [
    'default' => 'primary',

    'connections' => [
        'primary' => [
            'driver' => 'pgsql',

            'write' => [
                'host' => env('DB_PRIMARY_HOST'),
                'port' => 5432,
            ],

            'read' => [
                [
                    'host' => env('DB_REPLICA_1_HOST'),
                    'weight' => 2,
                ],
                [
                    'host' => env('DB_REPLICA_2_HOST'),
                    'weight' => 1,
                ],
            ],

            'database' => env('DB_DATABASE'),

            'credentials' => [
                'username' => env('DB_USERNAME'),
                'password' => env('DB_PASSWORD'),
            ],

            'pool' => [
                'mode' => 'internal',
                'min' => 1,
                'max' => 20,
                'acquire_timeout' => 5,
            ],

            'sticky_reads' => true,
        ],
    ],

    'query' => [
        'optimizer' => [
            'enabled' => true,
            'profile' => 'default',
        ],

        'timeout' => 30,
    ],

    'transactions' => [
        'isolation' => 'read_committed',
    ],

    'orm' => [
        'enabled' => true,
        'lazy_loading' => 'warn',
        'metadata' => 'attributes',
    ],

    'migrations' => [
        'safety' => 'strict',
    ],

    'telemetry' => [
        'enabled' => true,
        'slow_query_threshold' => 500,
        'capture_bindings' => false,
    ],

    'runtime' => [
        'strict_reset' => true,
    ],
];
```

---

# 229. Representación normalizada

La configuración anterior conceptualmente podrá convertirse a:

```text
DatabaseConfiguration
│
├── defaultConnection
│   └── ConnectionName("primary")
│
├── connections
│   └── ConnectionConfiguration("primary")
│       ├── DriverName("pgsql")
│       ├── Topology
│       │   ├── WriteTarget
│       │   └── ReplicaSet
│       ├── DatabaseName
│       ├── CredentialReference
│       ├── PoolConfiguration
│       └── StickyReadPolicy
│
├── QueryConfiguration
│   ├── OptimizationConfiguration
│   └── Duration(30s)
│
├── TransactionConfiguration
│   └── READ_COMMITTED
│
├── OrmConfiguration
│   ├── enabled
│   ├── lazyLoading: WARN
│   └── metadata: ATTRIBUTES
│
├── MigrationConfiguration
│   └── safety: STRICT
│
├── TelemetryConfiguration
│   ├── enabled
│   ├── slowQuery: 500ms
│   └── captureBindings: false
│
└── RuntimeDatabaseConfiguration
    └── strictReset: true
```

---

# 230. Consumer Model

Después del bootstrap:

```text
ConnectionFactory
    receives
ConnectionConfiguration

QueryOptimizer
    receives
OptimizationConfiguration

EntityManagerFactory
    receives
OrmConfiguration

MigrationManager
    receives
MigrationConfiguration

DatabaseLifecycle
    receives
RuntimeDatabaseConfiguration
```

Ninguno necesita consultar la raíz global durante ejecución normal.

---

# 231. Arquitectura resumida

```text
┌────────────────────────────────────┐
│ External Configuration Sources     │
│ Config / Env / Secrets / Defaults  │
└────────────────┬───────────────────┘
                 │
                 ▼
       Configuration Loader
                 │
                 ▼
              Merge
                 │
                 ▼
           Normalization
                 │
                 ▼
            Validation
                 │
                 ▼
      Typed DatabaseConfiguration
                 │
                 ▼
        Configuration Compiler
                 │
                 ▼
   CompiledDatabaseConfiguration
                 │
                 ▼
       Database Composition Root
                 │
        ┌────────┼─────────┐
        ▼        ▼         ▼
  Connection   Query      ORM
```

---

# 232. Resultado esperado

El Configuration System deberá proporcionar:

```text
strongly typed configuration
safe secrets
predictable defaults
early validation
runtime efficiency
environment independence
persistent-worker safety
optional integration support
clear extension configuration
zero/minimal configuration DX
```

mientras evita:

```text
global config access
environment reads in hot paths
mutable configuration
secret leakage
driver-specific conditionals everywhere
runtime configuration ambiguity
```

---

# 233. Principio final

La regla maestra será:

> Database consume configuración resuelta; nunca descubre su configuración de forma ad hoc durante la ejecución.

En forma resumida:

```text
Raw External Values
        │
        ▼
Normalize
        │
        ▼
Validate
        │
        ▼
Compile
        │
        ▼
Freeze
        │
        ▼
Inject
```

Nunca:

```text
Deep Runtime Component
        │
        ▼
env()
config()
global array
```

---

# 234. Conclusión

El sistema de configuración de VoltStack Database estará diseñado como una fase explícita de construcción de arquitectura.

Esto permitirá que una aplicación mantenga una superficie sencilla:

```php
DB::table('users')->get();
```

mientras internamente Database trabaja con:

```text
validated
typed
immutable
compiled
scope-safe
secret-safe
```

configuration objects.

La configuración será una entrada estructurada al sistema, no una dependencia global permanente.

---

# 235. Siguiente documento

El siguiente documento de la secuencia es:

```text
07_DATABASE_BOOTSTRAP_AND_SERVICE_CONTAINER_INTEGRATION.md
```

Ese documento deberá definir cómo la configuración compilada se transforma en el grafo real de servicios Database, incluyendo:

```text
Database Service Provider
Composition Root
service registration phases
singleton/scoped/transient lifetimes
driver registry bootstrap
platform registry bootstrap
type registry bootstrap
query engine construction
ORM lazy construction
connection factories
integration adapters
runtime lifecycle registration
extension registration
boot ordering
service freezing
container compilation
persistent worker compatibility
```

con especial énfasis en impedir errores como:

```text
singleton → request-scoped dependency capture
```

y garantizar que:

```text
Application Boot
      │
      ▼
Persistent Safe Services
      │
      ▼
Execution Scope
      │
      ▼
Scoped Database Services
```

sea el modelo de lifecycle utilizado por `VoltStack/Quantum/Database`.