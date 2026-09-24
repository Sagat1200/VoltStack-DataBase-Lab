# 226_DATABASE_SECURITY_ARCHITECTURE.md

## 1. Propósito

Este documento define la arquitectura oficial de seguridad de
`VoltStack/Quantum/Database` para **Database V1**.

El componente raíz se denomina conceptualmente:

``` text
DatabaseSecurityArchitecture
```

La seguridad de Database será una propiedad transversal de todo el
pipeline:

``` text
Configuration → Credentials → Connection → Query → Transaction
      → Persistence → Schema/Migrations → Telemetry → Runtime
```

Principio:

``` text
Database security is not one control.

It is the composition of least privilege,
trusted boundaries, safe query construction,
credential protection, data protection,
runtime isolation and auditable operations.
```

## 2. Objetivos

La arquitectura deberá proteger confidencialidad, integridad,
disponibilidad, aislamiento entre tenants, credenciales, queries,
transacciones, schema, evidencia de auditoría y estado de runtimes
persistentes.

No reemplaza firewalls, hardening del sistema operativo, seguridad
nativa del servidor DB, IAM empresarial, gestores de secretos, SIEM ni
autorización de negocio. VoltStack define contratos e integraciones con
esas capas.

## 3. Activos protegidos

``` text
database credentials
connection strings
TLS material
queries and bindings
schema metadata
migration state
transaction state
records and sensitive fields
tenant identifiers
audit evidence
connection/session state
persistence contexts
```

## 4. Threat Model

Database V1 deberá contemplar:

``` text
SQL injection
identifier injection
credential disclosure
privilege escalation
cross-tenant access
unsafe raw SQL
schema tampering
migration abuse
transaction manipulation
connection-state leakage
persistent-worker leakage
sensitive logging
malicious configuration
dependency compromise
query-based denial of service
```

Actores potenciales:

``` text
external attacker
compromised application account
malicious tenant
misconfigured application
compromised dependency
operator error
over-privileged database user
```

También se consideran fallos accidentales: base equivocada, credenciales
de producción usadas en desarrollo, tenant filter ausente, transacción
abierta entre requests o datos sensibles en logs.

## 5. Trust Boundaries

``` text
User Input
    │
    X
Application
    │
    X
VoltStack Database
    │
    X
Database Driver
    │
    X
Database Server
```

Cuando Multitenancy esté instalado:

``` text
Tenant A  X  Tenant B
```

será un boundary crítico.

## 6. DatabaseSecurityContext

Se define:

``` text
DatabaseSecurityContext
```

con contexto conceptual de:

``` text
environment
connection identity
tenant context
principal reference
operation category
security policy
correlation context
```

Será request-scoped, job-scoped o command-scoped. No dependerá de estado
global mutable inseguro.

## 7. Seguridad de conexiones

Toda conexión deberá pasar por `DatabaseConnectionSecurityPolicy`.

Podrá exigir:

``` text
TLS
certificate verification
approved host
approved port
approved database
approved driver
minimum TLS policy
```

Para conexiones remotas de producción se favorecerá transporte cifrado y
verificación de identidad del servidor.

Si TLS es obligatorio y no puede establecerse:

``` text
FAIL CLOSED
```

## 8. Credenciales

Principio:

``` text
Credentials are references to secrets,
not ordinary configuration values.
```

Podrán resolverse desde variables de entorno, secret managers,
container/orchestrator secrets o providers de runtime.

Nunca deberán aparecer sin redacción en logs, excepciones, dumps,
traces, reportes o configuración compilada.

Interfaz conceptual:

``` php
interface DatabaseCredentialProviderInterface
{
    public function resolve(
        DatabaseCredentialReference $reference
    ): DatabaseCredentials;
}
```

El diseño deberá permitir rotación sin acoplar Database a un proveedor
concreto.

## 9. Least Privilege

Cada conexión tendrá únicamente los privilegios necesarios.

Roles conceptuales:

``` text
READ_ONLY
READ_WRITE
MIGRATION
ADMINISTRATIVE
SHADOW_READ
REPORTING
```

Producción podrá usar credenciales distintas para migrations y runtime
normal. La identidad de aplicación no debería requerir `DROP DATABASE`,
`CREATE USER`, `GRANT` o privilegios equivalentes salvo necesidad
explícita.

## 10. DatabaseConnectionIdentity

Cada conexión podrá describirse mediante:

``` text
DatabaseConnectionIdentity
├── logical connection
├── role
├── environment
├── database
├── host fingerprint
└── tenant scope
```

Antes de una operación destructiva deberá poder verificarse:

``` text
expected environment
expected database
expected role
```

## 11. Seguridad del Query Pipeline

Regla:

``` text
Values are bound.
Identifiers are validated.
SQL structure is compiled.
```

Ruta preferida:

``` text
Application Intent
      ↓
Query Builder / ORM
      ↓
Query AST
      ↓
SQL Compiler
      ↓
Bound SQL
```

La separación entre estructura y valores es un principio de seguridad
central.

## 12. Parameter Binding

Valores dinámicos utilizarán parámetros cuando la plataforma lo permita.

Correcto:

``` sql
SELECT *
FROM users
WHERE email = ?
```

No recomendado:

``` php
$sql = "SELECT * FROM users WHERE email = '$email'";
```

Bindings podrán transportar metadata de tipo y sensibilidad sin exponer
el valor.

## 13. Identifier Safety

Los parámetros SQL no suelen sustituir nombres de tablas, columnas u
`ORDER BY`.

Identificadores dinámicos deberán provenir de:

``` text
validated schema metadata
known enum
compiler-generated identifier
explicit allowlist
```

Nunca se considerará seguro interpolar directamente un identificador
proveniente del request.

## 14. Raw SQL

VoltStack continuará soportando SQL nativo, pero la ruta deberá ser
explícita y auditable:

``` text
NativeSql
RawQuery
UnsafeSql
```

según API definitiva.

Podrá exigirse:

``` text
bound parameters
declared operation type
declared result mapping
security review metadata
```

Las APIs verdaderamente inseguras deberán ser explícitas, localizables y
desaconsejadas.

## 15. Defensa contra SQL Injection

Será defense-in-depth:

``` text
AST/compiler
+
parameter binding
+
identifier validation
+
raw SQL policy
+
least privilege
+
security tests
```

Prepared statements no solucionan interpolación insegura de estructura
SQL.

## 16. DatabaseQuerySecurityClassifier

Clasificará operaciones como:

``` text
READ
WRITE
DDL
ADMIN
UNKNOWN
```

Se utilizará para read-only connections, shadow execution, migrations,
policies y telemetry.

`UNKNOWN` nunca se asumirá `READ`.

## 17. DatabaseQuerySecurityPolicy

Podrá evaluar:

``` text
connection role
environment
runtime mode
tenant context
operation category
```

Ejemplo:

``` text
SHADOW_READ + UPDATE → BLOCK
```

Read-only podrá reforzarse con tres capas:

``` text
application policy
+
read-only DB credentials
+
DB read-only transaction/session
```

cuando el motor lo soporte.

## 18. ORM y autorización

``` text
ORM security != business authorization
```

El ORM puede ayudar a construir SQL seguro, pero no decide si un
principal puede leer una entidad.

Separación:

``` text
Authentication
      ↓
Authorization
      ↓
Persistence Intent
      ↓
Database Security
```

Database podrá consumir contexto autorizado sin duplicar el sistema
completo de Authorization.

## 19. Datos sensibles

Campos podrán etiquetarse conceptualmente:

``` text
SENSITIVE
SECRET
PII
ENCRYPTED
AUDIT_RESTRICTED
```

La metadata podrá controlar logging, telemetry, debug, serialization y
migration reports.

## 20. Cifrado

Se distinguirán:

``` text
encryption in transit
encryption at rest
application-level field encryption
```

TLS cubre tránsito. El cifrado at-rest normalmente pertenece a
DB/storage/cloud. Field encryption podrá integrarse con el módulo
Encryption de VoltStack.

Las keys no deberán almacenarse de forma que elimine la separación
respecto al ciphertext. El modelo permitirá key versioning y rotation.

Passwords usarán Hashing, no cifrado reversible.

## 21. Data Minimization

``` text
Do not retrieve sensitive data
that the operation does not need.
```

Se favorecerán projections específicas sobre `SELECT *` cuando el
dominio no necesite todos los campos.

## 22. Multitenancy

Multitenancy permanece como paquete oficial opcional.

Cuando esté instalado:

``` text
Tenant Context
      ↓
Database Tenant Resolver
      ↓
Connection / Schema / Query Scope
```

Podrá soportar:

``` text
shared DB/shared schema
shared DB/separate schema
database per tenant
```

Invariante:

``` text
A request for Tenant A must never observe Tenant B data
unless an explicit privileged operation allows it.
```

Defensas:

``` text
tenant context
query scoping
connection selection
DB-native controls
validation
runtime reset
```

Una fuga cross-tenant será un fallo crítico.

## 23. Database-native Isolation

Podrán aprovecharse capacidades como PostgreSQL Row Level Security,
schemas separados, credenciales separadas o database-per-tenant.

El core será capability-driven y no hará de una capacidad
vendor-specific un requisito universal.

## 24. Seguridad transaccional

Toda transacción deberá poder asociarse con:

``` text
owner
connection
tenant
scope
```

Cross-tenant transactions se bloquearán por defecto salvo operación
administrativa explícita.

Una transacción abierta al terminar request/job deberá cerrarse
intencionalmente o revertirse.

## 25. Connection Session State

Una conexión reutilizada puede conservar:

``` text
session variables
timezone
search_path
role
temporary tables
isolation level
```

Antes de reutilización deberá regresar a un estado seguro conocido.

## 26. FrankenPHP

FrankenPHP es el runtime servidor predeterminado de VoltStack.

Riesgo:

``` text
Request A
   ↓
Persistent Process State
   ↓
Request B
```

Contrato de reset:

``` text
tenant context reset
security context reset
transaction cleanup
persistence context reset
identity map clear
connection session reset
temporary policy reset
```

No deberá guardarse contexto mutable de seguridad en globals o static
state sin un lifecycle seguro.

Pruebas mínimas:

``` text
Request A → Tenant A
Request B → Tenant B
Request C → No Tenant
```

verificando cero leakage.

## 27. RoadRunner y OpenSwoole

Los paquetes oficiales opcionales deberán cumplir los mismos
invariantes:

``` text
request isolation
context reset
transaction cleanup
connection cleanup
concurrency-safe context
```

## 28. Concurrency

En ejecución concurrente:

``` text
Coroutine A  X  Coroutine B
```

El tenant, transaction, connection y security context deberán permanecer
aislados.

## 29. Schema Security

DDL será privilegiado.

Producción podrá requerir:

``` text
migration role
environment confirmation
schema fingerprint
migration lock
audit event
```

## 30. Migration Security

Migration files se tratarán como código privilegiado porque pueden
ejecutar DDL, raw SQL y data transformations.

`DatabaseMigrationSecurityGuard` podrá validar:

``` text
environment
database identity
connection role
destructive operations
migration fingerprint
```

Operaciones como `DROP TABLE`, `DROP COLUMN`, `TRUNCATE` o mass delete
podrán requerir policy/confirmación explícita.

Modificar una migration ya aplicada deberá ser detectable.

## 31. CLI Security

Los comandos sensibles verificarán:

``` text
environment
connection
database
operation
```

Ejemplo:

``` text
Target:
production / billing-primary

Operation:
DROP COLUMN

Explicit confirmation required.
```

CI no interactivo podrá usar una policy explícita, nunca una
confirmación implícita.

## 32. Configuration Security

Database config podrá contener hosts, database names, credential
references, TLS y pool policy.

Producción podrá rechazar:

``` text
required TLS disabled
prohibited plaintext credential source
unknown driver
unsafe debug configuration
```

Los entornos development/test/staging/production deberán poder tener
credenciales, DBs y policies separadas.

## 33. DatabaseIdentityGuard

Antes de operaciones críticas:

``` text
DatabaseIdentityGuard
```

verificará que la base real corresponde a la esperada usando, según
capacidades:

``` text
host
database name
server metadata/fingerprint
environment marker
```

Esto reduce el riesgo de ejecutar tooling destructivo sobre la base
equivocada.

## 34. Logging Security

Nunca registrar bindings sensibles por defecto.

Ejemplo:

``` text
query:
SELECT * FROM users WHERE email = ?

bindings:
[REDACTED:SENSITIVE]
```

Raw SQL también deberá sanitizarse porque puede contener literals.

## 35. Telemetry Security

No utilizar como labels:

``` text
email
token
raw query parameter
secret
unbounded record identifiers
```

Tracing evitará full sensitive SQL y PII.

Se preferirá:

``` text
normalized query fingerprint
```

## 36. Error Security

Separación:

``` text
Internal Diagnostic
       │
       ├── secure log
       └── normalized application exception
```

Detalles como host, driver, SQL state o query no deberán exponerse
automáticamente al cliente.

Excepciones conceptuales:

``` text
DatabaseSecurityException
DatabaseSecurityPolicyViolationException
DatabaseCredentialException
DatabaseTlsException
DatabaseUnsafeQueryException
DatabaseTenantIsolationException
DatabaseIdentityMismatchException
DatabaseMigrationSecurityException
DatabaseSecurityContextException
```

## 37. Auditoría

Eventos auditables:

``` text
migration applied
destructive schema operation
security policy violation
unsafe SQL use
tenant isolation failure
credential resolution failure
database identity mismatch
```

Database audit técnico y domain audit no son equivalentes.

VoltStack podrá coexistir con auditoría nativa de PostgreSQL,
MySQL/MariaDB, SQL Server y proveedores cloud.

## 38. Availability Security

Seguridad también incluye resistencia a abuso de recursos.

Controles posibles:

``` text
query timeout
statement timeout
connection limits
pool limits
result limits
bulk-operation guards
```

Una query sin `LIMIT` no es automáticamente insegura; la policy deberá
ser contextual.

## 39. Bulk Operations

Operaciones masivas podrán ofrecer:

``` text
affected-row expectations
transaction requirement
dry-run/count preview
```

especialmente en tooling administrativo.

## 40. Stored Procedures y Triggers

Procedures pueden ocultar writes, dynamic SQL y side effects. Se
clasificarán como:

``` text
READ
WRITE
ADMIN
UNKNOWN
```

cuando sea posible.

`UNKNOWN` no será read-only.

Triggers también deberán considerarse durante schema introspection y
migration analysis.

## 41. Driver Security

Cada driver respetará contratos para:

``` text
parameter binding
TLS configuration
error normalization
credential handling
connection reset
```

`DatabaseDriverSecurityCapabilities` declarará capacidades reales.

Principio:

``` text
Do not claim a security control
that the driver or database cannot enforce.
```

## 42. Policy Registry

``` text
DatabaseSecurityPolicyRegistry
```

podrá agrupar policies por environment, connection, role, operation y
runtime.

Resultados:

``` text
ALLOW
DENY
REQUIRE_REVIEW
UNSUPPORTED
```

Para operaciones críticas desconocidas se favorecerá `DENY` o
`REQUIRE_REVIEW`.

## 43. Component Architecture

``` text
DatabaseSecurityArchitecture
│
├── DatabaseSecurityManager
├── DatabaseSecurityContext
├── DatabaseSecurityPolicyRegistry
├── DatabaseConnectionSecurityPolicy
├── DatabaseQuerySecurityPolicy
├── DatabaseQuerySecurityClassifier
├── DatabaseCredentialProviderRegistry
├── DatabaseIdentityGuard
├── DatabaseTenantSecurityBridge
├── DatabaseSensitiveDataClassifier
├── DatabaseTelemetrySanitizer
├── DatabaseErrorSanitizer
├── DatabaseMigrationSecurityGuard
├── DatabaseRuntimeIsolationGuard
├── DatabaseSecurityAuditEmitter
└── DatabaseSecurityDiagnostics
```

Interfaz conceptual:

``` php
interface DatabaseSecurityManagerInterface
{
    public function authorizeConnection(
        DatabaseConnectionIntent $intent,
        DatabaseSecurityContext $context
    ): DatabaseSecurityDecision;

    public function authorizeQuery(
        DatabaseQueryIntent $intent,
        DatabaseSecurityContext $context
    ): DatabaseSecurityDecision;
}
```

No reemplaza Authorization de negocio.

## 44. Connection Security Pipeline

``` text
Connection Request
      ↓
Resolve Configuration
      ↓
Resolve Credential Reference
      ↓
Security Policy
      ↓
Identity Guard
      ↓
TLS Policy
      ↓
Driver Connect
      ↓
Session Hardening
      ↓
Ready Connection
```

## 45. Query Security Pipeline

``` text
Query Intent
    ↓
Security Context
    ↓
Query Classification
    ↓
Policy Evaluation
    ↓
AST / SQL Compilation
    ↓
Identifier Validation
    ↓
Parameter Binding
    ↓
Execution
    ↓
Safe Telemetry
```

## 46. Security Diagnostics

Namespaces conceptuales:

``` text
VSDB-SEC-CONNECTION-*
VSDB-SEC-CREDENTIAL-*
VSDB-SEC-QUERY-*
VSDB-SEC-TENANT-*
VSDB-SEC-MIGRATION-*
VSDB-SEC-RUNTIME-*
VSDB-SEC-TELEMETRY-*
```

Severidad:

``` text
INFO
WARNING
ERROR
CRITICAL
```

## 47. Security Events

``` text
DatabaseSecurityPolicyViolated
DatabaseUnsafeQueryDetected
DatabaseCredentialResolutionFailed
DatabaseIdentityMismatchDetected
DatabaseTenantIsolationViolationDetected
DatabaseMigrationSecurityBlocked
DatabaseRuntimeIsolationViolationDetected
```

Payloads deberán ser diagnósticos pero no contener secretos.

## 48. Security Metrics

``` text
database.security.policy_denials
database.security.unsafe_query_attempts
database.security.credential_failures
database.security.identity_mismatches
database.security.tenant_isolation_failures
database.security.migration_blocks
database.security.runtime_isolation_failures
```

No usar PII, tokens o query text como labels.

## 49. Cache Security

Metadata cache deberá usar formatos controlados, namespaces, versioning
y fingerprints.

Si un cache de resultados depende de tenant, el tenant deberá formar
parte segura de la clave o del aislamiento físico/lógico.

Cross-tenant cache leakage será crítico.

## 50. Supply Chain

La superficie incluye:

``` text
PHP extensions
database drivers
Composer packages
runtime integrations
```

Se favorecerán:

``` text
minimal dependencies
locked versions
dependency auditing
trusted package sources
```

## 51. Backup Security

Backups pueden contener la base completa.

Infraestructura deberá controlar:

``` text
encryption
access
retention
restore authorization
```

Database/Recovery deberá preservar estas propiedades y nunca exponer
credenciales o datos innecesarios en artifacts.

## 52. Failure Behavior

Para:

``` text
credential policy
tenant isolation
database identity
required TLS
critical query policy
```

el comportamiento será conservador:

``` text
FAIL CLOSED
```

Principio:

``` text
Security-critical uncertainty
must not silently become permission.
```

## 53. Testing

Unit tests:

``` text
policy evaluation
query classification
identifier validation
redaction
credential references
tenant context
environment guard
```

Integration tests:

``` text
TLS
driver binding
read-only credentials
migration role
tenant isolation
connection reset
```

Injection tests deberán cubrir values, identifiers, ORDER BY, raw SQL,
JSON paths, LIKE patterns y procedure calls según APIs disponibles.

## 54. Persistent Worker Tests

Obligatorios:

``` text
Tenant A → Tenant B
Transaction A → Request B
Role A → Role B
Connection Session A → B
```

Cualquier leakage será release blocker para persistent runtimes.

## 55. Static Analysis y CI

Reglas podrán detectar:

``` text
unsafe raw SQL
string-concatenated SQL
forbidden APIs
direct credential access
```

CI podrá bloquear:

``` text
critical tenant isolation failure
known credential leak
unsafe migration
query injection regression
persistent-worker context leak
```

## 56. Integración con VoltStack

Database Security se integra con:

``` text
Quantum/Config
Quantum/Container
Encryption
Hashing
Authentication
Authorization
Telemetry
Multitenancy (optional)
```

Responsabilidades:

``` text
Application:
input validation
business authorization
domain invariants
authentication

Database:
safe query execution
connection security
credential handling
technical policy
tenant persistence isolation
runtime isolation
safe diagnostics

DB Server / Infrastructure:
server privileges
patching
storage encryption
native auditing
network controls
backup protection
```

## 57. V1 Scope

Incluido:

``` text
security context
security policies
credential-provider abstraction
TLS policy
least-privilege connection roles
safe query construction
parameter binding
identifier safety
raw SQL policy
query classification
read-only policy
tenant-security integration
sensitive-data metadata
safe logging/telemetry
migration security
database identity guard
persistent-worker isolation
security diagnostics
security testing
extension contracts
```

Fuera del core Database V1:

``` text
enterprise secret vault implementation
network firewall manager
database server patch manager
SIEM platform
full IAM system
full DLP platform
autonomous threat-response platform
```

## 58. Decisiones arquitectónicas

1.  Security será transversal y no un wrapper opcional final.
2.  Los valores SQL se parametrizarán por defecto.
3.  Identificadores dinámicos requerirán validación explícita.
4.  Raw SQL seguirá soportado, pero será visible y auditable.
5.  Database Security no reemplazará Authorization de negocio.
6.  Credenciales serán secretos/referencias, no strings ordinarios.
7.  Least privilege será parte del modelo de conexiones.
8.  Migrations podrán usar credenciales distintas al runtime normal.
9.  TLS obligatorio por policy fallará cerrado.
10. Multitenancy será opcional, pero su aislamiento será crítico cuando
    esté instalado.
11. Cross-tenant leakage será un fallo crítico.
12. FrankenPHP tendrá contratos explícitos de reset por request.
13. RoadRunner/OpenSwoole cumplirán los mismos invariantes mediante
    perfiles opcionales.
14. Query logging y telemetry redaccionarán datos sensibles.
15. DDL y operaciones destructivas serán privilegiadas.
16. La identidad de la base podrá verificarse antes de operaciones
    peligrosas.
17. `UNKNOWN` no equivaldrá a seguro.
18. Controles platform-specific declararán capacidades reales.

## 59. Security Invariants

``` text
Untrusted values never become SQL structure implicitly.

Secrets never become ordinary diagnostics.

Tenant context never leaks across execution boundaries.

A persistent worker begins each unit of work
with a clean database security context.

A destructive operation knows which environment
and database it targets.

Observability explains database behavior
without becoming a data-exfiltration channel.
```

## 60. Completion Criteria

Database Security V1 estará completo cuando pueda:

``` text
protect credentials
enforce connection security policy
support TLS verification
model least-privilege roles
parameterize values
validate identifiers
identify unsafe raw SQL paths
classify query intent
block writes on read-only roles
integrate tenant isolation
sanitize logs/errors/traces
guard production migrations
verify database identity
reset security context in FrankenPHP
support equivalent optional runtimes
emit security diagnostics
pass injection/isolation tests
```

## 61. Arquitectura End-to-End

``` text
Application
    ↓
Authentication / Authorization
    ↓
Database Security Context
    ↓
Persistence / Query Intent
    ↓
Database Security Policy
    ├── Tenant Guard
    ├── Query Classifier
    ├── Role Policy
    └── Environment Guard
    ↓
Query AST / Compiler
    ├── Identifier Validation
    └── Parameter Binding
    ↓
Secure Connection
    ├── Credential Provider
    ├── TLS
    └── Least Privilege
    ↓
Database
    ↓
Sanitized Telemetry / Audit
```

## 62. Principio final

``` text
Treat every database boundary as a security boundary.

Bind values.

Validate structure.

Minimize privilege.

Protect credentials.

Isolate tenants.

Reset persistent state.

Audit privileged operations.

Fail closed when security-critical context is unknown.
```

## 63. Conclusión

`DATABASE_SECURITY_ARCHITECTURE` establece la seguridad transversal de
`VoltStack/Quantum/Database`.

La seguridad no queda concentrada únicamente en
`DatabaseSecurityManager`. Se distribuye deliberadamente a través de:

``` text
Configuration
Credentials
Connections
Queries
ORM
Transactions
Tenancy
Schema
Migrations
Runtime
Telemetry
Testing
```

El objetivo no es afirmar que Database es seguro únicamente porque
utiliza prepared statements. Cada frontera sensible deberá tener:

``` text
explicit context
explicit policy
explicit ownership
explicit capabilities
explicit diagnostics
```

Regla arquitectónica definitiva:

``` text
Secure the connection.

Secure the query.

Secure the context.

Secure the runtime.

Secure the evidence.

Never let convenience silently cross a security boundary.
```

------------------------------------------------------------------------

**Documento:** `226_DATABASE_SECURITY_ARCHITECTURE.md`\
**Proyecto:** VoltStack Framework\
**Módulo:** `VoltStack/Quantum/Database`\
**Versión objetivo:** Database V1\
**Estado:** Architectural Specification
