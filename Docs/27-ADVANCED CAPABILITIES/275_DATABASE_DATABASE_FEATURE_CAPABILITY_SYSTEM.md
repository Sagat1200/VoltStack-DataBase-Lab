# 275_DATABASE_DATABASE_FEATURE_CAPABILITY_SYSTEM.md

# VoltStack Quantum Database
## Database Feature Capability System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 275 — Database Feature Capability System  
**Bloque:** 27 — Advanced Database Capabilities  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `274_DATABASE_GEOGRAPHIC_DATA_EXTENSION_SYSTEM.md`  
**Siguiente documento:** `276_DATABASE_BACKUP_ARCHITECTURE.md`

---

# 1. Propósito

Este documento define la arquitectura del **Database Feature Capability System** de VoltStack.

Su responsabilidad será determinar qué características están realmente disponibles en una:

```text
Database Platform
Connection
Endpoint
Server
Extension
Execution Context
```

sin distribuir comprobaciones específicas del proveedor por todo el framework.

La regla fundamental será:

> **VoltStack tomará decisiones mediante capacidades explícitas respaldadas por evidencia; el nombre del proveedor o su versión podrán aportar evidencia, pero nunca sustituirán por sí solos al modelo de capacidades.**

Por tanto:

```text
Database Vendor
+
Version
≠
Capability
```

y:

```text
UNKNOWN
≠
SUPPORTED
```

---

# 2. Problema arquitectónico

Sin un Capability System centralizado, diferentes componentes terminarían implementando lógica como:

```php
if ($driver === 'pgsql') {
    // ...
}

if ($database === 'mysql' && $version >= '8.0') {
    // ...
}

if ($database === 'sqlite') {
    // ...
}
```

Esto generaría:

```text
vendor coupling
version coupling
duplicated feature detection
incorrect assumptions
extension blindness
configuration blindness
poor portability
difficult testing
```

VoltStack deberá sustituirlo por:

```text
Feature Requirement
        ↓
Capability Resolver
        ↓
Capability Evidence
        ↓
Capability Decision
```

---

# 3. Relación con Platform Capability System

El documento:

```text
18_DATABASE_PLATFORM_CAPABILITY_SYSTEM.md
```

estableció el concepto base de capabilities de plataforma.

Este documento amplía ese modelo hacia un sistema completo de:

```text
feature discovery
feature evidence
feature constraints
runtime probing
extension capabilities
endpoint capabilities
capability negotiation
capability snapshots
capability caching
capability invalidation
```

---

# 4. Arquitectura general

```text
                 Database Feature Request
                           │
                           ▼
                 Capability Requirement
                           │
                           ▼
                  Capability Resolver
                           │
        ┌──────────────────┼───────────────────┐
        │                  │                   │
        ▼                  ▼                   ▼
 Platform Provider   Runtime Discovery   Extension Providers
        │                  │                   │
        └──────────────────┼───────────────────┘
                           ▼
                    Evidence Set
                           │
                           ▼
                Capability Evaluator
                           │
                           ▼
                 Capability Result
                           │
       ┌───────────────────┼───────────────────┐
       ▼                   ▼                   ▼
    Planner             Compiler           Diagnostics
```

---

# 5. Objetivos

El sistema deberá:

- centralizar feature detection;
- evitar vendor conditionals dispersos;
- representar capacidades tipadas;
- distinguir disponibilidad de limitaciones;
- soportar capabilities dependientes de versión;
- soportar capabilities dependientes de configuración;
- soportar extensiones opcionales;
- soportar runtime discovery;
- soportar endpoint-specific capabilities;
- soportar capabilities compuestas;
- proporcionar evidencia;
- representar incertidumbre;
- integrarse con Planner;
- integrarse con Compiler;
- integrarse con Schema;
- integrarse con ORM;
- integrarse con Migration;
- integrarse con Resilience;
- integrarse con distributed database;
- integrarse con Telemetry;
- funcionar correctamente bajo runtimes persistentes.

---

# 6. Capability

Una capability representa:

> Una propiedad funcional observable o razonablemente demostrable del sistema de base de datos relevante para una operación de VoltStack.

Ejemplos:

```text
query.returning
query.cte
query.recursive_cte
query.window_functions

transaction.savepoints
transaction.isolation.serializable

locking.for_update
locking.skip_locked
locking.nowait

schema.generated_columns
schema.expression_indexes
schema.transactional_ddl

json.query
json.path
json.contains

spatial.geometry
spatial.index
spatial.transform

fulltext.search
```

---

# 7. Capability ID

Las capabilities utilizarán identificadores estables.

Ejemplo:

```php
final readonly class CapabilityId
{
    public function __construct(
        public string $value
    ) {}
}
```

---

# 8. Naming

Formato recomendado:

```text
domain.feature
```

o:

```text
domain.subsystem.feature
```

Ejemplos:

```text
query.returning
query.cte.recursive
schema.index.partial
schema.index.expression
transaction.savepoint
locking.skip_locked
json.path
spatial.index
```

---

# 9. Capability ID ≠ vendor feature name

Evitar:

```text
mysql.returning
postgres.returning
```

como capability lógica.

Preferir:

```text
query.returning
```

y dejar que cada provider determine su soporte.

---

# 10. Capability descriptor

```php
final readonly class CapabilityDescriptor
{
    public function __construct(
        public CapabilityId $id,
        public CapabilityDomain $domain,
        public CapabilitySemantics $semantics,
    ) {}
}
```

---

# 11. Capability domain

Ejemplos:

```php
enum CapabilityDomain
{
    case QUERY;
    case TRANSACTION;
    case LOCKING;
    case SCHEMA;
    case MIGRATION;
    case JSON;
    case FULL_TEXT;
    case SPATIAL;
    case CONNECTION;
    case DISTRIBUTION;
    case SECURITY;
    case PERFORMANCE;
}
```

---

# 12. Capability status

VoltStack utilizará estados explícitos.

```php
enum CapabilityStatus
{
    case SUPPORTED;
    case SUPPORTED_WITH_LIMITATIONS;
    case REQUIRES_EXTENSION;
    case REQUIRES_EMULATION;
    case UNSUPPORTED;
    case UNKNOWN;
}
```

---

# 13. SUPPORTED

Significa:

```text
la capability requerida está disponible
bajo el contexto evaluado
con la semántica solicitada
```

---

# 14. SUPPORTED_WITH_LIMITATIONS

La característica existe, pero posee restricciones relevantes.

Ejemplos:

```text
limited data types
specific query shapes only
DDL restrictions
transaction restrictions
index restrictions
configuration requirements
```

---

# 15. REQUIRES_EXTENSION

La plataforma puede soportar la característica mediante una extensión que no está confirmada como disponible.

Ejemplo:

```text
PostgreSQL
+
PostGIS
```

---

# 16. REQUIRES_EMULATION

VoltStack puede reproducir la intención mediante otra estrategia.

Pero:

```text
native support
≠
emulated support
```

---

# 17. UNSUPPORTED

Existe evidencia suficiente de que la capability solicitada no puede proporcionarse.

---

# 18. UNKNOWN

No existe evidencia suficiente para afirmar:

```text
SUPPORTED
```

ni:

```text
UNSUPPORTED
```

---

# 19. Regla crítica

> **La ausencia de evidencia de incompatibilidad no constituye evidencia de compatibilidad.**

Por tanto:

```text
UNKNOWN
≠
SUPPORTED
```

---

# 20. Capability result

```php
final readonly class CapabilityResult
{
    public function __construct(
        public CapabilityId $capability,
        public CapabilityStatus $status,
        public CapabilityEvidenceSet $evidence,
        public CapabilityConstraintSet $constraints,
        public CapabilityConfidence $confidence,
    ) {}
}
```

---

# 21. Evidence

Cada decisión deberá poder explicar:

```text
por qué
```

la capability obtuvo determinado estado.

---

# 22. Evidence sources

```text
PLATFORM_DEFINITION
SERVER_VERSION
SERVER_CONFIGURATION
RUNTIME_PROBE
INSTALLED_EXTENSION
CONNECTION_METADATA
SCHEMA_METADATA
DRIVER_CAPABILITY
USER_CONFIGURATION
PLUGIN_PROVIDER
```

---

# 23. CapabilityEvidence

```php
final readonly class CapabilityEvidence
{
    public function __construct(
        public CapabilityEvidenceType $type,
        public CapabilityEvidenceSource $source,
        public mixed $value,
        public CapabilityConfidence $confidence,
    ) {}
}
```

---

# 24. Evidence ≠ decision

```text
Evidence
    ↓
Evaluator
    ↓
Decision
```

Un provider aporta evidencia.

El resolver produce la decisión.

---

# 25. Confidence

Podrá representarse:

```php
enum CapabilityConfidence
{
    case CERTAIN;
    case HIGH;
    case MEDIUM;
    case LOW;
    case UNKNOWN;
}
```

---

# 26. Confidence ≠ status

Una capability puede resultar:

```text
SUPPORTED
```

con distinta calidad de evidencia.

---

# 27. Capability constraints

Ejemplo:

```text
RETURNING supported
BUT
only INSERT
```

Esto no debe convertirse simplemente en:

```text
true
```

---

# 28. Constraint model

```php
interface CapabilityConstraint
{
}
```

Ejemplos:

```text
OperationConstraint
DataTypeConstraint
VersionConstraint
ConfigurationConstraint
TransactionConstraint
SchemaConstraint
ExpressionConstraint
ExtensionConstraint
```

---

# 29. Boolean capabilities are insufficient

Evitar como modelo principal:

```php
supportsReturning(): bool
```

cuando la capability puede tener restricciones.

Podrá existir como convenience API:

```php
$capabilities->supports(DatabaseCapabilities::QUERY_RETURNING);
```

pero internamente deberá conservar el resultado completo.

---

# 30. Feature requirement

Los componentes no deberán preguntar simplemente:

```text
¿Existe feature X?
```

Podrán expresar:

```text
Necesito feature X
con estas propiedades.
```

---

# 31. CapabilityRequirement

```php
final readonly class CapabilityRequirement
{
    public function __construct(
        public CapabilityId $id,
        public CapabilityConstraintSet $requiredConstraints,
        public CapabilityRequirementPolicy $policy,
    ) {}
}
```

---

# 32. Example

Una operación puede requerir:

```text
query.returning

operation:
    INSERT

columns:
    MULTIPLE

result:
    generated identifiers
```

---

# 33. Capability matching

```text
Capability
+
Constraints
+
Requirement
=
Compatibility Decision
```

---

# 34. Capability registry

Existirá:

```text
CapabilityRegistry
```

para conocer las capabilities definidas por VoltStack y extensiones.

---

# 35. Registry responsibility

```text
register capability descriptors
resolve capability IDs
validate duplicates
freeze registry
```

---

# 36. Registry ≠ runtime state

No deberá almacenar:

```text
current connection capability result
current server version
current tenant
current endpoint
```

---

# 37. Registry freeze

Después del bootstrap:

```text
CapabilityRegistry
→ FROZEN
```

---

# 38. Extension registration

Plugins podrán registrar capabilities adicionales antes del freeze.

---

# 39. Stable IDs

Una capability pública deberá mantener un ID estable.

---

# 40. Provider architecture

```text
CapabilityProvider
├── PlatformCapabilityProvider
├── DriverCapabilityProvider
├── RuntimeCapabilityProvider
├── ExtensionCapabilityProvider
├── ConnectionCapabilityProvider
└── UserConfiguredCapabilityProvider
```

---

# 41. Provider contract

```php
interface CapabilityProvider
{
    public function supports(CapabilityId $id): bool;

    public function evidence(
        CapabilityId $id,
        CapabilityContext $context,
    ): CapabilityEvidenceSet;
}
```

---

# 42. Provider does not decide everything

Un provider individual no necesariamente produce la decisión final.

Puede aportar sólo una pieza de evidencia.

---

# 43. Platform provider

Ejemplo:

```text
PostgreSQLPlatformCapabilityProvider
```

conoce propiedades estructurales conocidas de PostgreSQL.

---

# 44. MySQL provider

```text
MySQLPlatformCapabilityProvider
```

---

# 45. MariaDB provider

Separado:

```text
MariaDBPlatformCapabilityProvider
```

---

# 46. SQLite provider

```text
SQLitePlatformCapabilityProvider
```

---

# 47. No MySQL/MariaDB aliasing

Aunque compartan historia:

```text
MySQL
≠
MariaDB
```

a nivel de capabilities.

---

# 48. Driver capabilities

El driver puede imponer restricciones independientes del servidor.

Ejemplo:

```text
server supports feature
BUT
driver cannot expose required behavior
```

---

# 49. Effective capability

Conceptualmente:

```text
EffectiveCapability
=
Platform
∩
Server
∩
Driver
∩
Connection
∩
Configuration
∩
Extension
∩
Context
```

---

# 50. Runtime discovery

Algunas capabilities sólo pueden determinarse conectándose al servidor.

---

# 51. Examples

```text
installed extensions
server variables
SQL modes
configuration flags
runtime limits
enabled modules
```

---

# 52. Runtime probing

Debe estar controlado.

Nunca ejecutar probes arbitrarios en cada query.

---

# 53. Probe lifecycle

```text
Connection Established
        ↓
Capability Discovery
        ↓
Capability Snapshot
        ↓
Reuse Snapshot
```

---

# 54. Probe safety

Un probe deberá ser:

```text
read-only
bounded
non-destructive
timeout-aware
auditable
```

---

# 55. No destructive feature testing

Nunca determinar soporte haciendo algo como:

```text
create production table
alter schema
write data
```

sin un contexto explícitamente diseñado para testing.

---

# 56. Probe strategy

```php
enum CapabilityProbePolicy
{
    case DISABLED;
    case SAFE_ONLY;
    case EXPLICIT;
}
```

---

# 57. Default

Producción deberá utilizar:

```text
SAFE_ONLY
```

o una política igualmente conservadora.

---

# 58. Capability context

```php
final readonly class CapabilityContext
{
    public function __construct(
        public PlatformId $platform,
        public ?ServerVersion $serverVersion,
        public ?ConnectionId $connection,
        public ?EndpointId $endpoint,
        public ExtensionSet $extensions,
        public ConfigurationFingerprint $configuration,
    ) {}
}
```

---

# 59. Tenant

El tenant sólo deberá formar parte del CapabilityContext si realmente puede alterar la capability efectiva.

---

# 60. Avoid unnecessary cardinality

No incluir:

```text
tenant ID
request ID
user ID
```

en capability identity cuando no afecten el resultado.

---

# 61. Capability snapshot

Resultado consolidado:

```php
final readonly class CapabilitySnapshot
{
    /** @var array<string, CapabilityResult> */
    public array $capabilities;
}
```

---

# 62. Snapshot properties

Debe ser:

```text
immutable
versioned
context-bound
explainable
cacheable when safe
```

---

# 63. Snapshot identity

Puede depender de:

```text
platform
server version
endpoint
driver
extensions
configuration fingerprint
discovery generation
```

---

# 64. Snapshot ≠ global truth

Una capability snapshot describe:

```text
un contexto determinado
```

no todo el cluster.

---

# 65. Cluster heterogeneity

Ejemplo:

```text
writer
    PostgreSQL + PostGIS

replica A
    PostgreSQL + PostGIS

replica B
    PostgreSQL without PostGIS
```

---

# 66. Routing consequence

Una spatial query sólo podrá ir a:

```text
writer
replica A
```

si requiere PostGIS.

---

# 67. Endpoint capabilities

Cada endpoint podrá poseer su propio:

```text
CapabilitySnapshot
```

---

# 68. Load balancing

Primero:

```text
Capability Eligibility
```

después:

```text
Load Balancing
```

---

# 69. Critical routing rule

```text
Endpoint Health
≠
Endpoint Eligibility
```

Un endpoint puede estar sano y no soportar la query.

---

# 70. Failover

Un failover target deberá satisfacer:

```text
required capabilities
```

antes de ser considerado válido.

---

# 71. Capability requirement set

Una operación puede requerir varias capabilities.

```php
$requirements = CapabilityRequirementSet::of(
    DatabaseCapabilities::QUERY_CTE,
    DatabaseCapabilities::QUERY_CTE_RECURSIVE,
);
```

---

# 72. Requirement composition

```text
ALL_OF
ANY_OF
NONE_OF
```

---

# 73. Example ALL_OF

Una operación puede requerir:

```text
transaction.savepoint
AND
transaction.nested_scope
```

---

# 74. Example ANY_OF

Una estrategia puede aceptar:

```text
native_returning
OR
safe_generated_key_retrieval
```

---

# 75. Capability alternatives

Esto permite que el planner busque:

```text
Plan A
Plan B
Plan C
```

en lugar de preguntar sólo si existe una feature.

---

# 76. Native vs emulated

El planner deberá distinguir:

```text
NATIVE
EMULATED
DEGRADED
UNAVAILABLE
```

---

# 77. Capability support mode

```php
enum CapabilitySupportMode
{
    case NATIVE;
    case EMULATED;
    case DEGRADED;
    case NONE;
}
```

---

# 78. Emulation

Ejemplo conceptual:

```text
feature unavailable natively
+
safe alternative sequence
=
emulation
```

---

# 79. Emulation ≠ equivalence automatically

La emulación sólo será válida cuando preserve el contrato requerido.

---

# 80. Semantic equivalence

El planner deberá demostrar:

```text
EmulatedSemantics
⊇
RequiredSemantics
```

---

# 81. No silent semantic downgrade

Si una emulación pierde garantías:

```text
SUPPORTED_WITH_LIMITATIONS
```

o:

```text
UNSUPPORTED
```

según el requisito.

---

# 82. Query capabilities

Catálogo inicial:

```text
query.cte
query.cte.recursive
query.window_functions
query.returning
query.union
query.intersect
query.except
query.lateral_join
query.subquery
query.distinct_on
query.row_value_expression
query.nulls_ordering
query.limit
query.offset
```

---

# 83. DML capabilities

```text
dml.insert
dml.multi_row_insert
dml.upsert
dml.update_join
dml.delete_join
dml.returning
dml.bulk_insert
dml.bulk_update
dml.bulk_delete
```

---

# 84. Transaction capabilities

```text
transaction.basic
transaction.savepoint
transaction.release_savepoint
transaction.read_only
transaction.deferrable
transaction.isolation.read_uncommitted
transaction.isolation.read_committed
transaction.isolation.repeatable_read
transaction.isolation.serializable
```

---

# 85. Locking capabilities

```text
locking.for_update
locking.for_share
locking.skip_locked
locking.nowait
locking.of_clause
```

---

# 86. Schema capabilities

```text
schema.foreign_keys
schema.check_constraints
schema.generated_columns
schema.identity_columns
schema.sequences
schema.partial_indexes
schema.expression_indexes
schema.included_columns
schema.concurrent_index_creation
schema.transactional_ddl
schema.rename_column
schema.drop_column
schema.alter_column
```

---

# 87. JSON capabilities

```text
json.native_type
json.path
json.extract
json.contains
json.overlap
json.update
json.index
json.aggregate
```

---

# 88. Full-text capabilities

```text
fulltext.search
fulltext.ranking
fulltext.boolean
fulltext.phrase
fulltext.language
fulltext.index
```

---

# 89. Spatial capabilities

```text
spatial.geometry
spatial.geography
spatial.srid
spatial.transform
spatial.index
spatial.knn
spatial.distance.planar
spatial.distance.geodesic
spatial.geojson
```

---

# 90. Connection capabilities

```text
connection.prepared_statements
connection.server_side_prepare
connection.multiple_statements
connection.cancel
connection.timeout
connection.session_variables
connection.read_only
```

---

# 91. Driver capabilities

```text
driver.named_parameters
driver.positional_parameters
driver.stream_lob
driver.fetch_cursor
driver.async_cancel
```

---

# 92. Migration capabilities

```text
migration.online_index
migration.concurrent_index
migration.nonblocking_add_column
migration.transactional_ddl
migration.advisory_lock
```

---

# 93. Capability granularity

Evitar capabilities excesivamente generales:

```text
supportsAdvancedSQL
```

Preferir:

```text
query.cte
query.window_functions
query.lateral_join
```

---

# 94. But avoid explosion

Tampoco deberá modelarse cada sintaxis mínima como una capability independiente si no cambia decisiones arquitectónicas.

---

# 95. Capability criterion

Una capability merece identidad propia cuando afecta:

```text
planning
compilation
correctness
safety
routing
portability
```

---

# 96. Capability parameters

Algunas capabilities podrán aceptar parámetros.

Ejemplo:

```text
query.returning(operation=INSERT)
```

---

# 97. Parameterized requirement

```php
CapabilityRequirement::for(
    DatabaseCapabilities::QUERY_RETURNING
)->with(
    operation: QueryOperation::INSERT
);
```

---

# 98. Static platform knowledge

VoltStack podrá incluir conocimiento estático sobre plataformas soportadas.

Pero deberá mantenerse aislado en:

```text
PlatformCapabilityProvider
```

---

# 99. Version rules

Ejemplo conceptual:

```text
feature X supported since version Y
```

puede ser evidencia.

---

# 100. Version comparison

Debe utilizar un modelo específico:

```text
ServerVersion
```

no comparación lexicográfica de strings.

---

# 101. Bad

```php
if ($version >= '10.2') {
}
```

---

# 102. Good

```php
if ($version->isAtLeast(
    ServerVersion::of(10, 2)
)) {
}
```

dentro del provider correspondiente.

---

# 103. Version still not capability

Aunque una regla diga:

```text
feature introduced in version X
```

pueden existir:

```text
disabled extension
configuration
build differences
driver limitations
```

---

# 104. Configuration capabilities

Ejemplo:

```text
server supports feature
BUT
configuration disables it
```

El resultado efectivo debe reflejarlo.

---

# 105. Extension capabilities

Ejemplo:

```text
PostgreSQL
    ↓
Extension discovery
    ↓
PostGIS detected
    ↓
spatial.transform = SUPPORTED
```

---

# 106. Extension version

Una extensión podrá aportar su propia versión.

---

# 107. Extension version ≠ server version

```text
PostgreSQL Version
≠
PostGIS Version
```

---

# 108. Capability dependencies

Una capability puede depender de otra.

Ejemplo:

```text
spatial.knn
    requires
spatial.geometry
+
spatial.index
```

---

# 109. Capability dependency graph

```text
Capability A
├── requires B
├── requires C
└── optionally improves with D
```

---

# 110. Cycles

El registry deberá detectar ciclos inválidos.

---

# 111. Requirement graph

El evaluator podrá construir:

```text
CapabilityRequirementGraph
```

---

# 112. Capability implication

Puede existir:

```text
Capability A
⇒
Capability B
```

sólo cuando la implicación sea formalmente segura.

---

# 113. No heuristic implications

No asumir:

```text
window functions
⇒
all advanced SQL
```

---

# 114. Capability resolution

Pipeline:

```text
Requirement
    ↓
Descriptor Resolution
    ↓
Provider Selection
    ↓
Evidence Collection
    ↓
Constraint Evaluation
    ↓
Dependency Evaluation
    ↓
Policy Evaluation
    ↓
Capability Result
```

---

# 115. Resolver

```php
interface CapabilityResolver
{
    public function resolve(
        CapabilityRequirement $requirement,
        CapabilityContext $context,
    ): CapabilityResult;
}
```

---

# 116. Multiple requirements

```php
public function resolveSet(
    CapabilityRequirementSet $requirements,
    CapabilityContext $context,
): CapabilityResolution;
```

---

# 117. Resolution

```php
final readonly class CapabilityResolution
{
    public function isSatisfied(): bool;

    public function failures(): array;

    public function limitations(): array;

    public function evidence(): CapabilityEvidenceSet;
}
```

---

# 118. Planner integration

Planner pregunta:

```text
¿Qué capacidades necesito?
```

No:

```text
¿Estoy usando PostgreSQL?
```

---

# 119. Example

```php
$resolution = $capabilities->resolve(
    CapabilityRequirement::for(
        DatabaseCapabilities::LOCKING_SKIP_LOCKED
    ),
    $context
);
```

---

# 120. Planner decision

```text
SUPPORTED
    → native plan

SUPPORTED_WITH_LIMITATIONS
    → validate query shape

REQUIRES_EMULATION
    → alternative plan

UNSUPPORTED
    → reject

UNKNOWN
    → conservative policy
```

---

# 121. UNKNOWN policy

```php
enum UnknownCapabilityPolicy
{
    case REJECT;
    case PROBE_IF_SAFE;
    case REQUIRE_EXPLICIT_OVERRIDE;
}
```

---

# 122. Default

Para operaciones que afectan correctness:

```text
UNKNOWN
→ REJECT
```

o safe probe.

---

# 123. Performance-only capability

En ciertos casos:

```text
UNKNOWN
```

puede significar usar un plan más conservador.

---

# 124. Correctness vs optimization

Muy importante:

```text
Capability required for correctness
```

no se maneja igual que:

```text
Capability useful only for optimization
```

---

# 125. Requirement criticality

```php
enum CapabilityCriticality
{
    case REQUIRED_FOR_CORRECTNESS;
    case REQUIRED_FOR_SAFETY;
    case REQUIRED_FOR_SEMANTICS;
    case OPTIMIZATION_ONLY;
}
```

---

# 126. Example

Spatial exact predicate:

```text
REQUIRED_FOR_SEMANTICS
```

Spatial index:

```text
OPTIMIZATION_ONLY
```

---

# 127. Compiler integration

Compiler recibe un plan ya validado.

No debería redescubrir capabilities arbitrariamente.

---

# 128. Compile context

Puede contener:

```text
CapabilitySnapshot
```

para validar invariantes.

---

# 129. Compiler guard

Si recibe un nodo imposible:

```text
CompilerCapabilityException
```

en vez de generar SQL incorrecto.

---

# 130. Schema integration

Schema Compiler utilizará capabilities para decidir si una definición puede representarse.

---

# 131. Example

```text
Partial Index requested
        ↓
schema.index.partial
        ↓
SUPPORTED?
```

---

# 132. Schema compatibility

Resultado:

```text
SUPPORTED
SUPPORTED_WITH_LIMITATIONS
REQUIRES_EMULATION
UNSUPPORTED
UNKNOWN
```

---

# 133. Migration integration

Migration Planner utilizará capabilities para:

```text
transactional DDL
online index
concurrent index
rename support
alter-column support
```

---

# 134. Zero downtime

Capabilities operacionales pueden modificar el plan de migración.

---

# 135. Capability ≠ safety

Que una base soporte:

```text
ALTER TABLE
```

no significa que la operación sea:

```text
safe on a 2 TB production table
```

---

# 136. Integration with Migration Safety

```text
Capability
    ↓
Can operation exist?

Migration Safety
    ↓
Should operation run now?
```

---

# 137. ORM integration

ORM podrá requerir capabilities como:

```text
RETURNING
generated key retrieval
row locking
savepoints
```

---

# 138. Persistence Planner

Puede seleccionar:

```text
INSERT ... RETURNING
```

o:

```text
INSERT
+
generated key retrieval
```

dependiendo de capabilities.

---

# 139. ORM does not vendor-switch

Nunca:

```php
if ($entityManager->database() === 'postgres') {
}
```

---

# 140. Query optimizer integration

El Optimizer puede utilizar capabilities para aplicar reglas.

---

# 141. Example

```text
tuple comparison available
```

puede permitir optimizar Cursor Pagination.

---

# 142. But

La optimización sólo se aplica si:

```text
capability semantics
```

coinciden con:

```text
required ordering semantics
```

---

# 143. Query cache integration

Capability generation podrá formar parte de:

```text
CompiledQueryCache key
```

cuando el SQL compilado dependa de capabilities.

---

# 144. Critical cache rule

Un compiled query generado para:

```text
CapabilitySnapshot A
```

no deberá reutilizarse bajo:

```text
incompatible CapabilitySnapshot B
```

---

# 145. Capability fingerprint

```php
final readonly class CapabilityFingerprint
{
    public function __construct(
        public string $value
    ) {}
}
```

---

# 146. Fingerprint

Debe representar únicamente las capabilities relevantes para el artefacto.

---

# 147. Avoid full snapshot fingerprint everywhere

Eso produciría invalidaciones innecesarias.

---

# 148. Dependency-aware fingerprint

Ejemplo:

```text
CompiledQuery
depends on:
    query.returning
    query.nulls_ordering
```

Sólo esas capabilities necesitan afectar su identity.

---

# 149. Connection pooling

Al reutilizar una conexión:

```text
capability state
```

debe seguir siendo válido.

---

# 150. Connection reset

Si reset cambia configuración relevante:

```text
CapabilitySnapshot
```

deberá invalidarse o reevaluarse.

---

# 151. Session configuration

Ejemplo:

```text
session mode changed
```

puede alterar capability efectiva.

---

# 152. Connection generation

Podrá utilizarse:

```text
ConnectionGeneration
```

para asociar snapshots.

---

# 153. Persistent runtime

Bajo:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

no debe existir:

```php
static $currentCapabilities;
```

dependiente del request.

---

# 154. Safe shared state

Puede compartirse:

```text
immutable capability descriptors
static platform definitions
frozen registry
```

---

# 155. Scoped state

Debe ser scoped:

```text
connection-specific snapshots
runtime probe results
endpoint state
request overrides
```

---

# 156. Worker lifecycle

Capability caches de proceso deberán tener:

```text
bounded lifetime
generation awareness
invalidation
```

---

# 157. FrankenPHP

El worker puede sobrevivir múltiples requests.

Por tanto:

```text
request A capability override
```

nunca debe contaminar:

```text
request B
```

---

# 158. RoadRunner

Misma regla.

---

# 159. OpenSwoole

Además deberá considerarse concurrencia entre coroutines.

No usar mutable singleton context.

---

# 160. Capability overrides

Podrán existir para:

```text
testing
development
known infrastructure contracts
```

---

# 161. Override policy

Nunca deberán ser una forma silenciosa de mentir al planner.

---

# 162. Override types

```text
ASSERT_SUPPORTED
ASSERT_UNSUPPORTED
FORCE_UNKNOWN
```

---

# 163. Production override

Deberá ser:

```text
explicit
auditable
configuration-driven
```

---

# 164. Testing overrides

Ejemplo:

```php
$capabilities->override(
    DatabaseCapabilities::QUERY_RETURNING,
    CapabilityStatus::UNSUPPORTED
);
```

para probar fallback.

---

# 165. Testability

Todos los planners deberán poder probarse contra:

```text
synthetic CapabilitySnapshot
```

sin levantar un servidor real.

---

# 166. Capability matrix

VoltStack podrá mantener una matriz documental/testable.

Ejemplo conceptual:

| Capability | MySQL | MariaDB | PostgreSQL | SQLite |
|---|---|---|---|---|
| Recursive CTE | provider | provider | provider | provider |
| Window functions | provider | provider | provider | provider |
| RETURNING | provider | provider | provider | provider |
| Savepoints | provider | provider | provider | provider |
| JSON | provider | provider | provider | provider |
| Spatial | provider | provider | extension-aware | extension-aware |

La tabla documental no será la fuente de verdad de runtime.

---

# 167. Matrix ≠ resolver

La fuente operacional será:

```text
Capability Providers
+
Evidence
+
Context
```

---

# 168. Discovery cache

Runtime discovery puede ser costoso.

Por tanto:

```text
Discovery
    ↓
CapabilitySnapshot Cache
```

---

# 169. Cache scope

Dependiendo del dato:

```text
connection-local
endpoint-local
worker-local
shared infrastructure cache
```

---

# 170. TTL

TTL puede utilizarse.

Pero:

```text
TTL
≠
correctness proof
```

---

# 171. Invalidation events

Ejemplos:

```text
connection recreated
endpoint changed
server version changed
extension changed
configuration changed
deployment generation changed
```

---

# 172. Extension installation

Si se instala una extensión:

```text
CapabilityGeneration++
```

o mecanismo equivalente.

---

# 173. Capability generation

```php
final readonly class CapabilityGeneration
{
    public function __construct(
        public string|int $value
    ) {}
}
```

---

# 174. Staleness

Un snapshot puede ser:

```text
FRESH
STALE
UNKNOWN
```

---

# 175. Stale snapshot

No siempre implica unusable.

Depende de:

```text
capability criticality
policy
operation
```

---

# 176. Security

Runtime discovery deberá evitar filtrar:

```text
server configuration
extension inventory
server version
infrastructure topology
```

a usuarios no autorizados.

---

# 177. Public error

Puede decir:

```text
Required database capability is unavailable.
```

---

# 178. Internal diagnostics

Puede mostrar:

```text
query.returning
UNSUPPORTED
evidence:
    platform: ...
    version: ...
```

sólo bajo contexto autorizado.

---

# 179. Capability injection

Inputs externos no podrán seleccionar arbitrariamente:

```text
CapabilityProvider FQCN
```

---

# 180. Provider registration

Sólo mediante bootstrap/plugin registry autorizado.

---

# 181. Runtime probe security

Probes utilizarán:

```text
prepared/static safe queries
strict timeouts
known result shapes
```

---

# 182. No arbitrary SQL probe

Plugins deberán cumplir el contrato de seguridad correspondiente.

---

# 183. Resource governance

Discovery tendrá presupuestos:

```text
max probes
probe timeout
max discovery duration
max result size
```

---

# 184. Discovery storm

En persistent workers o pools grandes debe evitarse:

```text
every request
×
every connection
×
every capability
```

---

# 185. Lazy discovery

Podrá resolverse una capability sólo cuando sea necesaria.

---

# 186. Eager discovery

Podrá usarse para un conjunto pequeño de capabilities fundamentales.

---

# 187. Hybrid strategy

Recomendada:

```text
Bootstrap:
    static capabilities

Connection establishment:
    essential runtime capabilities

On demand:
    specialized capabilities
```

---

# 188. Capability discovery levels

```php
enum CapabilityDiscoveryLevel
{
    case STATIC;
    case CONNECTION;
    case EXTENSION;
    case DEEP;
}
```

---

# 189. Deep discovery

Sólo explícito cuando sea necesario.

---

# 190. Telemetry

Eventos conceptuales:

```text
CapabilityResolutionStarted
CapabilityEvidenceCollected
CapabilityResolved
CapabilityProbeExecuted
CapabilityProbeFailed
CapabilitySnapshotCreated
CapabilitySnapshotInvalidated
CapabilityFallbackSelected
```

---

# 191. Avoid telemetry explosion

No emitir evento por cada consulta de capability si el resultado proviene de snapshot cache.

---

# 192. Metrics

```text
database_capability_resolution_total
database_capability_unknown_total
database_capability_probe_total
database_capability_probe_failures_total
database_capability_fallback_total
database_capability_snapshot_invalidations_total
```

---

# 193. Labels

Permitidas:

```text
platform
capability_family
status
evidence_type
```

---

# 194. Avoid capability ID cardinality blindly

Si el catálogo crece dinámicamente, capability ID como label deberá estar controlado.

---

# 195. No endpoint IDs

Evitar:

```text
host
IP
connection ID
tenant ID
```

como labels.

---

# 196. Debug information

Developer diagnostics podrán mostrar:

```text
Capability:
    query.returning

Status:
    SUPPORTED_WITH_LIMITATIONS

Context:
    platform: ...
    driver: ...

Evidence:
    platform definition
    runtime version
    configuration

Constraints:
    INSERT only

Confidence:
    CERTAIN
```

---

# 197. Capability explain

API conceptual:

```php
DB::capabilities()->explain(
    DatabaseCapabilities::QUERY_RETURNING
);
```

---

# 198. Explain result

```text
Capability Resolution
────────────────────────────────

Capability:
  query.returning

Status:
  SUPPORTED

Support:
  NATIVE

Evidence:
  platform       confirmed
  version        compatible
  driver         compatible

Constraints:
  none

Confidence:
  CERTAIN

Snapshot:
  generation 17
```

---

# 199. Explain ≠ probe

Solicitar diagnostics no deberá ejecutar automáticamente probes destructivos.

---

# 200. CLI

Podrá existir:

```text
volt database:capabilities
```

---

# 201. CLI output

```text
Database Capabilities

Query
  CTE                  supported
  Recursive CTE        supported
  Window Functions     supported
  RETURNING            supported

Transactions
  Savepoints           supported
  Serializable         supported

JSON
  Native JSON          supported
  JSON Path            supported

Spatial
  Geometry             supported
  Transform            requires extension
```

---

# 202. Detailed CLI

```text
volt database:capabilities --explain spatial.transform
```

---

# 203. Health checks

Health System podrá verificar:

```text
required production capabilities
```

---

# 204. Example

Una aplicación puede declarar:

```text
required:
    query.cte
    transaction.savepoint
    json.native_type
```

---

# 205. Deployment readiness

```text
RequiredCapabilities
        ↓
CapabilityResolver
        ↓
DeploymentCompatibilityReport
```

---

# 206. Missing required capability

Puede hacer fallar:

```text
application boot validation
deployment health check
```

si la política así lo exige.

---

# 207. Optional capability

Puede activar una estrategia alternativa.

---

# 208. Application requirements

Configuración conceptual:

```php
'database' => [
    'required_capabilities' => [
        'transaction.savepoint',
    ],

    'preferred_capabilities' => [
        'query.returning',
    ],
];
```

---

# 209. Required ≠ preferred

```text
REQUIRED missing
→ failure

PREFERRED missing
→ fallback
```

---

# 210. Portability profiles

Podrán definirse perfiles:

```text
PORTABLE_CORE
ADVANCED_SQL
JSON_REQUIRED
SPATIAL_REQUIRED
```

---

# 211. Profile ≠ platform

Un profile describe requisitos de aplicación.

---

# 212. Example

```text
PORTABLE_CORE
    requires capabilities common to intended supported platforms
```

---

# 213. Database portability

El futuro documento:

```text
327_DATABASE_DATABASE_PORTABILITY_SYSTEM.md
```

utilizará este Capability System como base.

---

# 214. Extension System integration

Plugins podrán aportar:

```text
new capabilities
new evidence providers
new emulation strategies
new extension discovery
```

---

# 215. Plugin isolation

Un plugin no deberá modificar silenciosamente la semántica de una capability core.

---

# 216. Namespaces

Custom capabilities deberán utilizar namespace lógico.

Ejemplo:

```text
vendor.package.feature
```

---

# 217. Core namespace protection

Plugins no podrán registrar:

```text
query.returning
```

si ya pertenece al core.

---

# 218. Capability aliases

Sólo para:

```text
deprecation
migration
compatibility
```

---

# 219. Versioning

Capability IDs públicos estarán sujetos a política de compatibilidad.

---

# 220. Deprecation

Si una capability se divide:

```text
old.feature
```

en:

```text
new.feature.a
new.feature.b
```

deberá existir transición documentada.

---

# 221. No silent semantic mutation

Una capability existente no deberá cambiar radicalmente de significado sin versionado/deprecación.

---

# 222. Error architecture

```text
DatabaseException
└── CapabilityException
    ├── CapabilityNotFoundException
    ├── CapabilityUnsupportedException
    ├── CapabilityUnknownException
    ├── CapabilityConstraintException
    ├── CapabilityDiscoveryException
    ├── CapabilityProbeException
    ├── CapabilityConflictException
    ├── CapabilityDependencyException
    ├── CapabilitySnapshotException
    ├── CapabilityProviderException
    └── CapabilitySecurityException
```

---

# 223. NotFound

Capability ID desconocida.

---

# 224. Unsupported

Capability conocida pero no soportada.

---

# 225. Unknown

No pudo determinarse.

---

# 226. Constraint

Feature disponible pero no satisface los requisitos concretos.

---

# 227. Discovery

Falló discovery general.

---

# 228. Probe

Falló un probe específico.

---

# 229. Conflict

Providers entregaron evidencia incompatible.

---

# 230. Dependency

No puede satisfacerse una dependencia.

---

# 231. Snapshot

Snapshot inválido/stale/incompatible.

---

# 232. Provider

Provider defectuoso.

---

# 233. Security

Intento de discovery/override no autorizado.

---

# 234. Evidence conflict

Ejemplo:

```text
static provider:
    SUPPORTED

runtime probe:
    feature disabled
```

---

# 235. Runtime evidence priority

El resolver deberá aplicar reglas explícitas.

No simplemente:

```text
first provider wins
```

---

# 236. Evidence precedence

Modelo conceptual:

```text
Direct Runtime Evidence
        >
Connection Evidence
        >
Configuration Evidence
        >
Extension Evidence
        >
Platform/Version Inference
```

cuando la evidencia sea comparable y confiable.

---

# 237. But

La precedencia exacta dependerá del tipo de capability.

---

# 238. Negative evidence

Una prueba directa de que una feature está deshabilitada puede superar una inferencia por versión.

---

# 239. Contradiction

Si la contradicción no puede resolverse:

```text
UNKNOWN
```

con diagnostics.

---

# 240. No optimistic merge

Nunca:

```text
SUPPORTED + UNKNOWN = SUPPORTED
```

automáticamente.

---

# 241. Capability evaluator

```php
interface CapabilityEvaluator
{
    public function evaluate(
        CapabilityRequirement $requirement,
        CapabilityEvidenceSet $evidence,
        CapabilityContext $context,
    ): CapabilityResult;
}
```

---

# 242. Determinism

Mismo:

```text
requirement
evidence
context
policy
```

debe producir el mismo resultado.

---

# 243. Side-effect free evaluator

El evaluator no hará I/O.

---

# 244. Discovery does I/O

Separación:

```text
Discovery
→ evidence

Evaluation
→ decision
```

---

# 245. Architecture purity

```text
CapabilityRegistry
    definitions

CapabilityProvider
    evidence

CapabilityDiscovery
    I/O

CapabilityEvaluator
    semantics

CapabilityResolver
    orchestration

CapabilitySnapshot
    immutable result
```

---

# 246. Proposed directory structure

```text
src/Quantum/Database/Capability/
│
├── Contract/
│   ├── CapabilityProvider.php
│   ├── CapabilityEvaluator.php
│   ├── CapabilityResolver.php
│   └── CapabilityProbe.php
│
├── Definition/
│   ├── CapabilityId.php
│   ├── CapabilityDescriptor.php
│   ├── CapabilityDomain.php
│   ├── DatabaseCapabilities.php
│   └── CapabilityRegistry.php
│
├── Requirement/
│   ├── CapabilityRequirement.php
│   ├── CapabilityRequirementSet.php
│   ├── CapabilityRequirementPolicy.php
│   ├── CapabilityCriticality.php
│   └── CapabilityComposition.php
│
├── Result/
│   ├── CapabilityResult.php
│   ├── CapabilityStatus.php
│   ├── CapabilitySupportMode.php
│   ├── CapabilityConfidence.php
│   └── CapabilityResolution.php
│
├── Constraint/
│   ├── CapabilityConstraint.php
│   ├── CapabilityConstraintSet.php
│   ├── OperationConstraint.php
│   ├── VersionConstraint.php
│   ├── ConfigurationConstraint.php
│   ├── TransactionConstraint.php
│   └── ExtensionConstraint.php
│
├── Evidence/
│   ├── CapabilityEvidence.php
│   ├── CapabilityEvidenceSet.php
│   ├── CapabilityEvidenceType.php
│   └── CapabilityEvidenceSource.php
│
├── Context/
│   ├── CapabilityContext.php
│   ├── CapabilityFingerprint.php
│   ├── CapabilityGeneration.php
│   └── ConfigurationFingerprint.php
│
├── Snapshot/
│   ├── CapabilitySnapshot.php
│   ├── CapabilitySnapshotCache.php
│   ├── CapabilitySnapshotState.php
│   └── CapabilitySnapshotInvalidator.php
│
├── Discovery/
│   ├── CapabilityDiscovery.php
│   ├── CapabilityDiscoveryLevel.php
│   ├── CapabilityProbePolicy.php
│   ├── CapabilityProbeRunner.php
│   └── CapabilityDiscoveryBudget.php
│
├── Dependency/
│   ├── CapabilityDependency.php
│   ├── CapabilityDependencyGraph.php
│   └── CapabilityDependencyResolver.php
│
├── Provider/
│   ├── PlatformCapabilityProvider.php
│   ├── DriverCapabilityProvider.php
│   ├── ConnectionCapabilityProvider.php
│   ├── RuntimeCapabilityProvider.php
│   └── ExtensionCapabilityProvider.php
│
├── Platform/
│   ├── MySQLCapabilityProvider.php
│   ├── MariaDBCapabilityProvider.php
│   ├── PostgreSQLCapabilityProvider.php
│   └── SQLiteCapabilityProvider.php
│
├── Override/
│   ├── CapabilityOverride.php
│   ├── CapabilityOverrideRegistry.php
│   └── CapabilityOverridePolicy.php
│
├── Diagnostics/
│   ├── CapabilityDiagnostics.php
│   ├── CapabilityExplain.php
│   └── CapabilityFormatter.php
│
├── Telemetry/
│   └── CapabilityTelemetry.php
│
└── Exception/
    ├── CapabilityException.php
    ├── CapabilityNotFoundException.php
    ├── CapabilityUnsupportedException.php
    ├── CapabilityUnknownException.php
    ├── CapabilityConstraintException.php
    ├── CapabilityDiscoveryException.php
    ├── CapabilityProbeException.php
    ├── CapabilityConflictException.php
    ├── CapabilityDependencyException.php
    ├── CapabilitySnapshotException.php
    ├── CapabilityProviderException.php
    └── CapabilitySecurityException.php
```

---

# 247. Public API

Convenience:

```php
if (
    DB::capabilities()->supports(
        DatabaseCapabilities::QUERY_RETURNING
    )
) {
    // ...
}
```

---

# 248. Preferred internal API

Internamente:

```php
$result = $capabilities->resolve(
    CapabilityRequirement::for(
        DatabaseCapabilities::QUERY_RETURNING
    ),
    $context
);
```

---

# 249. Why

Porque:

```text
supports(): bool
```

pierde:

```text
limitations
evidence
confidence
emulation
constraints
unknown state
```

---

# 250. Architecture invariants

## DB-CAP-001

Capability será una abstracción explícita.

## DB-CAP-002

Vendor será distinto de capability.

## DB-CAP-003

Version será distinta de capability.

## DB-CAP-004

Extension será distinta de capability.

## DB-CAP-005

Evidence será distinta de decision.

## DB-CAP-006

UNKNOWN será distinto de SUPPORTED.

## DB-CAP-007

UNKNOWN será distinto de UNSUPPORTED.

## DB-CAP-008

SUPPORTED_WITH_LIMITATIONS será distinto de SUPPORTED.

## DB-CAP-009

REQUIRES_EXTENSION será representable.

## DB-CAP-010

REQUIRES_EMULATION será representable.

## DB-CAP-011

Native support será distinto de emulated support.

## DB-CAP-012

Capability ID será estable.

## DB-CAP-013

Capability ID no incluirá vendor por default.

## DB-CAP-014

Capabilities tendrán dominio.

## DB-CAP-015

Capabilities podrán tener constraints.

## DB-CAP-016

Boolean support no será el modelo interno completo.

## DB-CAP-017

Requirements podrán expresar constraints.

## DB-CAP-018

Capability Registry contendrá definiciones.

## DB-CAP-019

Registry no contendrá runtime connection state.

## DB-CAP-020

Registry será congelable.

## DB-CAP-021

Plugins registrarán capabilities antes del freeze.

## DB-CAP-022

Core capability IDs estarán protegidos.

## DB-CAP-023

Providers aportarán evidencia.

## DB-CAP-024

Providers no serán necesariamente la decisión final.

## DB-CAP-025

MySQL tendrá provider propio.

## DB-CAP-026

MariaDB tendrá provider propio.

## DB-CAP-027

PostgreSQL tendrá provider propio.

## DB-CAP-028

SQLite tendrá provider propio.

## DB-CAP-029

MySQL no será alias de MariaDB.

## DB-CAP-030

Driver capabilities serán independientes de server capabilities.

## DB-CAP-031

Effective capability considerará toda restricción relevante.

## DB-CAP-032

Runtime discovery será bounded.

## DB-CAP-033

Runtime probes serán read-only por default.

## DB-CAP-034

Runtime probes serán timeout-aware.

## DB-CAP-035

Runtime probes no modificarán schema por default.

## DB-CAP-036

Runtime probes no modificarán datos por default.

## DB-CAP-037

Probe policy será explícita.

## DB-CAP-038

CapabilityContext será immutable.

## DB-CAP-039

CapabilitySnapshot será immutable.

## DB-CAP-040

Snapshot estará ligado a contexto.

## DB-CAP-041

Snapshot no representará global cluster truth.

## DB-CAP-042

Endpoints podrán tener snapshots diferentes.

## DB-CAP-043

Endpoint health será distinto de capability eligibility.

## DB-CAP-044

Capability eligibility precederá load balancing.

## DB-CAP-045

Failover target deberá satisfacer capabilities requeridas.

## DB-CAP-046

Requirement sets podrán usar ALL_OF.

## DB-CAP-047

Requirement sets podrán usar ANY_OF.

## DB-CAP-048

Requirement sets podrán expresar alternativas.

## DB-CAP-049

Planner podrá seleccionar fallback.

## DB-CAP-050

Fallback no degradará semántica silenciosamente.

## DB-CAP-051

Emulation requerirá equivalencia suficiente.

## DB-CAP-052

Capability required for correctness será conservadora.

## DB-CAP-053

Capability optimization-only podrá usar fallback conservador.

## DB-CAP-054

Planner no preguntará vendor cuando necesite capability.

## DB-CAP-055

Compiler no hará discovery arbitrario.

## DB-CAP-056

Compiler podrá validar snapshot.

## DB-CAP-057

Schema utilizará capabilities.

## DB-CAP-058

Migration utilizará capabilities.

## DB-CAP-059

Capability será distinta de migration safety.

## DB-CAP-060

ORM utilizará capabilities.

## DB-CAP-061

ORM no tendrá vendor switches.

## DB-CAP-062

Optimizer utilizará capabilities.

## DB-CAP-063

Optimizer preservará semantics.

## DB-CAP-064

Compiled Query Cache será capability-aware.

## DB-CAP-065

Incompatible capability snapshot no reutilizará compiled query.

## DB-CAP-066

Capability fingerprint será dependency-aware.

## DB-CAP-067

Connection reuse preservará snapshot validity.

## DB-CAP-068

Session changes podrán invalidar capabilities.

## DB-CAP-069

Persistent workers no usarán mutable static current capabilities.

## DB-CAP-070

Frozen descriptors podrán compartirse.

## DB-CAP-071

Connection snapshots serán scoped.

## DB-CAP-072

Request overrides serán scoped.

## DB-CAP-073

OpenSwoole tendrá coroutine isolation.

## DB-CAP-074

Overrides serán explícitos.

## DB-CAP-075

Production overrides serán auditables.

## DB-CAP-076

Testing podrá usar synthetic snapshots.

## DB-CAP-077

Capability matrix documental no será runtime truth.

## DB-CAP-078

Discovery cache tendrá invalidation.

## DB-CAP-079

TTL no será correctness proof.

## DB-CAP-080

Extension changes invalidarán evidencia relevante.

## DB-CAP-081

Snapshot staleness será representable.

## DB-CAP-082

Sensitive infrastructure evidence podrá redactarse.

## DB-CAP-083

External input no elegirá provider FQCN.

## DB-CAP-084

Provider registration será controlado.

## DB-CAP-085

Probe resource budgets serán obligatorios.

## DB-CAP-086

Discovery storms deberán evitarse.

## DB-CAP-087

Lazy discovery estará permitido.

## DB-CAP-088

Eager discovery estará limitado.

## DB-CAP-089

Telemetry será bounded.

## DB-CAP-090

Capability resolution cache hits no producirán event storm.

## DB-CAP-091

Endpoint IDs no serán metrics labels.

## DB-CAP-092

Diagnostics explicarán evidencia.

## DB-CAP-093

Explain no ejecutará destructive probes.

## DB-CAP-094

CLI podrá inspeccionar capabilities.

## DB-CAP-095

Health checks podrán validar required capabilities.

## DB-CAP-096

Required será distinto de preferred.

## DB-CAP-097

Missing required capability podrá bloquear deployment.

## DB-CAP-098

Missing preferred capability podrá seleccionar fallback.

## DB-CAP-099

Portability profiles describirán requirements.

## DB-CAP-100

Portability profile será distinto de platform.

## DB-CAP-101

Plugins podrán añadir capabilities.

## DB-CAP-102

Plugins no redefinirán core semantics silenciosamente.

## DB-CAP-103

Capability IDs estarán versionados mediante compatibility policy.

## DB-CAP-104

Capability semantics no cambiarán silenciosamente.

## DB-CAP-105

Evidence conflicts serán explícitos.

## DB-CAP-106

First provider wins no será política general.

## DB-CAP-107

Runtime evidence podrá superar version inference.

## DB-CAP-108

Negative evidence será considerada.

## DB-CAP-109

Unresolved contradiction producirá UNKNOWN.

## DB-CAP-110

Evaluator será deterministic.

## DB-CAP-111

Evaluator será side-effect free.

## DB-CAP-112

Discovery y evaluation estarán separados.

## DB-CAP-113

Capability graph detectará ciclos.

## DB-CAP-114

Capability implication requerirá regla explícita.

## DB-CAP-115

No existirán heuristic implications.

## DB-CAP-116

Capability granularity seguirá decisiones arquitectónicas.

## DB-CAP-117

No existirá capability genérica `advanced_sql`.

## DB-CAP-118

Parameterized requirements estarán soportados.

## DB-CAP-119

ServerVersion será value object.

## DB-CAP-120

Version strings no se compararán lexicográficamente.

## DB-CAP-121

Installed extension será evidence.

## DB-CAP-122

Extension version será distinta de server version.

## DB-CAP-123

Capability dependencies serán explícitas.

## DB-CAP-124

CapabilitySnapshot tendrá generation/fingerprint.

## DB-CAP-125

Snapshot generation podrá invalidarse.

## DB-CAP-126

Connection recreation podrá invalidar snapshot.

## DB-CAP-127

Configuration changes podrán invalidar snapshot.

## DB-CAP-128

Capability resolution será explainable.

## DB-CAP-129

Capability result conservará evidence.

## DB-CAP-130

Capability result conservará constraints.

## DB-CAP-131

Capability result conservará confidence.

## DB-CAP-132

Confidence será distinta de status.

## DB-CAP-133

Low confidence support podrá ser rechazado por policy.

## DB-CAP-134

Critical operations podrán requerir CERTAIN/HIGH evidence.

## DB-CAP-135

Optimization-only features podrán tolerar UNKNOWN mediante fallback.

## DB-CAP-136

Capability discovery no dependerá de ORM.

## DB-CAP-137

Capability system no dependerá de HTTP.

## DB-CAP-138

Capability system no dependerá de Multitenancy.

## DB-CAP-139

Multitenancy podrá aportar contexto cuando sea relevante.

## DB-CAP-140

Capability system no generará SQL de aplicación.

## DB-CAP-141

Capability probes utilizarán infrastructure-specific safe queries.

## DB-CAP-142

Probe failure no será automáticamente UNSUPPORTED.

## DB-CAP-143

Probe failure podrá producir UNKNOWN.

## DB-CAP-144

Timeout de probe no será evidence de unsupported.

## DB-CAP-145

Permission denied durante probe no demostrará ausencia de feature.

## DB-CAP-146

Capability security será separada de capability existence.

## DB-CAP-147

A user lacking permission no implica platform unsupported.

## DB-CAP-148

Permission capability podrá modelarse separadamente.

## DB-CAP-149

Capabilities de cluster no se inferirán desde un único endpoint.

## DB-CAP-150

Distributed planning evaluará capabilities por endpoint/shard.

## DB-CAP-151

Cross-shard plan no asumirá homogeneidad.

## DB-CAP-152

Replica routing será capability-aware.

## DB-CAP-153

Read/write routing será capability-aware.

## DB-CAP-154

Failover será capability-aware.

## DB-CAP-155

Load balancing será posterior a capability filtering.

## DB-CAP-156

Cache será capability-generation-aware.

## DB-CAP-157

Capability cache no será database truth eterna.

## DB-CAP-158

Capability changes podrán invalidar query plans.

## DB-CAP-159

Capability changes podrán invalidar compiled SQL.

## DB-CAP-160

Capability changes podrán invalidar migration plans.

## DB-CAP-161

Capability changes no invalidarán entidades ORM arbitrariamente.

## DB-CAP-162

Capability telemetry no expondrá secrets.

## DB-CAP-163

Capability diagnostics respetarán security/redaction.

## DB-CAP-164

Capability overrides no persistirán entre requests accidentalmente.

## DB-CAP-165

Capability state será reset-safe.

## DB-CAP-166

Capability snapshots podrán ser reused sólo bajo contexto compatible.

## DB-CAP-167

Snapshot compatibility será verificable.

## DB-CAP-168

Capability evaluator no accederá directamente al Driver.

## DB-CAP-169

Capability provider no ejecutará business queries.

## DB-CAP-170

Capability system será extensible sin modificar core planners.

## DB-CAP-171

Planner declarará requirements antes de compilation.

## DB-CAP-172

Capability mismatch fallará antes de ejecución cuando sea posible.

## DB-CAP-173

Runtime mismatch detectado tardíamente seguirá siendo error explícito.

## DB-CAP-174

VoltStack no fabricará portability mediante semantic downgrade.

## DB-CAP-175

VoltStack preferirá capability checks sobre vendor checks.

## DB-CAP-176

Capability System será fuente central de feature availability.

---

# 251. Modelo formal

Sea:

```text
F
```

una feature requerida.

Sea:

```text
C
```

el contexto efectivo:

```text
C =
(
    platform,
    server,
    version,
    driver,
    connection,
    configuration,
    extensions,
    endpoint
)
```

Sea:

```text
E(F,C)
```

el conjunto de evidencia disponible.

Entonces:

```text
CapabilityResult(F,C)
=
Evaluate(
    F,
    E(F,C),
    Constraints(F),
    Policy
)
```

---

# 252. Effective capability

Conceptualmente:

```text
EffectiveSupport(F)
=
PlatformSupport
∩
ServerSupport
∩
DriverSupport
∩
ConfigurationSupport
∩
ExtensionSupport
∩
ContextCompatibility
```

No necesariamente todos los términos aplican a cada capability.

---

# 253. Requirement satisfaction

Sea:

```text
R = (F, Cr, P)
```

donde:

```text
F  = feature
Cr = required constraints
P  = requirement policy
```

Entonces:

```text
Satisfied(R,C)
```

sólo será verdadero si el resultado efectivo satisface:

```text
feature semantics
+
required constraints
+
criticality policy
+
confidence policy
```

---

# 254. Planner model

```text
Operation
    ↓
Requirements(Operation)
    ↓
Capability Resolution
    ↓
Candidate Plans
    ↓
Valid Plans
    ↓
Cost / Safety Selection
    ↓
Selected Plan
```

---

# 255. Important consequence

El Capability System no decide:

```text
qué plan es mejor
```

Determina:

```text
qué capacidades están disponibles
y bajo qué condiciones
```

El Query Planner toma la decisión física.

---

# 256. Capability System ≠ Query Planner

```text
Capability System
    answers what can be done

Planner
    decides how to do it
```

---

# 257. Capability System ≠ Platform

```text
Platform
    describes database family semantics

Capability System
    resolves effective feature availability
```

---

# 258. Capability System ≠ Driver

```text
Driver
    communicates with database

Capability System
    models available behavior
```

---

# 259. Capability System ≠ Configuration

Configuration puede aportar:

```text
policy
overrides
expected infrastructure contracts
```

pero no reemplaza discovery/evidence.

---

# 260. Capability System ≠ Feature flags

Una application feature flag:

```text
enable_new_checkout
```

no pertenece aquí.

Este sistema modela:

```text
database capabilities
```

---

# 261. Final architecture

```text
                       VoltStack Database
                              │
                              ▼
                     Feature Requirement
                              │
                              ▼
                     Capability Resolver
                              │
          ┌───────────────────┼────────────────────┐
          │                   │                    │
          ▼                   ▼                    ▼
      Registry             Context             Providers
          │                   │                    │
          │        ┌──────────┼──────────┐         │
          │        ▼          ▼          ▼         │
          │     Platform   Endpoint   Connection    │
          │                                        │
          │       ┌────────────────────────────┐   │
          │       │ Evidence Providers         │   │
          │       ├────────────────────────────┤   │
          │       │ Platform                   │   │
          │       │ Driver                     │   │
          │       │ Server Version             │   │
          │       │ Runtime Discovery          │   │
          │       │ Configuration              │   │
          │       │ Extensions                 │   │
          │       └──────────────┬─────────────┘   │
          │                      │                 │
          └──────────────────────┼─────────────────┘
                                 ▼
                           Evidence Set
                                 │
                                 ▼
                       Capability Evaluator
                                 │
                                 ▼
                         Capability Result
                                 │
               ┌─────────────────┼──────────────────┐
               │                 │                  │
               ▼                 ▼                  ▼
            Status           Constraints         Evidence
               │                 │                  │
               └─────────────────┼──────────────────┘
                                 ▼
                       Capability Snapshot
                                 │
           ┌─────────────────────┼──────────────────────┐
           │                     │                      │
           ▼                     ▼                      ▼
      Query Planner         Schema Planner       Migration Planner
           │                     │                      │
           ▼                     ▼                      ▼
       Compiler              Compiler              Executor
```

---

# 262. Example: RETURNING

Application operation:

```text
Insert entity
+
obtain generated values
```

Persistence Planner expresa:

```text
Requirement:
    query.returning

Operation:
    INSERT
```

Resolver:

```text
Platform Evidence
+
Version Evidence
+
Driver Evidence
+
Connection Context
```

produce:

```text
SUPPORTED
```

Entonces:

```text
Persistence Planner
        ↓
INSERT ... RETURNING strategy
```

Si:

```text
UNSUPPORTED
```

puede buscar:

```text
GeneratedKeyRetrievalStrategy
```

Si ambas faltan:

```text
PersistencePlanningException
```

---

# 263. Example: spatial query

Query:

```text
nearest stores to location
```

Requirements:

```text
spatial.geometry
spatial.distance.geodesic
```

Preferred:

```text
spatial.index
spatial.knn
```

Resultado:

```text
geometry             SUPPORTED
geodesic distance    SUPPORTED
spatial index        SUPPORTED
KNN                   UNKNOWN
```

El planner puede producir:

```text
exact indexed distance plan
```

sin depender de KNN.

---

# 264. Example: migration

Migration:

```text
create index without long blocking
```

Requirements:

```text
schema.index
```

Preferred:

```text
schema.concurrent_index_creation
```

Si:

```text
concurrent index = UNSUPPORTED
```

Migration Planner no afirmará que la migración sea imposible.

Podrá producir:

```text
normal index creation
```

pero Migration Safety puede posteriormente decidir:

```text
unsafe for current production environment
```

---

# 265. Core rule

> **Capability responde “¿puede esta infraestructura proporcionar esta semántica?”, mientras que Planning responde “¿qué estrategia utilizaremos?” y Safety responde “¿es aceptable ejecutar esa estrategia en este contexto?”.**

---

# 266. Regla de evidencia

> **VoltStack nunca convertirá una inferencia débil en una certeza silenciosa. Una capability deberá conservar la evidencia, limitaciones y nivel de confianza que justificaron su resolución.**

---

# 267. Regla de portabilidad

> **La portabilidad de VoltStack no dependerá de fingir que MySQL, MariaDB, PostgreSQL y SQLite poseen las mismas características. Dependerá de expresar requisitos comunes y resolverlos contra las capacidades efectivas de cada plataforma.**

---

# 268. Regla de extensibilidad

> **Una extensión podrá ampliar las capacidades disponibles de una plataforma, pero no deberá contaminar el núcleo con dependencias obligatorias ni modificar silenciosamente la semántica de las capabilities existentes.**

---

# 269. Regla de runtime persistente

> **Las definiciones inmutables de capabilities pueden compartirse entre workers y requests; los resultados dependientes de conexión, endpoint, configuración o runtime deberán permanecer correctamente aislados, versionados e invalidados.**

---

# 270. Resultado del Bloque 27

Con este documento queda cerrado:

```text
BLOCK 27 — ADVANCED DATABASE CAPABILITIES
```

La arquitectura completa queda:

```text
Advanced Database Capabilities
│
├── Temporal Data
│
├── History & Versioning
│
├── Soft Delete
│
├── Data Retention
│
├── Data Archival
│
├── Full-Text Search
│
├── JSON Query
│
├── Geographic / Spatial Data
│
└── Database Feature Capabilities
```

---

# 271. Bloque 27 completado

```text
✓ 267_DATABASE_TEMPORAL_DATA_SYSTEM.md
✓ 268_DATABASE_HISTORY_AND_VERSIONING_SYSTEM.md
✓ 269_DATABASE_SOFT_DELETE_SYSTEM.md
✓ 270_DATABASE_DATA_RETENTION_SYSTEM.md
✓ 271_DATABASE_DATA_ARCHIVAL_SYSTEM.md
✓ 272_DATABASE_FULL_TEXT_SEARCH_SYSTEM.md
✓ 273_DATABASE_JSON_QUERY_SYSTEM.md
✓ 274_DATABASE_GEOGRAPHIC_DATA_EXTENSION_SYSTEM.md
✓ 275_DATABASE_DATABASE_FEATURE_CAPABILITY_SYSTEM.md
```

---

# 272. Integración acumulada

Con los documentos 267–275, VoltStack Database dispone conceptualmente de:

```text
                       Query Engine
                            │
       ┌────────────────────┼─────────────────────┐
       │                    │                     │
       ▼                    ▼                     ▼
 Temporal Queries       JSON Queries       Spatial Queries
       │                    │                     │
       └────────────────────┼─────────────────────┘
                            ▼
                    Capability System
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
          MySQL          MariaDB       PostgreSQL
                                           │
                                           ▼
                                      Extensions
             │                             │
             └──────────────┬──────────────┘
                            ▼
                          SQLite
```

mientras:

```text
History
Soft Delete
Retention
Archival
Full-Text
JSON
Spatial
```

permanecen capacidades especializadas que utilizan el mismo:

```text
Query AST
Semantic Engine
Planner
Compiler
Execution Engine
Type System
Schema System
ORM
Telemetry
Security
Resource Governance
Capability System
```

---

# 273. Siguiente bloque

A partir del siguiente documento comienza:

```text
BLOCK 28 — BACKUP AND OPERATIONS
```

Este bloque definirá:

```text
Database Backup
Restore
Maintenance
Health Checks
Diagnostics
Administration
```

sin mezclar estas responsabilidades con:

```text
ORM
Migration
Schema
Query Engine
Driver
```

---

# 274. Siguiente documento

```text
276_DATABASE_BACKUP_ARCHITECTURE.md
```

La arquitectura comenzará estableciendo una separación crítica:

```text
Backup
≠
Migration
≠
Replication
≠
High Availability
≠
Archive
≠
Export
≠
Snapshot
≠
Disaster Recovery
```

y modelará:

```text
                    Backup Intent
                         │
                         ▼
                    Backup Plan
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
     Logical          Physical        Snapshot
      Backup           Backup          Backup
        │                │                │
        └────────────────┼────────────────┘
                         ▼
                  Consistency Model
                         │
                         ▼
                    Backup Engine
                         │
                         ▼
                   Backup Artifact
                         │
                         ▼
                  Verification
                         │
                         ▼
                  Retention Policy
                         │
                         ▼
                   Restore System
```

con una regla central para el siguiente bloque:

> **Un backup no se considerará válido únicamente porque fue creado; deberá existir evidencia suficiente de que el artefacto es íntegro, corresponde al alcance esperado y puede participar en un proceso de restauración verificable.**