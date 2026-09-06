# 13_DATABASE_CONNECTION_CONFIGURATION_AND_RESOLUTION.md

# VoltStack Quantum Database
## Connection Configuration and Resolution System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 13 — Database Connection Configuration and Resolution  
**Estado:** Architecture Specification  
**Nivel:** Infrastructure Architecture  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura oficial del sistema encargado de transformar configuración externa de conexiones en definiciones tipadas, validadas, compiladas y posteriormente resolubles por el `ConnectionManager`.

La cadena fundamental será:

```text
External Configuration
        │
        ▼
Connection Configuration Loader
        │
        ▼
Normalization
        │
        ▼
Validation
        │
        ▼
ConnectionDefinition
        │
        ▼
Definition Registry
        │
        ▼
Connection Resolution
        │
        ▼
Logical Connection
```

El objetivo es impedir que las capas de ejecución consulten configuración dinámica durante cada operación.

---

# 2. Principio fundamental

> Database configuration is resolved before Database execution.

Por tanto:

```text
Configuration
→ Normalize
→ Validate
→ Compile
→ Freeze
→ Runtime
```

y nunca:

```text
Runtime Query
→ getenv()
→ config()
→ parse DSN
→ determine driver
→ execute
```

---

# 3. Relación con `06_DATABASE_CONFIGURATION_SYSTEM.md`

El documento `06_DATABASE_CONFIGURATION_SYSTEM.md` define el sistema general de configuración de Database.

Este documento especializa ese modelo para:

```text
Connection
```

La relación será:

```text
DatabaseConfiguration
        │
        ├── QueryConfiguration
        ├── ORMConfiguration
        ├── MigrationConfiguration
        ├── RuntimeConfiguration
        └── ConnectionConfiguration
                    │
                    ▼
            ConnectionDefinition
```

---

# 4. Responsabilidad del módulo

Este subsystem será responsable de:

```text
connection configuration schema
connection configuration normalization
connection validation
connection identity
connection definitions
connection aliases
default connection
driver references
platform hints
dialect hints
database endpoints
credential references
security configuration
timeouts
session profiles
pool references
role configuration
native options
definition fingerprints
definition generations
runtime resolution requests
resolution precedence
resolution diagnostics
```

---

# 5. No responsabilidades

No deberá encargarse de:

```text
opening sockets
creating PDO
executing SQL
query routing algorithms
ORM
transactions
pool internals
server health checks
replica balancing
shard algorithms
```

---

# 6. Arquitectura general

```text
Configuration Source
       │
       ▼
Connection Config Loader
       │
       ▼
RawConnectionConfiguration
       │
       ▼
Connection Config Normalizer
       │
       ▼
NormalizedConnectionConfiguration
       │
       ▼
Connection Config Validator
       │
       ▼
Connection Definition Compiler
       │
       ▼
ConnectionDefinition
       │
       ▼
ConnectionDefinitionRegistry
       │
       ▼
ConnectionResolver
       │
       ▼
ConnectionResolution
```

---

# 7. Configuración externa

VoltStack podrá aceptar configuración procedente de:

```text
application configuration
environment variables
DATABASE_URL
deployment configuration
secret references
programmatic configuration
testing overrides
official package configuration
```

Pero estas fuentes deberán converger antes de entrar al runtime Database.

---

# 8. Configuración conceptual

Ejemplo:

```yaml
database:
  default: primary

  connections:

    primary:
      driver: pgsql

      endpoint:
        host: db.internal
        port: 5432

      database: voltstack

      credentials:
        username: ${DB_USERNAME}
        password: ${DB_PASSWORD}

      role: primary

      tls:
        enabled: true
        verify_peer: true

      timeout:
        connect: 5s

    analytics:
      driver: mysql

      endpoint:
        host: analytics.internal
        port: 3306

      database: analytics

      credentials:
        reference: secrets/database/analytics

      role: read_only
```

---

# 9. Configuración externa no es runtime model

El array/YAML anterior no deberá circular internamente.

Será convertido a objetos.

```text
array
   │
   ▼
typed configuration
   │
   ▼
compiled definition
```

---

# 10. RawConnectionConfiguration

Podrá existir una representación intermedia:

```text
RawConnectionConfiguration
```

que conserva datos provenientes de configuración antes de su normalización.

No deberá llegar al Query Engine.

---

# 11. NormalizedConnectionConfiguration

Después de normalización:

```text
NormalizedConnectionConfiguration
```

deberá contener valores estructuralmente consistentes.

Ejemplo:

```text
"5s"
→
Duration(5 seconds)
```

---

# 12. ConnectionDefinition

El resultado principal será:

```php
final readonly class ConnectionDefinition
{
    public function __construct(
        public ConnectionIdentity $identity,
        public DriverId $driver,
        public DatabaseEndpoint $endpoint,
        public DatabaseName $database,
        public CredentialReference $credentials,
        public ConnectionRole $role,
        public ConnectionTimeoutConfiguration $timeouts,
        public ConnectionSecurityConfiguration $security,
        public ConnectionSessionProfile $session,
        public ConnectionNativeOptions $nativeOptions,
    ) {}
}
```

La API exacta podrá evolucionar.

---

# 13. ConnectionDefinition como runtime configuration

`ConnectionDefinition` será:

```text
typed
validated
immutable
secret-safe
runtime-ready
```

---

# 14. ConnectionDefinition no contiene recursos

No deberá contener:

```text
PDO
socket
Driver instance
ConnectionLease
Transaction
Result
Statement
```

---

# 15. ConnectionIdentity

Toda definición tendrá:

```text
ConnectionIdentity
```

Ejemplo:

```php
ConnectionIdentity::from('primary');
```

---

# 16. Reglas de identidad

Los nombres deberán ser:

```text
non-empty
normalized
unique
stable
```

y preferentemente case-sensitive o case-normalized mediante una regla única y documentada.

---

# 17. Canonical connection identity

Se recomienda normalizar internamente identidades a una forma canónica.

Ejemplo:

```text
"Primary"
```

podría rechazarse o convertirse consistentemente.

No deberán coexistir reglas ambiguas.

---

# 18. Identidad vs alias

```text
ConnectionIdentity
≠
ConnectionAlias
```

Identity identifica la definición.

Alias redirige a una identity.

---

# 19. Default connection

La configuración deberá definir:

```text
database.default
```

salvo escenarios especiales explícitamente permitidos.

---

# 20. Default connection invariant

Si:

```text
database.default = primary
```

debe existir:

```text
connections.primary
```

o un alias resoluble hacia una definición válida.

---

# 21. Default resolution

```text
connection()
    │
    ▼
DefaultConnectionIdentity
    │
    ▼
ConnectionDefinitionRegistry
```

---

# 22. Explicit connection resolution

```text
connection('analytics')
    │
    ▼
ConnectionIdentity
    │
    ▼
Definition Registry
```

---

# 23. Unknown explicit connection

Debe fallar.

```text
connection('missing')
→ ConnectionNotConfiguredException
```

No deberá caer silenciosamente al default.

---

# 24. Connection aliases

Configuración posible:

```yaml
database:
  aliases:
    write: primary
    reports: analytics
```

---

# 25. Alias resolution

```text
reports
   │
   ▼
analytics
   │
   ▼
ConnectionDefinition
```

---

# 26. Alias canonicalization

Cuando sea posible:

```text
alias
→ canonical identity
```

se resolverá durante bootstrap.

---

# 27. Alias chains

Puede permitirse:

```text
reporting
→ reports
→ analytics
```

pero deberán colapsarse a:

```text
reporting
→ analytics
```

durante compilación.

---

# 28. Alias cycles

Deberán rechazarse:

```text
a → b
b → c
c → a
```

---

# 29. Alias collision

No deberá permitirse que una identidad y alias creen semántica ambigua.

Ejemplo:

```text
connection = analytics
alias      = analytics
```

apuntando a otra definición.

---

# 30. Driver configuration

Toda conexión deberá declarar o resolver un:

```text
DriverId
```

---

# 31. Driver IDs

Ejemplos conceptuales:

```text
pdo.mysql
pdo.pgsql
pdo.sqlite
mysqli
pgsql.native
```

El ID concreto deberá pertenecer al Driver Registry.

---

# 32. Driver validation

Durante bootstrap:

```text
ConnectionDefinition
      │
      ▼
DriverRegistry
      │
      ├── registered → valid
      └── missing    → configuration failure
```

---

# 33. No `extension_loaded()` in hot path

La disponibilidad del driver deberá validarse centralmente.

No:

```php
if (extension_loaded('pdo_pgsql')) {
}
```

en cada conexión/query.

---

# 34. Driver availability vs driver registration

Se distinguirá:

```text
registered
```

de:

```text
runtime available
```

Un driver puede estar registrado pero faltar su extensión nativa.

---

# 35. Driver availability diagnostics

Ejemplo:

```text
Driver: pdo.pgsql
Registered: yes
Runtime available: no
Missing extension: pdo_pgsql
```

---

# 36. DatabaseEndpoint

No todos los motores utilizan:

```text
host + port
```

Por ello se requiere:

```text
DatabaseEndpoint
```

---

# 37. Endpoint variants

Podrán existir:

```text
TcpDatabaseEndpoint
UnixSocketDatabaseEndpoint
FileDatabaseEndpoint
MemoryDatabaseEndpoint
CustomDatabaseEndpoint
```

---

# 38. TCP endpoint

Ejemplo:

```php
new TcpDatabaseEndpoint(
    host: 'db.internal',
    port: 5432,
);
```

---

# 39. Unix socket

Ejemplo conceptual:

```php
new UnixSocketDatabaseEndpoint(
    path: '/var/run/postgresql/.s.PGSQL.5432',
);
```

---

# 40. SQLite file

```php
new FileDatabaseEndpoint(
    path: '/app/storage/database.sqlite',
);
```

---

# 41. SQLite memory

```text
MemoryDatabaseEndpoint
```

deberá modelarse explícitamente.

---

# 42. Endpoint validation

Cada endpoint tendrá reglas específicas.

Ejemplos:

```text
TCP:
host required
valid port range

File:
path required

Memory:
no host
no port
```

---

# 43. DatabaseName

Cuando corresponda deberá modelarse mediante:

```text
DatabaseName
```

y no como string sin semántica dispersa.

---

# 44. DatabaseName optionality

SQLite u otros drivers pueden tratar el target de manera diferente.

Por ello las restricciones deberán provenir de:

```text
Driver configuration schema
```

y no de una regla universal falsa.

---

# 45. CredentialReference

ConnectionDefinition preferirá:

```text
CredentialReference
```

en lugar de secretos resueltos permanentemente.

---

# 46. Credential model

```text
ConnectionDefinition
        │
        ▼
CredentialReference
        │
        ▼
CredentialResolver
        │
        ▼
DatabaseCredential
        │
        ▼
Driver
```

---

# 47. DatabaseCredential

Puede contener temporalmente:

```text
username
password
token
certificate references
```

según Driver.

---

# 48. Secret lifetime

El secreto deberá existir en memoria el menor tiempo práctico posible.

---

# 49. Secret providers

Podrán integrarse:

```text
environment
encrypted configuration
Vault-like providers
cloud secret managers
application secret stores
```

mediante contratos.

---

# 50. Secret provider boundary

Database no deberá depender directamente de un proveedor específico.

---

# 51. Credential resolution failure

Ejemplos:

```text
reference not found
permission denied
expired secret
malformed credential
```

deberán producir excepciones normalizadas.

---

# 52. Secret redaction

Toda representación diagnóstica deberá sustituir:

```text
password = secret
```

por:

```text
password = [REDACTED]
```

---

# 53. DSN support

VoltStack podrá aceptar:

```text
DATABASE_URL
```

o DSN similares como formato de entrada.

---

# 54. DSN is input syntax

Regla:

```text
DSN
≠
runtime configuration model
```

---

# 55. DSN parsing

```text
DATABASE_URL
     │
     ▼
DSN Parser
     │
     ▼
Structured Connection Configuration
```

---

# 56. Example DSN

```text
postgresql://user:password@db.internal:5432/voltstack
```

se convertirá a campos estructurados.

---

# 57. DSN secrets

El DSN original no deberá conservarse en diagnostics si contiene credenciales.

---

# 58. DSN precedence

Si existen simultáneamente:

```text
DATABASE_URL
+
structured fields
```

deberá existir una política explícita.

---

# 59. Recommended precedence

Una política posible:

```text
base DSN
→ structured overrides
```

pero deberá quedar formalizada por Config System.

---

# 60. Ambiguous configuration

En strict mode podrá preferirse:

```text
conflicting values
→ configuration error
```

en lugar de adivinar.

---

# 61. ConnectionRole

Cada definición podrá declarar:

```text
PRIMARY
REPLICA
READ_ONLY
READ_WRITE
MIGRATION
ADMIN
```

según el modelo final.

---

# 62. Role normalization

Strings externos:

```text
"primary"
```

se convertirán a:

```php
ConnectionRole::PRIMARY
```

antes del runtime.

---

# 63. Role validation

Una configuración incoherente deberá rechazarse.

Ejemplo:

```text
role = READ_ONLY
write_required = true
```

si ambos conceptos son incompatibles.

---

# 64. Role vs topology

Role describe propiedades del target.

Topology decide cómo seleccionar targets.

---

# 65. Platform hint

Una definición podrá contener:

```text
PlatformHint
```

para ayudar antes de abrir conexión.

---

# 66. Example platform hint

```yaml
platform:
  family: postgresql
  version: "17"
```

---

# 67. Hint is not absolute truth

Una vez conectado, VoltStack podrá verificar:

```text
actual server platform
```

contra el hint.

---

# 68. Platform mismatch

Dependiendo de policy:

```text
warn
reject
recompile capabilities
```

---

# 69. Dialect hint

Podrá existir cuando Driver no determine inequívocamente la sintaxis.

Pero deberá evitarse duplicación innecesaria.

---

# 70. Driver/Dialect/Platform separation

```text
Driver
=
communication

Dialect
=
syntax

Platform
=
capabilities/semantics
```

Connection configuration sólo referencia estas piezas; no las fusiona.

---

# 71. Native driver options

Cada driver puede requerir opciones específicas.

Ejemplo:

```yaml
native:
  emulate_prepares: false
```

---

# 72. Native options boundary

Las opciones nativas deberán permanecer bajo:

```text
ConnectionNativeOptions
```

o estructura driver-specific.

No deberán contaminar Query/ORM.

---

# 73. Native option validation

El Driver Extension podrá registrar su schema.

---

# 74. Unknown native option

En strict mode:

```text
unknown native option
→ configuration error
```

---

# 75. Portable options

Opciones comunes deberán modelarse fuera del bloque native.

Ejemplos:

```text
connect timeout
TLS requirement
application name
session timezone
```

---

# 76. Portable vs native

```text
portable option
→ VoltStack semantic model

native option
→ Driver-specific configuration
```

---

# 77. Connection timeout configuration

Se distinguirán varios timeouts.

```php
final readonly class ConnectionTimeoutConfiguration
{
    public function __construct(
        public Duration $connect,
        public Duration $acquire,
    ) {}
}
```

---

# 78. Connect timeout

Tiempo máximo para establecer conexión física.

---

# 79. Acquire timeout

Tiempo máximo esperando un recurso del pool.

---

# 80. Query timeout

No deberá confundirse con los anteriores.

Pertenece principalmente a Execution.

---

# 81. Transaction timeout

Pertenece a Transaction.

---

# 82. Duration normalization

Configuraciones como:

```text
500ms
5s
1m
```

deberán convertirse a un `Duration` tipado.

---

# 83. Negative timeout

Deberá rechazarse salvo semántica explícita como:

```text
infinite
```

representada formalmente.

---

# 84. TLS configuration

Podrá existir:

```php
ConnectionSecurityConfiguration
```

con:

```text
TLS enabled
peer verification
hostname verification
CA reference
client certificate reference
client key reference
minimum TLS policy
```

---

# 85. Secure fallback

Si:

```text
TLS required
```

y no puede establecerse:

```text
connection fails
```

Nunca:

```text
TLS failed
→ plaintext fallback
```

silencioso.

---

# 86. TLS capability validation

El Driver deberá declarar si soporta los requisitos configurados.

---

# 87. Credential/TLS secret references

Certificados y claves privadas deberán usar referencias seguras cuando sea posible.

---

# 88. Session profile

Una conexión podrá declarar:

```text
ConnectionSessionProfile
```

---

# 89. Session profile fields

Podrá incluir:

```text
timezone
charset
collation
schema/search path
application name
SQL mode
role
statement defaults
```

según plataforma.

---

# 90. Session profile purpose

Define:

> ¿En qué estado conocido debe estar una conexión antes de entregarse al consumidor?

---

# 91. Session profile application

```text
Physical Connection
       │
       ▼
Session Initializer
       │
       ▼
Known Session State
       │
       ▼
Lease
```

---

# 92. Session profile reset

Al liberar:

```text
actual session state
      │
      ▼
Reset
      │
      ▼
baseline session profile
```

---

# 93. Session profile capabilities

No todos los motores soportan los mismos settings.

Deberán validarse mediante Platform/Driver capabilities.

---

# 94. Pool configuration reference

La ConnectionDefinition no necesita contener toda la implementación del pool.

Puede contener:

```text
PoolPolicyReference
```

o configuración compilada.

---

# 95. Pool example

```yaml
pool:
  name: default
```

---

# 96. Pool definitions

Podrán configurarse separadamente:

```yaml
database:
  pools:
    default:
      min: 0
      max: 20
      acquire_timeout: 2s
```

---

# 97. Connection-to-pool association

```text
ConnectionDefinition
       │
       ▼
PoolPolicyReference
       │
       ▼
ConnectionPoolManager
```

---

# 98. Pool is optional

Una conexión podrá declarar:

```text
pool: disabled
```

---

# 99. Persistent runtime vs pool

Regla:

```text
persistent worker
≠
automatic persistent connection
```

---

# 100. Connection reuse policy

Podrá declararse:

```text
reuse:
  enabled: true
  reset: strict
```

pero el runtime siempre deberá respetar las invariantes de seguridad.

---

# 101. Connection reset configuration

Puede existir:

```text
ConnectionResetPolicy
```

con niveles como:

```text
standard
strict
discard
```

---

# 102. Unsafe reset setting

VoltStack no deberá permitir configurar una combinación conocida como insegura sin advertencia/error explícito.

---

# 103. Read/write configuration

Podrá existir configuración estructurada:

```yaml
database:
  connections:

    app:
      topology:
        primary: app-primary
        replicas:
          - app-replica-1
          - app-replica-2
```

---

# 104. Important distinction

`app` puede representar:

```text
logical topology entry
```

mientras:

```text
app-primary
app-replica-1
app-replica-2
```

representan targets concretos.

---

# 105. Alternative model

También podrá mantenerse la topología completamente separada:

```yaml
database:
  connections:
    primary: ...
    replica1: ...

  topologies:
    application:
      primary: primary
      replicas:
        - replica1
```

Esta separación es arquitectónicamente más limpia.

---

# 106. Recommended approach

Preferir:

```text
ConnectionDefinition
=
single logical target

DatabaseTopology
=
relationship among targets
```

---

# 107. Why?

Evita convertir `ConnectionDefinition` en:

```text
connection
+
replica router
+
load balancer
+
failover system
```

---

# 108. Topology references

La configuración de topología podrá referenciar identidades de Connection.

---

# 109. Cross-reference validation

Bootstrap deberá comprobar:

```text
topology references existing connection
```

sin necesidad de abrirla.

---

# 110. Replica configuration

Cada replica deberá seguir siendo una ConnectionDefinition completa.

---

# 111. Sticky reads

Sticky policy pertenece a topology/read-write routing.

No al ConnectionDefinition básico.

---

# 112. Dynamic definitions

Algunas conexiones no podrán conocerse en bootstrap.

Ejemplos:

```text
tenant database
temporary test database
ephemeral shard
runtime provisioned database
```

---

# 113. DynamicConnectionDefinition

Podrá existir:

```text
DynamicConnectionDefinition
```

o reutilizarse `ConnectionDefinition` con ownership scoped.

---

# 114. Critical rule

La diferencia entre static y dynamic definition será principalmente:

```text
lifecycle
ownership
resolution source
```

no una segunda arquitectura de conexión.

---

# 115. Dynamic definition validation

Deberá pasar por las mismas reglas esenciales:

```text
normalize
validate
capability check
secret safety
```

---

# 116. No bypass for tenants

Multitenancy no deberá crear arrays improvisados y entregarlos al Driver.

Debe producir una definición tipada.

---

# 117. Dynamic definition compiler

Podrá existir una ruta optimizada:

```text
Tenant Database Descriptor
       │
       ▼
Dynamic Definition Factory
       │
       ▼
Validated ConnectionDefinition
```

---

# 118. Scoped ownership

```text
ExecutionScope A
       │
       ▼
Dynamic Definition A
```

se destruye con A.

---

# 119. ConnectionDefinitionProvider

Contrato conceptual:

```php
interface ConnectionDefinitionProviderInterface
{
    public function find(
        ConnectionIdentity $identity,
        ConnectionResolutionContext $context
    ): ?ConnectionDefinition;
}
```

---

# 120. Static provider

```text
StaticConnectionDefinitionProvider
→ frozen registry
```

---

# 121. Scoped provider

```text
ScopedConnectionDefinitionProvider
→ DatabaseContext
```

---

# 122. Extension provider

Paquetes opcionales podrán registrar providers mediante Composition Root.

---

# 123. Provider chain

Si se utiliza una cadena:

```text
Scoped Provider
      │
      ▼
Static Provider
```

la precedencia deberá ser determinista.

---

# 124. Shadowing static connections

Una definición scoped con el mismo nombre que una static deberá requerir una política explícita.

---

# 125. Testing override

Tests sí pueden necesitar:

```text
primary static
→ primary scoped override
```

---

# 126. Override policy

Podrá existir:

```text
ConnectionDefinitionOverridePolicy
```

que determine qué scopes pueden hacer shadowing.

---

# 127. Production tenant definitions

En multitenancy podría ser preferible usar una identidad contextual separada en vez de sobrescribir una global.

La estrategia dependerá del integration design.

---

# 128. Resolution architecture

La resolución completa deberá distinguir:

```text
definition resolution
connection selection
physical resource acquisition
```

---

# 129. Three-stage model

```text
Connection Requirement
        │
        ▼
Connection Selection
        │
        ▼
ConnectionDefinition
        │
        ▼
Logical Connection
        │
        ▼
Resource Acquisition
```

---

# 130. ConnectionResolutionRequest

Una solicitud podrá contener:

```text
explicit identity
intent
required capabilities
consistency requirement
affinity
execution scope
resolution hints
```

---

# 131. Typed request

Ejemplo conceptual:

```php
final readonly class ConnectionResolutionRequest
{
    public function __construct(
        public ?ConnectionIdentity $identity,
        public ConnectionIntent $intent,
        public RequiredCapabilitySet $capabilities,
        public ?ConnectionAffinity $affinity,
    ) {}
}
```

---

# 132. Resolution context

Datos request-specific no deberán mezclarse con la request estructural si tienen lifecycle diferente.

Podrá existir:

```text
ConnectionResolutionContext
```

---

# 133. ResolutionContext

Puede contener referencias controladas a:

```text
ExecutionScope
TransactionContext
Tenant database integration state
Scoped definition providers
```

---

# 134. No generic context bag

Incorrecto:

```php
$context['anything'] = ...
```

Preferir contratos explícitos.

---

# 135. ConnectionIntent

Valores conceptuales:

```text
READ
WRITE
SCHEMA
MIGRATION
ADMIN
MAINTENANCE
```

---

# 136. Intent source

La intención deberá determinarse antes de la capa Connection.

Ejemplo:

```text
Query Planner
→ READ

Persistence Engine
→ WRITE

Migration Engine
→ MIGRATION
```

---

# 137. Connection resolution precedence

Una precedencia recomendada será:

```text
1. Mandatory transaction/affinity binding
2. Explicit connection identity
3. Explicit scoped override
4. Optional integration resolution
5. Topology/routing selection
6. Configured default connection
```

---

# 138. Affinity precedence

Si una operación pertenece a una transaction:

```text
Transaction affinity
```

deberá tener precedencia.

---

# 139. Explicit connection inside transaction

Si contradice la afinidad:

```text
reject
```

No cambiar de conexión silenciosamente.

---

# 140. Explicit identity

Fuera de una afinidad obligatoria:

```text
explicit identity
```

deberá respetarse.

---

# 141. Scoped override

Un test/application scope podrá modificar la definición de esa identity cuando esté autorizado.

---

# 142. Integration resolution

Multitenancy/sharding podrán participar mediante ports.

---

# 143. Topology selection

Si no hay identity explícita, un `ConnectionIntent` puede ser enviado a Topology.

---

# 144. Default fallback

El default se utiliza cuando:

```text
no explicit target
+
no mandatory contextual target
+
no topology target
```

---

# 145. No unknown fallback

Si se pidió explícitamente:

```text
analytics
```

y no existe:

```text
fail
```

---

# 146. ConnectionResolutionPolicy

La precedencia podrá encapsularse en:

```text
ConnectionResolutionPolicy
```

para hacerla verificable/testeable.

---

# 147. Determinism

Dados:

```text
same configuration
same request
same context
same topology state
```

la resolución deberá producir la misma decisión, salvo factores explícitamente dinámicos como load balancing.

---

# 148. Dynamic decisions

Cuando intervenga:

```text
replica health
load
circuit breaker
```

la resolución deberá registrar la razón.

---

# 149. ConnectionResolution

El resultado deberá ser un objeto.

Ejemplo:

```php
final readonly class ConnectionResolution
{
    public function __construct(
        public ConnectionIdentity $identity,
        public ConnectionDefinition $definition,
        public ConnectionResolutionSource $source,
        public ConnectionRole $effectiveRole,
    ) {}
}
```

---

# 150. Resolution source

Posibles valores:

```text
DEFAULT
EXPLICIT
ALIAS
TRANSACTION
SCOPED_OVERRIDE
TENANT
TOPOLOGY
SHARD
TEST
```

---

# 151. Resolution reason

Además de source podrá existir:

```text
ConnectionResolutionReason
```

para debugging.

---

# 152. Canonical identity

La resolution siempre deberá devolver la identity canónica.

---

# 153. Alias trace

Puede conservarse:

```text
requested = reports
canonical = analytics
```

para diagnostics.

---

# 154. Required capabilities

Una request podrá exigir:

```text
READ
WRITE
RETURNING
SAVEPOINT
STREAMING_CURSOR
```

según operación.

---

# 155. Capability resolution

```text
ConnectionDefinition
       │
       ▼
Driver
+
Platform
+
Role
+
Runtime
       │
       ▼
EffectiveCapabilitySet
```

---

# 156. Capability matching

```text
RequiredCapabilitySet
        │
        ▼
ConnectionCapabilityMatcher
        │
        ▼
EffectiveCapabilitySet
        │
        ├── satisfies
        └── reject
```

---

# 157. No driver-name branching

Incorrecto:

```php
if ($driver === 'pgsql') {
    return true;
}
```

Correcto:

```php
$capabilities->supports(
    DatabaseCapability::RETURNING
);
```

---

# 158. Unknown server capabilities

Antes de la primera conexión algunas capabilities pueden ser:

```text
UNKNOWN
```

---

# 159. Capability state

Podrá modelarse:

```text
SUPPORTED
UNSUPPORTED
UNKNOWN
```

---

# 160. Unknown handling

Dependiendo de la operación:

```text
defer
require discovery
use conservative fallback
reject
```

---

# 161. Platform discovery

La primera conexión podrá descubrir:

```text
server vendor
server version
extensions/features
```

---

# 162. Capability snapshot

El resultado podrá convertirse en:

```text
PlatformCapabilitySnapshot
```

cacheable mientras sea válido.

---

# 163. Capability snapshot scope

Podrá asociarse a:

```text
server target
configuration generation
```

y no necesariamente a cada ConnectionLease.

---

# 164. Configuration fingerprint

Cada `ConnectionDefinition` deberá poder producir una identidad técnica segura.

---

# 165. Fingerprint goals

Servirá para:

```text
pool compatibility
compiled configuration cache
change detection
diagnostics
resource retirement
```

---

# 166. Fingerprint content

Puede incluir:

```text
driver ID
endpoint
database
role
TLS policy
session profile
native option signature
credential reference identity
```

sin incluir secretos plaintext.

---

# 167. Fingerprint algorithm

Debe ser:

```text
deterministic
stable within configuration version
non-secret-leaking
```

---

# 168. Credential rotation and fingerprint

Si cambia el contenido del secret sin cambiar la referencia, el fingerprint puede no cambiar.

Por ello deberá existir otro mecanismo de:

```text
credential generation
```

cuando sea necesario.

---

# 169. ConfigurationGeneration

Podrá representar una revisión de configuración.

---

# 170. Example

```text
Generation 41
→ worker connections

Configuration reload

Generation 42
→ new acquisitions
```

---

# 171. Old resource retirement

Recursos de una generación anterior podrán:

```text
finish active operation
reset
close
```

en vez de volver al pool.

---

# 172. Atomic configuration publication

Una nueva configuración deberá publicarse sólo después de:

```text
load
normalize
validate
compile
```

completamente.

---

# 173. No partial configuration

Nunca:

```text
primary = new
analytics = old
aliases = half updated
```

por actualización no atómica.

---

# 174. Configuration snapshot

Cada worker/execution scope deberá observar un snapshot coherente.

---

# 175. Hot reload

No será obligatorio en v1.

Si se implementa:

```text
reload at safe boundary
```

será preferible a mutación arbitraria durante una operación.

---

# 176. Production recommendation

Para cambios estructurales importantes:

```text
compile new configuration
→ recycle worker
```

puede ser la estrategia más segura.

---

# 177. Credential rotation exception

La arquitectura sí deberá permitir rotar secrets sin exigir necesariamente recompilar toda la aplicación.

Esto es posible mediante `CredentialReference`.

---

# 178. Configuration provenance

Para debugging puede registrarse de dónde provino cada valor.

Ejemplo:

```text
database.connections.primary.driver
source:
config/database.php
```

---

# 179. Secret provenance

Puede indicarse:

```text
source = environment
```

sin mostrar el valor.

---

# 180. Config explain

Una futura CLI podrá ofrecer:

```text
php voltstack db:config explain primary
```

---

# 181. Example output

```text
Connection: primary

Driver:
  pdo.pgsql
  source: database.connections.primary.driver

Endpoint:
  db.internal:5432

Database:
  voltstack

Credentials:
  [REFERENCE: DB_PRIMARY]

Role:
  PRIMARY

TLS:
  required

Pool:
  default

Status:
  valid
```

---

# 182. Offline configuration validation

Podrá existir:

```text
php voltstack db:config validate
```

sin abrir conexiones.

---

# 183. Online validation

Separadamente:

```text
php voltstack db:connection check
```

podrá verificar:

```text
credentials
network
server version
capabilities
TLS
```

---

# 184. Offline vs online

```text
Configuration Valid
≠
Database Reachable
```

---

# 185. Validation phases

## Phase 1 — Structural

```text
required keys
types
formats
```

## Phase 2 — Semantic

```text
valid roles
valid timeout ranges
valid TLS combinations
```

## Phase 3 — Cross-reference

```text
default exists
aliases resolve
pool exists
driver registered
topology references exist
```

## Phase 4 — Capability-aware

```text
driver supports configured mechanism
```

## Phase 5 — Online

```text
server reachable
credentials accepted
actual capabilities
```

---

# 186. Strict configuration

Production y CI deberían poder activar:

```text
strict configuration validation
```

---

# 187. Unknown keys

Ejemplo:

```yaml
conection_timeout: 5s
```

deberá producir:

```text
Unknown configuration key:
conection_timeout

Did you mean:
connection_timeout
```

---

# 188. Configuration diagnostics

Los errores deberán indicar:

```text
connection identity
configuration path
invalid value category
expected value
possible correction
```

sin revelar secretos.

---

# 189. Duplicate definitions

Deberán rechazarse durante compilación.

---

# 190. Duplicate aliases

También deberán rechazarse o seguir una policy explícita.

---

# 191. Deprecated configuration

Podrá existir un sistema de:

```text
configuration deprecations
```

---

# 192. Example deprecation

```text
database.connections.primary.host
```

podría migrar en una versión futura a:

```text
database.connections.primary.endpoint.host
```

---

# 193. Deprecation diagnostics

Deberá indicar:

```text
old path
new path
removal version
```

---

# 194. Configuration migration

Podrán existir herramientas para migrar versiones de schema de configuración.

---

# 195. Configuration schema version

La configuración compilada podrá incluir:

```text
schemaVersion
```

---

# 196. Compiled representation version

También podrá existir:

```text
compiledFormatVersion
```

para invalidar caches incompatibles.

---

# 197. Connection config cache

En producción podrá generarse:

```text
CompiledConnectionConfiguration
```

---

# 198. Cached representation

Deberá contener objetos/datos ya:

```text
normalized
validated
canonicalized
```

---

# 199. Secrets in config cache

Preferentemente deberá almacenar:

```text
CredentialReference
```

y no el secreto resuelto.

---

# 200. Cache invalidation

La cache deberá invalidarse cuando cambie:

```text
configuration source
schema version
extension schema
driver registry compatibility
```

según estrategia.

---

# 201. Extension configuration

Un Driver de terceros podrá registrar su propio schema.

---

# 202. Extension registration order

```text
Core schemas
      │
      ▼
Extension schemas
      │
      ▼
Load configuration
      │
      ▼
Normalize
      │
      ▼
Validate
```

---

# 203. No late schema mutation

Después de compilar/freeze:

```text
register new driver config key
```

no deberá ocurrir durante requests normales.

---

# 204. Programmatic configuration

VoltStack podrá permitir:

```php
ConnectionDefinitionBuilder::make('analytics')
    ->driver('pdo.pgsql')
    ->tcp('analytics.internal', 5432)
    ->database('analytics')
    ->credentials($reference)
    ->readOnly()
    ->build();
```

---

# 205. Same validation pipeline

El builder deberá terminar en:

```text
same normalization/validation rules
```

que config files.

---

# 206. No privileged bypass

Programmatic configuration no deberá saltarse seguridad esencial.

---

# 207. Builder output

Idealmente produce:

```text
ConnectionDefinition
```

válida e inmutable.

---

# 208. Connection templates

Podrá soportarse herencia/config templates para reducir repetición.

Ejemplo:

```yaml
templates:
  postgres-default:
    driver: pgsql
    endpoint:
      port: 5432
    tls:
      enabled: true
```

---

# 209. Template resolution

```text
Template
   │
   ▼
Connection-specific overrides
   │
   ▼
Normalization
```

---

# 210. Template cycles

Deberán detectarse.

---

# 211. Template depth

Puede limitarse para evitar configuraciones incomprensibles.

---

# 212. Prefer composition over deep inheritance

La configuración deberá favorecer claridad.

---

# 213. Environment profiles

Podrán existir perfiles:

```text
development
testing
production
```

pero Database no deberá contener:

```php
if ($environment === 'production') {
    // hidden behavior
}
```

---

# 214. Profile resolution

El Config System produce la configuración final.

Database sólo consume el resultado.

---

# 215. Environment variable mapping

Ejemplo:

```text
DB_HOST
```

se resuelve en Config Layer.

No en:

```text
ConnectionFactory
```

---

# 216. No dotenv dependency

Database core no deberá depender de Dotenv.

---

# 217. Connection URL parser

El parser de URL podrá ser extensible por driver family.

---

# 218. URL scheme mapping

Ejemplos:

```text
mysql://
mariadb://
postgresql://
sqlite://
```

se convertirán a Driver/Platform hints apropiados.

---

# 219. Scheme is not necessarily DriverId

```text
postgresql://
```

puede resolverse a un driver predeterminado:

```text
pdo.pgsql
```

mediante policy.

---

# 220. Driver selection policy

Cuando varios drivers soportan la misma plataforma:

```text
PostgreSQL
├── PDO
└── future native async driver
```

deberá existir una selección explícita o default configurado.

---

# 221. No hidden driver switch

Una actualización de framework no deberá cambiar silenciosamente de Driver si eso modifica semántica importante.

---

# 222. Driver preference

Puede existir:

```text
DriverSelectionPolicy
```

con defaults versionados.

---

# 223. Platform portability

Configuración portable deberá evitar opciones específicas salvo que el usuario las solicite.

---

# 224. Native escape hatch

Cuando sean necesarias:

```text
native options
```

seguirán disponibles.

---

# 225. Connection security validation

La configuración deberá impedir combinaciones peligrosas conocidas.

Ejemplo:

```text
remote endpoint
TLS required
verify_peer = false
```

puede producir warning/error según policy.

---

# 226. Password in URL warning

Si un secret aparece directamente en un DSN de configuración persistente podrá emitirse recomendación de usar:

```text
CredentialReference
```

---

# 227. Sensitive configuration classification

Cada campo del schema podrá marcarse:

```text
PUBLIC
INTERNAL
SENSITIVE
SECRET
```

---

# 228. Diagnostic rendering

El renderer consulta esa clasificación antes de imprimir valores.

---

# 229. Native option sensitivity

Driver extensions deberán poder marcar opciones nativas como secretas.

---

# 230. Connection definition serialization

Si se serializa para config cache, deberá respetar sensibilidad.

---

# 231. No accidental `var_dump` secret

Objetos sensibles podrán implementar representación redacted.

---

# 232. Connection resolution diagnostics

Una resolución podrá producir un trace como:

```text
Requested:
  connection: null
  intent: READ

Transaction affinity:
  none

Scoped definition:
  none

Topology:
  not requested

Default:
  primary

Resolved:
  primary

Role:
  PRIMARY
```

---

# 233. Alias diagnostics

```text
Requested:
  reports

Alias:
  reports → analytics

Resolved:
  analytics
```

---

# 234. Capability rejection diagnostics

```text
Requested:
  analytics

Intent:
  WRITE

Resolved role:
  READ_ONLY

Result:
  rejected

Reason:
  connection does not satisfy WRITE capability
```

---

# 235. Transaction conflict diagnostics

```text
Transaction:
  pinned to primary

Requested:
  analytics

Result:
  rejected

Reason:
  connection conflicts with active transaction affinity
```

---

# 236. Diagnostics cost

Detailed traces deberán ser:

```text
optional
lazy
sampled
```

en producción.

---

# 237. Resolution telemetry

Métricas posibles:

```text
database.connection.resolution.count
database.connection.resolution.failure
database.connection.resolution.duration
database.connection.alias.hit
database.connection.dynamic_resolution
```

---

# 238. Cardinality controls

No utilizar por defecto:

```text
tenant database name
tenant ID
full host
```

como labels de métricas.

---

# 239. Resolution events

Podrán existir:

```text
ConnectionResolutionStarted
ConnectionResolved
ConnectionResolutionRejected
ConnectionDefinitionResolved
```

---

# 240. Event immutability

Los eventos deberán ser preferentemente inmutables.

---

# 241. No configuration mutation by event listener

Un listener no deberá modificar una `ConnectionDefinition` frozen.

---

# 242. Extension resolution hooks

Las extensiones deberán usar:

```text
registered resolvers/providers
```

en lugar de mutar el resultado desde listeners arbitrarios.

---

# 243. Thread/coroutine safety

Toda resolución contextual deberá funcionar bajo:

```text
FrankenPHP workers
RoadRunner workers
OpenSwoole coroutines
Fibers
parallel tests
```

---

# 244. No process-global current connection

Prohibido:

```php
static $currentConnection = 'primary';
```

---

# 245. No process-global tenant definition

Prohibido:

```php
static $tenantDatabase;
```

---

# 246. Scoped resolution

```text
ExecutionScope A
       │
       ▼
Resolution Context A

ExecutionScope B
       │
       ▼
Resolution Context B
```

---

# 247. Same static definitions

Ambos scopes pueden compartir:

```text
Frozen Definition Registry
```

porque es inmutable.

---

# 248. Different dynamic definitions

Pero:

```text
Scoped Definition A
≠
Scoped Definition B
```

---

# 249. Configuration and persistent workers

La configuración compilada puede sobrevivir múltiples requests.

---

# 250. Safe persistent state

Puede persistir:

```text
ConnectionDefinitionRegistry
Alias Registry
Config Schema
Compiled definitions
Driver mapping
```

---

# 251. Unsafe persistent state

No:

```text
current tenant
current connection lease
current transaction
resolved secret plaintext
request-specific override
```

---

# 252. Secret caching

Si un CredentialResolver cachea secrets, deberá tener una policy explícita de:

```text
TTL
rotation
memory protection
invalidation
```

y pertenecer al Security/Secret subsystem.

---

# 253. Database core assumption

Database no deberá asumir que un secret reference es eterno.

---

# 254. Resolution and retries

Connection resolution puede repetirse después de un fallo si Resilience solicita otro target.

---

# 255. Retry distinction

```text
resolve again
≠
repeat database operation
```

---

# 256. Failover resolution

En una futura arquitectura:

```text
Execution fails
      │
      ▼
Failure Classifier
      │
      ▼
Failover Policy
      │
      ▼
Topology chooses alternative
      │
      ▼
ConnectionResolver
```

---

# 257. Manager does not invent failover

No deberá hacer:

```text
primary missing
→ choose any connection
```

---

# 258. Connection configuration inheritance

Podrá existir herencia simple:

```yaml
connections:
  base-postgres:
    abstract: true
    driver: pgsql

  primary:
    extends: base-postgres
    database: app
```

---

# 259. Abstract definitions

Si se soportan templates dentro de `connections`, deberán marcarse explícitamente como no resolubles.

---

# 260. Prefer separate templates

Arquitectónicamente es más claro:

```text
templates
connections
```

como namespaces diferentes.

---

# 261. Connection tags

Podrán existir tags:

```text
region: mx
environment: production
purpose: reporting
```

para Topology/operations.

---

# 262. Tags are metadata

No deberán determinar comportamiento crítico mediante strings mágicos.

---

# 263. Structured properties first

Preferir:

```text
role = REPLICA
```

a:

```text
tag = replica
```

para semántica central.

---

# 264. Region

Podrá modelarse como value object si participa en routing.

---

# 265. Weight and priority

Pertenecen principalmente a Topology target configuration.

No al Connection core salvo metadata necesaria.

---

# 266. Connection description

Puede existir un label humano para diagnostics.

No forma parte de la identity técnica.

---

# 267. Configuration ownership

Cada configuración deberá tener un único owner.

Ejemplo:

```text
connect timeout
→ Connection

query timeout
→ Execution

transaction isolation
→ Transaction

replica weight
→ Topology
```

---

# 268. Avoid duplicate configuration

No deberá existir simultáneamente:

```text
connection.query_timeout
query.timeout
```

con semántica superpuesta.

---

# 269. Cross-domain configuration references

Cuando sea necesario, usar referencias tipadas.

---

# 270. Configuration compilation boundary

Una vez compilada:

```text
External config syntax
```

deja de importar.

---

# 271. Runtime representation

Runtime deberá consumir:

```text
ConnectionDefinition
ConnectionIdentity
ConnectionRole
DatabaseEndpoint
CredentialReference
ConnectionSessionProfile
ConnectionSecurityConfiguration
```

---

# 272. No stringly typed runtime

Evitar:

```php
$config['role'] === 'read_only';
```

en hot paths.

---

# 273. Enum/value object runtime

Preferir:

```php
$definition->role === ConnectionRole::READ_ONLY;
```

---

# 274. Connection configuration architecture

```text
                 External Sources
                        │
       ┌────────────────┼─────────────────┐
       ▼                ▼                 ▼
 Config Files       Environment       Secret Refs
       │                │                 │
       └────────────────┼─────────────────┘
                        ▼
                 Config Integration
                        │
                        ▼
                  Raw Connection
                  Configuration
                        │
                        ▼
                    Normalizer
                        │
                        ▼
                     Validator
                        │
                        ▼
              Definition Compiler
                        │
                        ▼
              ConnectionDefinition
                        │
                        ▼
             Frozen Definition Registry
```

---

# 275. Connection resolution architecture

```text
               Connection Requirement
                        │
                        ▼
                Resolution Request
                        │
                        ▼
            Mandatory Affinity Check
                        │
                        ▼
              Explicit Identity?
                 │            │
                yes           no
                 │            │
                 ▼            ▼
          Definition       Scoped /
          Resolution       Integration
                              │
                              ▼
                           Topology
                              │
                              ▼
                            Default
                              │
                              ▼
                     Capability Match
                              │
                              ▼
                    ConnectionResolution
                              │
                              ▼
                    Logical Connection
```

---

# 276. Configuration vs resolution

Estas dos fases no deberán confundirse.

```text
Configuration
=
what connections exist

Resolution
=
which connection should this operation use
```

---

# 277. Resolution vs acquisition

También:

```text
Resolution
=
which logical connection

Acquisition
=
which physical resource
```

---

# 278. Four-layer model

```text
Configuration
      │
      ▼
Definition
      │
      ▼
Resolution
      │
      ▼
Logical Connection
      │
      ▼
Acquisition
      │
      ▼
Physical Connection
```

---

# 279. Architectural invariants

## DB-CCR-001

Database internals no consultarán variables de entorno directamente.

## DB-CCR-002

Connection runtime no consumirá arrays de configuración sin normalizar.

## DB-CCR-003

Toda conexión estática tendrá una `ConnectionDefinition` validada.

## DB-CCR-004

Las definiciones globales serán inmutables después del bootstrap.

## DB-CCR-005

Los secretos se representarán preferentemente mediante referencias.

## DB-CCR-006

Los secretos nunca aparecerán en diagnostics públicos.

## DB-CCR-007

DSN será formato de entrada, no modelo runtime.

## DB-CCR-008

ConnectionIdentity y ConnectionAlias serán conceptos separados.

## DB-CCR-009

Los ciclos de aliases serán inválidos.

## DB-CCR-010

Una conexión explícita inexistente no caerá silenciosamente al default.

## DB-CCR-011

Driver será seleccionado mediante DriverId/registry, no condicionales dispersos.

## DB-CCR-012

Driver, Dialect y Platform permanecerán separados.

## DB-CCR-013

Los endpoints no asumirán siempre host/port.

## DB-CCR-014

Portable options y native options estarán separadas.

## DB-CCR-015

TLS requerido nunca degradará silenciosamente a plaintext.

## DB-CCR-016

SessionProfile describirá el estado baseline de una conexión reutilizable.

## DB-CCR-017

Pool configuration no implicará persistent connection automáticamente.

## DB-CCR-018

Una definición dinámica será scoped y validada.

## DB-CCR-019

Multitenancy no mutará el registry global.

## DB-CCR-020

Resolution será determinista salvo factores dinámicos explícitos.

## DB-CCR-021

Transaction affinity tendrá precedencia sobre una selección incompatible.

## DB-CCR-022

Capability checks reemplazarán driver-name conditionals.

## DB-CCR-023

Configuration fingerprint no revelará secretos.

## DB-CCR-024

Una nueva configuración se publicará atómicamente.

## DB-CCR-025

Configuration reload no producirá snapshots parcialmente actualizados.

## DB-CCR-026

Connection resolution no abrirá una conexión física por sí misma.

## DB-CCR-027

Configuration validation offline no requerirá un servidor Database disponible.

## DB-CCR-028

Online validation será una fase separada.

## DB-CCR-029

La configuración podrá sobrevivir workers persistentes; el estado contextual no.

## DB-CCR-030

Los overrides de test no contaminarán otros execution scopes.

## DB-CCR-031

La configuración programática utilizará las mismas invariantes que archivos de configuración.

## DB-CCR-032

Los campos sensibles estarán clasificados explícitamente.

## DB-CCR-033

Una extensión registrará su schema antes de la compilación/freeze.

## DB-CCR-034

El runtime no realizará parsing repetitivo de durations, DSN, roles o aliases.

## DB-CCR-035

La resolución devolverá una identity canónica.

## DB-CCR-036

Resolution y Resource Acquisition permanecerán como fases diferentes.

---

# 280. Anti-pattern — getenv in Driver

```php
$host = getenv('DB_HOST');
```

dentro de Driver.

**Prohibido.**

---

# 281. Anti-pattern — global config helper in Connection

```php
config('database.connections.primary');
```

dentro de Connection.

**Prohibido.**

---

# 282. Anti-pattern — arrays through runtime

```php
function connect(array $config)
```

como contrato arquitectónico central.

**Evitar.**

Preferir objetos tipados.

---

# 283. Anti-pattern — DSN everywhere

```text
ORM
→ DSN

Query
→ DSN

Connection
→ DSN

Driver
→ DSN
```

**Prohibido.**

DSN se parsea una vez.

---

# 284. Anti-pattern — secrets in ConnectionDefinition dumps

```text
ConnectionDefinition {
  password: "supersecret"
}
```

**Prohibido.**

---

# 285. Anti-pattern — implicit alias fallback

```text
missing connection
→ maybe alias
→ maybe default
→ maybe first configured connection
```

**Prohibido.**

La precedencia deberá ser formal.

---

# 286. Anti-pattern — mutable global definition

```php
$definitions['primary']['database'] = $tenantDatabase;
```

durante request.

**Prohibido.**

---

# 287. Anti-pattern — driver-specific core config

```php
if ($driver === 'pgsql') {
    // parse one configuration shape
}
```

disperso por Connection core.

Preferir schemas/adapters del Driver.

---

# 288. Anti-pattern — platform inferred forever from DSN

El DSN puede dar un hint.

La Platform efectiva deberá poder validarse contra el servidor real.

---

# 289. Anti-pattern — configuration-driven hidden behavior

Una opción ambigua como:

```text
smart = true
```

deberá evitarse.

Preferir policies explícitas.

---

# 290. Anti-pattern — infinite dynamic registry

```text
tenant 1 → global definition
tenant 2 → global definition
...
tenant 5,000,000 → global definition
```

**Prohibido.**

---

# 291. Anti-pattern — resolving secrets during config dump

Un comando:

```text
db:config show
```

no deberá resolver secrets sólo para mostrarlos.

---

# 292. Testing strategy

El subsystem deberá contar con:

```text
configuration parser tests
normalization tests
validation tests
alias tests
DSN tests
secret redaction tests
driver schema tests
dynamic definition tests
resolution tests
scope isolation tests
configuration generation tests
```

---

# 293. Normalization test

```text
connect_timeout = "5s"

normalize

assert Duration(seconds: 5)
```

---

# 294. Alias cycle test

```text
a → b
b → c
c → a

compile

assert ConnectionAliasCycleException
```

---

# 295. Unknown connection test

```text
resolve("missing")

assert ConnectionNotConfiguredException
```

---

# 296. Default test

```text
default = primary

resolve(null)

assert primary
```

---

# 297. Secret redaction test

```text
credential = secret

diagnostic dump

assert secret absent
```

---

# 298. DSN test

```text
postgresql://user:secret@host:5432/app
```

deberá producir una configuración estructurada equivalente sin filtrar `secret` en diagnostics.

---

# 299. TLS strictness test

```text
TLS required
Driver cannot satisfy requirement

assert configuration/capability failure
```

---

# 300. Dynamic scope test

```text
Scope A
→ dynamic primary A

Scope B
→ dynamic primary B

assert no cross-scope leakage
```

---

# 301. Persistent worker test

```text
Worker
├── Request A → Tenant A definition
├── reset
└── Request B → Tenant B definition

assert:
B cannot access A definition
```

---

# 302. Concurrent resolution test

```text
Coroutine A → definition A
Coroutine B → definition B

same frozen static registry

assert contextual definitions isolated
```

---

# 303. Configuration generation test

```text
Generation 1
→ pool resources created

publish Generation 2

assert:
new resources use generation 2
old resources retire according to policy
```

---

# 304. No eager connection test

```text
load configuration
normalize
validate
compile
resolve logical connection

assert:
Driver::connect() never called
```

---

# 305. Architecture tests

Deberán impedir dependencias como:

```text
Connection Configuration → ORM
Connection Configuration → Query Builder
Connection Configuration → HTTP Request
Connection Configuration → PDO
```

---

# 306. Suggested namespaces

```text
VoltStack\Quantum\Database\Connection
│
├── Configuration
│   ├── RawConnectionConfiguration.php
│   ├── NormalizedConnectionConfiguration.php
│   ├── ConnectionDefinition.php
│   ├── ConnectionDefinitionCompiler.php
│   ├── ConnectionConfigurationNormalizer.php
│   ├── ConnectionConfigurationValidator.php
│   ├── ConnectionConfigurationSchema.php
│   └── ConnectionConfigurationGeneration.php
│
├── Identity
│   ├── ConnectionIdentity.php
│   └── ConnectionAlias.php
│
├── Endpoint
│   ├── DatabaseEndpoint.php
│   ├── TcpDatabaseEndpoint.php
│   ├── UnixSocketDatabaseEndpoint.php
│   ├── FileDatabaseEndpoint.php
│   └── MemoryDatabaseEndpoint.php
│
├── Credential
│   ├── CredentialReference.php
│   ├── CredentialResolverInterface.php
│   └── DatabaseCredential.php
│
├── Security
│   └── ConnectionSecurityConfiguration.php
│
├── Session
│   └── ConnectionSessionProfile.php
│
├── Timeout
│   └── ConnectionTimeoutConfiguration.php
│
├── NativeOption
│   └── ConnectionNativeOptions.php
│
├── Definition
│   ├── ConnectionDefinitionRegistry.php
│   ├── ConnectionDefinitionProviderInterface.php
│   ├── StaticConnectionDefinitionProvider.php
│   └── ScopedConnectionDefinitionProvider.php
│
├── Resolution
│   ├── ConnectionResolverInterface.php
│   ├── ConnectionResolver.php
│   ├── ConnectionResolutionRequest.php
│   ├── ConnectionResolutionContext.php
│   ├── ConnectionResolution.php
│   ├── ConnectionResolutionPolicy.php
│   ├── ConnectionResolutionSource.php
│   └── ConnectionResolutionTrace.php
│
├── Capability
│   └── ConnectionCapabilityMatcher.php
│
└── Exception
```

---

# 307. Configuration object graph

```text
DatabaseConfiguration
        │
        ▼
ConnectionConfiguration
        │
        ├── DefaultConnectionIdentity
        │
        ├── ConnectionDefinitionRegistry
        │       │
        │       ├── primary
        │       ├── analytics
        │       └── legacy
        │
        ├── ConnectionAliasRegistry
        │
        ├── Pool References
        │
        └── Resolution Policy
```

---

# 308. ConnectionDefinition object graph

```text
ConnectionDefinition
│
├── ConnectionIdentity
├── DriverId
├── DatabaseEndpoint
├── DatabaseName
├── CredentialReference
├── ConnectionRole
├── PlatformHint
├── ConnectionTimeoutConfiguration
├── ConnectionSecurityConfiguration
├── ConnectionSessionProfile
├── PoolPolicyReference
├── ConnectionNativeOptions
├── ConnectionDefinitionFingerprint
└── ConfigurationGeneration
```

No todos los campos serán obligatorios para todos los Drivers.

---

# 309. Resolution object graph

```text
ConnectionResolutionRequest
│
├── Explicit Identity?
├── ConnectionIntent
├── Required Capabilities
├── Consistency Requirement?
└── Affinity?

        │
        ▼

ConnectionResolutionContext
│
├── Execution Scope
├── Transaction Context
├── Scoped Providers
└── Integration State

        │
        ▼

ConnectionResolver

        │
        ▼

ConnectionResolution
│
├── Canonical Identity
├── ConnectionDefinition
├── Effective Role
├── Resolution Source
└── Diagnostic Reason
```

---

# 310. Complete bootstrap flow

```text
Database Extension Registration
            │
            ▼
Connection Schemas Registered
            │
            ▼
Configuration Sources Loaded
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
Resolve Aliases
            │
            ▼
Validate Cross References
            │
            ▼
Compile Connection Definitions
            │
            ▼
Calculate Fingerprints
            │
            ▼
Build Definition Registry
            │
            ▼
Freeze Registry
            │
            ▼
ConnectionManager Ready
```

No physical connection es necesaria en este proceso.

---

# 311. Complete runtime flow

```text
Database Operation
      │
      ▼
ConnectionRequirement
      │
      ▼
ConnectionResolutionRequest
      │
      ▼
ConnectionResolver
      │
      ├── Transaction affinity
      ├── Explicit identity
      ├── Scoped definition
      ├── Integration
      ├── Topology
      └── Default
      │
      ▼
Capability Match
      │
      ▼
ConnectionResolution
      │
      ▼
ConnectionManager
      │
      ▼
Logical Connection
```

Después, y sólo cuando sea necesario:

```text
Logical Connection
      │
      ▼
Resource Acquisition
      │
      ▼
ConnectionLease
      │
      ▼
Native Connection
```

---

# 312. Persistent runtime flow

```text
Worker Boot
    │
    ├── Compiled Connection Configuration
    ├── Frozen Definition Registry
    ├── Alias Registry
    └── ConnectionManager
    │
    ├─────────────────────────────────────┐
    │                                     │
    ▼                                     ▼
Execution Scope A                  Execution Scope B
    │                                     │
Scoped Definitions A              Scoped Definitions B
    │                                     │
Resolution Context A              Resolution Context B
    │                                     │
Logical Connection A              Logical Connection B
    │                                     │
    └──────────── no state leak ───────────┘
```

---

# 313. Architectural equation

```text
Connection Configuration
=
External Input
+
Normalization
+
Validation
+
Compilation
+
Typed Definitions
+
Freeze
```

y:

```text
Connection Resolution
=
Requirement
+
Context
+
Resolution Policy
+
Capability Matching
=
ConnectionResolution
```

---

# 314. Master separation

```text
Configuration
≠
Resolution
≠
Acquisition
≠
Execution
```

Formalmente:

```text
Configuration
      │
      ▼
Definition
      │
      ▼
Resolution
      │
      ▼
Logical Connection
      │
      ▼
Acquisition
      │
      ▼
Physical Connection
      │
      ▼
Execution
```

---

# 315. Criterios de aceptación

La implementación se considerará arquitectónicamente correcta cuando:

```text
configuration can be validated offline
configuration does not open database connections
runtime receives typed definitions
DSNs are parsed before runtime
secrets remain redacted
default connection resolves deterministically
explicit missing connection fails
aliases are canonicalized
alias cycles fail
driver IDs are validated
endpoints support non-TCP databases
TLS requirements cannot silently downgrade
session profiles are explicit
dynamic definitions remain scoped
tenant definitions do not mutate global registry
transaction affinity cannot be bypassed
capabilities drive compatibility
resolution does not acquire physical resources
configuration snapshots are immutable
persistent workers do not leak contextual definitions
parallel contexts remain isolated
```

---

# 316. Principio final

> La configuración describe conexiones posibles; la resolución selecciona una conexión lógica; la adquisición obtiene temporalmente un recurso físico.

En forma compacta:

```text
Configuration
    │
    ▼
Definition
    │
    ▼
Resolution
    │
    ▼
Logical Connection
    │
    ▼
Lease
    │
    ▼
Native Resource
```

Cada fase deberá conservar su propia responsabilidad.

---

# 317. Conclusión

`13_DATABASE_CONNECTION_CONFIGURATION_AND_RESOLUTION.md` establece el puente entre la configuración declarativa de VoltStack y la infraestructura runtime de Database.

El sistema evitará el modelo tradicional:

```text
read env
→ build DSN
→ new PDO
```

disperso por la aplicación.

VoltStack utilizará:

```text
External Configuration
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
Immutable ConnectionDefinition
        │
        ▼
Deterministic Resolution
        │
        ▼
Logical Connection
        │
        ▼
Lazy Resource Acquisition
```

Esto permitirá que configuración, pooling, multitenancy, replicas, testing y workers persistentes evolucionen sin contaminar:

```text
Query Engine
ORM
Driver
Execution
```

con lógica de configuración ad hoc.

---

# 318. Siguiente documento

El siguiente documento será:

```text
14_DATABASE_CONNECTION_POOLING_SYSTEM.md
```

y deberá definir específicamente:

```text
Connection Pool architecture
Physical Connection ownership
Pool identity
Pool registry
Pool manager
Pool configuration
minimum/maximum capacity
idle resources
active resources
ConnectionLease
lease ownership
acquisition
release
waiting
backpressure
acquisition timeout
resource creation
resource retirement
max lifetime
max idle time
max query/use count
health validation
stale connection detection
connection reset before reuse
tainted/broken resources
transaction pinning
streaming cursor leases
pool shutdown
credential/configuration generation changes
persistent runtime behavior
FrankenPHP integration
RoadRunner compatibility
OpenSwoole concurrency
metrics
telemetry
diagnostics
security
failure handling
testing
```

La regla central deberá ser:

```text
Connection Pool
=
Physical Resource Reuse
```

y nunca:

```text
Connection Pool
=
Logical Connection Registry
```

La separación completa continuará siendo:

```text
ConnectionManager
      │
      ▼
Logical Connection
      │
      ▼
Resource Provider
      │
      ▼
Connection Pool
      │
      ▼
ConnectionLease
      │
      ▼
Physical Connection
```