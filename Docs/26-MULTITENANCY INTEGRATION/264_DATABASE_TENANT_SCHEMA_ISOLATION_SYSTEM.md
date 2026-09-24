# 264_DATABASE_TENANT_SCHEMA_ISOLATION_SYSTEM.md

# VoltStack Quantum Database
## Tenant Schema Isolation System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Integración:** `VoltStack/Quantum/Multitenancy`  
**Documento:** 264 — Tenant Schema Isolation System  
**Bloque:** 26 — Multitenancy Integration  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `263_DATABASE_TENANT_DATABASE_ISOLATION_SYSTEM.md`  
**Siguiente documento:** `265_DATABASE_TENANT_QUERY_CONTEXT_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura mediante la cual VoltStack podrá aislar tenants utilizando schemas independientes dentro de una misma base de datos física o lógica.

El modelo principal será:

```text
Database
│
├── tenant_a
│   ├── customers
│   ├── orders
│   └── invoices
│
├── tenant_b
│   ├── customers
│   ├── orders
│   └── invoices
│
└── tenant_c
    ├── customers
    ├── orders
    └── invoices
```

La regla central será:

> **Un Tenant Schema en VoltStack será una identidad estructural confiable derivada del TenantContext y del TenantPlacement; nunca será simplemente una cadena suministrada por el usuario que se concatena a nombres de tablas o sentencias SQL.**

Por tanto:

```text
TenantSchema
≠
Raw String
```

y:

```text
Schema Isolation
≠
SET search_path alone
```

---

# 2. Objetivos

El sistema deberá soportar:

1. schema-per-tenant;
2. resolución de schema;
3. identidad lógica de schema;
4. nombre físico de schema;
5. validación de identifiers;
6. schema registry;
7. schema generation;
8. schema version;
9. schema metadata;
10. schema introspection;
11. qualified identifiers;
12. session-based schema switching;
13. search paths;
14. connection reuse;
15. session reset;
16. ORM integration;
17. Query Engine integration;
18. Compiler integration;
19. migrations;
20. transactions;
21. cache;
22. relationships;
23. backup/restore;
24. tenant mobility;
25. persistent runtimes;
26. security;
27. telemetry;
28. diagnostics;
29. testing.

---

# 3. No objetivos

Este sistema no será responsable de:

```text
Tenant Authentication
Tenant Authorization
SaaS Billing
Database Provisioning
Cluster Provisioning
General Schema Builder
General Migration Engine
SQL Execution
Connection Pool Implementation
```

Utilizará los sistemas existentes de VoltStack.

---

# 4. Contexto arquitectónico

La arquitectura se apoya en:

```text
261_DATABASE_MULTITENANCY_INTEGRATION_ARCHITECTURE
        ↓
262_DATABASE_TENANT_CONNECTION_RESOLUTION_SYSTEM
        ↓
263_DATABASE_TENANT_DATABASE_ISOLATION_SYSTEM
        ↓
264_DATABASE_TENANT_SCHEMA_ISOLATION_SYSTEM
```

Posteriormente:

```text
265_DATABASE_TENANT_QUERY_CONTEXT_SYSTEM
```

transportará la identidad tenant/schema hacia las queries.

---

# 5. Schema-per-tenant

El modelo permite:

```text
One Database
      │
      ├── Schema Tenant A
      ├── Schema Tenant B
      ├── Schema Tenant C
      └── Schema Tenant D
```

compartiendo:

```text
Database Server
Database Process
Connection Infrastructure
Potentially Credentials
```

mientras se separan los objetos de cada tenant.

---

# 6. Beneficios

Entre sus ventajas potenciales:

```text
stronger structural separation than row-only tenancy
independent schema objects
tenant-specific backup possibilities
reduced tenant predicate dependence
easier tenant migration boundaries
```

---

# 7. Costos

También introduce:

```text
schema proliferation
migration fan-out
metadata growth
catalog pressure
connection session complexity
version skew
operational complexity
```

---

# 8. Schema isolation ≠ physical database isolation

Dos schemas pueden seguir compartiendo:

```text
CPU
RAM
I/O
Database Process
Failure Domain
Credentials
Connection Pool
```

Por tanto:

```text
Schema Isolation
≠
Dedicated Database
```

---

# 9. TenantSchemaId

VoltStack deberá distinguir identidad lógica y nombre físico.

```php
final readonly class TenantSchemaId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Ejemplo:

```text
TenantSchemaId:
schema:tenant:acme
```

---

# 10. PhysicalSchemaName

Separado:

```php
final readonly class PhysicalSchemaName
{
    public function __construct(
        public string $value,
    ) {}
}
```

Ejemplo:

```text
t_01HZX7M3K9
```

---

# 11. Logical ≠ physical

Nunca asumir:

```text
TenantId = SchemaName
```

ni:

```text
TenantSlug = SchemaName
```

---

# 12. Razón

Un tenant:

```text
acme
```

puede renombrarse comercialmente sin que su schema deba cambiar.

También puede migrar:

```text
schema_old
→
schema_new
```

manteniendo el mismo TenantId.

---

# 13. TenantSchemaDescriptor

Objeto central:

```php
final readonly class TenantSchemaDescriptor
{
    public function __construct(
        public TenantId $tenant,
        public TenantSchemaId $schemaId,
        public PhysicalSchemaName $physicalName,
        public LogicalDatabaseId $database,
        public TenantSchemaGeneration $generation,
        public TenantSchemaVersion $version,
        public TenantSchemaStatus $status,
    ) {}
}
```

---

# 14. Schema status

Estados conceptuales:

```text
PROVISIONING
ACTIVE
MIGRATING
READ_ONLY
SUSPENDED
ARCHIVED
DELETING
DELETED
FAILED
UNKNOWN
```

---

# 15. UNKNOWN ≠ ACTIVE

Una operación tenant normal no deberá asumir:

```text
UNKNOWN
→
ACTIVE
```

---

# 16. Schema generation

Representará cambios de identidad/placement relevantes.

```text
Generation 10
schema_a
```

podría pasar a:

```text
Generation 11
schema_a_v2
```

---

# 17. Schema version

No debe confundirse con generation.

```text
SchemaGeneration
≠
SchemaVersion
```

---

# 18. Generation

Responde:

> ¿Cuál es la instancia/identidad estructural actual del schema del tenant?

---

# 19. Version

Responde:

> ¿Qué versión estructural/migratoria tiene actualmente ese schema?

---

# 20. Ejemplo

```text
Tenant A

Schema Generation: 42
Schema Version:    118
```

Una migration:

```text
118 → 119
```

puede no cambiar generation.

---

# 21. Schema registry

Se propone:

```php
interface TenantSchemaRegistry
{
    public function resolve(
        TenantId $tenant
    ): TenantSchemaDescriptor;
}
```

---

# 22. Registry responsibility

Será autoridad sobre:

```text
tenant → schema mapping
schema status
schema generation
schema version
```

según implementación.

---

# 23. Registry ≠ database catalog

El registry representa conocimiento del framework/control plane.

El catálogo de la base representa:

```text
physical database reality
```

---

# 24. Registry vs introspection

```text
Registry says:
schema_a exists
```

no demuestra por sí solo que la base todavía lo tenga.

---

# 25. Introspection

El sistema general:

```text
96_DATABASE_SCHEMA_INTROSPECTION_SYSTEM.md
```

podrá comprobar realidad física.

---

# 26. Evidence model

VoltStack distinguirá:

```text
REGISTERED
OBSERVED
VERIFIED
STALE
UNKNOWN
```

---

# 27. Registered ≠ verified

Importante:

```text
RegistryEntry
≠
PhysicalExistenceProof
```

---

# 28. Schema resolution pipeline

```text
TenantContext
      ↓
TenantPlacement
      ↓
Isolation Strategy
      ↓
TenantSchemaRegistry
      ↓
TenantSchemaDescriptor
      ↓
Schema Policy Validation
      ↓
TenantSchemaResolution
```

---

# 29. TenantSchemaResolution

```php
final readonly class TenantSchemaResolution
{
    public function __construct(
        public TenantId $tenant,
        public TenantSchemaDescriptor $schema,
        public TenantPlacementGeneration $placementGeneration,
        public SchemaResolutionMode $mode,
    ) {}
}
```

---

# 30. Schema resolution mode

Puede ser:

```text
QUALIFIED_IDENTIFIER
SESSION_SCHEMA
SEARCH_PATH
PLATFORM_NATIVE
CUSTOM
```

---

# 31. Preferred strategy

VoltStack deberá favorecer, cuando sea viable:

```text
Explicit Qualified Identifiers
```

porque reducen dependencia del estado mutable de conexión.

---

# 32. Qualified identifier model

Ejemplo lógico:

```text
Table:
orders

Tenant Schema:
tenant_a
```

se convierte semánticamente en:

```text
QualifiedTableIdentifier(
    schema = tenant_a,
    table = orders
)
```

---

# 33. No string concatenation

Incorrecto:

```php
$table = $tenantSchema . '.orders';
```

Correcto conceptualmente:

```php
new QualifiedTableIdentifier(
    schema: $schema,
    table: new TableName('orders'),
);
```

---

# 34. Compiler responsibility

El Compiler será responsable de producir el SQL correcto:

```text
Query AST
+
Qualified Identifier
+
Platform
    ↓
SQL Compiler
```

---

# 35. Example PostgreSQL

Podría producir:

```sql
SELECT *
FROM "tenant_abc"."orders"
```

---

# 36. Platform syntax

VoltStack no asumirá que todas las plataformas implementan schemas de la misma manera.

---

# 37. Capability model

Usará:

```text
DatabasePlatformCapabilities
```

para conocer:

```text
supportsSchemas
supportsQualifiedTables
supportsSearchPath
supportsSessionSchemaSwitch
supportsCrossSchemaReferences
supportsSchemaPrivileges
```

---

# 38. Platform limitation

Si una plataforma no soporta el modelo requerido:

```text
UNSUPPORTED
```

no se simulará silenciosamente si la emulación rompe garantías.

---

# 39. SQLite

SQLite no ofrece el mismo concepto de schema que PostgreSQL.

Por tanto:

```text
Schema-per-Tenant
```

no deberá declararse soportado simplemente porque pueda aproximarse con otro mecanismo.

---

# 40. MySQL/MariaDB

La semántica de `database/schema` difiere de PostgreSQL.

VoltStack deberá normalizar capacidades sin fingir equivalencia perfecta.

---

# 41. PostgreSQL

Proporciona soporte natural para:

```text
schemas
search_path
schema-qualified objects
schema privileges
```

pero el sistema seguirá evitando depender ciegamente de `search_path`.

---

# 42. Version ≠ capability

Nunca:

```php
if ($version >= 15) {
    // assume schema behavior
}
```

como única evidencia.

---

# 43. Schema naming policy

VoltStack deberá controlar los nombres físicos.

---

# 44. Unsafe naming

No:

```text
tenant slug:
acme"; DROP SCHEMA public; --
```

convertido directamente en identifier.

---

# 45. Naming policy

Ejemplo:

```text
TenantId
    ↓
Stable Internal Identifier
    ↓
SchemaNameGenerator
    ↓
PhysicalSchemaName
```

---

# 46. Generated names

Podrían adoptar:

```text
t_000001
t_01HX9K2Y7P
tenant_8f21c0
```

---

# 47. Human-readable names

Pueden soportarse, pero deberán pasar validación estricta.

---

# 48. Identifier validator

```php
interface DatabaseIdentifierValidator
{
    public function validateSchema(
        string $identifier
    ): PhysicalSchemaName;
}
```

---

# 49. Validation ≠ quoting

Un identifier válido puede necesitar quoting.

Un identifier inválido no se vuelve seguro sólo por quoting.

---

# 50. Identifier normalization

Debe ser platform-aware.

---

# 51. Reserved names

Se deberán impedir o controlar schemas como:

```text
public
information_schema
pg_catalog
mysql
sys
```

según plataforma.

---

# 52. Internal schemas

VoltStack podrá reservar prefijos.

Ejemplo:

```text
volt_
system_
internal_
```

---

# 53. Collision prevention

El registry deberá impedir:

```text
Tenant A → schema_x
Tenant B → schema_x
```

si ambos pertenecen al mismo namespace y la estrategia exige exclusividad.

---

# 54. Schema ownership

Cada schema tendrá un owner lógico:

```text
Schema
   ↓
TenantSchemaId
   ↓
TenantId
```

---

# 55. Ownership invariant

```text
SchemaOwner(schema_x) = Tenant A
```

no podrá cambiar silenciosamente a B.

---

# 56. Schema reuse

Después de eliminar Tenant A, VoltStack debería evitar reutilizar inmediatamente el mismo schema identity para B.

---

# 57. Why

Puede existir:

```text
stale cache
stale jobs
stale cursors
stale workers
stale audit references
```

---

# 58. Tombstones

El registry podrá conservar:

```text
TenantSchemaTombstone
```

para evitar reutilización insegura.

---

# 59. Session-based schema switching

Algunas plataformas permiten:

```text
SET search_path
```

o mecanismos equivalentes.

---

# 60. Session strategy

Pipeline:

```text
ConnectionLease
      ↓
TenantSchemaResolution
      ↓
TenantSchemaSessionInitializer
      ↓
Schema State Applied
      ↓
Query Execution
```

---

# 61. Session schema state

Será considerado:

```text
tenant-sensitive mutable connection state
```

---

# 62. Session state ownership

Deberá pertenecer al:

```text
ConnectionLease
```

no al pool global.

---

# 63. Search path

No deberá aceptar listas arbitrarias provenientes del usuario.

---

# 64. Search path policy

Ejemplo seguro:

```text
tenant_schema
+
approved_system_schema
```

---

# 65. Search path ambiguity

Si:

```text
tenant_a.orders
```

y:

```text
public.orders
```

existen simultáneamente, una configuración incorrecta puede resolver el objeto equivocado.

---

# 66. Search path shadowing

Se considera un riesgo de seguridad y consistencia.

---

# 67. Preferred mitigation

Usar:

```text
qualified identifiers
```

para objetos tenant-sensitive cuando sea viable.

---

# 68. Search path as optimization/convenience

No deberá convertirse automáticamente en la única frontera de aislamiento.

---

# 69. Session reset

Antes de regresar una conexión al pool:

```text
Tenant Schema State
      ↓
Reset
      ↓
Verification
```

---

# 70. Reset result

```text
CLEAN
FAILED
UNKNOWN
DISCARD_REQUIRED
```

---

# 71. UNKNOWN

Regla:

```text
UNKNOWN
⇒
DISCARD CONNECTION
```

---

# 72. Reset example

```text
Lease A
Schema = tenant_a
      ↓
release
      ↓
reset schema/search_path
      ↓
verify
      ↓
Pool
```

---

# 73. Tenant B acquisition

Sólo después:

```text
Pool
 ↓
Lease B
 ↓
Schema = tenant_b
```

---

# 74. No direct A → B switch

Preferir:

```text
A
 ↓
RESET
 ↓
BASELINE
 ↓
B
```

sobre:

```text
A
 ↓
B
```

sin baseline verificable.

---

# 75. Baseline session state

Cada pool deberá conocer:

```text
BaselineSessionProfile
```

---

# 76. Schema baseline

Puede ser:

```text
default schema
restricted schema
no tenant schema
```

según plataforma.

---

# 77. Pool compatibility

Dos tenants pueden compartir pool si:

```text
same endpoint
same credentials compatibility
same platform
same database
session state safely resettable
```

---

# 78. Schema does not always belong in PoolKey

Si schema puede cambiarse/resetearse con seguridad:

```text
TenantSchema
```

no necesita crear un pool por tenant.

---

# 79. Pool explosion avoidance

Esto permite:

```text
10,000 tenant schemas
```

sin requerir:

```text
10,000 pools
```

---

# 80. Non-resettable schema state

Si una plataforma/driver no permite aislamiento seguro mediante reuse:

```text
schema
```

podrá formar parte del pool compatibility domain.

---

# 81. Safety > reuse

Nunca compartir pool sólo para reducir conexiones si no puede probarse un reset seguro.

---

# 82. ORM mapping

Las entidades no deberán tener que conocer su schema tenant físico.

Incorrecto:

```php
#[Table(name: 'tenant_a.orders')]
class Order {}
```

---

# 83. Logical mapping

Preferible:

```php
#[Table(name: 'orders', scope: TenantScope::SCHEMA)]
class Order {}
```

---

# 84. Runtime resolution

Entonces:

```text
Entity Metadata
+
TenantSchemaResolution
    ↓
Physical Table Resolution
```

---

# 85. Metadata reuse

La misma metadata ORM puede servir para:

```text
Tenant A
Tenant B
Tenant C
```

---

# 86. Physical table identity

Se deriva posteriormente.

---

# 87. ORM metadata cache

No deberá compilar:

```text
tenant_a
```

dentro de metadata compartida si esa metadata será usada por B.

---

# 88. Tenant-neutral metadata

Preferir:

```text
TableName = orders
SchemaScope = TENANT
```

---

# 89. Physical resolution

Durante query planning/compilation:

```text
orders
+
TenantSchemaDescriptor
→
tenant_a.orders
```

---

# 90. Query Builder

El Query Builder no concatenará schemas.

---

# 91. Query AST

Podrá representar:

```text
LogicalTableReference(
    table = orders,
    namespaceScope = TENANT
)
```

---

# 92. Semantic resolution

El Semantic Engine podrá resolver:

```text
TENANT namespace
```

hacia un `SchemaIdentity`.

---

# 93. Query planner

El planner conservará esa identidad.

---

# 94. Compiler

Sólo el compiler convertirá la identidad en sintaxis SQL específica.

---

# 95. Raw expressions

Una expresión:

```php
raw('tenant_a.orders')
```

deberá considerarse un escape hatch riesgoso.

---

# 96. Raw schema references

Podrán requerir:

```text
TRUSTED_INTERNAL
```

o una API typed.

---

# 97. Typed schema reference

Ejemplo:

```php
$schema->table('orders');
```

donde `$schema` es un objeto validado, no input arbitrario.

---

# 98. Cross-schema queries

VoltStack podrá soportarlas cuando:

```text
platform supports
policy permits
authorization permits
transaction semantics are valid
```

---

# 99. Tenant cross-schema query

Ejemplo:

```text
tenant_a.orders
JOIN
tenant_b.customers
```

será rechazado por default.

---

# 100. Tenant-to-global query

Puede ser válido:

```text
tenant_a.orders
JOIN
global.currencies
```

---

# 101. Namespace classification

Schemas podrán clasificarse:

```text
TENANT
GLOBAL
SYSTEM
ADMINISTRATIVE
TEMPORARY
```

---

# 102. SchemaCompatibility

Relaciones entre namespaces deberán ser explícitas.

---

# 103. Foreign keys

Cross-schema foreign keys dependerán de capacidades de plataforma.

---

# 104. Tenant-local FK

Ejemplo:

```text
tenant_a.orders.customer_id
        ↓
tenant_a.customers.id
```

válido.

---

# 105. Cross-tenant FK

```text
tenant_a.orders
      ↓
tenant_b.customers
```

rechazado por default.

---

# 106. Global FK

Puede permitirse:

```text
tenant_a.orders.currency_id
        ↓
global.currencies.id
```

si arquitectura/plataforma lo permiten.

---

# 107. Relationship metadata

Deberá expresar:

```text
source namespace scope
target namespace scope
cross-schema policy
```

---

# 108. Relationship loading

Eager/lazy/batch loading deberá resolver el mismo TenantSchemaDescriptor.

---

# 109. Relationship drift

No:

```text
Root entity → tenant_a
Lazy relation → current tenant_b
```

---

# 110. Entity domain capture

La entidad managed deberá estar asociada a su:

```text
TenantPersistenceDomain
```

---

# 111. Transaction semantics

Una transaction tenant schema-scoped deberá fijar:

```text
TenantId
Database
Schema
PlacementGeneration
SchemaGeneration
```

---

# 112. Schema switch during transaction

Rechazado por default.

---

# 113. Reason

Una transaction iniciada en:

```text
tenant_a
```

no debe continuar accidentalmente en:

```text
tenant_b
```

---

# 114. Cross-schema transaction

Una transaction administrativa que accede varios schemas podrá existir sólo explícitamente.

---

# 115. Same physical database

El hecho de que ambos schemas estén en la misma base puede permitir atomicidad física en algunas plataformas.

Pero:

```text
Physical Capability
≠
Authorization
```

---

# 116. Transaction context

Debe registrar el conjunto de namespaces autorizados.

---

# 117. Schema versioning

Cada tenant schema puede encontrarse en una versión distinta durante rollout.

Ejemplo:

```text
Tenant A → v120
Tenant B → v120
Tenant C → v119
Tenant D → v118
```

---

# 118. Version skew

El sistema deberá representar esta realidad explícitamente.

---

# 119. Version skew ≠ corruption

Durante despliegues graduales puede ser legítimo.

---

# 120. Compatibility window

La aplicación podrá declarar:

```text
Supported Schema Versions:
118..120
```

---

# 121. Unsupported schema

Si Tenant E está en:

```text
v110
```

y runtime requiere:

```text
>=118
```

la operación deberá fallar de forma controlada.

---

# 122. SchemaCompatibilityPolicy

```php
interface TenantSchemaCompatibilityPolicy
{
    public function evaluate(
        TenantSchemaVersion $version,
        ApplicationDatabaseVersion $application
    ): SchemaCompatibilityResult;
}
```

---

# 123. Compatibility states

```text
COMPATIBLE
COMPATIBLE_WITH_LIMITATIONS
MIGRATION_REQUIRED
TOO_NEW
TOO_OLD
UNKNOWN
```

---

# 124. UNKNOWN ≠ compatible

---

# 125. Schema migrations

Se integrará con:

```text
266_DATABASE_TENANT_MIGRATION_INTEGRATION_SYSTEM.md
```

---

# 126. Migration fan-out

Para 10,000 schemas:

```text
1 migration
×
10,000 tenant schemas
```

es un problema distribuido/operacional.

---

# 127. No monolithic transaction

VoltStack no intentará:

```text
BEGIN
migrate 10,000 schemas
COMMIT
```

como estrategia general.

---

# 128. Migration batches

Podrán organizarse:

```text
Batch 1 → tenants 1..100
Batch 2 → tenants 101..200
...
```

---

# 129. Migration status

Cada schema tendrá su estado/version independiente.

---

# 130. Migration failure

Si falla Tenant 501:

```text
Tenants 1..500 migrated
Tenant 501 failed
Tenants 502..N pending
```

no se declarará rollback global ficticio.

---

# 131. Migration compatibility

La aplicación deberá poder operar durante ventanas de version skew si el rollout lo requiere.

---

# 132. Expand/contract

Para zero downtime se favorecerá:

```text
EXPAND
  ↓
MIXED VERSION WINDOW
  ↓
BACKFILL
  ↓
CUTOVER
  ↓
CONTRACT
```

---

# 133. Schema creation

Provisionar un nuevo tenant puede requerir:

```text
Allocate Schema Identity
Validate Physical Name
Create Schema
Apply Baseline
Apply Migrations
Verify
Activate Registry Entry
```

---

# 134. Activation ordering

No marcar:

```text
ACTIVE
```

antes de verificar que el schema está utilizable.

---

# 135. Provisioning failure

Debe dejar:

```text
FAILED
```

o limpiar de forma segura.

Nunca publicar un schema parcialmente creado como activo.

---

# 136. Schema deletion

Pipeline conceptual:

```text
Suspend Tenant
      ↓
Drain Operations
      ↓
Verify No Active Transactions
      ↓
Archive/Backup if required
      ↓
Mark Deleting
      ↓
Drop Schema
      ↓
Verify
      ↓
Tombstone
```

---

# 137. DROP SCHEMA safety

Será una operación privilegiada y destructiva.

---

# 138. CASCADE

Nunca utilizar:

```sql
DROP SCHEMA ... CASCADE
```

como comportamiento implícito universal.

---

# 139. Destruction plan

Deberá conocerse qué objetos serán eliminados.

---

# 140. Schema introspection

VoltStack podrá observar:

```text
tables
columns
indexes
constraints
sequences
views
functions
triggers
```

según capabilities.

---

# 141. Introspection scope

Una introspection tenant-scoped deberá limitarse al schema correspondiente.

---

# 142. Metadata cache

Clave conceptual:

```text
Database
+
TenantSchemaId
+
SchemaGeneration
+
SchemaVersion
+
PlatformGeneration
```

cuando el contenido sea tenant-specific.

---

# 143. Shared structural metadata

Si todos los tenants usan exactamente la misma estructura lógica, puede existir metadata compilada compartida.

---

# 144. Logical vs observed metadata

Distinguir:

```text
Expected Tenant Schema Model
```

de:

```text
Observed Physical Tenant Schema
```

---

# 145. Drift

```text
Expected v120
Observed structure differs
```

produce:

```text
SCHEMA DRIFT
```

---

# 146. Version number ≠ structural proof

Un registro:

```text
version = 120
```

no demuestra que todos los objetos sean correctos.

---

# 147. Drift detection

Puede utilizar:

```text
schema introspection
schema fingerprints
migration history
selected structural checks
```

---

# 148. Drift response

Según política:

```text
WARN
READ_ONLY
REJECT_WRITES
QUARANTINE
REQUIRE_REPAIR
```

---

# 149. Schema fingerprint

Podrá representar una forma canónica de estructura relevante.

---

# 150. Fingerprint ≠ security signature

Su propósito principal es comparación estructural.

---

# 151. Backup

Schema-per-tenant puede facilitar backups lógicos por schema.

---

# 152. Backup identity

El artefacto deberá contener:

```text
TenantId
TenantSchemaId
SchemaGeneration
SchemaVersion
SourceDatabase
BackupTimestamp
BackupFormatVersion
```

según política.

---

# 153. Restore

Antes de restaurar:

```text
Backup Tenant
```

debe compararse con:

```text
Target Tenant
```

---

# 154. Cross-tenant restore

Rechazado por default.

---

# 155. Clone

Si se desea:

```text
Tenant A
→
New Tenant B
```

deberá utilizarse un workflow de clone.

---

# 156. Clone schema identity

El nuevo tenant obtendrá:

```text
new TenantSchemaId
new PhysicalSchemaName
new Generation
```

---

# 157. No physical name reuse

Copiar schema no significa reutilizar su identidad.

---

# 158. Tenant mobility

Un tenant podrá moverse:

```text
Database A / Schema X
        ↓
Database B / Schema Y
```

---

# 159. Mobility stages

```text
SOURCE_ACTIVE
PREPARING_TARGET
COPYING
CATCHING_UP
CUTOVER_PENDING
TARGET_ACTIVE
SOURCE_RETIRING
COMPLETED
```

---

# 160. Authoritative schema

En cada fase deberá existir una autoridad definida.

---

# 161. Read/write authority

Puede distinguir:

```text
ReadAuthority
WriteAuthority
```

pero nunca ser ambiguo.

---

# 162. Cutover

Al cambiar:

```text
Schema X
→
Schema Y
```

deberá actualizarse:

```text
PlacementGeneration
and/or
SchemaGeneration
```

---

# 163. Stale query context

Un query context creado para schema X no deberá ejecutar silenciosamente sobre Y.

---

# 164. Re-resolution

Antes de side effects puede re-resolverse.

---

# 165. Active transaction

No se moverá.

---

# 166. Cache invalidation

Cutover deberá invalidar:

```text
schema resolution cache
tenant metadata cache
connection session assumptions
query result cache
entity cache
```

cuando corresponda.

---

# 167. Jobs

Un job deberá almacenar:

```text
TenantId
```

no sólo:

```text
schema_name
```

---

# 168. Why

El schema puede cambiar antes de que el job se ejecute.

---

# 169. Job execution

```text
TenantId
   ↓
Current Placement
   ↓
Current Schema Resolution
```

---

# 170. Generation-bound jobs

Algunas tareas administrativas podrán exigir una generation concreta.

Ejemplo:

```text
Run migration against schema generation 42 only
```

---

# 171. Import

Import hacia tenant schema deberá resolver target schema desde TenantId.

---

# 172. Export

Export deberá usar schema resolution actual y conservar provenance.

---

# 173. Large datasets

Chunk/Lazy/Streaming operations deberán fijar el schema domain apropiadamente.

---

# 174. Streaming

Si mantiene una conexión abierta:

```text
schema session state
```

permanecerá ligado al lease hasta cierre.

---

# 175. Chunk processing

Cada chunk no deberá cambiar de schema arbitrariamente.

---

# 176. Long-running processing

Si ocurre migration/cutover durante procesamiento:

```text
generation mismatch
```

deberá detectarse según consistency policy.

---

# 177. Cache isolation

Result/entity cache keys deberán incluir tenant identity/domain.

---

# 178. Compiled query reuse

Un compiled query puede compartirse si utiliza un schema binding abstracto seguro.

---

# 179. Schema-bound compiled SQL

Si el SQL compilado contiene:

```text
"tenant_a"."orders"
```

no deberá reutilizarse para Tenant B.

---

# 180. Compiler cache key

Entonces deberá incluir:

```text
SchemaIdentity/Generation
```

si el SQL físico queda schema-bound.

---

# 181. Alternative compilation

Podría existir:

```text
Logical Compiled Plan
```

tenant-neutral y:

```text
Physical SQL Compilation
```

tenant-bound.

---

# 182. Preferred architecture

```text
Query AST
   ↓
Semantic Plan
   ↓
Logical Compiled Plan
   ↓
Tenant Schema Binding
   ↓
Platform SQL
```

cuando sea compatible con el pipeline existente.

---

# 183. Query plan cache

Planes lógicos podrán compartirse más ampliamente que SQL físico.

---

# 184. Prepared statements

Una sentencia preparada schema-bound no deberá reutilizarse para otro schema si su SQL contiene identifiers físicos diferentes.

---

# 185. Statement cache

Debe incluir:

```text
schema binding identity
```

cuando corresponda.

---

# 186. Security boundary

Tenant Schema Resolution será security-sensitive.

---

# 187. User input

Nunca permitir:

```text
?schema=tenant_b
```

como selector directo en una request tenant normal.

---

# 188. Trusted resolution

El schema se deriva de:

```text
Authorized TenantContext
+
TenantPlacement
+
SchemaRegistry
```

---

# 189. Identifier injection

Toda ruta:

```text
TenantContext
→
PhysicalSchemaName
→
Compiler
```

deberá permanecer typed.

---

# 190. No arbitrary FQ identifier

Evitar:

```php
DB::table($_GET['schema'] . '.orders');
```

---

# 191. Administrative selection

Herramientas admin podrán seleccionar schemas explícitamente, pero bajo:

```text
CrossTenantCapability
```

y auditoría.

---

# 192. Privileges

Cuando la plataforma lo permita, podrán utilizarse roles limitados por schema.

---

# 193. Defense in depth

Ejemplo:

```text
Tenant A Session
Role A
Schema A
```

puede impedir acceso a:

```text
Schema B
```

incluso ante error lógico.

---

# 194. Shared credentials caveat

Si las credenciales pueden acceder a todos los schemas, la seguridad depende más del framework.

---

# 195. Least privilege strategy

Opciones:

```text
Shared Role
Role per Tenant Group
Role per Tenant
Dedicated Credentials
```

---

# 196. Role-per-tenant scalability

No deberá imponerse universalmente debido a:

```text
role explosion
credential rotation complexity
catalog overhead
operational cost
```

---

# 197. Security policy

La estrategia será configurable según nivel de aislamiento.

---

# 198. Cross-schema privilege

No deberá concederse por convenience sin policy explícita.

---

# 199. Persistent runtime

Especialmente crítico con:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 200. Worker global state

Nunca:

```php
static $currentSchema;
```

---

# 201. Request scope

Cada request tendrá:

```text
TenantSchemaResolution
```

propio.

---

# 202. OpenSwoole

Cada coroutine tendrá su propia resolución.

---

# 203. Shared connection pool

Puede ser compartido.

Pero:

```text
Active Lease Schema State
```

será exclusivo del lease.

---

# 204. Coroutine contamination

Nunca:

```text
Coroutine A sets schema_a
Coroutine B uses same physical connection concurrently
```

---

# 205. Lease exclusivity

Una conexión mutable no multiplexable tendrá:

```text
one active owner
```

---

# 206. Request termination

Debe liberar:

```text
active results
transactions
schema session state
connection lease
schema context
```

---

# 207. Reset ordering

```text
Close Results
    ↓
Resolve Transaction
    ↓
Reset Schema State
    ↓
Reset Role
    ↓
Reset Other Session State
    ↓
Verify
    ↓
Pool
```

---

# 208. Tainted connection

Si el reset no puede probarse:

```text
TAINTED
```

y la conexión se descarta.

---

# 209. Telemetry

Eventos conceptuales:

```text
TenantSchemaResolutionStarted
TenantSchemaResolved
TenantSchemaResolutionFailed
TenantSchemaVersionMismatch
TenantSchemaDriftDetected
TenantSchemaSessionApplied
TenantSchemaSessionReset
TenantSchemaSessionResetFailed
TenantSchemaGenerationChanged
TenantSchemaSecurityViolation
```

---

# 210. Metrics

Ejemplos:

```text
database_tenant_schema_resolution_duration
database_tenant_schema_resolution_failures_total
database_tenant_schema_version_mismatch_total
database_tenant_schema_drift_total
database_tenant_schema_session_reset_failures_total
```

---

# 211. Cardinality

No utilizar:

```text
schema_name
tenant_id
```

como labels generales de métricas.

---

# 212. Bounded labels

Preferir:

```text
platform
resolution_mode
result
compatibility
failure_class
runtime
```

---

# 213. Trace attributes

Podrán incluir:

```text
tenant_schema_strategy
schema_version
schema_generation class/fingerprint
resolution mode
```

sin exponer identifiers sensibles innecesariamente.

---

# 214. Audit

Operaciones como:

```text
create schema
drop schema
clone schema
move schema
cross-schema query
privileged schema access
```

deberán ser auditables.

---

# 215. Diagnostics

Comando conceptual:

```text
php volt database:tenant:schema acme
```

---

# 216. Example output

```text
Tenant Schema Diagnostics
──────────────────────────────────────

Tenant
  Status:               ACTIVE
  Isolation Strategy:   SCHEMA_PER_TENANT

Database
  Logical Database:     customer-data
  Region:               mx-central

Schema
  Schema ID:            schema:tenant:***
  Physical Name:        [internal]
  Generation:           42
  Version:              120
  Status:               ACTIVE

Resolution
  Mode:                 QUALIFIED_IDENTIFIER
  Registry:             available
  Physical Verification: verified

Compatibility
  Application:          118..120
  Schema:               120
  Result:               COMPATIBLE

Connection
  Shared Pool:          yes
  Session Switching:    no
  Reset Requirement:    baseline

Security
  Identifier Validation: enabled
  Cross-Schema Access:   denied

Overall
  Status:               VALID
```

---

# 217. Explain mode

```text
php volt database:tenant:schema acme --explain
```

podrá mostrar:

```text
1. TenantContext resolved.
2. Placement strategy requires tenant schema.
3. Schema registry returned generation 42.
4. Schema status is ACTIVE.
5. Schema version 120 is application-compatible.
6. Physical schema evidence is valid.
7. Qualified identifier mode selected.
8. Connection does not require mutable search_path.
9. Query namespace will bind to tenant schema.
10. Cross-schema access remains disabled.
```

---

# 218. Health checks

Distinguir:

```text
Schema Registry Health
Schema Catalog Health
Migration Health
Schema Compatibility Health
Schema Drift Health
```

---

# 219. One tenant degraded

Un tenant puede tener:

```text
SCHEMA_VERSION_TOO_OLD
```

sin declarar toda la plataforma down.

---

# 220. Fleet health

Para miles de schemas podrán agregarse:

```text
compatible
migration_pending
migration_failed
drift_detected
unknown
```

---

# 221. Testing architecture

Se requerirán:

```text
Schema Resolution Tests
Identifier Security Tests
Schema Registry Tests
Generation Tests
Version Tests
Compatibility Tests
ORM Mapping Tests
Compiler Tests
Connection Session Tests
Pool Reuse Tests
Transaction Tests
Relationship Tests
Migration Tests
Backup Tests
Restore Tests
Mobility Tests
Persistent Runtime Tests
Concurrency Tests
Fault Injection Tests
```

---

# 222. Resolution test

```text
Tenant A
→
Schema A
```

deberá ser determinista bajo la misma generation.

---

# 223. Collision test

```text
Tenant A → schema_x
Tenant B → schema_x
```

deberá rechazarse cuando no sea permitido.

---

# 224. Injection test

Inputs maliciosos no deberán convertirse en schema identifiers.

---

# 225. Search-path contamination test

```text
Lease A → schema_a
release
Lease B → schema_b
```

B nunca deberá observar A.

---

# 226. Reset failure test

Forzar error al restaurar search path.

La conexión deberá descartarse.

---

# 227. ORM test

Misma entidad:

```text
Order
```

debe funcionar para:

```text
tenant_a.orders
tenant_b.orders
```

sin duplicar metadata ORM innecesariamente.

---

# 228. Compiler test

El mismo logical query deberá compilar correctamente con distintos schema bindings.

---

# 229. Cache test

SQL schema-bound de A no deberá reutilizarse incorrectamente para B.

---

# 230. Transaction test

Intentar cambiar schema dentro de transaction tenant-scoped deberá fallar.

---

# 231. Relationship test

Lazy relation de entidad A deberá resolver schema A incluso si otro request/coroutine usa B simultáneamente.

---

# 232. Version skew test

```text
A → v120
B → v119
C → v118
```

deberán evaluarse según compatibility window.

---

# 233. Unsupported version test

Tenant v110 deberá rechazarse si runtime soporta sólo v118..120.

---

# 234. Drift test

Registry:

```text
v120
```

pero introspection detecta columna faltante.

Debe reportarse drift.

---

# 235. Provisioning test

Schema parcialmente creado nunca deberá marcarse ACTIVE.

---

# 236. Migration failure test

Fallando Tenant 501 no se declararán revertidos Tenants 1..500 si ya hicieron commit.

---

# 237. Backup test

Backup de A deberá contener provenance correcta.

---

# 238. Restore test

Backup A → Tenant B deberá rechazarse por default.

---

# 239. Clone test

Clone A → B deberá generar nueva schema identity.

---

# 240. Mobility test

Mover:

```text
DB1/schema_a
→
DB2/schema_b
```

deberá incrementar generaciones apropiadas.

---

# 241. Stale context test

Una query preparada bajo generation anterior deberá detectarse si ya no es válida.

---

# 242. Worker reuse test

```text
Request A → schema_a
Request B → schema_b
Request C → schema_c
```

sin contaminación.

---

# 243. Coroutine test

Múltiples schemas simultáneos deberán permanecer aislados.

---

# 244. Large fleet test

Probar:

```text
10
100
1,000
10,000+
```

schemas para medir:

```text
registry performance
catalog pressure
metadata memory
migration throughput
pool behavior
```

---

# 245. Proposed directory

```text
src/Quantum/Multitenancy/Database/Schema/
│
├── Contract/
│   ├── TenantSchemaResolver.php
│   ├── TenantSchemaRegistry.php
│   ├── TenantSchemaNamingPolicy.php
│   ├── TenantSchemaCompatibilityPolicy.php
│   └── TenantSchemaSessionManager.php
│
├── Model/
│   ├── TenantSchemaId.php
│   ├── PhysicalSchemaName.php
│   ├── TenantSchemaDescriptor.php
│   ├── TenantSchemaGeneration.php
│   ├── TenantSchemaVersion.php
│   ├── TenantSchemaStatus.php
│   └── TenantSchemaResolution.php
│
├── Resolution/
│   ├── DefaultTenantSchemaResolver.php
│   ├── SchemaResolutionMode.php
│   └── TenantSchemaResolutionValidator.php
│
├── Registry/
│   ├── DefaultTenantSchemaRegistry.php
│   ├── TenantSchemaRegistryEntry.php
│   └── TenantSchemaTombstone.php
│
├── Naming/
│   ├── TenantSchemaNameGenerator.php
│   ├── TenantSchemaIdentifierValidator.php
│   └── ReservedSchemaPolicy.php
│
├── Session/
│   ├── TenantSchemaSessionInitializer.php
│   ├── TenantSchemaSessionResetter.php
│   ├── TenantSchemaSessionState.php
│   └── TenantSchemaSessionResetResult.php
│
├── Binding/
│   ├── TenantSchemaBinding.php
│   ├── QualifiedTenantTableResolver.php
│   └── TenantSchemaNamespaceResolver.php
│
├── Compatibility/
│   ├── TenantSchemaCompatibilityResult.php
│   ├── TenantSchemaVersionWindow.php
│   └── TenantSchemaDriftDetector.php
│
├── Security/
│   ├── TenantSchemaSecurityPolicy.php
│   ├── CrossSchemaAccessGuard.php
│   └── TenantSchemaPrivilegePolicy.php
│
├── Telemetry/
│   └── TenantSchemaTelemetry.php
│
├── Diagnostics/
│   └── TenantSchemaDiagnostics.php
│
└── Exception/
    ├── TenantSchemaException.php
    ├── TenantSchemaResolutionException.php
    ├── TenantSchemaNotFoundException.php
    ├── TenantSchemaCollisionException.php
    ├── TenantSchemaVersionException.php
    ├── TenantSchemaDriftException.php
    ├── TenantSchemaSessionException.php
    ├── UnsafeTenantSchemaIdentifierException.php
    └── CrossSchemaAccessException.php
```

---

# 246. Dependency direction

```text
Multitenancy
    ↓
Tenant Schema Integration
    ↓
Database Schema Contracts
    ↓
Query / ORM / Connection
    ↓
Compiler / Executor
```

Nunca:

```text
Driver
→
TenantSchemaRegistry
```

---

# 247. Schema invariants

## DB-TSI-001

Tenant Schema será una identidad typed.

## DB-TSI-002

Tenant Schema no será raw string.

## DB-TSI-003

TenantId no será PhysicalSchemaName.

## DB-TSI-004

Tenant slug no será PhysicalSchemaName por default.

## DB-TSI-005

Logical Schema ID será distinto del nombre físico.

## DB-TSI-006

Schema generation será distinta de schema version.

## DB-TSI-007

Schema registry será distinto del database catalog.

## DB-TSI-008

Registered no equivaldrá a physically verified.

## DB-TSI-009

UNKNOWN schema status no equivaldrá a ACTIVE.

## DB-TSI-010

Schema resolution partirá de trusted TenantContext.

## DB-TSI-011

User input no seleccionará directamente tenant schema.

## DB-TSI-012

Schema identifiers serán validados.

## DB-TSI-013

Schema identifiers serán platform-aware.

## DB-TSI-014

Quoting no sustituirá validation.

## DB-TSI-015

Reserved schemas estarán protegidos.

## DB-TSI-016

Schema name collisions serán detectadas.

## DB-TSI-017

Schema ownership será explícito.

## DB-TSI-018

Schema ownership no cambiará silenciosamente.

## DB-TSI-019

Deleted schema identities podrán conservar tombstones.

## DB-TSI-020

Schema names no se reutilizarán de forma insegura.

## DB-TSI-021

Qualified identifiers serán preferidos cuando sea viable.

## DB-TSI-022

Query Builder no concatenará schema strings.

## DB-TSI-023

Query AST podrá representar tenant namespace.

## DB-TSI-024

Semantic Engine resolverá namespace semánticamente.

## DB-TSI-025

Compiler generará syntax específica.

## DB-TSI-026

Compiler no resolverá TenantContext global.

## DB-TSI-027

Driver no resolverá TenantContext.

## DB-TSI-028

Platform capabilities gobernarán schema support.

## DB-TSI-029

Version no equivaldrá a capability.

## DB-TSI-030

Unsupported schema strategy no se emulará silenciosamente.

## DB-TSI-031

Search path será tenant-sensitive state.

## DB-TSI-032

Search path no será la única frontera por default.

## DB-TSI-033

Search path input será controlado.

## DB-TSI-034

Search path shadowing será considerado riesgo.

## DB-TSI-035

Schema session state pertenecerá al lease.

## DB-TSI-036

Schema session state no pertenecerá al worker.

## DB-TSI-037

Connection reuse requerirá schema reset.

## DB-TSI-038

Schema reset deberá verificarse.

## DB-TSI-039

UNKNOWN reset causará discard.

## DB-TSI-040

Tenant A state no llegará a Tenant B.

## DB-TSI-041

Baseline session state será explícito.

## DB-TSI-042

Pool compatibility será distinta de schema identity.

## DB-TSI-043

Schema por tenant no implicará pool por tenant.

## DB-TSI-044

Pool sharing sólo ocurrirá cuando reset sea seguro.

## DB-TSI-045

Safety tendrá prioridad sobre pool reuse.

## DB-TSI-046

ORM metadata será tenant-neutral cuando sea posible.

## DB-TSI-047

Entity classes no hardcodearán tenant schema.

## DB-TSI-048

Physical schema binding ocurrirá después de logical mapping.

## DB-TSI-049

IdentityMap conservará tenant persistence domain.

## DB-TSI-050

UnitOfWork conservará tenant persistence domain.

## DB-TSI-051

Relationships conservarán tenant schema.

## DB-TSI-052

Lazy loading no consultará global mutable schema.

## DB-TSI-053

Eager loading conservará schema domain.

## DB-TSI-054

Batch loading conservará schema domain.

## DB-TSI-055

Cross-tenant schema relationships serán rechazadas por default.

## DB-TSI-056

Tenant-to-global relationships podrán declararse.

## DB-TSI-057

Cross-schema FK será capability-driven.

## DB-TSI-058

Transaction fijará tenant schema domain.

## DB-TSI-059

Schema switching durante transaction será rechazado por default.

## DB-TSI-060

Nested transaction conservará schema domain.

## DB-TSI-061

Administrative cross-schema transaction será explícita.

## DB-TSI-062

Physical atomicity no equivaldrá a authorization.

## DB-TSI-063

Schema version skew será representable.

## DB-TSI-064

Schema version skew no será automáticamente corruption.

## DB-TSI-065

Application compatibility window será explícita.

## DB-TSI-066

Too-old schema será detectable.

## DB-TSI-067

Too-new schema será detectable.

## DB-TSI-068

UNKNOWN compatibility no equivaldrá a compatible.

## DB-TSI-069

Schema migration será tenant-aware.

## DB-TSI-070

Migration fan-out será bounded/orchestrated.

## DB-TSI-071

Miles de schemas no se migrarán en una transaction global ficticia.

## DB-TSI-072

Migration failure no fingirá rollback global.

## DB-TSI-073

Per-schema migration status será observable.

## DB-TSI-074

Expand/contract será soportable.

## DB-TSI-075

Provisioning no marcará ACTIVE antes de verification.

## DB-TSI-076

Partial provisioning no será considerado usable.

## DB-TSI-077

Schema deletion será privileged.

## DB-TSI-078

DROP SCHEMA no será consecuencia implícita de ORM remove.

## DB-TSI-079

CASCADE no será default universal.

## DB-TSI-080

Schema introspection será scoped.

## DB-TSI-081

Schema metadata distinguirá expected vs observed.

## DB-TSI-082

Migration version no probará estructura completa.

## DB-TSI-083

Schema drift será detectable.

## DB-TSI-084

Drift policy será explícita.

## DB-TSI-085

Schema fingerprints podrán utilizarse.

## DB-TSI-086

Fingerprint no será tratado como security signature.

## DB-TSI-087

Tenant backup conservará schema provenance.

## DB-TSI-088

Restore validará tenant identity.

## DB-TSI-089

Cross-tenant restore será rechazado por default.

## DB-TSI-090

Clone será distinto de restore.

## DB-TSI-091

Clone creará nueva schema identity.

## DB-TSI-092

Tenant mobility podrá cambiar database y schema.

## DB-TSI-093

Mobility tendrá authoritative schema.

## DB-TSI-094

Cutover actualizará generations correspondientes.

## DB-TSI-095

Stale query context no ejecutará silenciosamente.

## DB-TSI-096

Active transaction no migrará de schema.

## DB-TSI-097

Cutover invalidará tenant-sensitive caches.

## DB-TSI-098

Jobs almacenarán TenantId, no sólo schema name.

## DB-TSI-099

Jobs resolverán current schema cuando corresponda.

## DB-TSI-100

Administrative jobs podrán ser generation-bound.

## DB-TSI-101

Import resolverá schema desde TenantContext.

## DB-TSI-102

Export conservará schema provenance.

## DB-TSI-103

Chunk processing conservará schema domain.

## DB-TSI-104

Lazy collection conservará schema domain.

## DB-TSI-105

Streaming conservará schema lease state.

## DB-TSI-106

Long-running operations podrán detectar generation change.

## DB-TSI-107

Result cache será tenant/schema aware.

## DB-TSI-108

Entity cache será tenant/schema aware.

## DB-TSI-109

Tenant-neutral metadata podrá compartirse.

## DB-TSI-110

Schema-bound SQL no se reutilizará entre schemas incompatibles.

## DB-TSI-111

Statement cache será schema-aware cuando corresponda.

## DB-TSI-112

Logical plans podrán compartirse más ampliamente que physical SQL.

## DB-TSI-113

Prepared statement no cruzará schema binding incompatible.

## DB-TSI-114

Cross-schema queries estarán deshabilitadas por default para tenants.

## DB-TSI-115

Cross-schema privileged operations serán auditables.

## DB-TSI-116

Tenant-to-global queries podrán permitirse explícitamente.

## DB-TSI-117

Schema privileges podrán reforzar aislamiento.

## DB-TSI-118

Shared credentials no serán considerados aislamiento suficiente.

## DB-TSI-119

Least privilege será recomendado.

## DB-TSI-120

Role-per-tenant no será requisito universal.

## DB-TSI-121

Persistent workers no almacenarán current schema global.

## DB-TSI-122

Request schema state será scoped.

## DB-TSI-123

Coroutine schema state será scoped.

## DB-TSI-124

Physical connection tendrá un owner activo por default.

## DB-TSI-125

Concurrent coroutines no compartirán mutable session state.

## DB-TSI-126

Request termination reseteará schema state.

## DB-TSI-127

Tainted connection será descartada.

## DB-TSI-128

Schema resolution será observable.

## DB-TSI-129

Schema version mismatch será observable.

## DB-TSI-130

Schema drift será observable.

## DB-TSI-131

Metrics evitarán tenant/schema high-cardinality labels.

## DB-TSI-132

Secrets no aparecerán en schema diagnostics.

## DB-TSI-133

Physical schema names podrán redactarse.

## DB-TSI-134

Schema administrative operations serán auditables.

## DB-TSI-135

Schema resolution tendrá unit tests.

## DB-TSI-136

Schema isolation tendrá integration tests.

## DB-TSI-137

Identifier injection tendrá security tests.

## DB-TSI-138

Search-path contamination tendrá tests.

## DB-TSI-139

Connection reset failure tendrá fault tests.

## DB-TSI-140

ORM schema binding tendrá tests.

## DB-TSI-141

Compiler schema binding tendrá tests.

## DB-TSI-142

Statement cache isolation tendrá tests.

## DB-TSI-143

Transaction schema affinity tendrá tests.

## DB-TSI-144

Relationship schema affinity tendrá tests.

## DB-TSI-145

Version skew tendrá tests.

## DB-TSI-146

Drift tendrá tests.

## DB-TSI-147

Migration fan-out tendrá tests.

## DB-TSI-148

Backup/restore tendrá tests.

## DB-TSI-149

Mobility tendrá tests.

## DB-TSI-150

Persistent runtime tendrá tests.

## DB-TSI-151

Concurrent runtime tendrá tests.

## DB-TSI-152

Large schema fleets tendrán benchmarks.

## DB-TSI-153

Schema Registry no ejecutará queries de aplicación.

## DB-TSI-154

Schema Resolver no abrirá conexiones arbitrariamente.

## DB-TSI-155

Schema isolation no duplicará ORM.

## DB-TSI-156

Schema isolation no duplicará Query Engine.

## DB-TSI-157

Schema isolation no generará SQL fuera del Compiler.

## DB-TSI-158

Schema isolation no sustituirá Authorization.

## DB-TSI-159

Schema isolation no sustituirá Connection Security.

## DB-TSI-160

Schema isolation será una especialización de TenantIsolationDomain.

## DB-TSI-161

Tenant schema deberá poder evolucionar sin cambiar entity classes.

## DB-TSI-162

Tenant schema deberá poder moverse sin cambiar business queries.

## DB-TSI-163

Schema generation será verificable antes de side effects cuando corresponda.

## DB-TSI-164

Schema version será verificable antes de usar features incompatibles.

## DB-TSI-165

UNKNOWN schema ownership fallará cerrado.

## DB-TSI-166

UNKNOWN schema generation no se asumirá actual.

## DB-TSI-167

UNKNOWN schema version no se asumirá compatible.

## DB-TSI-168

UNKNOWN session cleanliness no se asumirá reusable.

## DB-TSI-169

No habrá implicit fallback al schema público/default para operaciones tenant-scoped.

## DB-TSI-170

Toda operación schema-per-tenant deberá poder demostrar qué TenantSchemaDescriptor gobierna su acceso.

---

# 248. Modelo formal

Sea:

```text
T = TenantContext
P = TenantPlacement
S = TenantSchemaDescriptor
G = TenantSchemaGeneration
V = TenantSchemaVersion
Q = Query
C = ConnectionLease
```

La resolución será:

```text
ResolveSchema(T,P)
→
S
```

y una operación será válida cuando:

```text
Active(S)
∧ PlacementCompatible(P,S)
∧ GenerationCurrent(G)
∧ VersionCompatible(V)
∧ QueryBound(Q,S)
∧ ConnectionCompatible(C,S)
```

para las dimensiones aplicables.

---

# 249. Query binding

Sea:

```text
L = LogicalTableReference
```

Entonces:

```text
Bind(L,S)
→
PhysicalTableReference
```

Ejemplo:

```text
L = orders
S = tenant_a
```

produce conceptualmente:

```text
tenant_a.orders
```

sin que `orders` ni el modelo ORM necesiten conocer `tenant_a`.

---

# 250. Connection reuse formal

Una conexión `C` usada por schema A podrá utilizarse para schema B sólo si:

```text
Released(C)
∧ NoTransaction(C)
∧ NoActiveResult(C)
∧ ResetToBaseline(C)
∧ ResetVerified(C)
∧ CompatiblePool(C,B)
```

Si:

```text
ResetVerified(C) = UNKNOWN
```

entonces:

```text
Reusable(C,B) = false
```

---

# 251. Schema safety

Formalmente:

```text
SchemaSafe(Operation)
=
ResolvedTenant
∧ ResolvedSchema
∧ ValidOwnership
∧ CurrentGeneration
∧ CompatibleVersion
∧ ValidNamespaceBinding
∧ ValidConnectionState
∧ ValidTransactionDomain
∧ SecurityPolicySatisfied
```

---

# 252. Arquitectura completa

```text
                       TenantContext
                            │
                            ▼
                     TenantPlacement
                            │
                            ▼
                  TenantSchemaResolver
                            │
               ┌────────────┼────────────┐
               ▼            ▼            ▼
            Registry     Generation    Version
               │            │            │
               └────────────┼────────────┘
                            ▼
                 TenantSchemaDescriptor
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
       Logical Query                Connection
              │                      Resolution
              ▼                           │
     Tenant Namespace Binding             ▼
              │                    ConnectionLease
              ▼                           │
       Semantic Query                     ▼
              │                   Session Initialization
              ▼                           │
          Query Plan                      │
              │                           │
              └─────────────┬─────────────┘
                            ▼
                       SQL Compiler
                            │
                            ▼
                    Qualified SQL
                            │
                            ▼
                        Executor
                            │
                            ▼
                        Database
                            │
                            ▼
                   Tenant Schema Data
                            │
                            ▼
                    Session Reset
                            │
                    ┌───────┴────────┐
                    ▼                ▼
                  CLEAN          UNKNOWN/FAILED
                    │                │
                    ▼                ▼
                  POOL            DISCARD
```

---

# 253. Estrategia recomendada para VoltStack

La arquitectura base deberá favorecer:

```text
Logical ORM Metadata
        ↓
Logical Query References
        ↓
TenantSchemaResolution
        ↓
Typed Schema Binding
        ↓
Platform Compiler
        ↓
Qualified SQL
```

Esto minimiza dependencia de:

```text
mutable connection search_path
```

y permite mantener:

```text
ORM Metadata
Query AST
Domain Models
Repositories
Model API
```

independientes del schema físico.

El cambio de schema mediante sesión seguirá disponible cuando la plataforma y la política lo justifiquen, pero será una estrategia explícita y no una suposición universal.

---

# 254. Regla arquitectónica definitiva

> **VoltStack tratará el schema de un tenant como una identidad estructural versionada y validada, no como una cadena SQL. El TenantSchemaResolver transformará el TenantContext y TenantPlacement en un TenantSchemaDescriptor confiable; Query Engine conservará esa identidad semánticamente, Compiler será el único responsable de convertirla en identifiers SQL específicos de plataforma y ConnectionManager garantizará que cualquier estado de schema aplicado a una conexión sea eliminado y verificado antes de reutilizarla.**

De esta manera:

```text
Tenant A
```

podrá utilizar:

```text
Database X / Schema A
```

y posteriormente migrar a:

```text
Database Y / Schema Z
```

sin modificar:

```php
Order::query()
    ->where('status', 'pending')
    ->get();
```

porque:

```text
Business Query
≠
Physical Schema
```

---

# 255. Estado del Bloque 26

```text
BLOCK 26 — MULTITENANCY INTEGRATION

✓ 261_DATABASE_MULTITENANCY_INTEGRATION_ARCHITECTURE.md
✓ 262_DATABASE_TENANT_CONNECTION_RESOLUTION_SYSTEM.md
✓ 263_DATABASE_TENANT_DATABASE_ISOLATION_SYSTEM.md
✓ 264_DATABASE_TENANT_SCHEMA_ISOLATION_SYSTEM.md
│
├── 265_DATABASE_TENANT_QUERY_CONTEXT_SYSTEM.md
└── 266_DATABASE_TENANT_MIGRATION_INTEGRATION_SYSTEM.md
```

---

# 256. Siguiente documento

```text
265_DATABASE_TENANT_QUERY_CONTEXT_SYSTEM.md
```

El siguiente documento definirá cómo VoltStack transportará de forma segura:

```text
TenantId
TenantIsolationDomain
TenantPersistenceDomain
TenantPlacementGeneration
TenantSchemaResolution
Shard Context
Read/Write Intent
Consistency Requirements
Authorization Scope
CrossTenantCapability
```

a través de:

```text
Model API
Repository
EntityManager
Query Builder
Query AST
Semantic Engine
Optimizer
Planner
Compiler
Executor
Relationship Loading
Pagination
Chunk Processing
Lazy Collections
Bulk Operations
Raw Queries
```

sin recurrir a:

```text
global current tenant
static tenant state
SQL string patching
implicit tenant fallbacks
```

bajo la regla:

> **Una query tenant-aware de VoltStack deberá transportar explícitamente su contexto de aislamiento desde el punto donde nace hasta el punto donde se ejecuta; ninguna etapa del Query Engine deberá “adivinar” el tenant consultando estado global mutable.**