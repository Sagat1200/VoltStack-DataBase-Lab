# 301_DATABASE_DATABASE_CAPABILITY_DISCOVERY_SYSTEM.md

# VoltStack Quantum Database
## Database Capability Discovery System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 301 — Database Capability Discovery System  
**Bloque:** 30 — Extensibility  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `300_DATABASE_CUSTOM_ORM_EXTENSION_SYSTEM.md`  
**Siguiente documento:** `302_DATABASE_PUBLIC_API_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura oficial del **Database Capability Discovery System** de:

```text
VoltStack/Quantum/Database
```

responsable de descubrir, recolectar, normalizar, validar y exponer **evidencia sobre las capacidades realmente disponibles** en un entorno Database.

VoltStack soportará múltiples:

```text
DBMS
Drivers
Versions
Server Configurations
Extensions
Connection Modes
Topologies
Replicas
Shards
Plugins
Runtimes
```

Por ello no será suficiente asumir:

```text
PostgreSQL 18
→ capability X

MySQL 9
→ capability Y
```

La capacidad efectiva puede depender de:

```text
Vendor
Version
Driver
Client Library
Server Configuration
Session Configuration
Installed Extensions
Permissions
Endpoint
Topology
Schema
Runtime
Plugin
```

La regla central será:

> **VoltStack no inferirá una capacidad únicamente por el nombre o versión declarada del DBMS cuando exista evidencia más precisa disponible. El Discovery System recopilará evidencia; el Capability System decidirá soporte.**

Formalmente:

```text
Capability Discovery
=
Discovery Request
+
Discovery Scope
+
Evidence Providers
+
Safe Probes
+
Evidence Normalization
+
Confidence
+
Provenance
+
Conflict Detection
+
Snapshot
+
Fingerprint
+
Diagnostics
```

---

# 2. Distinción fundamental

La separación más importante será:

```text
Discovery
≠
Decision
```

El Discovery System responde:

```text
¿Qué evidencia tenemos?
```

El Capability System responde:

```text
¿Qué soporte podemos afirmar?
```

Por tanto:

```text
Capability Discovery System
          ↓
Capability Evidence
          ↓
Capability Resolution System
          ↓
Capability Decision
```

---

# 3. Relación con `275_DATABASE_DATABASE_FEATURE_CAPABILITY_SYSTEM.md`

El documento 275 estableció el modelo general:

```text
Capability
Status
Evidence
Confidence
Requirement
Support Mode
Capability Snapshot
```

Este documento especializa la parte:

```text
¿Cómo obtenemos esa evidencia?
```

No crea un segundo Capability System.

---

# 4. Capability Discovery ≠ Capability Registry

El Registry define:

```text
qué capacidades conoce VoltStack
```

Discovery determina:

```text
qué evidencia existe en este entorno
```

---

# 5. Capability Discovery ≠ Platform

Platform contiene conocimiento estático del backend.

Discovery incorpora además conocimiento dinámico.

---

# 6. Capability Discovery ≠ Driver

El Driver puede aportar evidencia.

No es la autoridad universal de capacidades.

---

# 7. Capability Discovery ≠ Health Check

Un servidor puede estar:

```text
HEALTHY
```

y no soportar determinada capacidad.

Igualmente puede soportarla aunque un probe temporal falle.

---

# 8. Capability Discovery ≠ Schema Introspection

Schema Introspection observa:

```text
tables
columns
indexes
constraints
```

Capability Discovery observa características operativas o semánticas del entorno.

Ambos podrán colaborar.

---

# 9. Capability Discovery ≠ Version Detection

```text
Version Detection
⊂
Capability Discovery
```

pero:

```text
Version
≠
Capability
```

---

# 10. Capability Discovery ≠ Runtime Probe

Un runtime probe es sólo una fuente de evidencia.

---

# 11. Capability Discovery ≠ Feature Test

No será necesario ejecutar una operación destructiva para demostrar cada capability.

---

# 12. Objetivos

El sistema deberá:

1. descubrir capacidades de forma segura;
2. soportar múltiples fuentes de evidencia;
3. distinguir evidencia estática y dinámica;
4. preservar provenance;
5. representar confidence;
6. representar UNKNOWN correctamente;
7. detectar contradicciones;
8. soportar probes controlados;
9. evitar probes destructivos;
10. respetar permisos;
11. aplicar timeouts;
12. soportar caching;
13. producir snapshots;
14. producir fingerprints;
15. soportar endpoints heterogéneos;
16. soportar replicas;
17. soportar shards;
18. integrarse con plugins;
19. integrarse con custom drivers;
20. integrarse con custom dialects;
21. integrarse con custom compilers;
22. integrarse con Query Planner;
23. soportar runtimes persistentes;
24. proporcionar diagnostics;
25. emitir telemetry segura;
26. ser extensible;
27. ser determinista;
28. ser testeable.

---

# 13. Arquitectura general

```text
                  Database Environment
                          │
          ┌───────────────┼────────────────┐
          ▼               ▼                ▼
      Platform         Driver         Connection
          │               │                │
          └───────────────┼────────────────┘
                          ▼
                 Discovery Request
                          │
                          ▼
               Capability Discovery
                          │
          ┌───────────────┼──────────────────┐
          ▼               ▼                  ▼
   Static Providers   Dynamic Providers   Runtime Probes
          │               │                  │
          └───────────────┼──────────────────┘
                          ▼
                   Raw Evidence
                          │
                          ▼
              Evidence Normalization
                          │
                          ▼
               Conflict Detection
                          │
                          ▼
                Evidence Snapshot
                          │
                          ▼
               Capability Resolver
                          │
                          ▼
                Capability Snapshot
                          │
       ┌──────────────────┼──────────────────┐
       ▼                  ▼                  ▼
 Query Planner        Compiler          Diagnostics
```

---

# 14. Capability Identity

Toda capacidad deberá tener identidad estable:

```text
CapabilityId
```

Ejemplos:

```text
database.query.cte
database.query.recursive_cte
database.query.window_functions
database.query.returning
database.query.json
database.query.full_text
database.query.spatial

database.transaction.savepoint
database.transaction.isolation.serializable

database.schema.generated_columns
database.schema.partial_indexes

database.execution.cancel
database.execution.statement_timeout
```

---

# 15. CapabilityId ≠ SQL Keyword

Por ejemplo:

```text
database.query.returning
```

representa una semántica.

No simplemente la existencia del token:

```sql
RETURNING
```

---

# 16. CapabilityId ≠ Vendor Feature Name

La identidad deberá ser portable.

---

# 17. Capability Catalog

El sistema podrá mantener:

```text
CapabilityCatalog
```

con definiciones conocidas.

Conceptualmente:

```php
final readonly class CapabilityDefinition
{
    public function __construct(
        public CapabilityId $id,
        public CapabilityCategory $category,
        public CapabilityScope $scope,
        public ProbePolicy $probePolicy,
    ) {}
}
```

---

# 18. Capability Definition ≠ Capability Evidence

Definition:

```text
what capability means
```

Evidence:

```text
what was observed
```

---

# 19. Capability Scope

Una capability podrá tener alcance:

```text
PLATFORM
SERVER
DATABASE
SCHEMA
CONNECTION
SESSION
ENDPOINT
TOPOLOGY
DRIVER
RUNTIME
```

---

# 20. Scope Matters

Una capacidad detectada en:

```text
Connection A
```

no deberá asumirse automáticamente en:

```text
Connection B
```

si su scope es connection/session.

---

# 21. Discovery Request

Conceptualmente:

```php
final readonly class CapabilityDiscoveryRequest
{
    public function __construct(
        public CapabilityDiscoveryScope $scope,
        public ?CapabilityRequirementSet $requirements = null,
        public DiscoveryMode $mode = DiscoveryMode::SAFE,
    ) {}
}
```

---

# 22. Discovery Modes

La V1 podrá reconocer:

```text
STATIC_ONLY
SAFE
STANDARD
DEEP
```

---

# 23. STATIC_ONLY

Utiliza únicamente evidencia que no requiere interacción adicional con el DBMS.

---

# 24. SAFE

Podrá utilizar metadata y consultas read-only de bajo riesgo.

Será el modo normal.

---

# 25. STANDARD

Podrá ejecutar probes adicionales controlados.

---

# 26. DEEP

Reservado principalmente para:

```text
testing
diagnostics
installation verification
driver conformance
administration
```

---

# 27. DEEP ≠ Destructive

Incluso `DEEP` no significa:

```text
permission to destroy data
```

---

# 28. Discovery Context

Podrá existir:

```text
CapabilityDiscoveryContext
```

conteniendo referencias controladas a:

```text
Platform
DriverDescriptor
ConnectionMetadata
DatabaseContext
EndpointIdentity
TopologyIdentity
RuntimeIdentity
Permissions
ProbeBudget
```

---

# 29. Context ≠ Service Locator

No expondrá indiscriminadamente todo el Container.

---

# 30. Evidence Model

La unidad fundamental será:

```text
CapabilityEvidence
```

Conceptualmente:

```php
final readonly class CapabilityEvidence
{
    public function __construct(
        public CapabilityId $capability,
        public CapabilityEvidenceValue $value,
        public CapabilityEvidenceSource $source,
        public EvidenceConfidence $confidence,
        public EvidenceProvenance $provenance,
        public CapabilityScope $scope,
        public EvidenceObservedAt $observedAt,
    ) {}
}
```

---

# 31. Evidence Value

Podrá expresar:

```text
SUPPORTED
NOT_SUPPORTED
LIMITED
UNKNOWN
PRESENT
ABSENT
VALUE
RANGE
SET
```

dependiendo de la capability.

---

# 32. Evidence ≠ Decision

Crítico:

```text
Evidence(SUPPORTED)
≠
CapabilityStatus(SUPPORTED)
```

El resolver todavía deberá evaluar toda la evidencia.

---

# 33. Evidence Source

La arquitectura reconocerá inicialmente:

```text
PLATFORM_DEFINITION
SERVER_VERSION
SERVER_CONFIGURATION
DRIVER
CLIENT_LIBRARY
CONNECTION_METADATA
SESSION_METADATA
RUNTIME_PROBE
INSTALLED_EXTENSION
SCHEMA_METADATA
TOPOLOGY
USER_CONFIGURATION
PLUGIN
TEST_ENVIRONMENT
```

---

# 34. Evidence Provenance

Cada evidencia deberá poder responder:

```text
where did this information come from?
```

Ejemplo:

```text
Source:
SERVER_CONFIGURATION

Origin:
SHOW server_version

Endpoint:
writer-1

Observed:
2026-09-21T...

Provider:
PostgreSqlCapabilityDiscoveryProvider
```

---

# 35. Provenance ≠ Sensitive Data

No deberá contener:

```text
password
secret
token
private key
raw sensitive query parameter
```

---

# 36. Evidence Confidence

La V1 podrá utilizar:

```text
LOW
MEDIUM
HIGH
AUTHORITATIVE
```

---

# 37. Confidence ≠ Support Status

Ejemplo:

```text
Evidence:
"likely supported"

Confidence:
LOW
```

no equivale a:

```text
Capability:
SUPPORTED
```

---

# 38. Confidence ≠ Priority

Una evidencia puede tener alta confianza pero ser menos específica.

---

# 39. Evidence Specificity

Además de confidence, podrá evaluarse:

```text
scope specificity
```

Ejemplo:

```text
Platform documentation
```

vs:

```text
actual endpoint probe
```

---

# 40. Evidence Resolution Principle

Preferentemente:

```text
more specific verified evidence
>
generic inferred evidence
```

sin descartar automáticamente contradicciones.

---

# 41. Static Platform Evidence

Platform podrá aportar conocimiento como:

```text
PostgreSQL >= X
generally supports Y
```

---

# 42. Static Evidence Is Useful

Permite evitar probes innecesarios.

---

# 43. Static Evidence Is Not Absolute

Configuración, permisos o extensiones pueden alterar el resultado.

---

# 44. Vendor + Version

Será una fuente válida de evidencia.

Pero:

```text
Vendor + Version
≠
Capability
```

---

# 45. Version Detection

El sistema deberá distinguir:

```text
DBMS Product
DBMS Version
Driver Version
Client Library Version
Protocol Version
Extension Version
```

---

# 46. Version ≠ Version

Ejemplo:

```text
PDO extension version
≠
libpq version
≠
PostgreSQL server version
```

---

# 47. Product Identity

MySQL y MariaDB serán tratados como plataformas distintas.

---

# 48. MySQL ≠ MariaDB

No se inferirá:

```text
MySQL supports X
→ MariaDB supports X
```

ni inversamente.

---

# 49. SQLite

SQLite tendrá su propio discovery provider.

---

# 50. SQLite Build Capabilities

Algunas capacidades pueden depender de cómo SQLite fue compilado.

Por tanto:

```text
SQLite Version
≠
Complete Capability Set
```

---

# 51. Server Configuration Evidence

Algunas capabilities dependerán de configuración.

Ejemplos:

```text
extensions enabled
SQL modes
server variables
timezone behavior
read-only state
replication mode
```

---

# 52. Configuration Query Safety

Los providers deberán declarar qué queries utilizan.

---

# 53. Read-only Probe Preferred

Siempre que sea posible:

```text
read-only metadata query
```

será preferible.

---

# 54. Installed Extensions

PostgreSQL y otros sistemas pueden disponer de extensiones opcionales.

Ejemplo conceptual:

```text
database.extension.postgis
```

---

# 55. Extension Installed ≠ Usable

Una extensión puede existir pero:

```text
permission denied
not enabled in current database
wrong version
```

---

# 56. Extension Version

Cuando sea relevante deberá formar parte de la evidencia.

---

# 57. Driver Evidence

El Driver podrá declarar:

```text
binding support
streaming support
cancellation support
metadata access
protocol limitations
```

---

# 58. Driver Claim ≠ Proven Behavior

Para capacidades críticas:

```text
Driver declares X
```

podrá requerir conformance evidence.

---

# 59. Client Library Evidence

Ejemplo:

```text
native library supports cancellation
```

podrá ser necesario para ciertas capacidades.

---

# 60. Connection Metadata Evidence

Una conexión podrá revelar:

```text
server identity
server version
database
session state
transaction support
connection mode
```

---

# 61. Connection Metadata ≠ Global Server Truth

Algunas propiedades sólo aplican a esa conexión.

---

# 62. Session Capabilities

Ejemplo:

```text
current role
search path
timezone
isolation level
read-only mode
```

pueden cambiar.

---

# 63. Session Capability Cache

Deberá tener un lifetime apropiado.

No deberá cachearse indefinidamente como server-global.

---

# 64. Runtime Probe

Un probe será:

> una operación controlada diseñada para obtener evidencia específica sobre una capability.

---

# 65. Probe ≠ Application Query

No deberá depender de datos de negocio.

---

# 66. Probe ≠ Migration

No deberá alterar permanentemente schema.

---

# 67. Probe Contract

Conceptualmente:

```php
interface CapabilityProbe
{
    public function capability(): CapabilityId;

    public function probe(
        CapabilityProbeContext $context
    ): CapabilityProbeResult;
}
```

---

# 68. Probe Result

Podrá ser:

```text
OBSERVED_SUPPORTED
OBSERVED_UNSUPPORTED
OBSERVED_LIMITATION
INCONCLUSIVE
NOT_PERMITTED
TIMEOUT
FAILED
SKIPPED
```

---

# 69. Probe Failure ≠ Unsupported

Regla crítica:

```text
Probe Failure
≠
Capability Unsupported
```

---

# 70. Permission Denied ≠ Unsupported

Ejemplo:

```text
server may support feature
but current role cannot inspect/probe it
```

Resultado:

```text
UNKNOWN / INCONCLUSIVE
```

según resolver.

---

# 71. Timeout ≠ Unsupported

Igualmente:

```text
Probe Timeout
≠
Unsupported
```

---

# 72. Syntax Error

Incluso un syntax error deberá interpretarse dentro de contexto.

Puede significar:

```text
unsupported feature
wrong dialect
wrong server identity
probe incompatibility
```

---

# 73. Probe Safety Classification

Podrá existir:

```text
PURE_METADATA
READ_ONLY
TEMPORARY_SESSION
TEMPORARY_TRANSACTION
TEMPORARY_SCHEMA
DESTRUCTIVE
```

---

# 74. Default Allowed Classes

Runtime normal:

```text
PURE_METADATA
READ_ONLY
```

y algunas:

```text
TEMPORARY_SESSION
```

si son reversibles.

---

# 75. TEMPORARY_SCHEMA

No deberá ejecutarse automáticamente durante requests normales.

---

# 76. DESTRUCTIVE Probe

Prohibido por defecto.

---

# 77. Probe Side Effects

Todo probe deberá declarar:

```text
side effects
required permissions
transaction behavior
cleanup requirements
```

---

# 78. Probe Budget

El discovery deberá tener límites.

Ejemplo:

```php
final readonly class ProbeBudget
{
    public function __construct(
        public int $maxQueries,
        public int $maxDurationMs,
        public int $maxTemporaryObjects,
    ) {}
}
```

---

# 79. Probe Timeout

Cada probe tendrá timeout acotado.

---

# 80. Global Discovery Timeout

También podrá existir:

```text
DiscoveryDeadline
```

---

# 81. Deadline Exceeded

No implicará que capabilities restantes sean unsupported.

Serán:

```text
UNKNOWN
```

o no observadas.

---

# 82. Probe Cancellation

Cuando Driver/Execution Engine lo permita, deberá respetarse cancellation.

---

# 83. Probe Execution Path

Preferentemente:

```text
Discovery
   ↓
Probe
   ↓
Query/Execution Infrastructure
   ↓
Connection
   ↓
Driver
```

No acceso arbitrario directo a PDO.

---

# 84. Probe Isolation

Los probes no deberán contaminar el estado de sesión.

---

# 85. Probe Session Mutation

Si modifica:

```text
timezone
role
search_path
isolation
session variable
```

deberá restaurarse de forma verificable.

---

# 86. Failed Restoration

La conexión deberá considerarse potencialmente:

```text
DIRTY
```

y no volver al pool.

---

# 87. Probe Transaction

Cuando sea posible:

```text
BEGIN
probe
ROLLBACK
```

podrá utilizarse.

Pero:

```text
rollback
≠
guaranteed reversal of every DBMS operation
```

---

# 88. DDL Caveats

Los probes deberán conocer las semánticas DDL de cada plataforma.

---

# 89. Probe Temporary Objects

Si un probe crea recursos temporales deberá utilizar:

```text
unique owned names
```

y cleanup verificable.

---

# 90. Unknown Cleanup Outcome

Resultado:

```text
resource state UNKNOWN
```

y deberá emitirse diagnóstico.

---

# 91. Discovery Providers

La arquitectura utilizará:

```text
CapabilityDiscoveryProvider
```

---

# 92. Provider Contract

Conceptualmente:

```php
interface CapabilityDiscoveryProvider
{
    public function supports(
        CapabilityDiscoveryContext $context
    ): bool;

    public function discover(
        CapabilityDiscoveryRequest $request,
        CapabilityDiscoveryContext $context
    ): CapabilityEvidenceSet;
}
```

---

# 93. Provider Types

```text
PlatformCapabilityProvider
DriverCapabilityProvider
ServerCapabilityProvider
ConnectionCapabilityProvider
ExtensionCapabilityProvider
SchemaCapabilityProvider
TopologyCapabilityProvider
RuntimeProbeCapabilityProvider
PluginCapabilityProvider
```

---

# 94. Provider ≠ Resolver

Provider produce evidencia.

Resolver produce decisiones.

---

# 95. Provider Ordering

No deberá utilizarse para implementar:

```text
last provider wins
```

---

# 96. Evidence Aggregation

Toda evidencia relevante se agregará:

```text
E = {e1, e2, ..., en}
```

antes de resolver.

---

# 97. Conflicting Evidence

Ejemplo:

```text
PlatformProvider:
SUPPORTED

RuntimeProbe:
UNSUPPORTED
```

no deberá resolverse ocultando uno.

---

# 98. Conflict Model

Podrá existir:

```text
CapabilityEvidenceConflict
```

---

# 99. Conflict Reasons

```text
STALE_EVIDENCE
SCOPE_MISMATCH
VERSION_MISMATCH
ENDPOINT_DIFFERENCE
CONFIGURATION_DIFFERENCE
PROBE_FAILURE
PROVIDER_DISAGREEMENT
INVALID_DECLARATION
```

---

# 100. Conflict ≠ Failure Necessarily

Puede producir:

```text
SUPPORTED_WITH_LIMITATIONS
UNKNOWN
INCONCLUSIVE
```

según el Capability Resolver.

---

# 101. Contradiction Diagnostics

Deberá mostrar:

```text
Capability:
database.query.returning

Evidence A:
Platform definition
SUPPORTED
MEDIUM

Evidence B:
Endpoint probe
UNSUPPORTED
HIGH

Resolution:
UNKNOWN

Reason:
contradictory evidence
```

sin ocultar la contradicción.

---

# 102. Evidence Normalization

Providers podrán devolver representaciones diferentes.

El normalizador las convertirá al modelo canónico.

---

# 103. Raw Evidence ≠ Canonical Evidence

Separación:

```text
Raw Provider Result
       ↓
Normalizer
       ↓
CapabilityEvidence
```

---

# 104. Evidence Validation

Antes de aceptar evidencia se validará:

```text
known capability
valid scope
valid source
valid provenance
valid value
valid timestamp
compatible provider
```

---

# 105. Unknown Capability

Un plugin podrá registrar nuevas capabilities mediante el Extension System.

Una capability no registrada deberá ser:

```text
UNKNOWN_CAPABILITY
```

no inventada automáticamente.

---

# 106. Plugin Capability Registration

Un plugin podrá registrar:

```text
CapabilityDefinition
CapabilityDiscoveryProvider
CapabilityProbe
```

---

# 107. Plugin Cannot Override Core Silently

Un plugin no podrá redefinir:

```text
database.transaction.savepoint
```

sin un mecanismo de override explícito y validado.

---

# 108. Capability Provider Identity

Cada provider tendrá:

```text
CapabilityProviderId
```

estable.

---

# 109. Provider Version

Podrá formar parte de provenance/fingerprint cuando afecte discovery.

---

# 110. Discovery Registry

Durante bootstrap:

```text
Core Providers
     +
Driver Providers
     +
Platform Providers
     +
Plugin Providers
     ↓
Discovery Registry
     ↓
Validation
     ↓
Freeze
```

---

# 111. Registry Freeze

El registry será inmutable durante operación normal.

---

# 112. Runtime Discovery State

Los resultados dinámicos sí podrán variar.

---

# 113. Definition State ≠ Observation State

```text
Frozen Provider Registry
```

puede coexistir con:

```text
changing capability observations
```

---

# 114. Capability Snapshot

El resultado agregado de discovery podrá formar:

```text
CapabilityEvidenceSnapshot
```

---

# 115. Snapshot Contents

Podrá contener:

```text
snapshot id
environment identity
endpoint identity
platform identity
driver identity
server identity/version
observed capabilities
evidence
conflicts
discovery mode
timestamp
generation
fingerprint
```

---

# 116. Snapshot ≠ Eternal Truth

Representa:

```text
what was observed
at a specific time
under a specific context
```

---

# 117. Capability Snapshot vs Evidence Snapshot

Podrán distinguirse:

```text
CapabilityEvidenceSnapshot
```

y:

```text
ResolvedCapabilitySnapshot
```

---

# 118. Flow

```text
Discovery
   ↓
Evidence Snapshot
   ↓
Capability Resolver
   ↓
Resolved Capability Snapshot
```

---

# 119. Snapshot Immutability

Los snapshots serán inmutables.

---

# 120. Snapshot Generation

Cada actualización producirá nueva generación.

---

# 121. Capability Fingerprint

Permitirá detectar si el entorno relevante cambió.

Conceptualmente:

```text
CapabilityFingerprint
=
hash(
    platform
    + server identity/version
    + driver identity/version
    + relevant configuration
    + extensions
    + normalized evidence
)
```

---

# 122. Fingerprint ≠ Security Hash

Su objetivo principal será:

```text
identity/change detection
```

no almacenamiento de contraseñas.

---

# 123. Sensitive Inputs

Secrets no deberán incluirse.

---

# 124. Fingerprint Uses

Podrá utilizarse para:

```text
Compiled Query Cache
Metadata Cache
Plan Cache
Diagnostics
Benchmark Evidence
Testing
Persistent Runtime Revalidation
```

---

# 125. Capability Change

Si cambia el fingerprint:

```text
dependent compiled artifacts
```

podrán invalidarse.

---

# 126. Cache Architecture

Discovery puede ser costoso.

Por ello habrá:

```text
CapabilityDiscoveryCache
```

---

# 127. Discovery Cache ≠ Capability Decision Cache

Se distinguirán cuando sea necesario.

---

# 128. Cache Key

Podrá incluir:

```text
EnvironmentIdentity
EndpointIdentity
DatabaseIdentity
DriverIdentity
CapabilityScope
DiscoveryMode
ProviderGeneration
```

---

# 129. TTL

TTL podrá utilizarse.

Pero:

```text
TTL
≠
Correctness Guarantee
```

---

# 130. Cache Validity

Podrá depender de:

```text
connection generation
server restart identity
configuration generation
plugin generation
extension generation
topology generation
```

---

# 131. Session Evidence

No deberá almacenarse como server-global.

---

# 132. Connection Evidence

Podrá estar vinculada a:

```text
PhysicalConnectionId
```

o generation.

---

# 133. Server Evidence

Podrá reutilizarse entre conexiones al mismo server identity si la semántica lo permite.

---

# 134. Stale Evidence

Deberá poder marcarse:

```text
STALE
```

en vez de fingir actualidad.

---

# 135. Cache Failure

No deberá convertir automáticamente capabilities en unsupported.

---

# 136. Persistent Runtime

En FrankenPHP un worker puede vivir mucho tiempo.

Por tanto:

```text
capability snapshot
```

no deberá asumirse válido para siempre.

---

# 137. Persistent Worker Strategy

Podrá existir:

```text
initial discovery
+
generation validation
+
event-driven invalidation
+
bounded periodic revalidation
```

según capability.

---

# 138. Request Scope

Un request podrá recibir una referencia consistente a un:

```text
ResolvedCapabilitySnapshot
```

para evitar que las decisiones cambien a mitad de una operación.

---

# 139. Mid-operation Capability Change

No deberá cambiar retroactivamente el plan ya en ejecución.

---

# 140. Next Operation

La nueva snapshot podrá utilizarse en la siguiente operación segura.

---

# 141. RoadRunner

Aplicarán las mismas reglas.

---

# 142. OpenSwoole

Los snapshots inmutables podrán compartirse.

El state mutable de discovery por coroutine no.

---

# 143. Endpoint-specific Discovery

En topologías distribuidas:

```text
Endpoint A
≠
Endpoint B
```

---

# 144. Example

```text
Writer:
PostgreSQL 18
Extension X installed

Replica 1:
PostgreSQL 18
Extension X installed

Replica 2:
PostgreSQL 18
Extension X absent
```

La capability no puede tratarse simplemente como:

```text
cluster = supported
```

---

# 145. Endpoint Capability Snapshot

Cada endpoint podrá tener su snapshot.

---

# 146. Topology Capability View

Podrá derivarse una vista agregada:

```text
ALL_ENDPOINTS
ANY_ENDPOINT
WRITER
READ_ELIGIBLE_ENDPOINTS
SHARD
```

---

# 147. Aggregate Capability ≠ Endpoint Capability

Importante:

```text
ANY replica supports X
```

no significa:

```text
every read can use X
```

---

# 148. Read Routing Integration

Si un query requiere capability X:

```text
Query Requirement
      ↓
Read Routing
      ↓
Only endpoints satisfying X
```

---

# 149. Writer Requirements

Operaciones de escritura deberán validar capabilities del writer seleccionado.

---

# 150. Replica Lag ≠ Capability

Lag es una condición operacional.

No debe confundirse con feature capability.

---

# 151. Replica Eligibility

Puede depender de:

```text
Health
Lag
Capabilities
Transaction Rules
Consistency Policy
```

---

# 152. Sharding

Cada shard podrá tener:

```text
different endpoint capability snapshots
```

aunque idealmente la flota sea homogénea.

---

# 153. Shard Capability Drift

El sistema deberá detectarlo.

---

# 154. Drift ≠ Unsupported Globally

Debe reportarse como heterogeneidad.

---

# 155. Shard Routing Integration

Un query dirigido a shard específico deberá utilizar las capabilities de ese shard.

---

# 156. Scatter Query

Si una operación debe ejecutarse en múltiples shards:

```text
required capability
```

deberá estar disponible en todos los shards participantes, salvo estrategia explícita de degradación.

---

# 157. Multitenancy

Capabilities no deberán mezclarse con tenant identity.

Pero el tenant puede resolver a:

```text
different database/server/shard
```

y por tanto a distinta snapshot.

---

# 158. Tenant A Capability ≠ Tenant B Capability

Cuando utilizan infraestructuras distintas.

---

# 159. Tenant-specific Capability Cache

La key no necesita incluir TenantId si el tenant resuelve a una identidad de infraestructura única suficiente.

---

# 160. Avoid Cardinality Explosion

No se duplicará cache por tenant innecesariamente cuando varios tenants comparten exactamente el mismo environment identity.

---

# 161. Schema-dependent Capabilities

Algunas features dependen de schema.

Ejemplo:

```text
full-text index available for this query target
```

---

# 162. Feature Capability ≠ Schema Availability

Distinción:

```text
DBMS supports full-text
```

vs:

```text
this table has required full-text index
```

---

# 163. Capability Composition

Podrá expresarse:

```text
QueryExecutable
=
PlatformFeature
∧
DriverFeature
∧
SchemaRequirement
∧
PermissionRequirement
```

---

# 164. Schema Evidence

Deberá provenir del Schema Metadata/Introspection System.

---

# 165. Schema Evidence Coverage

Si introspection fue:

```text
PARTIAL
```

la ausencia observada no deberá asumirse definitiva.

---

# 166. Not Observed ≠ Absent

Regla heredada:

```text
NotObserved
≠
Absent
```

---

# 167. Permission-dependent Capabilities

El servidor puede soportar:

```text
COPY
```

pero el usuario actual no.

---

# 168. Platform Capability vs Effective Capability

Podrán distinguirse:

```text
PLATFORM_SUPPORTED
```

y:

```text
EFFECTIVELY_USABLE
```

---

# 169. Effective Capability

Puede requerir:

```text
feature exists
∧
driver supports
∧
permission available
∧
configuration compatible
```

---

# 170. Permission Probe Safety

No se intentarán operaciones privilegiadas sólo para comprobar si fallan.

---

# 171. Privilege Metadata

Cuando el DBMS ofrezca metadata segura, podrá utilizarse.

---

# 172. Security Principle

Discovery deberá seguir:

```text
least privilege
```

---

# 173. No Credential Escalation

Discovery no solicitará automáticamente credenciales administrativas para obtener más evidencia.

---

# 174. Admin Discovery

Podrá existir como modo explícito de administración.

---

# 175. Runtime Capability

Algunas capacidades dependen de PHP/runtime.

Ejemplos:

```text
fiber-safe behavior
coroutine compatibility
native cancellation integration
async I/O adapter
```

---

# 176. Runtime Capability ≠ Database Capability

Deberán mantenerse categorizadas.

---

# 177. Composite Requirements

Una feature de VoltStack puede necesitar:

```text
Database Capability
+
Driver Capability
+
Runtime Capability
```

---

# 178. Example

```text
AsyncStreaming
=
DriverStreaming
∧
RuntimeAsyncSupport
∧
ConnectionAdapterSupport
```

---

# 179. Requirement Model

El documento 275 definió composición:

```text
ALL_OF
ANY_OF
NONE_OF
```

Discovery aportará evidencia para resolverlas.

---

# 180. Requirement Evaluation

Ejemplo:

```text
Feature:
AdvancedJsonMutation

Requires:
ALL_OF(
    database.query.json,
    database.query.json_mutation
)
```

---

# 181. Planner Integration

El Query Planner preguntará:

```text
Can the selected execution target satisfy requirement R?
```

No:

```php
if ($platform === 'postgresql') {
```

---

# 182. Compiler Integration

El Compiler podrá seleccionar representación según:

```text
ResolvedCapabilitySnapshot
```

---

# 183. Compiler ≠ Discovery

El Compiler no deberá ejecutar probes.

---

# 184. Planner ≠ Discovery

El Planner tampoco deberá iniciar arbitrariamente network discovery.

---

# 185. Pre-resolved Snapshot

Planner y Compiler consumirán una snapshot ya resuelta.

---

# 186. Query Compilation Cache

La key podrá incorporar:

```text
relevant capability fingerprint
```

---

# 187. Relevant Fingerprint

Idealmente no será necesario invalidar todo por una capability no relacionada.

---

# 188. Capability Dependency Set

Cada compiled artifact podrá declarar:

```text
which capabilities influenced compilation
```

---

# 189. Selective Invalidation

Así:

```text
Capability X changed
```

invalidará artefactos dependientes de X.

---

# 190. Migration Integration

Migration Planner podrá consultar capabilities para determinar:

```text
native operation
online operation
emulation
unsupported
```

---

# 191. Discovery During Migration

Podrá permitirse discovery más profundo antes de una migration.

---

# 192. Migration Safety

Probe discovery no sustituirá:

```text
Migration Safety System
```

---

# 193. Backup Integration

Backup Planner podrá descubrir:

```text
native backup support
snapshot support
tool availability
permissions
```

pero:

```text
DB capability
≠
environment tool capability
```

---

# 194. Test Environment Integration

El documento 286 puede proporcionar:

```text
known environment capabilities
```

como evidencia.

---

# 195. Test Declaration ≠ Real Observation

Un profile de testing que diga:

```text
supports X
```

no deberá superar evidencia real contradictoria sin explicación.

---

# 196. Driver Conformance Integration

El documento 292 podrá producir evidencia sobre:

```text
driver capabilities actually verified
```

---

# 197. Conformance Evidence

Puede tener alta confianza.

Pero deberá corresponder a:

```text
same driver version
compatible environment
```

---

# 198. Performance Testing Integration

Los benchmark reports deberán registrar:

```text
CapabilityFingerprint
```

para reproducibilidad.

---

# 199. Diagnostics

El sistema deberá permitir preguntas como:

```text
Why does VoltStack think RETURNING is available?

Why is this feature UNKNOWN?

Which probe failed?

Which endpoint lacks capability X?

Which extension provides capability Y?

Did the capability snapshot become stale?
```

---

# 200. Capability Explain

API conceptual:

```php
$capabilities->explain(
    CapabilityId::of('database.query.returning')
);
```

---

# 201. Example Diagnostic

```text
Capability:
database.query.returning

Resolved Status:
SUPPORTED

Mode:
NATIVE

Confidence:
HIGH

Scope:
ENDPOINT

Endpoint:
writer-primary

Evidence:

[1]
Source:
PLATFORM_DEFINITION

Value:
SUPPORTED

Confidence:
MEDIUM

[2]
Source:
SERVER_VERSION

Value:
SUPPORTED

Server:
PostgreSQL 18.x

Confidence:
HIGH

Conflicts:
none

Fingerprint:
cap_8f...
```

---

# 202. Unknown Diagnostic

```text
Capability:
database.execution.cancel

Status:
UNKNOWN

Reason:
driver provider reported support,
but runtime probe timed out.

Probe timeout is not interpreted as unsupported.
```

---

# 203. CLI

La Developer Experience podrá ofrecer:

```bash
php voltstack database:capabilities
```

---

# 204. Detailed CLI

```bash
php voltstack database:capabilities --connection=default --verbose
```

---

# 205. Probe CLI

En entornos autorizados:

```bash
php voltstack database:capabilities --probe
```

---

# 206. Deep Discovery

```bash
php voltstack database:capabilities --mode=deep
```

deberá mostrar claramente qué probes adicionales se ejecutarán.

---

# 207. Explain Capability

```bash
php voltstack database:capability:explain database.query.returning
```

---

# 208. Refresh

Podrá existir:

```bash
php voltstack database:capabilities --refresh
```

---

# 209. Refresh ≠ Force Supported

Sólo vuelve a descubrir.

---

# 210. Machine-readable Output

Podrá soportarse:

```bash
--json
```

para CI/diagnostics.

---

# 211. Telemetry

El sistema podrá medir:

```text
discovery.duration
discovery.provider.duration
discovery.probe.duration
discovery.probe.count
discovery.probe.failure
discovery.conflict.count
discovery.cache.hit
discovery.cache.miss
discovery.snapshot.refresh
```

---

# 212. Metric Cardinality

Permitido:

```text
provider id
capability category
platform
result class
```

con cardinalidad controlada.

---

# 213. Endpoint Labels

Deberán evitar cardinalidad no acotada.

---

# 214. Sensitive Information

Telemetry no incluirá:

```text
connection string
credentials
query parameters
secret config
```

---

# 215. Discovery Events

Podrán existir eventos internos:

```text
CapabilityDiscoveryStarted
CapabilityEvidenceObserved
CapabilityEvidenceConflictDetected
CapabilityDiscoveryCompleted
CapabilitySnapshotChanged
CapabilityProbeFailed
```

---

# 216. Events ≠ Capability Decision

Mantienen separación arquitectónica.

---

# 217. Event Listener Failure

No deberá alterar silenciosamente evidence.

---

# 218. Discovery Failure Model

Fallos podrán clasificarse:

```text
PROVIDER_FAILURE
CONNECTION_FAILURE
AUTHORIZATION_FAILURE
PROBE_FAILURE
PROBE_TIMEOUT
PROBE_CANCELLED
NORMALIZATION_FAILURE
EVIDENCE_CONFLICT
CACHE_FAILURE
ENVIRONMENT_FAILURE
```

---

# 219. Discovery Failure ≠ Database Failure

La aplicación puede continuar si la capability no es necesaria.

---

# 220. Required Capability

Si una operación requiere capability X y X queda:

```text
UNKNOWN
```

la política normal será:

```text
do not execute an unsafe assumption
```

---

# 221. UNKNOWN Handling

Opciones del resolver podrán ser:

```text
REJECT
USE_SAFE_FALLBACK
USE_EMULATION
REQUIRE_EXPLICIT_OVERRIDE
```

según capability.

---

# 222. UNKNOWN ≠ UNSUPPORTED

Nunca:

```text
UNKNOWN → false
```

automáticamente como verdad semántica.

---

# 223. Unsupported May Allow Emulation

Ejemplo:

```text
native capability unsupported
```

pero:

```text
VoltStack emulation available
```

---

# 224. Native Support ≠ Effective Support Mode

El Capability System podrá resolver:

```text
NATIVE
EMULATED
DEGRADED
NONE
```

---

# 225. Discovery Only Supplies Evidence

Discovery no decide si una emulación es aceptable.

---

# 226. Custom Driver Integration

Un custom driver podrá proporcionar:

```text
DriverCapabilityDiscoveryProvider
```

---

# 227. Driver Provider Scope

Sólo deberá declarar lo que el Driver realmente puede conocer.

---

# 228. Custom Dialect Integration

Dialect podrá aportar:

```text
static representational evidence
```

pero no ejecutar probes.

---

# 229. Custom Compiler Integration

Compiler podrá declarar:

```text
capability requirements
```

No deberá declararse a sí mismo como prueba de que la capability existe.

---

# 230. Custom Query Extension Integration

Una Query Extension podrá declarar:

```text
required capabilities
```

---

# 231. Custom ORM Extension Integration

Una ORM Extension también podrá declarar requirements.

Ejemplo:

```text
TemporalOrmExtension
requires
database.query.temporal
```

---

# 232. Plugin Integration

Un plugin podrá aportar:

```text
CapabilityDefinition
DiscoveryProvider
Probe
Requirement
Diagnostic Formatter
```

---

# 233. Plugin Trust Boundary

Los probes de terceros deberán tratarse como código privilegiado.

---

# 234. Third-party Probe Policy

VoltStack podrá requerir:

```text
explicit enablement
```

para probes con side effects superiores a `READ_ONLY`.

---

# 235. Probe Manifest

Cada probe deberá declarar:

```text
id
capability
safety class
permissions
expected statements
timeout
side effects
cleanup
```

---

# 236. Hidden Probe Operations

No serán permitidas como práctica aceptada.

---

# 237. Probe Auditability

En modo diagnóstico podrá mostrarse qué probes se ejecutaron.

---

# 238. Probe SQL Visibility

Podrá mostrarse SQL sanitizado cuando sea seguro.

---

# 239. Probe Secrets

Nunca se mostrarán secretos.

---

# 240. Testing Architecture

El Discovery System requerirá:

```text
Unit Tests
Integration Tests
Driver Tests
Platform Tests
Failure Tests
Security Tests
Persistent Runtime Tests
Plugin Tests
```

---

# 241. Unit Tests

Adecuados para:

```text
evidence normalization
confidence rules
scope validation
fingerprints
cache keys
conflict detection
provider resolution
probe policy
```

---

# 242. Integration Tests

Necesarios para:

```text
server version detection
server config detection
extension detection
session detection
runtime probes
permissions
endpoint-specific behavior
```

---

# 243. Platform Matrix

Deberá incluir independientemente:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 244. Version Matrix

Capabilities sensibles a versión deberán probarse en varias versiones soportadas.

---

# 245. Configuration Matrix

Cuando sea relevante:

```text
feature enabled
feature disabled
different SQL mode
extension installed
extension absent
```

---

# 246. Permission Matrix

Ejemplo:

```text
admin user
runtime user
read-only user
restricted user
```

---

# 247. Probe Failure Tests

Deberán cubrir:

```text
timeout
permission denied
connection lost
syntax error
server shutdown
cancelled query
cleanup failure
```

---

# 248. Expected Rule

Ninguno deberá convertirse automáticamente en:

```text
UNSUPPORTED
```

sin evidencia semántica suficiente.

---

# 249. Conflict Tests

Deberán generar deliberadamente:

```text
static says supported
dynamic says unsupported
```

y comprobar que la contradicción se conserva.

---

# 250. Cache Tests

Cubrirán:

```text
hit
miss
expiration
generation change
endpoint change
connection change
session change
plugin generation
```

---

# 251. Persistent Runtime Tests

```text
Request A
  ↓
Snapshot A
  ↓
Environment changes
  ↓
Revalidation
  ↓
Request B
  ↓
Snapshot B
```

sin leakage mutable.

---

# 252. Concurrency Tests

Dos requests podrán consultar discovery simultáneamente.

No deberán producir corrupción de cache/snapshot.

---

# 253. Single-flight Discovery

Opcionalmente:

```text
same discovery key
+
concurrent refresh
→
one actual discovery
```

podrá utilizarse.

---

# 254. Single-flight Failure

No deberá bloquear indefinidamente otros callers.

---

# 255. Testing with Fakes

Los fakes podrán probar:

```text
resolver orchestration
conflict handling
cache logic
```

pero no:

```text
actual DBMS capability
```

---

# 256. Real DB Evidence Required

Cuando se afirma comportamiento del DBMS:

```text
real integration environment
```

será necesario.

---

# 257. Capability Discovery Conformance

Custom providers podrán tener:

```text
CapabilityDiscoveryConformanceKit
```

---

# 258. Conformance Requirements

Un provider deberá demostrar:

```text
stable identity
correct scope
safe behavior
deterministic normalization
proper UNKNOWN handling
timeout handling
permission handling
cleanup
redaction
```

---

# 259. Directory Structure

Propuesta:

```text
src/Quantum/Database/Capability/
├── Contract/
│   ├── CapabilityId.php
│   ├── CapabilityDefinition.php
│   └── CapabilityScope.php
│
├── Discovery/
│   ├── CapabilityDiscoverySystem.php
│   ├── CapabilityDiscoveryRequest.php
│   ├── CapabilityDiscoveryContext.php
│   ├── CapabilityDiscoveryMode.php
│   └── CapabilityDiscoveryResult.php
│
├── Provider/
│   ├── CapabilityDiscoveryProvider.php
│   ├── CapabilityProviderId.php
│   ├── CapabilityProviderRegistry.php
│   ├── PlatformCapabilityProvider.php
│   ├── DriverCapabilityProvider.php
│   ├── ServerCapabilityProvider.php
│   ├── ConnectionCapabilityProvider.php
│   ├── SchemaCapabilityProvider.php
│   ├── TopologyCapabilityProvider.php
│   └── PluginCapabilityProvider.php
│
├── Evidence/
│   ├── CapabilityEvidence.php
│   ├── CapabilityEvidenceSet.php
│   ├── CapabilityEvidenceValue.php
│   ├── CapabilityEvidenceSource.php
│   ├── EvidenceConfidence.php
│   ├── EvidenceProvenance.php
│   ├── EvidenceNormalizer.php
│   └── CapabilityEvidenceConflict.php
│
├── Probe/
│   ├── CapabilityProbe.php
│   ├── CapabilityProbeId.php
│   ├── CapabilityProbeContext.php
│   ├── CapabilityProbeResult.php
│   ├── CapabilityProbeRegistry.php
│   ├── ProbeSafetyClass.php
│   ├── ProbePolicy.php
│   ├── ProbeBudget.php
│   └── ProbeManifest.php
│
├── Snapshot/
│   ├── CapabilityEvidenceSnapshot.php
│   ├── CapabilitySnapshotId.php
│   ├── CapabilitySnapshotGeneration.php
│   └── CapabilityFingerprint.php
│
├── Cache/
│   ├── CapabilityDiscoveryCache.php
│   ├── CapabilityDiscoveryCacheKey.php
│   └── CapabilityDiscoveryCachePolicy.php
│
├── Topology/
│   ├── EndpointCapabilitySnapshot.php
│   ├── TopologyCapabilityView.php
│   └── CapabilityDriftDetector.php
│
├── Diagnostics/
│   ├── CapabilityDiscoveryInspector.php
│   ├── CapabilityExplanation.php
│   └── CapabilityDiscoveryReport.php
│
├── Telemetry/
│   └── CapabilityDiscoveryTelemetry.php
│
└── Exception/
    ├── CapabilityDiscoveryException.php
    ├── CapabilityProviderException.php
    ├── CapabilityProbeException.php
    ├── CapabilityProbeTimeoutException.php
    ├── CapabilityEvidenceException.php
    └── CapabilityConflictException.php
```

---

# 260. Platform-specific Structure

```text
src/Quantum/Database/Platform/
├── MySql/
│   └── Capability/
│       ├── MySqlCapabilityDiscoveryProvider.php
│       └── Probe/
│
├── MariaDb/
│   └── Capability/
│
├── PostgreSql/
│   └── Capability/
│
└── Sqlite/
    └── Capability/
```

---

# 261. Testing Structure

```text
tests/Quantum/Database/Capability/
├── Unit/
├── Integration/
├── Platform/
│   ├── MySql/
│   ├── MariaDb/
│   ├── PostgreSql/
│   └── Sqlite/
├── Probe/
├── Cache/
├── Topology/
├── Failure/
├── Security/
├── Runtime/
└── Conformance/
```

---

# 262. Architectural Invariants

## DB-CAP-DISC-001

Discovery ≠ Capability Decision.

## DB-CAP-DISC-002

Evidence ≠ Support.

## DB-CAP-DISC-003

Capability Registry ≠ Discovery System.

## DB-CAP-DISC-004

Capability Discovery ≠ Platform.

## DB-CAP-DISC-005

Capability Discovery ≠ Driver.

## DB-CAP-DISC-006

Capability Discovery ≠ Health Check.

## DB-CAP-DISC-007

Capability Discovery ≠ Schema Introspection.

## DB-CAP-DISC-008

Capability Discovery ≠ Version Detection.

## DB-CAP-DISC-009

Capability Discovery ≠ Runtime Probe.

## DB-CAP-DISC-010

CapabilityId será estable.

## DB-CAP-DISC-011

CapabilityId ≠ SQL Keyword.

## DB-CAP-DISC-012

CapabilityId ≠ Vendor Feature Name.

## DB-CAP-DISC-013

Capability Definition ≠ Capability Evidence.

## DB-CAP-DISC-014

Capability scope será explícito.

## DB-CAP-DISC-015

Connection capability no se asumirá server-global.

## DB-CAP-DISC-016

DEEP discovery ≠ destructive permission.

## DB-CAP-DISC-017

Discovery context no será Service Locator.

## DB-CAP-DISC-018

Evidence conservará provenance.

## DB-CAP-DISC-019

Evidence ≠ Decision.

## DB-CAP-DISC-020

Confidence ≠ Capability Status.

## DB-CAP-DISC-021

Confidence ≠ Provider Priority.

## DB-CAP-DISC-022

Más evidencia específica no ocultará contradicciones.

## DB-CAP-DISC-023

Vendor + Version ≠ Capability.

## DB-CAP-DISC-024

DBMS Version ≠ Driver Version.

## DB-CAP-DISC-025

Driver Version ≠ Client Library Version.

## DB-CAP-DISC-026

MySQL ≠ MariaDB.

## DB-CAP-DISC-027

SQLite Version ≠ Complete Capability Set.

## DB-CAP-DISC-028

Extension Installed ≠ Extension Usable.

## DB-CAP-DISC-029

Driver Claim ≠ Proven Behavior.

## DB-CAP-DISC-030

Connection Metadata ≠ Global Server Truth.

## DB-CAP-DISC-031

Session evidence tendrá scope apropiado.

## DB-CAP-DISC-032

Probe ≠ Application Query.

## DB-CAP-DISC-033

Probe ≠ Migration.

## DB-CAP-DISC-034

Probe Failure ≠ Unsupported.

## DB-CAP-DISC-035

Permission Denied ≠ Unsupported.

## DB-CAP-DISC-036

Timeout ≠ Unsupported.

## DB-CAP-DISC-037

Probe safety será explícita.

## DB-CAP-DISC-038

Destructive probes estarán prohibidos por defecto.

## DB-CAP-DISC-039

Probe budget será acotado.

## DB-CAP-DISC-040

Discovery deadline no transformará evidencia faltante en unsupported.

## DB-CAP-DISC-041

Probe deberá preservar session state.

## DB-CAP-DISC-042

Failed session restoration marcará connection como insegura.

## DB-CAP-DISC-043

Rollback no se asumirá reversible para todo DDL.

## DB-CAP-DISC-044

Temporary resources tendrán ownership explícito.

## DB-CAP-DISC-045

Provider ≠ Resolver.

## DB-CAP-DISC-046

Provider ordering ≠ last provider wins.

## DB-CAP-DISC-047

Conflicting evidence será preservada.

## DB-CAP-DISC-048

Raw Evidence ≠ Canonical Evidence.

## DB-CAP-DISC-049

Evidence será validada antes de resolución.

## DB-CAP-DISC-050

Plugins no redefinirán core capabilities silenciosamente.

## DB-CAP-DISC-051

Discovery Registry será frozen después de bootstrap.

## DB-CAP-DISC-052

Definition State ≠ Observation State.

## DB-CAP-DISC-053

Capability Snapshot ≠ Eternal Truth.

## DB-CAP-DISC-054

Evidence Snapshot ≠ Resolved Capability Snapshot.

## DB-CAP-DISC-055

Snapshots serán inmutables.

## DB-CAP-DISC-056

Capability Fingerprint no incluirá secrets.

## DB-CAP-DISC-057

TTL ≠ Correctness Guarantee.

## DB-CAP-DISC-058

Session evidence no será cacheada como server-global.

## DB-CAP-DISC-059

Cache Failure ≠ Capability Unsupported.

## DB-CAP-DISC-060

Persistent runtime no asumirá snapshot eterna.

## DB-CAP-DISC-061

Una operación utilizará una vista consistente de capabilities.

## DB-CAP-DISC-062

Mid-operation discovery change no modificará retroactivamente el plan.

## DB-CAP-DISC-063

Endpoint A Capability ≠ Endpoint B Capability.

## DB-CAP-DISC-064

Aggregate Capability ≠ Endpoint Capability.

## DB-CAP-DISC-065

Read Routing respetará requirements.

## DB-CAP-DISC-066

Replica Lag ≠ Capability.

## DB-CAP-DISC-067

Shard capability drift será observable.

## DB-CAP-DISC-068

Scatter operation requerirá capabilities de todos los targets necesarios.

## DB-CAP-DISC-069

Tenant identity ≠ Capability.

## DB-CAP-DISC-070

Tenant A puede resolver a capabilities distintas de Tenant B.

## DB-CAP-DISC-071

DBMS Feature Capability ≠ Schema Availability.

## DB-CAP-DISC-072

NotObserved ≠ Absent.

## DB-CAP-DISC-073

Platform Supported ≠ Effectively Usable.

## DB-CAP-DISC-074

Discovery seguirá least privilege.

## DB-CAP-DISC-075

Discovery no escalará credenciales automáticamente.

## DB-CAP-DISC-076

Runtime Capability ≠ Database Capability.

## DB-CAP-DISC-077

Composite requirements podrán combinar múltiples capability domains.

## DB-CAP-DISC-078

Planner preguntará capabilities, no vendor names.

## DB-CAP-DISC-079

Compiler no ejecutará discovery.

## DB-CAP-DISC-080

Planner no ejecutará probes arbitrarios.

## DB-CAP-DISC-081

Planner/Compiler consumirán snapshots resueltas.

## DB-CAP-DISC-082

Compiled artifacts declararán relevant capability dependencies cuando sea viable.

## DB-CAP-DISC-083

Migration capability discovery ≠ Migration Safety.

## DB-CAP-DISC-084

DB Capability ≠ Environment Tool Capability.

## DB-CAP-DISC-085

Test declaration ≠ Real Observation.

## DB-CAP-DISC-086

Conformance evidence deberá corresponder a versión/contexto compatibles.

## DB-CAP-DISC-087

Diagnostics preservarán evidence provenance.

## DB-CAP-DISC-088

UNKNOWN ≠ UNSUPPORTED.

## DB-CAP-DISC-089

Unsupported ≠ No Possible Emulation.

## DB-CAP-DISC-090

Discovery no decidirá support mode.

## DB-CAP-DISC-091

Custom Driver podrá contribuir evidence.

## DB-CAP-DISC-092

Custom Dialect no ejecutará probes.

## DB-CAP-DISC-093

Compiler requirement ≠ Capability Evidence.

## DB-CAP-DISC-094

Query Extensions podrán declarar requirements.

## DB-CAP-DISC-095

ORM Extensions podrán declarar requirements.

## DB-CAP-DISC-096

Third-party probes serán tratados como privileged code.

## DB-CAP-DISC-097

Probe manifest declarará side effects.

## DB-CAP-DISC-098

Hidden probe operations estarán prohibidas.

## DB-CAP-DISC-099

Fakes no probarán DBMS capabilities reales.

## DB-CAP-DISC-100

Real DB behavior requerirá real integration evidence.

---

# 263. Invariantes adicionales de seguridad

## DB-CAP-DISC-101

Secrets no aparecerán en provenance.

## DB-CAP-DISC-102

Secrets no formarán parte de fingerprints.

## DB-CAP-DISC-103

Probe SQL será sanitizado en diagnostics.

## DB-CAP-DISC-104

Discovery no ejecutará arbitrary user SQL.

## DB-CAP-DISC-105

Probe permissions serán explícitas.

## DB-CAP-DISC-106

Admin discovery requerirá activación explícita.

## DB-CAP-DISC-107

Third-party destructive probes permanecerán deshabilitados por defecto.

## DB-CAP-DISC-108

Capability cache no almacenará credentials.

## DB-CAP-DISC-109

Telemetry no expondrá connection strings sensibles.

## DB-CAP-DISC-110

Probe cleanup failure será visible.

---

# 264. Invariantes de runtime

## DB-CAP-DISC-111

Frozen definitions podrán compartirse entre requests.

## DB-CAP-DISC-112

Mutable discovery state será scoped.

## DB-CAP-DISC-113

No habrá static mutable current connection.

## DB-CAP-DISC-114

No habrá static mutable current tenant.

## DB-CAP-DISC-115

No habrá static mutable current capability snapshot por request.

## DB-CAP-DISC-116

Connection-specific evidence no sobrevivirá a connection generation inválida.

## DB-CAP-DISC-117

Server evidence deberá estar vinculada a server identity.

## DB-CAP-DISC-118

Topology changes podrán invalidar aggregate capability views.

## DB-CAP-DISC-119

Provider registry generation formará parte de invalidation cuando aplique.

## DB-CAP-DISC-120

Probe execution no dejará state residual entre requests.

---

# 265. Invariantes de observabilidad

## DB-CAP-DISC-121

Toda decisión explicable conservará acceso a evidence relevante.

## DB-CAP-DISC-122

Evidence conflicts serán diagnosticables.

## DB-CAP-DISC-123

Probe failures serán distinguibles de unsupported results.

## DB-CAP-DISC-124

Cache hits serán distinguibles de fresh discovery.

## DB-CAP-DISC-125

Stale evidence será identificable.

## DB-CAP-DISC-126

Snapshot generation será observable.

## DB-CAP-DISC-127

Capability drift será observable.

## DB-CAP-DISC-128

Provider identity será observable.

## DB-CAP-DISC-129

Probe identity será observable.

## DB-CAP-DISC-130

Diagnostics no modificarán capability state.

---

# 266. Anti-patrones

## 266.1 `if PostgreSQL then supported`

Incorrecto como regla universal.

---

## 266.2 `if version >= X then true`

Puede ser evidencia.

No decisión universal.

---

## 266.3 `probe failed => unsupported`

Incorrecto.

---

## 266.4 `permission denied => unsupported`

Incorrecto.

---

## 266.5 `timeout => unsupported`

Incorrecto.

---

## 266.6 Usar SQLite para inferir MySQL/PostgreSQL capabilities

Incorrecto.

---

## 266.7 Tratar MySQL y MariaDB como la misma plataforma

Incorrecto.

---

## 266.8 Cachear session state como server-global

Incorrecto.

---

## 266.9 Ejecutar CREATE/DROP en cada request para descubrir capacidades

Incorrecto.

---

## 266.10 Probe sin timeout

Incorrecto.

---

## 266.11 Probe sin cleanup

Incorrecto.

---

## 266.12 Probe modificando search_path y no restaurándolo

Incorrecto.

---

## 266.13 Devolver conexión dirty al pool

Incorrecto.

---

## 266.14 Compiler ejecutando probes

Incorrecto.

---

## 266.15 Query Planner haciendo network calls ocultas

Incorrecto.

---

## 266.16 Plugin redefiniendo capability core por orden de registro

Incorrecto.

---

## 266.17 Last provider wins

Incorrecto.

---

## 266.18 Ocultar conflicting evidence

Incorrecto.

---

## 266.19 Tratar snapshot como verdad eterna

Incorrecto.

---

## 266.20 Utilizar un único capability set para todo un cluster heterogéneo

Incorrecto.

---

## 266.21 `writer capability == replica capability`

Incorrecto.

---

## 266.22 `shard A capability == shard B capability`

Incorrecto.

---

## 266.23 Incluir passwords en fingerprint

Prohibido.

---

## 266.24 Discovery con superuser por defecto

Incorrecto.

---

## 266.25 Capability probe dependiendo de production business data

Incorrecto.

---

# 267. Modelo formal de evidencia

Sea una capability:

```text
C
```

y su conjunto de evidencia:

```text
E(C) = {e₁, e₂, ..., eₙ}
```

Cada evidencia podrá modelarse como:

```text
eᵢ =
(
 value,
 source,
 scope,
 confidence,
 provenance,
 time
)
```

El Discovery System produce:

```text
D(C) = E(C)
```

No:

```text
D(C) = Supported(C)
```

La decisión pertenece al resolver:

```text
Resolve(E(C), Policy)
→ CapabilityDecision
```

---

# 268. UNKNOWN formal

Si no existe evidencia suficiente:

```text
EvidenceInsufficient(C)
```

entonces:

```text
Resolve(C)
=
UNKNOWN
```

No:

```text
UNSUPPORTED
```

---

# 269. Probe failure formal

```text
Probe(C) = FAILED
```

no implica:

```text
C = UNSUPPORTED
```

Formalmente:

```text
ProbeFailure(C)
↛
Unsupported(C)
```

---

# 270. Scope formal

Para una capability contextual:

```text
C(scope₁)
```

no puede inferirse necesariamente:

```text
C(scope₂)
```

Por ejemplo:

```text
C(endpointA)
↛
C(endpointB)
```

---

# 271. Effective Usability

Una capability efectivamente utilizable podrá requerir:

```text
Usable(C)
=
PlatformSupport(C)
∧
DriverSupport(C)
∧
Permission(C)
∧
Configuration(C)
∧
ContextCompatibility(C)
```

cuando esas dimensiones sean relevantes.

---

# 272. Composite Feature

Para una feature `F`:

```text
F
requires
{C₁, C₂, ..., Cₙ}
```

entonces:

```text
Usable(F)
=
RequirementExpression(
    Resolve(C₁),
    Resolve(C₂),
    ...,
    Resolve(Cₙ)
)
```

---

# 273. Capability Drift

Sean:

```text
S₁ = snapshot at t₁
S₂ = snapshot at t₂
```

Existe drift relevante si:

```text
RelevantCapabilities(S₁)
≠
RelevantCapabilities(S₂)
```

---

# 274. Fingerprint

Podrá modelarse:

```text
F =
H(
 PlatformIdentity
 || ServerIdentity
 || ServerVersion
 || DriverIdentity
 || DriverVersion
 || RelevantConfiguration
 || InstalledExtensions
 || NormalizedEvidence
)
```

con exclusión explícita de secretos.

---

# 275. Aggregate Topology Capability

Para endpoints:

```text
E = {e₁ ... eₙ}
```

podrán existir vistas:

```text
ALL(C)
=
∀ e ∈ E : Supports(e, C)
```

y:

```text
ANY(C)
=
∃ e ∈ E : Supports(e, C)
```

Pero:

```text
ANY(C)
≠
ALL(C)
```

---

# 276. Query Target Requirement

Si un Query Plan requiere:

```text
C
```

y su target set es:

```text
T
```

entonces deberá cumplirse:

```text
∀ target ∈ T:
    Satisfies(target, C)
```

salvo que el plan contenga explícitamente una estrategia de fallback válida.

---

# 277. Probe Safety Formula

Un probe sólo podrá ejecutarse cuando:

```text
Allowed(Probe)
=
SafetyPolicyAllows
∧
PermissionsKnownSufficient
∧
BudgetAvailable
∧
DeadlineAvailable
∧
ContextCompatible
```

---

# 278. Discovery Cache Validity

```text
CacheValid
=
SameEnvironmentIdentity
∧
SameRelevantGeneration
∧
NotExpiredByPolicy
∧
ScopeCompatible
∧
NoKnownInvalidation
```

---

# 279. Arquitectura consolidada

```text
                    Capability Catalog
                           │
                           ▼
                 Discovery Request
                           │
                           ▼
              Capability Discovery System
                           │
        ┌──────────────────┼────────────────────┐
        ▼                  ▼                    ▼
 Platform Provider    Driver Provider      Plugin Provider
        │                  │                    │
        ├──────────────────┼────────────────────┤
        ▼                  ▼                    ▼
 Server Metadata    Connection Metadata    Runtime Probes
        │                  │                    │
        └──────────────────┼────────────────────┘
                           ▼
                     Raw Evidence
                           │
                           ▼
                 Evidence Normalizer
                           │
                           ▼
                  Evidence Validator
                           │
                           ▼
                  Conflict Detector
                           │
                           ▼
             Capability Evidence Snapshot
                           │
                           ▼
                 Capability Resolver
                           │
                           ▼
             Resolved Capability Snapshot
                           │
       ┌───────────────────┼────────────────────┐
       ▼                   ▼                    ▼
 Query Planner         Compiler             Routing
       │                   │                    │
       └───────────────────┼────────────────────┘
                           ▼
                     Execution
```

Cross-cutting:

```text
Cache
Telemetry
Diagnostics
Security
Testing
Persistent Runtime
Topology
Plugins
```

---

# 280. Flujo de bootstrap

```text
Framework Bootstrap
      ↓
Database Configuration
      ↓
Platform Registry
      ↓
Driver Registry
      ↓
Capability Catalog
      ↓
Core Discovery Providers
      ↓
Plugin Discovery Providers
      ↓
Provider Validation
      ↓
Conflict Validation
      ↓
Freeze Registry
```

Todavía no implica necesariamente ejecutar probes.

---

# 281. Flujo de conexión

```text
Connection Requested
      ↓
Physical Connection Established
      ↓
Connection Identity
      ↓
Server Identity
      ↓
Cached Evidence Lookup
      ↓
Required Discovery
      ↓
Evidence Snapshot
      ↓
Capability Resolution
      ↓
Resolved Snapshot
```

---

# 282. Flujo de query

```text
Query Model
     ↓
Semantic Analysis
     ↓
Capability Requirements
     ↓
Resolved Capability Snapshot
     ↓
Planner
     ↓
Physical Plan
     ↓
Compiler
     ↓
Execution
```

No:

```text
Compiler
  ↓
probe database
```

---

# 283. Flujo de topología

```text
Topology
│
├── Writer
│      └── Capability Snapshot W
│
├── Replica A
│      └── Capability Snapshot A
│
└── Replica B
       └── Capability Snapshot B
            │
            ▼
      Aggregate Topology View
            │
            ▼
        Routing Engine
```

---

# 284. Flujo de invalidación

```text
Server Restart
Driver Change
Configuration Change
Extension Change
Topology Change
Plugin Generation Change
       │
       ▼
Capability Evidence Invalidation
       │
       ▼
Snapshot Generation Change
       │
       ▼
Capability Fingerprint Change
       │
       ▼
Dependent Artifact Invalidation
```

---

# 285. Estrategia V1

La primera implementación deberá priorizar:

```text
CapabilityId
CapabilityDefinition
CapabilityCatalog

CapabilityDiscoverySystem
CapabilityDiscoveryRequest
CapabilityDiscoveryContext

CapabilityDiscoveryProvider
CapabilityProviderRegistry

CapabilityEvidence
CapabilityEvidenceSet
CapabilityEvidenceSource
EvidenceConfidence
EvidenceProvenance

CapabilityEvidenceSnapshot
CapabilityFingerprint

CapabilityProbe
CapabilityProbeResult
ProbeSafetyClass
ProbeBudget

CapabilityDiscoveryCache

MySQL Discovery Provider
MariaDB Discovery Provider
PostgreSQL Discovery Provider
SQLite Discovery Provider

Driver Discovery Provider

Conflict Detection
UNKNOWN Handling
Diagnostics
Telemetry
Testing
```

---

# 286. Probes iniciales V1

La V1 deberá ser conservadora.

Priorizará probes:

```text
metadata-only
read-only
low-cost
non-destructive
```

---

# 287. V1 Runtime Rule

Durante requests normales:

```text
SAFE discovery
```

será el máximo habitual.

`DEEP` deberá reservarse para operaciones explícitas.

---

# 288. Evolución futura

Posteriormente podrán añadirse:

```text
distributed capability discovery
cluster-wide drift detection
capability negotiation
remote capability registries
deployment compatibility checks
pre-deployment capability validation
capability change notifications
fleet capability inventory
automated extension compatibility analysis
```

---

# 289. Deployment Compatibility

En el futuro podrá verificarse:

```text
Application Requirements
          ↓
Target Environment Capabilities
          ↓
Compatibility Report
```

antes del despliegue.

---

# 290. Example

Una aplicación podría declarar:

```text
requires:
- database.query.cte
- database.transaction.savepoint
- database.query.json

optional:
- database.query.returning
```

VoltStack podría validar el entorno antes de iniciar tráfico.

---

# 291. Capability Negotiation

Una futura capa podrá elegir:

```text
native strategy
fallback strategy
emulated strategy
degraded strategy
```

según capabilities.

Pero esta negociación pertenecerá a los consumidores/resolvers apropiados.

No al Discovery System.

---

# 292. Regla final

> **El Database Capability Discovery System de VoltStack será una infraestructura de observación y evidencia, no una colección de `if vendor/version`. Su responsabilidad será determinar qué se conoce realmente sobre un entorno, con qué alcance, confianza y procedencia, preservando incertidumbre y contradicciones en lugar de convertirlas en certezas artificiales.**

Por tanto:

```text
Discovery
≠
Decision
```

```text
Evidence
≠
Support
```

```text
Vendor + Version
≠
Capability
```

```text
Version
≠
Configuration
```

```text
Driver Claim
≠
Proven Behavior
```

```text
Probe
≠
Capability
```

```text
Probe Failure
≠
Unsupported
```

```text
Permission Denied
≠
Unsupported
```

```text
Timeout
≠
Unsupported
```

```text
UNKNOWN
≠
UNSUPPORTED
```

```text
Extension Installed
≠
Extension Usable
```

```text
Platform Supported
≠
Effectively Usable
```

```text
Feature Capability
≠
Schema Availability
```

```text
NotObserved
≠
Absent
```

```text
Endpoint A
≠
Endpoint B
```

```text
ANY
≠
ALL
```

```text
Capability Snapshot
≠
Eternal Truth
```

```text
TTL
≠
Correctness
```

y finalmente:

```text
Reliable Capability Discovery
=
Stable Capability Identity
+
Explicit Scope
+
Multiple Evidence Sources
+
Provenance
+
Confidence
+
Safe Probes
+
Permission Awareness
+
Timeouts
+
Conflict Preservation
+
Correct UNKNOWN Semantics
+
Endpoint Awareness
+
Topology Awareness
+
Immutable Snapshots
+
Capability Fingerprints
+
Cache Invalidation
+
Persistent Runtime Safety
+
Diagnostics
+
Telemetry
+
Security
+
Real Integration Testing
```

---

# 293. Cierre del Bloque 30 — Extensibility

Con este documento queda definida la arquitectura principal de extensibilidad de VoltStack Database:

```text
294_DATABASE_EXTENSION_ARCHITECTURE
│
├── 295_DATABASE_PLUGIN_SYSTEM
│
├── 296_DATABASE_CUSTOM_DRIVER_SYSTEM
│
├── 297_DATABASE_CUSTOM_DIALECT_SYSTEM
│
├── 298_DATABASE_CUSTOM_COMPILER_SYSTEM
│
├── 299_DATABASE_CUSTOM_QUERY_EXTENSION_SYSTEM
│
├── 300_DATABASE_CUSTOM_ORM_EXTENSION_SYSTEM
│
└── 301_DATABASE_DATABASE_CAPABILITY_DISCOVERY_SYSTEM
```

La arquitectura resultante permite extender:

```text
Database
├── Plugins
├── Drivers
├── Dialects
├── Compilers
├── Query Language
├── ORM
└── Capability Discovery
```

sin permitir que la extensibilidad destruya las fronteras centrales:

```text
ORM
    ↓
Query Engine
    ↓
Planner
    ↓
Compiler
    ↓
Execution Engine
    ↓
Connection
    ↓
Driver
    ↓
DBMS
```

y manteniendo:

```text
Extension
≠
Core Mutation
```

como principio estructural.

---

# 294. Siguiente bloque

El siguiente documento inicia:

```text
Block 31 — Developer Experience / Public API
```

con:

```text
302_DATABASE_PUBLIC_API_SYSTEM.md
```

Este bloque definirá cómo toda la arquitectura construida hasta ahora será presentada al programador de VoltStack mediante una API coherente, simple y fuertemente tipada.

La secuencia será:

```text
302_DATABASE_PUBLIC_API_SYSTEM.md
303_DATABASE_FACADE_SYSTEM.md
304_DATABASE_HELPER_SYSTEM.md
305_DATABASE_MODEL_DEVELOPER_EXPERIENCE.md
306_DATABASE_QUERY_DEVELOPER_EXPERIENCE.md
307_DATABASE_SCHEMA_DEVELOPER_EXPERIENCE.md
308_DATABASE_ERROR_MESSAGE_AND_DIAGNOSTIC_EXPERIENCE.md
309_DATABASE_CLI_SYSTEM.md
310_DATABASE_CODE_GENERATION_SYSTEM.md
```

La siguiente regla comenzará a tomar especial importancia:

> **La complejidad interna de VoltStack Database no deberá convertirse automáticamente en complejidad para el desarrollador.**

Es decir:

```text
Internal Architectural Sophistication
≠
Public API Complexity
```

y:

```text
Simple Public API
≠
Simplistic Internal Architecture
```