# 229_DATABASE_CREDENTIAL_SECURITY_SYSTEM.md

# VoltStack Quantum Database
## Credential Security System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 229 — Credential Security System  
**Bloque:** 22 — Security  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `228_DATABASE_QUERY_INPUT_SECURITY_SYSTEM.md`  
**Siguiente documento:** `230_DATABASE_CONNECTION_SECURITY_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura oficial para la gestión segura de credenciales utilizadas por VoltStack Database.

El sistema deberá proteger secretos utilizados para:

- conexiones de aplicación;
- migraciones;
- administración;
- réplicas;
- backups;
- restore;
- introspección;
- health checks;
- operaciones de mantenimiento;
- tenants;
- servicios internos;
- credenciales dinámicas;
- tokens temporales;
- proveedores cloud.

La regla central será:

> **Una credencial de base de datos no es configuración ordinaria. Es un secreto con propósito, alcance, autoridad, procedencia, ciclo de vida y política de exposición explícitos.**

---

# 2. Problema fundamental

Una configuración tradicional puede contener:

```php
[
    'host' => 'db.internal',
    'database' => 'production',
    'username' => 'voltstack',
    'password' => 'secret123',
]
```

Esto mezcla:

```text
Connection Configuration
+
Credential Material
```

VoltStack deberá separarlos.

La configuración podrá conocer:

```text
qué credencial necesita
```

pero no necesariamente:

```text
el secreto mismo
```

---

# 3. Modelo objetivo

```text
Database Configuration
        │
        ├── host
        ├── database
        ├── port
        └── credential reference
                    │
                    ▼
             Credential Resolver
                    │
                    ▼
              Secret Provider
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
 Environment     Vault      Cloud Secrets
        │           │           │
        └───────────┼───────────┘
                    ▼
             Secret Material
                    │
                    ▼
          Credential Validation
                    │
                    ▼
           Connection Factory
                    │
                    ▼
             DB Connection
```

---

# 4. Credential ≠ Configuration

Distinción obligatoria:

```text
Database Configuration
≠
Database Credential
```

La configuración describe:

```text
where/how to connect
```

La credencial describe:

```text
under which database identity
```

---

# 5. Credential ≠ Secret

También:

```text
Credential
≠
Secret
```

Una credencial puede estar compuesta por:

```text
username
password
token
certificate
private key reference
authentication metadata
expiration
```

Mientras que un `SecretValue` representa material sensible concreto.

---

# 6. CredentialReference ≠ CredentialValue

VoltStack distinguirá:

```text
CredentialReference
```

de:

```text
ResolvedCredential
```

Ejemplo:

```text
vault://database/production/application
```

es una referencia.

No es la contraseña.

---

# 7. Regla de exposición mínima

La mayor parte del framework deberá trabajar con:

```text
CredentialReference
```

y no con:

```text
SecretValue
```

El secreto solo deberá resolverse cerca del punto donde sea realmente necesario.

---

# 8. Principio de resolución tardía

```text
Configuration Load
       ↓
CredentialReference
       ↓
Connection Requested
       ↓
Resolve Secret
       ↓
Create Connection
```

No:

```text
Application Boot
       ↓
Resolve every database secret
       ↓
Store everything indefinitely
```

---

# 9. Threat model

El sistema deberá contemplar:

```text
credential leakage
logs
stack traces
debug dumps
telemetry
serialization
configuration exports
exception messages
process memory
core dumps
source repositories
environment exposure
CI/CD logs
cache leaks
worker reuse
tenant crossover
stale credentials
credential replay
overprivileged credentials
secret provider compromise
rotation races
connection pool reuse
```

---

# 10. Secret lifecycle

Una credencial podrá recorrer:

```text
REFERENCE
   ↓
RESOLVING
   ↓
RESOLVED
   ↓
ACTIVE
   ↓
EXPIRING
   ↓
EXPIRED
   ↓
DISPOSED
```

También:

```text
RESOLVING → FAILED
```

---

# 11. CredentialReference

Contrato conceptual:

```php
final readonly class CredentialReference
{
    public function __construct(
        public CredentialProviderId $provider,
        public CredentialKey $key,
        public CredentialPurpose $purpose,
    ) {}
}
```

---

# 12. CredentialReference seguro

Debe ser posible:

```php
(string) $reference;
```

sin revelar secretos.

Ejemplo seguro:

```text
vault:database/production/application
```

---

# 13. SecretValue

El material secreto deberá encapsularse.

```php
final class SecretValue
{
    private string $value;

    public function reveal(SecretAccessContext $context): string
    {
        // controlled access
    }
}
```

---

# 14. SecretValue no deberá comportarse como string ordinario

Evitar APIs que permitan:

```php
echo $secret;
```

o:

```php
logger()->debug($secret);
```

---

# 15. __toString()

`SecretValue` no deberá implementar un `__toString()` que revele material secreto.

---

# 16. Debug representation

```php
var_dump($secret);
```

deberá producir conceptualmente:

```text
SecretValue {
    value: [REDACTED]
}
```

---

# 17. ResolvedCredential

```php
final class ResolvedCredential
{
    public function __construct(
        public readonly CredentialIdentity $identity,
        private SecretValue $secret,
        public readonly CredentialMetadata $metadata,
    ) {}
}
```

---

# 18. CredentialIdentity

Puede contener información no secreta como:

```text
username
authentication mechanism
database role
```

según política.

---

# 19. CredentialMetadata

Podrá contener:

```text
provider
purpose
issuedAt
expiresAt
generation
leaseId
renewable
scope
```

sin almacenar el secreto.

---

# 20. CredentialPurpose

VoltStack deberá distinguir para qué se usa una credencial.

```php
enum CredentialPurpose
{
    case APPLICATION_READ_WRITE;
    case APPLICATION_READ_ONLY;
    case MIGRATION;
    case SCHEMA_INTROSPECTION;
    case BACKUP;
    case RESTORE;
    case ADMINISTRATION;
    case HEALTH_CHECK;
    case REPLICATION;
    case MAINTENANCE;
    case CUSTOM;
}
```

---

# 21. Principle of least privilege

No deberá utilizarse automáticamente:

```text
ADMINISTRATION
```

cuando basta:

```text
APPLICATION_READ_WRITE
```

---

# 22. Separate operational credentials

Arquitectura recomendada:

```text
Application
    ↓
Limited runtime credential

Migration CLI
    ↓
Migration credential

Backup system
    ↓
Backup credential

Administrative tooling
    ↓
Administrative credential
```

---

# 23. Runtime application credential

Idealmente solo podrá realizar las operaciones necesarias para el workload normal.

---

# 24. Migration credential

Puede necesitar:

```text
CREATE
ALTER
DROP
INDEX
```

que la aplicación normal no debería poseer.

---

# 25. Backup credential

Puede necesitar permisos de lectura más amplios, pero no necesariamente escritura.

---

# 26. Restore credential

Puede requerir privilegios distintos del backup.

Por tanto:

```text
BACKUP
≠
RESTORE
```

---

# 27. Credential provider architecture

```text
CredentialReference
        ↓
CredentialResolver
        ↓
CredentialProviderRegistry
        ↓
CredentialProvider
        ↓
ResolvedCredential
```

---

# 28. CredentialProvider

```php
interface CredentialProvider
{
    public function resolve(
        CredentialReference $reference,
        CredentialResolutionContext $context,
    ): ResolvedCredential;
}
```

---

# 29. Provider responsibilities

Un provider puede:

```text
retrieve
authenticate to secret backend
validate format
obtain lease metadata
return credential material
```

No deberá crear conexiones de base de datos.

---

# 30. Provider ≠ Connection

Regla:

```text
CredentialProvider
≠
ConnectionFactory
```

---

# 31. CredentialResolver

Coordina:

```text
reference validation
provider selection
purpose validation
resolution policy
expiration checks
telemetry
redaction
```

---

# 32. CredentialProviderRegistry

```php
interface CredentialProviderRegistry
{
    public function get(
        CredentialProviderId $provider
    ): CredentialProvider;
}
```

---

# 33. Registry freeze

Los providers deberán registrarse durante bootstrap y congelarse cuando corresponda.

Input externo nunca deberá registrar providers.

---

# 34. Provider types

VoltStack podrá soportar:

```text
EnvironmentCredentialProvider
FileCredentialProvider
EncryptedConfigCredentialProvider
VaultCredentialProvider
CloudSecretCredentialProvider
DynamicCredentialProvider
CustomCredentialProvider
```

---

# 35. Environment provider

Ejemplo:

```text
DB_USERNAME
DB_PASSWORD
```

seguirá siendo válido para despliegues sencillos.

Pero:

> **Environment variable support no convierte las variables de entorno en un secret manager.**

---

# 36. Environment limitations

Las variables pueden aparecer en:

```text
process inspection
debugging tools
container metadata
deployment configuration
crash diagnostics
```

según entorno.

---

# 37. Production recommendation

VoltStack podrá recomendar secret providers especializados para despliegues sensibles.

---

# 38. FileCredentialProvider

Solo deberá aceptar archivos con políticas estrictas.

Consideraciones:

```text
ownership
permissions
symlink handling
path validation
atomic replacement
```

---

# 39. Arbitrary secret path

Input externo no deberá poder elegir libremente:

```text
/etc/...
```

como secret reference.

---

# 40. Encrypted configuration

VoltStack podrá soportar:

```text
encrypted database credentials
```

dentro de configuración.

Pero surge una cuestión:

```text
Where is the decryption key?
```

---

# 41. Encryption does not eliminate secret management

```text
Encrypted Secret
+
Decryption Key
=
Secret Management Problem
```

Por tanto, encrypted config será una estrategia, no una solución universal.

---

# 42. External secret managers

La arquitectura deberá permitir adaptadores para sistemas como:

```text
HashiCorp Vault
AWS Secrets Manager
Azure Key Vault
Google Cloud Secret Manager
other providers
```

sin acoplar el núcleo Database a ninguno.

---

# 43. Core provider agnostic

`Quantum/Database` dependerá de contratos.

Integraciones concretas podrán vivir en paquetes separados.

---

# 44. Dynamic credentials

Algunos sistemas pueden generar:

```text
temporary database username
temporary password/token
expiration
lease
```

VoltStack deberá modelarlos explícitamente.

---

# 45. Static vs dynamic credential

```php
enum CredentialLifetimeKind
{
    case STATIC;
    case ROTATABLE;
    case LEASED;
    case EPHEMERAL;
}
```

---

# 46. Expiration

Una credencial podrá incluir:

```php
$credential->metadata->expiresAt;
```

---

# 47. Expiration ≠ refresh time

VoltStack podrá calcular:

```text
refreshAt < expiresAt
```

para evitar esperar al último instante.

---

# 48. Refresh margin

Ejemplo conceptual:

```text
expiration = 14:00
refresh margin = 5 min
refresh at = 13:55
```

---

# 49. CredentialLease

```php
final readonly class CredentialLease
{
    public function __construct(
        public CredentialLeaseId $id,
        public DateTimeImmutable $issuedAt,
        public DateTimeImmutable $expiresAt,
        public bool $renewable,
    ) {}
}
```

---

# 50. Lease ≠ Connection lifetime

Una conexión creada con una credencial temporal puede seguir existiendo cuando la credencial haya expirado, dependiendo del DBMS.

Eso requiere política explícita.

---

# 51. Credential generation

Cada rotación podrá producir:

```text
CredentialGeneration
```

Ejemplo:

```text
generation 41
generation 42
```

---

# 52. Generation use

Las conexiones podrán registrar:

```text
credentialGeneration=42
```

sin almacenar el secreto.

---

# 53. Credential generation ≠ secret

Seguro para telemetry/debug cuando corresponda.

---

# 54. Rotation architecture

```text
Credential Provider
      ↓
New Generation
      ↓
Credential Rotation Coordinator
      ↓
Connection Manager
      ↓
Mark old connections stale
      ↓
Drain / replace
      ↓
New connections use new credential
```

---

# 55. Rotation ≠ immediate global disconnect

Cerrar todas las conexiones instantáneamente puede causar:

```text
request failures
transaction interruption
thundering herd
```

---

# 56. Rotation policy

Podrán existir:

```php
enum CredentialRotationStrategy
{
    case NEW_CONNECTIONS_ONLY;
    case GRACEFUL_DRAIN;
    case IMMEDIATE_REVOKE;
    case CUSTOM;
}
```

---

# 57. Graceful drain

Normalmente:

```text
existing in-flight transaction
        ↓
finish
        ↓
connection not returned as reusable
        ↓
close
```

---

# 58. Immediate revoke

Para compromiso de credenciales puede requerirse:

```text
close as soon as safely possible
```

según política.

---

# 59. Transaction safety

VoltStack no deberá interrumpir silenciosamente una transacción solo porque llegó una nueva generación, salvo política explícita de emergencia.

---

# 60. Old generation

Una conexión de generación anterior podrá marcarse:

```text
STALE_CREDENTIAL
```

---

# 61. Pool integration

El Connection Pooling System deberá conocer:

```text
credential generation
expiration
rotation status
```

sin conocer el secreto.

---

# 62. Pool checkout

Antes de reutilizar una conexión podrá verificarse:

```text
connection credential generation
vs
required generation
```

---

# 63. Stale pooled connection

Puede:

```text
DRAIN
REJECT
ALLOW_UNTIL_DEADLINE
```

según rotation policy.

---

# 64. Secret cache

Resolver el mismo secreto constantemente puede ser costoso.

Podrá existir caching controlado.

Pero:

```text
Secret Cache
≠
normal framework cache
```

---

# 65. No Redis secret caching by default

VoltStack no deberá enviar credenciales automáticamente a:

```text
Redis
Memcached
distributed cache
```

---

# 66. In-memory secret cache

Podrá existir:

```text
process-local bounded credential cache
```

con lifetime corto.

---

# 67. Secret cache key

Debe utilizar:

```text
provider
credential key
purpose
generation/version context
```

y nunca el secreto mismo.

---

# 68. Secret cache lifetime

No deberá superar la vigencia de la credencial.

---

# 69. Credential cache entry

```php
final class CredentialCacheEntry
{
    public function __construct(
        private ResolvedCredential $credential,
        public readonly DateTimeImmutable $refreshAt,
        public readonly DateTimeImmutable $expiresAt,
    ) {}
}
```

---

# 70. Cache invalidation

Eventos como:

```text
rotation
revocation
provider notification
expiration
policy change
```

podrán invalidarlo.

---

# 71. Credential cache ≠ Connection pool

Uno almacena material/autenticación temporal.

El otro administra conexiones establecidas.

---

# 72. In-memory protection

PHP no puede garantizar de forma universal que un secreto desaparezca físicamente de RAM inmediatamente.

VoltStack no deberá prometerlo.

---

# 73. Best-effort disposal

El sistema podrá:

```text
release references
overwrite mutable buffers where practical
avoid unnecessary copies
shorten lifetime
```

pero deberá documentarse como best effort.

---

# 74. Immutable PHP strings

Las strings PHP pueden copiarse internamente.

Por ello:

> **Zeroization absoluta de secretos no puede tratarse como garantía portable del framework.**

---

# 75. Security objective

La estrategia principal será:

```text
minimize exposure
minimize copies
minimize lifetime
minimize observers
```

---

# 76. SecretAccessContext

Revelar un secreto deberá tener propósito.

```php
final readonly class SecretAccessContext
{
    public function __construct(
        public SecretConsumer $consumer,
        public CredentialPurpose $purpose,
    ) {}
}
```

---

# 77. Allowed consumers

Ejemplo:

```text
ConnectionFactory
TLSCredentialLoader
CredentialRotationValidator
```

No:

```text
DebugToolbar
Logger
QueryProfiler
```

---

# 78. Secret reveal boundary

Idealmente:

```text
ResolvedCredential
       ↓
ConnectionFactory
       ↓
Driver authentication
```

es el punto principal de revelación.

---

# 79. Driver boundary

El Driver puede requerir:

```text
username/password
```

en formato concreto.

El adapter deberá mantener el secreto solo durante el tiempo necesario.

---

# 80. DSN security

Nunca construir logs como:

```text
mysql://user:password@host/database
```

---

# 81. Safe DSN

Representación diagnóstica:

```text
mysql://user:[REDACTED]@host/database
```

o preferiblemente:

```text
mysql://user@host/database
```

---

# 82. URL encoded secrets

La redacción debe considerar secretos codificados.

No basta buscar únicamente el plaintext original.

---

# 83. Structured logging

Preferir:

```php
[
    'host' => 'db.internal',
    'database' => 'app',
    'credential_ref' => 'vault:db/app',
]
```

a registrar DSNs completos.

---

# 84. Credential redaction architecture

```text
Sensitive Value
      ↓
Redaction Registry
      ↓
Logger
Telemetry
Debugger
Exception Renderer
Profiler
```

---

# 85. SecretRedactor

```php
interface SecretRedactor
{
    public function redact(mixed $value): mixed;
}
```

---

# 86. Redaction by type

El sistema deberá reconocer tipos como:

```text
SecretValue
ResolvedCredential
CredentialMaterial
PrivateKeyMaterial
AuthenticationToken
```

---

# 87. Type-based redaction

Es preferible a depender exclusivamente de búsqueda textual.

---

# 88. String fallback redaction

Podrá existir una capa adicional para strings conocidas como sensibles.

Pero no será la única defensa.

---

# 89. Credential logging

Por default:

```text
CredentialReference → maybe safe
CredentialIdentity → policy-dependent
SecretValue → REDACTED
ResolvedCredential → REDACTED
```

---

# 90. Username sensitivity

Incluso el username puede considerarse sensible en ciertos entornos.

Debe existir política.

---

# 91. Exception safety

Incorrecto:

```text
Authentication failed using password "abc123"
```

Correcto:

```text
Database authentication failed.
```

---

# 92. Internal exception context

Puede incluir:

```text
provider
credential reference fingerprint
purpose
generation
host
```

pero nunca secret material.

---

# 93. Provider exceptions

Una excepción de proveedor externo podría incluir el secreto accidentalmente.

VoltStack deberá sanear mensajes antes de propagarlos a capas observables.

---

# 94. Exception wrapping

```text
Provider Exception
      ↓
Sanitizer
      ↓
CredentialResolutionException
```

---

# 95. Original exception

Conservar `$previous` puede reintroducir secretos en logs.

Debe existir política segura para chaining.

---

# 96. Safe exception cause

No deberá asumirse que una excepción third-party es segura.

---

# 97. Telemetry

Métricas permitidas:

```text
provider
purpose
resolution latency
resolution result
generation
credential age bucket
refresh result
rotation result
```

---

# 98. Telemetry forbidden data

No:

```text
password
token
private key
raw certificate private material
full secret URI containing embedded secret
```

---

# 99. Credential reference telemetry

Incluso referencias podrán tener información sensible.

Preferir:

```text
credential_reference_hash
```

cuando corresponda.

---

# 100. Query telemetry

Nunca deberá incluir credenciales.

---

# 101. Connection telemetry

Podrá indicar:

```text
auth_mode=password
credential_generation=42
```

pero no el secreto.

---

# 102. Debug toolbar

Podrá mostrar:

```text
Credential Provider: Vault
Purpose: APPLICATION_READ_WRITE
Generation: 42
Expires: 13:55
Status: ACTIVE
```

No:

```text
Password: ...
Token: ...
```

---

# 103. Debug mode ≠ secret exposure permission

Regla obligatoria.

---

# 104. Development environment

Tampoco deberá revelar secretos automáticamente.

---

# 105. Serialization

`SecretValue` no deberá ser serializable por default.

---

# 106. PHP serialization

Debe bloquearse o controlarse:

```php
serialize($secret);
```

---

# 107. JSON serialization

No deberá producir el secreto.

---

# 108. Safe JSON

Como máximo:

```json
{
  "type": "SecretValue",
  "value": "[REDACTED]"
}
```

aunque preferiblemente ni siquiera serializarlo.

---

# 109. Queue jobs

No incluir:

```text
ResolvedCredential
```

dentro del payload de un job.

---

# 110. Job payload

Debe contener:

```text
CredentialReference
```

si realmente es necesario.

El worker resolverá la credencial al ejecutarse.

---

# 111. Why

Un job puede permanecer:

```text
minutes
hours
days
```

en almacenamiento.

Una credencial podría rotar antes de ejecutarse.

---

# 112. Configuration cache

VoltStack no deberá compilar secretos resueltos dentro de un config cache persistente.

---

# 113. Safe config cache

Puede contener:

```text
credential reference
provider ID
purpose
```

---

# 114. Generated files

Comandos de diagnóstico/configuración no deberán generar archivos con secretos salvo operación explícita diseñada para ello.

---

# 115. Source control

Secret material no deberá formar parte de archivos generados destinados a Git.

---

# 116. `.env.example`

Deberá contener:

```text
DB_PASSWORD=
```

no credenciales reales.

---

# 117. Credential resolution context

```php
final readonly class CredentialResolutionContext
{
    public function __construct(
        public CredentialPurpose $purpose,
        public DatabaseDomainId $database,
        public RuntimeContextId $runtime,
        public ?TenantId $tenant,
        public CredentialResolutionPolicy $policy,
    ) {}
}
```

---

# 118. Tenant credentials

Multitenancy podrá requerir:

```text
Tenant A → Credential A
Tenant B → Credential B
```

---

# 119. Credential identity isolation

La cache deberá incluir tenant cuando el secreto sea tenant-specific.

---

# 120. Critical rule

```text
Tenant A credential
≠
Tenant B credential
```

aunque tengan:

```text
same provider
same logical database name
```

---

# 121. No tenant credential fallback

Si la credencial de Tenant A no puede resolverse, no usar automáticamente una credencial global más privilegiada.

---

# 122. Privilege escalation risk

Un fallback global podría convertir:

```text
tenant-specific failure
```

en:

```text
cross-tenant privileged access
```

---

# 123. Tenant context stability

Una conexión resuelta para Tenant A no deberá reutilizarse para Tenant B.

Esto deberá integrarse con Connection State/Reset/Pooling.

---

# 124. Sharding

Cada shard puede poseer credenciales diferentes.

```text
Shard 1 → Credential X
Shard 2 → Credential Y
```

---

# 125. Replica credentials

Writer y replicas también pueden tener credenciales distintas.

---

# 126. Endpoint credential binding

La resolución final podrá depender de:

```text
logical database
endpoint
role
tenant
shard
purpose
```

---

# 127. CredentialSelectionContext

```php
final readonly class CredentialSelectionContext
{
    public function __construct(
        public DatabaseDomainId $database,
        public EndpointId $endpoint,
        public EndpointRole $role,
        public CredentialPurpose $purpose,
        public ?TenantId $tenant,
        public ?ShardId $shard,
    ) {}
}
```

---

# 128. Credential selection ≠ secret resolution

Primero:

```text
Which credential reference?
```

Después:

```text
Resolve its secret.
```

---

# 129. CredentialSelector

```php
interface CredentialSelector
{
    public function select(
        CredentialSelectionContext $context,
    ): CredentialReference;
}
```

---

# 130. Benefits

Esto permite:

```text
writer credentials
replica credentials
tenant credentials
migration credentials
backup credentials
```

sin contaminar Connection Manager con secret-provider logic.

---

# 131. Credential refresh

```text
Connection requested
      ↓
Credential cache
      │
      ├── fresh → use
      │
      └── refresh required
               ↓
         Credential Provider
               ↓
         new credential
```

---

# 132. Refresh coordination

En persistent workers, múltiples requests podrían detectar refresh simultáneamente.

---

# 133. Refresh stampede

Debe evitarse:

```text
100 requests
↓
100 Vault refresh calls
```

---

# 134. Single-flight refresh

Podrá existir coordinación:

```text
first caller refreshes
others wait/use still-valid credential
```

según policy.

---

# 135. Refresh lock scope

Debe ser bounded y no convertirse en global bottleneck.

---

# 136. Refresh failure

Si la credencial actual todavía es válida:

```text
refresh failure
```

podría permitir temporalmente continuar con ella.

---

# 137. Expired credential + refresh failure

No usarla silenciosamente.

Resultado:

```text
CREDENTIAL_UNAVAILABLE
```

---

# 138. Stale-while-valid

Concepto permitido:

```text
refresh failed
+
old credential not expired
=
temporarily usable under policy
```

---

# 139. Stale-after-expiration

Por default:

```text
DENY
```

---

# 140. Clock

Expiration decisions deberán usar un `Clock` inyectable.

---

# 141. Clock ≠ system static calls everywhere

Facilita testing determinista.

---

# 142. Clock skew

Proveedores remotos pueden tener diferencias de reloj.

La policy podrá usar:

```text
expiration safety margin
```

---

# 143. Authentication retry

Un fallo de autenticación puede significar:

```text
rotated credential
expired token
revoked secret
wrong configuration
database unavailable
```

No asumir automáticamente credencial inválida.

---

# 144. Auth failure classification

```php
enum CredentialAuthenticationFailureKind
{
    case INVALID;
    case EXPIRED;
    case REVOKED;
    case UNKNOWN;
}
```

cuando pueda inferirse con evidencia suficiente.

---

# 145. UNKNOWN ≠ invalid credential

Regla consistente con VoltStack.

---

# 146. Credential retry

Puede ser seguro:

```text
authentication fails
↓
invalidate credential cache
↓
resolve latest generation
↓
retry connection establishment
```

bajo policy.

---

# 147. Retry scope

Solo conexión nueva.

Nunca:

```text
retry arbitrary transaction
```

como consecuencia de credential refresh.

---

# 148. Established transaction

Si una credencial es revocada durante una transacción:

```text
Transaction Outcome
```

seguirá siendo responsabilidad del Transaction System.

---

# 149. Rotation event

Podrán existir eventos:

```text
CredentialResolving
CredentialResolved
CredentialRefreshStarted
CredentialRefreshed
CredentialRotationDetected
CredentialRevoked
CredentialResolutionFailed
```

---

# 150. Event payload safety

Los eventos tampoco contendrán secretos.

---

# 151. Credential events ≠ secret transport

Nunca utilizar Event System para transportar `SecretValue`.

---

# 152. Secret provider authentication

El secret provider puede requerir su propia credencial.

Ejemplo:

```text
Application
↓
Vault authentication
↓
DB credential
```

---

# 153. Recursive secret problem

VoltStack Database no deberá intentar resolver universalmente todas las credenciales de infraestructura.

---

# 154. Provider bootstrap credential

La autenticación del provider será responsabilidad de:

```text
provider integration
runtime identity
platform credential system
```

---

# 155. Workload identity

Cuando sea posible, proveedores pueden utilizar:

```text
instance identity
container identity
service account
managed identity
```

en lugar de secretos estáticos adicionales.

---

# 156. Credential security policy

```php
interface CredentialSecurityPolicy
{
    public function evaluate(
        CredentialReference $reference,
        CredentialResolutionContext $context,
    ): CredentialSecurityDecision;
}
```

---

# 157. CredentialSecurityDecision

```php
enum CredentialSecurityDecision
{
    case ALLOW;
    case DENY;
}
```

---

# 158. Policy checks

Podrán incluir:

```text
provider allowed
purpose allowed
environment allowed
tenant allowed
credential lifetime acceptable
authentication mode acceptable
rotation policy satisfied
```

---

# 159. Production secure defaults

Ejemplo:

```text
plaintext config password → warn/policy-dependent
secret in URL → reject/warn
administrative credential for app runtime → reject/warn
expired credential → reject
unknown provider → reject
```

---

# 160. Security mode

Podrían existir perfiles:

```text
DEVELOPMENT
STANDARD
HARDENED
```

pero no deberán debilitar reglas fundamentales de redacción.

---

# 161. HARDENED

Puede exigir:

```text
external secret manager
short-lived credentials
purpose separation
TLS
rotation support
```

según despliegue.

---

# 162. Credential source trust

Cada provider podrá tener clasificación:

```php
enum CredentialProviderTrustClass
{
    case LOCAL;
    case ENCRYPTED_LOCAL;
    case REMOTE_MANAGED;
    case DYNAMIC;
    case CUSTOM;
}
```

---

# 163. Trust class ≠ automatically secure

Es metadata/policy input.

---

# 164. Credential provenance

El sistema deberá saber:

```text
where the credential came from
```

sin revelar su contenido.

---

# 165. Provenance model

```php
final readonly class CredentialProvenance
{
    public function __construct(
        public CredentialProviderId $provider,
        public CredentialReferenceFingerprint $reference,
        public CredentialPurpose $purpose,
    ) {}
}
```

---

# 166. Fingerprinting

Una referencia sensible podrá convertirse a:

```text
H(reference + framework secret/salt)
```

según necesidad.

---

# 167. Secret fingerprint

Podría existir para detectar rotaciones.

Pero almacenar hash de secretos débiles puede ser riesgoso.

No deberá ser estrategia default.

---

# 168. Prefer provider version

Preferir:

```text
provider version
generation
lease ID
```

a hash de password.

---

# 169. Credential comparison

No comparar secretos para decidir identidad si existe metadata de versión confiable.

---

# 170. Credential equality

`ResolvedCredential` no deberá implementar igualdad basada ingenuamente en plaintext.

---

# 171. Credential storage in ORM

Prohibido tratar credenciales de infraestructura como entidades ORM ordinarias por default.

---

# 172. Why

Evita:

```text
accidental hydration
entity cache
result cache
serialization
debug exposure
```

---

# 173. Application-managed credentials

Si una aplicación almacena secretos propios en DB, eso pertenece al Sensitive Data Protection System, no a este sistema de credenciales de infraestructura.

---

# 174. Credential configuration example

```php
'database' => [
    'connections' => [
        'main' => [
            'driver' => 'pgsql',
            'host' => 'db.internal',
            'database' => 'voltstack',

            'credential' => [
                'provider' => 'vault',
                'reference' => 'database/prod/application',
                'purpose' => 'application_read_write',
            ],
        ],
    ],
],
```

---

# 175. Environment example

```php
'credential' => [
    'provider' => 'environment',
    'username' => 'DB_USERNAME',
    'password' => 'DB_PASSWORD',
],
```

Aquí:

```text
DB_PASSWORD
```

es el nombre de la variable, no el valor resuelto.

---

# 176. Dynamic provider example

```php
'credential' => [
    'provider' => 'vault-database',
    'reference' => 'database/creds/voltstack-app',
    'purpose' => 'application_read_write',
],
```

---

# 177. Credential resolution flow

```text
ConnectionManager
      ↓
ConnectionDefinition
      ↓
CredentialSelector
      ↓
CredentialReference
      ↓
CredentialSecurityPolicy
      ↓
CredentialResolver
      ↓
CredentialCache
      │
      ├── fresh
      │      ↓
      │ ResolvedCredential
      │
      └── miss/stale
             ↓
       CredentialProvider
             ↓
       ResolvedCredential
             ↓
       ConnectionFactory
             ↓
            Driver
```

---

# 178. Failure hierarchy

```text
CredentialSecurityException
├── CredentialReferenceException
├── CredentialSelectionException
├── CredentialProviderNotFoundException
├── CredentialResolutionException
├── CredentialUnavailableException
├── CredentialExpiredException
├── CredentialRevokedException
├── CredentialPurposeViolationException
├── CredentialPolicyViolationException
├── CredentialRefreshException
├── CredentialRotationException
├── CredentialSerializationException
└── CredentialProviderException
```

---

# 179. Credential provider failure

Debe distinguirse de:

```text
database connection failure
```

---

# 180. Example

```text
Vault unavailable
```

no es:

```text
PostgreSQL unavailable
```

aunque ambos impidan crear una conexión.

---

# 181. Error classification

```php
enum CredentialFailureKind
{
    case CONFIGURATION;
    case PROVIDER_UNAVAILABLE;
    case AUTHENTICATION;
    case EXPIRED;
    case REVOKED;
    case POLICY;
    case TIMEOUT;
    case UNKNOWN;
}
```

---

# 182. Failure evidence

`UNKNOWN` deberá preservarse.

---

# 183. Credential provider timeout

Secret resolution deberá tener:

```text
deadline
timeout
cancellation
```

cuando el provider sea remoto.

---

# 184. Connection deadline integration

Si crear una conexión tiene deadline total:

```text
credential resolution
+
DNS/network
+
DB authentication
```

deberán respetar el presupuesto global.

---

# 185. Secret resolution retries

Podrán existir para:

```text
transient provider errors
```

con backoff bounded.

---

# 186. No unbounded retries

Obligatorio.

---

# 187. Retry ≠ use expired secret

Una política de retry no autoriza reutilizar una credencial expirada.

---

# 188. Circuit breaker integration

Un provider remoto podrá integrarse posteriormente con resilience.

Pero Credential System no implementará un circuit breaker paralelo propio.

---

# 189. Persistent runtime architecture

Especialmente importante con:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 190. Persistent worker risk

Un secreto resuelto puede permanecer en memoria durante muchas requests.

---

# 191. Required separation

```text
Shared immutable:
provider registry
compiled policy
provider configuration

Scoped/controlled:
resolved credential
tenant binding
secret access context
refresh state
```

---

# 192. Credential cache scope

Si process-local:

```text
bounded
expiration-aware
tenant-aware
generation-aware
```

---

# 193. No request leakage

Una request no deberá poder inspeccionar secretos resueltos por otra.

---

# 194. OpenSwoole

Coroutines concurrentes no deberán compartir mutable credential resolution context.

---

# 195. Single-flight safety

Refresh coordination deberá ser coroutine/thread/process-aware según runtime adapter.

---

# 196. FrankenPHP

Los secrets pueden permanecer process-local si policy lo permite, pero request-specific state deberá resetearse.

---

# 197. RoadRunner

Misma regla.

---

# 198. Worker recycle

Un runtime puede utilizar worker recycling como defensa adicional.

No sustituye secret lifecycle management.

---

# 199. Forking

Si un runtime realiza fork después de cargar secretos, estos pueden heredarse.

La integración deberá evitar pre-resolver secretos antes de fork cuando sea relevante.

---

# 200. CLI

Comandos como:

```text
migrate
schema
backup
restore
```

deberán solicitar el `CredentialPurpose` apropiado.

---

# 201. CLI prompt secrets

Si se permite introducir password manualmente:

```text
terminal echo disabled
```

y nunca guardarlo automáticamente en shell history.

---

# 202. Command arguments

Evitar:

```bash
--password=mysecret
```

porque puede quedar visible en process lists/history.

---

# 203. Preferred CLI secret input

Usar:

```text
secret provider
stdin secure prompt
reference
```

---

# 204. Testing

El sistema deberá probar:

```text
redaction
serialization
provider resolution
purpose separation
expiration
refresh
rotation
revocation
cache isolation
tenant isolation
pool drain
persistent workers
exception safety
telemetry safety
```

---

# 205. Fake credential provider

```php
final class FakeCredentialProvider implements CredentialProvider
{
    // deterministic testing implementation
}
```

---

# 206. Testing secrets

Usar secretos claramente ficticios.

Nunca copiar credenciales reales a fixtures.

---

# 207. Leak detection tests

Se deberá poder ejecutar una suite que busque un canary secret en:

```text
logs
exceptions
telemetry
debug output
serialized payloads
config dumps
```

---

# 208. Canary secret

Ejemplo:

```text
VOLTSTACK_TEST_SECRET_CANARY_7F9...
```

---

# 209. Test invariant

Después de una operación:

```text
canary ∉ observable outputs
```

---

# 210. Rotation test

```text
generation 1
↓
connection A
↓
rotate
↓
generation 2
↓
connection B
```

y verificar:

```text
B uses generation 2
A follows drain policy
```

---

# 211. Expiration test

Con Clock controlado:

```text
13:50 → valid
13:55 → refresh
14:00 → expired
```

---

# 212. Tenant isolation test

```text
Tenant A
→ credential A

Tenant B
→ credential B
```

y comprobar que caches/pools nunca crucen identidades.

---

# 213. Exception leak test

Provocar provider exception que contenga el canary secret y verificar que no llegue a salida observable.

---

# 214. Telemetry leak test

Misma estrategia.

---

# 215. Serialization leak test

Intentar:

```text
serialize
json_encode
queue payload
debug dump
```

---

# 216. Performance considerations

Secret resolution remoto puede añadir latencia.

Por ello:

```text
lazy resolution
bounded local cache
single-flight refresh
connection reuse
```

serán importantes.

---

# 217. Performance ≠ weaker security

No deberán resolverse problemas de rendimiento almacenando secretos indefinidamente.

---

# 218. Credential metrics

Ejemplos:

```text
db.credential.resolve.duration
db.credential.resolve.failures
db.credential.refresh.duration
db.credential.refresh.failures
db.credential.rotation.count
db.credential.cache.hit
db.credential.cache.miss
```

---

# 219. Metrics labels

Bounded:

```text
provider
purpose
result
lifetime_kind
```

No:

```text
raw reference
username
tenant ID unbounded
secret
```

---

# 220. Security audit events

Eventos relevantes:

```text
credential.policy_denied
credential.expired
credential.revoked
credential.rotation
credential.provider_failure
credential.admin_purpose_used
```

según policy.

---

# 221. Audit record

Podrá contener:

```text
timestamp
provider ID
reference fingerprint
purpose
principal/workload identity if appropriate
database domain
decision
reason
generation
```

---

# 222. Audit never contains secret material

Invariante absoluto.

---

# 223. Proposed namespace

```text
VoltStack\Quantum\Database\Security\Credential
```

---

# 224. Directory structure

```text
src/Quantum/Database/Security/Credential/
│
├── Contract/
│   ├── CredentialProvider.php
│   ├── CredentialResolver.php
│   ├── CredentialSelector.php
│   ├── CredentialProviderRegistry.php
│   ├── CredentialSecurityPolicy.php
│   └── SecretRedactor.php
│
├── Model/
│   ├── CredentialReference.php
│   ├── ResolvedCredential.php
│   ├── CredentialIdentity.php
│   ├── CredentialMetadata.php
│   ├── CredentialPurpose.php
│   ├── CredentialLifetimeKind.php
│   ├── CredentialGeneration.php
│   ├── CredentialLease.php
│   └── CredentialProvenance.php
│
├── Secret/
│   ├── SecretValue.php
│   ├── SecretAccessContext.php
│   ├── SecretConsumer.php
│   ├── SecretDisposer.php
│   └── DefaultSecretRedactor.php
│
├── Reference/
│   ├── CredentialProviderId.php
│   ├── CredentialKey.php
│   ├── CredentialReferenceFingerprint.php
│   └── CredentialReferenceParser.php
│
├── Provider/
│   ├── EnvironmentCredentialProvider.php
│   ├── FileCredentialProvider.php
│   ├── EncryptedConfigCredentialProvider.php
│   └── CustomCredentialProviderAdapter.php
│
├── Selection/
│   ├── CredentialSelectionContext.php
│   ├── DefaultCredentialSelector.php
│   └── CredentialSelectionPolicy.php
│
├── Resolution/
│   ├── DefaultCredentialResolver.php
│   ├── CredentialResolutionContext.php
│   ├── CredentialResolutionPolicy.php
│   └── CredentialResolutionResult.php
│
├── Cache/
│   ├── CredentialCache.php
│   ├── CredentialCacheEntry.php
│   ├── CredentialCacheKey.php
│   └── CredentialRefreshCoordinator.php
│
├── Rotation/
│   ├── CredentialRotationCoordinator.php
│   ├── CredentialRotationStrategy.php
│   ├── CredentialRotationEvent.php
│   └── CredentialGenerationTracker.php
│
├── Policy/
│   ├── DefaultCredentialSecurityPolicy.php
│   ├── CredentialSecurityDecision.php
│   ├── CredentialProviderTrustClass.php
│   └── CredentialPurposePolicy.php
│
├── Redaction/
│   ├── CredentialRedactionRegistry.php
│   ├── CredentialExceptionSanitizer.php
│   ├── CredentialTelemetrySanitizer.php
│   └── CredentialDebugSanitizer.php
│
├── Event/
│   ├── CredentialResolving.php
│   ├── CredentialResolved.php
│   ├── CredentialRefreshed.php
│   ├── CredentialRotationDetected.php
│   ├── CredentialRevoked.php
│   └── CredentialResolutionFailed.php
│
├── Telemetry/
│   └── CredentialTelemetry.php
│
├── Testing/
│   ├── FakeCredentialProvider.php
│   ├── CredentialLeakAssertions.php
│   ├── CredentialCanary.php
│   └── CredentialRotationTestKit.php
│
└── Exception/
    ├── CredentialSecurityException.php
    ├── CredentialReferenceException.php
    ├── CredentialSelectionException.php
    ├── CredentialProviderNotFoundException.php
    ├── CredentialResolutionException.php
    ├── CredentialUnavailableException.php
    ├── CredentialExpiredException.php
    ├── CredentialRevokedException.php
    ├── CredentialPurposeViolationException.php
    ├── CredentialPolicyViolationException.php
    ├── CredentialRefreshException.php
    ├── CredentialRotationException.php
    └── CredentialSerializationException.php
```

---

# 225. Architectural invariants

## DB-CRED-001
Database credentials no serán configuración ordinaria.

## DB-CRED-002
Credential será distinto de SecretValue.

## DB-CRED-003
CredentialReference será distinto de ResolvedCredential.

## DB-CRED-004
El framework preferirá referencias sobre secretos resueltos.

## DB-CRED-005
Los secretos se resolverán tan tarde como sea razonable.

## DB-CRED-006
SecretValue no expondrá plaintext mediante `__toString()`.

## DB-CRED-007
Debugging no revelará SecretValue.

## DB-CRED-008
Serialization no revelará SecretValue.

## DB-CRED-009
JSON serialization no revelará SecretValue.

## DB-CRED-010
Logs no contendrán secretos.

## DB-CRED-011
Telemetry no contendrá secretos.

## DB-CRED-012
Audit no contendrá secretos.

## DB-CRED-013
Exception messages no contendrán secretos.

## DB-CRED-014
Debug Toolbar no contendrá secretos.

## DB-CRED-015
Debug mode no autorizará exposición de secretos.

## DB-CRED-016
CredentialPurpose será explícito.

## DB-CRED-017
Runtime credential será separable de migration credential.

## DB-CRED-018
Backup credential será separable de restore credential.

## DB-CRED-019
Administrative credential no será runtime default.

## DB-CRED-020
Credential providers serán extensibles.

## DB-CRED-021
CredentialProvider no creará DB connections.

## DB-CRED-022
ConnectionFactory no será secret provider.

## DB-CRED-023
Provider registry podrá congelarse.

## DB-CRED-024
External input no registrará credential providers.

## DB-CRED-025
Environment provider será soportable pero no tratado como secret manager universal.

## DB-CRED-026
External secret managers serán opcionales.

## DB-CRED-027
Core Database será provider-agnostic.

## DB-CRED-028
Dynamic credentials serán first-class.

## DB-CRED-029
Expiration será explícita cuando sea conocida.

## DB-CRED-030
Refresh time será distinto de expiration time.

## DB-CRED-031
Lease será distinto de connection lifetime.

## DB-CRED-032
Credential generation será distinta del secreto.

## DB-CRED-033
Connections podrán registrar credential generation.

## DB-CRED-034
Rotation no requerirá global disconnect automático.

## DB-CRED-035
Graceful drain será soportable.

## DB-CRED-036
Emergency revocation será política explícita.

## DB-CRED-037
Rotation no interrumpirá silenciosamente caller transactions.

## DB-CRED-038
Connection Pool respetará credential generations.

## DB-CRED-039
Stale credential connections serán detectables.

## DB-CRED-040
Secret cache será distinto del framework cache general.

## DB-CRED-041
Distributed secret cache estará deshabilitado por default.

## DB-CRED-042
In-memory credential cache será bounded.

## DB-CRED-043
Credential cache respetará expiration.

## DB-CRED-044
Credential cache respetará rotation.

## DB-CRED-045
Credential cache respetará tenant identity.

## DB-CRED-046
Credential cache respetará shard/endpoint cuando corresponda.

## DB-CRED-047
VoltStack no prometerá portable absolute memory zeroization.

## DB-CRED-048
Secret disposal será best effort.

## DB-CRED-049
El sistema minimizará secret lifetime.

## DB-CRED-050
El sistema minimizará secret copies.

## DB-CRED-051
Secret reveal requerirá consumer context cuando sea práctico.

## DB-CRED-052
Logger no será secret consumer.

## DB-CRED-053
Profiler no será secret consumer.

## DB-CRED-054
Debug Toolbar no será secret consumer.

## DB-CRED-055
ConnectionFactory será un secret consumer válido.

## DB-CRED-056
DSN diagnostics no incluirán passwords.

## DB-CRED-057
Redaction será type-aware.

## DB-CRED-058
Text redaction será defensa adicional, no única defensa.

## DB-CRED-059
Third-party exceptions serán consideradas potencialmente sensibles.

## DB-CRED-060
Exception chaining no deberá reintroducir secrets observables.

## DB-CRED-061
Credential references también podrán ser sensibles.

## DB-CRED-062
Reference fingerprints podrán usarse para observabilidad.

## DB-CRED-063
Queue payloads no contendrán ResolvedCredential.

## DB-CRED-064
Jobs podrán transportar CredentialReference cuando sea necesario.

## DB-CRED-065
Config cache no contendrá resolved secrets.

## DB-CRED-066
Generated diagnostic files no contendrán secrets por default.

## DB-CRED-067
Tenant credentials estarán aisladas.

## DB-CRED-068
No habrá fallback tenant→global privileged credential por default.

## DB-CRED-069
Tenant connection no será reusable por otro tenant.

## DB-CRED-070
Shard credentials podrán diferir.

## DB-CRED-071
Replica credentials podrán diferir.

## DB-CRED-072
Writer credentials podrán diferir de replica credentials.

## DB-CRED-073
Credential selection será distinta de secret resolution.

## DB-CRED-074
Credential selection podrá depender de endpoint.

## DB-CRED-075
Credential selection podrá depender de purpose.

## DB-CRED-076
Credential selection podrá depender de tenant.

## DB-CRED-077
Credential selection podrá depender de shard.

## DB-CRED-078
Refresh stampedes deberán poder evitarse.

## DB-CRED-079
Refresh coordination será bounded.

## DB-CRED-080
Refresh failure no invalidará automáticamente una credencial todavía válida.

## DB-CRED-081
Expired credential no será usada por default.

## DB-CRED-082
Revoked credential no será usada por default.

## DB-CRED-083
Expiration utilizará Clock abstraído.

## DB-CRED-084
Clock skew podrá mitigarse con safety margin.

## DB-CRED-085
Authentication failure no implicará automáticamente invalid credential.

## DB-CRED-086
UNKNOWN auth failure permanecerá UNKNOWN.

## DB-CRED-087
Credential refresh retry solo afectará establecimiento de conexión.

## DB-CRED-088
Credential refresh no reintentará transacciones arbitrariamente.

## DB-CRED-089
Credential events no transportarán secret material.

## DB-CRED-090
Secret provider bootstrap authentication estará desacoplada del Database core.

## DB-CRED-091
Workload identity será compatible con provider integrations.

## DB-CRED-092
Credential policy podrá restringir providers.

## DB-CRED-093
Credential policy podrá restringir purposes.

## DB-CRED-094
Credential policy podrá restringir lifetime kinds.

## DB-CRED-095
Production policy podrá detectar overprivileged credentials.

## DB-CRED-096
Trust class no equivaldrá automáticamente a security guarantee.

## DB-CRED-097
Credential provenance será observable sin secret.

## DB-CRED-098
Secret fingerprinting no será default para passwords débiles.

## DB-CRED-099
Provider version/generation será preferida sobre password hash.

## DB-CRED-100
Infrastructure credentials no serán ORM entities por default.

## DB-CRED-101
Application-owned secrets pertenecerán al Sensitive Data Protection System.

## DB-CRED-102
Credential resolution tendrá timeout cuando aplique.

## DB-CRED-103
Credential resolution respetará cancellation cuando aplique.

## DB-CRED-104
Credential resolution respetará global connection deadline.

## DB-CRED-105
Provider retries serán bounded.

## DB-CRED-106
Provider retries no autorizarán expired credentials.

## DB-CRED-107
Credential System no duplicará Circuit Breaker architecture.

## DB-CRED-108
Persistent workers no mantendrán request credential context estático.

## DB-CRED-109
Compiled credential policies podrán compartirse.

## DB-CRED-110
Mutable resolved credential state estará controlado.

## DB-CRED-111
No habrá cross-request tenant secret leakage.

## DB-CRED-112
OpenSwoole credential contexts serán coroutine-safe.

## DB-CRED-113
FrankenPHP request state será aislado.

## DB-CRED-114
RoadRunner request/job state será aislado.

## DB-CRED-115
Worker recycling no sustituirá secret lifecycle management.

## DB-CRED-116
Secrets no deberán pre-resolverse innecesariamente antes de process fork.

## DB-CRED-117
CLI migration usará MIGRATION purpose.

## DB-CRED-118
CLI backup usará BACKUP purpose.

## DB-CRED-119
CLI restore usará RESTORE purpose.

## DB-CRED-120
CLI admin usará ADMINISTRATION purpose.

## DB-CRED-121
Passwords no deberán pasarse como CLI argument por default.

## DB-CRED-122
Interactive secret prompts no mostrarán input cuando sea posible.

## DB-CRED-123
Testing no utilizará production secrets.

## DB-CRED-124
Leak detection mediante canary será soportable.

## DB-CRED-125
Canary secrets no aparecerán en logs.

## DB-CRED-126
Canary secrets no aparecerán en telemetry.

## DB-CRED-127
Canary secrets no aparecerán en exceptions.

## DB-CRED-128
Canary secrets no aparecerán en serialized payloads.

## DB-CRED-129
Rotation será testeable determinísticamente.

## DB-CRED-130
Expiration será testeable con fake clock.

## DB-CRED-131
Tenant credential isolation tendrá integration tests.

## DB-CRED-132
Pool rotation tendrá integration tests.

## DB-CRED-133
Secret resolution performance no justificará indefinite caching.

## DB-CRED-134
Credential metrics tendrán bounded cardinality.

## DB-CRED-135
Credential metrics no usarán tenant IDs como labels por default.

## DB-CRED-136
Credential audit será policy-driven.

## DB-CRED-137
Credential audit nunca contendrá secret material.

## DB-CRED-138
Credential availability failure será distinto de DB availability failure.

## DB-CRED-139
Provider failure será distinto de authentication failure.

## DB-CRED-140
Expiration será distinta de revocation.

## DB-CRED-141
Rotation será distinta de expiration.

## DB-CRED-142
Rotation será distinta de revocation.

## DB-CRED-143
Credential cache será distinta de Connection Pool.

## DB-CRED-144
Credential generation será parte de connection reuse eligibility cuando corresponda.

## DB-CRED-145
A connection with old generation podrá ser drained sin destruir in-flight work por default.

## DB-CRED-146
Credential Security no generará SQL.

## DB-CRED-147
Credential Security no ejecutará queries.

## DB-CRED-148
Credential Security no interpretará ORM entities.

## DB-CRED-149
Credential Security no implementará DB authorization rules.

## DB-CRED-150
Credential Security se integrará con Connection System mediante contratos.

## DB-CRED-151
Secrets no serán cacheados en Result Cache.

## DB-CRED-152
Secrets no serán cacheados en Query Cache.

## DB-CRED-153
Secrets no serán cacheados en Metadata Cache.

## DB-CRED-154
Secrets no serán almacenados en IdentityMap.

## DB-CRED-155
Secrets no serán incluidos en compiled query cache.

## DB-CRED-156
Secrets no serán usados como telemetry attributes.

## DB-CRED-157
Secrets no serán usados como cache keys.

## DB-CRED-158
Secrets no serán usados como exception codes.

## DB-CRED-159
Secret values no formarán parte de equality/debug fingerprints.

## DB-CRED-160
Credential references serán validadas antes de provider resolution.

## DB-CRED-161
Credential purposes serán validados antes de secret reveal cuando sea posible.

## DB-CRED-162
Credential resolution será auditable sin exponer secret material.

## DB-CRED-163
Least privilege será principio arquitectónico.

## DB-CRED-164
Short-lived credentials serán soportadas como first-class capability.

## DB-CRED-165
Static credentials seguirán siendo soportables bajo policy.

## DB-CRED-166
Credential rotation será compatible con persistent runtimes.

## DB-CRED-167
Credential rotation será compatible con connection pooling.

## DB-CRED-168
Credential rotation será compatible con read/write endpoints.

## DB-CRED-169
Credential rotation será compatible con sharding.

## DB-CRED-170
Credential rotation será compatible con multitenancy.

## DB-CRED-171
Un secret nunca se convertirá accidentalmente en configuration diagnostic output.

## DB-CRED-172
Un secret nunca será transportado por event payload por default.

## DB-CRED-173
Un secret nunca será transportado por telemetry context.

## DB-CRED-174
Un secret nunca será incluido en debug snapshots.

## DB-CRED-175
Un secret nunca será requerido para identificar una conexión en observabilidad.

## DB-CRED-176
Connection identity utilizará IDs/generations, no passwords.

## DB-CRED-177
Secret provider identity será independiente de DB driver.

## DB-CRED-178
MySQL/MariaDB/PostgreSQL/SQLite adapters consumirán la misma arquitectura general cuando authentication aplique.

## DB-CRED-179
SQLite podrá no requerir credential material, sin romper el modelo.

## DB-CRED-180
NoCredential será una condición explícita, no un password vacío ambiguo.

---

# 226. `NoCredential`

Para plataformas que no requieren autenticación:

```php
final readonly class NoCredential
{
}
```

Esto evita representar:

```text
password=""
```

como si fuera una credencial real.

---

# 227. SQLite

Un archivo SQLite normalmente dependerá más de:

```text
filesystem permissions
file ownership
path security
encryption extensions
```

que de username/password.

Por tanto:

```text
Credential Security
```

seguirá existiendo conceptualmente, pero podrá producir:

```text
NoCredential
```

---

# 228. MySQL/MariaDB/PostgreSQL

Podrán soportar distintos mecanismos:

```text
password
certificate
token
plugin authentication
cloud-generated authentication
```

mediante capabilities/adapters.

---

# 229. Authentication mechanism

```php
enum DatabaseAuthenticationMechanism
{
    case PASSWORD;
    case TOKEN;
    case CERTIFICATE;
    case EXTERNAL_IDENTITY;
    case NONE;
    case CUSTOM;
}
```

---

# 230. Mechanism ≠ Driver

Un driver PostgreSQL no implica siempre password authentication.

---

# 231. Capability-driven authentication

El adapter podrá indicar:

```text
supportsPasswordAuthentication()
supportsTokenAuthentication()
supportsClientCertificateAuthentication()
```

según plataforma/driver.

---

# 232. Certificate credentials

Cuando se utilice TLS client authentication:

```text
certificate
private key
key password
```

deberán seguir las mismas reglas de secret handling.

---

# 233. Private key

Será `SecretValue` o un tipo especializado igualmente protegido.

---

# 234. Certificate public material

El certificado público no necesariamente es secreto.

No debe confundirse con:

```text
private key
```

---

# 235. Secret material classification

```php
enum SecretMaterialKind
{
    case PASSWORD;
    case TOKEN;
    case PRIVATE_KEY;
    case KEY_PASSPHRASE;
    case CUSTOM;
}
```

---

# 236. Anti-patterns

## 236.1 Password en configuración compilada

Incorrecto:

```php
return [
    'password' => 'production-password',
];
```

Preferido:

```text
CredentialReference
```

---

## 236.2 Password en DSN observable

Incorrecto:

```text
pgsql://admin:secret@db/prod
```

---

## 236.3 Resolver todos los secretos al boot

Incorrecto por default.

---

## 236.4 Guardar secretos en static properties

Incorrecto:

```php
private static string $password;
```

---

## 236.5 Cachear secrets en Redis

Incorrecto por default.

---

## 236.6 Loggear configuración completa

Incorrecto:

```php
logger()->debug($databaseConfig);
```

si contiene material secreto.

---

## 236.7 Mostrar secretos en debug toolbar

Prohibido.

---

## 236.8 Serializar ResolvedCredential

Prohibido por default.

---

## 236.9 Reutilizar admin credentials para runtime

Contrario a least privilege.

---

## 236.10 Tenant fallback privilegiado

Incorrecto:

```text
tenant secret missing
↓
use root DB credential
```

---

## 236.11 Rotation mediante kill global

No será estrategia default.

---

## 236.12 Credential refresh dentro de una transacción arbitraria

No deberá cambiar silenciosamente la identidad de una conexión ya establecida.

---

## 236.13 Password como cache key

Prohibido.

---

## 236.14 Secret como metric label

Prohibido.

---

## 236.15 Secret como exception metadata

Prohibido.

---

# 237. Modelo formal

Sea:

```text
R = CredentialReference
P = CredentialPurpose
C = CredentialResolutionContext
S = CredentialSecurityPolicy
```

Entonces:

```text
Resolve(R, P, C)
```

solo será permitido si:

```text
S(R, P, C) = ALLOW
```

---

# 238. Resolution

El resultado:

```text
K = ResolvedCredential
```

deberá satisfacer:

```text
Valid(K)
∧
PurposeCompatible(K, P)
∧
¬Expired(K)
∧
¬KnownRevoked(K)
```

---

# 239. Connection eligibility

Para una conexión reutilizable `Conn`:

```text
Reusable(Conn)
⇒
Healthy(Conn)
∧
ContextCompatible(Conn)
∧
CredentialGenerationCompatible(Conn)
∧
¬CredentialRevoked(Conn)
```

---

# 240. Rotation

Si:

```text
Generation(K_new) > Generation(K_old)
```

entonces las nuevas conexiones deberán utilizar:

```text
K_new
```

una vez que la nueva generación sea authoritative.

---

# 241. Old connection

Una conexión existente podrá continuar únicamente si:

```text
RotationPolicyAllows(Conn)
```

---

# 242. Secret observability invariant

Para todo output observable `O`:

```text
SecretMaterial(K) ∉ O
```

donde `O` incluye:

```text
logs
telemetry
debug
exceptions
audit
serialization
cache diagnostics
CLI diagnostics
```

---

# 243. Arquitectura final

```text
                  Database Operation
                         │
                         ▼
                 Connection Manager
                         │
                         ▼
               Connection Definition
                         │
                         ▼
                Credential Selector
                         │
                         ▼
               Credential Reference
                         │
                         ▼
               Credential Security
                       Policy
                         │
                         ▼
                Credential Resolver
                         │
                ┌────────┴────────┐
                ▼                 ▼
        Credential Cache    Provider Registry
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼             ▼
               Environment      Vault        Cloud
                    │             │             │
                    └─────────────┼─────────────┘
                                  ▼
                         ResolvedCredential
                                  │
                           Secret Boundary
                                  │
                                  ▼
                         Connection Factory
                                  │
                                  ▼
                               Driver
                                  │
                                  ▼
                              Database

Cross-cutting:

Credential Rotation
Credential Redaction
Credential Telemetry
Credential Audit
Credential Cache
Persistent Runtime Isolation
Tenant Isolation
Shard/Endpoint Binding
```

---

# 244. Regla maestra final

> **VoltStack nunca deberá tratar una contraseña, token o clave privada como un simple string de configuración que puede circular libremente por el framework. La aplicación deberá operar principalmente con referencias y metadata; el material secreto se resolverá de forma controlada, se revelará únicamente en la frontera de autenticación, tendrá un ciclo de vida acotado y quedará excluido de logs, telemetry, eventos, caches generales, excepciones, serialización y herramientas de debugging.**

La arquitectura puede resumirse como:

```text
Reference broadly.
Resolve late.
Reveal narrowly.
Use minimally.
Rotate safely.
Redact always.
Dispose early.
```

---

# 245. Relación con la arquitectura de seguridad

```text
226_DATABASE_SECURITY_ARCHITECTURE
              │
              ├── 227 SQL Injection Prevention
              │
              ├── 228 Query Input Security
              │
              └── 229 Credential Security
                         │
                         ▼
                  Connection Security
```

Esto establece una separación importante:

```text
Query Security
      │
      └── protege qué se consulta y cómo se representa

Credential Security
      │
      └── protege con qué identidad nos autenticamos

Connection Security
      │
      └── protegerá el canal y sesión establecida
```

---

# 246. Estado del Bloque 22

```text
BLOCK 22 — SECURITY

✓ 226_DATABASE_SECURITY_ARCHITECTURE.md
✓ 227_DATABASE_SQL_INJECTION_PREVENTION_SYSTEM.md
✓ 228_DATABASE_QUERY_INPUT_SECURITY_SYSTEM.md
✓ 229_DATABASE_CREDENTIAL_SECURITY_SYSTEM.md
○ 230_DATABASE_CONNECTION_SECURITY_SYSTEM.md
○ 231_DATABASE_DATA_ACCESS_SECURITY_SYSTEM.md
○ 232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM.md
○ 233_DATABASE_QUERY_AUDIT_SYSTEM.md
○ 234_DATABASE_DATABASE_PERMISSION_MODEL.md
```

---

# 247. Siguiente documento

```text
230_DATABASE_CONNECTION_SECURITY_SYSTEM.md
```

El siguiente documento deberá definir la seguridad de la **sesión y canal de conexión** una vez seleccionada/resuelta la credencial, incluyendo:

```text
TLS
certificate verification
hostname verification
CA trust
client certificates
TLS policy
minimum TLS versions
cipher policy
encrypted transport requirements
local socket security
connection endpoint validation
DNS considerations
connection hijacking
MITM protection
session initialization
session security state
secure session defaults
connection reset
connection reuse
pool security
tenant isolation
credential generation binding
read/write endpoints
replicas
shards
connection state contamination
session variables
SQL modes
search_path
timezone
roles
temporary objects
prepared statements
persistent workers
FrankenPHP
RoadRunner
OpenSwoole
connection telemetry
security diagnostics
failure classification
secure defaults
testing
```

bajo la regla:

> **Una conexión autenticada no deberá considerarse automáticamente una conexión segura: VoltStack deberá verificar el endpoint, proteger el transporte, establecer un estado de sesión conocido y garantizar que ninguna conexión reutilizada pueda transportar identidad o estado sensible entre contextos incompatibles.**