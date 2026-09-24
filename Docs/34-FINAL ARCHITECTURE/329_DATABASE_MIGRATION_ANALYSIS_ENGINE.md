# 329_DATABASE_MIGRATION_ANALYSIS_ENGINE.md

## 1. Propósito

Este documento define la arquitectura del **Migration Analysis Engine** de `VoltStack/Quantum/Database`.

El componente se denomina conceptualmente:

```text
DatabaseMigrationAnalysisEngine
```

y constituye el coordinador central encargado de analizar una aplicación antes de transformar su capa de persistencia.

Se apoya directamente en:

```text
325_DATABASE_MIGRATION_FROM_EXTERNAL_ORM_SYSTEM.md
326_DATABASE_ELOQUENT_MIGRATION_ADAPTER.md
327_DATABASE_DOCTRINE_MIGRATION_ADAPTER.md
328_DATABASE_LEGACY_DATABASE_MIGRATION_SYSTEM.md
```

Su función no es modificar código ni schema.

Su función es:

```text
discover
collect
correlate
classify
validate
measure
explain
plan inputs
```

antes de permitir cualquier migración.

---

## 2. Problema

Una aplicación real puede contener simultáneamente:

```text
Eloquent
Doctrine ORM
Doctrine DBAL
PDO
raw SQL
custom repositories
legacy DAOs
stored procedures
multiple databases
custom database abstractions
```

Analizar cada tecnología de forma aislada no es suficiente.

Es necesario construir una visión unificada:

```text
Application Persistence Architecture
```

---

## 3. Objetivo principal

Convertir múltiples fuentes de evidencia:

```text
Source Code
Composer Metadata
Framework Configuration
ORM Metadata
Migration Files
Database Schema
Runtime Observations
Tests
Developer Overrides
```

en un único:

```text
Migration Analysis Model
```

que permita decidir:

```text
what exists
where it exists
what depends on it
what can migrate automatically
what requires adaptation
what is risky
what blocks migration
```

---

## 4. Principio fundamental

```text
Analyze before transforming.
```

VoltStack no deberá ejecutar una transformación relevante sin disponer previamente de suficiente evidencia sobre su impacto.

---

## 5. Arquitectura general

```text
Application
    │
    ├── Source Code
    ├── Configuration
    ├── Composer
    ├── ORM Metadata
    ├── Schema
    └── Runtime Evidence
          │
          ▼
Migration Source Adapters
          │
          ▼
Migration Analysis Engine
          │
    ┌─────┼──────────┐
    ▼     ▼          ▼
Discover Correlate  Classify
    │     │          │
    └─────┴────┬─────┘
               ▼
      Migration Analysis Model
               │
               ▼
      Intermediate Model
               │
               ▼
       Migration Rule Engine
```

---

## 6. Responsabilidades

El Analysis Engine deberá:

```text
detect migration sources
invoke source adapters
collect findings
merge duplicate findings
resolve source relationships
detect conflicts
build dependency graphs
calculate migration boundaries
identify blockers
classify migration compatibility
assign confidence
identify security findings
identify transaction boundaries
identify runtime lifecycle risks
produce analysis snapshots
feed planning/reporting systems
```

---

## 7. No responsabilidades

El Analysis Engine no deberá:

```text
rewrite PHP
modify database schema
execute migrations
remove packages
change configuration
perform cutover
```

Estas operaciones pertenecen a componentes posteriores.

---

## 8. Source Adapter Registry

El motor utilizará:

```text
MigrationSourceAdapterRegistry
```

que podrá contener:

```text
EloquentMigrationAdapter
DoctrineMigrationAdapter
LegacyDatabaseMigrationAdapter
ThirdPartyMigrationAdapter
ProjectSpecificMigrationAdapter
```

---

## 9. Adapter Contract

Conceptualmente:

```php
interface MigrationSourceAdapterInterface
{
    public function supports(
        MigrationAnalysisContext $context
    ): bool;

    public function discover(
        MigrationAnalysisContext $context
    ): MigrationSourceDiscovery;

    public function analyze(
        MigrationSourceDiscovery $source
    ): MigrationSourceAnalysis;
}
```

---

## 10. Multi-source Analysis

Una aplicación no estará limitada a un adapter.

Ejemplo:

```text
Application
├── Eloquent
├── PDO
└── Stored Procedures
```

activará:

```text
Eloquent Adapter
+
Legacy Adapter
```

simultáneamente.

---

## 11. Analysis Context

Cada ejecución utilizará un contexto explícito:

```text
MigrationAnalysisContext
```

con:

```text
project root
source paths
environment
configuration
target VoltStack version
database platforms
enabled adapters
analysis mode
security policy
runtime evidence
developer overrides
```

---

## 12. Request-local / Run-local State

El Analysis Engine no dependerá de estado global mutable.

Cada ejecución deberá aislar:

```text
findings
caches
graphs
runtime observations
configuration
```

en su propio contexto.

Esto es importante para ejecución concurrente y runtimes persistentes.

---

## 13. Analysis Modes

El sistema soportará:

```text
STATIC
STATIC_PLUS_SCHEMA
STATIC_PLUS_RUNTIME
FULL
```

---

## 14. STATIC

Analiza únicamente:

```text
source code
composer metadata
configuration
migration files
metadata files
```

sin conexión a database.

---

## 15. STATIC_PLUS_SCHEMA

Añade introspección de:

```text
tables
columns
indexes
constraints
foreign keys
views
sequences
triggers
```

---

## 16. STATIC_PLUS_RUNTIME

Combina análisis estático con evidencia de ejecución controlada.

---

## 17. FULL

Combina:

```text
static
schema
runtime
tests
developer overrides
```

para máxima confianza.

---

## 18. Evidence Model

Toda conclusión deberá tener evidencia asociada.

Conceptualmente:

```text
MigrationEvidence
```

con:

```text
type
source
location
value/reference
confidence
timestamp when relevant
adapter
```

---

## 19. Evidence Types

```text
SOURCE_CODE
COMPOSER
CONFIG
ORM_METADATA
MIGRATION_HISTORY
DATABASE_SCHEMA
RUNTIME
TEST
USER_OVERRIDE
INFERRED
```

---

## 20. Evidence Provenance

El motor deberá poder responder:

```text
Why do we believe this?
```

Ejemplo:

```text
Table:
users

Evidence:
Eloquent convention
Laravel migration
production-like schema introspection
```

---

## 21. Confidence Model

Cada conclusión podrá tener:

```text
HIGH
MEDIUM
LOW
UNKNOWN
```

La confianza no sustituye la clasificación de compatibilidad.

---

## 22. Compatibility Classification

Se conservará:

```text
DIRECT
TRANSFORMABLE
ADAPTABLE
MANUAL
UNSUPPORTED
RISKY
```

---

## 23. Confidence vs Compatibility

Ejemplo:

```text
Compatibility:
DIRECT

Confidence:
LOW
```

significa que la transformación sería directa **si** la inferencia es correcta, pero falta evidencia.

---

## 24. Finding Model

Cada hallazgo se representará conceptualmente mediante:

```text
MigrationFinding
```

con:

```text
id
category
severity
source adapter
location
subject
description
evidence
compatibility
confidence
suggested action
dependencies
```

---

## 25. Stable Finding IDs

Los findings deberán usar identificadores estables cuando representen reglas conocidas.

Ejemplos:

```text
VSDB-MIG-ELOQ-...
VSDB-MIG-DOC-...
VSDB-MIG-LEGACY-...
VSDB-MIG-CORE-...
```

---

## 26. Severity

Los niveles podrán ser:

```text
INFO
NOTICE
WARNING
ERROR
BLOCKER
```

---

## 27. BLOCKER

Un blocker impide una fase de migración concreta.

Ejemplos:

```text
unknown transaction semantics
unresolved schema conflict
unknown custom type storing critical data
unsupported identity strategy
```

No necesariamente impide analizar el resto del proyecto.

---

## 28. Discovery Phase

Primera fase:

```text
Project
  │
  ▼
Discover Sources
```

Se detectarán:

```text
ORM packages
DB libraries
database configuration
model/entity paths
repositories
raw SQL
migration systems
schema sources
database connections
```

---

## 29. Discovery Result

Ejemplo:

```text
Detected Persistence Sources

Eloquent              YES
Doctrine ORM           NO
Doctrine DBAL          YES
PDO                    YES
Custom DB Wrapper      YES
Stored Procedures      YES
```

---

## 30. Source Inventory

El motor generará un inventario inicial:

```text
Models / Entities
Repositories
Queries
Connections
Transactions
Types
Relationships
Callbacks
Procedures
Schema Objects
```

---

## 31. Static Code Scanner

El scanner común deberá proveer servicios de AST para los adapters.

Esto evita que cada adapter implemente un parser PHP completo.

---

## 32. Shared AST Index

Conceptualmente:

```text
Project AST Index
│
├── classes
├── interfaces
├── traits
├── methods
├── properties
├── calls
├── imports
├── literals
└── source locations
```

---

## 33. Symbol Resolution

El sistema deberá resolver:

```text
namespace
use statements
inheritance
interfaces
traits
method declarations
class constants
```

para reducir falsos positivos.

---

## 34. Call Graph

Cuando sea posible se construirá:

```text
Method A
   ↓
Repository B
   ↓
Query C
```

para comprender dependencias.

---

## 35. Persistence Call Graph

Una vista especializada mostrará:

```text
Controller
   ↓
Service
   ↓
Repository
   ↓
ORM/DBAL/PDO
   ↓
Database
```

---

## 36. Dependency Graph

El grafo general podrá contener nodos:

```text
module
class
entity
repository
query
connection
transaction
table
procedure
type
listener
```

---

## 37. Edge Types

Ejemplos:

```text
DEPENDS_ON
QUERIES
WRITES
READS
MANAGES
CALLS
MAPS_TO
OWNS_TRANSACTION
USES_TYPE
OBSERVES
```

---

## 38. Strongly Connected Components

El motor deberá detectar grupos fuertemente conectados.

Ejemplo:

```text
Entity A ↔ Entity B ↔ Repository C
```

Esto ayuda a identificar unidades de migración que no deben separarse arbitrariamente.

---

## 39. Migration Boundary Detection

El motor podrá sugerir fronteras:

```text
module
bounded context
repository group
entity aggregate
database
connection
```

basándose en dependencias observadas.

---

## 40. Boundary Safety

Una frontera propuesta deberá considerar:

```text
shared transactions
shared writes
shared identity
cross-module relations
shared custom types
global listeners
```

---

## 41. Query Inventory

El motor consolidará queries provenientes de:

```text
Eloquent Builder
Doctrine DQL
Doctrine DBAL
PDO
raw SQL
custom query builders
stored procedures
```

---

## 42. Query Normalization

Sin transformar aún código, podrá producir:

```text
QueryDescriptor
```

con:

```text
operation
sources
joins
predicates
projection
ordering
pagination
locking
parameters
result shape
platform
```

---

## 43. Query Fingerprinting

Queries semánticamente similares podrán agruparse.

Ejemplo:

```text
User::where('email', ...)
PDO SELECT ... WHERE email = ?
Doctrine DQL ... WHERE u.email = :email
```

podrían representar la misma intención.

---

## 44. Duplicate Persistence Logic

El motor podrá detectar múltiples implementaciones de la misma consulta o regla de acceso.

Esto será información de modernización, no una orden automática de consolidación.

---

## 45. Read/Write Classification

Cada query deberá clasificarse:

```text
READ
WRITE
DDL
TRANSACTION_CONTROL
PROCEDURE
UNKNOWN
```

---

## 46. Table Access Matrix

Ejemplo:

```text
                 users   orders   invoices

UserRepo          RW
OrderService       R       RW
BillingDAO                         RW
ReportService      R       R        R
```

Esta matriz ayuda a definir ownership.

---

## 47. Write Ownership Analysis

Las escrituras tienen mayor riesgo que las lecturas.

El motor deberá identificar:

```text
single writer
multiple writers
unknown writer
cross-module writer
```

---

## 48. Transaction Analysis

El sistema construirá:

```text
TransactionDescriptor
```

con:

```text
begin location
commit location
rollback paths
participating queries
participating repositories
connection
isolation
retry behavior
nested behavior
```

---

## 49. Transaction Graph

Ejemplo:

```text
CheckoutService
      │
      ▼
BEGIN
 ├── OrderRepository
 ├── InventoryDAO
 ├── PaymentRepository
 └── Audit SQL
      │
      ▼
COMMIT
```

Migrar solo una parte puede ser riesgoso.

---

## 50. Cross-technology Transactions

El motor deberá detectar:

```text
Eloquent + PDO
Doctrine + DBAL
legacy DAO + stored procedure
```

dentro de una misma transacción.

---

## 51. Transaction Blocker

Si no puede demostrarse que una frontera de migración conserva atomicidad:

```text
BLOCKER
```

para esa frontera.

---

## 52. Schema Evidence

El motor podrá obtener schema mediante:

```text
migration files
ORM metadata
SQL files
database introspection
```

---

## 53. Three-way / Multi-way Comparison

Ejemplo:

```text
ORM Metadata
      +
Migration History
      +
Actual Schema
      +
Legacy SQL assumptions
```

deberán correlacionarse.

---

## 54. Schema Conflict Model

Un conflicto deberá incluir:

```text
object
property
source A value
source B value
source C value
impact
resolution required
```

---

## 55. Actual Database Is Not Always Design Truth

El schema actual representa realidad operacional, pero puede contener:

```text
drift
manual hotfixes
obsolete columns
temporary structures
```

Por tanto, tampoco será asumido automáticamente como intención arquitectónica.

---

## 56. Runtime Evidence

El motor podrá importar evidencia capturada por observers.

Ejemplos:

```text
executed query fingerprints
connection usage
transaction duration
lazy loads
query counts
runtime-selected table
runtime-selected connection
```

---

## 57. Runtime Coverage

Se deberá registrar:

```text
observation period
test suite coverage
traffic source
environment
```

para no presentar evidencia parcial como exhaustiva.

---

## 58. Runtime Observation Safety

El Analysis Engine nunca requerirá capturar:

```text
passwords
tokens
PII
full query values
```

para funcionar.

---

## 59. Test Evidence

Los tests existentes pueden revelar:

```text
expected result shape
exceptions
transaction behavior
relationship behavior
```

El sistema podrá vincular tests con componentes migrables.

---

## 60. Characterization Gap

Si un componente crítico carece de tests:

```text
CHARACTERIZATION_GAP
```

podrá generarse como finding.

---

## 61. Configuration Analysis

El motor deberá descubrir:

```text
database config
environment variable names
connection names
read/write topology
tenant resolvers
pooling options
timeouts
```

sin almacenar secretos.

---

## 62. Composer Analysis

Se analizarán:

```text
database packages
ORM packages
extensions
versions
plugins
abandoned packages
```

---

## 63. Extension Detection

Los adapters podrán registrar extensiones conocidas.

Una extensión desconocida que modifique persistencia deberá elevar la incertidumbre.

---

## 64. Custom Type Analysis

El motor consolidará tipos provenientes de:

```text
Doctrine Types
Eloquent Casts
legacy codecs
database-native types
VoltStack mappings
```

---

## 65. Type Dependency Graph

Ejemplo:

```text
MoneyType
  │
  ├── Invoice.total
  ├── Payment.amount
  └── Refund.amount
```

Esto permite migrar el tipo antes de las entidades dependientes.

---

## 66. Identifier Analysis

Se deberán identificar:

```text
auto increment
sequence
UUID
ULID
composite ID
assigned ID
foreign ID
custom generator
```

---

## 67. Identifier Blockers

Una estrategia de identidad no comprendida será crítica porque puede causar colisiones o pérdida de identidad.

---

## 68. Relationship Analysis

El motor consolidará:

```text
ORM relationships
foreign keys
join tables
legacy joins
implicit associations
```

sin asumir que un JOIN ocasional constituye una relación de dominio.

---

## 69. Relationship Confidence

Ejemplo:

```text
Doctrine metadata + FK
→ HIGH

Repeated SQL JOIN only
→ MEDIUM/LOW
```

---

## 70. Lifecycle Analysis

Se consolidarán:

```text
Eloquent events
Doctrine callbacks
observers
listeners
triggers
legacy hooks
```

---

## 71. Duplicate Side Effect Detection

Ejemplo:

```text
Eloquent created event
+
database trigger
```

ambos escriben auditoría.

La migración deberá evitar duplicar el efecto.

---

## 72. Stored Procedure Analysis

El motor deberá relacionar procedures con:

```text
call sites
tables affected
transactions
result sets
```

cuando pueda obtener evidencia suficiente.

---

## 73. Trigger Analysis

Los triggers se incorporarán al grafo de comportamiento para explicar efectos que no aparecen en PHP.

---

## 74. View Analysis

Views y materialized views deberán registrarse como fuentes de consulta y dependencias de schema.

---

## 75. Security Analysis

El motor podrá producir findings de migración sobre:

```text
SQL injection risk
hard-coded credentials
unsafe dynamic identifiers
sensitive SQL logging
excessive database privileges
```

---

## 76. Security vs Migration

El Analysis Engine no sustituye una auditoría de seguridad completa.

Su objetivo es impedir que una migración preserve o amplifique patrones inseguros evidentes.

---

## 77. Persistent Runtime Analysis

VoltStack deberá evaluar compatibilidad con:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

## 78. Persistent State Findings

Se detectarán patrones:

```text
static EntityManager
static PDO
global connection
static tenant
unreset filters
open transaction
persistent Identity Map
session-level DB state
```

---

## 79. FrankenPHP Profile

Como runtime predeterminado, el reporte deberá incluir una sección:

```text
FrankenPHP Readiness
```

con findings sobre aislamiento entre requests.

---

## 80. Analysis Cache

El análisis estático puede ser costoso.

Se podrá mantener:

```text
AnalysisCache
```

basado en:

```text
file hash
configuration hash
adapter version
rule version
target VoltStack version
```

---

## 81. Incremental Analysis

Si cambia un archivo:

```text
re-analyze changed node
+
affected graph dependents
```

en lugar de analizar siempre todo el proyecto.

---

## 82. Determinism

Con:

```text
same source
same configuration
same schema snapshot
same adapters
same rules
```

el resultado deberá ser determinista.

---

## 83. Analysis Snapshot

Cada ejecución podrá producir:

```text
MigrationAnalysisSnapshot
```

con:

```text
snapshot id
source revision
timestamp
tool versions
adapter versions
schema fingerprint
findings
graphs
statistics
```

---

## 84. Source Revision

Si Git está disponible podrá registrarse:

```text
commit hash
dirty working tree flag
```

sin requerir Git para funcionar.

---

## 85. Schema Fingerprint

El schema podrá representarse mediante un fingerprint para detectar cambios entre análisis.

---

## 86. Diff Analysis

Dos snapshots podrán compararse:

```text
New findings
Resolved findings
Changed findings
New legacy dependencies
Removed dependencies
Schema changes
```

---

## 87. CI Mode

Ejemplo:

```bash
php volt database:migrate:analyze --ci
```

podrá fallar ante:

```text
new blockers
new critical legacy dependencies
new unsafe raw SQL
migration regression
```

---

## 88. Baseline Mode

Proyectos grandes podrán crear:

```text
migration-analysis.baseline
```

para aceptar temporalmente deuda existente.

---

## 89. No New Debt

Política:

```text
New migration debt <= baseline
```

hasta reducirla progresivamente.

---

## 90. Analysis Profiles

Podrán existir:

```text
quick
standard
deep
ci
production-safe
```

---

## 91. Quick Profile

Analiza principalmente:

```text
composer
source symbols
known models/entities
obvious raw SQL
```

para feedback rápido.

---

## 92. Deep Profile

Incluye:

```text
full AST
call graph
schema
runtime evidence
transaction graph
cross-source correlation
```

---

## 93. Production-safe Profile

No ejecutará:

```text
mutations
unsafe introspection
data sampling
unbounded runtime tracing
```

---

## 94. Developer Overrides

El usuario podrá declarar:

```text
known table mapping
known custom type mapping
known transaction boundary
known morph map
known legacy wrapper
known false positive
```

---

## 95. Override Provenance

Un override será evidencia:

```text
USER_OVERRIDE
```

y aparecerá explícitamente en reportes.

No deberá disfrazarse de detección automática.

---

## 96. Suppressions

Un finding podrá suprimirse mediante ID y justificación.

Ejemplo conceptual:

```text
VSDB-MIG-CORE-0123
reason: "Legacy reporting module intentionally remains SQL-first."
```

---

## 97. Suppression Rules

Una suppression:

```text
must have reason
must be scoped
must be auditable
```

y opcionalmente podrá expirar.

---

## 98. False Positive Management

El sistema deberá permitir marcar findings falsos sin eliminar la regla global.

---

## 99. Analysis Result

Conceptualmente:

```php
final readonly class MigrationAnalysisResult
{
    public function sources(): array;

    public function findings(): array;

    public function dependencyGraph(): MigrationDependencyGraph;

    public function transactions(): array;

    public function schemaConflicts(): array;

    public function blockers(): array;

    public function statistics(): MigrationStatistics;
}
```

---

## 100. Statistics

Podrá incluir:

```text
source files scanned
models/entities
repositories
queries
raw SQL
transactions
connections
custom types
procedures
blockers
manual items
transformable items
```

---

## 101. No Single Migration Score

El motor no deberá reducir todo el proyecto a un número opaco.

En su lugar mostrará dimensiones concretas:

```text
Schema
Queries
Transactions
Types
Lifecycle
Runtime
Security
Tests
```

---

## 102. Readiness Matrix

Ejemplo:

```text
Area             Direct  Manual  Risky  Blocked

Entities             42       3      1        0
Queries             186      12      4        2
Transactions          14       2      1        1
Custom Types           5       2      0        0
Lifecycle             21       5      2        1
```

---

## 103. Blocker Graph

Un blocker deberá mostrar qué componentes impide migrar.

```text
Custom Money Type
      │
      ├── Invoice
      ├── Payment
      └── Refund
```

---

## 104. Critical Path

El motor podrá identificar dependencias que deben resolverse primero.

Ejemplo:

```text
Connection
   ↓
Custom Type
   ↓
Entity Group
   ↓
Repository
   ↓
Module Cutover
```

---

## 105. Migration Unit Candidate

Conceptualmente:

```text
MigrationUnitCandidate
```

con:

```text
name
components
dependencies
read/write tables
transactions
blockers
confidence
```

---

## 106. Planning Boundary

El Analysis Engine propone candidatos.

El sistema de planificación decide finalmente qué migrar y en qué orden.

---

## 107. Analysis Events

Podrán emitirse:

```text
MigrationAnalysisStarted
MigrationSourceDetected
MigrationAdapterStarted
MigrationFindingCreated
MigrationConflictDetected
MigrationAnalysisCompleted
```

---

## 108. Telemetry

Métricas posibles:

```text
database.migration.analysis.duration
database.migration.analysis.files
database.migration.analysis.findings
database.migration.analysis.blockers
database.migration.analysis.cache_hits
database.migration.analysis.adapters
```

---

## 109. Logging

Canal:

```text
database.migration.analysis
```

con redacción obligatoria de información sensible.

---

## 110. Parallel Analysis

Adapters independientes podrán analizarse concurrentemente cuando:

```text
no shared mutable state
deterministic merge
resource limits respected
```

---

## 111. Merge Phase

Los resultados paralelos deberán converger mediante un proceso determinista.

```text
Adapter Results
      │
      ▼
Canonical Sort
      │
      ▼
Deduplication
      │
      ▼
Correlation
      │
      ▼
Conflict Detection
```

---

## 112. Deduplication

Ejemplo:

```text
Eloquent detects users table
Schema analyzer detects users table
Legacy SQL detects users table
```

no debe producir tres objetos físicos diferentes.

---

## 113. Entity Resolution

El motor deberá distinguir:

```text
same class
same table
same entity
```

porque no son equivalentes.

Dos entidades pueden mapear la misma tabla.

---

## 114. Shared Table Mapping

Este caso deberá aparecer explícitamente porque puede afectar:

```text
identity
writes
lifecycle
migration ownership
```

---

## 115. Cross-adapter Conflict

Ejemplo:

```text
Eloquent:
users.id = integer

Doctrine:
users.id = UUID

Actual schema:
users.id = bigint
```

Resultado:

```text
MIGRATION_METADATA_CONFLICT
```

---

## 116. Conflict Resolver

El sistema no deberá escoger silenciosamente.

Podrá:

```text
request override
accept explicit rule
use verified schema evidence
mark blocker
```

---

## 117. Confidence Escalation

La confianza podrá aumentar al agregar evidencia.

```text
Static inference      LOW
+ migration file      MEDIUM
+ actual schema       HIGH
```

cuando no existan contradicciones.

---

## 118. Confidence Reduction

Una contradicción deberá reducir confianza aunque existan muchas fuentes.

Cantidad de evidencia no sustituye consistencia.

---

## 119. Unknowns

El reporte deberá tener una sección explícita:

```text
Unknowns
```

No deberán ocultarse bajo valores por defecto.

---

## 120. Assumptions

Toda suposición utilizada para continuar deberá registrarse:

```text
Assumption
Evidence
Impact
How to verify
```

---

## 121. Analysis Completeness

El motor podrá reportar qué fuentes fueron analizadas:

```text
Source code      COMPLETE
Schema           COMPLETE
Runtime          NOT PROVIDED
Tests            PARTIAL
Production data  NOT REQUIRED
```

---

## 122. No False Certainty

El Analysis Engine nunca deberá reportar:

```text
Safe to migrate
```

solo porque no encontró errores.

Deberá diferenciar:

```text
verified
not observed
not analyzed
unknown
```

---

## 123. Migration Manifest Input

Los resultados podrán alimentar:

```text
.orm-migration-manifest.json
```

o el manifest general definido por Migration Core.

---

## 124. Machine-readable Output

El motor deberá producir una representación estructurada para:

```text
CLI
CI
IDE
reports
code transformer
rule engine
```

---

## 125. Human-readable Output

También deberá producir información comprensible:

```text
what was found
where
why it matters
what can be done
what requires review
```

---

## 126. IDE Integration

Un IDE podrá mostrar findings directamente sobre:

```text
model
repository
query
transaction
```

mediante códigos estables.

---

## 127. Incremental IDE Analysis

Para DX, el motor podrá ejecutar análisis parcial sobre archivos modificados.

---

## 128. Memory Limits

Grandes codebases requieren:

```text
streamed indexing
bounded caches
graph partitioning
incremental persistence
```

para no mantener todo el AST simultáneamente.

---

## 129. Large Repository Strategy

Pipeline:

```text
Index
 ↓
Persist Symbols
 ↓
Analyze Partitions
 ↓
Merge Findings
 ↓
Build Global Graph
```

---

## 130. Analysis Storage

Los artefactos temporales podrán almacenarse en:

```text
.voltstack/cache/database-migration/
```

o ubicación configurable.

No deberán incluir secretos.

---

## 131. Cache Invalidation

Cambios en:

```text
source
composer.lock
database config
adapter version
migration rules
schema fingerprint
```

invalidarán las partes relevantes.

---

## 132. Extension API

Terceros podrán añadir analyzers mediante:

```php
interface MigrationAnalyzerExtensionInterface
{
    public function analyze(
        MigrationAnalysisContext $context,
        MigrationAnalysisCollector $collector
    ): void;
}
```

---

## 133. Extension Isolation

Una extensión no deberá poder alterar findings de otro analyzer silenciosamente.

Podrá:

```text
add evidence
add findings
propose resolution
```

pero las modificaciones deberán quedar trazables.

---

## 134. Rule Registry Integration

El Analysis Engine podrá consultar el Rule Registry para determinar si un finding posee:

```text
known transformer
known validator
known workaround
```

sin ejecutar todavía la transformación.

---

## 135. Target Version Awareness

El análisis deberá conocer:

```text
target VoltStack Database version
```

porque una feature puede ser:

```text
supported in V1
unsupported in V1
planned for V2
```

---

## 136. V1 Scope Enforcement

Para esta especificación, el target será:

```text
VoltStack Database V1
```

El Analysis Engine no deberá considerar una capacidad futura de V2 como disponible durante la migración V1.

---

## 137. Unsupported-but-Preservable

Una feature puede no tener abstracción nativa pero ser preservable mediante:

```text
native SQL
stored procedure
compatibility adapter
```

Esto es diferente de `UNSUPPORTED`.

---

## 138. Risk Categories

Además de severity podrán existir categorías:

```text
DATA_INTEGRITY
TRANSACTION
SECURITY
BEHAVIOR
PERFORMANCE
RUNTIME
PORTABILITY
MAINTAINABILITY
```

---

## 139. Data Integrity Priority

Findings de:

```text
identity
foreign keys
decimal conversion
NULL behavior
encryption
```

tendrán prioridad alta.

---

## 140. Transaction Priority

Findings que puedan cambiar atomicidad serán blockers potenciales aunque el código sea fácil de transformar.

---

## 141. Security Priority

Una migración automática no deberá convertir una vulnerabilidad conocida en código nuevo sin advertencia.

---

## 142. Behavior Priority

La equivalencia de comportamiento tiene prioridad sobre similitud sintáctica.

---

## 143. Performance Findings

Los findings de performance normalmente no bloquearán análisis, pero podrán bloquear cutover si la regresión supera políticas definidas.

---

## 144. Runtime Findings

Problemas de state leakage bajo FrankenPHP deberán tratarse como críticos para producción.

---

## 145. Analysis Policy

El proyecto podrá configurar:

```text
block_on_schema_conflict
block_on_unknown_transactions
block_on_security_findings
require_tests_for_critical_paths
require_runtime_analysis_for_dynamic_queries
```

---

## 146. Strict Mode

En:

```text
strict=true
```

la herramienta preferirá:

```text
manual review
```

sobre inferencias de confianza media.

---

## 147. Assisted Mode

En modo asistido podrá proponer transformaciones con:

```text
confidence
evidence
required verification
```

pero sin aplicarlas automáticamente.

---

## 148. Automation Eligibility

Un elemento solo será elegible para transformación automática cuando:

```text
source semantics known
target semantics known
mapping rule exists
confidence sufficient
no unresolved blocker
validator available
```

---

## 149. Safe Automation Formula

```text
Automation Eligible
=
Known Source
+
Known Target
+
Deterministic Rule
+
Sufficient Evidence
+
Validation Path
```

---

## 150. Analysis Pipeline

```text
BOOT
 ↓
LOAD CONFIG
 ↓
INDEX PROJECT
 ↓
DETECT SOURCES
 ↓
RUN ADAPTERS
 ↓
COLLECT EVIDENCE
 ↓
NORMALIZE FINDINGS
 ↓
CORRELATE
 ↓
BUILD GRAPHS
 ↓
DETECT CONFLICTS
 ↓
CLASSIFY
 ↓
CALCULATE CONFIDENCE
 ↓
IDENTIFY BLOCKERS
 ↓
GENERATE SNAPSHOT
```

---

## 151. Failure Isolation

Si un adapter falla:

```text
Doctrine adapter failed
```

el motor podrá continuar con:

```text
Eloquent
Legacy
Schema
```

y marcar:

```text
ANALYSIS_INCOMPLETE
```

---

## 152. Partial Analysis

Un fallo parcial nunca deberá presentarse como análisis completo.

---

## 153. Timeout Handling

Analyzers costosos deberán soportar límites.

Si expiran:

```text
TIMED_OUT
```

será registrado junto con el alcance no analizado.

---

## 154. Resource Budgets

Podrán configurarse:

```text
max memory
max runtime
max runtime observations
max SQL samples
```

---

## 155. Reproducibility

Cada snapshot deberá registrar versiones de:

```text
PHP
VoltStack
Analysis Engine
adapters
rules
database platform
```

cuando sean relevantes.

---

## 156. Testing Architecture

El Analysis Engine tendrá:

```text
unit tests
adapter contract tests
fixture projects
integration tests
schema tests
determinism tests
performance tests
security tests
```

---

## 157. Fixture Projects

Se mantendrán proyectos de prueba representativos:

```text
Eloquent-only
Doctrine-only
PDO legacy
Hybrid
Multi-database
Dynamic tenancy
Stored-procedure-heavy
```

---

## 158. Golden Analysis Snapshots

Fixtures podrán producir snapshots esperados para detectar regresiones del analyzer.

---

## 159. Determinism Tests

Dos análisis sobre la misma fixture deberán producir findings equivalentes y orden estable.

---

## 160. False Positive Tests

Cada regla deberá incluir casos negativos.

---

## 161. Security Tests

Se comprobará que:

```text
passwords
tokens
SQL values
PII
```

no aparezcan accidentalmente en outputs.

---

## 162. Concurrency Tests

El motor deberá poder analizar proyectos en paralelo sin mezclar:

```text
findings
cache namespaces
configuration
snapshots
```

---

## 163. Persistent Worker Tests

Aunque normalmente sea CLI, los servicios reutilizables deberán ser seguros si se ejecutan dentro de procesos persistentes.

---

## 164. Performance Targets

El analyzer deberá privilegiar:

```text
incremental analysis
shared indexing
lazy graph expansion
bounded memory
```

en proyectos grandes.

---

## 165. CLI Base

Comando conceptual:

```bash
php volt database:migrate:analyze
```

---

## 166. Source Selection

```bash
php volt database:migrate:analyze \
    --source=eloquent
```

o:

```bash
--source=doctrine
--source=legacy
```

---

## 167. Full Analysis

```bash
php volt database:migrate:analyze \
    --profile=deep
```

---

## 168. Schema Analysis

```bash
php volt database:migrate:analyze \
    --with-schema
```

---

## 169. Snapshot

```bash
php volt database:migrate:analyze \
    --snapshot
```

---

## 170. Compare Snapshots

Conceptualmente:

```bash
php volt database:migrate:analysis:diff \
    baseline.json \
    current.json
```

---

## 171. Example Summary

```text
VoltStack Database Migration Analysis

Sources
-------
Eloquent            detected
Doctrine DBAL       detected
Legacy PDO          detected

Inventory
---------
Entities                 84
Repositories              31
Queries                  426
Raw SQL                   73
Transactions              29
Custom Types               7
Stored Procedures         11

Findings
--------
Direct                   318
Transformable            142
Adaptable                 47
Manual                    26
Risky                     12
Unsupported                2
Blockers                   4
```

---

## 172. Example Blocker

```text
[VSDB-MIG-CORE-0107] BLOCKER

Transaction:
CheckoutService::complete()

Sources:
Eloquent
Legacy PDO

Problem:
Both persistence systems participate in the
same transaction, but PDO is created from a
different physical connection.

Impact:
Atomicity cannot be preserved automatically.

Required action:
Unify connection ownership or define an
explicit transaction migration strategy.
```

---

## 173. Example Conflict

```text
[VSDB-MIG-CORE-0214] ERROR

Table:
payments

Column:
amount

Doctrine:
decimal(12,2)

Legacy mapper:
float

Actual schema:
decimal(18,4)

Action:
Resolve expected precision before automatic
type migration.
```

---

## 174. Example Persistent Runtime Finding

```text
[VSDB-MIG-CORE-0312] WARNING

Class:
Legacy\Database

Finding:
Static PDO connection retained globally.

Target Runtime:
FrankenPHP

Risk:
Connection session state may leak between
requests.

Action:
Move connection ownership to VoltStack
ConnectionManager and verify request reset.
```

---

## 175. Integration with 330

La salida normalizada del Analysis Engine alimentará:

```text
330_DATABASE_MIGRATION_INTERMEDIATE_MODEL.md
```

El documento 330 formalizará las estructuras neutrales.

---

## 176. Integration with 331

El Rule Engine utilizará:

```text
findings
evidence
compatibility
confidence
target capabilities
```

para seleccionar reglas.

---

## 177. Integration with 332

El Code Transformer solo podrá transformar elementos previamente identificados por el Analysis Engine.

---

## 178. Integration with 333

Los conflictos de schema se enviarán al Schema Compatibility System.

---

## 179. Integration with 334

Los contratos de comportamiento identificados alimentarán el Behavior Verification System.

---

## 180. Integration with 335

Los límites donde coexistirán runtimes se utilizarán para configurar Dual ORM Runtime.

---

## 181. Integration with 336

Los QueryDescriptors servirán para configurar shadow comparisons.

---

## 182. Integration with 337

Los gaps y critical paths se utilizarán para generar la estrategia de testing.

---

## 183. Integration with 338

CLI y DX presentarán el análisis de forma navegable.

---

## 184. Integration with 339

El sistema de reporting transformará findings y snapshots en reportes detallados.

---

## 185. Integration with 340

Rollback utilizará snapshots previos para conocer:

```text
what changed
what dependencies were replaced
what compatibility layers existed
```

---

## 186. Arquitectura de componentes

```text
DatabaseMigrationAnalysisEngine
│
├── MigrationAnalysisContextFactory
├── MigrationSourceAdapterRegistry
├── ProjectIndexer
├── AstIndex
├── ComposerAnalyzer
├── ConfigurationAnalyzer
├── SchemaEvidenceProvider
├── RuntimeEvidenceProvider
├── TestEvidenceProvider
├── MigrationEvidenceStore
├── MigrationFindingCollector
├── FindingDeduplicator
├── EvidenceCorrelator
├── DependencyGraphBuilder
├── TransactionGraphBuilder
├── SchemaConflictDetector
├── CompatibilityClassifier
├── ConfidenceEvaluator
├── MigrationBoundaryAnalyzer
├── BlockerDetector
├── AnalysisCache
└── AnalysisSnapshotBuilder
```

---

## 187. Flujo interno

```text
Context
  │
  ▼
Project Index
  │
  ▼
Source Detection
  │
  ▼
Adapters
  │
  ▼
Evidence Store
  │
  ▼
Finding Collector
  │
  ▼
Deduplication
  │
  ▼
Correlation
  │
  ├── Dependency Graph
  ├── Transaction Graph
  └── Schema Graph
  │
  ▼
Classification
  │
  ▼
Confidence
  │
  ▼
Blockers
  │
  ▼
Analysis Snapshot
```

---

## 188. Decisiones arquitectónicas

### Decisión 1

Toda migración Database V1 deberá comenzar con una fase formal de análisis.

### Decisión 2

El Analysis Engine será independiente de cualquier ORM específico.

### Decisión 3

Eloquent, Doctrine y Legacy serán adapters especializados.

### Decisión 4

Una aplicación podrá activar múltiples adapters simultáneamente.

### Decisión 5

Toda conclusión relevante deberá mantener provenance de evidencia.

### Decisión 6

Compatibilidad y confianza serán dimensiones diferentes.

### Decisión 7

Los conflictos entre fuentes nunca se resolverán silenciosamente.

### Decisión 8

El sistema construirá grafos de dependencias y transacciones.

### Decisión 9

La atomicidad tendrá prioridad sobre facilidad de transformación.

### Decisión 10

El análisis será static-first y runtime-assisted.

### Decisión 11

El runtime evidence nunca será considerado cobertura exhaustiva por sí solo.

### Decisión 12

El motor será determinista y reproducible.

### Decisión 13

Se soportará análisis incremental y cacheado.

### Decisión 14

El análisis podrá ejecutarse en CI para impedir nueva deuda legacy.

### Decisión 15

Los outputs serán simultáneamente machine-readable y human-readable.

### Decisión 16

VoltStack Database V1 no utilizará features futuras de V2 para declarar una migración compatible.

### Decisión 17

No se reducirá el estado de migración a un score opaco.

### Decisión 18

La automatización solo será elegible cuando exista una ruta verificable.

---

## 189. Resultado esperado

Antes:

```text
Application
├── Eloquent
├── Doctrine
├── PDO
├── SQL
└── Stored Procedures
```

Después del análisis:

```text
Unified Migration Analysis
│
├── Sources
├── Entities
├── Queries
├── Connections
├── Transactions
├── Schema
├── Types
├── Lifecycle
├── Security Findings
├── Runtime Risks
├── Dependency Graph
├── Migration Boundaries
├── Manual Items
└── Blockers
```

Este modelo se convierte en la base objetiva para las siguientes fases.

---

## 190. Principio final

```text
Discover
   ↓
Collect Evidence
   ↓
Correlate
   ↓
Detect Conflicts
   ↓
Classify
   ↓
Measure Confidence
   ↓
Build Dependencies
   ↓
Identify Blockers
   ↓
Only Then Transform
```

---

## 191. Conclusión

`DATABASE_MIGRATION_ANALYSIS_ENGINE` es la capa que transforma una migración desde una colección de sustituciones técnicas en un proceso arquitectónico controlado.

Su propósito es impedir que VoltStack migre código que todavía no comprende suficientemente.

La regla central será:

```text
No transformation without analysis.

No analysis conclusion without evidence.

No automation without a verification path.
```

Con este motor, Eloquent, Doctrine, PDO, SQL directo y sistemas legacy dejan de analizarse como mundos aislados y pasan a formar parte de una única representación de la arquitectura de persistencia que VoltStack deberá migrar.

---

**Documento:** `329_DATABASE_MIGRATION_ANALYSIS_ENGINE.md`  
**Proyecto:** VoltStack Framework  
**Módulo:** `VoltStack/Quantum/Database`  
**Versión objetivo:** Database V1  
**Estado:** Architectural Specification
