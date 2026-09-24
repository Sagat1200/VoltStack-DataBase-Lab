# 336_DATABASE_SHADOW_QUERY_AND_RESULT_COMPARISON_SYSTEM.md

## 1. Propósito

Este documento define la arquitectura oficial del **Database Shadow Query and Result Comparison System** de `VoltStack/Quantum/Database`.

El componente se denomina conceptualmente:

```text
DatabaseShadowQueryAndResultComparisonSystem
```

y permite ejecutar de forma controlada una operación de lectura mediante:

```text
Primary Persistence Runtime
        +
Shadow Persistence Runtime
```

para comparar sus resultados sin permitir que el runtime shadow determine la respuesta entregada al consumidor.

Su objetivo principal es aportar evidencia real para migraciones progresivas:

```text
Legacy Read
    vs
VoltStack Read
```

antes de mover tráfico productivo al nuevo sistema.

---

## 2. Dependencias documentales

Este documento continúa:

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
```

y alimentará:

```text
337_DATABASE_MIGRATION_TESTING_AND_VALIDATION_SYSTEM.md
338_DATABASE_MIGRATION_CLI_AND_DEVELOPER_EXPERIENCE.md
339_DATABASE_MIGRATION_REPORTING_AND_DIAGNOSTICS.md
340_DATABASE_MIGRATION_ROLLBACK_AND_RECOVERY_SYSTEM.md
```

---

## 3. Principio fundamental

```text
The shadow path observes.

The primary path decides.
```

---

## 4. Problema

Las pruebas offline pueden demostrar gran parte de la compatibilidad, pero no siempre representan:

```text
real production data
real query parameters
real tenant distribution
real null patterns
real legacy data anomalies
real query frequency
real edge cases
```

---

## 5. Estrategia

Para operaciones de lectura elegibles:

```text
Application Request
        │
        ▼
Primary Query
        │
        ├──────────────► Shadow Query
        │                     │
        ▼                     ▼
Primary Result          Shadow Result
        │                     │
        └──────────┬──────────┘
                   ▼
               Normalize
                   │
                   ▼
                Compare
                   │
                   ▼
             Record Evidence
```

La aplicación continúa usando:

```text
Primary Result
```

---

# Parte I — Objetivos

## 6. Objetivos principales

El sistema deberá:

```text
execute shadow reads safely
normalize source/target representations
compare semantic results
classify differences
sample production traffic
limit operational overhead
protect sensitive data
detect persistent divergence
produce migration evidence
support cutover decisions
```

---

## 7. No objetivos

No deberá:

```text
dual-write production data
automatically repair mismatches
automatically switch write ownership
treat every SQL difference as failure
expose shadow failures to end users by default
```

---

# Parte II — Terminología

## 8. Primary Runtime

Es el runtime autorizado para producir el resultado de aplicación.

---

## 9. Shadow Runtime

Ejecuta una operación secundaria para observación.

---

## 10. Primary Result

Resultado utilizado por la aplicación.

---

## 11. Shadow Result

Resultado descartable utilizado exclusivamente para comparación.

---

## 12. Comparison Evidence

Registro estructurado de la comparación.

---

## 13. Shadow Operation

Una ejecución shadow completa:

```text
MigrationShadowOperation
```

---

# Parte III — Elegibilidad

## 14. Shadow Eligibility

No toda operación puede ejecutarse como shadow.

---

## 15. Safe Default

Por defecto serán elegibles:

```text
read-only deterministic queries
```

---

## 16. Potentially Eligible

Con configuración:

```text
aggregates
reports
paginated reads
relationship loads
stored-procedure reads
```

---

## 17. Not Eligible by Default

```text
INSERT
UPDATE
DELETE
DDL
procedure with side effects
SELECT that acquires persistent/advisory locks
read that mutates session state
```

---

## 18. Eligibility Analyzer

Componente:

```text
MigrationShadowEligibilityAnalyzer
```

---

## 19. Evidence

Utilizará:

```text
MIM query descriptors
query classification
transaction metadata
procedure metadata
runtime observations
developer overrides
```

---

## 20. Unknown Query

Una query:

```text
UNKNOWN
```

no será shadow-safe automáticamente.

---

# Parte IV — Read-only Enforcement

## 21. Shadow Connection

Siempre que sea posible se utilizará:

```text
read-only connection
```

---

## 22. Database Enforcement

Si la plataforma lo permite:

```text
read-only transaction/session
read-only database user
replica
```

podrá reforzar la seguridad.

---

## 23. Defense in Depth

La clasificación estática no sustituye controles del DB.

---

## 24. Violation

Si shadow intenta escribir:

```text
SHADOW_WRITE_ATTEMPT
```

y la operación deberá abortarse.

---

# Parte V — Arquitectura

## 25. Componentes

```text
DatabaseShadowQueryAndResultComparisonSystem
│
├── MigrationShadowCoordinator
├── MigrationShadowPolicy
├── MigrationShadowEligibilityAnalyzer
├── MigrationShadowSampler
├── MigrationShadowExecutionContext
├── MigrationPrimaryQueryExecutor
├── MigrationShadowQueryExecutor
├── MigrationShadowTimeoutManager
├── MigrationShadowResultNormalizer
├── MigrationShadowResultComparator
├── MigrationShadowDifferenceClassifier
├── MigrationShadowEvidenceStore
├── MigrationShadowCircuitBreaker
├── MigrationShadowCapacityGuard
├── MigrationShadowPrivacyFilter
└── MigrationShadowReporter
```

---

# Parte VI — Integración con Dual Runtime

## 26. Runtime States

El sistema se activa principalmente durante:

```text
SHADOW_READ
```

de 335.

---

## 27. Primary Legacy

Configuración inicial:

```text
primary = LEGACY
shadow  = VOLTSTACK
```

---

## 28. Reverse Shadow

También podrá utilizarse:

```text
primary = VOLTSTACK
shadow  = LEGACY
```

durante una ventana posterior al cutover.

---

## 29. Purpose

Esto permite detectar regresiones después de mover reads a VoltStack.

---

# Parte VII — Query Identity

## 30. Query Identity

Cada operación deberá asociarse a:

```text
MigrationQueryIdentity
```

---

## 31. Identity Fields

```text
migration unit
logical query ID
query fingerprint
repository/query-service operation
source adapter
target mapping
```

---

## 32. Logical Identity

La comparación debe correlacionar:

```text
same logical operation
```

aunque el SQL sea diferente.

---

## 33. SQL Fingerprint

Los fingerprints ayudan a diagnóstico, pero no son la identidad semántica definitiva.

---

# Parte VIII — Execution Context

## 34. Context

```text
MigrationShadowExecutionContext
```

contendrá:

```text
operation ID
request correlation ID
migration unit
tenant context
input fingerprint
primary runtime
shadow runtime
sampling decision
deadline
comparison policy
privacy policy
```

---

## 35. No Sensitive Inputs

Los valores completos de parámetros no deberán persistirse automáticamente.

---

## 36. Input Fingerprint

Podrá utilizarse:

```text
salted hash
structural descriptor
type profile
```

según policy.

---

# Parte IX — Sampling

## 37. Sampling

Ejecutar shadow para el 100% de tráfico puede ser innecesario o costoso.

---

## 38. Sampler

```text
MigrationShadowSampler
```

---

## 39. Strategies

```text
FIXED_RATE
DETERMINISTIC_KEY
TENANT
QUERY_TYPE
RISK_BASED
TIME_WINDOW
MANUAL
```

---

## 40. Fixed Rate

Ejemplo:

```text
5%
```

de operaciones elegibles.

---

## 41. Deterministic Sampling

Un hash estable puede mantener un cohort consistente.

---

## 42. Risk-based

Queries críticas podrán recibir mayor cobertura.

---

## 43. No Hidden Bias

El reporte deberá indicar la estrategia de sampling para no presentar la muestra como cobertura total.

---

# Parte X — Capacity Guard

## 44. Capacity

Shadow incrementa carga sobre:

```text
database
connections
CPU
memory
network
workers
```

---

## 45. Capacity Guard

```text
MigrationShadowCapacityGuard
```

podrá limitar ejecución según:

```text
connection pool saturation
DB latency
worker load
queue backlog
memory pressure
```

---

## 46. Protection Rule

```text
Protect primary traffic before shadow coverage.
```

---

## 47. Load Shedding

El sistema podrá reducir o suspender shadow temporalmente.

---

## 48. Primary Independence

Suspender shadow no deberá cambiar el primary runtime.

---

# Parte XI — Synchronous vs Deferred

## 49. Synchronous

```text
primary + shadow
```

pueden ejecutarse dentro de la misma request.

---

## 50. Advantage

Contexto simple y comparación inmediata.

---

## 51. Disadvantage

Puede incrementar:

```text
latency
memory
connection occupancy
```

---

## 52. Deferred Comparison

Cuando sea seguro:

```text
primary executes
shadow execution/comparison deferred
```

---

## 53. Requirement

El sistema deberá conservar suficiente contexto sin almacenar datos sensibles innecesarios.

---

## 54. V1

V1 podrá soportar ejecución síncrona como baseline y permitir adapters para ejecución diferida.

---

# Parte XII — Timeout

## 55. Shadow Deadline

El shadow tendrá:

```text
separate timeout
```

del primary.

---

## 56. Rule

```text
Shadow must not extend user-visible latency beyond policy.
```

---

## 57. Timeout Result

```text
SHADOW_TIMEOUT
```

no equivale a:

```text
RESULT_MISMATCH
```

---

## 58. Classification

Será una diferencia operacional/inconclusa.

---

# Parte XIII — Failure Isolation

## 59. Shadow Failure

Errores shadow no deberán fallar la request primary por defecto.

---

## 60. Captured Failure

Se registrará:

```text
exception category
runtime
query identity
schema fingerprint
migration version
```

---

## 61. No Secret Stack Dumps

Los logs deberán aplicar redacción.

---

# Parte XIV — Result Model

## 62. Observed Result

Cada runtime producirá:

```text
MigrationObservedQueryResult
```

---

## 63. Fields

```text
status
result shape
row count
normalized payload
type metadata
ordering metadata
execution metadata
exception descriptor
```

---

## 64. Result Shapes

```text
SCALAR
ROW
ROWSET
ENTITY
ENTITY_COLLECTION
DTO
AGGREGATE
STREAM
PAGINATED
UNKNOWN
```

---

# Parte XV — Normalization

## 65. Normalizer

```text
MigrationShadowResultNormalizer
```

convierte ambos resultados a una representación comparable.

---

## 66. Neutral Representation

No deberá contener:

```text
Eloquent Model
Doctrine Proxy
VoltStack Entity internals
```

---

## 67. Canonical Values

Podrán normalizarse:

```text
booleans
dates
UUIDs
decimals
enums
JSON
collections
```

---

## 68. Explicit Rules

Cada normalización deberá ser:

```text
declared
versioned
auditable
```

---

## 69. No Over-normalization

La normalización nunca deberá ocultar diferencias funcionales.

---

# Parte XVI — Scalar Comparison

## 70. Scalars

Tipos básicos:

```text
null
boolean
integer
decimal
string
date/time
binary fingerprint
```

---

## 71. Decimal

Comparación mediante representación decimal exacta cuando sea requerida.

---

## 72. Float

Podrá utilizar tolerancia explícita.

---

# Parte XVII — Date/Time

## 73. Date Normalization

Podrá convertir a:

```text
UTC instant + precision
```

si el contrato lo permite.

---

## 74. Timezone

No deberá descartarse una diferencia de timezone cuando sea semánticamente relevante.

---

# Parte XVIII — Entity/Row Comparison

## 75. Row

Se normalizará como:

```text
field → canonical value
```

---

## 76. Field Mapping

Los nombres podrán diferir entre source y target.

El MIM proveerá mapping.

---

## 77. Missing Field

Se distinguirá:

```text
FIELD_MISSING
FIELD_NULL
FIELD_NOT_SELECTED
```

---

# Parte XIX — Collections

## 78. Collection Comparison

Debe decidirse si:

```text
order matters
```

---

## 79. Ordered

Para queries con orden contractual:

```text
position-by-position comparison
```

---

## 80. Unordered

Para conjuntos sin orden contractual:

```text
canonical multiset comparison
```

---

## 81. Duplicates

No deberán perderse al convertir una colección en set.

---

# Parte XX — Stable Identity

## 82. Identity Key

Para colecciones grandes podrá compararse mediante:

```text
stable row identity
```

---

## 83. Examples

```text
primary key
composite key
declared comparison key
```

---

## 84. No Key

Si no existe identidad estable:

```text
content fingerprint
```

podrá ayudar, pero deberá registrarse menor confianza.

---

# Parte XXI — Ordering

## 85. Order Contract

El comparator consultará:

```text
MigrationQueryBehaviorContract
```

de 334.

---

## 86. Unspecified Order

No deberá reportar mismatch únicamente por diferente orden si el contrato no garantiza orden.

---

## 87. Specified Order

Si el orden es contractual:

```text
ORDER_MISMATCH
```

---

# Parte XXII — Pagination

## 88. Paginated Result

Se comparará:

```text
items
page size
cursor/page
has next
total when applicable
ordering
```

---

## 89. Cursor Representation

El cursor puede tener encoding distinto.

Se comparará su significado cuando exista normalizador.

---

## 90. Boundary Tests

Especial atención a:

```text
first item
last item
ties
page transition
```

---

# Parte XXIII — Aggregates

## 91. Aggregate Queries

Ejemplos:

```text
COUNT
SUM
AVG
MIN
MAX
GROUP BY
```

---

## 92. Decimal Aggregates

Deberán conservar precisión.

---

## 93. Empty Aggregate

Se verificará diferencia entre:

```text
NULL
0
empty result
```

---

# Parte XXIV — JSON

## 94. JSON Objects

El orden de keys normalmente podrá ignorarse.

---

## 95. JSON Arrays

El orden deberá conservarse salvo contrato contrario.

---

## 96. Numeric Types

```text
1
1.0
"1"
```

no serán equivalentes automáticamente.

---

# Parte XXV — Exceptions

## 97. Exception Comparison

Si primary y shadow fallan:

```text
compare semantic error category
```

---

## 98. Example

```text
ModelNotFoundException
```

vs:

```text
EntityNotFoundException
```

pueden mapear a:

```text
NOT_FOUND
```

si el contrato público así lo define.

---

## 99. One Side Fails

```text
primary success
shadow failure
```

produce:

```text
SHADOW_EXECUTION_DIFFERENCE
```

---

# Parte XXVI — Query Count

## 100. Query Observation

El sistema podrá registrar:

```text
SQL count
round trips
duration
rows examined where available
```

---

## 101. N+1

Si:

```text
legacy = 2 queries
VoltStack = 102 queries
```

puede generarse:

```text
QUERY_COUNT_REGRESSION
```

aunque el resultado sea igual.

---

## 102. Separate Dimension

Esto no deberá mezclarse con:

```text
RESULT_EQUIVALENCE
```

---

# Parte XXVII — Performance Comparison

## 103. Metrics

Podrán compararse:

```text
duration
query count
memory
rows returned
connection time
```

---

## 104. Production Noise

La latencia en producción tiene variabilidad.

No deberá inferirse regresión por una sola muestra.

---

## 105. Aggregation

Se utilizarán ventanas:

```text
count
median
percentiles
error rate
```

cuando exista suficiente muestra.

---

## 106. Relative Threshold

Policies podrán definir:

```text
warning if target median > source median * threshold
```

pero nunca ocultando entorno y tamaño de muestra.

---

# Parte XXVIII — Difference Model

## 107. Difference

```text
MigrationShadowDifference
```

---

## 108. Categories

```text
VALUE
TYPE
NULL
FIELD
ROW_COUNT
CARDINALITY
ORDER
DUPLICATE
PAGINATION
AGGREGATE
EXCEPTION
TIMEOUT
EXECUTION
QUERY_COUNT
PERFORMANCE
SECURITY
TENANCY
UNKNOWN
```

---

## 109. Severity

```text
INFO
LOW
MEDIUM
HIGH
CRITICAL
```

---

## 110. Confidence

```text
HIGH
MEDIUM
LOW
UNKNOWN
```

se mantendrá separado de severity.

---

# Parte XXIX — Difference Fingerprint

## 111. Fingerprint

Diferencias similares podrán agruparse mediante:

```text
MigrationShadowDifferenceFingerprint
```

---

## 112. Purpose

Evitar reportar:

```text
10,000 identical mismatches
```

como 10,000 problemas independientes.

---

## 113. Grouping

Podrá agrupar por:

```text
query identity
difference category
field
runtime versions
schema fingerprint
```

---

# Parte XXX — Evidence

## 114. Evidence Record

```text
MigrationShadowComparisonEvidence
```

---

## 115. Contents

```text
comparison ID
query identity
migration unit
primary runtime
shadow runtime
comparison policy
sample strategy
result status
difference fingerprints
performance observations
schema fingerprint
code/version fingerprint
timestamp
```

---

## 116. Payload Storage

Por defecto no almacenará resultados completos de producción.

---

## 117. Safe Evidence

Preferirá:

```text
counts
hashes
field names
type descriptors
difference summaries
```

---

# Parte XXXI — Privacy

## 118. Privacy Filter

```text
MigrationShadowPrivacyFilter
```

se ejecutará antes de persistir evidencia.

---

## 119. Sensitive Fields

Ejemplos:

```text
password
token
secret
card data
personal identifiers
```

---

## 120. Policy

Podrán marcarse:

```text
DROP
HASH
REDACT
STRUCTURE_ONLY
```

---

## 121. Compare Without Store

El sistema podrá comparar valores en memoria y almacenar únicamente:

```text
matched = true/false
```

---

# Parte XXXII — Security

## 122. Shadow Authorization

El shadow deberá ejecutar bajo el mismo contexto lógico de seguridad requerido para la operación.

---

## 123. No Broader Access

No deberá usar credenciales con acceso a datos que el primary no podría consultar, salvo una infraestructura técnica aislada y autorizada que preserve el mismo scope lógico.

---

## 124. Tenant Context

Debe propagarse correctamente al shadow.

---

## 125. Cross-tenant Mismatch

Si shadow devuelve filas de otro tenant:

```text
CRITICAL_SECURITY_DIFFERENCE
```

---

# Parte XXXIII — Multitenancy

## 126. Tenant Sampling

El sistema podrá hacer sampling:

```text
tenant-by-tenant
```

---

## 127. Tenant Coverage

Los reportes deberán indicar:

```text
number of tenants observed
distribution
excluded tenants
```

sin revelar información innecesaria.

---

## 128. Schema-per-tenant

La comparación deberá utilizar el schema fingerprint correspondiente al tenant.

---

# Parte XXXIV — Read Replicas

## 129. Replica Lag

Si primary y shadow consultan replicas distintas:

```text
temporary mismatch
```

puede deberse a replication lag.

---

## 130. Strategy

Cuando sea posible, comparar contra:

```text
same consistency boundary
same primary database
or lag-aware policy
```

---

## 131. Lag Classification

Diferencias plausibles por lag deberán marcarse:

```text
POTENTIAL_REPLICATION_LAG
```

y no ocultarse.

---

# Parte XXXV — Transactions

## 132. Transaction Visibility

Un shadow en otra conexión puede no ver writes no committed del primary request.

---

## 133. Rule

No deberán compararse ingenuamente queries ejecutadas dentro de una transacción con estado privado.

---

## 134. Eligibility

Estas operaciones podrán ser:

```text
NOT_SHADOW_ELIGIBLE
```

o requerir ejecución fuera de esa ventana.

---

# Parte XXXVI — Locks

## 135. Locking Reads

Queries como:

```text
SELECT ... FOR UPDATE
```

no serán shadow reads normales.

---

## 136. Reason

El shadow podría alterar:

```text
locking
contention
transaction timing
```

---

# Parte XXXVII — Stored Procedures

## 137. Procedures

Solo procedimientos demostrablemente read-only serán elegibles.

---

## 138. Unknown Side Effects

Si no se conocen:

```text
BLOCK SHADOW
```

---

# Parte XXXVIII — Streaming

## 139. Streams

Resultados tipo:

```text
cursor
generator
stream
```

pueden ser muy grandes.

---

## 140. Comparison Strategy

Podrá utilizar:

```text
streaming canonical hash
row count
sampled rows
boundary rows
```

sin materializar todo en memoria.

---

## 141. Exact Streaming Comparison

Cuando sea requerido, podrá compararse incrementalmente.

---

# Parte XXXIX — Large Results

## 142. Memory Safety

No deberá cargar automáticamente millones de filas en memoria para comparar.

---

## 143. Strategy

```text
chunked comparison
streaming hash
key-based merge
sampling
```

---

# Parte XL — Canonical Hash

## 144. Canonical Result Hash

Para resultados grandes podrá calcularse:

```text
H(canonicalized result)
```

---

## 145. Hash Limitation

Un hash distinto indica diferencia.

Un hash igual demuestra igualdad de la representación canonicalizada utilizada, no necesariamente todos los aspectos externos no incluidos en esa representación.

---

# Parte XLI — Comparison Policies

## 146. Policy

```text
MigrationShadowComparisonPolicy
```

---

## 147. Configuration

Podrá definir:

```text
ordered
ignored fields
normalizers
tolerances
identity key
max rows
timeout
sampling
performance thresholds
privacy rules
```

---

## 148. Policy Source

Se derivará preferentemente de:

```text
334 behavior contracts
```

---

# Parte XLII — Approved Differences

## 149. Intentional Difference

Una diferencia conocida podrá declararse:

```text
MigrationApprovedShadowDifference
```

---

## 150. Requirements

```text
difference fingerprint
reason
contract reference
approval
expiry/review condition
```

---

## 151. No Global Ignore

No se permitirá un:

```text
ignore all differences
```

como estado equivalente a verificación.

---

# Parte XLIII — Baseline

## 152. Baseline

Durante migración podrá existir:

```text
MigrationShadowBaseline
```

---

## 153. Purpose

Separar:

```text
known accepted differences
```

de:

```text
new regressions
```

---

## 154. Burn-down

El baseline deberá tender a:

```text
zero unexplained differences
```

---

# Parte XLIV — Comparison Status

## 155. Status

Cada comparación tendrá:

```text
MATCH
MATCH_AFTER_NORMALIZATION
APPROVED_DIFFERENCE
MISMATCH
SHADOW_ERROR
SHADOW_TIMEOUT
INCONCLUSIVE
SKIPPED
```

---

## 156. Aggregate Status

Una query identity podrá tener:

```text
STABLE
UNSTABLE
REGRESSING
INSUFFICIENT_EVIDENCE
BLOCKED
```

---

# Parte XLV — Confidence

## 157. Evidence Confidence

Dependerá de:

```text
sample count
coverage
input diversity
tenant diversity
time window
data volume
difference rate
```

---

## 158. No Arbitrary Score

V1 evitará una cifra única opaca de “99% compatible”.

---

## 159. Dimensional Report

Preferirá:

```text
coverage
match rate
critical mismatch count
sample count
query identities observed
tenant coverage
runtime window
```

---

# Parte XLVI — Cutover Gate

## 160. Gate

El sistema podrá aportar evidencia a:

```text
MigrationRuntimeStateTransitionGuard
```

de 335.

---

## 161. Example Policy

Para pasar:

```text
SHADOW_READ
→ VOLTSTACK_READ
```

puede requerirse:

```text
0 critical mismatches
0 unexplained high mismatches
required query identities observed
minimum sample coverage
behavior contracts passed
schema compatible
```

---

## 162. No Automatic Universal Threshold

Cada proyecto/unidad podrá definir criterios según riesgo.

---

# Parte XLVII — Post-cutover Shadow

## 163. Reverse Verification

Después del cutover:

```text
VoltStack = primary
Legacy = shadow
```

puede mantenerse temporalmente.

---

## 164. Purpose

Detectar diferencias reales antes de eliminar legacy.

---

## 165. Expiration

Debe existir una fecha/fase de salida.

Shadow no deberá convertirse en dependencia permanente.

---

# Parte XLVIII — Circuit Breaker

## 166. Circuit Breaker

```text
MigrationShadowCircuitBreaker
```

---

## 167. Triggers

Podrá abrirse por:

```text
shadow timeout rate
shadow error rate
DB saturation
connection exhaustion
memory pressure
```

---

## 168. Effect

```text
disable/reduce shadow
```

sin modificar primary.

---

## 169. Recovery

Podrá cerrar después de:

```text
cooldown
health check
manual reset
```

según policy.

---

# Parte XLIX — Backpressure

## 170. Deferred Shadow

Si se usa ejecución diferida:

```text
queue backlog
```

deberá tener límite.

---

## 171. Drop Policy

Es preferible descartar evidencia shadow no crítica antes que degradar tráfico principal.

---

## 172. Reporting

Las muestras descartadas deberán contabilizarse.

---

# Parte L — FrankenPHP

## 173. Persistent Worker Safety

El sistema deberá evitar persistencia accidental de:

```text
comparison payload
tenant context
sampling decision
normalizer state
difference buffer
primary/shadow runtime selection
```

entre requests.

---

## 174. Request-scoped Context

```text
MigrationShadowExecutionContext
```

será request-scoped.

---

## 175. Reset

Al finalizar:

```text
clear buffers
release result references
reset context
release temporary resources
```

---

## 176. Memory Retention

Resultados grandes no deberán quedar referenciados por servicios singleton.

---

## 177. Worker Tests

Se probarán múltiples requests consecutivas con:

```text
different tenants
different policies
different sampling decisions
different result sizes
```

---

# Parte LI — RoadRunner/OpenSwoole

## 178. Runtime-neutral Core

La misma arquitectura se reutilizará.

Adapters podrán integrar lifecycle hooks específicos.

---

# Parte LII — Observability

## 179. Metrics

```text
database.migration.shadow.eligible
database.migration.shadow.executed
database.migration.shadow.skipped
database.migration.shadow.match
database.migration.shadow.mismatch
database.migration.shadow.timeout
database.migration.shadow.error
database.migration.shadow.duration
database.migration.shadow.query_count_delta
database.migration.shadow.samples_dropped
```

---

## 180. Metrics Labels

Ejemplos controlados:

```text
migration_unit
query_group
primary_runtime
shadow_runtime
status
```

---

## 181. No High-cardinality Labels

No usar:

```text
user ID
raw SQL
full query fingerprint if unbounded
request ID
```

como labels métricos.

---

## 182. Tracing

Un span podrá incluir:

```text
migration.shadow=true
migration.unit
migration.query_group
migration.comparison_status
```

---

## 183. Logs

Canal:

```text
database.migration.shadow
```

---

# Parte LIII — Diagnostics

## 184. Diagnostic

Ejemplo:

```text
VSDB-MIG-SHADOW-0041

Query:
billing.invoice.list

Primary:
Legacy Eloquent

Shadow:
VoltStack

Status:
MISMATCH

Difference:
ROW_COUNT

Primary rows:
20

Shadow rows:
21

Suspected dimension:
soft-delete filter

Severity:
HIGH

Evidence:
shadow comparison #...
```

---

## 185. Stable IDs

```text
VSDB-MIG-SHADOW-xxxx
```

---

# Parte LIV — Root Cause Hints

## 186. Hints

El sistema podrá correlacionar diferencias con:

```text
missing global scope
type conversion
relationship mapping
ordering
soft delete
tenant filter
schema drift
replica lag
timestamp precision
```

---

## 187. No Unsupported Certainty

Los hints deberán presentarse como:

```text
possible cause
```

hasta tener evidencia suficiente.

---

# Parte LV — CLI

## 188. Status

Conceptualmente:

```bash
php volt database:migrate:shadow --status
```

---

## 189. Enable

```bash
php volt database:migrate:shadow \
    --unit=Billing \
    --enable
```

---

## 190. Sampling

```bash
php volt database:migrate:shadow \
    --unit=Billing \
    --sample=5%
```

---

## 191. Report

```bash
php volt database:migrate:shadow \
    --report
```

---

## 192. Query Scope

```bash
php volt database:migrate:shadow \
    --query=billing.invoice.list
```

---

## 193. Dry Analysis

```bash
php volt database:migrate:shadow \
    --analyze-eligibility
```

---

# Parte LVI — Configuration

## 194. Example

```php
return [
    'shadow' => [
        'enabled' => true,

        'default_sample_rate' => 0.05,

        'timeout_ms' => 250,

        'units' => [
            'billing' => [
                'primary' => 'legacy',
                'shadow' => 'voltstack',
                'sample_rate' => 0.10,
            ],
        ],
    ],
];
```

---

## 195. Security

La configuración no deberá contener:

```text
raw credentials
sensitive comparison payloads
```

---

# Parte LVII — Storage

## 196. Evidence Store

```text
MigrationShadowEvidenceStore
```

---

## 197. Storage Options

Podrá utilizar:

```text
structured logs
telemetry backend
migration evidence database
local development files
```

según entorno.

---

## 198. Retention

La evidencia tendrá:

```text
retention policy
```

especialmente en producción.

---

## 199. Sensitive Retention

Hashes y summaries deberán preferirse sobre payloads completos.

---

# Parte LVIII — Testing

## 200. Unit Tests

Para:

```text
normalizers
comparators
sampling
difference classification
eligibility
privacy filters
```

---

## 201. Integration Tests

```text
legacy executor
VoltStack executor
same DB
read-only enforcement
timeouts
```

---

## 202. Result Fixtures

Casos:

```text
exact match
normalized match
missing row
extra row
wrong order
duplicate
null difference
type difference
exception difference
```

---

## 203. Large Dataset

Verificar:

```text
streaming comparison
memory ceiling
chunking
hashing
```

---

## 204. Security Tests

```text
no secret persistence
tenant isolation
shadow read-only
redacted diagnostics
```

---

## 205. FrankenPHP Tests

Múltiples requests para detectar:

```text
context leakage
memory retention
tenant leakage
buffer leakage
```

---

## 206. Capacity Tests

Simular:

```text
DB saturation
pool exhaustion
shadow timeout storm
```

y confirmar que primary permanece protegido.

---

# Parte LIX — Error Model

## 207. Exceptions

Conceptualmente:

```text
MigrationShadowException
MigrationShadowEligibilityException
MigrationShadowExecutionException
MigrationShadowNormalizationException
MigrationShadowComparisonException
MigrationShadowPrivacyException
MigrationShadowCapacityException
```

---

# Parte LX — Integration with 333

## 208. Schema Compatibility

El comparator deberá conocer:

```text
approved schema type differences
column mappings
representation changes
```

para normalizar correctamente.

---

# Parte LXI — Integration with 334

## 209. Behavior Contracts

334 define:

```text
what must be equivalent
```

336 define:

```text
how real query observations are compared at runtime
```

---

## 210. Reuse

Se reutilizarán:

```text
behavior normalizers
difference categories
approved differences
invariants where applicable
```

---

# Parte LXII — Integration with 335

## 211. Dual Runtime

335 controla:

```text
when shadow is enabled
which runtime is primary
which runtime is shadow
```

336 ejecuta y compara.

---

# Parte LXIII — Integration with 337

## 212. Testing

337 consumirá:

```text
shadow evidence
coverage
mismatches
query identities
performance deltas
```

como parte de validación global.

---

# Parte LXIV — Integration with 338

## 213. Developer Experience

338 expondrá:

```text
status
enable/disable
sampling
reports
diagnostics
```

de forma coherente.

---

# Parte LXV — Integration with 339

## 214. Reporting

339 consolidará:

```text
match rates
critical mismatches
coverage
query groups
trend
burn-down
```

---

# Parte LXVI — Integration with 340

## 215. Recovery

Una tasa creciente de diferencias después de cutover puede alimentar:

```text
rollback/recovery decision
```

pero 336 no ejecutará rollback por sí mismo.

---

# Parte LXVII — Example: Soft Delete

## 216. Primary

Legacy:

```text
SELECT *
FROM users
WHERE deleted_at IS NULL
```

retorna:

```text
100 rows
```

---

## 217. Shadow

VoltStack omite filtro y retorna:

```text
103 rows
```

---

## 218. Result

```text
MISMATCH
ROW_COUNT
HIGH
possible cause: soft-delete filter
```

---

# Parte LXVIII — Example: Type Normalization

## 219. Primary

```text
active = 1
```

---

## 220. Shadow

```text
active = true
```

---

## 221. Contract

Si 334 declara boolean semantic equivalence:

```text
MATCH_AFTER_NORMALIZATION
```

---

# Parte LXIX — Example: Ordering

## 222. Primary

```text
[5, 4, 3, 2, 1]
```

Shadow:

```text
[1, 2, 3, 4, 5]
```

---

## 223. If Order Contract Exists

```text
ORDER_MISMATCH
```

---

## 224. If Order Is Undefined

Podrá compararse como multiset y producir:

```text
MATCH_AFTER_NORMALIZATION
```

---

# Parte LXX — Example: Replica Lag

## 225. Primary

Consulta primary DB inmediatamente después de write.

---

## 226. Shadow

Consulta read replica retrasada.

---

## 227. Result

La diferencia deberá poder clasificarse:

```text
POTENTIAL_REPLICATION_LAG
```

en vez de asumir un bug del nuevo ORM.

---

# Parte LXXI — Example: Tenant Leak

## 228. Request

```text
tenant = A
```

---

## 229. Primary

```text
10 rows from tenant A
```

---

## 230. Shadow

```text
11 rows
including tenant B
```

---

## 231. Result

```text
CRITICAL_SECURITY_DIFFERENCE
```

y deberá bloquear el cutover de esa unidad.

---

# Parte LXXII — Example: Large Report

## 232. Query

```text
500,000 rows
```

---

## 233. Strategy

No materializar dos arrays completos.

Usar:

```text
streaming canonicalization
stable-key comparison
incremental hash
difference sampling
```

---

# Parte LXXIII — Example: Shadow Timeout

## 234. Primary

```text
25 ms
```

Shadow:

```text
> 250 ms timeout
```

---

## 235. Result

Usuario recibe primary normalmente.

Evidence:

```text
SHADOW_TIMEOUT
PERFORMANCE/EXECUTION
```

---

# Parte LXXIV — V1 Scope

## 236. Included

Database V1 incluye:

```text
read shadowing
eligibility analysis
sampling
result normalization
semantic comparison
difference classification
privacy filtering
capacity protection
timeouts
evidence
telemetry
persistent-worker safety
cutover evidence
```

---

## 237. Excluded

No pertenece a V1:

```text
automatic data reconciliation
distributed active-active ORM writes
autonomous AI cutover decisions
cross-region shadow fabric
generic CDC replication engine
```

---

# Parte LXXV — Decisiones arquitectónicas

## 238. Decisión 1

Shadow será read-only por defecto.

## 239. Decisión 2

El primary será siempre la autoridad de respuesta.

## 240. Decisión 3

Los errores shadow no romperán primary traffic por defecto.

## 241. Decisión 4

La equivalencia será semántica, no igualdad obligatoria de SQL.

## 242. Decisión 5

Las reglas de normalización serán explícitas y auditables.

## 243. Decisión 6

Queries desconocidas o con side effects no serán elegibles automáticamente.

## 244. Decisión 7

El sistema utilizará sampling y capacity guards.

## 245. Decisión 8

Los resultados productivos completos no se persistirán por defecto.

## 246. Decisión 9

La privacidad se aplicará antes de almacenar evidencia.

## 247. Decisión 10

Orden solo será significativo cuando el contrato lo requiera.

## 248. Decisión 11

Collections conservarán duplicados durante comparación.

## 249. Decisión 12

Los errores operacionales se distinguirán de mismatches funcionales.

## 250. Decisión 13

La cobertura y confianza serán multidimensionales, no un score opaco.

## 251. Decisión 14

Shadow podrá ejecutarse en ambas direcciones durante la transición.

## 252. Decisión 15

El sistema protegerá tráfico principal antes que cobertura shadow.

## 253. Decisión 16

FrankenPHP requerirá contexto y buffers request-scoped.

## 254. Decisión 17

Shadow evidence podrá bloquear una transición, pero no cambiará el runtime automáticamente.

## 255. Decisión 18

Shadow deberá retirarse una vez completada la ventana de verificación.

---

# Parte LXXVI — Criterios de preparación para cutover

## 256. Read Cutover Readiness

Una unidad podrá considerarse preparada cuando:

```text
schema compatibility passes
critical behavior contracts pass
required queries are shadow eligible
required query identities observed
no critical security mismatch
no unresolved high mismatch
sampling coverage satisfies policy
capacity impact acceptable
performance regressions understood
```

---

## 257. Insufficient Evidence

Si la query crítica apenas fue observada:

```text
INSUFFICIENT_EVIDENCE
```

aunque todas las muestras disponibles coincidan.

---

# Parte LXXVII — Completion Criteria

## 258. Shadow Phase Completion

La fase podrá cerrarse cuando:

```text
required coverage reached
differences resolved or approved
cutover criteria met
post-cutover verification completed
legacy shadow no longer required
```

---

# Parte LXXVIII — Resultado esperado

## 259. Antes

```text
"We think the VoltStack query behaves the same."
```

---

## 260. Después

```text
Shadow Evidence
│
├── Query Identity
├── Production-like Inputs
├── Primary Observation
├── Shadow Observation
├── Normalization
├── Semantic Comparison
├── Differences
├── Performance Delta
├── Coverage
├── Confidence
└── Cutover Evidence
```

---

# Parte LXXIX — Principio final

## 261. Regla

```text
Do not switch reads because the new query looks correct.

Observe it beside the old query.

Compare what matters.

Measure the differences.

Then decide whether the migration can advance.
```

---

# Parte LXXX — Conclusión

## 262. Arquitectura final

`DATABASE_SHADOW_QUERY_AND_RESULT_COMPARISON_SYSTEM` proporciona la capa de evidencia runtime que conecta las pruebas de migración con tráfico y datos representativos.

La arquitectura queda:

```text
                  Application
                      │
                      ▼
             Dual Runtime Router
                      │
                      ▼
               Shadow Policy
                      │
             ┌────────┴────────┐
             ▼                 ▼
        Primary Runtime    Shadow Runtime
             │                 │
             ▼                 ▼
       Primary Result      Shadow Result
             │                 │
             └────────┬────────┘
                      ▼
                  Normalize
                      │
                      ▼
                   Compare
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       Match      Difference   Operational
                                 Failure
          │           │           │
          └───────────┴───────────┘
                      │
                      ▼
               Evidence Store
                      │
                      ▼
             Migration Reporting
                      │
                      ▼
              Cutover Validation
```

Su relación con la arquitectura anterior queda:

```text
333 verifies physical schema compatibility.

334 defines behavioral equivalence.

335 controls coexistence and runtime ownership.

336 observes and compares real read behavior.

337 will combine all evidence into migration testing and validation.
```

La regla arquitectónica definitiva será:

```text
Primary serves.

Shadow observes.

Comparison produces evidence.

Evidence enables controlled migration.
```

---

**Documento:** `336_DATABASE_SHADOW_QUERY_AND_RESULT_COMPARISON_SYSTEM.md`  
**Proyecto:** VoltStack Framework  
**Módulo:** `VoltStack/Quantum/Database`  
**Versión objetivo:** Database V1  
**MIM objetivo:** `1.x`  
**Estado:** Architectural Specification
