# 281_DATABASE_DIAGNOSTICS_SYSTEM.md

# VoltStack Quantum Database
## Database Diagnostics System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 281 — Database Diagnostics System  
**Bloque:** 28 — Backup and Operations  
**Estado:** System Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `280_DATABASE_HEALTH_CHECK_SYSTEM.md`  
**Siguiente documento:** `282_DATABASE_ADMINISTRATION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura del **Database Diagnostics System** de:

```text
VoltStack/Quantum/Database
```

El sistema será responsable de recopilar, correlacionar, clasificar y presentar evidencia técnica que permita investigar problemas relacionados con:

- conexiones;
- consultas;
- transacciones;
- locks;
- deadlocks;
- connection pools;
- replicas;
- replicación;
- capacidad;
- almacenamiento;
- rendimiento;
- ORM;
- hidratación;
- caché;
- mantenimiento;
- backups;
- recuperación;
- multitenancy;
- sharding;
- topología;
- runtimes persistentes.

La regla fundamental será:

> **Un síntoma observado no constituye por sí mismo una causa demostrada.**

Formalmente:

```text
ObservedSymptom
≠
RootCause
```

y:

```text
Correlation
≠
Causation
```

Diagnostics deberá ayudar a reducir el espacio de investigación sin inventar certezas que la evidencia disponible no permita establecer.

---

# 2. Objetivo arquitectónico

El sistema deberá transformar:

```text
"Database is degraded"
```

en información estructurada como:

```text
Observed problem
    ↓
Relevant evidence
    ↓
Correlated observations
    ↓
Candidate explanations
    ↓
Confidence
    ↓
Recommended investigation
```

sin convertir automáticamente:

```text
Candidate explanation
```

en:

```text
Proven root cause
```

---

# 3. Health vs Diagnostics

El documento anterior definió:

```text
Health
    ↓
What is the current state?
```

Diagnostics responde:

```text
Diagnostics
    ↓
What evidence explains that state?
```

Por tanto:

```text
Health
≠
Diagnostics
```

---

# 4. Ejemplo

Health detecta:

```text
Query latency = HIGH
```

Diagnostics puede encontrar:

```text
connection pool saturation
lock waits
replica lag
high CPU evidence
slow query pattern
large result sets
```

Pero no deberá concluir automáticamente:

```text
"Missing index"
```

sin evidencia suficiente.

---

# 5. Principio de evidencia

Toda conclusión diagnóstica deberá poder rastrearse hasta:

```text
DiagnosticFinding
        │
        ▼
DiagnosticEvidence[]
```

Nunca:

```text
Finding
    ↓
unsupported assumption
```

---

# 6. Alcance

Diagnostics podrá investigar:

```text
Connection
Query
Transaction
Lock
Deadlock
Pool
Replica
Replication
Storage
Capacity
Performance
ORM
Hydration
Cache
Maintenance
Backup
Recovery
Tenant
Shard
Topology
Runtime
```

---

# 7. No objetivos

Diagnostics no será directamente responsable de:

```text
repair
automatic schema modification
index creation
query rewriting
failover execution
backup execution
restore execution
migration execution
resource scaling
incident resolution
```

---

# 8. Diagnostics ≠ Remediation

```text
Diagnostics
    ↓
Explain
```

mientras:

```text
Remediation
    ↓
Change system state
```

Son responsabilidades diferentes.

---

# 9. Diagnostics ≠ Monitoring

Monitoring observa continuamente.

Diagnostics investiga una condición o pregunta concreta.

---

# 10. Diagnostics ≠ Telemetry

Telemetry produce observaciones.

Diagnostics puede consumirlas.

```text
Telemetry
    ↓
Evidence
    ↓
Diagnostics
```

---

# 11. Diagnostics ≠ Profiler

Profiler analiza ejecución detallada de determinadas operaciones.

Diagnostics tiene alcance operacional más amplio.

---

# 12. Diagnostics ≠ Health

Health puede ser:

```text
DEGRADED
```

Diagnostics explica:

```text
why the system may be degraded
```

---

# 13. Modelo general

```text
Diagnostic Request
       │
       ▼
Diagnostic Context
       │
       ▼
Diagnostic Planner
       │
       ├── Health Evidence
       ├── Telemetry
       ├── Runtime State
       ├── Database Metadata
       ├── Platform Capabilities
       └── Historical Evidence
       │
       ▼
Evidence Collectors
       │
       ▼
Evidence Normalization
       │
       ▼
Correlation Engine
       │
       ▼
Diagnostic Rules
       │
       ▼
Findings
       │
       ▼
Candidate Causes
       │
       ▼
Confidence Analysis
       │
       ▼
Diagnostic Report
```

---

# 14. DiagnosticRequest

Contrato conceptual:

```php
final readonly class DiagnosticRequest
{
    public function __construct(
        public DiagnosticTarget $target,
        public DiagnosticProfile $profile,
        public ?DiagnosticQuestion $question = null,
        public ?DiagnosticTimeWindow $window = null,
        public ?DiagnosticBudget $budget = null,
    ) {}
}
```

---

# 15. DiagnosticTarget

Podrá representar:

```text
logical database
connection
endpoint
writer
replica
query
transaction
tenant
shard
backup
runtime worker
```

---

# 16. DiagnosticQuestion

Permitirá expresar una investigación concreta.

Ejemplos:

```text
Why are queries slow?
Why is the writer unavailable?
Why is replication lagging?
Why is the pool saturated?
Why did this transaction fail?
Why is backup health degraded?
```

---

# 17. Structured questions

Internamente deberán preferirse identificadores estructurados.

```php
enum DiagnosticQuestionType
{
    case CONNECTION_FAILURE;
    case QUERY_LATENCY;
    case QUERY_FAILURE;
    case TRANSACTION_FAILURE;
    case LOCK_CONTENTION;
    case DEADLOCK;
    case POOL_SATURATION;
    case REPLICATION_LAG;
    case STORAGE_PRESSURE;
    case BACKUP_FAILURE;
    case RECOVERY_DEGRADATION;
    case GENERAL_HEALTH;
}
```

---

# 18. DiagnosticProfile

```php
enum DiagnosticProfile
{
    case QUICK;
    case STANDARD;
    case DEEP;
    case FORENSIC;
}
```

---

# 19. QUICK

Debe ser:

```text
fast
bounded
low-cost
mostly passive
```

---

# 20. STANDARD

Recopila evidencia operacional suficiente para problemas comunes.

---

# 21. DEEP

Puede inspeccionar:

```text
query plans
locks
long transactions
replication state
storage pressure
metadata
cache behavior
ORM state
```

---

# 22. FORENSIC

Perfil de investigación avanzada.

Puede incluir:

```text
historical telemetry
audit records
health history
transaction evidence
topology transitions
failure timelines
```

Debe requerir autorización adecuada.

---

# 23. Deep ≠ destructive

Incluso un diagnóstico profundo deberá ser no destructivo por defecto.

---

# 24. DiagnosticContext

```php
final readonly class DiagnosticContext
{
    public function __construct(
        public DatabaseIdentity $database,
        public Platform $platform,
        public ?TenantContext $tenant,
        public ?ShardContext $shard,
        public DiagnosticPolicy $policy,
        public DiagnosticBudget $budget,
        public Deadline $deadline,
        public CancellationToken $cancellation,
    ) {}
}
```

---

# 25. Context isolation

Todo estado mutable será:

```text
diagnostic-operation scoped
```

---

# 26. Evidence model

```php
final readonly class DiagnosticEvidence
{
    public function __construct(
        public DiagnosticEvidenceId $id,
        public DiagnosticEvidenceType $type,
        public mixed $value,
        public DiagnosticEvidenceSource $source,
        public DiagnosticConfidence $confidence,
        public Instant $observedAt,
        public ?Duration $validFor,
        public array $attributes = [],
    ) {}
}
```

---

# 27. Evidence sources

Ejemplos:

```text
HEALTH_CHECK
TELEMETRY
QUERY_PROFILER
CONNECTION
DATABASE_METADATA
PLATFORM
TRANSACTION
REPLICATION
BACKUP
MAINTENANCE
RUNTIME
AUDIT
EXTERNAL_PROVIDER
```

---

# 28. Source provenance

Cada evidencia deberá conservar su procedencia.

---

# 29. Provenance

Conceptualmente:

```text
Evidence
├── source
├── collection method
├── observed time
├── confidence
└── scope
```

---

# 30. Evidence scope

Una observación podrá ser:

```text
QUERY
CONNECTION
NODE
SHARD
TENANT
DATABASE
APPLICATION_INSTANCE
GLOBAL
```

---

# 31. Scope mismatch

No deberá atribuirse automáticamente:

```text
application instance pool saturation
```

a:

```text
database server failure
```

---

# 32. Temporal evidence

Toda evidencia temporal deberá incluir:

```text
observedAt
```

---

# 33. Evidence freshness

Cuando aplique:

```text
validFor
```

---

# 34. Historical evidence

Podrá utilizarse para analizar:

```text
trend
regression
change point
repeated failure
```

---

# 35. Historical ≠ current

Una anomalía de ayer no demuestra la causa del incidente actual.

---

# 36. Evidence quality

```php
enum DiagnosticEvidenceQuality
{
    case DIRECT;
    case DERIVED;
    case INFERRED;
    case HEURISTIC;
    case UNKNOWN;
}
```

---

# 37. Direct evidence

Ejemplo:

```text
database reports transaction blocked by PID 8472
```

---

# 38. Derived evidence

Ejemplo:

```text
pool utilization = active / capacity
```

---

# 39. Inferred evidence

Ejemplo:

```text
query latency increase correlates with lock waits
```

---

# 40. Heuristic evidence

Ejemplo:

```text
query pattern may benefit from index review
```

No deberá presentarse como certeza.

---

# 41. Evidence confidence

```php
enum DiagnosticConfidence
{
    case HIGH;
    case MEDIUM;
    case LOW;
    case UNKNOWN;
}
```

---

# 42. Evidence collector

```php
interface DiagnosticEvidenceCollector
{
    public function supports(
        DiagnosticRequest $request,
        DiagnosticContext $context
    ): bool;

    public function collect(
        DiagnosticRequest $request,
        DiagnosticContext $context
    ): iterable;
}
```

---

# 43. Collector categories

```text
ConnectionEvidenceCollector
QueryEvidenceCollector
TransactionEvidenceCollector
LockEvidenceCollector
ReplicationEvidenceCollector
StorageEvidenceCollector
PerformanceEvidenceCollector
OrmEvidenceCollector
CacheEvidenceCollector
BackupEvidenceCollector
TopologyEvidenceCollector
```

---

# 44. Collector ≠ analyzer

Collector obtiene evidencia.

Analyzer la interpreta.

---

# 45. Diagnostic Analyzer

```php
interface DiagnosticAnalyzer
{
    public function analyze(
        DiagnosticEvidenceSet $evidence,
        DiagnosticContext $context
    ): array;
}
```

---

# 46. DiagnosticFinding

```php
final readonly class DiagnosticFinding
{
    public function __construct(
        public DiagnosticFindingCode $code,
        public DiagnosticSeverity $severity,
        public string $summary,
        public array $evidenceIds,
        public DiagnosticConfidence $confidence,
        public array $candidateCauses = [],
        public array $recommendedInvestigations = [],
    ) {}
}
```

---

# 47. Finding ≠ cause

Ejemplo:

```text
Finding:
High query latency
```

puede tener candidatos:

```text
lock contention
pool saturation
storage latency
query plan regression
replica overload
```

---

# 48. CandidateCause

```php
final readonly class CandidateCause
{
    public function __construct(
        public CandidateCauseCode $code,
        public DiagnosticConfidence $confidence,
        public array $supportingEvidence,
        public array $contradictingEvidence,
    ) {}
}
```

---

# 49. Contradicting evidence

Será de primera clase.

Ejemplo:

```text
Candidate:
Storage saturation

Supporting:
query latency increased

Contradicting:
storage latency normal
IO utilization normal
```

Resultado:

```text
LOW CONFIDENCE
```

---

# 50. Negative evidence

La ausencia de una señal esperada puede reducir confianza.

---

# 51. Absence ≠ proof

Sin embargo:

```text
No lock evidence
```

no siempre significa:

```text
Locks definitely not involved
```

si la cobertura es incompleta.

---

# 52. Diagnostic coverage

```php
final readonly class DiagnosticCoverage
{
    public function __construct(
        public int $plannedCollectors,
        public int $completedCollectors,
        public int $failedCollectors,
        public int $unsupportedCollectors,
        public int $blockedCollectors,
    ) {}
}
```

---

# 53. Coverage affects confidence

Diagnóstico:

```text
HIGH CONFIDENCE
```

con:

```text
20% evidence coverage
```

deberá ser excepcional y justificable.

---

# 54. Correlation Engine

Responsable de correlacionar evidencia por:

```text
time
query
connection
transaction
tenant
shard
endpoint
worker
trace
```

---

# 55. Correlation ≠ causation

Regla absoluta.

---

# 56. Temporal correlation

Ejemplo:

```text
10:00 pool saturation
10:01 query latency spike
```

puede elevar la relevancia.

No prueba causalidad.

---

# 57. Correlation identifiers

Podrán utilizarse:

```text
trace_id
span_id
query_fingerprint
transaction_id
connection_id
operation_id
```

sin exponer datos sensibles.

---

# 58. Query diagnostics

Integración con:

```text
216_DATABASE_TELEMETRY_ARCHITECTURE.md
217_DATABASE_QUERY_TELEMETRY_SYSTEM.md
221_DATABASE_QUERY_PROFILER_SYSTEM.md
222_DATABASE_SLOW_QUERY_DETECTION_SYSTEM.md
```

---

# 59. Query evidence

Podrá incluir:

```text
semantic query fingerprint
compiled query fingerprint
duration
rows examined
rows returned
execution plan
timeouts
retries
connection role
shard
replica
```

según disponibilidad.

---

# 60. Raw SQL

No deberá incluirse por defecto en reportes públicos.

---

# 61. Query fingerprint

Se preferirá:

```text
semantic fingerprint
```

sobre SQL completo.

---

# 62. Query plan diagnostics

Podrá analizar:

```text
full scan
index usage
join strategy
sort
temporary structure
estimated cardinality
actual cardinality
```

cuando la plataforma lo permita.

---

# 63. Plan ≠ diagnosis

Un full scan no implica automáticamente un problema.

Ejemplo:

```text
table = 20 rows
```

puede hacer el scan perfectamente razonable.

---

# 64. Plan regression

Si existe evidencia histórica:

```text
previous plan
    ↓
new plan
```

podrá detectarse un cambio.

---

# 65. Regression ≠ cause

Será candidato de alta relevancia, no causalidad automática.

---

# 66. Slow query investigation

Pipeline:

```text
Slow Query
    ↓
Query Profile
    ↓
Plan
    ↓
Lock Evidence
    ↓
Connection Evidence
    ↓
Storage Evidence
    ↓
Load Evidence
    ↓
Candidate Causes
```

---

# 67. Connection diagnostics

Podrá analizar:

```text
DNS
transport
TLS
authentication
session setup
acquisition latency
reset failures
disconnects
protocol errors
```

---

# 68. Connection failure classification

```text
RESOLUTION_FAILURE
NETWORK_FAILURE
TLS_FAILURE
AUTHENTICATION_FAILURE
SERVER_REJECTION
POOL_EXHAUSTION
TIMEOUT
PROTOCOL_FAILURE
UNKNOWN
```

---

# 69. Pool exhaustion ≠ database down

Regla explícita.

---

# 70. Connection timeout

Puede ocurrir por:

```text
network
pool wait
server overload
DNS
TLS
firewall
```

Diagnostics deberá preservar esta ambigüedad.

---

# 71. Transaction diagnostics

Integración con:

```text
164–175
```

---

# 72. Transaction evidence

```text
transaction age
isolation
retry count
savepoints
locks
deadlocks
rollback reason
commit outcome
connection failure
```

---

# 73. UNKNOWN commit

Si:

```text
COMMIT sent
connection lost
```

Diagnostics deberá conservar:

```text
TRANSACTION_OUTCOME_UNKNOWN
```

---

# 74. No invented rollback

Nunca:

```text
connection lost
→ transaction rolled back
```

sin evidencia.

---

# 75. Lock diagnostics

Podrá construir:

```text
Wait-For Graph
```

---

# 76. Wait-for graph

```text
Tx A
 │ waits for
 ▼
Tx B
 │ waits for
 ▼
Tx C
```

---

# 77. Lock chain

Permitirá identificar:

```text
blocker
blocked transactions
duration
resource
```

---

# 78. Deadlock

```text
Tx A → Tx B
 ↑       ↓
 └───────┘
```

representa ciclo.

---

# 79. Deadlock evidence

Si el motor identifica directamente la víctima y el ciclo:

```text
DIRECT evidence
```

---

# 80. Deadlock inference

Si sólo existen síntomas:

```text
LOW/MEDIUM confidence
```

según evidencia.

---

# 81. Long transaction diagnostics

Puede correlacionar:

```text
long transaction
    ↓
locks
    ↓
blocked queries
    ↓
pool pressure
```

---

# 82. Root blocker candidate

Una transacción que bloquea 500 operaciones será un candidato importante.

Pero el sistema deberá distinguir:

```text
Root blocker
```

de:

```text
Business reason why transaction remained open
```

---

# 83. Replication diagnostics

Integración con:

```text
178_DATABASE_REPLICA_SYSTEM.md
179_DATABASE_REPLICA_LAG_AWARENESS_SYSTEM.md
```

---

# 84. Replication evidence

```text
source connectivity
receive position
replay position
lag
replication errors
worker state
queue depth
WAL/binlog position
```

según plataforma.

---

# 85. Lag decomposition

Cuando sea posible:

```text
ReplicationLag =
TransportDelay
+
ApplyDelay
+
QueueDelay
```

conceptualmente.

---

# 86. Lag cause candidates

```text
source unavailable
network delay
replica overloaded
long-running apply
storage pressure
replication worker stopped
large transaction
```

---

# 87. Replica health vs lag cause

No deberán fusionarse.

---

# 88. Storage diagnostics

Podrá consumir:

```text
database storage evidence
filesystem/infrastructure evidence
capacity evidence
IO evidence
```

---

# 89. Storage boundary

Database subsystem no se convertirá en un OS monitoring framework.

---

# 90. External infrastructure evidence

Será incorporada mediante adapters.

---

# 91. Capacity diagnostics

Podrá analizar:

```text
current usage
growth rate
remaining capacity
historical trend
estimated exhaustion
quota
tablespace
temporary space
```

---

# 92. Forecast confidence

Toda proyección deberá indicar confianza.

---

# 93. Performance diagnostics

Podrá correlacionar:

```text
query latency
throughput
pool pressure
locks
storage
replication
CPU evidence
memory evidence
```

cuando estén disponibles.

---

# 94. Performance bottleneck

No se declarará:

```text
CPU bottleneck
```

únicamente porque CPU sea alta.

---

# 95. Saturation vs utilization

```text
High utilization
≠
Saturation
```

Un recurso puede estar muy utilizado sin degradación.

---

# 96. ORM diagnostics

Integración con:

```text
220_DATABASE_ORM_TELEMETRY_SYSTEM.md
244_DATABASE_ORM_PERFORMANCE_SYSTEM.md
248_DATABASE_MEMORY_MANAGEMENT_SYSTEM.md
```

---

# 97. ORM evidence

```text
managed entity count
IdentityMap size
UnitOfWork size
dirty entities
flush duration
change sets
hydration count
relationship loads
lazy loads
N+1 evidence
```

---

# 98. ORM memory growth

Podrá detectarse:

```text
IdentityMap continuously increasing
```

durante procesamiento masivo.

---

# 99. Memory growth ≠ leak

Puede deberse a comportamiento esperado del IdentityMap.

El diagnóstico deberá distinguir:

```text
retained by ORM semantics
```

de:

```text
unexplained retention
```

cuando sea posible.

---

# 100. N+1 diagnostics

Integración con:

```text
154_DATABASE_N_PLUS_ONE_DETECTION_SYSTEM.md
223_DATABASE_N_PLUS_ONE_TELEMETRY_SYSTEM.md
```

---

# 101. N+1 finding

Podrá incluir:

```text
root query fingerprint
relationship
repeated child query fingerprint
observed count
estimated impact
```

---

# 102. N+1 ≠ database engine failure

Se clasificará como:

```text
application data access behavior
```

---

# 103. Hydration diagnostics

Podrá analizar:

```text
rows consumed
entities hydrated
deduplication ratio
hydration duration
conversion duration
relationship assembly
memory
```

---

# 104. Row amplification

Ejemplo:

```text
100 root entities
12,000 physical joined rows
```

podrá producir:

```text
HYDRATION_ROW_AMPLIFICATION
```

---

# 105. Amplification ≠ incorrect query

Puede ser semánticamente correcta pero costosa.

---

# 106. Cache diagnostics

Integración con:

```text
186–192
```

---

# 107. Cache evidence

```text
hit
miss
bypass
stale rejection
invalidation
provider failure
key mismatch
consistency rejection
```

---

# 108. Cache miss ≠ cache failure

Distinción obligatoria.

---

# 109. Physical hit ≠ usable hit

Podrá diagnosticarse:

```text
cache entry found
    ↓
consistency policy rejects
    ↓
logical miss
```

---

# 110. Cache invalidation diagnostics

Podrá investigar:

```text
unexpected stale result
```

mediante:

```text
write event
transaction outcome
invalidation event
cache generation
read event
```

---

# 111. Maintenance diagnostics

Integración con:

```text
279_DATABASE_DATABASE_MAINTENANCE_SYSTEM.md
```

---

# 112. Maintenance evidence

```text
statistics age
maintenance history
vacuum state
index maintenance state
failed maintenance
blocked maintenance
```

---

# 113. Maintenance correlation

Ejemplo:

```text
stale statistics
+
plan regression
+
query latency increase
```

puede elevar:

```text
STATISTICS_STALENESS
```

como causa candidata.

---

# 114. Pero no demostrarla automáticamente

Debe preservarse:

```text
candidate
```

hasta contar con evidencia suficiente.

---

# 115. Backup diagnostics

Integración con:

```text
276_DATABASE_BACKUP_ARCHITECTURE.md
277_DATABASE_BACKUP_SYSTEM.md
```

---

# 116. Backup failure evidence

```text
backup start
backup stage
provider
storage
checksum
encryption
manifest
upload
verification
failure
```

---

# 117. Backup diagnostic example

```text
Backup failed
    ↓
Database snapshot created
    ↓
Upload started
    ↓
Object storage timeout
```

Puede concluir con alta confianza:

```text
backup pipeline failure at storage upload stage
```

sin afirmar:

```text
database failure
```

---

# 118. Restore diagnostics

Integración con:

```text
278_DATABASE_RESTORE_SYSTEM.md
```

---

# 119. Restore evidence

```text
artifact resolution
decryption
checksum
base restore
log replay
schema validation
application validation
```

---

# 120. Recoverability diagnostics

Puede investigar:

```text
RPO violation
broken PITR chain
missing backup
expired retention
missing encryption key
failed restore verification
```

---

# 121. Recovery chain

```text
Base Backup
    ↓
Log Segment A
    ↓
Log Segment B
    ↓
Log Segment C
    ↓
Recovery Point
```

Un segmento ausente puede explicar pérdida de continuidad.

---

# 122. Multitenancy diagnostics

Integración con:

```text
261–266
```

---

# 123. Tenant-specific evidence

```text
tenant resolution
connection resolution
database mapping
schema mapping
migration generation
query context
```

---

# 124. Context leakage

Diagnostics deberá ayudar a detectar:

```text
Tenant A context
    ↓
unexpected query
    ↓
Tenant B database
```

como condición crítica de seguridad.

---

# 125. Sensitive diagnostic

Los detalles deberán restringirse fuertemente.

---

# 126. Sharding diagnostics

Integración con:

```text
183–185
```

---

# 127. Shard evidence

```text
routing key
resolved shard
map generation
ownership
endpoint
query scope
```

---

# 128. Routing failure

Podrá distinguir:

```text
NO_SHARD_RESOLVED
AMBIGUOUS_SHARD
STALE_SHARD_MAP
SHARD_UNAVAILABLE
TOPOLOGY_MISMATCH
```

---

# 129. Global query diagnostics

Para queries distribuidas:

```text
Global Query
    ↓
Shard Plans
    ↓
Shard Executions
    ↓
Merge
```

Diagnostics podrá identificar:

```text
which shard delayed or failed
```

---

# 130. Partial distributed result

Nunca deberá presentarse como complete success si falta un shard requerido.

---

# 131. Topology diagnostics

Podrá comparar:

```text
ExpectedTopology
vs
ObservedTopology
```

---

# 132. Topology findings

```text
unexpected writer
missing replica
role mismatch
orphan node
unknown shard
generation mismatch
```

---

# 133. Split-brain diagnostics

Podrá recopilar evidencia sobre múltiples nodos que reclaman autoridad.

---

# 134. Split-brain safety

Diagnostics no deberá intentar reconciliar automáticamente ownership.

---

# 135. Runtime diagnostics

Especialmente importante para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 136. Runtime evidence

```text
worker id
request scope
connection reuse
state reset
IdentityMap reset
UnitOfWork reset
transaction reset
tenant reset
```

---

# 137. State leakage diagnostics

Ejemplo:

```text
Request A
tenant = A
     ↓
worker reused
     ↓
Request B
unexpected tenant = A
```

Debe producir finding crítico.

---

# 138. Connection reuse diagnostics

Podrá verificar evidencia de:

```text
connection acquired
session configured
query executed
reset attempted
reset completed
connection reused
```

---

# 139. Dirty reused connection

Si una conexión conserva:

```text
transaction
session variable
tenant context
temporary state
```

de un scope anterior:

```text
CRITICAL
```

---

# 140. Diagnostic rule

Las reglas deberán ser explícitas y registrables.

```php
interface DiagnosticRule
{
    public function evaluate(
        DiagnosticEvidenceSet $evidence,
        DiagnosticContext $context
    ): ?DiagnosticFinding;
}
```

---

# 141. Rule Registry

```text
DiagnosticRuleRegistry
```

será frozen tras bootstrap por defecto.

---

# 142. No arbitrary runtime rule mutation

Importante para workers persistentes.

---

# 143. Rule identity

Cada regla tendrá:

```text
stable rule ID
version
required evidence
output finding codes
```

---

# 144. Rule versioning

Permitirá explicar por qué un mismo dataset histórico produjo una interpretación diferente tras actualizar reglas.

---

# 145. Platform-specific rules

Podrán existir:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

sin contaminar el core con:

```php
if ($driver === 'mysql') { ... }
```

---

# 146. Capability-driven diagnostics

```text
Capability
    ↓
Collector eligibility
    ↓
Diagnostic plan
```

---

# 147. Version ≠ capability

Se mantiene la regla global de VoltStack.

---

# 148. MySQL ≠ MariaDB

Tendrán proveedores/capacidades independientes cuando sea necesario.

---

# 149. SQLite

No deberá diagnosticarse usando conceptos de servidor que no apliquen.

Ejemplo:

```text
replica lag
```

puede ser:

```text
NOT_APPLICABLE
```

---

# 150. DiagnosticPlanner

```php
interface DiagnosticPlanner
{
    public function plan(
        DiagnosticRequest $request,
        DiagnosticContext $context
    ): DiagnosticPlan;
}
```

---

# 151. Planner responsibilities

```text
resolve profile
resolve capabilities
select collectors
select rules
build dependencies
apply budget
apply security
apply platform constraints
```

---

# 152. Planner no ejecuta

Regla:

```text
Planner
≠
Runner
```

---

# 153. DiagnosticPlan

```php
final readonly class DiagnosticPlan
{
    public function __construct(
        public array $collectors,
        public array $rules,
        public DiagnosticBudget $budget,
        public DiagnosticSecurityPolicy $security,
    ) {}
}
```

---

# 154. DiagnosticRunner

```php
interface DiagnosticRunner
{
    public function run(
        DiagnosticPlan $plan,
        DiagnosticContext $context
    ): DiagnosticReport;
}
```

---

# 155. Execution pipeline

```text
Plan
 ↓
Collect
 ↓
Normalize
 ↓
Correlate
 ↓
Analyze
 ↓
Rank candidate causes
 ↓
Build report
```

---

# 156. Candidate ranking

El ranking deberá basarse en:

```text
supporting evidence
contradicting evidence
evidence quality
coverage
freshness
rule specificity
```

---

# 157. Ranking ≠ certainty

La causa candidata número uno no será automáticamente root cause.

---

# 158. Candidate score

Podrá existir:

```text
0..1
```

o:

```text
LOW
MEDIUM
HIGH
```

pero siempre acompañado de explicación.

---

# 159. RootCauseStatus

```php
enum RootCauseStatus
{
    case PROVEN;
    case STRONGLY_SUPPORTED;
    case PLAUSIBLE;
    case WEAKLY_SUPPORTED;
    case UNKNOWN;
}
```

---

# 160. PROVEN

Deberá reservarse para evidencia suficientemente directa.

Ejemplo:

```text
database explicitly reports:
disk write failed because device is full
```

---

# 161. Strongly supported

Múltiples evidencias consistentes sin prueba directa absoluta.

---

# 162. Plausible

Explicación compatible, pero no suficientemente demostrada.

---

# 163. Unknown

Cuando no exista suficiente evidencia.

---

# 164. Unknown is valid

El sistema deberá poder concluir:

```text
Cause could not be determined
```

---

# 165. No forced diagnosis

Es preferible:

```text
UNKNOWN
```

que inventar:

```text
"Probably the index"
```

sin fundamento.

---

# 166. DiagnosticReport

```php
final readonly class DiagnosticReport
{
    public function __construct(
        public DiagnosticReportId $id,
        public DiagnosticTarget $target,
        public DiagnosticStatus $status,
        public array $findings,
        public array $candidateCauses,
        public DiagnosticCoverage $coverage,
        public DiagnosticConfidence $confidence,
        public Instant $startedAt,
        public Instant $completedAt,
    ) {}
}
```

---

# 167. DiagnosticStatus

```php
enum DiagnosticStatus
{
    case COMPLETED;
    case PARTIAL;
    case FAILED;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 168. Completed ≠ cause found

Un diagnóstico puede terminar correctamente y producir:

```text
cause = UNKNOWN
```

---

# 169. Partial diagnostics

Debe indicar qué evidencia faltó.

---

# 170. Diagnostic timeline

Podrá construirse:

```text
09:58 deployment completed
10:01 query plan changed
10:02 latency increased
10:03 pool wait increased
10:05 health degraded
```

---

# 171. Timeline ≠ causality

El deployment anterior al incidente no demuestra que lo causó.

---

# 172. Change correlation

Podrá correlacionarse con:

```text
schema migration
deployment
configuration change
topology change
maintenance
failover
```

si las integraciones correspondientes aportan evidencia.

---

# 173. External changes

El core Database no deberá depender de CI/CD.

Podrá consumir eventos externos mediante adapters.

---

# 174. Diagnostic graph

Para investigaciones complejas:

```text
Symptom
   │
   ├── Evidence A
   │      └── Candidate X
   │
   ├── Evidence B
   │      ├── Candidate X
   │      └── Candidate Y
   │
   └── Evidence C
          └── contradicts Candidate Y
```

---

# 175. DiagnosticGraph

Podrá modelar:

```text
SymptomNode
EvidenceNode
FindingNode
CandidateCauseNode
ContradictionEdge
SupportEdge
CorrelationEdge
```

---

# 176. Graph ≠ Query Semantic Graph

No confundir con:

```text
DATABASE_QUERY_SEMANTIC_GRAPH_SYSTEM
```

---

# 177. Recommended investigation

Un finding podrá sugerir:

```text
inspect blocking transaction
compare query plan
verify replica apply worker
check storage latency
review backup storage provider
```

---

# 178. Recommended investigation ≠ automatic fix

Nunca ejecutar automáticamente desde Diagnostics.

---

# 179. RemediationHint

```php
final readonly class RemediationHint
{
    public function __construct(
        public RemediationHintCode $code,
        public string $description,
        public RemediationRisk $risk,
        public bool $automaticExecutionAllowed = false,
    ) {}
}
```

---

# 180. Default

```text
automaticExecutionAllowed = false
```

---

# 181. Destructive suggestions

Ejemplos como:

```text
kill transaction
drop index
restart database
promote replica
```

deberán presentarse con advertencias y autorización fuera del Diagnostic Core.

---

# 182. Security architecture

Diagnostics puede revelar más información que Health.

Por tanto tendrá políticas más estrictas.

---

# 183. Sensitive evidence

```text
SQL
bindings
PII
credentials
hostnames
IP addresses
schema names
table names
tenant IDs
backup paths
filesystem paths
topology
```

---

# 184. DiagnosticDetailLevel

```php
enum DiagnosticDetailLevel
{
    case PUBLIC;
    case DEVELOPER;
    case OPERATOR;
    case ADMINISTRATOR;
    case FORENSIC;
}
```

---

# 185. PUBLIC

Información mínima.

---

# 186. DEVELOPER

Información útil para desarrollo sin secretos.

---

# 187. OPERATOR

Detalles operacionales.

---

# 188. ADMINISTRATOR

Topología y configuración permitida.

---

# 189. FORENSIC

Máximo detalle autorizado.

---

# 190. Parameter redaction

Bindings sensibles deberán pasar por:

```text
SensitiveDataProtectionSystem
```

definido en:

```text
232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM.md
```

---

# 191. Credential protection

Integración con:

```text
229_DATABASE_CREDENTIAL_SECURITY_SYSTEM.md
```

---

# 192. Query audit

Integración con:

```text
233_DATABASE_QUERY_AUDIT_SYSTEM.md
```

cuando la investigación requiera evidencia histórica autorizada.

---

# 193. Audit ≠ Diagnostics

Diagnostics puede leer evidencia auditada.

No redefine el audit trail.

---

# 194. Diagnostic audit

Investigaciones sensibles podrán generar:

```text
DiagnosticAccessedSensitiveEvidence
```

---

# 195. Least privilege

Collectors deberán utilizar el mínimo privilegio necesario.

---

# 196. Privileged collector

Ejemplo:

```text
LockGraphCollector
```

puede necesitar privilegios adicionales.

---

# 197. Privilege unavailable

Resultado:

```text
collector = BLOCKED
reason = INSUFFICIENT_PRIVILEGE
```

No:

```text
no locks exist
```

---

# 198. Diagnostic budget

```php
final readonly class DiagnosticBudget
{
    public function __construct(
        public Duration $maxDuration,
        public int $maxQueries,
        public int $maxConnections,
        public int $maxEvidenceItems,
        public int $maxHistoricalRecords,
    ) {}
}
```

---

# 199. Resource protection

Diagnostics no deberá agravar el incidente que investiga.

---

# 200. Diagnostic storm

Durante una caída podrían iniciarse muchas investigaciones.

Se requerirán:

```text
rate limiting
single-flight
coalescing
budgeting
cancellation
```

---

# 201. Expensive queries

Collectors no deberán ejecutar scans masivos sin policy explícita.

---

# 202. EXPLAIN

Podrá utilizarse cuando:

```text
safe
supported
authorized
within budget
```

---

# 203. EXPLAIN ANALYZE

Es más invasivo.

No deberá ejecutarse automáticamente sobre consultas arbitrarias de producción.

---

# 204. Safety classification

```php
enum DiagnosticOperationSafety
{
    case PASSIVE;
    case SAFE_READ;
    case POTENTIALLY_EXPENSIVE;
    case INVASIVE;
    case FORBIDDEN;
}
```

---

# 205. FORENSIC no elimina safety

Incluso el perfil FORENSIC deberá respetar políticas explícitas.

---

# 206. Cancellation

Todo collector deberá cooperar con:

```text
CancellationToken
```

cuando sea posible.

---

# 207. Deadline

Ningún diagnóstico deberá ejecutarse indefinidamente.

---

# 208. Telemetry

Spans conceptuales:

```text
database.diagnostics
├── diagnostics.plan
├── diagnostics.collect
│   ├── connection
│   ├── query
│   ├── transaction
│   ├── locks
│   ├── replication
│   ├── storage
│   └── backup
├── diagnostics.correlate
├── diagnostics.analyze
└── diagnostics.report
```

---

# 209. Metrics

```text
database_diagnostics_total
database_diagnostics_duration_seconds
database_diagnostics_partial_total
database_diagnostics_failed_total
database_diagnostic_findings_total
database_diagnostic_unknown_total
```

---

# 210. Cardinality

No utilizar directamente como labels:

```text
SQL
query bindings
tenant ID
transaction ID
connection ID
trace ID
```

---

# 211. Diagnostic telemetry recursion

Diagnostics genera telemetría.

Pero dicha telemetría no deberá contaminar la misma investigación de forma recursiva.

---

# 212. Self-observation marker

Las operaciones diagnósticas podrán marcarse:

```text
diagnostic_operation = true
```

---

# 213. Query profiler exclusion

Podrá excluir o clasificar consultas de diagnóstico para evitar:

```text
health/diagnostic query
→ slow query detector
→ diagnostic trigger
→ more diagnostic queries
```

---

# 214. Feedback loop protection

Regla obligatoria.

---

# 215. Persistent runtime

Compatible con:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 216. Immutable shared components

Podrán compartirse:

```text
DiagnosticRuleRegistry
collector definitions
compiled policies
platform capability metadata
```

---

# 217. Scoped components

Deberán ser locales:

```text
DiagnosticContext
EvidenceSet
CorrelationContext
ReportBuilder
current tenant
current shard
deadline
cancellation
```

---

# 218. Forbidden state

```php
Diagnostics::$currentEvidence;
Diagnostics::$currentTenant;
Diagnostics::$currentReport;
Diagnostics::$currentConnection;
```

---

# 219. Worker cleanup

Al terminar:

```text
cancel collectors
release diagnostic connections
clear evidence
clear context
release privileged credentials
close result cursors
flush bounded telemetry
```

---

# 220. Coroutine isolation

OpenSwoole deberá mantener contextos independientes por coroutine.

---

# 221. Connection reuse

Una conexión usada para diagnóstico deberá:

```text
reset
```

antes de volver al pool.

---

# 222. Uncertain connection state

Si no puede demostrarse limpieza:

```text
discard connection
```

---

# 223. Transaction safety

Collectors no dejarán:

```text
open transaction
savepoint
lock
temporary session mutation
```

tras finalizar.

---

# 224. CLI

Interfaz conceptual:

```text
volt database:diagnose
```

---

# 225. Profiles

```text
volt database:diagnose --profile=quick
volt database:diagnose --profile=standard
volt database:diagnose --profile=deep
volt database:diagnose --profile=forensic
```

---

# 226. Question

```text
volt database:diagnose \
  --question=query-latency
```

---

# 227. Connection

```text
volt database:diagnose \
  --connection=primary
```

---

# 228. Query fingerprint

```text
volt database:diagnose \
  --query-fingerprint=<fingerprint>
```

---

# 229. Shard

```text
volt database:diagnose \
  --shard=shard-03
```

---

# 230. Tenant

Cuando esté autorizado:

```text
volt database:diagnose \
  --tenant=<tenant-reference>
```

---

# 231. Explain report

```text
volt database:diagnose --explain
```

---

# 232. JSON

```text
volt database:diagnose --format=json
```

---

# 233. Example output

```text
VoltStack Database Diagnostics

Target
  primary

Question
  query-latency

Status
  COMPLETED

Confidence
  HIGH

Observed Symptoms
  Query latency increased 4.8x
  Pool acquisition latency increased 7.2x

Findings

  DB_DIAG_LOCK_CONTENTION
    Severity: HIGH
    Confidence: HIGH

    Evidence:
      37 blocked transactions
      longest wait: 14.8s
      common blocker transaction: tx-***

  DB_DIAG_LONG_TRANSACTION
    Severity: HIGH
    Confidence: HIGH

    Evidence:
      transaction age: 18m 42s
      blocked operations: 31

Candidate Causes

  1. Long-running transaction producing lock contention
     Confidence: HIGH

     Supporting:
       blocker appears in 31 waits
       latency spike began after transaction start
       affected queries target locked resources

     Contradicting:
       none observed

  2. Connection pool saturation
     Confidence: MEDIUM

     Supporting:
       pool utilization 96%
       acquisition latency high

     Contradicting:
       saturation began after lock waits increased

Recommended Investigation

  Inspect blocking transaction.
  Determine why it remained open.
  Review transaction ownership before cancellation.

Coverage
  14 / 15 collectors

Unavailable Evidence
  OS-level storage latency
```

---

# 234. Developer API

```php
$report = DB::diagnostics()->run(
    DiagnosticProfile::STANDARD
);
```

---

# 235. Question-specific

```php
$report = DB::diagnostics()->investigate(
    DiagnosticQuestion::queryLatency()
);
```

---

# 236. Query

```php
$report = DB::diagnostics()
    ->query($fingerprint)
    ->run();
```

---

# 237. Connection

```php
$report = DB::connection('primary')
    ->diagnostics()
    ->run();
```

---

# 238. No hidden forensic work

Una llamada normal no deberá iniciar automáticamente:

```text
full forensic analysis
large historical scans
EXPLAIN ANALYZE
privileged queries
```

---

# 239. Diagnostic escalation

Podrá existir:

```text
QUICK
  ↓
STANDARD
  ↓
DEEP
  ↓
FORENSIC
```

---

# 240. Escalation policy

Si QUICK encuentra suficiente evidencia:

```text
no need for DEEP
```

---

# 241. Escalation bounded

Nunca escalar indefinidamente.

---

# 242. Automatic escalation

Puede permitirse únicamente dentro de:

```text
policy
budget
authorization
safety limits
```

---

# 243. Integration with Health

```text
Health Check
    ↓
HealthFinding
    ↓
DiagnosticRequest
    ↓
Diagnostics
```

---

# 244. Health evidence reuse

No repetir probes recientes innecesariamente.

---

# 245. Reuse requires freshness

Evidencia vieja no deberá reutilizarse como current evidence.

---

# 246. Integration with Resource Governance

```text
Diagnostics
    ↓
Resource Governance
    ↓
budget / throttle / deny
```

---

# 247. Integration with Resilience

Diagnostics puede aportar evidencia a:

```text
Retry
Circuit Breaker
Failover
Recovery
```

pero no toma esas decisiones por sí mismo.

---

# 248. Integration with Administration

El próximo:

```text
282_DATABASE_ADMINISTRATION_SYSTEM.md
```

podrá presentar y consumir diagnósticos.

---

# 249. Directory structure

```text
src/Quantum/Database/Diagnostics/
├── Contract/
│   ├── DiagnosticPlanner.php
│   ├── DiagnosticRunner.php
│   ├── DiagnosticAnalyzer.php
│   ├── DiagnosticEvidenceCollector.php
│   └── DiagnosticRule.php
│
├── Request/
│   ├── DiagnosticRequest.php
│   ├── DiagnosticTarget.php
│   └── DiagnosticQuestion.php
│
├── Context/
│   └── DiagnosticContext.php
│
├── Profile/
│   └── DiagnosticProfile.php
│
├── Evidence/
│   ├── DiagnosticEvidence.php
│   ├── DiagnosticEvidenceSet.php
│   ├── DiagnosticEvidenceType.php
│   ├── DiagnosticEvidenceSource.php
│   ├── DiagnosticEvidenceQuality.php
│   └── DiagnosticEvidenceNormalizer.php
│
├── Collector/
│   ├── ConnectionEvidenceCollector.php
│   ├── QueryEvidenceCollector.php
│   ├── TransactionEvidenceCollector.php
│   ├── LockEvidenceCollector.php
│   ├── ReplicationEvidenceCollector.php
│   ├── StorageEvidenceCollector.php
│   ├── PerformanceEvidenceCollector.php
│   ├── OrmEvidenceCollector.php
│   ├── CacheEvidenceCollector.php
│   ├── MaintenanceEvidenceCollector.php
│   ├── BackupEvidenceCollector.php
│   └── TopologyEvidenceCollector.php
│
├── Planning/
│   ├── DiagnosticPlanner.php
│   ├── DiagnosticPlan.php
│   └── DiagnosticDependencyGraph.php
│
├── Correlation/
│   ├── DiagnosticCorrelationEngine.php
│   ├── DiagnosticTimeline.php
│   └── DiagnosticGraph.php
│
├── Analysis/
│   ├── DiagnosticAnalyzer.php
│   ├── CandidateCause.php
│   ├── RootCauseStatus.php
│   └── DiagnosticConfidence.php
│
├── Rule/
│   ├── DiagnosticRule.php
│   ├── DiagnosticRuleRegistry.php
│   ├── Connection/
│   ├── Query/
│   ├── Transaction/
│   ├── Lock/
│   ├── Replication/
│   ├── Performance/
│   ├── ORM/
│   ├── Cache/
│   ├── Backup/
│   └── Recovery/
│
├── Finding/
│   ├── DiagnosticFinding.php
│   ├── DiagnosticFindingCode.php
│   └── DiagnosticSeverity.php
│
├── Recommendation/
│   ├── RemediationHint.php
│   └── RecommendedInvestigation.php
│
├── Report/
│   ├── DiagnosticReport.php
│   ├── DiagnosticReportBuilder.php
│   └── DiagnosticCoverage.php
│
├── Security/
│   ├── DiagnosticDetailLevel.php
│   ├── DiagnosticSecurityPolicy.php
│   └── DiagnosticEvidenceSanitizer.php
│
├── Telemetry/
│   └── ...
│
└── Exception/
    └── ...
```

---

# 250. Dependency model

```text
Diagnostics
    │
    ├── Health
    ├── Telemetry
    ├── Query Profiler
    ├── Connection
    ├── Transaction
    ├── Replica
    ├── Distribution
    ├── ORM Telemetry
    ├── Cache
    ├── Maintenance
    ├── Backup
    ├── Restore
    ├── Security
    └── Resource Governance
```

---

# 251. Dependency direction

Diagnostics consume evidencia.

Los subsistemas inferiores no deberán depender del Diagnostic Engine.

Incorrecto:

```text
Connection
    ↓
Diagnostics
```

como dependencia obligatoria.

Correcto:

```text
Diagnostics
    ↓
Connection evidence API
```

---

# 252. Testing

Se requerirán:

```text
unit tests
integration tests
platform tests
failure injection
correlation tests
confidence tests
security tests
budget tests
runtime isolation tests
distributed tests
```

---

# 253. Synthetic evidence tests

Deberá poder probarse el analyzer sin una base real.

```php
$evidence = DiagnosticEvidenceSet::from([
    Evidence::lockWait(...),
    Evidence::longTransaction(...),
    Evidence::poolSaturation(...),
]);

$report = $analyzer->analyze($evidence);
```

---

# 254. Correlation tests

Verificar:

```text
related evidence
```

contra:

```text
coincidental evidence
```

---

# 255. Contradiction tests

Si aparece evidencia contradictoria, confidence deberá reducirse.

---

# 256. Missing evidence tests

El sistema deberá producir:

```text
UNKNOWN
```

cuando corresponda.

---

# 257. Platform tests

Verificar:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

sin asumir equivalencias falsas.

---

# 258. Persistent runtime tests

```text
Request A diagnostics
    ↓
cleanup
    ↓
Request B diagnostics
```

sin fuga de:

```text
tenant
shard
evidence
connection
transaction
report
```

---

# 259. Security tests

Verificar redacción de:

```text
passwords
bindings
PII
internal hosts
backup paths
tenant identifiers
```

---

# 260. Resource tests

Verificar que Diagnostics respete:

```text
query budget
connection budget
time budget
evidence budget
```

---

# 261. Feedback-loop tests

Asegurar:

```text
diagnostic query
```

no active infinitamente:

```text
slow query detector
→ diagnostics
→ slow query detector
→ diagnostics
```

---

# 262. Invariantes arquitectónicos

## DB-DIAG-001

Diagnostics no será Health.

## DB-DIAG-002

Diagnostics no será Telemetry.

## DB-DIAG-003

Diagnostics no será Monitoring.

## DB-DIAG-004

Diagnostics no será Remediation.

## DB-DIAG-005

Diagnostics no será Failover.

## DB-DIAG-006

Síntoma no será root cause.

## DB-DIAG-007

Correlación no será causalidad.

## DB-DIAG-008

Finding no será automáticamente cause.

## DB-DIAG-009

Candidate cause no será proven cause.

## DB-DIAG-010

Toda conclusión deberá tener evidencia rastreable.

## DB-DIAG-011

Evidence tendrá provenance.

## DB-DIAG-012

Evidence tendrá scope.

## DB-DIAG-013

Evidence temporal tendrá timestamp.

## DB-DIAG-014

Freshness será representable.

## DB-DIAG-015

Evidence quality será explícita.

## DB-DIAG-016

Confidence será explícita.

## DB-DIAG-017

Contradicting evidence será de primera clase.

## DB-DIAG-018

Missing evidence no será negative evidence automáticamente.

## DB-DIAG-019

Incomplete coverage reducirá certeza cuando corresponda.

## DB-DIAG-020

UNKNOWN será resultado válido.

## DB-DIAG-021

El sistema no estará obligado a encontrar causa.

## DB-DIAG-022

Collector no será Analyzer.

## DB-DIAG-023

Analyzer no adquirirá conexiones directamente.

## DB-DIAG-024

Planner no ejecutará collectors.

## DB-DIAG-025

Runner no redefinirá rules.

## DB-DIAG-026

Rules serán explícitas.

## DB-DIAG-027

Rules tendrán stable identity.

## DB-DIAG-028

Rules podrán versionarse.

## DB-DIAG-029

Platform-specific rules estarán encapsuladas.

## DB-DIAG-030

Version no sustituirá Capability.

## DB-DIAG-031

MySQL no será MariaDB.

## DB-DIAG-032

SQLite no será forzado a conceptos no aplicables.

## DB-DIAG-033

Unsupported no será failure.

## DB-DIAG-034

Permission denied no demostrará ausencia del problema.

## DB-DIAG-035

Probe failure no será target failure automáticamente.

## DB-DIAG-036

Pool exhaustion no será database-down.

## DB-DIAG-037

High CPU no demostrará CPU bottleneck.

## DB-DIAG-038

High utilization no será saturation.

## DB-DIAG-039

Full scan no será automáticamente defectuoso.

## DB-DIAG-040

Plan regression no será automáticamente causa.

## DB-DIAG-041

Slow query no será automáticamente missing index.

## DB-DIAG-042

Long transaction podrá ser blocker sin explicar su origen.

## DB-DIAG-043

Deadlock inference preservará confidence.

## DB-DIAG-044

UNKNOWN commit permanecerá UNKNOWN.

## DB-DIAG-045

Connection loss no implicará rollback.

## DB-DIAG-046

Replica lag tendrá evidencia separada de causa.

## DB-DIAG-047

Storage forecast no será hecho observado.

## DB-DIAG-048

ORM memory growth no será automáticamente memory leak.

## DB-DIAG-049

N+1 no será database-engine failure.

## DB-DIAG-050

Hydration amplification no será automáticamente incorrecta.

## DB-DIAG-051

Cache miss no será cache failure.

## DB-DIAG-052

Physical cache hit no será usable hit.

## DB-DIAG-053

Maintenance correlation no probará causalidad.

## DB-DIAG-054

Backup failure stage será preservable.

## DB-DIAG-055

Backup failure no será database failure automáticamente.

## DB-DIAG-056

Restore diagnostics preservará stages.

## DB-DIAG-057

Recovery chain gaps serán explícitos.

## DB-DIAG-058

Tenant context leakage será condición crítica.

## DB-DIAG-059

Shard routing evidence será explícita.

## DB-DIAG-060

Partial distributed result no será complete success.

## DB-DIAG-061

Topology expected y observed permanecerán separados.

## DB-DIAG-062

Diagnostics no resolverá split-brain automáticamente.

## DB-DIAG-063

Persistent runtime leakage será detectable.

## DB-DIAG-064

Dirty reused connection será detectable.

## DB-DIAG-065

Mutable diagnostic state será operation-scoped.

## DB-DIAG-066

No existirá static current evidence.

## DB-DIAG-067

No existirá static current tenant.

## DB-DIAG-068

No existirá static current report.

## DB-DIAG-069

No existirá static current connection.

## DB-DIAG-070

Registry podrá compartirse sólo si es immutable/frozen.

## DB-DIAG-071

FrankenPHP deberá aislar diagnostics entre requests.

## DB-DIAG-072

RoadRunner deberá aislar diagnostics entre jobs.

## DB-DIAG-073

OpenSwoole deberá aislar diagnostics entre coroutines.

## DB-DIAG-074

Diagnostic connections deberán limpiarse.

## DB-DIAG-075

Uncertain connection state deberá descartarse.

## DB-DIAG-076

Collectors no dejarán transacciones abiertas.

## DB-DIAG-077

Collectors no dejarán locks.

## DB-DIAG-078

Collectors no dejarán session state sin reset.

## DB-DIAG-079

Diagnostics tendrá resource budget.

## DB-DIAG-080

Diagnostics tendrá deadline.

## DB-DIAG-081

Diagnostics será cancelable.

## DB-DIAG-082

Diagnostic storms serán limitables.

## DB-DIAG-083

Collectors costosos requerirán policy.

## DB-DIAG-084

EXPLAIN ANALYZE no será automático en producción.

## DB-DIAG-085

Forensic no significará unrestricted.

## DB-DIAG-086

Sensitive evidence será protegida.

## DB-DIAG-087

Credentials nunca aparecerán en reports.

## DB-DIAG-088

Sensitive bindings serán redacted.

## DB-DIAG-089

Diagnostic detail será authorization-aware.

## DB-DIAG-090

Privileged collectors utilizarán mínimo privilegio.

## DB-DIAG-091

Diagnostic access podrá auditarse.

## DB-DIAG-092

Raw SQL no será público por defecto.

## DB-DIAG-093

Query fingerprints serán preferidos para correlación.

## DB-DIAG-094

Tenant IDs no serán métricas de cardinalidad ilimitada.

## DB-DIAG-095

Telemetry tendrá cardinalidad controlada.

## DB-DIAG-096

Diagnostic queries serán identificables.

## DB-DIAG-097

Diagnostic telemetry no provocará loops infinitos.

## DB-DIAG-098

Self-observation será controlada.

## DB-DIAG-099

Health evidence podrá reutilizarse si es fresca.

## DB-DIAG-100

Stale health evidence no será current evidence.

## DB-DIAG-101

Historical evidence no será current evidence.

## DB-DIAG-102

Timeline ordering no demostrará causalidad.

## DB-DIAG-103

Deployment correlation no demostrará deployment causality.

## DB-DIAG-104

External evidence entrará mediante adapters.

## DB-DIAG-105

Core Diagnostics no dependerá de CI/CD.

## DB-DIAG-106

Core Diagnostics no dependerá de HTTP.

## DB-DIAG-107

Core Diagnostics no dependerá de UI.

## DB-DIAG-108

Core Diagnostics no dependerá de Kubernetes.

## DB-DIAG-109

Core Diagnostics no dependerá de Prometheus.

## DB-DIAG-110

Core Diagnostics no dependerá de Grafana.

## DB-DIAG-111

Core Diagnostics no dependerá de un cloud provider.

## DB-DIAG-112

DiagnosticReport será immutable.

## DB-DIAG-113

DiagnosticPlan será immutable.

## DB-DIAG-114

DiagnosticEvidence será immutable.

## DB-DIAG-115

CandidateCause será immutable.

## DB-DIAG-116

Finding codes serán estables.

## DB-DIAG-117

Messages humanos podrán localizarse.

## DB-DIAG-118

Machine identity no dependerá del mensaje.

## DB-DIAG-119

Candidate ranking será explicable.

## DB-DIAG-120

Ranking no será certeza.

## DB-DIAG-121

PROVEN requerirá evidencia suficiente.

## DB-DIAG-122

STRONGLY_SUPPORTED no será PROVEN.

## DB-DIAG-123

PLAUSIBLE no será PROVEN.

## DB-DIAG-124

LOW confidence será visible.

## DB-DIAG-125

Contradictions reducirán confidence cuando corresponda.

## DB-DIAG-126

No se ocultará evidence contradictoria.

## DB-DIAG-127

Coverage será visible.

## DB-DIAG-128

Partial report será visible como partial.

## DB-DIAG-129

Completed report podrá tener unknown cause.

## DB-DIAG-130

Failed collector no invalidará necesariamente todo el diagnóstico.

## DB-DIAG-131

Critical collector failure podrá afectar report status.

## DB-DIAG-132

Diagnostic escalation será bounded.

## DB-DIAG-133

Automatic escalation respetará authorization.

## DB-DIAG-134

Automatic escalation respetará budget.

## DB-DIAG-135

Automatic escalation respetará safety.

## DB-DIAG-136

Diagnostics no modificará schema.

## DB-DIAG-137

Diagnostics no creará indexes automáticamente.

## DB-DIAG-138

Diagnostics no matará transacciones automáticamente.

## DB-DIAG-139

Diagnostics no reiniciará servidores automáticamente.

## DB-DIAG-140

Diagnostics no promoverá replicas automáticamente.

## DB-DIAG-141

Diagnostics no ejecutará restore automáticamente.

## DB-DIAG-142

Diagnostics no ejecutará backup automáticamente.

## DB-DIAG-143

Diagnostics no ejecutará maintenance automáticamente.

## DB-DIAG-144

Remediation hints serán sólo recomendaciones.

## DB-DIAG-145

Dangerous remediation tendrá risk explícito.

## DB-DIAG-146

Health y Diagnostics compartirán evidence contracts cuando sea razonable.

## DB-DIAG-147

Diagnostics podrá alimentar Resilience.

## DB-DIAG-148

Diagnostics podrá alimentar Administration.

## DB-DIAG-149

Diagnostics podrá alimentar Developer Tooling.

## DB-DIAG-150

Diagnostics podrá alimentar Telemetry.

## DB-DIAG-151

Diagnostics no redefinirá Query Engine.

## DB-DIAG-152

Diagnostics no generará SQL de aplicación.

## DB-DIAG-153

Platform providers generarán probes específicos cuando sea necesario.

## DB-DIAG-154

Queries diagnósticas usarán parameter binding cuando tengan parámetros.

## DB-DIAG-155

Queries diagnósticas respetarán timeout.

## DB-DIAG-156

Queries diagnósticas respetarán cancellation cuando sea posible.

## DB-DIAG-157

Queries diagnósticas respetarán connection policy.

## DB-DIAG-158

Queries diagnósticas no deberán contaminar Result Cache ordinaria.

## DB-DIAG-159

Diagnostics preservará tenant isolation.

## DB-DIAG-160

Diagnostics preservará shard isolation.

## DB-DIAG-161

Cross-tenant diagnostics requerirá autorización explícita.

## DB-DIAG-162

Cross-shard diagnostics será explícito.

## DB-DIAG-163

Cross-shard diagnostics no fingirá atomicidad global.

## DB-DIAG-164

DiagnosticGraph no será QuerySemanticGraph.

## DB-DIAG-165

EvidenceSet no será global mutable state.

## DB-DIAG-166

Correlation context no sobrevivirá accidentalmente al scope.

## DB-DIAG-167

Report builders serán scope-local.

## DB-DIAG-168

Diagnostic rules no dependerán de UI.

## DB-DIAG-169

CLI y API usarán el mismo engine.

## DB-DIAG-170

Future administration UI usará el mismo engine.

## DB-DIAG-171

No existirán motores diagnósticos paralelos por interfaz.

## DB-DIAG-172

La ausencia de causa demostrable será representable.

## DB-DIAG-173

La incertidumbre nunca será convertida silenciosamente en certeza.

## DB-DIAG-174

La evidencia siempre tendrá prioridad sobre heurísticas.

## DB-DIAG-175

VoltStack deberá explicar lo que sabe, lo que infiere y lo que no puede determinar.

---

# 263. Modelo formal

Sea un conjunto de evidencias:

```text
E = {e₁, e₂, ..., eₙ}
```

y una causa candidata:

```text
C
```

Definimos:

```text
Support(C)
```

como las evidencias compatibles con `C`.

Y:

```text
Contradict(C)
```

como las evidencias que reducen la plausibilidad de `C`.

---

# 264. Confidence conceptual

```text
Confidence(C) =
f(
    EvidenceQuality,
    Support,
    Contradictions,
    Coverage,
    Freshness,
    RuleSpecificity
)
```

---

# 265. No fórmula universal

VoltStack no deberá fingir precisión matemática donde sólo existen heurísticas.

Por tanto:

```text
0.8732 confidence
```

no será preferible automáticamente a:

```text
HIGH
```

si el número no posee significado estadístico real.

---

# 266. Root cause proof

Conceptualmente:

```text
PROVEN(C)
```

sólo será válido cuando exista evidencia que establezca directamente la relación causal bajo las reglas del dominio.

---

# 267. Temporal relationship

```text
A happened before B
```

sólo establece:

```text
TemporalPrecedence(A, B)
```

no:

```text
Caused(A, B)
```

---

# 268. Diagnostic graph formal

```text
G = (V, E)
```

donde `V` puede contener:

```text
Symptoms
Evidence
Findings
CandidateCauses
```

y las aristas:

```text
SUPPORTS
CONTRADICTS
CORRELATES_WITH
DERIVED_FROM
OBSERVED_BEFORE
OBSERVED_AFTER
```

---

# 269. Arquitectura final

```text
                    Diagnostic Request
                           │
                           ▼
                    DiagnosticPlanner
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
        Capabilities    Security       Budget
             │             │             │
             └─────────────┼─────────────┘
                           ▼
                     DiagnosticPlan
                           │
                           ▼
                    DiagnosticRunner
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
   Health Evidence    Live Collectors    Historical Data
        │                  │                  │
        └──────────────────┼──────────────────┘
                           ▼
                    EvidenceNormalizer
                           │
                           ▼
                     EvidenceSet
                           │
                           ▼
                   CorrelationEngine
                           │
                           ▼
                    DiagnosticRules
                           │
                           ▼
                       Findings
                           │
                           ▼
                    CandidateCauses
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
          Support      Contradiction   Coverage
             │             │             │
             └─────────────┼─────────────┘
                           ▼
                    ConfidenceEngine
                           │
                           ▼
                    DiagnosticReport
                           │
           ┌───────────────┼────────────────┐
           ▼               ▼                ▼
       Operator        Administration     Telemetry
```

---

# 270. Ejemplo integral

Supongamos:

```text
Application latency increased
```

Health detecta:

```text
Database Health
DEGRADED

Query Performance
DEGRADED

Connection Pool
DEGRADED
```

Diagnostics recopila:

```text
query latency        +480%
pool utilization      97%
pool wait             +650%
blocked transactions  84
long transaction       1
long transaction age  22 minutes
storage latency        normal
replication lag        normal
CPU                     normal
```

Correlation Engine encuentra:

```text
Long Transaction
      │
      ├── blocks 73 queries
      │
      └── started before latency spike
```

El reporte puede concluir:

```text
Candidate Cause:
Long-running transaction causing lock contention

Confidence:
HIGH
```

Pero no necesariamente:

```text
Root Cause:
Developer forgot to commit()
```

porque esa explicación todavía no está demostrada.

La siguiente investigación puede ser:

```text
Why did the transaction remain open?
```

---

# 271. Principio final

La filosofía del sistema será:

> **VoltStack Diagnostics no deberá intentar parecer inteligente inventando explicaciones; deberá ser útil haciendo explícita la relación entre síntomas, evidencia, hipótesis, contradicciones y nivel de confianza.**

Por tanto:

```text
Observation
    ↓
Evidence
    ↓
Correlation
    ↓
Hypothesis
    ↓
Confidence
```

nunca deberá reducirse a:

```text
Observation
    ↓
Guess
    ↓
"Root cause"
```

La arquitectura deberá ser capaz de responder tres cosas claramente:

```text
What do we know?

What do we suspect?

What do we not know?
```

Esta separación será una de las garantías fundamentales del sistema de diagnóstico de VoltStack.

---

# 272. Estado del Bloque 28

```text
BLOCK 28 — BACKUP AND OPERATIONS

✓ 276_DATABASE_BACKUP_ARCHITECTURE.md
✓ 277_DATABASE_BACKUP_SYSTEM.md
✓ 278_DATABASE_RESTORE_SYSTEM.md
✓ 279_DATABASE_DATABASE_MAINTENANCE_SYSTEM.md
✓ 280_DATABASE_HEALTH_CHECK_SYSTEM.md
✓ 281_DATABASE_DIAGNOSTICS_SYSTEM.md
→ 282_DATABASE_ADMINISTRATION_SYSTEM.md
```

---

# 273. Siguiente documento

```text
282_DATABASE_ADMINISTRATION_SYSTEM.md
```

Este documento cerrará el **Bloque 28 — Backup and Operations** definiendo la capa administrativa que coordinará de forma segura:

```text
Database Administration
│
├── Administration Context
├── Administration Commands
├── Database Inventory
├── Connection Administration
├── Session Administration
├── Transaction Administration
├── Query Administration
├── Lock Administration
├── Replica Administration
├── Replication Administration
├── Shard Administration
├── Tenant Administration
├── Schema Operations
├── Migration Operations
├── Maintenance Operations
├── Backup Operations
├── Restore Operations
├── Health
├── Diagnostics
├── Capacity
├── Security
├── Authorization
├── Audit
├── Dry Run
├── Confirmation
├── Approval
├── Safety Policies
├── Operation Planning
├── Execution
├── Cancellation
├── Progress
├── Result Reporting
├── CLI Integration
├── Administration API
└── Persistent Runtime Safety
```

manteniendo una separación crítica:

> **`Observation ≠ Administration ≠ Execution privilege`.**