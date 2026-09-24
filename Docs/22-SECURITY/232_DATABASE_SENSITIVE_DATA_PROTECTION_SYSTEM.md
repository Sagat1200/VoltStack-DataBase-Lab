# 232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM.md

# VoltStack Quantum Database
## Sensitive Data Protection System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 232 — Sensitive Data Protection System  
**Bloque:** 22 — Security  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `231_DATABASE_DATA_ACCESS_SECURITY_SYSTEM.md`  
**Siguiente documento:** `233_DATABASE_QUERY_AUDIT_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura de VoltStack Database para identificar, clasificar, transformar, proteger y controlar el ciclo de vida de información sensible.

El sistema deberá proteger información durante:

```text
Application
    ↓
ORM / Query Builder
    ↓
Parameter Binding
    ↓
Database
    ↓
Storage
    ↓
Replication
    ↓
Backup
    ↓
Result
    ↓
Hydration
    ↓
Cache
    ↓
Telemetry
    ↓
Debugging
    ↓
Export
```

La protección no podrá limitarse a cifrar columnas.

La regla central será:

> **La sensibilidad de un dato deberá formar parte de su metadata semántica y acompañarlo, directa o indirectamente, a través de las fronteras del sistema que puedan almacenarlo, transportarlo, observarlo, cachearlo, registrarlo o exponerlo.**

---

# 2. Principio fundamental

VoltStack mantendrá estrictamente separadas las siguientes responsabilidades:

```text
Classification
≠
Authorization
≠
Encryption
≠
Hashing
≠
Tokenization
≠
Masking
≠
Redaction
≠
Auditing
```

Cada una responde a una pregunta distinta.

| Sistema | Pregunta |
|---|---|
| Classification | ¿Qué tan sensible es este dato? |
| Authorization | ¿Quién puede acceder? |
| Encryption | ¿Cómo impedimos lectura sin la clave? |
| Hashing | ¿Cómo obtenemos representación irreversible? |
| Tokenization | ¿Cómo sustituimos el valor sensible? |
| Masking | ¿Qué representación limitada mostramos? |
| Redaction | ¿Qué debemos eliminar de observabilidad? |
| Auditing | ¿Quién realizó qué operación y cuándo? |

---

# 3. Sensitive Data Protection ≠ Authorization

El documento 231 determina:

```text
Can Principal P access Resource R?
```

Este sistema determina:

```text
How must Resource R be protected?
```

Ejemplo:

```text
Principal = Payroll Administrator
Access = ALLOW
Field = salary
Classification = CONFIDENTIAL
```

Aunque exista autorización:

```text
salary
```

no deberá aparecer automáticamente en:

```text
logs
traces
debug toolbar
exception messages
query dumps
profilers
```

---

# 4. Authorization ≠ safe observability

Un usuario puede estar autorizado a consultar:

```text
Customer.credit_card_token
```

sin que el valor deba aparecer en:

```text
QueryTelemetry
DebugInformation
Profiler
Audit payload
```

Por tanto:

```text
Authorized
≠
SafeToLog
```

---

# 5. Encryption ≠ Authorization

Un valor cifrado correctamente puede ser entregado a un principal no autorizado si el sistema descifra antes de verificar acceso.

Por tanto:

```text
Encrypted
≠
Authorized
```

---

# 6. Encryption ≠ Redaction

Cifrar:

```text
customer@example.com
```

en almacenamiento no evita que aparezca en:

```text
SQL parameter telemetry
exception context
debug dumps
```

una vez descifrado.

---

# 7. Masking ≠ Encryption

Ejemplo:

```text
Original:
4111111111111111

Masked:
************1111
```

El masking protege presentación.

No necesariamente almacenamiento.

---

# 8. Hashing ≠ Encryption

```text
Encryption
→ reversible with key

Hashing
→ intentionally irreversible
```

No deben utilizarse indistintamente.

---

# 9. Tokenization ≠ Hashing

Tokenización:

```text
4111111111111111
        ↓
tok_8F4A19...
```

puede mantener un mapping seguro externo.

Hashing normalmente no.

---

# 10. Objetivos

El sistema deberá proporcionar:

1. clasificación de datos;
2. metadata de sensibilidad;
3. políticas de protección;
4. cifrado a nivel de aplicación;
5. integración con cifrado nativo;
6. hashing;
7. tokenización;
8. masking;
9. redaction;
10. protección de parámetros;
11. protección de resultados;
12. protección de caches;
13. protección de telemetry;
14. protección de debugging;
15. protección de exports;
16. protección de backups;
17. gestión abstracta de claves;
18. rotación;
19. versionado criptográfico;
20. aislamiento tenant-aware;
21. extensibilidad;
22. compatibilidad multi-database;
23. seguridad en runtimes persistentes.

---

# 11. Modelo general

```text
Data Metadata
     ↓
Sensitivity Classification
     ↓
Protection Policy
     ↓
Protection Planner
     ↓
Required Controls
     ├── Encrypt
     ├── Hash
     ├── Tokenize
     ├── Mask
     ├── Redact
     ├── Restrict Cache
     ├── Restrict Export
     ├── Require Audit
     └── Require Secure Transport
```

---

# 12. DataClassification

VoltStack deberá disponer de clasificación semántica.

Propuesta inicial:

```php
enum DataClassification: int
{
    case PUBLIC = 0;
    case INTERNAL = 10;
    case CONFIDENTIAL = 20;
    case RESTRICTED = 30;
    case SECRET = 40;
}
```

---

# 13. PUBLIC

Información diseñada para exposición pública.

Ejemplos:

```text
public product name
public article title
published description
```

Esto no significa que cualquier operación de escritura esté permitida.

---

# 14. INTERNAL

Información destinada al funcionamiento interno.

Ejemplos:

```text
internal identifiers
operational metadata
internal notes
```

---

# 15. CONFIDENTIAL

Información que requiere acceso controlado.

Ejemplos:

```text
email
phone
customer address
commercial information
```

---

# 16. RESTRICTED

Información con riesgo elevado.

Ejemplos:

```text
financial account information
government identifiers
sensitive business records
security-related information
```

---

# 17. SECRET

Máximo nivel para información cuya exposición tendría impacto crítico.

Ejemplos:

```text
cryptographic secrets
private credentials
high-value security material
```

Sin embargo, credenciales que puedan evitar almacenarse directamente deberán manejarse mediante sistemas especializados.

---

# 18. Classification ordering

Podrá definirse:

```text
PUBLIC
<
INTERNAL
<
CONFIDENTIAL
<
RESTRICTED
<
SECRET
```

pero esto no significa que todas las políticas puedan reducirse a un número.

---

# 19. Classification ≠ Policy

Dos campos `RESTRICTED` pueden necesitar controles distintos.

Ejemplo:

```text
national_identifier
→ encryption + redaction

password
→ password hashing + redaction
```

Por tanto:

```text
Classification
+
Data Category
+
Protection Policy
```

---

# 20. SensitiveDataCategory

```php
enum SensitiveDataCategory
{
    case PERSONAL;
    case CONTACT;
    case FINANCIAL;
    case CREDENTIAL;
    case AUTHENTICATION_SECRET;
    case SECURITY_TOKEN;
    case GOVERNMENT_IDENTIFIER;
    case BUSINESS_CONFIDENTIAL;
    case TENANT_SECRET;
    case CUSTOM;
}
```

---

# 21. Category extensibility

Las aplicaciones podrán registrar categorías adicionales mediante identificadores estables.

No mediante FQCN almacenados en DB.

---

# 22. SensitiveFieldMetadata

```php
final readonly class SensitiveFieldMetadata
{
    public function __construct(
        public FieldId $field,
        public DataClassification $classification,
        public array $categories,
        public ProtectionPolicyId $policy,
    ) {}
}
```

---

# 23. Metadata sources

La metadata podrá declararse mediante:

```text
PHP Attributes
Mapping files
Compiled metadata
Configuration
Application extensions
```

---

# 24. Attribute example

```php
final class Customer
{
    #[Sensitive(
        classification: DataClassification::CONFIDENTIAL,
        category: SensitiveDataCategory::CONTACT,
        policy: 'encrypted_contact'
    )]
    private string $email;
}
```

---

# 25. Multiple categories

Un campo podrá pertenecer a varias categorías.

Ejemplo:

```text
employee_bank_account

FINANCIAL
+
PERSONAL
```

---

# 26. ProtectionPolicy

```php
interface SensitiveDataProtectionPolicy
{
    public function id(): ProtectionPolicyId;

    public function requirements(
        SensitiveDataContext $context,
    ): ProtectionRequirements;
}
```

---

# 27. ProtectionRequirements

Podrá contener:

```text
storage protection
transport requirements
cache policy
logging policy
telemetry policy
debug policy
export policy
backup policy
masking policy
audit requirements
```

---

# 28. ProtectionAction

```php
enum ProtectionAction
{
    case ENCRYPT;
    case HASH;
    case TOKENIZE;
    case MASK;
    case REDACT;
    case DENY_CACHE;
    case DENY_EXPORT;
    case REQUIRE_AUDIT;
}
```

---

# 29. Protection actions can compose

Ejemplo:

```text
CONFIDENTIAL email

ENCRYPT_AT_APPLICATION_LAYER
+
REDACT_FROM_TELEMETRY
+
MASK_IN_DEBUG
+
AUDIT_EXPORT
```

---

# 30. SensitiveDataContext

```php
final readonly class SensitiveDataContext
{
    public function __construct(
        public ProtectionPurpose $purpose,
        public ?TenantId $tenant,
        public ?DataPrincipal $principal,
        public DataOperation $operation,
        public ProtectionPolicyGeneration $generation,
    ) {}
}
```

---

# 31. ProtectionPurpose

Ejemplos:

```text
PERSIST
HYDRATE
QUERY
CACHE
LOG
TRACE
DEBUG
EXPORT
BACKUP
REPLICATE
IMPORT
```

---

# 32. Context-specific protection

La misma información puede requerir distintas representaciones.

Ejemplo:

```text
DATABASE:
ciphertext

APPLICATION:
plaintext

DEBUG:
masked

TELEMETRY:
redacted

EXPORT:
policy-dependent
```

---

# 33. Protection lifecycle

```text
Application Value
      ↓
Protection Policy
      ↓
Persistence Protection
      ↓
Database Representation
      ↓
Database Result
      ↓
Unprotection Policy
      ↓
Application Value
```

Observability paths tendrán controles independientes.

---

# 34. Application-level encryption

VoltStack deberá soportar cifrado antes de enviar valores al Driver.

```text
PHP value
   ↓
Encryptor
   ↓
Ciphertext Envelope
   ↓
Parameter Binding
   ↓
Database
```

---

# 35. Benefit

La DB recibe ciphertext en vez del plaintext cuando la arquitectura lo permite.

---

# 36. Limitation

Esto puede impedir operaciones como:

```text
LIKE
ORDER BY plaintext
range query
database aggregation
```

sobre el campo.

---

# 37. Encryption must affect query capabilities

Metadata deberá declarar capacidades efectivas.

Ejemplo:

```text
Encrypted randomized field:
    equality search = false
    range search = false
    ordering = false
    aggregation = false
```

---

# 38. Randomized encryption

Dos valores iguales deberán poder producir ciphertext distinto.

Conceptualmente:

```text
Encrypt("alice@example.com")
→ C1

Encrypt("alice@example.com")
→ C2

C1 != C2
```

---

# 39. Deterministic encryption

Puede permitir igualdad criptográfica:

```text
same plaintext
→ same protected representation
```

bajo el mismo contexto.

Pero expone patrones.

---

# 40. Deterministic encryption policy

No deberá activarse simplemente por conveniencia.

Requerirá policy explícita.

---

# 41. Searchable encrypted fields

VoltStack no deberá prometer búsqueda arbitraria sobre ciphertext.

Podrán utilizarse estrategias específicas como:

```text
blind indexes
deterministic derived tokens
external secure search provider
```

mediante extensiones controladas.

---

# 42. Blind index

Ejemplo conceptual:

```text
email_plaintext
      ↓
normalized
      ↓
keyed derivation
      ↓
email_search_index
```

El valor original permanece cifrado.

---

# 43. Blind index ≠ encryption

Es una representación auxiliar para búsqueda.

---

# 44. Search index leakage

Incluso índices derivados pueden revelar:

```text
equality
frequency
membership
```

según diseño.

Deben tener policy propia.

---

# 45. CiphertextEnvelope

VoltStack no deberá almacenar solamente bytes opacos sin metadata suficiente para evolución.

Propuesta:

```php
final readonly class CiphertextEnvelope
{
    public function __construct(
        public CipherSuiteId $suite,
        public KeyReference $key,
        public KeyVersion $keyVersion,
        public ProtectionVersion $formatVersion,
        public string $nonce,
        public string $ciphertext,
        public string $authenticationTag,
    ) {}
}
```

La representación física puede compactarse.

---

# 46. Envelope ≠ key material

Nunca deberá contener la clave secreta.

---

# 47. Authenticated encryption

El diseño deberá favorecer cifrado autenticado.

Objetivo:

```text
Confidentiality
+
Integrity
```

---

# 48. Associated data

Cuando corresponda, podrán autenticarse contextos como:

```text
field identity
tenant identity
record identity
protection version
```

sin almacenarlos como plaintext secreto.

---

# 49. Context binding

Esto puede impedir mover ciphertext válido entre contextos.

Ejemplo:

```text
Tenant A ciphertext
→ copied to Tenant B
→ authentication failure
```

si tenant forma parte del associated context.

---

# 50. Record identity caveat

Si el ID se genera después del INSERT, no siempre podrá utilizarse en la primera operación de cifrado.

El Protection Planner deberá conocer estas dependencias.

---

# 51. Key abstraction

Database no deberá manejar claves mediante strings dispersos.

Contrato:

```php
interface KeyProvider
{
    public function resolve(
        KeyReference $reference,
        KeyPurpose $purpose,
    ): ResolvedKey;
}
```

---

# 52. KeyReference

Referencia lógica:

```text
customer-data-key
tenant-data-key
export-key
```

No secreto.

---

# 53. Key material

Deberá tener vida mínima necesaria.

---

# 54. KeyProvider implementations

Podrán integrarse:

```text
local development provider
environment-backed provider
secret manager
KMS
HSM-backed provider
cloud key service
custom enterprise provider
```

sin acoplar Database a uno.

---

# 55. Database ≠ KMS

Quantum Database define contratos e integración.

No deberá reinventar un sistema completo de gestión de claves.

---

# 56. Key hierarchy

Podrá utilizarse:

```text
Root/Master Key
      ↓
Key Encryption Key
      ↓
Data Encryption Key
      ↓
Ciphertext
```

según provider/policy.

---

# 57. Envelope encryption

Para datasets grandes podrá favorecerse:

```text
Data Encryption Key
      ↓ encrypts
Data

Key Encryption Key
      ↓ encrypts
Data Encryption Key
```

---

# 58. Tenant key separation

VoltStack deberá soportar:

```text
shared key
per-domain key
per-tenant key
per-classification key
custom hierarchy
```

---

# 59. Per-tenant keys

Beneficios potenciales:

```text
tenant isolation
targeted rotation
crypto-erasure possibilities
reduced blast radius
```

pero aumentan complejidad operacional.

---

# 60. Key policy

La estrategia será explícita.

Nunca asumida por TenantId solamente.

---

# 61. Key versioning

Toda protección rotatable deberá conocer:

```text
KeyVersion
```

---

# 62. Rotation

VoltStack deberá soportar al menos:

```text
write-new/read-old
lazy re-encryption
batch re-encryption
full migration
```

---

# 63. Write-new/read-old

Después de rotar:

```text
writes → key v4

reads:
v1 → allowed temporarily
v2 → allowed temporarily
v3 → allowed temporarily
v4 → current
```

según policy.

---

# 64. Rotation ≠ migration transaction

Rotar millones de registros no deberá requerir una única transacción.

---

# 65. Lazy re-encryption

Al leer:

```text
ciphertext key v2
current key v4
```

podrá programarse re-encryption.

Pero no deberá convertir cada lectura en escritura oculta por default.

---

# 66. Batch re-encryption

Deberá utilizar Large Dataset Processing/Chunk infrastructure.

---

# 67. Re-encryption checkpoints

Necesarios para datasets grandes.

---

# 68. Re-encryption idempotency

El proceso deberá poder identificar:

```text
already current
needs rotation
invalid envelope
unknown version
```

---

# 69. ProtectionVersion

Debe separarse de `KeyVersion`.

```text
ProtectionVersion
≠
KeyVersion
```

---

# 70. Why?

Puede cambiar:

```text
serialization format
cipher suite
associated data strategy
normalization
```

sin cambiar conceptualmente la key identity.

---

# 71. Algorithm agility

La arquitectura deberá permitir reemplazar algoritmos mediante:

```text
CipherSuiteId
ProtectionVersion
```

sin hardcodear un algoritmo en Entity metadata.

---

# 72. No algorithm names in domain model

Evitar:

```php
#[Encrypted('aes-...')]
```

como contrato rígido de dominio.

Preferir:

```php
#[ProtectedBy('customer_contact')]
```

y resolver policy.

---

# 73. Cryptographic implementation boundary

La criptografía concreta deberá residir en componentes especializados.

```text
Database Protection Engine
       ↓
Cryptography Contract
       ↓
Approved Crypto Provider
```

---

# 74. No custom crypto

VoltStack no deberá inventar primitivas criptográficas propias.

---

# 75. Hashing

Campos que no necesiten recuperación podrán utilizar hashing.

Ejemplo:

```text
password
verification secret
lookup token
```

según propósito.

---

# 76. Password hashing

Será responsabilidad principal del sistema Authentication/Hashing del framework.

Database podrá almacenar la representación.

No deberá redefinir la política completa de password hashing.

---

# 77. Generic hashing

Podrá existir para necesidades de Database:

```text
blind indexes
integrity fingerprints
deduplication fingerprints
```

pero con purposes distintos.

---

# 78. Hash purpose separation

Nunca reutilizar una misma derivación sin distinguir propósito.

Conceptualmente:

```text
HMAC(key, "email-index" || value)

≠

HMAC(key, "dedup-index" || value)
```

---

# 79. Tokenization

VoltStack podrá integrar tokenization providers.

Contrato:

```php
interface TokenizationProvider
{
    public function tokenize(
        SensitiveValue $value,
        TokenizationContext $context,
    ): TokenizedValue;

    public function detokenize(
        TokenizedValue $value,
        TokenizationContext $context,
    ): SensitiveValue;
}
```

---

# 80. Tokenization provider boundary

El mapping puede existir fuera de la DB principal.

---

# 81. Token format

Un token no deberá contener accidentalmente el plaintext.

---

# 82. Masking

Masking genera representación limitada.

Ejemplos:

```text
john@example.com
→ j***@example.com

4111111111111111
→ ************1111
```

---

# 83. Masking policy

```php
interface MaskingPolicy
{
    public function mask(
        SensitiveValue $value,
        MaskingContext $context,
    ): MaskedValue;
}
```

---

# 84. Masking is purpose-aware

Puede variar para:

```text
UI
debug
support tooling
export
```

---

# 85. Debug masking

Deberá ser más conservador que UI cuando sea necesario.

---

# 86. Redaction

Redaction elimina el valor de una superficie.

Ejemplo:

```text
password = [REDACTED]
```

---

# 87. Redaction ≠ masking

```text
Mask:
j***@example.com

Redact:
[REDACTED]
```

---

# 88. Redaction policy

Podrá producir:

```text
REMOVE
REPLACE
HASH_FOR_CORRELATION
SHOW_METADATA_ONLY
```

según contexto.

---

# 89. Correlation fingerprint

En observabilidad puede ser útil:

```text
same sensitive value?
```

sin mostrarlo.

Podrá utilizarse fingerprint seguro y purpose-bound cuando policy lo permita.

---

# 90. Telemetry redaction

Integración con:

```text
216_DATABASE_TELEMETRY_ARCHITECTURE
217_DATABASE_QUERY_TELEMETRY_SYSTEM
218_DATABASE_CONNECTION_TELEMETRY_SYSTEM
219_DATABASE_TRANSACTION_TELEMETRY_SYSTEM
220_DATABASE_ORM_TELEMETRY_SYSTEM
221_DATABASE_QUERY_PROFILER_SYSTEM
224_DATABASE_DEBUG_INFORMATION_SYSTEM
225_DATABASE_DEVELOPER_DEBUG_TOOLBAR_INTEGRATION
```

---

# 91. Query parameter protection

Query telemetry nunca deberá asumir que parameters son seguros.

---

# 92. Parameter sensitivity

Cada parameter binding podrá transportar:

```text
Type
Origin
Sensitivity
Redaction Policy
```

---

# 93. SensitiveParameterMetadata

```php
final readonly class SensitiveParameterMetadata
{
    public function __construct(
        public ParameterId $parameter,
        public DataClassification $classification,
        public RedactionPolicyId $redaction,
    ) {}
}
```

---

# 94. Parameter lineage

Cuando sea posible:

```text
Entity.email
      ↓
Query Parameter :email
```

deberá propagar sensibilidad.

---

# 95. Input sensitivity

También podrá existir metadata desde Validation/HTTP/application layer.

---

# 96. Raw Query parameters

Raw queries no deberán perder metadata de sensibilidad.

---

# 97. Unknown sensitivity

En superficies de alto riesgo:

```text
UNKNOWN
≠
PUBLIC
```

---

# 98. Conservative telemetry

Cuando no se conoce la sensibilidad de un parameter, production telemetry deberá favorecer redaction.

---

# 99. SQL statement logging

SQL estructural podrá registrarse.

Valores sensibles no.

Ejemplo:

```text
SELECT * FROM users WHERE email = ?
```

es preferible a:

```text
SELECT * FROM users WHERE email = 'john@example.com'
```

---

# 100. Interpolated SQL

No deberá producirse automáticamente para debugging en producción.

---

# 101. Debug Toolbar

Podrá mostrar:

```text
Parameter 1
type: string
classification: CONFIDENTIAL
value: [REDACTED]
```

---

# 102. Query Profiler

Deberá medir:

```text
duration
rows
query fingerprint
plan
```

sin necesitar plaintext sensible.

---

# 103. Slow Query Detection

Una slow query no justifica exponer sus parámetros.

---

# 104. ORM telemetry

ChangeSets sensibles deberán redacted.

---

# 105. ChangeSet example

En vez de:

```text
email:
old: john@example.com
new: john2@example.com
```

usar:

```text
email:
changed: true
classification: CONFIDENTIAL
values: [REDACTED]
```

---

# 106. Exception protection

Exceptions no deberán incluir valores sensibles por default.

---

# 107. Sensitive exception context

Podrá incluir metadata:

```text
field = customer.email
classification = CONFIDENTIAL
protection = encrypted_contact
```

sin valor.

---

# 108. Stack traces

El stack trace puede conservarse.

Los argumentos sensibles requieren sanitización.

---

# 109. Dumping entities

Debug dump de una Entity deberá respetar metadata sensible.

---

# 110. `__debugInfo()`

Model/Entity integrations podrán utilizar metadata para evitar exposición accidental.

---

# 111. Cache protection

Cada tipo de cache deberá evaluar sensibilidad.

---

# 112. Query Cache

Normalmente almacena estructura, no valores sensibles.

Aun así puede contener literals si existen raw expressions.

---

# 113. Compiled Query Cache

No deberá cachear plaintext request-specific accidentalmente.

---

# 114. Result Cache

Puede contener datos sensibles completos.

Requiere policy explícita.

---

# 115. Entity Cache

También.

---

# 116. CachePolicy

```php
enum SensitiveCachePolicy
{
    case ALLOW;
    case ALLOW_ENCRYPTED;
    case REQUEST_LOCAL_ONLY;
    case DENY_SHARED;
    case DENY;
}
```

---

# 117. Shared cache

Información `RESTRICTED`/`SECRET` no deberá entrar automáticamente a cache distribuido.

---

# 118. Cache encryption

Cuando policy lo permita:

```text
Sensitive Result
      ↓
Cache Protection
      ↓
Encrypted Cache Payload
```

---

# 119. Cache key leakage

No solo importa el valor.

Una key como:

```text
user:john@example.com
```

también filtra información.

---

# 120. Sensitive cache keys

Deberán derivarse mediante representaciones seguras.

---

# 121. Cache tags

También pueden filtrar datos.

---

# 122. Cache invalidation metadata

No deberá contener plaintext sensible innecesario.

---

# 123. ORM integration

El ORM deberá distinguir:

```text
Application Value
Database Protected Value
```

---

# 124. Protection ≠ ORM Type

Un `string` puede estar cifrado o no.

Por tanto:

```text
Type
≠
Protection Policy
```

---

# 125. Protection ≠ Cast

Un cast transforma representación de aplicación.

La protección tiene objetivos de seguridad.

---

# 126. Persistence pipeline

```text
Entity Value
    ↓
ORM Type Conversion
    ↓
Sensitive Protection
    ↓
Database Binding
```

El orden exacto deberá definirse por cada tipo/policy.

---

# 127. Conversion vs encryption

Normalmente deberá existir una representación canónica antes del cifrado.

```text
Domain Value
    ↓
Canonical Persistent Representation
    ↓
Encrypt
```

---

# 128. Hydration pipeline

```text
Database Value
    ↓
Detect Protection Envelope
    ↓
Decrypt/Detokenize
    ↓
Type Conversion
    ↓
Domain Value
```

según policy.

---

# 129. Decryption failure

No deberá convertirse en `null`.

---

# 130. Protection errors

Deberán ser explícitos.

---

# 131. IdentityMap

Decrypted sensitive values pueden permanecer en managed entities.

Esto significa:

```text
Database encryption
≠
memory encryption
```

---

# 132. Memory lifetime

El ORM deberá evitar mantener entidades sensibles más tiempo del necesario cuando la aplicación utilice políticas read-only/detach.

Pero no podrá garantizar borrado físico perfecto de memoria PHP.

---

# 133. SensitiveValue wrapper

Para casos especiales podrá existir:

```php
final readonly class SensitiveValue
{
    // controlled access semantics
}
```

sin pretender que elimine todas las copias de memoria.

---

# 134. Sensitive string copies

PHP puede copiar/reubicar strings.

VoltStack no deberá prometer secure memory zeroization que el runtime no pueda garantizar.

---

# 135. UnitOfWork snapshots

Un problema importante:

```text
Entity current state
+
Snapshot
+
ChangeSet
```

puede multiplicar plaintext sensible en memoria.

---

# 136. Sensitive snapshot policy

Podrán existir estrategias:

```text
FULL_VALUE
FINGERPRINT
ENCRYPTED_SNAPSHOT
NO_SNAPSHOT_IF_SAFE
CUSTOM
```

---

# 137. Change detection

Para algunos campos puede bastar un fingerprint para detectar cambios.

---

# 138. Fingerprint collision/security

La estrategia deberá ser apropiada para el propósito y no usar hashes inseguros casuales.

---

# 139. Persistence planner

Nunca deberá insertar plaintext sensible en debug representation del plan.

---

# 140. Result hydration

Hydrator deberá conocer protección requerida por metadata.

---

# 141. Partial hydration

No cambia la clasificación del campo.

---

# 142. Projections

Una DTO projection que incluye datos sensibles conserva su sensibilidad.

---

# 143. Tuple/scalar results

La protección no depende de que exista Entity.

---

# 144. Scalar sensitivity lineage

Query metadata deberá intentar mantener:

```text
source field
→ scalar result
→ sensitivity
```

---

# 145. Derived expressions

Ejemplo:

```sql
CONCAT(first_name, ' ', last_name)
```

El resultado puede seguir siendo sensible.

---

# 146. Sensitivity propagation

Para una expresión:

```text
f(A, B)
```

la clasificación podrá derivarse mediante reglas.

---

# 147. Conservative propagation

Por default:

```text
classification(result)
≥
max relevant input classification
```

salvo transformación conocida que reduzca exposición.

---

# 148. Aggregation

Agregación puede reducir o mantener sensibilidad dependiendo del contexto.

No deberá desclasificarse automáticamente.

---

# 149. COUNT

Incluso un count puede ser sensible.

---

# 150. Explicit declassification

Solo policies autorizadas podrán marcar una transformación como:

```text
SAFE_DECLASSIFICATION
```

---

# 151. Declassification ≠ masking

Una transformación estadística puede producir información menos sensible.

Eso requiere reglas explícitas.

---

# 152. Query expression metadata

Podrá incluir:

```text
SensitivityDescriptor
```

---

# 153. SensitivityDescriptor

```php
final readonly class SensitivityDescriptor
{
    public function __construct(
        public DataClassification $classification,
        public array $categories,
        public SensitivityConfidence $confidence,
    ) {}
}
```

---

# 154. SensitivityConfidence

```text
KNOWN
DERIVED
PARTIAL
UNKNOWN
```

---

# 155. UNKNOWN

Nunca deberá degradarse automáticamente a PUBLIC.

---

# 156. Bulk Insert

Deberá proteger cada campo antes de binding.

---

# 157. Bulk Update

Misma regla.

---

# 158. Bulk operations and encryption

El engine podrá vectorizar/batchear transformaciones sin cambiar semántica.

---

# 159. Database-side bulk update limitation

Operación:

```sql
SET encrypted_salary = encrypted_salary + 100
```

no es posible semánticamente sobre ciphertext arbitrario.

---

# 160. Protection capability analysis

Bulk Planner deberá detectar operaciones incompatibles con protección.

---

# 161. Import

Import puede recibir plaintext sensible.

Debe evitar que aparezca en:

```text
error reports
failed-row dumps
telemetry
temporary files
```

---

# 162. Import staging

Si se utiliza staging, deberá respetar la misma clasificación.

---

# 163. Import rejection files

Pueden contener información sensible.

Policy explícita.

---

# 164. Export

Export es una frontera de alto riesgo.

---

# 165. Export protection

Puede requerir:

```text
authorization
masking
redaction
encryption
row limit
audit
destination restrictions
expiration
```

---

# 166. Exported plaintext

No deberá considerarse protegido simplemente porque la DB original estaba cifrada.

---

# 167. Export encryption

Podrá utilizar policy específica independiente del storage encryption.

---

# 168. CSV

CSV no proporciona protección criptográfica por sí mismo.

---

# 169. Large Dataset Processing

Procesos de:

```text
rotation
migration
export
archival
reclassification
```

deberán utilizar la infraestructura de large dataset processing.

---

# 170. Chunk boundaries

No deberán exponer plaintext sensible en checkpoints.

---

# 171. Checkpoint protection

Checkpoint deberá almacenar:

```text
safe identifiers
opaque continuation
protected metadata
```

sin datos sensibles innecesarios.

---

# 172. Lazy Collections

Un pipeline lazy puede mantener plaintext durante iteración.

---

# 173. Lazy transformations

Callbacks de aplicación son frontera fuera del control total de Database.

VoltStack deberá documentar que una vez entregado plaintext autorizado a código de aplicación, la aplicación participa en su protección.

---

# 174. Streaming

Streaming sensible requiere especial cuidado porque mantiene:

```text
connection
result
decrypted values
consumer
```

durante periodos prolongados.

---

# 175. Cancellation

Al cancelar deberán liberarse recursos.

No implica garantía de zeroization de memoria PHP.

---

# 176. Pagination

Los cursors no deberán contener plaintext sensible salvo diseño explícitamente protegido.

---

# 177. Cursor boundary values

Si `ORDER BY` usa un campo sensible:

```text
email
```

el cursor podría contener el boundary.

---

# 178. Cursor protection

Debe aplicar:

```text
signing
+
optional encryption
+
sensitivity-aware serialization
```

según metadata.

---

# 179. Signed cursor ≠ confidential cursor

Firma protege integridad/autenticidad.

No confidencialidad.

---

# 180. Encrypted cursor

Será necesario cuando los boundaries no puedan exponerse.

---

# 181. Chunk checkpoints

Misma distinción.

---

# 182. Transactions

Los datos sensibles pueden aparecer en:

```text
transaction context
retry state
deadlock diagnostics
```

Deben redacted.

---

# 183. Retry

Retry no deberá guardar plaintext en logs.

---

# 184. Deadlock diagnostics

Deberán usar query fingerprints/IDs, no sensitive parameter dumps.

---

# 185. Connection Security

Documento 230 garantiza canal/configuración segura.

Sensitive Data Protection podrá exigir:

```text
TLS_REQUIRED
```

para determinadas clasificaciones.

---

# 186. TransportProtectionRequirement

```php
enum TransportProtectionRequirement
{
    case DEFAULT;
    case ENCRYPTED;
    case VERIFIED_ENCRYPTED;
    case CUSTOM;
}
```

---

# 187. Classification-driven transport

Ejemplo:

```text
SECRET
→ VERIFIED_ENCRYPTED
```

según policy.

---

# 188. Connection acquisition

Si endpoint no cumple protección:

```text
ProtectionPolicy
      ↓
ConnectionCapability
      ↓
DENY
```

---

# 189. Replica security

Una réplica que contiene ciphertext sigue formando parte del dominio sensible.

---

# 190. Replica eligibility

Podrá incluir:

```text
encryption requirements
region requirements
security capabilities
```

mediante integración.

---

# 191. Sharding

Protection policy no deberá romperse al mover datos entre shards.

---

# 192. Shard migration

Requiere mantener:

```text
classification
key context
tenant context
protection version
```

---

# 193. Cross-region distribution

Puede requerir policies superiores.

Database deberá proporcionar metadata/capabilities, no hardcodear legislación.

---

# 194. Multitenancy

Tenant deberá participar en protection context cuando corresponda.

---

# 195. Tenant isolation

Un tenant no deberá resolver key references de otro.

---

# 196. Key provider tenant binding

```text
Tenant A
+
KeyReference X
```

no necesariamente equivale a:

```text
Tenant B
+
KeyReference X
```

---

# 197. Tenant deletion

Cuando se use key separation adecuada, destrucción controlada de key material puede participar en crypto-erasure.

---

# 198. Crypto-erasure ≠ guaranteed deletion everywhere

Puede haber:

```text
exports
external copies
logs
backups
plaintext caches
```

si otros sistemas no respetaron policy.

---

# 199. Backup protection

Backups deberán considerarse parte del sensitive data lifecycle.

---

# 200. Backup encryption

La política podrá exigir cifrado independiente.

---

# 201. Backup keys

No deberán almacenarse junto al backup sin separación adecuada.

---

# 202. Backup metadata

También puede contener:

```text
table names
tenant identifiers
schema details
```

que pueden ser sensibles.

---

# 203. Restore

Restore deberá verificar:

```text
protection format support
key availability
key versions
policy compatibility
```

---

# 204. Old backups

Rotar la clave actual no implica que backups antiguos dejen de necesitar claves históricas.

---

# 205. Key retention

Debe coordinarse con backup retention.

---

# 206. Key destruction

No deberá ejecutarse antes de verificar necesidades de restore, salvo que la intención sea crypto-erasure.

---

# 207. Schema metadata

Schema introspection no debería exponer valores, pero nombres de columnas pueden ser sensibles operacionalmente.

---

# 208. Metadata classification

Podrá existir protección separada para metadata operacional.

---

# 209. Migration system

Migraciones pueden manipular campos sensibles.

---

# 210. Migration logs

No deberán imprimir datos migrados.

---

# 211. Data migrations

Una data migration que descifra/re-encripta deberá usar explicit privileged protection context.

---

# 212. Zero-downtime rotation

Cambios criptográficos podrán requerir:

```text
old column/new column
dual-read
dual-write
backfill
cutover
cleanup
```

según estrategia.

---

# 213. Dual write

Debe evitar inconsistencias entre representaciones protegidas.

---

# 214. Protection migration state

Podrán modelarse:

```text
LEGACY
DUAL_READ
DUAL_WRITE
BACKFILLING
CURRENT
RETIRING
```

---

# 215. Query compatibility during migration

Query Planner deberá conocer qué representación está activa.

---

# 216. Raw expressions

No deberán poder obtener automáticamente plaintext de campos protegidos.

---

# 217. Raw SQL

Si el cifrado es application-level, raw SQL recibirá ciphertext.

Esto es intencional.

---

# 218. Trusted decrypt escape hatch

No deberá existir una función SQL genérica añadida automáticamente que exponga claves de aplicación.

---

# 219. Database-native encryption

Puede utilizarse como capability complementaria.

---

# 220. Database-native encryption ≠ application-level encryption

Con DB-native encryption:

```text
Application
→ plaintext
→ DB
→ encrypted storage
```

Con application-level:

```text
Application
→ encryption
→ ciphertext
→ DB
```

La superficie de confianza es distinta.

---

# 221. Storage encryption

Full-disk/TDE/storage encryption protege principalmente datos en reposo.

No evita que una query autorizada por DB vea plaintext.

---

# 222. Defense in depth

Podrán combinarse:

```text
disk encryption
+
database encryption
+
application field encryption
+
access control
+
redaction
```

---

# 223. Capability model

```php
interface SensitiveDataPlatformCapabilities
{
    public function supportsEncryptedTransport(): bool;

    public function supportsNativeEncryption(): bool;

    public function supportsColumnEncryption(): bool;

    public function supportsSecureSessionContext(): bool;
}
```

---

# 224. Capability ≠ configuration

Que una plataforma soporte TLS no significa que la conexión actual lo esté utilizando.

---

# 225. Protection evidence

Se deberá distinguir:

```text
CAPABLE
CONFIGURED
VERIFIED
UNKNOWN
```

---

# 226. UNKNOWN ≠ protected

Para policy estricta:

```text
UNKNOWN
→ reject
```

---

# 227. DataProtectionPlanner

Componente central:

```php
interface DataProtectionPlanner
{
    public function plan(
        SensitiveDataDescriptor $data,
        SensitiveDataContext $context,
    ): DataProtectionPlan;
}
```

---

# 228. DataProtectionPlan

```php
final readonly class DataProtectionPlan
{
    public function __construct(
        public ProtectionPolicyId $policy,
        public ProtectionPolicyGeneration $generation,
        public array $transformations,
        public CacheProtectionDecision $cache,
        public ObservabilityProtectionDecision $observability,
        public ExportProtectionDecision $export,
        public AuditRequirement $audit,
    ) {}
}
```

---

# 229. Plan ≠ execution

Planner decide.

Protection Engine ejecuta transformaciones.

---

# 230. SensitiveDataProtectionEngine

```php
interface SensitiveDataProtectionEngine
{
    public function protect(
        SensitiveValue $value,
        DataProtectionPlan $plan,
    ): ProtectedValue;

    public function reveal(
        ProtectedValue $value,
        DataProtectionPlan $plan,
    ): SensitiveValue;
}
```

---

# 231. Reveal terminology

`reveal()` no significa autorización.

Debe ejecutarse solo después de satisfacer las condiciones relevantes.

---

# 232. ProtectedValue

Puede representar:

```text
ciphertext
token
hash
masked representation
```

según purpose.

---

# 233. One universal ProtectedValue type?

La API pública puede tener interfaz común, pero internamente deberán mantenerse tipos específicos:

```text
EncryptedValue
HashedValue
TokenizedValue
MaskedValue
RedactedValue
```

para evitar confusión.

---

# 234. Protection pipeline

```text
Value
  ↓
Canonicalization
  ↓
Classification
  ↓
Policy Resolution
  ↓
Protection Planning
  ↓
Transformation
  ↓
Protected Representation
  ↓
Binding / Cache / Export / Telemetry
```

---

# 235. Reveal pipeline

```text
Protected Representation
        ↓
Envelope Validation
        ↓
Policy Resolution
        ↓
Key Resolution
        ↓
Integrity Verification
        ↓
Decryption/Detokenization
        ↓
Type Conversion
        ↓
Authorized Application Value
```

---

# 236. Integrity failure

Nunca:

```text
decrypt failure
→ return null
```

Debe fallar explícitamente.

---

# 237. Unknown key

También explícito.

---

# 238. Unsupported protection version

También.

---

# 239. Error hierarchy

```text
SensitiveDataProtectionException
├── DataClassificationException
├── ProtectionPolicyException
├── ProtectionPlanningException
├── ProtectionViolationException
├── SensitiveDataAccessException
├── EncryptionException
│   ├── EncryptionFailureException
│   ├── DecryptionFailureException
│   ├── IntegrityVerificationException
│   └── UnsupportedCipherSuiteException
├── KeyManagementException
│   ├── KeyNotFoundException
│   ├── KeyUnavailableException
│   ├── KeyVersionException
│   └── KeyContextMismatchException
├── TokenizationException
├── HashingException
├── MaskingException
├── RedactionException
├── SensitiveCacheException
├── SensitiveExportException
├── SensitiveTelemetryException
└── ProtectionVersionException
```

---

# 240. Error messages

Nunca deberán contener el valor cuya protección falló.

Correcto:

```text
Unable to decrypt field Customer.email using protection policy encrypted_contact.
```

Incorrecto:

```text
Unable to decrypt value john@example.com.
```

---

# 241. Fail closed

Cuando una policy requiere protección y ésta falla:

```text
ProtectionFailure
→ operation failure
```

No:

```text
ProtectionFailure
→ store plaintext
```

---

# 242. Never fallback to plaintext

Invariante crítica:

> Si un campo requiere cifrado y el KeyProvider está caído, VoltStack no almacenará el valor sin cifrar como fallback.

---

# 243. Import fail-closed

Misma regla.

---

# 244. Cache fail-closed

Si cache exige encrypted payload y encryption falla:

```text
skip cache
```

puede ser válido.

Pero:

```text
store plaintext
```

no.

---

# 245. Telemetry fail-safe

Si redaction falla:

```text
drop sensitive payload
```

antes que registrar plaintext.

---

# 246. Debug fail-safe

Misma regla.

---

# 247. Policy generation

Protection policies tendrán:

```text
ProtectionPolicyGeneration
```

---

# 248. Generation changes

Podrán invalidar:

```text
compiled protection plans
cache representations
debug metadata
export policy decisions
```

---

# 249. Generation ≠ key version

Tres dimensiones separadas:

```text
ProtectionPolicyGeneration
ProtectionVersion
KeyVersion
```

---

# 250. Persistent runtime

Crítico para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 251. Shared immutable state

Puede compartirse:

```text
compiled field sensitivity metadata
immutable protection policies
algorithm registry
```

---

# 252. Request-local state

Debe ser scoped:

```text
tenant
principal
resolved key handles
protection context
temporary plaintext
export context
```

---

# 253. Key caching

Podrá existir solamente bajo provider policy explícita.

---

# 254. Key cache ≠ generic application cache

Debe tener lifecycle y controles especializados.

---

# 255. Worker leakage

Una key resuelta para Tenant A nunca deberá reutilizarse para Tenant B por key incompleta.

---

# 256. Coroutine isolation

OpenSwoole deberá mantener context isolation.

---

# 257. Reset

Al finalizar request/job:

```text
SensitiveDataContext
ResolvedKey references
temporary protection state
```

deberán liberarse del scope.

---

# 258. Memory zeroization

VoltStack realizará best effort donde la abstracción lo permita, pero no prometerá garantías imposibles sobre el memory allocator/runtime de PHP.

---

# 259. Telemetry

Eventos propuestos:

```text
SensitiveDataProtectionApplied
SensitiveDataRevealPerformed
SensitiveDataRedacted
SensitiveDataMaskApplied
SensitiveDataProtectionFailed
KeyResolutionFailed
LegacyProtectionVersionDetected
SensitiveDataRotationRequired
SensitiveExportAttempted
SensitiveCacheDenied
```

---

# 260. Avoid per-value telemetry

No emitir evento por cada campo cifrado en bulk operations por default.

---

# 261. Metrics

Ejemplos bounded:

```text
db.sensitive.protection.operations
db.sensitive.protection.failures
db.sensitive.rotation.pending
db.sensitive.legacy_versions
db.sensitive.cache.denied
db.sensitive.export.denied
```

---

# 262. Labels

Permitidos:

```text
classification
protection_policy
operation
result
```

si cardinalidad es acotada.

Evitar:

```text
field value
principal ID
email
record ID
ciphertext
```

---

# 263. Audit integration

Acciones relevantes:

```text
reveal SECRET
export RESTRICTED
key rotation
protection policy change
privileged decryption
bulk re-encryption
```

podrán exigir audit.

---

# 264. Audit payload

Tampoco deberá contener plaintext sensible.

---

# 265. Query Audit

El siguiente documento definirá esta integración con mayor precisión.

---

# 266. Testing

La suite deberá incluir:

```text
classification tests
policy resolution tests
encryption round-trip tests
integrity failure tests
wrong-key tests
tenant-key isolation tests
rotation tests
legacy version tests
redaction tests
masking tests
cache tests
telemetry tests
debug tests
ORM tests
bulk tests
import/export tests
pagination cursor tests
checkpoint tests
backup/restore integration tests
persistent worker tests
```

---

# 267. Round-trip property

Para encryption válida:

```text
Decrypt(Encrypt(x)) = x
```

bajo mismo contexto compatible.

---

# 268. Wrong-context property

Cuando context binding lo exija:

```text
Decrypt(
    Encrypt(x, ContextA),
    ContextB
)
→ failure
```

---

# 269. Tamper property

Modificar ciphertext/tag:

```text
tampered ciphertext
→ integrity failure
```

no plaintext corrupto silencioso.

---

# 270. Redaction property

Para toda superficie marcada:

```text
SensitiveValue
∉
ObservablePayload
```

---

# 271. Cache property

Si policy es:

```text
DENY_SHARED
```

entonces:

```text
SensitiveValue
∉
SharedCache
```

---

# 272. Telemetry property

```text
SECRET plaintext
∉
logs
traces
metrics
profiler payload
debug toolbar
```

bajo defaults seguros.

---

# 273. Rotation property

Después de una rotación completa:

```text
all active protected values
→ current accepted protection/key version
```

según policy.

---

# 274. Tenant property

```text
Tenant A key context
≠
Tenant B key context
```

cuando la policy exige separación.

---

# 275. Export property

```text
Database Read Allowed
```

no deberá implicar automáticamente:

```text
Plaintext Export Allowed
```

---

# 276. Fuzz testing

Útil para:

```text
malformed envelopes
truncated ciphertext
unknown versions
invalid tags
invalid encodings
oversized payloads
malformed tokens
```

---

# 277. Performance

La protección tendrá costo.

Debe medirse:

```text
encryption throughput
decryption throughput
key resolution latency
bulk overhead
hydration overhead
cache protection overhead
rotation throughput
```

---

# 278. Security > micro-optimization

Nunca eliminar controles criptográficos necesarios por obtener pequeñas mejoras de rendimiento.

---

# 279. Bulk cryptographic processing

Puede utilizar batching seguro cuando el provider lo soporte.

---

# 280. Key provider latency

Deberá evitarse resolver remotamente la misma key para cada campo individual cuando un scoped key handle seguro pueda reutilizarse.

---

# 281. Bounded key reuse

Reutilización deberá respetar:

```text
tenant
key reference
key version
purpose
scope
provider policy
```

---

# 282. Algorithm registry

```php
interface CipherSuiteRegistry
{
    public function resolve(CipherSuiteId $id): CipherSuite;
}
```

---

# 283. Registry freeze

Después de bootstrap deberá ser immutable/frozen en producción.

---

# 284. No FQCN from ciphertext

El envelope no deberá inducir instanciación arbitraria de clases.

---

# 285. Stable identifiers

Utilizar:

```text
CipherSuiteId
ProtectionPolicyId
ProtectionVersion
KeyReference
```

---

# 286. Serialization safety

Nunca usar mecanismos inseguros de deserialización para protected payload metadata.

---

# 287. Envelope size limits

Parsing deberá imponer límites.

---

# 288. Denial-of-service protection

Un envelope malicioso no deberá provocar:

```text
unbounded allocation
unbounded decompression
unbounded key lookup
```

---

# 289. Compression

Comprimir información sensible antes de cifrar puede introducir riesgos dependiendo del contexto.

No deberá habilitarse como optimización genérica automática.

---

# 290. Protected value equality

Dos `EncryptedValue` no deberán compararse como si ciphertext equality implicara necesariamente plaintext equality.

---

# 291. Randomized ciphertext

Especialmente:

```text
C1 != C2
```

puede representar el mismo plaintext.

---

# 292. ORM dirty checking

Por ello dirty checking no deberá comparar ciphertext regenerado para randomized encryption.

---

# 293. Canonical plaintext fingerprint

Podrá utilizarse un fingerprint separado cuando sea seguro y necesario.

---

# 294. Queryability metadata

Cada protection strategy deberá declarar capacidades:

```php
final readonly class ProtectedFieldCapabilities
{
    public function __construct(
        public bool $equalitySearch,
        public bool $rangeSearch,
        public bool $ordering,
        public bool $grouping,
        public bool $aggregation,
        public bool $databaseMutation,
    ) {}
}
```

---

# 295. Query validation

Si una aplicación intenta:

```php
->orderBy('encrypted_secret')
```

y la policy no soporta ordering:

```text
ProtectedFieldOperationNotSupported
```

---

# 296. No silent client-side fallback

VoltStack no deberá hacer:

```text
fetch all rows
→ decrypt
→ sort in PHP
```

automáticamente para emular una query prohibitiva.

---

# 297. Reason

Esto puede:

```text
expose excessive data
destroy performance
break pagination
break memory bounds
change security semantics
```

---

# 298. Explicit specialized processing

Si la aplicación necesita procesamiento client-side, deberá solicitarlo conscientemente bajo policy/resource budgets.

---

# 299. Directory structure

```text
src/Quantum/Database/Security/SensitiveData/
│
├── Contract/
│   ├── SensitiveDataProtectionPolicy.php
│   ├── SensitiveDataProtectionEngine.php
│   ├── DataProtectionPlanner.php
│   ├── KeyProvider.php
│   ├── MaskingPolicy.php
│   ├── RedactionPolicy.php
│   └── TokenizationProvider.php
│
├── Classification/
│   ├── DataClassification.php
│   ├── SensitiveDataCategory.php
│   ├── SensitivityDescriptor.php
│   ├── SensitivityConfidence.php
│   └── SensitiveFieldMetadata.php
│
├── Context/
│   ├── SensitiveDataContext.php
│   ├── ProtectionPurpose.php
│   ├── MaskingContext.php
│   ├── RedactionContext.php
│   └── TokenizationContext.php
│
├── Policy/
│   ├── ProtectionPolicyId.php
│   ├── ProtectionPolicyGeneration.php
│   ├── ProtectionRequirements.php
│   ├── ProtectionAction.php
│   ├── SensitiveCachePolicy.php
│   └── TransportProtectionRequirement.php
│
├── Plan/
│   ├── DataProtectionPlan.php
│   ├── ProtectionTransformation.php
│   ├── CacheProtectionDecision.php
│   ├── ObservabilityProtectionDecision.php
│   └── ExportProtectionDecision.php
│
├── Value/
│   ├── SensitiveValue.php
│   ├── ProtectedValue.php
│   ├── EncryptedValue.php
│   ├── HashedValue.php
│   ├── TokenizedValue.php
│   ├── MaskedValue.php
│   └── RedactedValue.php
│
├── Encryption/
│   ├── CipherSuite.php
│   ├── CipherSuiteId.php
│   ├── CipherSuiteRegistry.php
│   ├── CiphertextEnvelope.php
│   ├── ProtectionVersion.php
│   ├── EncryptionContext.php
│   └── AssociatedDataBuilder.php
│
├── Key/
│   ├── KeyReference.php
│   ├── KeyVersion.php
│   ├── KeyPurpose.php
│   ├── ResolvedKey.php
│   ├── KeyResolver.php
│   └── KeyRotationPlanner.php
│
├── Search/
│   ├── BlindIndex.php
│   ├── BlindIndexPolicy.php
│   ├── SearchToken.php
│   └── ProtectedFieldCapabilities.php
│
├── Masking/
│   ├── MaskingEngine.php
│   ├── MaskingPolicyRegistry.php
│   └── MaskingResult.php
│
├── Redaction/
│   ├── RedactionEngine.php
│   ├── RedactionPolicyRegistry.php
│   └── RedactionResult.php
│
├── Tokenization/
│   ├── TokenizationEngine.php
│   └── TokenizationProviderRegistry.php
│
├── Query/
│   ├── SensitiveParameterMetadata.php
│   ├── SensitiveQueryAnalyzer.php
│   ├── SensitivityPropagationEngine.php
│   └── ProtectedFieldQueryValidator.php
│
├── ORM/
│   ├── SensitiveFieldProtector.php
│   ├── SensitiveFieldRevealer.php
│   ├── SensitiveSnapshotPolicy.php
│   └── SensitiveChangeSetSanitizer.php
│
├── Cache/
│   ├── SensitiveCacheProtector.php
│   ├── SensitiveCacheKeyDeriver.php
│   └── SensitiveCachePolicyResolver.php
│
├── Observability/
│   ├── SensitiveTelemetrySanitizer.php
│   ├── SensitiveDebugSanitizer.php
│   ├── SensitiveExceptionSanitizer.php
│   └── SensitiveProfilerSanitizer.php
│
├── Export/
│   ├── SensitiveExportPolicy.php
│   ├── SensitiveExportProtector.php
│   └── ExportProtectionDecision.php
│
├── Rotation/
│   ├── ProtectionRotationPlan.php
│   ├── ProtectionRotationRunner.php
│   ├── ProtectionRotationCheckpoint.php
│   └── ProtectionMigrationState.php
│
├── Platform/
│   ├── SensitiveDataPlatformCapabilities.php
│   └── NativeEncryptionAdapter.php
│
├── Telemetry/
│   └── SensitiveDataProtectionTelemetry.php
│
├── Testing/
│   ├── SensitiveDataAssertions.php
│   ├── FakeKeyProvider.php
│   ├── FakeTokenizationProvider.php
│   └── SensitiveDataConformanceSuite.php
│
└── Exception/
    ├── SensitiveDataProtectionException.php
    ├── ProtectionPolicyException.php
    ├── ProtectionPlanningException.php
    ├── EncryptionException.php
    ├── DecryptionFailureException.php
    ├── IntegrityVerificationException.php
    ├── KeyNotFoundException.php
    ├── KeyUnavailableException.php
    ├── TokenizationException.php
    ├── MaskingException.php
    ├── RedactionException.php
    ├── SensitiveCacheException.php
    └── SensitiveExportException.php
```

---

# 300. Architectural invariants

## DB-SENSITIVE-001
Classification será distinta de Authorization.

## DB-SENSITIVE-002
Encryption será distinta de Authorization.

## DB-SENSITIVE-003
Masking será distinto de Encryption.

## DB-SENSITIVE-004
Redaction será distinta de Masking.

## DB-SENSITIVE-005
Hashing será distinto de Encryption.

## DB-SENSITIVE-006
Tokenization será distinta de Hashing.

## DB-SENSITIVE-007
Audit será distinto de protection.

## DB-SENSITIVE-008
Authorized no significará safe-to-log.

## DB-SENSITIVE-009
Sensitive metadata será first-class.

## DB-SENSITIVE-010
Classification no será por sí sola la policy completa.

## DB-SENSITIVE-011
Categories serán extensibles mediante IDs estables.

## DB-SENSITIVE-012
UNKNOWN sensitivity no equivaldrá a PUBLIC.

## DB-SENSITIVE-013
Protection policy será purpose-aware.

## DB-SENSITIVE-014
Storage policy podrá diferir de telemetry policy.

## DB-SENSITIVE-015
Application-level encryption ocurrirá antes del Driver cuando corresponda.

## DB-SENSITIVE-016
Encryption strategy declarará query capabilities.

## DB-SENSITIVE-017
Randomized encryption no prometerá equality query.

## DB-SENSITIVE-018
Deterministic encryption requerirá policy explícita.

## DB-SENSITIVE-019
Searchable encryption no será asumida.

## DB-SENSITIVE-020
Blind index será distinto de ciphertext.

## DB-SENSITIVE-021
Blind index tendrá security policy.

## DB-SENSITIVE-022
Ciphertext envelope no contendrá key material.

## DB-SENSITIVE-023
ProtectionVersion será distinta de KeyVersion.

## DB-SENSITIVE-024
ProtectionPolicyGeneration será distinta de ambas.

## DB-SENSITIVE-025
Cipher suite será identificada mediante stable ID.

## DB-SENSITIVE-026
VoltStack no inventará primitivas criptográficas propias.

## DB-SENSITIVE-027
Database no será un KMS.

## DB-SENSITIVE-028
KeyProvider será abstracción.

## DB-SENSITIVE-029
KeyReference no será key material.

## DB-SENSITIVE-030
Key material tendrá scope/lifetime mínimo.

## DB-SENSITIVE-031
Tenant key separation será soportada.

## DB-SENSITIVE-032
Tenant key separation no será obligatoria para toda aplicación.

## DB-SENSITIVE-033
Key rotation soportará read-old/write-new.

## DB-SENSITIVE-034
Rotation no requerirá una transacción global.

## DB-SENSITIVE-035
Lazy re-encryption no escribirá ocultamente por default.

## DB-SENSITIVE-036
Batch rotation reutilizará Large Dataset Processing.

## DB-SENSITIVE-037
Rotation será resumable cuando corresponda.

## DB-SENSITIVE-038
Protection algorithms serán evolutivos.

## DB-SENSITIVE-039
Domain metadata no deberá hardcodear primitive crypto innecesariamente.

## DB-SENSITIVE-040
Purpose separation se aplicará a derived fingerprints.

## DB-SENSITIVE-041
Masking será context-aware.

## DB-SENSITIVE-042
Redaction podrá eliminar completamente valores.

## DB-SENSITIVE-043
Telemetry nunca requerirá plaintext sensible para funcionar.

## DB-SENSITIVE-044
Query parameters transportarán sensitivity metadata cuando sea conocida.

## DB-SENSITIVE-045
Raw Query no perderá automáticamente sensitivity metadata.

## DB-SENSITIVE-046
Unknown parameters serán tratados conservadoramente en production observability.

## DB-SENSITIVE-047
Interpolated SQL con secretos no será default.

## DB-SENSITIVE-048
Profiler no expondrá sensitive parameters.

## DB-SENSITIVE-049
Slow query detection no expondrá sensitive parameters.

## DB-SENSITIVE-050
ORM ChangeSet telemetry será sanitizada.

## DB-SENSITIVE-051
Exceptions no incluirán sensitive values por default.

## DB-SENSITIVE-052
Entity debug output respetará sensitive metadata.

## DB-SENSITIVE-053
Result Cache evaluará sensitivity.

## DB-SENSITIVE-054
Entity Cache evaluará sensitivity.

## DB-SENSITIVE-055
Shared cache no recibirá sensitive plaintext contra policy.

## DB-SENSITIVE-056
Cache keys también serán evaluadas por leakage.

## DB-SENSITIVE-057
Cache tags también serán evaluadas por leakage.

## DB-SENSITIVE-058
ORM Type será distinto de Protection Policy.

## DB-SENSITIVE-059
Cast será distinto de Protection Policy.

## DB-SENSITIVE-060
Canonical persistent representation precederá encryption cuando la policy lo requiera.

## DB-SENSITIVE-061
Decryption failure nunca será convertido silenciosamente en NULL.

## DB-SENSITIVE-062
Integrity failure será explícito.

## DB-SENSITIVE-063
IdentityMap puede contener plaintext y deberá reconocerse como tal.

## DB-SENSITIVE-064
VoltStack no prometerá perfect memory zeroization en PHP.

## DB-SENSITIVE-065
UoW snapshots serán sensitivity-aware.

## DB-SENSITIVE-066
Dirty checking no dependerá de randomized ciphertext equality.

## DB-SENSITIVE-067
Hydration preservará protection semantics.

## DB-SENSITIVE-068
Partial hydration no desclasificará datos.

## DB-SENSITIVE-069
DTO projection no desclasificará datos.

## DB-SENSITIVE-070
Scalar result podrá conservar sensitivity lineage.

## DB-SENSITIVE-071
Derived expressions propagarán sensitivity.

## DB-SENSITIVE-072
Aggregation no desclasificará automáticamente.

## DB-SENSITIVE-073
Explicit declassification requerirá policy.

## DB-SENSITIVE-074
Bulk Insert aplicará protection antes de binding.

## DB-SENSITIVE-075
Bulk Update respetará protected field capabilities.

## DB-SENSITIVE-076
Unsupported encrypted-field operations fallarán explícitamente.

## DB-SENSITIVE-077
No habrá silent client-side query fallback.

## DB-SENSITIVE-078
Import tratará incoming sensitive plaintext como sensible inmediatamente.

## DB-SENSITIVE-079
Import error files serán protegidos.

## DB-SENSITIVE-080
Export será una security boundary.

## DB-SENSITIVE-081
DB read permission no implicará plaintext export permission.

## DB-SENSITIVE-082
Export podrá requerir encryption.

## DB-SENSITIVE-083
CSV no será considerado encrypted format.

## DB-SENSITIVE-084
Large Dataset Processing preservará protection context.

## DB-SENSITIVE-085
Checkpoints no almacenarán sensitive plaintext innecesariamente.

## DB-SENSITIVE-086
Lazy Collection no elimina sensitive memory concerns.

## DB-SENSITIVE-087
Streaming no elimina protection requirements.

## DB-SENSITIVE-088
Signed cursor no será considerado confidential.

## DB-SENSITIVE-089
Sensitive cursor boundaries podrán requerir encryption.

## DB-SENSITIVE-090
Transaction diagnostics serán redacted.

## DB-SENSITIVE-091
Retry diagnostics serán redacted.

## DB-SENSITIVE-092
Deadlock diagnostics serán redacted.

## DB-SENSITIVE-093
Protection policy podrá exigir encrypted transport.

## DB-SENSITIVE-094
Platform capability no significará configured protection.

## DB-SENSITIVE-095
Configured no significará verified.

## DB-SENSITIVE-096
UNKNOWN protection no satisfará strict policy.

## DB-SENSITIVE-097
Replica mantendrá sensitive classification.

## DB-SENSITIVE-098
Shard movement preservará protection metadata.

## DB-SENSITIVE-099
Tenant A no resolverá key context de Tenant B.

## DB-SENSITIVE-100
Crypto-erasure no prometerá eliminar copias externas.

## DB-SENSITIVE-101
Backup será parte del sensitive data lifecycle.

## DB-SENSITIVE-102
Backup encryption podrá ser independiente de field encryption.

## DB-SENSITIVE-103
Backup keys deberán separarse apropiadamente.

## DB-SENSITIVE-104
Restore verificará protection/key compatibility.

## DB-SENSITIVE-105
Key retention considerará backup retention.

## DB-SENSITIVE-106
Migration logs no contendrán migrated sensitive plaintext.

## DB-SENSITIVE-107
Data migration sensible requerirá privileged context.

## DB-SENSITIVE-108
Protection migration podrá ser zero-downtime.

## DB-SENSITIVE-109
Raw SQL no recibirá application encryption keys automáticamente.

## DB-SENSITIVE-110
DB-native encryption será distinta de application encryption.

## DB-SENSITIVE-111
Disk/TDE encryption no sustituirá field protection.

## DB-SENSITIVE-112
Defense in depth será soportada.

## DB-SENSITIVE-113
DataProtectionPlanner no ejecutará crypto.

## DB-SENSITIVE-114
Protection Engine no decidirá authorization.

## DB-SENSITIVE-115
Reveal no significará authorized.

## DB-SENSITIVE-116
ProtectedValue tendrá semántica explícita.

## DB-SENSITIVE-117
Protection failure será fail-closed.

## DB-SENSITIVE-118
Encryption failure nunca caerá a plaintext persistence.

## DB-SENSITIVE-119
Cache encryption failure nunca caerá a plaintext cache.

## DB-SENSITIVE-120
Telemetry sanitization failure favorecerá drop/redaction.

## DB-SENSITIVE-121
Debug sanitization failure favorecerá redaction.

## DB-SENSITIVE-122
Policy generations serán explícitas.

## DB-SENSITIVE-123
Worker-global mutable tenant protection context estará prohibido.

## DB-SENSITIVE-124
Worker-global mutable resolved tenant keys estarán prohibidas sin safe scoped cache.

## DB-SENSITIVE-125
OpenSwoole utilizará coroutine isolation.

## DB-SENSITIVE-126
RoadRunner reseteará sensitive context.

## DB-SENSITIVE-127
FrankenPHP reseteará sensitive context.

## DB-SENSITIVE-128
Resolved keys serán liberadas del scope al terminar.

## DB-SENSITIVE-129
Sensitive telemetry tendrá bounded cardinality.

## DB-SENSITIVE-130
Per-value crypto telemetry estará desactivada por default.

## DB-SENSITIVE-131
Audit payload no incluirá plaintext sensible.

## DB-SENSITIVE-132
Round-trip encryption será probado.

## DB-SENSITIVE-133
Wrong-context decryption será probado.

## DB-SENSITIVE-134
Tamper detection será probado.

## DB-SENSITIVE-135
Redaction será probado en todas las superficies.

## DB-SENSITIVE-136
Cache protection será probado.

## DB-SENSITIVE-137
Tenant key isolation será probado.

## DB-SENSITIVE-138
Rotation será probado con múltiples versiones.

## DB-SENSITIVE-139
Unknown protection versions fallarán explícitamente.

## DB-SENSITIVE-140
Malformed envelopes estarán bounded.

## DB-SENSITIVE-141
Cipher metadata no podrá instanciar arbitrary classes.

## DB-SENSITIVE-142
Protected payload parsing evitará unsafe unserialize.

## DB-SENSITIVE-143
Envelope size estará limitado.

## DB-SENSITIVE-144
Compression sensible no será generic default.

## DB-SENSITIVE-145
Randomized ciphertext equality no representará logical equality.

## DB-SENSITIVE-146
Protected fields declararán queryability.

## DB-SENSITIVE-147
Query Planner validará protected-field capabilities.

## DB-SENSITIVE-148
Pagination preservará sensitive cursor semantics.

## DB-SENSITIVE-149
Chunk Processing preservará sensitive checkpoint semantics.

## DB-SENSITIVE-150
Lazy processing preservará protection context.

## DB-SENSITIVE-151
Export processing preservará classification.

## DB-SENSITIVE-152
Import processing preservará classification.

## DB-SENSITIVE-153
Bulk processing preservará classification.

## DB-SENSITIVE-154
Cache invalidation metadata no expondrá sensitive values.

## DB-SENSITIVE-155
Event payloads serán sanitizados.

## DB-SENSITIVE-156
Security diagnostics serán sanitizados.

## DB-SENSITIVE-157
Connection diagnostics serán sanitizados.

## DB-SENSITIVE-158
Query debug information será sanitizada.

## DB-SENSITIVE-159
Sensitive metadata será preservada a través del Query Model cuando sea necesaria.

## DB-SENSITIVE-160
Sensitivity lineage sobrevivirá transformations conocidas.

## DB-SENSITIVE-161
UNKNOWN lineage no se desclasificará automáticamente.

## DB-SENSITIVE-162
Explicit declassification será auditable cuando policy lo requiera.

## DB-SENSITIVE-163
Protection context será immutable/scoped.

## DB-SENSITIVE-164
Key context será purpose-bound.

## DB-SENSITIVE-165
Protection policy será capability-aware.

## DB-SENSITIVE-166
Security tendrá precedencia sobre micro-optimizations.

## DB-SENSITIVE-167
Sensitive values no aparecerán en generated error messages.

## DB-SENSITIVE-168
Sensitive values no aparecerán en default SQL interpolation.

## DB-SENSITIVE-169
Sensitive values no aparecerán en default profiler output.

## DB-SENSITIVE-170
Sensitive values no aparecerán en default debug toolbar output.

## DB-SENSITIVE-171
Sensitive values no aparecerán en default telemetry payload.

## DB-SENSITIVE-172
Sensitive values no aparecerán en default audit payload.

## DB-SENSITIVE-173
Sensitive values no aparecerán en checkpoints salvo necesidad explícita y protección adecuada.

## DB-SENSITIVE-174
Sensitive values no aparecerán en cache keys como plaintext.

## DB-SENSITIVE-175
Sensitive values no aparecerán en exception context como plaintext.

## DB-SENSITIVE-176
Sensitive Data Protection será independiente del vendor de DB.

## DB-SENSITIVE-177
Vendor-native features serán integradas mediante capabilities.

## DB-SENSITIVE-178
MySQL, MariaDB, PostgreSQL y SQLite podrán tener capacidades diferentes.

## DB-SENSITIVE-179
Application-level protection mantendrá semántica consistente entre plataformas.

## DB-SENSITIVE-180
No existirán rutas silenciosas de degradación de protected a plaintext.

---

# 301. Modelo formal

Sea:

```text
V = valor
C = clasificación
K = categoría
P = política
X = contexto
```

La resolución será:

```text
ResolveProtection(C, K, P, X)
→ ProtectionPlan
```

y la transformación:

```text
Protect(V, ProtectionPlan)
→ Vp
```

donde:

```text
Vp
```

es una representación protegida apropiada para el propósito.

---

# 302. Protección por superficie

Sea:

```text
S ∈ {
    DATABASE,
    CACHE,
    TELEMETRY,
    DEBUG,
    EXPORT,
    BACKUP
}
```

Entonces:

```text
Representation(V, S)
=
Apply(
    Policy(V.classification, V.category, S)
)
```

Por tanto:

```text
Representation(V, DATABASE)
```

no tiene por qué ser igual a:

```text
Representation(V, DEBUG)
```

---

# 303. Sensitivity propagation

Para expresión:

```text
E = f(V1, V2, ..., Vn)
```

la clasificación derivada será:

```text
Classification(E)
=
PropagationRule(
    f,
    Classification(V1...Vn)
)
```

Si no existe una regla segura:

```text
Classification(E)
=
conservative classification
```

y nunca:

```text
UNKNOWN → PUBLIC
```

---

# 304. Protection version identity

Una representación protegida deberá poder resolverse conceptualmente mediante:

```text
ProtectionIdentity
=
(
    PolicyId,
    ProtectionVersion,
    CipherSuiteId,
    KeyReference,
    KeyVersion
)
```

sin incluir key material.

---

# 305. Query capability model

Para campo protegido `F`:

```text
Capabilities(F)
=
{
    equality,
    range,
    ordering,
    grouping,
    aggregation,
    mutation
}
```

Una operación `O` solo será planificable si:

```text
O ∈ Capabilities(F)
```

o existe una estrategia explícita y segura capaz de preservar semántica.

---

# 306. Fail-closed model

Si:

```text
RequiredProtection(V) = ENCRYPT
```

y:

```text
Encrypt(V) = FAILURE
```

entonces:

```text
Persist(V) = DENIED
```

Nunca:

```text
Persist(V) = PLAINTEXT
```

---

# 307. Arquitectura integrada

```text
                         VoltStack Authorization
                                  │
                                  ▼
                         Data Access Security
                                  │
                                  ▼
Application Value ──────► Sensitive Metadata
                                  │
                                  ▼
                       Protection Policy Engine
                                  │
                     ┌────────────┼────────────┐
                     │            │            │
                     ▼            ▼            ▼
                  Storage      Observability   Export
                     │            │            │
                     ▼            ▼            ▼
                 Encrypt       Redact/Mask   Protect/Deny
                     │
                     ▼
               Query Parameter
                     │
                     ▼
                Query Engine
                     │
                     ▼
                  Driver
                     │
                     ▼
                  Database
```

---

# 308. Integración ORM

```text
Entity
  │
  ▼
Entity Metadata
  │
  ├── Type Metadata
  ├── Mapping Metadata
  └── Sensitive Metadata
          │
          ▼
     Protection Plan
          │
          ▼
Canonical Persistent Value
          │
          ▼
Protected Database Value
          │
          ▼
Query Engine
```

Lectura:

```text
Database Result
      │
      ▼
Protected Value
      │
      ▼
Protection Metadata
      │
      ▼
Integrity Verification
      │
      ▼
Reveal
      │
      ▼
Type Conversion
      │
      ▼
Entity Value
```

---

# 309. Integración con seguridad completa

Con los documentos 226–232, la arquitectura comienza a formar:

```text
Database Security
│
├── Security Architecture
│
├── SQL Injection Prevention
│
├── Query Input Security
│
├── Credential Security
│
├── Connection Security
│
├── Data Access Security
│   ├── Principal
│   ├── Resources
│   ├── Rows
│   ├── Fields
│   └── Operations
│
└── Sensitive Data Protection
    ├── Classification
    ├── Encryption
    ├── Key Abstraction
    ├── Tokenization
    ├── Hashing
    ├── Masking
    ├── Redaction
    ├── Cache Protection
    ├── Telemetry Protection
    ├── Export Protection
    ├── Backup Protection
    └── Rotation
```

---

# 310. Secure defaults

VoltStack deberá favorecer:

```text
UNKNOWN sensitivity → conservative handling
UNKNOWN protection → deny when required
encryption failure → fail
redaction failure → hide/drop
sensitive cache uncertainty → bypass cache
sensitive export uncertainty → deny
unknown key → fail
unknown cipher suite → fail
unknown protection version → fail
tenant key mismatch → fail
```

Nunca:

```text
security uncertainty
→ plaintext convenience fallback
```

---

# 311. Regla maestra final

> **VoltStack deberá tratar la sensibilidad como una propiedad semántica del dato y no únicamente como una característica de una columna física. Esa propiedad deberá influir en persistencia, consultas, hidratación, caching, observabilidad, debugging, exports, backups, distribución y operaciones masivas.**

La arquitectura deberá preservar:

```text
Sensitive Data
      ↓
Known Classification
      ↓
Known Protection Policy
      ↓
Controlled Representation
      ↓
Controlled Exposure
```

y evitar:

```text
Sensitive Data
      ↓
Generic string
      ↓
Unknown propagation
      ↓
logs / cache / exports / errors
```

La propiedad fundamental será:

```text
Exposure(V)
⊆
AllowedExposure(
    Classification(V),
    Policy(V),
    Context
)
```

mientras que para almacenamiento:

```text
StoredRepresentation(V)
=
RequiredProtection(V)
```

y nunca:

```text
ProtectionFailure
→ PlaintextFallback
```

---

# 312. Estado del Bloque 22

```text
BLOCK 22 — SECURITY

✓ 226_DATABASE_SECURITY_ARCHITECTURE.md
✓ 227_DATABASE_SQL_INJECTION_PREVENTION_SYSTEM.md
✓ 228_DATABASE_QUERY_INPUT_SECURITY_SYSTEM.md
✓ 229_DATABASE_CREDENTIAL_SECURITY_SYSTEM.md
✓ 230_DATABASE_CONNECTION_SECURITY_SYSTEM.md
✓ 231_DATABASE_DATA_ACCESS_SECURITY_SYSTEM.md
✓ 232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM.md
○ 233_DATABASE_QUERY_AUDIT_SYSTEM.md
○ 234_DATABASE_DATABASE_PERMISSION_MODEL.md
```

---

# 313. Siguiente documento

```text
233_DATABASE_QUERY_AUDIT_SYSTEM.md
```

El siguiente documento deberá definir la infraestructura de auditoría de operaciones Database:

```text
Audit Event
Audit Record
Audit Context
Audit Actor
Audit Subject
Audit Operation
Query Audit
Mutation Audit
Bulk Operation Audit
Sensitive Data Audit
Privileged Access Audit
Transaction Correlation
Query Fingerprints
Resource Identity
Tenant Context
Shard Context
Decision Evidence
Before/After metadata
ChangeSet audit
Audit redaction
Audit integrity
Audit immutability
Audit storage
Audit buffering
Audit batching
Audit failure policy
Audit transaction semantics
beforeCommit / afterCommit
rollback semantics
unknown transaction outcome
distributed operations
retention
archival
search
export
telemetry integration
security integration
persistent runtime safety
```

bajo una separación esencial:

```text
Audit
≠
Application Log
≠
Telemetry
≠
Debug Information
≠
Event Stream
≠
Transaction Log
```

y una regla central:

> **Una auditoría de Database deberá registrar evidencia suficiente para reconstruir quién realizó una operación relevante, sobre qué recurso, bajo qué contexto y cuál fue su resultado conocido, sin convertir el propio registro de auditoría en una nueva fuente de exposición de información sensible.**