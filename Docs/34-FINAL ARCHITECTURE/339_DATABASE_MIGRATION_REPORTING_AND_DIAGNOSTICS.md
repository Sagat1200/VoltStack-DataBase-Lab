# 339_DATABASE_MIGRATION_REPORTING_AND_DIAGNOSTICS.md

## 1. Propósito

Este documento define la arquitectura oficial del **Database Migration Reporting and Diagnostics System** de `VoltStack/Quantum/Database`.

El componente se denomina conceptualmente:

```text
DatabaseMigrationReportingAndDiagnosticsSystem
```

y constituye la capa que transforma toda la evidencia producida por el pipeline de migración en información:

```text
understandable
traceable
actionable
auditable
comparable over time
machine-readable
```

Su responsabilidad principal es responder:

```text
What is the real state of the migration?

Why is it in that state?

What changed?

What remains?

What is blocking progress?

What evidence supports the conclusion?

What should be investigated next?
```

---

## 2. Posición dentro de Database V1

Este documento es el penúltimo documento de la especificación `VoltStack/Quantum/Database V1`.

Continúa:

```text
325_DATABASE_MIGRATION_FROM_EXTERNAL_ORM_SYSTEM.md
326_DATABASE_ELOQUENT_MIGRATION_ADAPTER.md
327_DATABASE_DOCTRINE_MIGRATION_ADAPTER.md
328_DATABASE_LEGACY_DATABASE_MIGRATION_SYSTEM.md
329_DATABASE_MIGRATION_ANALYSIS_ENGINE.md
330_DATABASE_MIGRATION_INTERMEDIATE_MODEL.md
331_DATABASE_MIGRATION_RULE_ENGINE.md
332_DATABASE_MIGRATION_CODE_TRANSFORMER.md
333_DATABASE_MIGRATION_SCHEMA_COMPATIBILITY_SYSTEM.md
334_DATABASE_MIGRATION_BEHAVIOR_VERIFICATION_SYSTEM.md
335_DATABASE_DUAL_ORM_RUNTIME_SYSTEM.md
336_DATABASE_SHADOW_QUERY_AND_RESULT_COMPARISON_SYSTEM.md
337_DATABASE_MIGRATION_TESTING_AND_VALIDATION_SYSTEM.md
338_DATABASE_MIGRATION_CLI_AND_DEVELOPER_EXPERIENCE.md
```

y alimenta directamente:

```text
340_DATABASE_MIGRATION_ROLLBACK_AND_RECOVERY_SYSTEM.md
```

---

## 3. Principio fundamental

```text
A migration report is not a collection of logs.

It is a structured explanation of migration state
backed by verifiable evidence.
```

---

# Parte I — Objetivos

## 4. Objetivos principales

El sistema deberá:

```text
aggregate migration evidence
normalize diagnostics
correlate findings
track migration progress
show blockers
show unresolved unknowns
measure coverage
show approved risks
compare snapshots
detect regressions
produce technical reports
produce operational summaries
export machine-readable artifacts
support auditability
support recovery decisions
```

---

## 5. No objetivos

No deberá:

```text
invent missing evidence
hide uncertainty
replace validation
replace observability
automatically approve migration states
automatically perform rollback
reduce migration safety to one opaque score
```

---

# Parte II — Reporting vs Diagnostics

## 6. Reporting

Reporting responde:

```text
What is the migration state?
```

---

## 7. Diagnostics

Diagnostics responde:

```text
Why is it in that state?
```

---

## 8. Relationship

```text
Evidence
   │
   ├── Reporting → summary / trends / status
   │
   └── Diagnostics → cause / evidence / next investigation
```

---

# Parte III — Evidence Sources

## 9. Sources

El sistema consumirá evidencia de:

```text
329 Migration Analysis
330 Intermediate Model
331 Migration Rules
332 Transformation Journal
333 Schema Compatibility
334 Behavior Verification
335 Dual Runtime
336 Shadow Comparison
337 Testing & Validation
338 CLI/DX
```

---

## 10. Recovery Evidence

También preparará información para:

```text
340 Rollback and Recovery
```

---

## 11. Source Independence

El sistema de reporting no deberá conocer detalles internos innecesarios de:

```text
Eloquent
Doctrine
PDO
legacy wrappers
```

cuando exista una representación neutral.

---

# Parte IV — Architecture

## 12. Components

```text
DatabaseMigrationReportingAndDiagnosticsSystem
│
├── MigrationReportingCoordinator
├── MigrationEvidenceAggregator
├── MigrationEvidenceResolver
├── MigrationDiagnosticNormalizer
├── MigrationDiagnosticCorrelator
├── MigrationRootCauseAnalyzer
├── MigrationProgressAnalyzer
├── MigrationCoverageAnalyzer
├── MigrationRiskAnalyzer
├── MigrationBlockerAnalyzer
├── MigrationTrendAnalyzer
├── MigrationSnapshotComparator
├── MigrationReportBuilder
├── MigrationReportRendererRegistry
├── MigrationDiagnosticCatalog
├── MigrationArtifactLinker
├── MigrationReportStore
└── MigrationReportingTelemetry
```

---

# Parte V — Canonical Reporting Model

## 13. Model

Todo renderer consumirá:

```text
MigrationReportModel
```

---

## 14. Purpose

Separar:

```text
evidence aggregation
```

de:

```text
presentation
```

---

## 15. Conceptual Structure

```text
MigrationReportModel
├── metadata
├── executiveSummary
├── migrationState
├── sourceSystems
├── migrationUnits
├── progress
├── blockers
├── risks
├── unknowns
├── schema
├── behavior
├── shadow
├── validation
├── runtime
├── performance
├── security
├── recoveryReadiness
├── legacyDependencies
├── coverage
├── trends
├── evidenceReferences
└── recommendedInvestigations
```

---

# Parte VI — Report Metadata

## 16. Metadata

Todo reporte deberá registrar:

```text
report ID
report version
generated at
migration session
source revision
dirty state where available
schema fingerprint
MIM version
rule-set version
runtime manifest version
validation plan fingerprint
environment classification
```

---

## 17. Reproducibility

El reporte deberá permitir identificar exactamente:

```text
which migration state was observed
```

---

# Parte VII — Report Types

## 18. Standard Reports

V1 deberá contemplar:

```text
Migration Status Report
Migration Analysis Report
Migration Plan Report
Migration Transformation Report
Migration Schema Report
Migration Behavior Report
Migration Shadow Report
Migration Validation Report
Migration Risk Report
Migration Progress Report
Migration Cutover Readiness Report
Migration Legacy Dependency Report
Migration Recovery Readiness Report
Migration Diagnostic Report
```

---

# Parte VIII — Status Report

## 19. Purpose

Mostrar una visión global y breve.

---

## 20. Example

```text
VoltStack Database Migration Status

Migration units: 8

LEGACY               1
PREPARED             1
SHADOW_READ          2
VOLTSTACK_READ       1
VOLTSTACK_WRITE      1
VOLTSTACK_PRIMARY    1
LEGACY_REMOVED       1

Blocking findings:   3
Critical risks:      1
Unknown semantics:   4
Legacy dependencies: 29
```

---

## 21. Important

El resumen no deberá ocultar:

```text
critical blockers
security failures
data integrity failures
```

---

# Parte IX — Migration Unit Report

## 22. Unit View

Cada unidad deberá poder inspeccionarse individualmente.

---

## 23. Example

```text
Migration Unit: Billing

State:
SHADOW_READ

Source:
Eloquent + Legacy PDO

Target:
VoltStack Database

Schema:
PASS

Behavior:
PASS_WITH_APPROVED_DIFFERENCE

Shadow:
2 unresolved mismatches

Transactions:
PASS

Security:
PASS

Runtime:
PASS

Recovery:
READY

Next Gate:
VOLTSTACK_READ

Gate Status:
BLOCKED
```

---

# Parte X — Executive Summary

## 24. Purpose

Ofrecer una vista compacta para:

```text
technical leads
architects
operations
project stakeholders
```

---

## 25. Content

```text
current migration stage
units migrated
units remaining
critical blockers
major risks
trend
next planned transitions
recovery readiness
```

---

## 26. No False Simplification

La vista ejecutiva deberá enlazar siempre a evidencia técnica.

---

# Parte XI — Technical Report

## 27. Technical View

Deberá contener:

```text
findings
source locations
evidence
dependency paths
schema conflicts
behavior contracts
query differences
transaction findings
runtime findings
validation failures
```

---

# Parte XII — Diagnostic Model

## 28. Diagnostic

Unidad fundamental:

```text
MigrationDiagnostic
```

---

## 29. Fields

```text
diagnostic ID
category
severity
status
title
summary
migration unit
subject
source
location
evidence
confidence
risk
possible causes
related diagnostics
blocking state
next investigation
documentation key
first seen
last seen
```

---

# Parte XIII — Stable Diagnostic IDs

## 30. IDs

Cada subsistema conservará sus namespaces.

Ejemplos:

```text
VSDB-MIG-ELOQ-xxxx
VSDB-MIG-DOC-xxxx
VSDB-MIG-LEGACY-xxxx
VSDB-MIG-SCHEMA-xxxx
VSDB-MIG-BEHAVIOR-xxxx
VSDB-MIG-SHADOW-xxxx
VSDB-MIG-VALIDATION-xxxx
VSDB-MIG-CLI-xxxx
VSDB-MIG-REPORT-xxxx
```

---

## 31. Stability

Un mismo problema lógico deberá conservar el mismo diagnostic identity cuando sea razonablemente posible.

---

# Parte XIV — Severity

## 32. Severity Levels

```text
INFO
NOTICE
WARNING
ERROR
BLOCKER
```

---

## 33. Severity vs Risk

Severity representa gravedad diagnóstica.

Risk representa impacto potencial sobre:

```text
data
security
behavior
operations
performance
recovery
```

---

## 34. Separate Concepts

No deberán fusionarse en un score único.

---

# Parte XV — Confidence

## 35. Confidence

```text
HIGH
MEDIUM
LOW
UNKNOWN
```

---

## 36. Example

```text
Severity: BLOCKER
Confidence: LOW
```

significa:

```text
if true, impact is severe,
but evidence is incomplete
```

---

# Parte XVI — Diagnostic Status

## 37. Status

```text
OPEN
ACKNOWLEDGED
APPROVED_DIFFERENCE
SUPPRESSED
RESOLVED
STALE
INCONCLUSIVE
```

---

## 38. Resolved

No significa que el registro desaparezca.

Debe conservarse para historial.

---

# Parte XVII — Evidence References

## 39. Evidence

Cada diagnóstico deberá apuntar a:

```text
analysis finding
MIM node
rule
transformation journal
schema snapshot
behavior contract
shadow comparison
validation run
runtime state
checkpoint
```

---

## 40. Evidence Graph

Conceptualmente:

```text
Diagnostic
    │
    ├── Evidence A
    ├── Evidence B
    └── Evidence C
```

---

# Parte XVIII — Evidence Provenance

## 41. Provenance

Toda evidencia deberá indicar:

```text
producer
timestamp
source revision
schema fingerprint
environment
```

cuando aplique.

---

## 42. Stale Evidence

El reporte deberá distinguir:

```text
valid evidence
```

de:

```text
stale evidence
```

---

# Parte XIX — Diagnostic Correlation

## 43. Correlator

```text
MigrationDiagnosticCorrelator
```

agrupará findings relacionados.

---

## 44. Example

```text
Schema:
users.deleted_at exists

Behavior:
soft-delete contract fails

Shadow:
3 extra rows

Root correlation:
target global soft-delete scope missing
```

---

## 45. Benefit

Evitar presentar tres síntomas como tres problemas independientes.

---

# Parte XX — Root Cause Analysis

## 46. Analyzer

```text
MigrationRootCauseAnalyzer
```

---

## 47. Important Limitation

V1 podrá producir:

```text
probable cause
supporting evidence
alternative causes
```

pero no afirmará certeza sin evidencia suficiente.

---

## 48. Example

```text
Possible root cause:
missing tenant scope

Evidence:
- target query lacks tenant predicate
- shadow result contains foreign tenant row
- behavior contract tenant isolation failed

Confidence:
HIGH
```

---

# Parte XXI — Causal Graph

## 49. Graph

Podrá representarse:

```text
Missing Tenant Filter
       │
       ├── Shadow mismatch
       ├── Security validation failure
       └── Cutover blocker
```

---

## 50. Purpose

Mostrar:

```text
cause → symptoms → consequences
```

---

# Parte XXII — Blocker Analysis

## 51. Blocker

```text
MigrationBlocker
```

---

## 52. Fields

```text
blocker ID
migration unit
blocked transition
reason
evidence
dependencies
possible resolution
```

---

## 53. Blocker Graph

Ejemplo:

```text
VOLTSTACK_WRITE
      │
      X
Transaction Validation
      │
      X
Unknown Cross-connection Transaction
```

---

## 54. Critical Path

El sistema podrá identificar:

```text
minimum blocker chain
```

que impide el siguiente gate.

---

# Parte XXIII — Recommended Investigation

## 55. Recommendation

El diagnóstico podrá sugerir:

```text
inspect transaction ownership
verify schema mapping
run deep validation
increase shadow coverage
review custom type
```

---

## 56. No Automatic Decision

La recomendación será una acción técnica sugerida, no una autorización automática.

---

# Parte XXIV — Progress Model

## 57. Progress

No se representará únicamente como:

```text
75%
```

---

## 58. Dimensions

```text
migration units
legacy dependencies
schema compatibility
behavior contracts
query coverage
validation coverage
runtime states
blockers
```

---

## 59. Example

```text
Migration Progress

Units:
6 / 10 at VoltStack primary or later

Legacy dependencies:
42 → 17

Critical blockers:
5 → 1

Behavior contracts:
124 / 130 verified

Required shadow queries:
89 / 94 observed
```

---

# Parte XXV — Burn-down

## 60. Burn-down Metrics

Especialmente:

```text
legacy imports
legacy DB calls
unresolved findings
unknown semantics
critical blockers
approved temporary adapters
```

---

## 61. Direction

La tendencia esperada es:

```text
toward zero
```

para dependencias que deben eliminarse.

---

# Parte XXVI — Migration Velocity

## 62. Velocity

Podrá medirse:

```text
units advanced per period
blockers resolved
legacy dependencies removed
```

---

## 63. Caution

No deberá usarse como sustituto de seguridad o calidad.

---

# Parte XXVII — Trend Analysis

## 64. Trend Analyzer

```text
MigrationTrendAnalyzer
```

---

## 65. Trends

```text
new blockers
resolved blockers
shadow mismatch rate
validation regressions
legacy dependency burn-down
coverage growth
performance changes
```

---

## 66. Snapshot Requirement

Los trends deberán basarse en snapshots comparables.

---

# Parte XXVIII — Snapshot Comparison

## 67. Command Concept

```bash
php volt database:migrate:report \
    --compare=<snapshot-a>:<snapshot-b>
```

---

## 68. Output

```text
new findings
resolved findings
changed severity
changed confidence
new unknowns
resolved unknowns
state transitions
schema changes
validation regressions
legacy dependency changes
```

---

# Parte XXIX — Regression Detection

## 69. Regression

Ejemplos:

```text
new Eloquent dependency
new schema conflict
new shadow mismatch
behavior contract changed from PASS to FAIL
new persistent-worker leak
performance regression
```

---

## 70. Regression Diagnostic

```text
MIGRATION_REGRESSION
```

---

# Parte XXX — Coverage Reporting

## 71. Coverage

Utilizará el modelo dimensional de 337.

---

## 72. Dimensions

```text
entities
tables
queries
transactions
behavior contracts
migration units
tenants
runtime profiles
```

---

## 73. Example

```text
Coverage

Entities:           42 / 42
Queries:           318 / 331
Transactions:       28 / 30
Behavior contracts: 95 / 98
Tenants observed:   17 / 24
Runtime profiles:    1 / 1
```

---

## 74. Missing Coverage

Se deberá listar lo no cubierto.

---

# Parte XXXI — Shadow Report

## 75. Source

Consume 336.

---

## 76. Content

```text
eligible operations
executed samples
matches
approved differences
mismatches
timeouts
errors
coverage
tenant distribution
performance delta
```

---

## 77. Example

```text
Shadow Report: Catalog

Samples:
128,421

MATCH:
128,372

APPROVED:
31

MISMATCH:
18

Critical:
0

High:
2

Required query coverage:
97%
```

---

# Parte XXXII — Mismatch Groups

## 78. Grouping

No listar miles de repeticiones idénticas.

---

## 79. Example

```text
Mismatch Group:
catalog.product.list / ORDER

Occurrences:
8,412

First seen:
...

Last seen:
...

Affected tenants:
3

Probable cause:
missing deterministic secondary sort
```

---

# Parte XXXIII — Schema Report

## 80. Content

```text
source expectations
actual schema
target expectations
compatibility
conflicts
drift
unsafe changes
data probes
```

---

## 81. High-risk Changes

Destacar:

```text
column narrowing
type conversion
constraint addition
identifier change
table/column removal
```

---

# Parte XXXIV — Behavior Report

## 82. Content

```text
contracts
status
approved differences
unverified contracts
critical failures
```

---

## 83. Contract Trace

Cada contrato deberá poder enlazar:

```text
source evidence
target implementation
tests
shadow evidence
```

---

# Parte XXXV — Transaction Report

## 84. Content

```text
known transactions
ownership
connections
isolation
nested behavior
retries
locks
cross-runtime risks
validation
```

---

## 85. Example

```text
Transaction:
billing.invoice.create

Owner:
VoltStack

Atomicity:
PASS

Isolation:
READ COMMITTED

Connections:
1

Rollback:
PASS

Concurrency:
PASS
```

---

# Parte XXXVI — Security Report

## 86. Content

```text
unsafe SQL
tenant isolation
credential leakage
PII logging
sensitive serialization
encryption mappings
auth-related data
```

---

## 87. Priority

Los findings de seguridad críticos deberán aparecer en:

```text
technical report
executive summary
cutover readiness
```

sin quedar ocultos por filtros por defecto.

---

# Parte XXXVII — Performance Report

## 88. Content

```text
query count
latency
memory
throughput
connection use
shadow overhead
```

---

## 89. Context

Toda comparación deberá incluir:

```text
environment
dataset
sample size
runtime
DB platform
```

---

## 90. No Misleading Benchmark

No comparar directamente mediciones de entornos incompatibles como si fueran equivalentes.

---

# Parte XXXVIII — Runtime Report

## 91. Content

```text
runtime state
primary runtime
shadow runtime
fallback status
persistent-worker readiness
connection lifecycle
tenant reset
transaction cleanup
```

---

## 92. FrankenPHP

Debe existir sección específica:

```text
FrankenPHP Persistent Worker Readiness
```

---

## 93. Example

```text
Request context reset       PASS
Tenant reset                PASS
Transaction cleanup         PASS
Persistence context reset   PASS
Memory retention            PASS
Connection session reset    PASS
```

---

# Parte XXXIX — Legacy Dependency Report

## 94. Purpose

Mostrar qué impide retirar:

```text
Eloquent
Doctrine
legacy persistence
```

---

## 95. Categories

```text
imports
runtime services
queries
serialized payloads
queue jobs
CLI commands
scheduled jobs
listeners
tests
configuration
packages
```

---

## 96. Burn-down

Ejemplo:

```text
Eloquent dependencies

Week 1: 184
Week 2: 132
Week 3:  71
Week 4:  23
```

---

# Parte XL — Approved Differences Report

## 97. Approved Differences

Deben mostrarse explícitamente.

---

## 98. Content

```text
difference
scope
reason
approval reference
risk
expiry/review
```

---

## 99. No Hidden Waivers

Un approved difference no desaparecerá de reporting.

---

# Parte XLI — Suppressed Diagnostics

## 100. Suppression Report

Debe existir:

```text
suppressed diagnostics
```

---

## 101. Required Fields

```text
ID
reason
scope
created
expiry
```

---

## 102. Expired Suppression

Volverá a:

```text
OPEN
```

o estado definido por policy.

---

# Parte XLII — Unknowns Report

## 103. First-class Unknowns

Los `UNKNOWN`, `AMBIGUOUS`, `CONFLICTED` y `OPAQUE` del MIM deberán aparecer.

---

## 104. Example

```text
Unknown:
legacy stored procedure side effects

Impact:
transaction behavior cannot be verified

Blocks:
Billing → VOLTSTACK_WRITE

Resolution:
manual inspection or controlled runtime observation
```

---

# Parte XLIII — Risk Report

## 105. Risk Categories

```text
DATA_INTEGRITY
TRANSACTION
SECURITY
BEHAVIOR
RUNTIME
PERFORMANCE
PORTABILITY
MAINTAINABILITY
RECOVERY
```

---

## 106. Risk Matrix

Podrá presentar:

```text
risk category
severity
confidence
affected units
blocking state
mitigation
```

---

## 107. No Aggregate Risk Score

V1 no utilizará un número único opaco.

---

# Parte XLIV — Cutover Readiness Report

## 108. Purpose

Preparar una transición de runtime.

---

## 109. Content

```text
requested transition
gate requirements
passed requirements
failed requirements
inconclusive requirements
stale evidence
rollback readiness
deployment compatibility
```

---

## 110. Example

```text
Cutover Readiness

Unit:
Billing

Transition:
SHADOW_READ → VOLTSTACK_READ

Schema             PASS
Behavior           PASS
Shadow coverage    PASS
Security           PASS
Performance        PASS
Recovery           PASS
Stale evidence     NONE

Decision state:
GATE SATISFIED
```

---

## 111. Important

El reporte informa el resultado del gate.

No ejecuta la transición.

---

# Parte XLV — Recovery Readiness Report

## 112. Purpose

Preparar 340.

---

## 113. Content

```text
last checkpoint
backup evidence
fallback runtime compatibility
schema reversibility
data reversibility
irreversible operations
recovery tests
recovery dependencies
```

---

## 114. Example

```text
Recovery Readiness

Runtime fallback:
READY

Schema rollback:
READY

Data rollback:
NOT REQUIRED

Backup:
VERIFIED

Irreversible operations:
NONE
```

---

# Parte XLVI — Irreversible Change Reporting

## 115. Barrier

Cambios irreversibles deberán destacarse.

---

## 116. Example

```text
IRREVERSIBLE BARRIER

Operation:
Drop legacy_status column

Impact:
Legacy runtime becomes incompatible.

Recovery:
Restore database snapshot or forward recovery.

Status:
NOT YET EXECUTED
```

---

# Parte XLVII — Report Profiles

## 117. Profiles

```text
summary
developer
technical
security
operations
cutover
audit
ci
```

---

## 118. Summary

Compacto.

---

## 119. Developer

Prioriza:

```text
files
rules
findings
next actions
```

---

## 120. Operations

Prioriza:

```text
runtime
capacity
deployment
rollback
health
```

---

## 121. Audit

Prioriza:

```text
decisions
evidence
approvals
state changes
timestamps
```

---

# Parte XLVIII — Renderer System

## 122. Registry

```text
MigrationReportRendererRegistry
```

---

## 123. Renderers

V1 podrá soportar:

```text
Console
JSON
Markdown
HTML
JUnit where applicable
SARIF where applicable
```

---

## 124. Separation

Los renderers no recalcularán estado.

---

# Parte XLIX — Console Report

## 125. Command

```bash
php volt database:migrate:report
```

---

## 126. Unit

```bash
php volt database:migrate:report \
    --unit=Billing
```

---

## 127. Profile

```bash
php volt database:migrate:report \
    --profile=cutover
```

---

# Parte L — Markdown Report

## 128. Export

```bash
php volt database:migrate:report \
    --format=markdown \
    --output=migration-report.md
```

---

## 129. Purpose

Útil para:

```text
code review
architecture review
change management
documentation
```

---

# Parte LI — JSON Report

## 130. Export

```bash
php volt database:migrate:report \
    --format=json \
    --output=migration-report.json
```

---

## 131. Versioning

El JSON tendrá:

```text
reportSchemaVersion
```

---

## 132. Stable Contract

Cambios incompatibles requerirán versión nueva.

---

# Parte LII — HTML Report

## 133. Purpose

Puede ofrecer una vista navegable offline.

---

## 134. Security

No deberá incluir automáticamente:

```text
secrets
full sensitive query parameters
production PII
```

---

# Parte LIII — SARIF

## 135. Use

Findings de código podrán exportarse a SARIF.

---

## 136. Scope

No todos los diagnósticos son naturalmente SARIF.

El exporter solo mapeará los compatibles.

---

# Parte LIV — JUnit

## 137. Use

Resultados de validación podrán representarse como JUnit para CI.

---

## 138. No Semantic Loss

El reporte nativo seguirá siendo la fuente rica.

---

# Parte LV — Diagnostic Catalog

## 139. Catalog

```text
MigrationDiagnosticCatalog
```

---

## 140. Entry

Cada diagnostic ID podrá definir:

```text
title
category
meaning
common causes
risk
recommended investigation
related documentation
```

---

## 141. Runtime Evidence

El catálogo describe el tipo de problema.

No reemplaza la evidencia concreta.

---

# Parte LVI — Diagnostic Explain

## 142. CLI

```bash
php volt database:migrate:report \
    --explain=VSDB-MIG-SHADOW-0041
```

---

## 143. Output

```text
Diagnostic
Meaning
Observed evidence
Affected unit
Possible causes
Related diagnostics
Blocked transitions
Recommended investigation
```

---

# Parte LVII — Artifact Linking

## 144. Linker

```text
MigrationArtifactLinker
```

---

## 145. Links

Podrá relacionar:

```text
report
analysis snapshot
MIM snapshot
plan
transformation journal
schema snapshot
validation manifest
checkpoint
```

---

## 146. No Broken Evidence

Un reporte no deberá referenciar un artifact inexistente como si estuviera disponible.

---

# Parte LVIII — Report Store

## 147. Store

```text
MigrationReportStore
```

---

## 148. Workspace

Por defecto:

```text
.voltstack/database-migration/reports/
```

---

## 149. Naming

Conceptualmente:

```text
<timestamp>-<session>-<profile>.<format>
```

---

## 150. Retention

Configurable.

---

# Parte LIX — Report Integrity

## 151. Fingerprint

Los reportes críticos podrán incluir:

```text
content fingerprint
```

---

## 152. Purpose

Detectar modificación accidental.

---

## 153. Audit Use

Cuando se requiera mayor auditabilidad podrán almacenarse fingerprints externamente.

---

# Parte LX — Audit Trail

## 154. State Changes

El reporte deberá poder reconstruir:

```text
who/what initiated change where available
previous state
new state
plan
gate
evidence
timestamp
```

---

## 155. Technical Actor

Puede ser:

```text
developer
CI service
deployment system
operator
```

sin exigir un sistema de identidad propio del módulo Database.

---

# Parte LXI — Migration Timeline

## 156. Timeline

Ejemplo:

```text
09:12 Analysis snapshot created
09:20 Transformation applied
09:31 Validation PASS
10:05 SHADOW_READ enabled
14:40 Shadow coverage threshold reached
15:02 Cutover gate satisfied
15:10 VOLTSTACK_READ enabled
```

---

## 157. Purpose

Facilitar diagnóstico y recovery.

---

# Parte LXII — Correlation IDs

## 158. IDs

Podrán correlacionarse:

```text
migration session
validation run
shadow comparison
runtime transition
checkpoint
report
```

---

## 159. No High-cardinality Metrics

Correlation IDs no deberán utilizarse como labels métricos.

---

# Parte LXIII — Observability Integration

## 160. Telemetry

El reporting consumirá, cuando exista:

```text
metrics
logs
traces
migration events
```

---

## 161. No Telemetry Dependency

El sistema deberá funcionar también sin backend externo de observabilidad.

---

## 162. VoltStack Telemetry

Cuando el módulo oficial esté instalado, la integración deberá ser profunda.

---

# Parte LXIV — Reporting Metrics

## 163. Metrics

```text
database.migration.reporting.reports_generated
database.migration.reporting.diagnostics_open
database.migration.reporting.blockers
database.migration.reporting.unknowns
database.migration.reporting.legacy_dependencies
database.migration.reporting.regressions
database.migration.reporting.duration
```

---

# Parte LXV — Diagnostic Events

## 164. Events

```text
MigrationDiagnosticOpened
MigrationDiagnosticUpdated
MigrationDiagnosticResolved
MigrationDiagnosticReopened
MigrationReportGenerated
MigrationRegressionDetected
MigrationBlockerDetected
```

---

# Parte LXVI — Security

## 165. Redaction

Antes de renderizar:

```text
MigrationReportingPrivacyFilter
```

---

## 166. Protected Data

```text
passwords
tokens
credentials
PII
query parameter values
sensitive payloads
```

---

## 167. Report Profile Does Not Disable Security

Ni:

```text
technical
debug
audit
```

deberán desactivar redacción automáticamente.

---

# Parte LXVII — Data Minimization

## 168. Principle

Guardar:

```text
enough evidence to diagnose
```

pero no:

```text
unnecessary production data
```

---

## 169. Preferred Evidence

```text
fingerprints
counts
types
field names
structural summaries
safe excerpts
```

---

# Parte LXVIII — Access Control

## 170. Responsibility Boundary

El módulo podrá clasificar reportes por sensibilidad.

La autorización de acceso se integrará con:

```text
VoltStack Authorization
deployment environment
external storage permissions
```

---

## 171. Classification

Ejemplo:

```text
PUBLIC
INTERNAL
SENSITIVE
RESTRICTED
```

según policy del proyecto.

---

# Parte LXIX — Multitenancy

## 172. Optional Package

Multitenancy continúa siendo un paquete oficial opcional.

---

## 173. Reporting

Cuando esté instalado, los reportes podrán incluir:

```text
tenant coverage
tenant-specific blockers
cross-tenant security findings
schema-per-tenant drift
```

---

## 174. Privacy

No se expondrán identificadores sensibles innecesariamente.

---

# Parte LXX — SaaS

## 175. Separation

El paquete SaaS seguirá separado de Multitenancy y Database core.

---

## 176. Integration

Si está instalado, podrá aportar contexto adicional de migración sin convertirse en dependencia del sistema de reporting.

---

# Parte LXXI — FrankenPHP

## 177. Persistent Worker Diagnostics

El reporte deberá identificar problemas como:

```text
tenant leakage
open transaction leakage
identity map retention
session state leakage
listener accumulation
memory growth
```

---

## 178. Dedicated Section

```text
Persistent Runtime Diagnostics
```

---

## 179. Request Correlation

Los datos deberán agregarse sin conservar payloads completos de requests.

---

# Parte LXXII — RoadRunner/OpenSwoole

## 180. Profiles

La arquitectura soportará perfiles equivalentes mediante paquetes oficiales opcionales.

---

# Parte LXXIII — Performance

## 181. Large Repositories

No todos los reportes deberán cargar toda la evidencia en memoria.

---

## 182. Streaming

Podrán utilizarse:

```text
streaming readers
iterators
paged evidence
incremental aggregation
```

---

## 183. Indexed Store

La evidencia podrá indexarse por:

```text
unit
diagnostic ID
category
severity
status
artifact
```

---

# Parte LXXIV — Incremental Reporting

## 184. Incremental

Si solo cambia una unidad:

```text
recalculate affected report sections
```

cuando sea seguro.

---

## 185. Global Sections

Los totales globales deberán actualizarse correctamente.

---

# Parte LXXV — Determinism

## 186. Same Inputs

Con evidencia idéntica:

```text
same canonical report model
```

---

## 187. Time Fields

Campos como:

```text
generatedAt
```

no deberán alterar fingerprints semánticos cuando se calcule equivalencia lógica.

---

# Parte LXXVI — Sorting

## 188. Canonical Ordering

```text
severity
unit
diagnostic ID
subject
```

u orden estable equivalente.

---

## 189. Purpose

Evitar diffs ruidosos.

---

# Parte LXXVII — Report Diff

## 190. Semantic Diff

Comparará:

```text
migration state
findings
coverage
risks
blockers
dependencies
```

en lugar de comparar únicamente texto renderizado.

---

# Parte LXXVIII — CI Integration

## 191. CI Report

```bash
php volt database:migrate:report \
    --profile=ci \
    --format=json
```

---

## 192. Pull Request

Podrá destacar:

```text
new migration blocker
new legacy dependency
new schema drift
new validation failure
```

---

## 193. No Approval Logic in Renderer

La policy de CI reside en validation/gates, no en el renderer.

---

# Parte LXXIX — Deployment Integration

## 194. Pre-deployment

Generar:

```text
cutover readiness report
```

---

## 195. Post-deployment

Generar:

```text
post-cutover diagnostic report
```

---

## 196. Compare

Ambos podrán compararse para detectar regresiones.

---

# Parte LXXX — Incident Support

## 197. During Incident

El sistema podrá generar:

```text
migration incident context
```

---

## 198. Content

```text
recent state transition
recent schema change
recent transformation
validation status
shadow mismatches
runtime diagnostics
checkpoint
```

---

## 199. No Incident Manager

Reporting aporta contexto; no reemplaza sistemas generales de incident management.

---

# Parte LXXXI — Recovery Integration

## 200. Recovery Context

340 podrá consumir:

```text
last known good state
last successful validation
last checkpoint
recent changes
irreversible barriers
runtime compatibility
```

---

## 201. Diagnostic Snapshot

Antes de rollback podrá generarse:

```text
pre-recovery diagnostic snapshot
```

---

## 202. After Recovery

Generar:

```text
post-recovery validation/report
```

---

# Parte LXXXII — Report Retention

## 203. Policy

Definirá:

```text
retention duration
maximum storage
sensitive report handling
archival
```

---

## 204. CI Reports

Pueden tener retención distinta a:

```text
audit/cutover reports
```

---

# Parte LXXXIII — Versioning

## 205. Report Schema

```text
MigrationReportSchemaVersion
```

---

## 206. Diagnostic Schema

También:

```text
MigrationDiagnosticSchemaVersion
```

---

## 207. Compatibility

Readers deberán rechazar o degradar explícitamente versiones desconocidas, nunca interpretar silenciosamente campos incompatibles.

---

# Parte LXXXIV — Extensions

## 208. Extension Interface

Conceptualmente:

```php
interface MigrationReportContributorInterface
{
    public function contribute(
        MigrationReportContext $context,
        MigrationReportModelBuilder $builder
    ): void;
}
```

---

## 209. Use

Paquetes podrán agregar:

```text
domain-specific diagnostics
custom report sections
platform-specific metrics
```

---

## 210. Namespace

Las extensiones deberán usar namespaces propios.

---

## 211. Core Integrity

No podrán eliminar silenciosamente findings core.

---

# Parte LXXXV — Custom Renderers

## 212. Interface

```php
interface MigrationReportRendererInterface
{
    public function supports(string $format): bool;

    public function render(
        MigrationReportModel $report
    ): MigrationRenderedReport;
}
```

---

# Parte LXXXVI — Testing

## 213. Unit Tests

Para:

```text
aggregation
normalization
correlation
sorting
redaction
trend calculation
snapshot comparison
```

---

## 214. Golden Reports

Podrán existir fixtures:

```text
golden report model
```

para detectar cambios inesperados.

---

## 215. Renderer Tests

Cada renderer deberá probar:

```text
same semantics
correct escaping
redaction
stable structure
```

---

## 216. Security Tests

Confirmar que reportes no contienen:

```text
secrets
raw credentials
forbidden PII
```

---

## 217. Large Evidence Tests

Probar:

```text
thousands of findings
millions of shadow samples aggregated
large dependency graphs
```

---

# Parte LXXXVII — Diagnostic Correlation Tests

## 218. Cases

```text
same root cause / multiple symptoms
same symptom / different causes
stale evidence
conflicting evidence
low confidence
```

---

# Parte LXXXVIII — Example: Missing Soft Delete

## 219. Inputs

```text
Behavior:
FAIL soft-delete contract

Shadow:
3 extra rows

Schema:
deleted_at exists
```

---

## 220. Correlated Diagnostic

```text
VSDB-MIG-REPORT-0101

Possible root cause:
Target soft-delete filtering missing.

Severity:
ERROR

Confidence:
HIGH

Blocks:
Catalog → VOLTSTACK_READ

Evidence:
behavior + shadow + schema
```

---

# Parte LXXXIX — Example: Persistent Worker Leak

## 221. Evidence

```text
FrankenPHP runtime test:
tenant B received tenant A filter state
```

---

## 222. Report

```text
CRITICAL RUNTIME DIAGNOSTIC

Category:
SECURITY / RUNTIME

Affected:
all persistent-worker traffic

Blocks:
VOLTSTACK_PRIMARY

Required:
fix request-scoped tenant/filter reset
```

---

# Parte XC — Example: Performance Regression

## 223. Evidence

```text
Legacy:
4 queries

VoltStack:
204 queries
```

---

## 224. Diagnostic

```text
N+1 regression

Behavior:
equivalent

Performance:
failed guardrail

Likely area:
relationship loading strategy
```

---

# Parte XCI — Example: Stale Validation

## 225. Evidence

```text
validation PASS
```

pero posteriormente:

```text
repository code changed
```

---

## 226. Report

```text
Validation:
STALE

Previous result:
PASS

Reason:
target dependency changed

Cutover readiness:
BLOCKED
```

---

# Parte XCII — Example: Legacy Dependency

## 227. Finding

```text
Queue job payload still serializes Eloquent model.
```

---

## 228. Report

```text
Legacy removal blocker

Unit:
Orders

Dependency:
queued legacy model serialization

Blocks:
LEGACY_DISABLED → LEGACY_REMOVED
```

---

# Parte XCIII — Example: Unknown Stored Procedure

## 229. Finding

```text
Procedure:
close_month()

Side effects:
UNKNOWN
```

---

## 230. Report

```text
Unknown semantic dependency

Risk:
TRANSACTION / DATA_INTEGRITY

Confidence:
UNKNOWN

Blocks:
Finance write migration

Next investigation:
manual procedure review or controlled observation
```

---

# Parte XCIV — Example: Approved Difference

## 231. Difference

```text
legacy API returns decimal as string
target API normalizes to Money DTO
```

---

## 232. Approved

Si existe contrato explícito:

```text
APPROVED_DIFFERENCE
```

---

## 233. Report

Debe mostrar:

```text
reason
scope
contract
risk
review condition
```

---

# Parte XCV — CLI

## 234. General

```bash
php volt database:migrate:report
```

---

## 235. Technical

```bash
php volt database:migrate:report \
    --profile=technical
```

---

## 236. Unit

```bash
php volt database:migrate:report \
    --unit=Billing
```

---

## 237. Risks

```bash
php volt database:migrate:report \
    --risks
```

---

## 238. Blockers

```bash
php volt database:migrate:report \
    --blockers
```

---

## 239. Unknowns

```bash
php volt database:migrate:report \
    --unknowns
```

---

## 240. Legacy Dependencies

```bash
php volt database:migrate:report \
    --legacy-dependencies
```

---

## 241. Cutover

```bash
php volt database:migrate:report \
    --profile=cutover \
    --unit=Billing
```

---

## 242. Recovery

```bash
php volt database:migrate:report \
    --profile=recovery
```

---

# Parte XCVI — Integration with 338

## 243. CLI/DX

338 proporciona:

```text
commands
rendering entry points
workspace
artifacts
```

339 proporciona:

```text
report semantics
diagnostic model
correlation
trend
report building
```

---

# Parte XCVII — Integration with 337

## 244. Validation

337 decide:

```text
PASS / FAIL / BLOCKED / INCONCLUSIVE / STALE
```

339 explica y presenta esos resultados.

---

# Parte XCVIII — Integration with 336

## 245. Shadow

336 genera comparación.

339 agrega:

```text
coverage
mismatch groups
trend
risk
```

---

# Parte XCIX — Integration with 335

## 246. Runtime

335 define estados y transiciones.

339 muestra:

```text
current state
history
blocked transitions
cutover readiness
```

---

# Parte C — Integration with 334

## 247. Behavior

334 define contratos.

339 muestra:

```text
verified
failed
approved
unverified
```

---

# Parte CI — Integration with 333

## 248. Schema

333 produce compatibilidad y conflictos.

339 los convierte en:

```text
schema report
risk report
cutover evidence
```

---

# Parte CII — Integration with 332

## 249. Transformation

332 genera journal.

339 muestra:

```text
files changed
rules applied
manual changes
unresolved transformations
```

---

# Parte CIII — Integration with 331

## 250. Rules

Los reportes podrán explicar:

```text
which rule produced which transformation/finding
```

---

# Parte CIV — Integration with 330

## 251. MIM

Los diagnósticos podrán enlazar:

```text
canonical entity
query
transaction
schema object
unknown
```

---

# Parte CV — Integration with 329

## 252. Analysis

Findings iniciales deberán conservar trazabilidad hasta reportes finales.

---

# Parte CVI — Integration with 340

## 253. Recovery

340 consumirá especialmente:

```text
migration timeline
last known good state
checkpoints
irreversible barriers
recent diagnostics
validation status
runtime state
schema state
```

---

# Parte CVII — V1 Scope

## 254. Included

Database V1 incluye:

```text
canonical report model
diagnostic model
evidence aggregation
correlation
root-cause hints
blocker analysis
progress
coverage
trends
snapshot comparison
regression detection
risk reporting
cutover reporting
recovery readiness reporting
legacy dependency reporting
console/json/markdown reporting
extension interfaces
security/redaction
persistent-worker diagnostics
```

---

## 255. Excluded

No pertenece a Database V1:

```text
AI-autonomous root cause authority
enterprise BI platform
remote multi-organization migration dashboard
cross-cloud fleet analytics
predictive autonomous cutover
automatic remediation engine
```

Estos podrán evolucionar en Database V2 o productos superiores.

---

# Parte CVIII — Architectural Decisions

## 256. Decisión 1

Reporting y diagnostics compartirán evidencia, pero tendrán responsabilidades distintas.

## 257. Decisión 2

Los reportes se construirán sobre un modelo canónico independiente del renderer.

## 258. Decisión 3

Todo reporte crítico tendrá provenance suficiente para reproducir su contexto.

## 259. Decisión 4

Severity, confidence y risk permanecerán separados.

## 260. Decisión 5

Los unknowns serán first-class citizens.

## 261. Decisión 6

Los findings resueltos no desaparecerán del historial.

## 262. Decisión 7

El sistema correlacionará síntomas para evitar ruido diagnóstico.

## 263. Decisión 8

Root-cause hints no se presentarán como certeza sin evidencia.

## 264. Decisión 9

El progreso será multidimensional.

## 265. Decisión 10

No existirá un migration health score único y opaco.

## 266. Decisión 11

Los approved differences permanecerán visibles.

## 267. Decisión 12

Los suppressed diagnostics permanecerán auditables.

## 268. Decisión 13

Los reportes podrán compararse semánticamente entre snapshots.

## 269. Decisión 14

Security y data-integrity blockers no podrán quedar ocultos en resúmenes.

## 270. Decisión 15

Los renderers no recalcularán lógica de migración.

## 271. Decisión 16

La redacción de secretos se aplicará antes del render.

## 272. Decisión 17

FrankenPHP tendrá diagnósticos específicos de persistent-worker.

## 273. Decisión 18

Reporting deberá proporcionar a 340 suficiente contexto para recovery y rollback.

---

# Parte CIX — Complete Reporting Pipeline

## 274. Pipeline

```text
Analysis
   │
MIM
   │
Rules
   │
Transformation
   │
Schema
   │
Behavior
   │
Dual Runtime
   │
Shadow
   │
Validation
   │
   ▼
Migration Evidence Aggregator
   │
   ▼
Normalize Diagnostics
   │
   ▼
Correlate
   │
   ├── Root Causes
   ├── Blockers
   ├── Risks
   ├── Unknowns
   ├── Coverage
   └── Trends
   │
   ▼
Canonical Report Model
   │
   ├── Console
   ├── JSON
   ├── Markdown
   ├── HTML
   ├── SARIF
   └── JUnit
   │
   ▼
Developer / CI / Operator / Recovery
```

---

# Parte CX — Completion Criteria

## 275. Reporting System Complete

V1 estará completo cuando pueda:

```text
aggregate all migration evidence
trace findings to sources
show migration state
show migration history
show blockers
show unknowns
show risks
show coverage
show trends
detect regressions
group repeated diagnostics
correlate probable root causes
report legacy dependencies
report cutover readiness
report recovery readiness
export structured reports
protect sensitive information
support large repositories
support FrankenPHP diagnostics
provide recovery context to 340
```

---

# Parte CXI — Resultado esperado

## 276. Antes

```text
logs
terminal output
test reports
schema diffs
shadow metrics
manual notes
```

existen como fuentes separadas.

---

## 277. Después

```text
Migration Evidence
        │
        ▼
Canonical Diagnostic Model
        │
        ▼
Correlation
        │
        ▼
Migration Report Model
        │
   ┌────┼────┬─────┐
   ▼    ▼    ▼     ▼
 Dev    CI   Ops   Recovery
```

---

# Parte CXII — Principio final

## 278. Regla

```text
Do not report that a migration is healthy
because no error is visible.

Report what is known,
what is verified,
what is unknown,
what changed,
what is blocked,
and which evidence proves each conclusion.
```

---

# Parte CXIII — Conclusión

## 279. Arquitectura final

`DATABASE_MIGRATION_REPORTING_AND_DIAGNOSTICS` convierte toda la arquitectura de migración en un sistema observable y explicable.

La cadena de V1 queda:

```text
325 External ORM Migration Architecture
        ↓
326 Eloquent Adapter
327 Doctrine Adapter
328 Legacy Migration
        ↓
329 Analysis Engine
        ↓
330 Intermediate Model
        ↓
331 Rule Engine
        ↓
332 Code Transformer
        ↓
333 Schema Compatibility
        ↓
334 Behavior Verification
        ↓
335 Dual ORM Runtime
        ↓
336 Shadow Comparison
        ↓
337 Testing & Validation
        ↓
338 CLI & Developer Experience
        ↓
339 Reporting & Diagnostics
        ↓
340 Rollback & Recovery
```

339 establece que ningún estado importante de migración deberá depender exclusivamente de interpretación manual de logs.

En su lugar:

```text
Evidence
   ↓
Normalized Diagnostics
   ↓
Correlation
   ↓
Structured Reports
   ↓
Explainable Migration State
```

La regla arquitectónica definitiva será:

```text
Every migration conclusion must be traceable.

Every blocker must be explainable.

Every unknown must remain visible.

Every critical state change must leave evidence.
```

---

**Documento:** `339_DATABASE_MIGRATION_REPORTING_AND_DIAGNOSTICS.md`  
**Proyecto:** VoltStack Framework  
**Módulo:** `VoltStack/Quantum/Database`  
**Versión objetivo:** Database V1  
**MIM objetivo:** `1.x`  
**Estado:** Architectural Specification
