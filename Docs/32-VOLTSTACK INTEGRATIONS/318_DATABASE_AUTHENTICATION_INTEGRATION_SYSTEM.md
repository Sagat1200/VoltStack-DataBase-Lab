# 318_DATABASE_AUTHENTICATION_INTEGRATION_SYSTEM.md

# VoltStack Quantum Database
## Database Authentication Integration System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 318 — Database Authentication Integration System  
**Bloque:** 32 — VoltStack Integration  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `317_DATABASE_VALIDATION_INTEGRATION_SYSTEM.md`  
**Siguiente documento:** `319_DATABASE_AUTHORIZATION_INTEGRATION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura de integración entre:

```text
VoltStack/Quantum/Database
```

y:

```text
VoltStack Authentication
```

El objetivo es permitir que el sistema de autenticación utilice capacidades de persistencia de VoltStack Database sin acoplar:

- guards;
- firewalls;
- authenticators;
- identity providers;
- sesiones;
- tokens;
- MFA;
- WebAuthn;
- OAuth2/OIDC;
- device trust;
- risk engines;

directamente a:

- SQL;
- PDO;
- drivers;
- conexiones físicas;
- dialectos;
- detalles particulares de MySQL, MariaDB, PostgreSQL o SQLite.

La regla fundamental será:

> **Database persiste y recupera datos relacionados con autenticación; Authentication determina cómo esos datos participan en la demostración y mantenimiento de una identidad autenticada.**

Por tanto:

```text
Database
≠
Authentication
```

```text
User Record
≠
Authenticated Identity
```

```text
Credential Exists
≠
Credential Valid
```

```text
Successful Database Lookup
≠
Successful Authentication
```

---

# 2. Contexto arquitectónico

VoltStack Authentication ya constituye un subsistema independiente con responsabilidades como:

```text
Authentication
├── Identity
├── Identity Providers
├── Guards
├── Firewalls
├── Authenticators
├── Credentials
├── Password Authentication
├── Sessions
├── Tokens
├── MFA
├── WebAuthn
├── OAuth2 / OIDC
├── Device Trust
├── Risk Engine
├── Throttling
└── Authentication Policy
```

Database proporciona:

```text
Database
├── Query Engine
├── ORM
├── Persistence
├── Transactions
├── Connections
├── Drivers
├── Cache Integration
├── Telemetry
├── Security
├── Multitenancy Integration
├── Sharding
└── Persistent Runtime Support
```

La integración debe mantener ambos dominios separados.

---

# 3. Problema arquitectónico

Una implementación ingenua podría producir:

```text
LoginController
      ↓
Authenticator
      ↓
PDO
      ↓
SELECT * FROM users...
```

o:

```text
Guard
 ↓
User::where(...)
 ↓
Database
```

con conocimiento directo del modelo de persistencia.

Esto genera:

- acoplamiento;
- consultas duplicadas;
- imposibilidad de cambiar estrategia de almacenamiento;
- dificultad para probar Authentication;
- filtración de conceptos ORM;
- riesgo de consultas inseguras;
- dependencia de tablas concretas;
- dificultad para utilizar identidades externas;
- dificultad para soportar múltiples providers.

VoltStack utilizará una frontera explícita.

---

# 4. Arquitectura general

```text
HTTP / CLI / Job / API
          │
          ▼
   Authentication
          │
          ▼
 Authentication Provider
          │
          ▼
Database Authentication Bridge
          │
          ▼
Authentication Persistence Adapter
          │
          ▼
      Database API
          │
   ┌──────┴──────┐
   ▼             ▼
 Query Engine    ORM
   │             │
   └──────┬──────┘
          ▼
  Execution Engine
          │
          ▼
 Connection System
          │
          ▼
        Driver
          │
          ▼
         DBMS
```

---

# 5. Dirección de dependencias

La dependencia permitida será:

```text
Authentication
      ↓
Authentication Persistence Contract
      ↑
Database Integration Adapter
      ↓
Database
```

Esto evita que:

```text
Authentication Core
```

dependa directamente de:

```text
VoltStack/Quantum/Database
```

cuando no sea necesario.

---

# 6. Dependency inversion

Authentication podrá definir contratos como:

```php
interface IdentityProvider
{
    public function findByIdentifier(
        AuthenticationIdentifier $identifier,
        AuthenticationContext $context,
    ): ?AuthenticatableIdentity;
}
```

Database proporcionará:

```php
final class DatabaseIdentityProvider implements IdentityProvider
{
    // Database-backed implementation.
}
```

Por tanto:

```text
Authentication owns the contract.
Database integration implements it.
```

---

# 7. Regla de separación

No deberá existir:

```text
Authenticator
→ PDO
```

ni:

```text
Guard
→ Driver
```

ni:

```text
Firewall
→ SQL Compiler
```

ni:

```text
PasswordVerifier
→ Query Builder
```

---

# 8. Responsabilidades de Authentication

Authentication será responsable de:

- resolver mecanismo de autenticación;
- identificar credenciales presentadas;
- seleccionar identity provider;
- validar credenciales;
- verificar passwords;
- evaluar MFA;
- verificar WebAuthn;
- procesar tokens;
- evaluar account state;
- aplicar políticas de autenticación;
- crear authenticated principal;
- administrar authentication state;
- emitir eventos de autenticación.

---

# 9. Responsabilidades de Database

Database será responsable de:

- recuperar datos persistidos;
- persistir cambios;
- ejecutar consultas;
- controlar conexiones;
- transacciones;
- consistency routing;
- constraints;
- concurrency control;
- errores Database;
- persistencia de sesiones/tokens cuando se configure;
- observabilidad Database;
- aislamiento tenant/shard.

---

# 10. Responsabilidades de la integración

La capa de integración deberá:

- implementar providers Database-backed;
- traducir identidad Authentication ↔ datos persistentes;
- construir queries seguras;
- seleccionar consistencia;
- aplicar contexto tenant;
- aplicar contexto shard;
- mapear errores;
- proteger datos sensibles;
- controlar persistence boundaries;
- aportar telemetry contextual;
- preservar lifecycle y runtime isolation.

---

# 11. Modelo conceptual

Debe distinguirse:

```text
Database Row
```

de:

```text
ORM Entity
```

de:

```text
Authentication Identity
```

de:

```text
Authenticated Principal
```

---

# 12. Ejemplo

Una tabla:

```text
users
```

puede contener:

```text
id
email
password_hash
status
created_at
```

Pero esto no significa que:

```text
users row
```

sea automáticamente:

```text
AuthenticatedIdentity
```

---

# 13. Identity projection

Se recomienda que Authentication reciba una proyección explícita:

```php
final readonly class DatabaseAuthenticatableIdentity
{
    public function __construct(
        public IdentityId $id,
        public AuthenticationIdentifier $identifier,
        public CredentialReference $credential,
        public AccountState $state,
        public array $attributes,
    ) {}
}
```

---

# 14. Entity-backed identity

También podrá utilizarse una entidad:

```php
User::class
```

si la aplicación lo desea.

Pero:

```text
Entity
≠
Authentication Principal
```

---

# 15. Authentication identity mapping

Se propone:

```text
Database Record / Entity
          ↓
Identity Mapper
          ↓
AuthenticatableIdentity
          ↓
Authentication
          ↓
AuthenticatedPrincipal
```

---

# 16. IdentityProvider

Contrato conceptual:

```php
interface IdentityProvider
{
    public function findByIdentifier(
        AuthenticationIdentifier $identifier,
        AuthenticationContext $context,
    ): ?AuthenticatableIdentity;

    public function refresh(
        IdentityReference $identity,
        AuthenticationContext $context,
    ): ?AuthenticatableIdentity;
}
```

---

# 17. DatabaseIdentityProvider

Implementación:

```php
final class DatabaseIdentityProvider implements IdentityProvider
{
    public function __construct(
        private DatabaseIdentityRepository $repository,
        private AuthenticationDatabaseContextResolver $contextResolver,
    ) {}
}
```

---

# 18. Repository boundary

Se podrá utilizar:

```text
DatabaseIdentityRepository
```

como frontera especializada.

Ejemplo:

```php
interface DatabaseIdentityRepository
{
    public function findByAuthenticationIdentifier(
        AuthenticationIdentifier $identifier,
        AuthenticationDatabaseContext $context,
    ): ?DatabaseIdentityRecord;
}
```

---

# 19. Repository ≠ Authentication

El repository sólo recupera información.

No deberá decidir:

```text
password valid
MFA passed
risk acceptable
account authenticated
```

---

# 20. Authentication lookup

Flujo:

```text
Login Request
     ↓
Identifier Normalization
     ↓
IdentityProvider
     ↓
DatabaseIdentityRepository
     ↓
Query Engine
     ↓
Database
     ↓
Identity Record?
```

---

# 21. Credential verification

Posteriormente:

```text
Identity Record
     ↓
Credential Reference
     ↓
Password Hasher / Credential Verifier
     ↓
Credential Result
```

Database no verifica passwords.

---

# 22. Regla fundamental de passwords

```text
Database
stores password hash
```

pero:

```text
Database
does not authenticate password
```

---

# 23. Password storage

Database deberá persistir únicamente la representación aprobada por Authentication:

```text
password_hash
```

Nunca:

```text
plaintext password
```

---

# 24. Plaintext lifecycle

El password presentado deberá existir únicamente durante el mínimo scope necesario dentro de Authentication.

No deberá pasar a:

- Query logs;
- Database telemetry;
- ORM snapshots;
- audit payloads;
- cache keys;
- exceptions;
- debug toolbar.

---

# 25. Credential persistence boundary

Conceptualmente:

```text
Plain Credential
      ↓
Authentication Credential Processor
      ↓
Password Hasher
      ↓
Password Hash
      ↓
Database Persistence
```

Nunca:

```text
Plain Credential
      ↓
Database
```

---

# 26. Password rehash

Cuando Authentication detecte:

```text
needsRehash()
```

podrá solicitar persistencia del nuevo hash.

---

# 27. Rehash flow

```text
Successful Password Verification
          ↓
needsRehash?
          ↓ yes
Generate New Hash
          ↓
Credential Persistence Adapter
          ↓
Database
```

---

# 28. Rehash ≠ login success requirement

Si el rehash falla después de una autenticación correctamente verificada, la política deberá decidir el comportamiento.

No deberá asumirse universalmente:

```text
rehash failure
=
credential invalid
```

---

# 29. AuthenticationDatabaseContext

Se propone:

```text
AuthenticationDatabaseContext
├── OperationId
├── AuthenticationAttemptId
├── LogicalDatabase
├── ConnectionIntent
├── ConsistencyRequirement
├── TenantContext?
├── ShardContext?
├── TransactionContext?
├── IdentityProviderId
├── FirewallId?
├── GuardId?
└── SecurityPolicy
```

---

# 30. Context lifecycle

Será scoped a:

```text
authentication operation
```

y nunca:

```text
global mutable state
```

---

# 31. AuthenticationAttemptId

Cada intento podrá tener un identificador de correlación:

```text
AuthenticationAttemptId
```

para:

- telemetry;
- audit;
- tracing;
- debugging;
- rate-limit correlation.

No deberá contener credenciales.

---

# 32. Read consistency

Las consultas de autenticación pueden ser sensibles a consistencia.

Ejemplo:

```text
User changes password
        ↓
Writer updated
        ↓
Replica delayed
```

Una autenticación inmediata contra la réplica podría verificar el hash antiguo.

---

# 33. Authentication-sensitive reads

Por defecto, datos como:

```text
password hash
account disabled state
MFA state
credential revocation
security version
token revocation
```

deberán tratarse como:

```text
security-sensitive reads
```

---

# 34. Writer preference

Las operaciones críticas podrán requerir:

```text
writer
```

o una fuente que demuestre consistencia suficiente.

---

# 35. Replica safety

Regla:

> **Una réplica no deberá utilizarse para una decisión de autenticación sensible si su frescura respecto al estado de seguridad requerido no puede demostrarse.**

---

# 36. UNKNOWN lag

```text
ReplicaLag = UNKNOWN
```

no deberá interpretarse como:

```text
ReplicaFresh = TRUE
```

---

# 37. Session consistency

Tras un cambio de seguridad realizado por el usuario:

```text
password change
MFA activation
session revocation
account disable
```

la sesión podrá activar:

```text
sticky writer
```

o política equivalente.

---

# 38. Read intent

Se podrán definir intenciones:

```text
AUTH_IDENTITY_LOOKUP
AUTH_CREDENTIAL_READ
AUTH_ACCOUNT_STATE_READ
AUTH_SESSION_READ
AUTH_TOKEN_READ
AUTH_SECURITY_STATE_READ
```

---

# 39. Connection routing

Database utilizará estas intenciones junto con:

```text
ConsistencyRequirement
```

para seleccionar endpoint.

---

# 40. Authentication ≠ routing engine

Authentication declara:

```text
required semantics
```

Database decide:

```text
eligible connection
```

---

# 41. User lookup optimization

Una consulta de login no deberá cargar necesariamente toda la entidad User.

Podrá utilizar:

```text
Authentication Projection
```

---

# 42. Example projection

```text
id
normalized_identifier
password_hash
account_status
security_version
mfa_required
tenant_reference
```

---

# 43. SELECT *

Deberá evitarse:

```sql
SELECT *
FROM users
WHERE email = ?
```

cuando sólo se necesitan datos específicos.

---

# 44. Sensitive columns

Columnas no necesarias como:

```text
personal data
billing data
profile data
internal notes
```

no deberán recuperarse durante autenticación.

---

# 45. Minimal privilege

El runtime identity lookup deberá operar con:

```text
least database privilege
```

compatible con la operación.

---

# 46. Identifier normalization

Authentication será responsable de normalizar:

```text
email
username
phone
external subject
```

según su dominio.

Database realizará la comparación conforme al query recibido.

---

# 47. Database collation

La normalización Authentication deberá ser compatible con la semántica de persistencia.

Pero:

```text
PHP normalization
≠
DB collation semantics
```

automáticamente.

---

# 48. Stable normalized identifier

Para casos sensibles podrá persistirse:

```text
normalized_identifier
```

además del valor de presentación.

---

# 49. Example

```text
email
normalized_email
```

con constraint sobre:

```text
normalized_email
```

si ésa es la semántica de identidad deseada.

---

# 50. Authentication identifier uniqueness

La unicidad de un identificador persistido deberá garantizarse mediante:

```text
Database UNIQUE constraint
```

cuando corresponda.

---

# 51. Validation integration

El documento 317 puede realizar:

```text
Rule::unique(...)
```

para UX.

Pero Authentication dependerá de la garantía final del constraint.

---

# 52. Ambiguous identity

Una consulta de autenticación que encuentre múltiples registros cuando se esperaba identidad única deberá producir:

```text
IdentityIntegrityViolation
```

no seleccionar arbitrariamente el primero.

---

# 53. Enumeration resistance

El sistema deberá evitar revelar innecesariamente:

```text
user exists
user does not exist
```

a clientes externos.

---

# 54. Internal distinction

Internamente sí deberán distinguirse:

```text
IDENTITY_NOT_FOUND
INVALID_CREDENTIAL
ACCOUNT_DISABLED
MFA_REQUIRED
```

para control correcto.

Pero la respuesta externa podrá ser deliberadamente genérica.

---

# 55. Timing considerations

El flujo deberá reducir diferencias observables entre:

```text
unknown identity
```

y:

```text
known identity + invalid password
```

cuando Authentication defina protección contra enumeration/timing analysis.

---

# 56. Dummy credential verification

Authentication podrá ejecutar una verificación dummy cuando no existe identidad.

Database no será responsable de ello.

---

# 57. Database telemetry

No deberá producir métricas externas que permitan inferir fácilmente:

```text
identity exists
```

mediante dimensiones de alta cardinalidad o datos sensibles.

---

# 58. Account state

Database puede almacenar:

```text
ACTIVE
LOCKED
DISABLED
SUSPENDED
PENDING
```

pero Authentication interpreta su significado.

---

# 59. Database constraint

Database puede garantizar:

```text
valid stored enum representation
```

pero no decide:

```text
whether PENDING may authenticate
```

---

# 60. Account lockout

Authentication podrá persistir:

```text
failed_attempt_count
locked_until
```

pero la política de lockout pertenece a Authentication.

---

# 61. Atomic lockout updates

Cuando sea necesario evitar races:

```text
increment failed attempts
```

deberá realizarse mediante operaciones atómicas o transaccionales adecuadas.

---

# 62. Read-modify-write danger

Incorrecto:

```text
read failed_attempts = 4
+1 in PHP
write 5
```

bajo concurrencia sin control.

---

# 63. Atomic mutation

Database podrá proporcionar una operación semántica equivalente a:

```text
increment failed attempts
```

sin que Authentication genere SQL.

---

# 64. Authentication state version

Se recomienda considerar:

```text
security_version
```

o:

```text
authentication_version
```

para invalidar estados derivados.

---

# 65. Example

Cambiar:

```text
password
MFA configuration
critical security settings
```

podría incrementar:

```text
security_version
```

---

# 66. Session comparison

Una sesión puede guardar:

```text
security_version = 8
```

mientras Database contiene:

```text
security_version = 9
```

permitiendo detectar invalidación.

---

# 67. Session persistence

VoltStack Authentication podrá utilizar:

```text
DatabaseSessionStore
```

como una implementación opcional.

---

# 68. Session Store contract

Conceptualmente:

```php
interface SessionStore
{
    public function read(SessionId $id): ?SessionRecord;

    public function write(SessionRecord $session): void;

    public function delete(SessionId $id): void;
}
```

---

# 69. DatabaseSessionStore

```text
Authentication Session
        ↓
DatabaseSessionStore
        ↓
Database
```

---

# 70. Session ≠ ORM Entity

Una sesión podrá persistirse mediante Query Engine directamente sin requerir ORM.

---

# 71. Session data security

Los datos de sesión deberán:

- minimizarse;
- protegerse;
- tener expiración;
- evitar plaintext sensitive credentials;
- evitar serialización insegura;
- poder invalidarse.

---

# 72. Session ID

Nunca deberá registrarse completo en:

```text
logs
telemetry
debug toolbar
exceptions
```

---

# 73. Session token storage

Cuando sea posible, deberá persistirse:

```text
token digest / hash
```

en lugar del bearer secret original.

---

# 74. Remember-me tokens

Igualmente:

```text
remember token plaintext
```

no deberá almacenarse cuando pueda utilizarse una representación verificable segura.

---

# 75. Token architecture

Debe distinguirse:

```text
Token Secret
Token Identifier
Token Hash
Token Metadata
Token State
```

---

# 76. Token lookup

Un patrón posible:

```text
Presented Token
      ↓
Extract public selector
      ↓
Database lookup
      ↓
Stored token hash
      ↓
Constant-time verification
```

---

# 77. Database does not validate bearer secret

Database recupera el material persistido.

Authentication realiza la verificación criptográfica.

---

# 78. Token revocation

La revocación podrá persistirse como:

```text
revoked_at
status
version
```

según el modelo.

---

# 79. Revocation-sensitive reads

Deberán tener consistencia apropiada.

Una réplica stale podría aceptar temporalmente un token revocado.

---

# 80. Access tokens

Si son:

```text
self-contained
```

Database puede no intervenir en cada request.

Si son:

```text
opaque
```

puede requerirse lookup persistente.

---

# 81. Refresh tokens

Deberán almacenarse siguiendo una estrategia de protección equivalente a credenciales de alto valor.

---

# 82. Refresh token rotation

La rotación puede requerir:

```text
transaction
```

para garantizar:

```text
consume old
+
create new
```

de forma coherente.

---

# 83. Token replay detection

Puede requerir estado persistido:

```text
family id
rotation counter
consumed_at
replay state
```

pero Authentication define la política.

---

# 84. MFA persistence

Database podrá persistir:

```text
MFA enrollment metadata
recovery code digests
TOTP encrypted secret
factor state
factor timestamps
```

---

# 85. TOTP secret

No deberá almacenarse en plaintext sin protección.

---

# 86. Encryption integration

El secreto deberá pasar por:

```text
Authentication / Security
        ↓
Encryption Service
        ↓
Encrypted Payload
        ↓
Database
```

---

# 87. Database does not own encryption key

Las llaves criptográficas no deberán vivir en:

```text
Database config
database rows
query metadata
```

salvo arquitecturas explícitas de key management.

---

# 88. Recovery codes

Preferiblemente se almacenarán como:

```text
digests
```

cuando sólo necesiten verificación.

---

# 89. Recovery code consumption

Debe ser atómico:

```text
valid
+
unused
→
consume
```

---

# 90. Concurrent recovery attempts

Sólo uno deberá poder consumir el mismo código cuando la política así lo requiera.

---

# 91. WebAuthn persistence

Podrá almacenar:

```text
credential_id
public_key
sign_count
transports
aaguid
created_at
last_used_at
status
```

---

# 92. Private key

Database nunca deberá recibir una private key del autenticador.

---

# 93. WebAuthn counter

La actualización de:

```text
sign_count
```

puede requerir control de concurrencia.

---

# 94. WebAuthn credential ID

Debe tener:

```text
binary-safe representation
```

y mapping correcto del Type System.

---

# 95. OAuth2/OIDC identities

Database podrá persistir:

```text
provider
subject
local_identity_id
claims snapshot?
token references?
consent metadata?
```

según arquitectura.

---

# 96. Stable external identity

La clave recomendada conceptualmente será:

```text
issuer/provider + subject
```

no simplemente:

```text
email
```

---

# 97. External email ≠ stable subject

Authentication no deberá tratar automáticamente un email OIDC como identificador externo permanente.

---

# 98. OAuth tokens

Si se almacenan:

```text
access token
refresh token
client secret-derived material
```

deberán aplicar protección de datos sensibles.

---

# 99. External token encryption

Los tokens recuperables podrán necesitar:

```text
encryption at application layer
```

antes de Database.

---

# 100. Device trust

Database podrá persistir:

```text
device_id
identity_id
trust_state
credential reference
last_seen
risk metadata
revoked_at
```

---

# 101. Device fingerprint

No deberá asumirse que un fingerprint constituye identidad criptográfica por sí mismo.

---

# 102. Risk Engine

Database podrá suministrar evidencia persistida:

```text
last login
known devices
failed attempts
security events
```

pero:

```text
Database
≠
Risk Engine
```

---

# 103. Risk state freshness

Las decisiones de riesgo pueden requerir datos recientes.

La integración deberá declarar la consistencia requerida.

---

# 104. Authentication transactions

Algunas operaciones deberán ser transaccionales.

Ejemplo:

```text
rotate refresh token
```

podría requerir:

```text
BEGIN
invalidate old token
insert new token
record rotation state
COMMIT
```

---

# 105. Authentication owns business operation

Database TransactionManager proporciona la transacción.

Authentication coordina la operación.

---

# 106. Transaction ≠ Authentication Attempt

Un intento de login no necesita necesariamente una transacción global.

---

# 107. Avoid oversized transactions

No mantener una transacción abierta mientras se realizan:

```text
external IdP request
WebAuthn user interaction
email OTP delivery
slow network calls
```

salvo diseño específico.

---

# 108. Transaction boundary

Debe limitarse a la parte persistente que realmente necesita atomicidad.

---

# 109. Unknown commit

Si se pierde conexión después de enviar COMMIT:

```text
CommitOutcome = UNKNOWN
```

Authentication no deberá asumir:

```text
rolled back
```

---

# 110. Sensitive operation example

Durante refresh-token rotation, UNKNOWN puede requerir:

```text
reconciliation
deny/retry with token family state
```

en lugar de repetir ciegamente.

---

# 111. Authentication retry

Database retry policy no deberá repetir automáticamente operaciones Authentication no idempotentes.

---

# 112. Example

Esto puede ser inseguro:

```text
consume one-time recovery code
connection lost
retry blindly
```

---

# 113. Replayability

La integración deberá aportar metadata:

```text
IDEMPOTENT
REPLAYABLE
NON_REPLAYABLE
UNKNOWN
```

cuando corresponda.

---

# 114. Deadlocks

Un deadlock puede permitir retry de una transacción completa sólo cuando Authentication declare que la operación es replayable.

---

# 115. ORM integration

Una aplicación podrá tener:

```php
final class User
{
    // ...
}
```

como entidad ORM y authenticatable identity source.

---

# 116. Authentication mapping

No se recomienda hacer que todas las entidades User implementen obligatoriamente numerosos contratos internos de Authentication.

Puede utilizarse un adapter.

---

# 117. Entity adapter

```php
final class UserAuthenticationAdapter
{
    public function toIdentity(User $user): AuthenticatableIdentity
    {
        // ...
    }
}
```

---

# 118. Model API

También podrá existir una experiencia Laravel-like:

```php
class User extends Model implements Authenticatable
{
}
```

como API de conveniencia.

---

# 119. Model convenience ≠ architecture

Internamente seguirá convergiendo en:

```text
IdentityProvider
+
Database Integration
```

---

# 120. IdentityMap concern

Authentication no deberá depender de que:

```text
IdentityMap
```

contenga una versión actual de datos críticos.

---

# 121. Security-sensitive refresh

Para verificar:

```text
account disabled
security version
credential changed
```

puede requerirse una lectura fresca.

---

# 122. Managed entity stale state

Una entidad ya cargada:

```text
User
```

podría contener estado anterior.

Por tanto:

```text
Managed Entity State
≠
Fresh Authentication Security State
```

---

# 123. Security projection

Se recomienda una ruta especializada para obtener:

```text
AuthenticationSecurityState
```

cuando se requiera frescura explícita.

---

# 124. Cache integration

Authentication puede utilizar cache.

Pero deberá distinguirse:

```text
Identity Cache
Credential Metadata Cache
Session Cache
Token Cache
Authorization Cache
Database Entity Cache
```

---

# 125. Cache ≠ authority

Una decisión crítica no deberá confiar en cache stale si:

```text
revocation
disable
credential change
```

debe tener efecto inmediato.

---

# 126. Cache consistency

Authentication deberá declarar políticas como:

```text
CACHE_ALLOWED
CACHE_WITH_VERSION_CHECK
FRESH_DATABASE_REQUIRED
```

---

# 127. Password hash cache

No se recomienda distribuir hashes de password innecesariamente por caches compartidos.

---

# 128. Sensitive cache data

Cualquier cache que contenga material Authentication deberá aplicar:

- scope;
- encryption cuando corresponda;
- TTL;
- invalidation;
- redaction;
- access controls.

---

# 129. Cache invalidation

Eventos como:

```text
PasswordChanged
MfaChanged
AccountDisabled
TokenRevoked
SecurityVersionChanged
```

podrán invalidar estados derivados.

---

# 130. afterCommit

La invalidación/publicación correspondiente a una escritura deberá coordinarse con:

```text
afterCommit
```

cuando dependa de que la escritura sea definitiva.

---

# 131. Rollback

Si la transacción hace rollback:

```text
security state changed
```

no deberá publicarse como hecho persistido.

---

# 132. UNKNOWN commit

Deberá aplicarse política conservadora.

Por ejemplo:

```text
invalidate cache
```

puede ser preferible a conservar estado potencialmente obsoleto.

---

# 133. Multitenancy

Authentication deberá poder operar con:

```text
TenantContext
```

cuando la aplicación sea multi-tenant.

---

# 134. Tenant resolution ordering

Es crítico definir:

```text
How is tenant resolved before authentication?
```

Podría provenir de:

- hostname;
- route;
- organization slug;
- trusted gateway metadata;
- explicit login selection.

---

# 135. Tenant context before identity query

Cuando las identidades sean tenant-scoped:

```text
Tenant Resolution
      ↓
Authentication Database Context
      ↓
Identity Lookup
```

---

# 136. User input danger

No deberá confiarse ciegamente:

```text
tenant_id from form
```

como contexto de aislamiento.

---

# 137. Tenant-scoped identity

Puede existir:

```text
same email
```

en varios tenants.

Entonces la identidad lógica podría ser:

```text
TenantId + NormalizedEmail
```

---

# 138. Global identity

Otra arquitectura puede tener:

```text
global user
+
tenant memberships
```

La integración deberá soportar ambos modelos.

---

# 139. Database-per-tenant

Flujo:

```text
TenantContext
     ↓
TenantConnectionResolver
     ↓
Tenant Database
     ↓
Identity Query
```

---

# 140. Shared database tenancy

Flujo:

```text
Identity Query
WHERE tenant_id = ?
```

mediante scope estructurado.

---

# 141. Tenant isolation

Una consulta Authentication jamás deberá omitir accidentalmente el scope tenant requerido.

---

# 142. Cross-tenant administration

Deberá ser:

```text
explicit
authorized
audited
```

y no una consecuencia de omitir un filtro.

---

# 143. Sharding

Authentication puede operar sobre bases shardeadas.

---

# 144. Shard key

Idealmente el identificador de login permitirá resolver:

```text
Shard
```

sin scatter global.

---

# 145. Global email lookup problem

Si los usuarios están shardeados por:

```text
user_id
```

pero el login utiliza:

```text
email
```

puede no saberse el shard.

---

# 146. Solutions

Podrían utilizarse:

```text
Global Identity Directory
Email → Shard Mapping
Shard by normalized identifier
Central Authentication Store
```

---

# 147. Silent scatter prohibited

Authentication no deberá consultar todos los shards silenciosamente para cada login salvo arquitectura explícita.

---

# 148. Enumeration amplification

Un scatter global podría:

- aumentar latencia;
- aumentar carga;
- facilitar DoS;
- ampliar side channels.

---

# 149. Authentication directory

Se podrá definir:

```text
AuthenticationIdentityDirectory
```

como servicio especializado.

---

# 150. Directory ≠ user database

El directory puede contener únicamente:

```text
identity locator
→ shard/tenant/reference
```

sin almacenar toda la entidad.

---

# 151. Authentication security and query input

Todos los identifiers deberán viajar como:

```text
bound parameters
```

---

# 152. Dynamic table names

No se aceptarán desde request.

---

# 153. Provider metadata

Tablas, columnas y mappings deberán provenir de:

```text
trusted provider configuration
compiled metadata
ORM metadata
```

---

# 154. Raw SQL

Los providers oficiales no deberán necesitar raw SQL para operaciones comunes.

---

# 155. Authentication-specific query builder

Podrá existir internamente:

```text
AuthenticationIdentityQueryFactory
```

que produzca Query Models.

---

# 156. Query factory

Ejemplo conceptual:

```php
$query = $identityQueryFactory->byIdentifier(
    provider: $provider,
    identifier: $identifier,
    context: $context,
);
```

---

# 157. Query factory ≠ SQL compiler

Sólo genera intención/query model.

---

# 158. Sensitive parameter telemetry

Aunque Database soporte query telemetry:

```text
email
token
credential selector
session id
```

deberán clasificarse según política de sensibilidad.

---

# 159. Query fingerprints

Se utilizarán preferentemente:

```text
semantic fingerprint
```

sin incluir valores.

---

# 160. Telemetry architecture

La integración podrá producir spans como:

```text
authentication.identity.lookup
authentication.session.load
authentication.token.lookup
authentication.credential.rehash.persist
```

---

# 161. Database spans

Database producirá además:

```text
database.query
database.connection
database.transaction
```

y podrán correlacionarse mediante:

```text
OperationId
AuthenticationAttemptId
TraceId
```

---

# 162. Telemetry separation

Authentication telemetry explica:

```text
why
```

Database telemetry explica:

```text
how persistence behaved
```

---

# 163. No password telemetry

Nunca:

```text
password
password_hash
TOTP secret
refresh token
recovery code
WebAuthn challenge secret
session secret
```

---

# 164. Authentication audit

Debe distinguirse:

```text
Authentication Audit
```

de:

```text
Database Query Audit
```

---

# 165. Authentication audit examples

```text
login succeeded
login failed
MFA enrolled
password changed
token revoked
device trusted
```

---

# 166. Database audit examples

```text
security record updated
credential row modified
session deleted
```

---

# 167. Correlation

Ambos podrán correlacionarse sin duplicar material sensible.

---

# 168. Events

Authentication puede emitir:

```text
AuthenticationSucceeded
AuthenticationFailed
PasswordChanged
MfaEnabled
TokenRevoked
```

Database no deberá redefinir esos eventos.

---

# 169. Database events

Database seguirá emitiendo sus propios:

```text
QueryExecuted
TransactionCommitted
PersistenceCompleted
```

---

# 170. Event causality

Podrá existir:

```text
PasswordChanged
caused by
AuthenticationOperationId
```

y Database events relacionados mediante el mismo contexto.

---

# 171. afterCommit event publication

Un evento que afirme:

```text
PasswordChanged
```

deberá emitirse/publicarse según la semántica transaccional apropiada.

---

# 172. Authentication failure

Un:

```text
invalid password
```

no es:

```text
Database failure
```

---

# 173. Database failure

Un:

```text
connection timeout
```

no deberá transformarse automáticamente en:

```text
invalid credentials
```

internamente.

---

# 174. External response

Por seguridad, la capa HTTP puede devolver un mensaje genérico.

Pero internamente la clasificación deberá preservarse.

---

# 175. Error taxonomy

Se propone:

```text
AuthenticationDatabaseException
├── IdentityLookupException
├── AuthenticationPersistenceException
├── AuthenticationDatabaseUnavailableException
├── AuthenticationConsistencyException
├── AuthenticationStateConflictException
└── AuthenticationDatabaseSecurityException
```

---

# 176. NotFound

```text
identity not found
```

no necesariamente será excepción.

Podrá representarse:

```php
?AuthenticatableIdentity
```

o mediante resultado tipado.

---

# 177. Ambiguous result

```text
>1 identity
```

cuando la identidad debe ser única sí constituye inconsistencia.

---

# 178. Constraint violation

Ejemplo:

```text
duplicate external identity
```

deberá mapearse a un error semántico apropiado.

---

# 179. Optimistic locking

Puede utilizarse para estados como:

```text
security settings
device trust
credential metadata
```

cuando sea apropiado.

---

# 180. Pessimistic locking

Sólo en operaciones que realmente lo requieran.

No para cada login.

---

# 181. Login concurrency

Dos logins simultáneos pueden ser válidos dependiendo de política.

Database no deberá imponer exclusividad universal.

---

# 182. Single-session policy

Si Authentication define:

```text
one active session
```

deberá implementarse explícitamente mediante constraints/transacciones/locking/versioning apropiados.

---

# 183. Throttling

Authentication Throttling podrá usar:

```text
Cache
Database
specialized rate-limit backend
```

según configuración.

---

# 184. Database-backed throttling

Si se usa Database, deberá evitar hot rows y contention excesivo.

---

# 185. Authentication lookup performance

Las consultas críticas deberán estar respaldadas por índices adecuados.

Ejemplo:

```text
normalized_email
```

o:

```text
tenant_id + normalized_email
```

---

# 186. Index ≠ validation

El sistema podrá diagnosticar ausencia de índice esperado, pero no deberá crear índices automáticamente durante login.

---

# 187. Query timeout

Authentication lookup deberá tener un timeout bounded.

---

# 188. Slow authentication query

Podrá ser detectada por:

```text
Slow Query Detection
```

y correlacionada con Authentication telemetry.

---

# 189. Resource governance

Authentication podrá definir budgets:

```text
max DB queries per attempt
max DB duration per attempt
max connection wait
max result rows
```

---

# 190. Identity lookup cardinality

Normalmente:

```text
0 or 1
```

será la cardinalidad esperada.

---

# 191. Excess rows

Si aparecen:

```text
2+
```

no deberá hidratarse una colección completa innecesariamente.

Puede detectarse con límite semántico:

```text
up to 2
```

para comprobar ambigüedad.

---

# 192. Authentication N+1

Un login no deberá disparar accidentalmente:

```text
user query
roles query
permissions query
MFA query
devices query
organization query
...
```

si no son necesarios para demostrar autenticación.

---

# 193. Authentication vs Authorization

Especialmente:

```text
roles
permissions
policies
```

pertenecen principalmente a Authorization.

No deberán cargarse durante Authentication salvo necesidad explícita.

---

# 194. Lazy loading danger

Serializar un principal Authentication no deberá activar:

```text
lazy ORM queries
```

accidentalmente.

---

# 195. Authentication projection advantage

Por ello se favorecen DTO/projections especializados.

---

# 196. Persistent runtime architecture

FrankenPHP mantendrá workers vivos.

Por tanto:

```text
AuthenticationDatabaseContext
```

no podrá sobrevivir entre requests.

---

# 197. Worker model

```text
FrankenPHP Worker
│
├── Request A
│   ├── AuthenticationContext A
│   └── DatabaseContext A
│
├── reset
│
└── Request B
    ├── AuthenticationContext B
    └── DatabaseContext B
```

---

# 198. No static authenticated identity

Database Integration nunca almacenará:

```php
static $currentUser;
```

---

# 199. No static tenant

Tampoco:

```php
static $currentTenant;
```

---

# 200. No static credentials

Especialmente prohibido:

```php
static $passwordHash;
static $token;
```

---

# 201. Shared immutable components

Sí podrán compartirse:

```text
Provider Metadata
Identity Mapping Metadata
Query Factories
Compiled Metadata
Constraint Mapping
Stateless Adapters
```

---

# 202. Scoped components

Deberán ser scoped:

```text
AuthenticationDatabaseContext
Attempt correlation
Transaction reference
Tenant reference
Shard reference
Temporary identity state
```

---

# 203. Reset

Al finalizar la operación/request:

```text
authentication DB context
temporary results
sensitive references
transaction references
tenant references
```

deberán liberarse.

---

# 204. RoadRunner

Aplicará las mismas reglas.

---

# 205. OpenSwoole

Deberá garantizar:

```text
coroutine isolation
```

entre autenticaciones concurrentes.

---

# 206. Coroutine example

```text
Coroutine A
Tenant A
User A

Coroutine B
Tenant B
User B
```

no deberán compartir ningún estado mutable Authentication/Database.

---

# 207. Configuration

Ejemplo conceptual:

```php
'authentication' => [

    'providers' => [

        'users' => [
            'driver' => 'database',
            'entity' => App\Entity\User::class,

            'identifier' => [
                'field' => 'normalized_email',
            ],

            'security_fields' => [
                'password_hash',
                'status',
                'security_version',
            ],

            'consistency' => 'security_sensitive',
        ],

    ],

];
```

---

# 208. Table-based provider

También podrá configurarse sin ORM:

```php
'providers' => [

    'users' => [
        'driver' => 'database-table',
        'table' => 'users',
        'identifier' => 'normalized_email',
        'id' => 'id',
        'credential' => 'password_hash',
    ],

];
```

---

# 209. Trusted configuration

Nombres de:

```text
table
columns
connection
entity
```

deberán provenir de configuración confiable.

---

# 210. Configuration compilation

Idealmente estos mappings serán:

```text
validated
normalized
compiled
frozen
```

durante bootstrap.

---

# 211. Runtime config mutation

No deberá cambiarse el provider mapping arbitrariamente durante una request.

---

# 212. Provider metadata

Se propone:

```text
DatabaseAuthenticationProviderMetadata
├── ProviderId
├── StorageKind
├── EntityType?
├── Table?
├── IdentifierMapping
├── CredentialMapping
├── StateMapping
├── ConnectionIntent
├── ConsistencyPolicy
├── TenantPolicy
├── ShardPolicy
└── SecurityClassification
```

---

# 213. Provider registry

```text
DatabaseAuthenticationProviderRegistry
```

deberá congelarse después de bootstrap.

---

# 214. Multiple providers

VoltStack podrá soportar:

```text
customers
employees
administrators
service_accounts
```

con providers distintos.

---

# 215. Provider ≠ Guard

Un Guard puede utilizar un Provider.

Pero:

```text
Provider
≠
Guard
```

---

# 216. Provider ≠ Firewall

Igualmente:

```text
Provider
≠
Firewall
```

---

# 217. Database provider ≠ authentication mechanism

Un mismo DatabaseIdentityProvider podrá servir:

```text
password authentication
MFA continuation
session refresh
remember-me
```

sin convertirse en authenticator.

---

# 218. Service accounts

Podrán tener storage separado y credential types distintos.

---

# 219. Machine credentials

API keys deberán seguir:

```text
selector
+
secret verification
```

cuando corresponda.

---

# 220. API key storage

Preferir:

```text
key identifier
key digest
metadata
status
expires_at
```

sobre plaintext key.

---

# 221. API key lookup

```text
presented API key
      ↓
extract selector
      ↓
Database lookup
      ↓
verify secret against digest
```

---

# 222. Secret rotation

Podrá requerir períodos:

```text
current
previous
grace
revoked
```

definidos por Authentication.

---

# 223. Database security model integration

Este sistema deberá respetar:

```text
DATABASE_SECURITY_ARCHITECTURE
DATABASE_CREDENTIAL_SECURITY_SYSTEM
DATABASE_CONNECTION_SECURITY_SYSTEM
DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM
DATABASE_QUERY_AUDIT_SYSTEM
```

---

# 224. Database credentials vs user credentials

Deben distinguirse claramente:

```text
Database Credential
```

de:

```text
Application User Credential
```

---

# 225. Example

```text
DB username/password
```

permite conectar al DBMS.

```text
User password
```

demuestra identidad de aplicación.

Nunca deberán compartir lifecycle ni storage.

---

# 226. Debug toolbar

No deberá mostrar:

- password hashes;
- tokens;
- session secrets;
- MFA secrets;
- recovery codes;
- credential IDs sensibles completos.

---

# 227. Query debugging

Podrá mostrar:

```text
SELECT ... WHERE normalized_email = ?
```

pero no necesariamente el valor del parámetro.

---

# 228. Redaction classification

Campos Authentication deberán clasificarse, por ejemplo:

```text
PUBLIC
INTERNAL
PERSONAL
SENSITIVE
SECRET
CREDENTIAL
```

---

# 229. SECRET / CREDENTIAL

Por defecto:

```text
never log
never trace
never expose in debug toolbar
never include in exception context
```

---

# 230. Backup implications

Los backups pueden contener:

```text
password hashes
encrypted MFA secrets
session state
tokens
```

por tanto deberán respetar el modelo de seguridad definido en Backup Architecture.

---

# 231. Restore implications

Restaurar un backup antiguo puede:

```text
restore revoked credentials
restore old password hashes
restore old sessions
restore old security state
```

---

# 232. Authentication-aware recovery

Los planes de disaster recovery deberán considerar este riesgo.

---

# 233. Restore ≠ safe authentication state

Después de ciertos restores puede requerirse:

```text
global session invalidation
token revocation
security version bump
credential reconciliation
```

según política.

---

# 234. Testing architecture

La integración deberá probarse en:

```text
Unit
Integration
Security
Concurrency
Transaction
Replica
Multitenancy
Sharding
Persistent Runtime
Failure
Performance
```

---

# 235. Unit tests

Deberán cubrir:

- identity mapping;
- provider metadata;
- context resolution;
- query construction;
- sensitive field classification;
- error mapping;
- consistency policy;
- retry classification.

---

# 236. Integration tests

Con DBMS real:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

según capacidad.

---

# 237. Authentication lookup test

Crear identidad:

```text
alice@example.com
```

y verificar:

```text
normalized identifier
→ exactly one identity
```

---

# 238. Missing identity

Deberá producir:

```text
not found
```

sin Database exception.

---

# 239. Duplicate identity corruption

Crear un escenario sin constraint o mediante fixture controlada y comprobar:

```text
2 matching identities
→ integrity violation
```

---

# 240. Password test

Comprobar que:

```text
plaintext password
```

nunca aparezca en:

- SQL;
- query log;
- telemetry;
- exception;
- debug data.

---

# 241. Password hash test

Verificar que el hash sólo se recupere cuando sea necesario para autenticación.

---

# 242. Replica stale test

Escenario:

```text
Writer password = H2
Replica password = H1
```

Authentication sensible no deberá aceptar H1 por utilizar una réplica stale.

---

# 243. Account disable test

```text
Writer: DISABLED
Replica: ACTIVE
```

La política crítica deberá impedir autenticación basada en estado obsoleto.

---

# 244. Token revocation test

```text
Writer: REVOKED
Replica: ACTIVE
```

no deberá aceptarse bajo política de revocación inmediata.

---

# 245. Transaction test

Probar operaciones como:

```text
refresh token rotation
recovery code consumption
security version increment
```

bajo transacciones reales.

---

# 246. Concurrent recovery code test

Dos procesos intentan consumir el mismo código.

Resultado esperado:

```text
at most one successful consumption
```

cuando ésa sea la política.

---

# 247. Concurrent refresh rotation

Verificar protección frente a replay/races según Authentication specification.

---

# 248. Tenant isolation test

```text
Tenant A / alice@example.com
Tenant B / alice@example.com
```

deberán resolverse correctamente.

---

# 249. Cross-tenant leak test

Una autenticación en Tenant A jamás deberá recuperar la identidad de Tenant B.

---

# 250. Shard test

El identity directory deberá resolver el shard correcto.

---

# 251. Unknown shard

No deberá interpretarse como:

```text
invalid credentials
```

internamente.

Debe preservarse la causa operacional.

---

# 252. Persistent runtime test

En el mismo worker:

```text
Request 1
Tenant A
Alice

Request 2
Tenant B
Bob
```

no deberá sobrevivir ningún estado mutable de la primera autenticación.

---

# 253. Failure injection

Deberán probarse:

```text
connection loss
timeout
deadlock
commit ambiguity
replica unavailable
writer unavailable
pool exhaustion
```

---

# 254. Authentication failure classification

Los tests deberán demostrar:

```text
Database unavailable
≠
Invalid credentials
```

aunque la respuesta HTTP externa pueda ser deliberadamente genérica.

---

# 255. Performance tests

Medir:

```text
identity lookup latency
session lookup latency
token lookup latency
connection acquisition
projection hydration
security-state refresh
```

sin sacrificar semántica de seguridad.

---

# 256. Correctness first

Una consulta más rápida contra réplica stale no constituye una optimización válida si permite credenciales revocadas.

---

# 257. Proposed namespace

```text
VoltStack\Quantum\Database\Integration\Authentication
```

---

# 258. Proposed directory structure

```text
src/Quantum/Database/Integration/Authentication/
├── Contract/
│   ├── DatabaseIdentityRepository.php
│   ├── AuthenticationPersistenceAdapter.php
│   ├── AuthenticationStateStore.php
│   └── AuthenticationDirectory.php
│
├── Context/
│   ├── AuthenticationDatabaseContext.php
│   ├── AuthenticationDatabaseContextResolver.php
│   ├── AuthenticationConnectionIntent.php
│   └── AuthenticationConsistencyPolicy.php
│
├── Provider/
│   ├── DatabaseIdentityProvider.php
│   ├── EntityIdentityProvider.php
│   ├── TableIdentityProvider.php
│   ├── DatabaseAuthenticationProviderMetadata.php
│   └── DatabaseAuthenticationProviderRegistry.php
│
├── Identity/
│   ├── DatabaseIdentityRecord.php
│   ├── DatabaseIdentityMapper.php
│   ├── AuthenticationSecurityState.php
│   └── AuthenticationIdentityProjection.php
│
├── Query/
│   ├── AuthenticationIdentityQueryFactory.php
│   ├── AuthenticationSecurityStateQueryFactory.php
│   ├── SessionQueryFactory.php
│   └── TokenQueryFactory.php
│
├── Credential/
│   ├── DatabaseCredentialPersistenceAdapter.php
│   ├── PasswordHashPersistenceAdapter.php
│   └── CredentialVersionStore.php
│
├── Session/
│   ├── DatabaseSessionStore.php
│   ├── DatabaseSessionRecord.php
│   └── SessionPersistenceMapper.php
│
├── Token/
│   ├── DatabaseTokenStore.php
│   ├── RefreshTokenStore.php
│   ├── RememberTokenStore.php
│   └── ApiKeyStore.php
│
├── MFA/
│   ├── DatabaseMfaStore.php
│   ├── TotpCredentialStore.php
│   └── RecoveryCodeStore.php
│
├── WebAuthn/
│   ├── DatabaseWebAuthnCredentialStore.php
│   └── WebAuthnCredentialMapper.php
│
├── OAuth/
│   ├── DatabaseExternalIdentityStore.php
│   └── DatabaseOAuthTokenStore.php
│
├── Device/
│   ├── DatabaseDeviceTrustStore.php
│   └── DeviceTrustMapper.php
│
├── Tenant/
│   └── AuthenticationTenantContextContributor.php
│
├── Sharding/
│   ├── AuthenticationShardResolver.php
│   └── DatabaseAuthenticationDirectory.php
│
├── Security/
│   ├── AuthenticationDataClassification.php
│   ├── AuthenticationDatabaseRedactor.php
│   └── AuthenticationDatabaseSecurityPolicy.php
│
├── Telemetry/
│   └── AuthenticationDatabaseTelemetry.php
│
├── Extension/
│   ├── AuthenticationDatabaseExtension.php
│   └── AuthenticationDatabaseExtensionRegistry.php
│
└── Exception/
    ├── AuthenticationDatabaseException.php
    ├── IdentityLookupException.php
    ├── IdentityIntegrityViolationException.php
    ├── AuthenticationDatabaseUnavailableException.php
    ├── AuthenticationStateConflictException.php
    └── AuthenticationDatabaseSecurityException.php
```

---

# 259. Dependency diagram

```text
VoltStack Authentication
         │
         │ contracts
         ▼
Authentication Integration
         │
         ├───────────────┐
         ▼               ▼
     Query API          ORM API
         │               │
         └───────┬───────┘
                 ▼
        Database Core
                 │
         ┌───────┼────────┐
         ▼       ▼        ▼
    Transaction Cache  Telemetry
         │
         ▼
     Connection
         │
         ▼
       Driver
```

No deberá existir dependencia inversa:

```text
Database Core
     ↓
Authentication Core
```

---

# 260. Core invariants

## DB-AUTH-INT-001

Database ≠ Authentication.

## DB-AUTH-INT-002

User Record ≠ Authenticated Identity.

## DB-AUTH-INT-003

Entity ≠ Authenticated Principal.

## DB-AUTH-INT-004

Credential Exists ≠ Credential Valid.

## DB-AUTH-INT-005

Successful Database Lookup ≠ Successful Authentication.

## DB-AUTH-INT-006

Database no verificará passwords.

## DB-AUTH-INT-007

Database no ejecutará MFA policy.

## DB-AUTH-INT-008

Database no decidirá authentication success.

## DB-AUTH-INT-009

Authentication no accederá directamente al Driver.

## DB-AUTH-INT-010

Authentication no generará SQL directamente.

---

# 261. Credential invariants

## DB-AUTH-INT-011

Plaintext passwords no serán persistidos.

## DB-AUTH-INT-012

Plaintext passwords no aparecerán en telemetry.

## DB-AUTH-INT-013

Plaintext passwords no aparecerán en query logs.

## DB-AUTH-INT-014

Plaintext passwords no aparecerán en exceptions.

## DB-AUTH-INT-015

Password hashes serán tratados como sensitive data.

## DB-AUTH-INT-016

TOTP secrets estarán protegidos.

## DB-AUTH-INT-017

Recovery codes no se almacenarán en plaintext cuando sólo requieran verificación.

## DB-AUTH-INT-018

Bearer secrets no se almacenarán en plaintext cuando pueda utilizarse hash/digest.

## DB-AUTH-INT-019

Application credentials ≠ Database credentials.

## DB-AUTH-INT-020

Credential verification pertenecerá a Authentication.

---

# 262. Identity invariants

## DB-AUTH-INT-021

Identity lookup tendrá cardinalidad explícita.

## DB-AUTH-INT-022

Multiple matches no elegirán arbitrariamente un registro.

## DB-AUTH-INT-023

Identifier uniqueness persistente utilizará constraint cuando corresponda.

## DB-AUTH-INT-024

Identifier normalization pertenecerá a Authentication/domain.

## DB-AUTH-INT-025

DB collation ≠ application normalization.

## DB-AUTH-INT-026

External email ≠ external stable subject.

## DB-AUTH-INT-027

OIDC identity utilizará issuer/provider + subject cuando corresponda.

## DB-AUTH-INT-028

Identity projection recuperará sólo datos necesarios.

## DB-AUTH-INT-029

IdentityMap ≠ fresh security state.

## DB-AUTH-INT-030

ORM entity state no se asumirá actualizado para decisiones críticas.

---

# 263. Consistency invariants

## DB-AUTH-INT-031

Security-sensitive reads declararán consistencia.

## DB-AUTH-INT-032

UNKNOWN replica lag ≠ fresh replica.

## DB-AUTH-INT-033

Stale password hash no deberá aceptarse cuando se requiere estado actual.

## DB-AUTH-INT-034

Stale account state no deberá decidir autenticación crítica.

## DB-AUTH-INT-035

Stale revocation state no deberá aceptarse bajo revocación inmediata.

## DB-AUTH-INT-036

Authentication declara consistencia; Database selecciona endpoint.

## DB-AUTH-INT-037

Requested consistency ≠ effective consistency.

## DB-AUTH-INT-038

Incapacidad para satisfacer consistencia será explícita.

## DB-AUTH-INT-039

Sticky writer podrá utilizarse tras security mutations.

## DB-AUTH-INT-040

Cache stale ≠ security authority.

---

# 264. Transaction invariants

## DB-AUTH-INT-041

Authentication Attempt ≠ Transaction.

## DB-AUTH-INT-042

Transaction scope será mínimo.

## DB-AUTH-INT-043

External network interaction no mantendrá transaction abierta por defecto.

## DB-AUTH-INT-044

Unknown commit permanecerá UNKNOWN.

## DB-AUTH-INT-045

Unknown commit no se reinterpretará como rollback.

## DB-AUTH-INT-046

Non-replayable auth operation no se reintentará ciegamente.

## DB-AUTH-INT-047

Recovery code consumption será atómico cuando corresponda.

## DB-AUTH-INT-048

Token rotation preservará atomicidad definida.

## DB-AUTH-INT-049

Deadlock retry requerirá replayability.

## DB-AUTH-INT-050

Rollback ≠ authentication object-state rewind.

---

# 265. Session/token invariants

## DB-AUTH-INT-051

Session secret no será logueado.

## DB-AUTH-INT-052

Token secret no será logueado.

## DB-AUTH-INT-053

Session Store ≠ Authentication Engine.

## DB-AUTH-INT-054

Token Store ≠ Token Verifier.

## DB-AUTH-INT-055

Token revocation será consistency-aware.

## DB-AUTH-INT-056

Refresh rotation será concurrency-aware.

## DB-AUTH-INT-057

Token selector ≠ token secret.

## DB-AUTH-INT-058

API key digest ≠ API key plaintext.

## DB-AUTH-INT-059

Security version podrá invalidar estado derivado.

## DB-AUTH-INT-060

Session persistence no requerirá ORM obligatoriamente.

---

# 266. Tenant invariants

## DB-AUTH-INT-061

Tenant context se resolverá antes del lookup cuando la identidad sea tenant-scoped.

## DB-AUTH-INT-062

Tenant ID no será confiado desde input arbitrario.

## DB-AUTH-INT-063

Authentication no cruzará tenants accidentalmente.

## DB-AUTH-INT-064

Same identifier podrá existir en tenants distintos si el modelo lo permite.

## DB-AUTH-INT-065

Database-per-tenant utilizará TenantConnectionResolver.

## DB-AUTH-INT-066

Shared-table tenancy aplicará tenant scope.

## DB-AUTH-INT-067

Cross-tenant authentication será explícita.

## DB-AUTH-INT-068

Cross-tenant operations serán auditadas.

## DB-AUTH-INT-069

Multitenancy continuará siendo opcional.

## DB-AUTH-INT-070

Tenant state no será global mutable.

---

# 267. Sharding invariants

## DB-AUTH-INT-071

Authentication deberá resolver shard antes de query cuando sea posible.

## DB-AUTH-INT-072

Unknown shard ≠ identity not found.

## DB-AUTH-INT-073

No habrá scatter global silencioso.

## DB-AUTH-INT-074

Global identity directory podrá resolver shard.

## DB-AUTH-INT-075

Directory ≠ identity database.

## DB-AUTH-INT-076

Shard lookup no expondrá side channels innecesarios.

## DB-AUTH-INT-077

Shard map generation podrá formar parte del contexto.

## DB-AUTH-INT-078

Stale routing evidence no será certeza.

## DB-AUTH-INT-079

Cross-shard transaction no se fingirá.

## DB-AUTH-INT-080

Shard context será operation-scoped.

---

# 268. Runtime invariants

## DB-AUTH-INT-081

AuthenticationDatabaseContext será scoped.

## DB-AUTH-INT-082

No habrá authenticated identity mutable global en Database Integration.

## DB-AUTH-INT-083

No habrá tenant mutable global.

## DB-AUTH-INT-084

No habrá credential mutable global.

## DB-AUTH-INT-085

FrankenPHP requests estarán aisladas.

## DB-AUTH-INT-086

RoadRunner requests estarán aisladas.

## DB-AUTH-INT-087

OpenSwoole coroutines estarán aisladas.

## DB-AUTH-INT-088

Temporary sensitive references serán liberadas.

## DB-AUTH-INT-089

Transaction references no sobrevivirán al scope.

## DB-AUTH-INT-090

Reset failure será visible.

---

# 269. Security invariants

## DB-AUTH-INT-091

Todos los query values serán parameterized.

## DB-AUTH-INT-092

Provider identifiers serán trusted metadata.

## DB-AUTH-INT-093

User input no seleccionará tablas arbitrarias.

## DB-AUTH-INT-094

User input no seleccionará columnas arbitrarias.

## DB-AUTH-INT-095

Sensitive fields estarán clasificados.

## DB-AUTH-INT-096

Debug toolbar aplicará redaction.

## DB-AUTH-INT-097

Database telemetry aplicará redaction.

## DB-AUTH-INT-098

Authentication telemetry aplicará redaction.

## DB-AUTH-INT-099

Internal error classification se preservará.

## DB-AUTH-INT-100

External messages podrán ser deliberadamente genéricos.

---

# 270. Performance invariants

## DB-AUTH-INT-101

Login no cargará datos innecesarios.

## DB-AUTH-INT-102

SELECT * no será default para identity lookup.

## DB-AUTH-INT-103

Authentication no cargará Authorization graph por defecto.

## DB-AUTH-INT-104

Lazy loading accidental deberá evitarse.

## DB-AUTH-INT-105

Identity lookup tendrá índices apropiados.

## DB-AUTH-INT-106

Performance optimization no reducirá security consistency.

## DB-AUTH-INT-107

Authentication query budgets serán bounded.

## DB-AUTH-INT-108

Slow queries serán observables.

## DB-AUTH-INT-109

Connection acquisition será observable.

## DB-AUTH-INT-110

Cache optimization preservará revocation semantics.

---

# 271. Event/audit invariants

## DB-AUTH-INT-111

Authentication Event ≠ Database Event.

## DB-AUTH-INT-112

Authentication Audit ≠ Database Audit.

## DB-AUTH-INT-113

Ambos podrán correlacionarse.

## DB-AUTH-INT-114

afterCommit respetará transaction outcome.

## DB-AUTH-INT-115

Rollback no publicará cambios no persistidos como definitivos.

## DB-AUTH-INT-116

UNKNOWN commit no producirá certeza falsa.

## DB-AUTH-INT-117

Audit no incluirá secrets.

## DB-AUTH-INT-118

Trace no incluirá secrets.

## DB-AUTH-INT-119

Query fingerprints no incluirán credentials.

## DB-AUTH-INT-120

AuthenticationAttemptId no contendrá PII/credentials.

---

# 272. Anti-pattern: authentication SQL

Incorrecto:

```php
$sql = "
    SELECT *
    FROM users
    WHERE email = '$email'
      AND password = '$password'
";
```

Viola:

- parameter binding;
- password hashing;
- Authentication boundary;
- sensitive-data handling.

---

# 273. Anti-pattern: password comparison in SQL

Incorrecto:

```sql
WHERE password_hash = ?
```

utilizando directamente una transformación insegura del password.

La verificación pertenece al Password Hasher.

---

# 274. Anti-pattern: plaintext password

Nunca:

```text
users.password = "my-password"
```

---

# 275. Anti-pattern: stale replica

Incorrecto:

```text
Account disabled on writer
        ↓
Login checks replica
        ↓
Replica says active
        ↓
Login accepted
```

---

# 276. Anti-pattern: user entity as session

No serializar una entidad ORM completa como:

```text
session authentication state
```

sin contrato explícito.

---

# 277. Anti-pattern: authorization eager loading

No cargar:

```text
User
├── Roles
├── Permissions
├── Organizations
├── Projects
├── Teams
└── Policies
```

durante cada password lookup salvo necesidad real.

---

# 278. Anti-pattern: token plaintext storage

Evitar:

```text
database.refresh_token = original bearer token
```

cuando sea posible una estrategia más segura.

---

# 279. Anti-pattern: raw tenant ID

Incorrecto:

```php
$tenant = $_POST['tenant_id'];

DB::forTenant($tenant)->findUser(...);
```

sin resolver/verificar contexto.

---

# 280. Anti-pattern: global current user

Incorrecto en persistent runtimes:

```php
DatabaseAuthentication::$currentUser = $user;
```

---

# 281. Anti-pattern: retry unknown commit

Incorrecto:

```text
COMMIT sent
connection lost
→ retry token consumption
```

sin reconciliar outcome.

---

# 282. Anti-pattern: Database exception = invalid password

Incorrecto:

```text
ConnectionException
→ "wrong password"
```

como clasificación interna.

---

# 283. Anti-pattern: cache forever

Nunca conservar indefinidamente:

```text
account = ACTIVE
```

cuando puede ser revocado/deshabilitado.

---

# 284. Anti-pattern: WebAuthn private material

Database nunca deberá esperar:

```text
authenticator private key
```

---

# 285. Anti-pattern: OIDC identity by email only

No asumir:

```text
same email
=
same external identity
```

sin política explícita.

---

# 286. Formal identity model

Sea:

```text
I = Authentication Identifier
P = Identity Provider
C = Authentication Context
```

el lookup Database produce:

```text
L(P, I, C) → {0,1,n}
```

donde:

```text
0 = no identity
1 = unique identity
n > 1 = integrity violation
```

para providers cuya cardinalidad esperada sea única.

---

# 287. Authentication model

Posteriormente:

```text
A(
    Identity,
    PresentedCredential,
    AuthenticationPolicy,
    Context
)
→
AuthenticationResult
```

Por tanto:

```text
L(...) = 1
```

no implica:

```text
AuthenticationResult = SUCCESS
```

---

# 288. Credential model

Sea:

```text
S = stored credential representation
P = presented credential
V = credential verifier
```

entonces:

```text
V(P, S) → VerificationResult
```

Database sólo participa en obtener/persistir:

```text
S
```

---

# 289. Consistency model

Una decisión sensible:

```text
D
```

será válida respecto al estado observado:

```text
S(t)
```

y nivel de consistencia:

```text
C
```

Por tanto:

```text
AuthenticationEvidence =
f(
    IdentityState,
    CredentialState,
    SecurityState,
    Consistency,
    Tenant,
    Shard,
    Time
)
```

---

# 290. Revocation principle

Si:

```text
revocation(t1)
```

debe tener efecto inmediato, una lectura posterior:

```text
authentication(t2)
```

no puede depender de una fuente que sólo garantice:

```text
state < t1
```

---

# 291. Security correctness equation

Conceptualmente:

```text
Authentication Database Correctness
=
Correct Identity Resolution
∧
Required Consistency
∧
Tenant Isolation
∧
Shard Correctness
∧
Credential Protection
∧
Transaction Correctness
∧
Runtime Isolation
```

---

# 292. Recovery model

Para una operación de seguridad:

```text
Write
↓
Commit request
↓
Connection lost
```

si no puede demostrarse:

```text
COMMITTED
```

ni:

```text
ROLLED_BACK
```

entonces:

```text
Outcome = UNKNOWN
```

y Authentication deberá reaccionar conservadoramente.

---

# 293. V1 scope

La V1 deberá incluir como mínimo:

```text
DatabaseIdentityProvider
Entity-backed provider
Table-backed provider
AuthenticationDatabaseContext
Security-sensitive read intent
Writer-aware routing
Identity projection
Password hash persistence
Session store
Remember-token store
Basic token persistence
MFA persistence contracts
WebAuthn persistence contracts
External identity persistence
Tenant-aware provider integration
Sensitive-data redaction
Authentication/Database telemetry correlation
FrankenPHP isolation
Unit tests
Real DB integration tests
Concurrency tests
Replica consistency tests
```

---

# 294. V2

Podrá incorporar:

```text
Authentication Identity Directory
Shard-aware identity resolution
Advanced token-family persistence
Adaptive security-state caching
Device trust persistence
Risk evidence persistence
Compiled authentication provider metadata
Advanced account lockout atomic operations
Authentication-specific diagnostics
```

---

# 295. V3

Podrá incorporar:

```text
distributed identity directory
multi-region security-state consistency policies
advanced credential replication policies
cross-region revocation coordination
security-state reconciliation
authentication persistence performance optimizer
```

sin convertir Database en Authentication Engine.

---

# 296. Resultado arquitectónico

La integración completa quedará conceptualmente:

```text
                       VoltStack Authentication
                                  │
             ┌────────────────────┼────────────────────┐
             │                    │                    │
         Password              WebAuthn             OAuth/OIDC
             │                    │                    │
             └────────────────────┼────────────────────┘
                                  │
                         Identity Provider
                                  │
                                  ▼
                  Database Authentication Bridge
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
              ▼                   ▼                   ▼
       Identity Store       Session Store        Token Store
              │                   │                   │
              └───────────────────┼───────────────────┘
                                  ▼
                         Database Query/ORM
                                  │
                                  ▼
                         Execution Engine
                                  │
                                  ▼
                     Read/Write Routing System
                                  │
                 ┌────────────────┴────────────────┐
                 ▼                                 ▼
              Writer                            Replica
                 │                                 │
                 └──────────────┬──────────────────┘
                                ▼
                             Driver
                                │
                                ▼
                              DBMS
```

con la regla:

```text
Security-sensitive operation
        ↓
Required Consistency
        ↓
Eligible Endpoint
```

y nunca:

```text
Fastest Endpoint
        ↓
Authentication Decision
```

sin considerar seguridad.

---

# 297. Principio definitivo de identidad

> **La existencia de datos de identidad en Database únicamente demuestra que existe un registro persistido compatible con la consulta realizada; no demuestra por sí misma que el actor que presenta una credencial sea esa identidad.**

Por tanto:

```text
Identity Lookup
≠
Identity Proof
```

---

# 298. Principio definitivo de credenciales

> **Database deberá almacenar únicamente las representaciones persistentes necesarias para verificar o administrar credenciales; la interpretación criptográfica y la decisión sobre su validez pertenecerán a Authentication y Security.**

```text
Database stores.
Authentication verifies.
Security protects.
```

---

# 299. Principio definitivo de consistencia

> **Una optimización de lectura nunca deberá permitir que un estado de seguridad obsoleto sea tratado como estado actual cuando la política de autenticación requiera consistencia más fuerte.**

Por tanto:

```text
Lower latency
≠
Valid optimization
```

si produce:

```text
stale credential state
stale revocation state
stale account state
```

---

# 300. Principio definitivo de persistencia

```text
Authentication
defines meaning.

Database
defines persistence mechanics.

Constraints
protect persistent invariants.

Transactions
protect atomic boundaries.

Security
protects secrets.

Telemetry
observes without exposing them.
```

---

# 301. Principio definitivo de runtime

Para FrankenPHP, RoadRunner y OpenSwoole:

```text
Shared Immutable Authentication Infrastructure
+
Scoped Authentication State
+
Scoped Database Context
+
Deterministic Reset
=
Persistent Runtime Safety
```

---

# 302. Regla arquitectónica final

VoltStack deberá mantener permanentemente:

```text
Database
≠
Authentication
```

```text
Database Record
≠
Identity Proof
```

```text
User Entity
≠
Authenticated Principal
```

```text
Credential Storage
≠
Credential Verification
```

```text
Identity Exists
≠
Credential Valid
```

```text
Credential Valid
≠
Authentication Successful
```

```text
Authentication Successful
≠
Authorization Granted
```

```text
Replica Available
≠
Replica Safe For Authentication
```

```text
Cache Hit
≠
Current Security State
```

```text
Fast Query
≠
Secure Query
```

y especialmente:

```text
Successful Database Lookup
≠
Successful Authentication
```

---

# 303. Siguiente documento

```text
319_DATABASE_AUTHORIZATION_INTEGRATION_SYSTEM.md
```

Este documento definirá la integración entre:

```text
VoltStack/Quantum/Database
        ↕
VoltStack Authorization
```

incluyendo:

```text
authorization data access
policy subject resolution
resource loading
authorization-aware query scopes
row-level access filtering
tenant-aware authorization
ownership constraints
policy context propagation
authorization projections
bulk authorization
authorization query planning
repository integration
ORM integration
Model API integration
authorization cache boundaries
security-sensitive consistency
TOCTOU between authorization and mutation
transaction-aware authorization
database constraints vs authorization rules
authorization audit
telemetry
persistent runtime isolation
testing
```

manteniendo como invariantes centrales:

```text
Authentication
≠
Authorization
```

```text
Database
≠
Authorization
```

```text
Record Exists
≠
Access Granted
```

```text
Query Can Find Resource
≠
Actor May Access Resource
```

```text
Authorization Check
≠
Database Constraint
```

y:

```text
Authorized At t1
≠
Automatically Authorized At t2
```

cuando el estado relevante pueda cambiar concurrentemente.