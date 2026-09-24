# 313_DATABASE_CONFIG_INTEGRATION_SYSTEM.md

# VoltStack Quantum Database
## Database Config Integration System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 313 — Database Config Integration System  
**Bloque:** 32 — VoltStack Integration  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `312_DATABASE_CONTAINER_INTEGRATION_SYSTEM.md`  
**Siguiente documento:** `314_DATABASE_CACHE_INTEGRATION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura mediante la cual `VoltStack/Quantum/Database` obtiene, transforma, valida, compila y consume configuración proveniente del sistema general de configuración de VoltStack.

La integración deberá proporcionar una experiencia sencilla para el desarrollador:

```php
'connections' => [
    'default' => [
        'driver' => 'postgresql',
        'host' => env('DB_HOST', '127.0.0.1'),
        'port' => env('DB_PORT', 5432),
        'database' => env('DB_DATABASE'),
    ],
],
```

sin permitir que el núcleo Database termine dependiendo internamente de:

```text
arrays arbitrarios
$_ENV
getenv()
.env
strings sin tipado
variables globales
nombres de variables de entorno
secretos materializados innecesariamente
```

La regla central será:

> **VoltStack Database deberá consumir configuración normalizada, validada, tipada e inmutable; el origen de esa configuración será responsabilidad de la capa de integración con Config.**

Formalmente:

```text
Configuration Source
        ↓
Resolution
        ↓
Normalization
        ↓
Validation
        ↓
Compilation
        ↓
Typed Immutable Configuration
        ↓
Database Runtime
```

Nunca:

```text
Database Runtime
      ↓
.env
```

---

# 2. Principios fundamentales

La arquitectura deberá preservar:

```text
Configuration Source
≠
Raw Configuration
≠
Resolved Configuration
≠
Normalized Configuration
≠
Validated Configuration
≠
Compiled Configuration
≠
Runtime State
```

y además:

```text
Configuration
≠
Secret
≠
Capability
≠
Connection
≠
DatabaseContext
```

---

# 3. Objetivos

El sistema deberá proporcionar:

1. configuración tipada;
2. defaults explícitos;
3. normalización;
4. validación temprana;
5. integración con variables de entorno;
6. referencias seguras a secretos;
7. múltiples conexiones;
8. configuración por driver;
9. configuración por plataforma;
10. configuración de pools;
11. timeouts;
12. retries;
13. read/write routing;
14. replicas;
15. sharding;
16. cache;
17. ORM;
18. telemetry;
19. security;
20. persistent runtimes;
21. multitenancy opcional;
22. extensiones;
23. configuración compilable;
24. configuración cacheable;
25. generaciones de configuración;
26. reload controlado;
27. diagnostics seguros;
28. testing overrides;
29. backward compatibility;
30. extensibilidad sin convertir Config en un God Object.

---

# 4. Arquitectura general

```text
config/database.php
       +
Environment Variables
       +
Secret References
       +
Extension Configuration
       ↓
VoltStack Config System
       ↓
DatabaseConfigSourceAdapter
       ↓
DatabaseConfigResolver
       ↓
DatabaseConfigNormalizer
       ↓
DatabaseConfigValidator
       ↓
DatabaseConfigCompiler
       ↓
DatabaseConfiguration
       ↓
Container
       ↓
Database Runtime
```

---

# 5. Config System ≠ Database Configuration

El sistema general de Config puede manejar:

```text
application
HTTP
cache
mail
queues
telemetry
database
```

pero Database deberá recibir únicamente su dominio:

```text
DatabaseConfiguration
```

---

# 6. Boundary

La frontera será conceptualmente:

```php
interface DatabaseConfigurationProvider
{
    public function configuration(): DatabaseConfiguration;
}
```

Database no deberá conocer cómo se obtuvo la configuración.

---

# 7. Configuración pública vs configuración interna

El desarrollador podrá utilizar una representación ergonómica:

```php
return [
    'default' => 'main',

    'connections' => [
        'main' => [
            'driver' => 'postgresql',
            'host' => env('DB_HOST', '127.0.0.1'),
            'port' => env('DB_PORT', 5432),
            'database' => env('DB_DATABASE', 'app'),
            'username' => env('DB_USERNAME'),
            'password' => secret('database.main.password'),
        ],
    ],
];
```

Pero Database internamente recibirá algo equivalente a:

```text
DatabaseConfiguration
└── ConnectionRegistryConfiguration
    └── ConnectionConfiguration("main")
```

---

# 8. Raw configuration

Se denomina `RawDatabaseConfiguration` a la estructura inmediatamente obtenida desde Config.

Podrá contener:

```text
strings
integers
booleans
arrays
environment references
secret references
extension configuration
```

Todavía no deberá considerarse válida.

---

# 9. Resolved configuration

Después de resolver referencias no sensibles apropiadas:

```text
Raw
 ↓
Resolved
```

pero una referencia a secreto podrá mantenerse sin materializar.

---

# 10. Normalized configuration

La normalización transforma diferentes representaciones válidas a una forma canónica.

Ejemplo:

```text
"pgsql"
"postgres"
"postgresql"
```

podrían normalizarse a:

```text
postgresql
```

si el sistema decide soportar esos aliases.

---

# 11. Alias normalization

Los aliases serán explícitos.

No deberán inferirse mediante coincidencias ambiguas.

---

# 12. Validated configuration

Una configuración validada garantiza propiedades estructurales y semánticas conocidas antes de iniciar Database.

Ejemplo:

```text
driver exists
port is valid
pool limits coherent
timeouts non-negative
default connection exists
```

---

# 13. Compiled configuration

La configuración compilada estará optimizada para consumo runtime.

Podrá:

```text
resolve aliases
canonicalize identifiers
precompute derived values
validate extension sections
assign configuration IDs
freeze collections
```

---

# 14. Typed configuration

El runtime no deberá depender de:

```php
$config['connections']['default']['options']['timeout'];
```

Preferirá:

```php
$connection->timeouts()->connect();
```

---

# 15. DatabaseConfiguration

Se propone conceptualmente:

```php
final readonly class DatabaseConfiguration
{
    public function __construct(
        public ConnectionRegistryConfiguration $connections,
        public OrmConfiguration $orm,
        public QueryConfiguration $query,
        public TransactionConfiguration $transactions,
        public CacheIntegrationConfiguration $cache,
        public TelemetryIntegrationConfiguration $telemetry,
        public SecurityConfiguration $security,
        public RuntimeDatabaseConfiguration $runtime,
    ) {}
}
```

La estructura definitiva podrá dividirse aún más.

---

# 16. Inmutabilidad

Después de compilación:

```text
DatabaseConfiguration
=
immutable
```

No deberá existir:

```php
$config->connections['main']['host'] = '...';
```

---

# 17. Configuration Registry

Las conexiones deberán almacenarse en un registry tipado:

```php
interface ConnectionConfigurationRegistry
{
    public function default(): ConnectionName;

    public function has(ConnectionName $name): bool;

    public function get(ConnectionName $name): ConnectionConfiguration;
}
```

---

# 18. ConnectionName

No deberá circular cualquier string sin validación cuando sea posible.

```php
final readonly class ConnectionName
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 19. ConnectionConfiguration

Conceptualmente:

```php
final readonly class ConnectionConfiguration
{
    public function __construct(
        public ConnectionName $name,
        public DriverId $driver,
        public DatabaseEndpointConfiguration $endpoint,
        public DatabaseCredentialReference $credentials,
        public ConnectionTimeoutConfiguration $timeouts,
        public ConnectionOptionSet $options,
    ) {}
}
```

---

# 20. Driver ≠ platform

La configuración deberá mantener:

```text
Driver
≠
Database Platform
```

Ejemplo:

```text
driver:
    pdo_pgsql

platform:
    postgresql
```

aunque una configuración simplificada pueda inferir el driver predeterminado.

---

# 21. Explicit driver selection

Configuración avanzada:

```php
'driver' => 'pdo_pgsql',
'platform' => 'postgresql',
```

---

# 22. Simplified configuration

Experiencia sencilla:

```php
'driver' => 'postgresql',
```

podrá resolverse a:

```text
platform = postgresql
driver = default driver for PostgreSQL
```

mediante política explícita.

---

# 23. Inference ≠ capability detection

Inferir:

```text
driver = pdo_pgsql
```

desde configuración no demuestra:

```text
server supports RETURNING
```

Las capacidades reales pertenecen al Capability System.

---

# 24. Platform version

Podrá configurarse:

```php
'server_version' => '17',
```

como hint o restricción.

Pero:

```text
Configured Version
≠
Observed Version
```

---

# 25. Version policy

Se propone:

```text
AUTO
DECLARED
VERIFY_DECLARED
```

---

# 26. AUTO

El sistema intenta obtener la versión efectiva.

---

# 27. DECLARED

Utiliza la versión declarada cuando el entorno no permite introspección o cuando así se configura.

---

# 28. VERIFY_DECLARED

Compara:

```text
configured version
vs
observed version
```

y falla o advierte ante discrepancia según policy.

---

# 29. Configured capability ≠ proven capability

Una configuración:

```php
'supports_returning' => true,
```

no deberá convertirse automáticamente en verdad arquitectónica.

---

# 30. Capability overrides

Si se permiten overrides deberán clasificarse:

```text
ASSUME
REQUIRE
DISABLE
```

y conservar evidencia de su origen.

---

# 31. REQUIRE capability

Ejemplo:

```php
'require_capabilities' => [
    'sql.returning',
],
```

significa:

```text
Database must demonstrate capability
```

no:

```text
force capability = true
```

---

# 32. DISABLE capability

Puede utilizarse para impedir una feature aunque el servidor la soporte.

---

# 33. ASSUME capability

Sólo para escenarios especializados y con diagnostics explícitos.

---

# 34. Variables de entorno

El Config System podrá utilizar:

```text
DB_HOST
DB_PORT
DB_DATABASE
DB_USERNAME
```

pero Database Core no deberá leerlas directamente.

---

# 35. Environment boundary

Correcto:

```text
Environment
    ↓
Config
    ↓
Typed Database Configuration
```

Incorrecto:

```text
ConnectionFactory
    ↓
getenv("DB_HOST")
```

---

# 36. Environment variable typing

Un valor:

```text
DB_PORT="5432"
```

deberá normalizarse a:

```text
int(5432)
```

antes de llegar al runtime.

---

# 37. Boolean normalization

Valores aceptados deberán definirse explícitamente.

No depender de:

```php
(bool) "false"
```

porque produce semántica incorrecta.

---

# 38. Null semantics

Debe distinguirse:

```text
missing
null
empty string
default
```

---

# 39. Missing ≠ null

Ejemplo:

```text
password missing
```

puede significar:

```text
use another credential provider
```

mientras:

```text
password = null
```

puede significar autenticación sin password si la plataforma lo soporta.

---

# 40. Secret architecture

Las credenciales no deberán tratarse como strings normales de configuración.

---

# 41. SecretReference

Ejemplo:

```php
'password' => secret('database.main.password'),
```

produce conceptualmente:

```text
SecretReference
```

no necesariamente el valor real.

---

# 42. Late secret resolution

```text
Config Compile
    ↓
SecretReference
    ↓
Connection Acquisition
    ↓
SecretProvider
    ↓
Credential Materialization
```

---

# 43. Why late resolution

Permite:

```text
secret rotation
reduced memory lifetime
external vault integration
worker longevity
credential refresh
```

---

# 44. Secret value ≠ configuration cache

Un configuration cache no deberá serializar secretos materializados salvo diseño explícito y seguro.

---

# 45. CredentialConfiguration

Se propone:

```text
CredentialConfiguration
├── UsernameReference
├── PasswordSecretReference
├── CertificateReference
├── TokenReference
└── AuthenticationMethod
```

según driver.

---

# 46. DSN security

Deberá evitarse almacenar:

```text
postgres://user:password@host/db
```

como representación interna cuando contenga secretos.

---

# 47. DSN parsing

Si el desarrollador proporciona URL/DSN:

```text
parse
↓
separate credentials
↓
redact original representation
↓
typed configuration
```

---

# 48. Connection URL

Podrá soportarse por ergonomía:

```php
'url' => env('DATABASE_URL'),
```

pero será una fuente de configuración, no la representación runtime final.

---

# 49. URL precedence

Si se combinan:

```text
URL
+
individual options
```

la precedencia deberá ser determinista y documentada.

---

# 50. Recommended precedence

Conceptualmente:

```text
Framework defaults
      ↓
Platform defaults
      ↓
Connection URL
      ↓
Connection explicit config
      ↓
Environment-specific override
      ↓
Testing override
```

aunque la política definitiva será establecida por Config.

---

# 51. Explicit ≠ implicit

Los valores explícitos deberán prevalecer normalmente sobre defaults inferidos.

---

# 52. Default connection

Debe existir:

```text
default connection name
```

si las APIs públicas dependen de ella.

---

# 53. Missing default

Si:

```text
DB::query(...)
```

requiere default y ninguno existe:

```text
MissingDefaultDatabaseConnectionException
```

---

# 54. Multiple connections

Ejemplo:

```php
'connections' => [
    'main' => [...],
    'analytics' => [...],
    'legacy' => [...],
],
```

---

# 55. Duplicate connection

Después de normalización:

```text
Main
main
```

no deberán convertirse silenciosamente en dos conexiones si los nombres son case-insensitive.

La política de identidad deberá definirse.

---

# 56. Connection inheritance

Podrá soportarse:

```php
'connections' => [
    'base-postgres' => [
        'driver' => 'postgresql',
        'port' => 5432,
    ],

    'main' => [
        'extends' => 'base-postgres',
        'database' => 'main',
    ],
],
```

si se considera útil.

---

# 57. Inheritance resolution

```text
Raw Definitions
    ↓
Inheritance Graph
    ↓
Cycle Detection
    ↓
Merge
    ↓
Normalization
```

---

# 58. Inheritance cycle

```text
A extends B
B extends A
```

deberá producir error de configuración.

---

# 59. Merge semantics

No se deberá utilizar un merge recursivo genérico sin semántica.

Cada sección definirá:

```text
replace
merge
append
forbid
```

---

# 60. Connection endpoint

Se propone:

```text
DatabaseEndpointConfiguration
├── Host
├── Port
├── Socket
├── DatabaseName
├── Charset
└── TransportOptions
```

---

# 61. Host vs socket

La configuración deberá permitir distinguir:

```text
TCP connection
Unix socket
in-memory SQLite
file SQLite
```

sin hacks de strings.

---

# 62. SQLite configuration

Ejemplo:

```php
'driver' => 'sqlite',
'database' => database_path('database.sqlite'),
```

---

# 63. SQLite memory

Deberá distinguirse:

```text
:memory:
```

de:

```text
temporary file
shared memory URI
persistent file
```

porque sus lifetimes son distintos.

---

# 64. MySQL and MariaDB

Aunque compartan protocolo/compatibilidad:

```text
mysql
≠
mariadb
```

en Platform Configuration.

---

# 65. PostgreSQL

Configuración podrá incluir:

```text
search_path
application_name
sslmode
timezone
```

cuando corresponda.

---

# 66. Driver-specific options

No deberán contaminar el contrato común.

---

# 67. Option namespaces

Ejemplo:

```php
'options' => [
    'common' => [...],
    'driver' => [...],
],
```

o una estructura tipada equivalente.

---

# 68. Common options

Podrán incluir:

```text
connect timeout
statement timeout
read timeout
application name
persistent/reuse policy
```

cuando sean semánticamente portables.

---

# 69. Driver options

Opciones específicas permanecerán bajo namespace del driver.

---

# 70. Unknown option

Una opción desconocida deberá:

```text
fail
warn
or be delegated to registered extension
```

según política.

No ignorarse silenciosamente por defecto.

---

# 71. Strict configuration mode

Se recomienda:

```text
strict = true
```

en producción y CI.

---

# 72. Unknown configuration key

Ejemplo:

```text
conection_timeout
```

en vez de:

```text
connection_timeout
```

deberá detectarse.

---

# 73. Suggestions

Diagnostics podrán sugerir:

```text
Unknown key "conection_timeout".
Did you mean "connection_timeout"?
```

---

# 74. Timeout configuration

Debe distinguirse:

```text
ConnectTimeout
AcquireTimeout
StatementTimeout
TransactionTimeout
LockTimeout
IdleTimeout
CancellationTimeout
```

---

# 75. Timeout ≠ one universal number

No:

```php
'timeout' => 30
```

como única semántica para todo el subsistema.

---

# 76. Duration type

Internamente:

```text
Duration
```

no strings arbitrarios.

---

# 77. Human-friendly duration

Config pública podría aceptar:

```text
500ms
5s
2m
```

y normalizarlo.

---

# 78. Invalid duration

```text
"fast"
```

deberá fallar durante validación.

---

# 79. Pool configuration

Ejemplo conceptual:

```php
'pool' => [
    'enabled' => true,
    'min' => 1,
    'max' => 20,
    'idle_timeout' => '60s',
    'acquire_timeout' => '5s',
],
```

---

# 80. Pool validation

Debe cumplirse:

```text
0 <= min <= max
```

---

# 81. Pool support

Configurar:

```text
pool.enabled = true
```

no significa que el runtime/driver lo soporte.

---

# 82. Pool requirement resolution

```text
Configuration
+
Runtime Capability
+
Driver Capability
→
Effective Pool Strategy
```

---

# 83. Unsupported required pool

Si pool es obligatorio:

```text
required = true
```

y no puede implementarse:

```text
boot failure
```

---

# 84. Optional pool

Si:

```text
enabled = auto
```

podrá degradarse a conexión no pooled con diagnostics.

---

# 85. Runtime configuration

Ejemplo:

```php
'runtime' => [
    'connection_reuse' => true,
    'scope_validation' => true,
    'reset_policy' => 'strict',
],
```

---

# 86. Runtime defaults

Deberán depender del Runtime Adapter, no de suposiciones globales.

---

# 87. FrankenPHP defaults

FrankenPHP podrá utilizar:

```text
persistent worker safe defaults
strict scope reset
connection reuse only when sanitized
```

---

# 88. RoadRunner/OpenSwoole

Podrán proporcionar perfiles de runtime propios.

---

# 89. Runtime profile

Se propone:

```text
DatabaseRuntimeProfile
```

que combine:

```text
runtime adapter defaults
+
application configuration
```

---

# 90. Configuration ≠ runtime state

Aunque se configure:

```text
connection_reuse = true
```

el runtime todavía debe decidir si una conexión concreta es reusable.

---

# 91. Retry configuration

Debe distinguirse:

```text
connection retry
transaction retry
deadlock retry
serialization retry
```

---

# 92. Generic retry anti-pattern

No:

```php
'retries' => 5
```

sin definir:

```text
what
when
boundary
idempotency requirements
```

---

# 93. Retry policy

Ejemplo:

```text
TransactionRetryPolicyConfiguration
├── maxAttempts
├── backoff
├── jitter
├── retryableCategories
└── unknownOutcomePolicy
```

---

# 94. UNKNOWN outcome

No podrá configurarse:

```text
retry_unknown_commit = true
```

como atajo inseguro sin mecanismo de idempotencia/reconciliation explícito.

---

# 95. Transaction configuration

Podrá incluir:

```text
default isolation
nested transaction policy
savepoint policy
retry policy
timeout
```

---

# 96. Isolation

Valores tipados:

```text
READ_UNCOMMITTED
READ_COMMITTED
REPEATABLE_READ
SERIALIZABLE
PLATFORM_DEFAULT
```

---

# 97. Requested isolation ≠ effective isolation

La configuración define lo solicitado.

El Transaction System determina lo efectivo.

---

# 98. ORM configuration

Ejemplo:

```php
'orm' => [
    'enabled' => true,
    'metadata' => [...],
    'lazy_loading' => 'warn',
    'change_tracking' => 'snapshot',
],
```

---

# 99. ORM disabled

Database Query Engine deberá poder funcionar sin ORM si:

```text
orm.enabled = false
```

---

# 100. Metadata configuration

Podrá definir:

```text
entity paths
mapping drivers
compiled metadata cache
naming strategy
proxy/lazy strategy
```

según arquitectura final.

---

# 101. Entity discovery

No deberá repetirse por request.

---

# 102. Metadata compile

```text
Config
    ↓
Entity discovery
    ↓
Metadata compile
    ↓
Frozen metadata
```

durante bootstrap/build cuando sea posible.

---

# 103. Query configuration

Podrá incluir:

```text
compiled query cache
optimizer
planner
raw expression policy
default limits
diagnostic mode
```

---

# 104. Optimizer rules

La configuración podrá:

```text
enable
disable
configure
```

reglas registradas.

No crear reglas inexistentes mediante strings arbitrarios.

---

# 105. Schema configuration

Podrá incluir:

```text
default schema
introspection policy
diff safety
identifier strategy
```

---

# 106. Migration configuration

Podrá incluir:

```text
paths
table/repository
locking
batch policy
safety policy
zero-downtime policy
```

---

# 107. Migration credentials

Podrán utilizar credenciales diferentes de runtime.

---

# 108. Least privilege

Ejemplo:

```text
runtime connection
→ application DB role

migration connection
→ schema migration role

backup connection
→ backup role

admin connection
→ administrative role
```

---

# 109. Role configuration

No deberá implicar necesariamente conexiones duplicadas manualmente.

Podrá utilizar:

```text
CredentialProfile
```

sobre un mismo endpoint lógico.

---

# 110. Read/write configuration

Ejemplo:

```php
'read_write' => [
    'writer' => 'primary',
    'replicas' => [
        'replica-1',
        'replica-2',
    ],
],
```

---

# 111. Routing configuration

Podrá incluir:

```text
read preference
sticky policy
lag policy
load balancing policy
failover policy
```

---

# 112. Configuration ≠ routing decision

La configuración define policy.

El Router decide por operación.

---

# 113. Replica configuration

Cada replica deberá ser una entidad de configuración explícita.

---

# 114. Replica weight

Podrá configurarse:

```text
weight
```

pero sólo será considerado después de verificar elegibilidad.

---

# 115. Weight ≠ health

Una replica con peso 100 y no saludable:

```text
not eligible
```

---

# 116. Sharding configuration

Podrá definir:

```text
shard map
routing strategy
shard key strategy
topology generation
```

---

# 117. Static vs dynamic topology

Debe distinguirse:

```text
STATIC_CONFIGURATION
DYNAMIC_PROVIDER
```

---

# 118. Dynamic topology

Config sólo deberá indicar:

```text
provider
credentials/reference
refresh policy
```

No almacenar necesariamente todo el mapa.

---

# 119. Multitenancy integration

Como Multitenancy es un paquete oficial opcional:

```text
Database Core
```

no dependerá de su configuración.

---

# 120. Multitenancy bridge

Cuando esté instalado:

```text
Multitenancy Config
        ↓
Database Multitenancy Adapter
        ↓
Tenant Connection Resolution
```

---

# 121. Tenant config ≠ global Database config

Datos específicos de tenant pueden provenir dinámicamente de otro provider.

---

# 122. Tenant database credentials

No deberán compilarse todas necesariamente en el container global.

---

# 123. Tenant connection template

Podrá existir:

```text
ConnectionTemplate
+
Tenant-specific parameters
→
Resolved Tenant ConnectionConfiguration
```

---

# 124. Template security

Sólo campos explícitamente permitidos podrán sustituirse.

---

# 125. Cache integration configuration

Ejemplo:

```php
'cache' => [
    'metadata' => true,
    'compiled_queries' => true,
    'results' => false,
    'entities' => false,
],
```

---

# 126. Cache provider

Database deberá referenciar:

```text
CacheProviderId
```

o adapter lógico.

No construir directamente Redis/Memcached.

---

# 127. Cache semantics

Config podrá seleccionar:

```text
provider
namespace
TTL defaults
consistency policy
invalidation strategy
```

pero no redefinir semántica fundamental.

---

# 128. Telemetry configuration

Ejemplo:

```php
'telemetry' => [
    'enabled' => true,
    'tracing' => true,
    'metrics' => true,
    'query_parameters' => false,
],
```

---

# 129. Safe defaults

Por defecto:

```text
raw query parameters
=
not exported
```

---

# 130. SQL recording

Si se habilita SQL para debugging:

```text
redaction
sampling
environment policy
```

deberán aplicarse.

---

# 131. Debug configuration

Debug no deberá ser una única bandera que desactive seguridad.

---

# 132. Security configuration

Podrá incluir:

```text
raw SQL policy
credential policy
TLS policy
sensitive data policy
audit policy
identifier policy
```

---

# 133. Secure defaults

Ejemplo:

```text
parameter binding required
raw expressions restricted
credentials redacted
TLS verification enabled when TLS required
```

---

# 134. Environment-specific security downgrade

Si desarrollo permite una configuración menos estricta, deberá ser explícita.

No inferirse únicamente porque:

```text
APP_ENV=local
```

---

# 135. Production guard

Podrá existir:

```text
DatabaseProductionConfigurationGuard
```

que detecte:

```text
debug SQL parameter logging
disabled TLS verification
unsafe raw SQL policy
admin credentials as runtime credentials
```

---

# 136. Warning vs failure

Cada regla podrá clasificarse:

```text
INFO
WARNING
ERROR
FATAL
```

---

# 137. Config validation architecture

Se propone:

```text
Structural Validation
        ↓
Semantic Validation
        ↓
Cross-section Validation
        ↓
Extension Validation
        ↓
Environment/Security Validation
```

---

# 138. Structural validation

Ejemplos:

```text
required key
type
enum
range
format
```

---

# 139. Semantic validation

Ejemplo:

```text
pool.min <= pool.max
```

---

# 140. Cross-section validation

Ejemplo:

```text
result cache enabled
+
no cache provider
→ invalid
```

---

# 141. Extension validation

Un custom driver podrá proporcionar su propio schema de configuración.

---

# 142. Security validation

Ejemplo:

```text
production
+
TLS required
+
verify_peer = false
→ fatal
```

según policy.

---

# 143. Validation error model

Se propone:

```text
DatabaseConfigurationViolation
├── path
├── code
├── severity
├── message
├── expected
├── actualSafeDescription
└── suggestion
```

---

# 144. Configuration path

Ejemplo:

```text
database.connections.main.pool.max
```

---

# 145. No secret actual values

Error:

```text
password must not be empty; actual="hunter2"
```

prohibido.

---

# 146. Safe description

Usar:

```text
password reference resolved to empty secret
```

sin valor.

---

# 147. Multiple errors

El validator deberá intentar reportar múltiples errores independientes en una ejecución.

---

# 148. Fail fast vs aggregate

Errores estructurales críticos pueden impedir validaciones posteriores, pero deberán agregarse cuando sea seguro.

---

# 149. Example diagnostics

```text
DATABASE CONFIGURATION INVALID

database.connections.main.port
  Expected: integer between 1 and 65535
  Received: "postgres"

database.connections.main.pool
  min (20) cannot exceed max (10)

database.default
  References unknown connection "production"
```

---

# 150. Configuration schema

Database podrá publicar un schema machine-readable.

Por ejemplo conceptualmente:

```text
DatabaseConfigurationSchema
```

---

# 151. Schema uses

Permitirá:

```text
IDE completion
validation
documentation generation
CLI inspection
config migration
plugin integration
```

---

# 152. Schema ≠ database schema

Debe mantenerse:

```text
Configuration Schema
≠
Database Schema
```

---

# 153. Extension configuration schema

Cada extensión podrá registrar:

```text
ExtensionConfigSchema
```

bajo su namespace.

---

# 154. Namespace ownership

Ejemplo:

```text
database.extensions.acme_vector
```

sólo podrá ser interpretado por esa extensión.

---

# 155. Unknown extension

Configuración para una extensión no instalada deberá:

```text
fail
```

por defecto en strict mode.

---

# 156. Config compilation

El compilador podrá producir:

```text
CompiledDatabaseConfiguration
```

optimizada.

---

# 157. Compilation steps

```text
Read
 ↓
Resolve
 ↓
Normalize
 ↓
Validate
 ↓
Canonicalize
 ↓
Compile
 ↓
Freeze
```

---

# 158. Configuration ID

Cada configuración compilada podrá obtener:

```text
DatabaseConfigurationId
```

---

# 159. Fingerprint

Además:

```text
DatabaseConfigurationFingerprint
```

para detectar cambios estructurales.

---

# 160. Secret exclusion

El fingerprint no deberá incluir secretos en claro.

---

# 161. Secret reference fingerprint

Podrá incluir:

```text
secret reference identity
```

pero no el valor.

---

# 162. Configuration generation

Se define:

```text
DatabaseConfigurationGeneration
```

para identificar una versión activa.

---

# 163. Generation consistency

Una operación iniciada con:

```text
Generation G1
```

deberá usar configuración coherente de G1 durante toda su ejecución.

---

# 164. No mid-request mutation

No:

```text
Request
  uses G1
  ↓
config reload
  ↓
same request uses G2
```

---

# 165. Reload model

```text
Current G1
    ↓
Detect change
    ↓
Build G2
    ↓
Normalize
    ↓
Validate
    ↓
Compile
    ↓
Freeze
    ↓
Publish G2
```

---

# 166. Existing operations

Continúan con:

```text
G1
```

hasta finalizar.

---

# 167. New operations

Usan:

```text
G2
```

---

# 168. Resource implications

Cambiar:

```text
connection endpoint
credentials
pool size
TLS policy
driver
```

puede requerir reconfiguración de infraestructura.

---

# 169. Configuration swap ≠ resource mutation

No deberá mutarse un pool activo de forma insegura sólo porque cambió Config.

---

# 170. Resource generation

Podrá existir:

```text
ConnectionInfrastructureGeneration
```

relacionada con Config Generation.

---

# 171. Graceful rotation

Ejemplo:

```text
G1 pool
    ↓ draining

G2 pool
    ↓ active
```

---

# 172. Worker recycle

Algunos cambios podrán requerir:

```text
worker restart
```

en lugar de hot reload.

---

# 173. Reload classification

Se propone:

```text
LIVE_RELOADABLE
DRAIN_AND_REPLACE
NEW_SCOPE_ONLY
WORKER_RESTART_REQUIRED
APPLICATION_RESTART_REQUIRED
```

---

# 174. Example classifications

Cambiar:

```text
slow query threshold
```

podría ser:

```text
NEW_SCOPE_ONLY
```

Cambiar:

```text
driver implementation
```

podría requerir:

```text
WORKER_RESTART_REQUIRED
```

---

# 175. Configuration Diff

Podrá existir:

```text
DatabaseConfigurationDiff
```

---

# 176. Diff categories

```text
connections
credentials references
pool
ORM
cache
telemetry
security
runtime
extensions
```

---

# 177. Config diff ≠ schema diff

Son sistemas independientes.

---

# 178. Configuration cache

VoltStack podrá cachear:

```text
normalized/compiled configuration
```

para acelerar bootstrap.

---

# 179. Config cache safety

El cache deberá invalidarse cuando cambien:

```text
configuration sources
framework version
database module version
extension set
config schema version
```

---

# 180. Secrets in cache

Por defecto:

```text
No Materialized Secrets
```

---

# 181. Cache artifact

Podría almacenar código PHP generado:

```php
return new CompiledDatabaseConfiguration(...);
```

o formato equivalente.

---

# 182. Configuration cache ≠ Database cache

Debe distinguirse:

```text
Framework Config Cache
≠
Database Query Cache
≠
Result Cache
≠
Metadata Cache
```

---

# 183. Build-time config

VoltStack podrá precompilar configuración durante:

```text
deployment/build
```

cuando los valores sean conocidos.

---

# 184. Runtime config

Algunos valores sólo estarán disponibles durante ejecución:

```text
secret values
dynamic tenant endpoints
service discovery
temporary credentials
```

---

# 185. Static vs dynamic configuration

Debe distinguirse:

```text
StaticConfiguration
DynamicConfigurationProvider
```

---

# 186. Dynamic configuration restrictions

No todo podrá ser dinámico.

Ejemplo:

```text
driver class
```

normalmente será estructural.

Mientras:

```text
temporary password
```

puede ser dinámico.

---

# 187. Dynamic provider

Conceptualmente:

```php
interface DynamicDatabaseConfigurationProvider
{
    public function resolve(
        DynamicConfigurationRequest $request,
    ): DynamicConfigurationFragment;
}
```

---

# 188. Dynamic provider ≠ arbitrary config override

Sólo podrá modificar campos autorizados.

---

# 189. Allowlist

Cada template/configuración declarará qué campos pueden resolverse dinámicamente.

---

# 190. Config source provenance

Cada valor deberá poder conservar, cuando sea útil, su procedencia:

```text
default
config file
environment
secret reference
extension
runtime provider
test override
```

---

# 191. Provenance ≠ secret exposure

Se puede indicar:

```text
source: secret-provider:vault
```

sin mostrar el secreto.

---

# 192. Provenance diagnostics

Ejemplo:

```text
database.connections.main.host
value: db.internal
source: environment:DB_HOST
```

---

# 193. Secret diagnostics

Ejemplo:

```text
database.connections.main.password
value: [REDACTED]
source: secret:database/main/password
```

---

# 194. CLI inspection

Podrá existir:

```bash
php voltstack database:config
```

---

# 195. Safe output

Ejemplo:

```text
Default connection: main

main
  platform: postgresql
  driver: pdo_pgsql
  host: db.internal
  port: 5432
  database: app
  username: [REDACTED/POLICY]
  password: [SECRET REFERENCE]
  pool: enabled
```

---

# 196. CLI validate

```bash
php voltstack database:config --validate
```

---

# 197. CLI source

```bash
php voltstack database:config --sources
```

deberá respetar redaction.

---

# 198. CLI effective

```bash
php voltstack database:config --effective
```

podrá mostrar configuración efectiva no sensible.

---

# 199. Config debug

Debug deberá diferenciar:

```text
configured
normalized
effective
observed
```

---

# 200. Example

```text
Configured platform version: auto
Observed platform version: 17.2

Configured pool max: 20
Effective pool max: 20

Configured TLS: required
Observed TLS: active
```

---

# 201. Configured ≠ observed

Esta distinción será fundamental.

---

# 202. Effective configuration

Se define:

```text
EffectiveConfiguration
=
CompiledConfiguration
+
Resolved Runtime Policy
+
Capability Decisions
```

pero deberá evitarse mezclarlo todo en un único mutable object.

---

# 203. Runtime policy objects

Preferir:

```text
ConnectionPolicy
TransactionPolicy
CachePolicy
SecurityPolicy
```

derivados de Config + capabilities cuando sea necesario.

---

# 204. Config should not query database

El proceso de normalización/validación estructural no deberá necesitar DB.

---

# 205. Runtime verification

Validaciones que requieren servidor real pertenecerán a:

```text
Database readiness
capability discovery
connection verification
```

---

# 206. Validation levels

Se propone:

```text
STATIC
RUNTIME
CONNECTIVITY
OPERATIONAL
```

---

# 207. STATIC

Sin I/O externo.

---

# 208. RUNTIME

Comprueba compatibilidad con runtime/driver instalados.

---

# 209. CONNECTIVITY

Abre conexión cuando se solicita explícitamente.

---

# 210. OPERATIONAL

Comprueba propiedades como permisos/capabilities.

---

# 211. Boot behavior

Por defecto:

```text
STATIC + RUNTIME
```

podrán ejecutarse en boot.

---

# 212. Connectivity behavior

Se ejecutará en:

```text
readiness check
health check
explicit warmup
CLI validate --connect
```

---

# 213. Testing configuration

Tests deberán poder construir configuración sin `.env`.

---

# 214. Test builder

Ejemplo:

```php
$config = DatabaseTestConfiguration::builder()
    ->sqliteMemory()
    ->strict()
    ->build();
```

---

# 215. Test overrides

Podrán aplicarse antes de compilation/freeze.

---

# 216. Test override ≠ production runtime mutation

Los tests crean una nueva generación/container.

---

# 217. Configuration fixtures

Podrán existir fixtures para:

```text
minimal valid config
invalid config
pool config
replica config
sharded config
TLS config
multi-connection config
```

---

# 218. Config fuzz testing

El parser/normalizer podrá someterse a:

```text
property-based tests
fuzz tests
```

para valores arbitrarios.

---

# 219. Extension testing

Cada plugin deberá demostrar:

```text
valid configuration accepted
invalid configuration rejected
unknown keys rejected
secrets redacted
defaults deterministic
```

---

# 220. Backward compatibility

Cambios en nombres/config deberán seguir:

```text
322_DATABASE_BACKWARD_COMPATIBILITY_SYSTEM
323_DATABASE_VERSIONING_SYSTEM
324_DATABASE_DEPRECATION_POLICY
```

---

# 221. Deprecated config key

Ejemplo:

```text
database.pool_size
```

migrado a:

```text
database.connections.main.pool.max
```

---

# 222. Deprecation handling

Podrá:

```text
accept old key
normalize to new representation
emit deprecation
```

durante ventana de compatibilidad.

---

# 223. Old + new key

Si ambos aparecen:

```text
explicit conflict
```

o precedencia documentada.

Preferible error si son contradictorios.

---

# 224. Config version

Podrá existir:

```text
DatabaseConfigurationSchemaVersion
```

---

# 225. Config migration

Una herramienta futura podrá transformar:

```text
schema v1
→
schema v2
```

---

# 226. Config code generation

`310_DATABASE_CODE_GENERATION_SYSTEM.md` podrá generar:

```text
database config templates
connection templates
IDE schemas
```

---

# 227. Public API relation

El usuario podrá consultar configuración pública limitada.

Pero no deberá obtener fácilmente secretos.

---

# 228. Runtime configuration API

Ejemplo:

```php
DB::connection('main')->configuration();
```

deberá devolver una vista segura o descriptor, no credenciales materializadas.

---

# 229. Safe descriptor

```text
ConnectionDescriptor
├── name
├── platform
├── endpoint safe representation
├── capabilities
└── runtime policy
```

---

# 230. Config logging

Nunca:

```php
logger()->debug($databaseConfig);
```

si puede serializar secretos.

---

# 231. Secret-aware serialization

Objetos sensibles deberán implementar comportamiento equivalente a:

```text
redacted debug output
```

---

# 232. `__toString()` safety

SecretReference no deberá devolver el secreto desde:

```php
(string) $secret;
```

---

# 233. Exception safety

Las excepciones de configuración deberán usar:

```text
safe values
```

---

# 234. Configuration telemetry

Métricas posibles:

```text
database.config.validation.duration
database.config.validation.failures
database.config.reload.count
database.config.reload.failures
database.config.generation
```

con cardinalidad controlada.

---

# 235. Generation IDs in metrics

No usar hashes arbitrariamente largos/high-cardinality como labels.

---

# 236. Configuration events

Eventos internos:

```text
DatabaseConfigurationLoaded
DatabaseConfigurationValidated
DatabaseConfigurationCompiled
DatabaseConfigurationReloaded
DatabaseConfigurationRejected
```

---

# 237. Config events ≠ secret payload

Los eventos no transportarán valores sensibles.

---

# 238. Persistent runtime

En workers persistentes:

```text
Config
```

no podrá asumirse que se recarga automáticamente por request.

---

# 239. FrankenPHP

Modelo recomendado:

```text
Worker boot
    ↓
load compiled config G1
    ↓
requests use G1
```

---

# 240. Config change

Puede provocar:

```text
worker reload
```

o generación nueva según política.

---

# 241. Safe default

En producción inicial:

> Los cambios estructurales de Database Configuration deberán favorecer worker recycle sobre hot mutation compleja.

---

# 242. Future hot reload

La arquitectura de generaciones permitirá agregarlo posteriormente sin convertir Config en mutable global state.

---

# 243. RoadRunner

Misma regla:

```text
worker lifetime
≠
request lifetime
```

---

# 244. OpenSwoole

Concurrent requests deberán ver una generación consistente.

---

# 245. Atomic reference

El runtime podrá mantener conceptualmente:

```text
CurrentConfigurationGeneration
```

como referencia atómica/read-only.

---

# 246. Existing scope capture

Al crear el OperationScope:

```text
scope.configurationGeneration = currentGeneration
```

---

# 247. Cross-generation prohibition

Un mismo DatabaseContext no deberá combinar componentes incompatibles de diferentes generaciones.

---

# 248. Container integration

`312_DATABASE_CONTAINER_INTEGRATION_SYSTEM.md` registrará:

```text
DatabaseConfiguration
```

como:

```text
SHARED IMMUTABLE
```

por generación.

---

# 249. Config provider

El provider puede ser shared.

---

# 250. Secret provider

El SecretProvider puede ser shared si es seguro.

---

# 251. Resolved credential

Una credencial materializada tendrá lifetime mucho más corto.

---

# 252. No singleton credential

No convertir:

```text
ResolvedPassword
```

en singleton de Container.

---

# 253. Configuration dependency graph

```text
VoltStack Config
      ↓
DatabaseConfigAdapter
      ↓
RawDatabaseConfiguration
      ↓
Resolver
      ↓
Normalizer
      ↓
Validator
      ↓
Compiler
      ↓
DatabaseConfiguration
      ↓
Container
      ↓
Database Components
```

---

# 254. Proposed namespaces

```text
VoltStack\Quantum\Database\Config
├── Contract
├── Model
├── Resolver
├── Normalizer
├── Validation
├── Compiler
├── Schema
├── Secret
├── Reload
└── Diagnostics
```

---

# 255. Proposed directory structure

```text
src/Quantum/Database/Config/
├── DatabaseConfiguration.php
├── DatabaseConfigurationProvider.php
├── DatabaseConfigurationId.php
├── DatabaseConfigurationGeneration.php
├── DatabaseConfigurationFingerprint.php
│
├── Connection/
│   ├── ConnectionConfiguration.php
│   ├── ConnectionConfigurationRegistry.php
│   ├── ConnectionName.php
│   ├── EndpointConfiguration.php
│   ├── CredentialConfiguration.php
│   ├── PoolConfiguration.php
│   └── TimeoutConfiguration.php
│
├── ORM/
├── Query/
├── Transaction/
├── Cache/
├── Telemetry/
├── Security/
├── Runtime/
│
├── Resolver/
│   ├── DatabaseConfigResolver.php
│   ├── EnvironmentConfigResolver.php
│   └── SecretReferenceResolver.php
│
├── Normalizer/
│   └── DatabaseConfigNormalizer.php
│
├── Validation/
│   ├── DatabaseConfigValidator.php
│   ├── DatabaseConfigurationViolation.php
│   └── DatabaseProductionConfigurationGuard.php
│
├── Compiler/
│   └── DatabaseConfigCompiler.php
│
├── Schema/
│   ├── DatabaseConfigurationSchema.php
│   └── ExtensionConfigSchema.php
│
├── Reload/
│   ├── DatabaseConfigurationDiff.php
│   ├── ReloadClassification.php
│   └── DatabaseConfigurationGenerationManager.php
│
└── Diagnostics/
    └── DatabaseConfigurationDiagnostics.php
```

---

# 256. Exception hierarchy

Se propone:

```text
DatabaseConfigurationException
├── InvalidDatabaseConfigurationException
├── MissingDatabaseConfigurationException
├── UnknownDatabaseConfigurationKeyException
├── UnknownConnectionConfigurationException
├── MissingDefaultConnectionException
├── ConfigurationInheritanceCycleException
├── UnsupportedConfigurationException
├── ConfigurationConflictException
├── SecretReferenceConfigurationException
├── ConfigurationGenerationMismatchException
├── ConfigurationReloadException
└── UnsafeProductionDatabaseConfigurationException
```

---

# 257. Configuration invariants

## DB-CONFIG-001

Database Core no leerá `.env` directamente.

## DB-CONFIG-002

Database Core no utilizará `$_ENV` directamente.

## DB-CONFIG-003

Database Core no utilizará `getenv()` directamente.

## DB-CONFIG-004

Runtime Database consumirá configuración tipada.

## DB-CONFIG-005

Configuración compilada será inmutable.

## DB-CONFIG-006

Raw config ≠ validated config.

## DB-CONFIG-007

Configured value ≠ observed runtime value.

## DB-CONFIG-008

Configured capability ≠ proven capability.

## DB-CONFIG-009

Secret ≠ normal config value.

## DB-CONFIG-010

Configuration generation será explícita.

---

# 258. Connection invariants

## DB-CONFIG-011

Default connection deberá referenciar una conexión existente.

## DB-CONFIG-012

Connection names serán normalizados determinísticamente.

## DB-CONFIG-013

Driver ≠ Platform.

## DB-CONFIG-014

MySQL ≠ MariaDB.

## DB-CONFIG-015

Configured version ≠ observed version.

## DB-CONFIG-016

Unknown connection option no será ignorada silenciosamente en strict mode.

## DB-CONFIG-017

Endpoint configuration será tipada.

## DB-CONFIG-018

Connection URL será fuente, no modelo interno final.

## DB-CONFIG-019

Credential materialization será separada de endpoint configuration.

## DB-CONFIG-020

Multiple connections serán first-class.

---

# 259. Secret invariants

## DB-CONFIG-021

Secret materializado no será almacenado en config cache por defecto.

## DB-CONFIG-022

SecretReference no expondrá su valor mediante string conversion.

## DB-CONFIG-023

Diagnostics no mostrarán secretos.

## DB-CONFIG-024

Exceptions no mostrarán secretos.

## DB-CONFIG-025

Telemetry no exportará secretos.

## DB-CONFIG-026

Config events no transportarán secretos.

## DB-CONFIG-027

DSNs sensibles serán redacted.

## DB-CONFIG-028

Secret resolution podrá ocurrir late.

## DB-CONFIG-029

Secret rotation no requerirá necesariamente recompilar configuración.

## DB-CONFIG-030

Credential lifetime será menor o igual al necesario para su operación.

---

# 260. Pool/runtime invariants

## DB-CONFIG-031

Pool configuration ≠ pool runtime state.

## DB-CONFIG-032

`min <= max`.

## DB-CONFIG-033

Configured pool support ≠ actual support.

## DB-CONFIG-034

Connection reuse sólo ocurrirá si runtime/driver lo permiten.

## DB-CONFIG-035

Connection reuse configuration no evitará reset/sanitization.

## DB-CONFIG-036

Runtime defaults podrán variar por adapter.

## DB-CONFIG-037

FrankenPHP tendrá persistent-runtime-safe defaults.

## DB-CONFIG-038

RoadRunner podrá proporcionar perfil propio.

## DB-CONFIG-039

OpenSwoole podrá proporcionar perfil coroutine-aware.

## DB-CONFIG-040

Runtime config ≠ operation runtime state.

---

# 261. Transaction invariants

## DB-CONFIG-041

Configured isolation ≠ effective isolation.

## DB-CONFIG-042

Retry policy deberá indicar boundary.

## DB-CONFIG-043

UNKNOWN commit no será convertido en retryable por simple flag.

## DB-CONFIG-044

Nested transaction policy será explícita.

## DB-CONFIG-045

Savepoint policy será independiente.

## DB-CONFIG-046

Timeouts tendrán semántica específica.

## DB-CONFIG-047

Connect timeout ≠ statement timeout.

## DB-CONFIG-048

Lock timeout ≠ transaction timeout.

## DB-CONFIG-049

Retry attempts serán acotados.

## DB-CONFIG-050

Backoff configuration será validada.

---

# 262. Security invariants

## DB-CONFIG-051

Secure defaults prevalecerán.

## DB-CONFIG-052

Debug mode no desactivará seguridad automáticamente.

## DB-CONFIG-053

Production guard detectará configuraciones peligrosas.

## DB-CONFIG-054

TLS required + verification disabled deberá ser explícitamente tratado como unsafe.

## DB-CONFIG-055

Runtime credentials no serán admin credentials por defecto.

## DB-CONFIG-056

Raw SQL policy será explícita.

## DB-CONFIG-057

Sensitive logging será disabled por defecto.

## DB-CONFIG-058

Unknown security key fallará en strict mode.

## DB-CONFIG-059

Environment name no será prueba suficiente de seguridad.

## DB-CONFIG-060

Configuration convenience no justificará secret exposure.

---

# 263. Generation invariants

## DB-CONFIG-061

Una operación utilizará una generación coherente.

## DB-CONFIG-062

No habrá mid-operation config mutation.

## DB-CONFIG-063

Nueva generación será validada antes de publicación.

## DB-CONFIG-064

Generación inválida no reemplazará generación activa.

## DB-CONFIG-065

Structural reload podrá requerir worker recycle.

## DB-CONFIG-066

Existing operations podrán terminar con generación anterior.

## DB-CONFIG-067

New operations utilizarán generación publicada.

## DB-CONFIG-068

Config fingerprint excluirá secrets.

## DB-CONFIG-069

Cross-generation component mixing será detectado cuando sea relevante.

## DB-CONFIG-070

Hot reload no será requisito de V1 para todos los campos.

---

# 264. Extension invariants

## DB-CONFIG-071

Custom driver podrá registrar config schema.

## DB-CONFIG-072

Extension config permanecerá namespaced.

## DB-CONFIG-073

Unknown extension config fallará en strict mode.

## DB-CONFIG-074

Extension defaults serán deterministas.

## DB-CONFIG-075

Extension validator no ejecutará queries durante static validation.

## DB-CONFIG-076

Extension config no podrá redefinir core config arbitrariamente.

## DB-CONFIG-077

Extension schema version será identificable.

## DB-CONFIG-078

Plugin removal invalidará config cache relevante.

## DB-CONFIG-079

Plugin installation invalidará config compilation relevante.

## DB-CONFIG-080

Extension secret fields deberán declararse como sensibles.

---

# 265. Developer experience invariants

## DB-CONFIG-081

Errores señalarán config path.

## DB-CONFIG-082

Errores indicarán expected type cuando sea seguro.

## DB-CONFIG-083

Typos podrán generar sugerencias.

## DB-CONFIG-084

Múltiples errores independientes podrán agregarse.

## DB-CONFIG-085

CLI inspection aplicará redaction.

## DB-CONFIG-086

IDE schema podrá generarse.

## DB-CONFIG-087

Defaults deberán ser inspeccionables.

## DB-CONFIG-088

Config provenance podrá inspeccionarse de forma segura.

## DB-CONFIG-089

Public config API no expondrá credenciales.

## DB-CONFIG-090

Configuration complexity deberá permanecer progresiva.

---

# 266. Testing invariants

## DB-CONFIG-091

Tests no dependerán obligatoriamente de `.env`.

## DB-CONFIG-092

Minimal valid configuration tendrá fixture.

## DB-CONFIG-093

Invalid configurations tendrán fixtures.

## DB-CONFIG-094

Secrets deberán probarse contra accidental serialization.

## DB-CONFIG-095

Config cache deberá probar invalidation.

## DB-CONFIG-096

Generation swap deberá probarse.

## DB-CONFIG-097

Concurrent scopes deberán mantener generación coherente.

## DB-CONFIG-098

Deprecated keys deberán probar normalización.

## DB-CONFIG-099

Unknown keys deberán probar strict mode.

## DB-CONFIG-100

Platform-specific config deberá probarse independientemente.

---

# 267. Anti-pattern: Database leyendo `.env`

```php
$host = env('DB_HOST');
```

dentro de:

```text
ConnectionFactory
```

Incorrecto.

---

# 268. Anti-pattern: arrays por todo el sistema

```php
$options = $config['database']['connections'][$name];
```

en múltiples subsistemas.

Debe sustituirse por configuración tipada.

---

# 269. Anti-pattern: mutable configuration singleton

```php
$config->set('database.host', 'other');
```

durante un request.

Prohibido para la configuración compilada.

---

# 270. Anti-pattern: password in DSN logs

```text
Connecting to:
postgres://admin:secret123@db.internal/app
```

Prohibido.

---

# 271. Anti-pattern: one timeout

```php
'timeout' => 30,
```

utilizado simultáneamente para:

```text
connect
query
transaction
lock
pool acquire
```

Incorrecto.

---

# 272. Anti-pattern: capability by config

```php
'supports_returning' => true,
```

y asumir que el DBMS lo soporta.

Incorrecto.

---

# 273. Anti-pattern: platform by PHP extension

```text
pdo_mysql installed
→ platform is MySQL
```

Incorrecto.

---

# 274. Anti-pattern: MySQL = MariaDB

Una configuración común puede compartir defaults, pero deberán conservar identidad independiente.

---

# 275. Anti-pattern: pool enabled means safe

```text
pool.enabled = true
→ reuse every connection
```

Incorrecto.

La conexión debe ser reusable y sanitizada.

---

# 276. Anti-pattern: debug disables security

```php
if ($debug) {
    $tlsVerify = false;
    $logParameters = true;
}
```

Prohibido como comportamiento implícito.

---

# 277. Anti-pattern: silent typo

```text
statment_timeout
```

ignorado y usando default.

En strict mode deberá fallar.

---

# 278. Anti-pattern: auto blessing new baseline config

Cambiar configuración de producción automáticamente para adaptarse a una capability observada no deberá ocurrir sin policy.

---

# 279. Anti-pattern: all tenant credentials in global config

No es escalable ni seguro para multitenancy dinámico.

---

# 280. Anti-pattern: config reload mutates active objects

```text
Config changed
→ mutate current EntityManager
```

Incorrecto.

---

# 281. Anti-pattern: secret fingerprint

Nunca:

```text
hash(password)
```

como fingerprint general de configuración si puede facilitar correlación o ataques innecesarios.

---

# 282. Anti-pattern: test config accepted as production

Una configuración diseñada para:

```text
SQLite :memory:
fake credentials
disabled TLS
```

deberá ser detectada cuando se utilice accidentalmente en producción si existe evidencia suficiente.

---

# 283. Testing architecture

Se propone:

```text
tests/Quantum/Database/Config/
├── DatabaseConfigurationTest.php
├── DatabaseConfigResolverTest.php
├── DatabaseConfigNormalizerTest.php
├── DatabaseConfigValidatorTest.php
├── DatabaseConfigCompilerTest.php
├── ConnectionConfigurationTest.php
├── EnvironmentResolutionTest.php
├── SecretReferenceTest.php
├── ConfigurationRedactionTest.php
├── ConfigurationCacheTest.php
├── ConfigurationGenerationTest.php
├── ConfigurationReloadTest.php
├── ConfigurationDiffTest.php
├── ExtensionConfigTest.php
├── ProductionConfigurationGuardTest.php
├── DeprecatedConfigurationTest.php
├── DatabaseUrlConfigurationTest.php
├── PoolConfigurationTest.php
├── TimeoutConfigurationTest.php
├── RuntimeConfigurationTest.php
└── PersistentWorkerConfigurationTest.php
```

---

# 284. Unit testing

Unit tests deberán cubrir:

```text
normalization
validation
merging
inheritance
duration parsing
enum parsing
defaults
fingerprinting
diffing
reload classification
redaction
```

sin DB real.

---

# 285. Integration testing

Integration tests deberán verificar:

```text
Config
→ Container
→ Driver
→ Connection
```

cuando la propiedad lo requiera.

---

# 286. Secret integration testing

Con proveedores reales o test providers controlados:

```text
SecretReference
→ SecretProvider
→ Credential
→ Connection
```

sin registrar el valor.

---

# 287. Platform matrix

Configuraciones específicas deberán validarse para:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

independientemente.

---

# 288. Persistent worker testing

Ejecutar múltiples scopes con:

```text
same config generation
```

y comprobar ausencia de mutación accidental.

---

# 289. Generation testing

```text
Request A starts G1
publish G2
Request B starts G2
Request A remains G1
```

---

# 290. Failed generation

```text
G1 active
build G2
G2 invalid
→ reject G2
→ G1 remains active
```

---

# 291. Configuration performance

La compilación podrá ser costosa.

El acceso runtime deberá ser barato.

---

# 292. Performance objective

```text
Expensive:
load + normalize + validate + compile

Cheap:
read typed immutable configuration
```

---

# 293. No repeated parsing

No deberá parsearse:

```text
"5s"
```

a `Duration` en cada query.

Se hará durante compilation.

---

# 294. No repeated driver alias resolution

```text
"pgsql"
→ PostgreSQL
```

deberá resolverse una vez por generación.

---

# 295. No repeated inheritance merge

Connection inheritance deberá resolverse durante config compilation.

---

# 296. Memory

Compiled configuration deberá compartir estructuras inmutables cuando sea seguro.

---

# 297. Configuration size

No deberán incluirse objetos runtime grandes:

```text
connections
pools
EntityManagers
metadata objects with runtime state
```

dentro de config.

---

# 298. Configuration is description

Regla:

> **Configuration describes intended Database behavior; it does not contain the runtime objects that implement that behavior.**

---

# 299. Formal configuration model

Sea:

```text
R = Raw Configuration
E = Environment Resolution
N = Normalization
V = Validation
C = Compilation
```

Entonces:

```text
D = C(V(N(E(R))))
```

donde:

```text
D = DatabaseConfiguration
```

---

# 300. Validity

Una configuración será válida si:

```text
Valid(D)
=
StructuralValid
∧
SemanticValid
∧
CrossSectionValid
∧
ExtensionValid
∧
SecurityPolicySatisfied
```

para el nivel de validación solicitado.

---

# 301. Effective runtime policy

Sea:

```text
D = compiled configuration
K = capability snapshot
R = runtime profile
```

Entonces:

```text
EffectivePolicy
=
Resolve(D, K, R)
```

pero:

```text
EffectivePolicy
≠
D mutation
```

---

# 302. Generation model

```text
Gₙ
=
(
ConfigurationId,
Fingerprint,
SchemaVersion,
ExtensionSet,
CreatedAt
)
```

sin incluir secretos materializados.

---

# 303. Operation consistency

Para operación `O`:

```text
ConfigGeneration(O, t₀)
=
ConfigGeneration(O, t₁)
```

durante todo su lifetime.

---

# 304. Reload safety

Una nueva generación `G₂` podrá publicarse sólo si:

```text
Parsed(G₂)
∧
Normalized(G₂)
∧
Validated(G₂)
∧
Compiled(G₂)
∧
SafeToPublish(G₂)
```

---

# 305. Config simplicity

Aunque internamente exista esta arquitectura, la experiencia básica deberá seguir siendo simple.

Ejemplo mínimo:

```php
return [
    'default' => 'main',

    'connections' => [
        'main' => [
            'driver' => 'postgresql',
            'host' => env('DB_HOST', '127.0.0.1'),
            'database' => env('DB_DATABASE', 'app'),
            'username' => env('DB_USERNAME', 'app'),
            'password' => secret('database.main.password'),
        ],
    ],
];
```

---

# 306. Progressive disclosure

Características avanzadas sólo deberán aparecer cuando se necesiten:

```text
replicas
sharding
pooling
custom drivers
capability requirements
dynamic topology
tenant resolution
custom security
```

---

# 307. Zero configuration aspirations

Cuando sea seguro, VoltStack podrá inferir defaults razonables.

Pero:

> **Zero configuration deberá significar defaults seguros y deterministas, no semántica oculta.**

---

# 308. Config philosophy

VoltStack Database deberá combinar:

```text
Laravel-like configuration ergonomics
+
typed configuration internals
+
Symfony-like explicit service integration
+
Database capability awareness
+
persistent runtime safety
```

sin copiar necesariamente la implementación de ninguno.

---

# 309. Implementation phases

## Phase 1 — V1 Core

Implementar:

```text
DatabaseConfiguration
ConnectionConfiguration
Environment integration
SecretReference
Normalizer
Validator
Config compiler
MySQL
MariaDB
PostgreSQL
SQLite
typed timeouts
basic runtime configuration
```

---

# 310. Phase 2

Agregar:

```text
pool configuration
read/write
replicas
cache
telemetry
security policies
config cache
diagnostics CLI
```

---

# 311. Phase 3

Agregar:

```text
configuration generations
diff
reload classification
dynamic providers
tenant templates
sharding topology
```

---

# 312. Phase 4

Agregar:

```text
controlled hot reload
resource draining
dynamic topology
temporary credential rotation
advanced service discovery
```

---

# 313. Checklist

Antes de considerar completo el sistema:

- [ ] Database no lee `.env`.
- [ ] Database no lee `$_ENV`.
- [ ] configuración tipada implementada.
- [ ] configuración inmutable implementada.
- [ ] default connection validada.
- [ ] múltiples conexiones soportadas.
- [ ] Driver separado de Platform.
- [ ] MySQL separado de MariaDB.
- [ ] URL/DSN parsing seguro.
- [ ] secrets separados de config normal.
- [ ] SecretReference implementado.
- [ ] late secret resolution soportado.
- [ ] secret redaction probada.
- [ ] timeout types separados.
- [ ] pool config validada.
- [ ] runtime profiles definidos.
- [ ] FrankenPHP profile definido.
- [ ] ORM config tipada.
- [ ] transaction config tipada.
- [ ] cache config tipada.
- [ ] telemetry config tipada.
- [ ] security config tipada.
- [ ] strict unknown-key validation.
- [ ] extension config schema.
- [ ] config compiler.
- [ ] config cache.
- [ ] config fingerprint.
- [ ] generation model.
- [ ] reload classification.
- [ ] production safety guard.
- [ ] CLI diagnostics seguros.
- [ ] testing builders.
- [ ] backward compatibility strategy.
- [ ] persistent worker tests.

---

# 314. Relación con Container Integration

La arquitectura conjunta queda:

```text
VoltStack Config
      ↓
Database Config Integration
      ↓
Typed Immutable DatabaseConfiguration
      ↓
VoltStack Container
      ↓
Shared Immutable Services
      +
Operation Scoped Services
      ↓
Database Runtime
```

Config define:

```text
what should be configured
```

Container define:

```text
how objects are constructed
```

Database Runtime define:

```text
how database operations behave
```

---

# 315. Regla conjunta

```text
Config
≠
Container
≠
Runtime State
```

pero:

```text
Config
→
Container Wiring
→
Runtime Behavior
```

mediante contratos explícitos.

---

# 316. Relación con Capability Discovery

Debe mantenerse:

```text
Configuration
    ↓
Desired / Required Behavior

Capability Discovery
    ↓
Observed Evidence

Capability Resolver
    ↓
Effective Support Decision
```

Por ejemplo:

```text
config:
    require RETURNING

server:
    PostgreSQL 17

capability discovery:
    RETURNING supported

resolver:
    requirement satisfied
```

---

# 317. Caso contrario

```text
config:
    require capability X

discovery:
    UNSUPPORTED
```

resultado:

```text
CapabilityRequirementNotSatisfied
```

No:

```text
pretend X is supported
```

---

# 318. Relación con Persistent Runtime

La combinación de:

```text
immutable configuration
+
configuration generations
+
operation scopes
```

permite que VoltStack opere de forma segura bajo FrankenPHP.

---

# 319. Modelo completo

```text
Worker
│
├── Configuration Generation G1
│
├── Root Container G1
│
├── Shared Database Infrastructure G1
│
│
├── Request A
│   └── Operation Scope → G1
│
├── Request B
│   └── Operation Scope → G1
│
└── Configuration Reload
    │
    ├── Build G2
    ├── Validate G2
    ├── Compile G2
    └── Publish G2
          │
          └── Request C → G2
```

Las operaciones A y B no cambian de generación a mitad de ejecución.

---

# 320. Principio definitivo

> **VoltStack Database deberá tratar la configuración como una descripción tipada, validada, inmutable y versionable de la intención operativa del sistema; nunca como un almacén global mutable ni como sustituto de la evidencia obtenida del runtime o del DBMS.**

Esto establece:

```text
Config says what is requested.

Capabilities say what is supported.

Runtime Context says what applies now.

Database Engine decides how to execute safely.
```

---

# 321. Resultado arquitectónico

Con este sistema, VoltStack podrá ofrecer:

```text
simple configuration
+
strong typing
+
secure secret handling
+
multiple databases
+
persistent runtime safety
+
extension support
+
configuration compilation
+
future hot reload
+
clear diagnostics
```

sin introducir dependencia directa entre Database Core y los mecanismos externos de configuración.

---

# 322. Siguiente documento

```text
314_DATABASE_CACHE_INTEGRATION_SYSTEM.md
```

Definirá la integración entre:

```text
VoltStack/Quantum/Database
        ↕
VoltStack/Quantum/Cache
```

incluyendo:

```text
Database Cache Adapter
cache provider contracts
cache namespaces
metadata cache
compiled query cache
result cache
entity cache
cache key generation
tenant/shard isolation
cache invalidation
transaction-aware invalidation
after-commit publication
UNKNOWN transaction outcomes
cache consistency policies
cache generations
cache tags
distributed cache
local worker cache
persistent runtime cache safety
cache serialization
cache stampede protection
cache telemetry
cache failure handling
cache security
optional Cache package integration
Null cache adapters
testing
```

manteniendo como regla central:

```text
Database Cache
≠
Database Truth
```

y:

```text
Cache Hit
≠
Usable Result
```

hasta que la política de consistencia determine que el valor almacenado puede utilizarse de forma segura.